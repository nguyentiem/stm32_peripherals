# ADC và DAC trên STM32H533CE: từ tín hiệu analog đến mẫu số

## Mục tiêu

Đọc được đặc tính ADC/DAC của **đúng STM32H533CE 48 chân**, chọn chân và nguồn tham chiếu, lấy mẫu đúng với trở kháng nguồn, kiểm tra sai số, rồi xây dựng chuỗi timer → ADC → DMA → xử lý → DAC. Chương này dùng cách học theo phụ thuộc: phần cứng analog → clock/reference → cấu hình → phép đo → đánh giá sai số.

Nguồn chính: [DS14539 Rev 3](../doc/DS_stm32h533ce%20%281%29.pdf), Table 2 tr. 15–16, bảng chân §4 tr. 61 trở đi, electrical characteristics phần ADC/DAC; [RM0481 Rev 5](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf), §28 ADC (tr. 1195 trở đi) và §30 DAC (tr. 1335 trở đi). Số trang là số in trên tài liệu.

## Khái niệm, nguyên lý và chọn chế độ

**ADC** biến điện áp liên tục thành mã số để firmware đo cảm biến, nguồn hay dòng điện; **DAC** đổi mã số thành mức điện áp để tạo ngưỡng, tín hiệu thử hoặc dạng sóng. ADC lần lượt chọn kênh, nạp tụ lấy mẫu, giữ điện áp rồi lượng tử hóa; DAC chốt mã từ thanh ghi dữ liệu và cho đầu ra ổn định sau thời gian settling. Độ phân giải 12 bit chỉ nói về bước lượng tử, không bảo đảm độ chính xác 12 bit. Tốc độ, sai số và công suất phụ thuộc clock, thời gian lấy mẫu, VREF, mạch nguồn và tải (RM0481 §28, §30; DS14539 phần ADC/DAC electrical characteristics).

| Chọn cấu hình | Vì sao có và cơ chế | Khi dùng, đánh đổi và lỗi dễ gặp |
|---|---|---|
| Single conversion, software trigger | Firmware quyết định thời điểm bắt đầu một lần đo. | Đo pin/nhiệt độ chậm; đơn giản nhưng thời điểm lấy mẫu có jitter và polling tốn CPU. |
| Continuous conversion | ADC tự bắt đầu lần kế tiếp khi xong lần trước. | Theo dõi tín hiệu biến thiên không cần mốc thời gian tuyệt đối; khó bảo đảm chu kỳ lấy mẫu chính xác nếu thay clock hoặc số kênh. |
| Timer external trigger | Cạnh timer khởi động phép đo tại mốc phần cứng. | Lọc số, đo pha, điều khiển; cần bảo đảm lần chuyển đổi trước đã xong trước trigger tiếp theo, nếu không sẽ overrun/mất mẫu. |
| Một kênh / scan nhiều rank | Sequencer chuyển qua danh sách kênh, kết quả đến theo thứ tự rank. | Nhiều cảm biến; mỗi kênh cần sampling time phù hợp, tín hiệu kênh trước có thể ảnh hưởng kênh sau khi nguồn có trở kháng cao. |
| Single-ended / differential nếu cặp kênh và cấu hình hỗ trợ | Đo so với mass analog hoặc đo hiệu điện áp giữa hai đầu vào. | Differential hữu ích với tín hiệu vi sai nhỏ và nhiễu chung, nhưng tốn cặp chân và phải kiểm tra dải common-mode trong DS. Không suy ra mọi chân đều tạo được cặp vi sai. |
| DAC software / timer trigger | Cập nhật theo lệnh firmware hoặc cạnh timer; buffer đầu ra cách ly phần nào mạng DAC khỏi tải. | Mức DC dùng software; waveform dùng timer + DMA. Buffer giúp lái tải trong giới hạn DS nhưng có headroom; tắt buffer chỉ khi tải và dải điện áp cho phép. |

**Chọn tần số lấy mẫu:** xác định băng thông tín hiệu và yêu cầu lọc chống alias ở đầu vào trước, rồi chọn tần số lấy mẫu, số kênh và sampling time sao cho tổng thời gian mỗi scan còn dư biên. DMA chỉ chuyển kết quả, không chữa được aliasing, điện áp chưa settle hay overrun. Với DAC, lọc thông thấp đầu ra khi cần dạng sóng mượt; tăng số mẫu mỗi chu kỳ làm tần số ra giảm nếu giữ nguyên tốc độ trigger.

## 1. Phạm vi phần cứng của CE

Theo DS14539 Table 2, H533CE có **2 ADC 12 bit**, **10 kênh ADC** được đưa ra theo cấu hình sản phẩm, và **1 DAC 12 bit với 2 kênh**. CE **không có VREFBUF**; không lập kế hoạch dùng điện áp tham chiếu từ buffer nội như một số mã H533 khác. RM0481 mô tả chung cả họ và cả ADC3 cho các mã khác: đừng suy ra CE có ADC3 hay mọi chân ADC của bảng chung.

Ví dụ chân có trên package 48: **PA0 = ADC12_INP0**, **PA4 = DAC1_OUT1**, **PA5 = DAC1_OUT2** (DS14539 bảng pin/ball §4.2, tr. 69–70). Chúng có thể trùng chức năng số; phải cấu hình analog và tránh tải/driver số tranh chân. `ADC12_INP0` nghĩa là kênh đầu vào được chia sẻ theo sơ đồ kết nối ADC1/2; kiểm tra chế độ ADC cụ thể trước khi dùng đồng thời.

## 2. Mô hình phép đo

ADC xấp xỉ điện áp đầu vào so với `VREF+`/`VREF−`. Với single-ended 12 bit, công thức lý tưởng gần đúng là `code ≈ Vin / VREF+ × 4095` khi `VREF− = VSSA`, trong khoảng cho phép. Đây là **mô hình tính toán**, không thay thế sai số offset, gain, INL/DNL, nhiễu, lượng tử hóa và sai số VREF. Một LSB lý tưởng bằng `VREF+ / 4096`; với VREF+ 3.3 V, khoảng 0.806 mV. Không báo kết quả mV chính xác hơn chuỗi đo cho phép.

Tụ sample-and-hold của ADC cần thời gian nạp. Nguồn tín hiệu trở kháng cao, RC lọc lớn, multiplexer hoặc op-amp không ổn định có thể làm mẫu đầu sai. Tăng **sampling time**, giảm trở kháng nguồn bằng buffer, và đo settling sau khi chuyển kênh. Thời gian chuyển đổi gồm sampling + conversion cycles; ADC clock quá nhanh không cứu được nguồn analog chưa kịp ổn định. Xem RM0481 §28.4–28.5 và thông số ADC trong DS electrical characteristics.

`VDDA`, `VSSA`, `VREF+` và layout quyết định chất lượng đo. Trên package CE, xem quan hệ chân nguồn analog với VREF trong DS; decoupling đặt sát chân, tránh dòng hồi xung số đi qua VSSA. Vì CE không có VREFBUF, phải biết điện áp reference thực sự từ mạch board. Không dùng `3300 mV` như hằng số nếu nguồn thay đổi.

### Kênh nội

Theo RM0481 §28.4.2 và sơ đồ §28.4.4: **ADC1 VINP16** đo `VSENSE` (cảm biến nhiệt), **ADC1 VINP17** đo `VREFINT`; **ADC2 VINP16** đo `VBAT/4`, **ADC2 VINP17** đo `VDDCORE`. Bật đường kênh nội theo bit điều khiển tương ứng và chờ thời gian ổn định theo RM/DS trước khi đo. `VREFINT` có thể suy ra VREF+ bằng hệ số hiệu chuẩn nhà máy nếu áp dụng đúng điều kiện hiệu chuẩn và địa chỉ hằng hiệu chuẩn của part. Nhiệt độ cần **hai điểm hiệu chuẩn** và công thức nội suy; giá trị ADC của VSENSE không phải °C trực tiếp. VBAT đọc là giá trị đã chia 4; trước khi suy ra VBAT, tính cả sai số bộ chia và VREF. Đừng đếm kênh nội như chân analog ngoài.

## 3. Trình tự ADC an toàn

1. Xác nhận chân ADC tồn tại trên LQFP48/UFQFPN48 và board đưa tín hiệu vào trong dải điện áp hợp lệ. Đặt GPIO analog mode, không pull.
2. Bật clock ADC và chọn kernel clock/prescaler trong giới hạn DS/RM. Bật regulator nội ADC, chờ thời gian ổn định.
3. **Calibration** trước khi enable/chuyển đổi; phân biệt single-ended/differential và kiểu offset/linearity theo RM và phiên bản HAL đang dùng. Lưu hoặc khôi phục hệ số khi chế độ nguồn yêu cầu.
4. Cấu hình resolution, alignment, một hay nhiều rank, sampling time **cho từng channel**, trigger software/timer, DMA hay interrupt. Thứ tự rank quyết định thứ tự buffer.
5. Đọc `EOC/EOS/OVR` hoặc callback HAL; xử lý overrun rõ ràng. Với dữ liệu liên tục, timer trigger + DMA circular hoặc double buffer cho chu kỳ mẫu đều. Không dùng vòng `HAL_ADC_Start` liên tục để giả định tốc độ mẫu cố định.
6. Đo điện áp đã biết tại ít nhất hai điểm và so với đồng hồ đo; báo cả sai số và điều kiện đo.

**Trigger và DMA:** timer TRGO tạo mốc lấy mẫu; ADC hoàn thành conversion rồi phát request DMA để chép `DR` vào SRAM. DMA không làm ADC nhanh hơn, chỉ giảm tải CPU và jitter phục vụ. Kiểm tra tốc độ mẫu × số kênh < năng lực ADC và bus; callback nửa/full buffer phải hoàn tất xử lý trước khi DMA ghi đè vòng tiếp theo.

## 4. DAC: phát mức và dạng sóng

DAC1 có hai output 12 bit. Theo DS §4.2, OUT1 ở PA4, OUT2 ở PA5 trên gói 48. Mô hình lý tưởng: `Vout ≈ D / 4095 × VREF+` (tùy cấu hình buffer/output và điều kiện tải; dùng đặc tính DAC trong DS để xác nhận swing thực tế). Kiểm tra output buffer, thời gian settling, tải điện trở/tụ và drive capability; DAC không phải nguồn công suất. Đặt chân ở analog mode.

Ghi `DHR12R1/2` rồi trigger sẽ chuyển sang `DOR`; với software trigger có thể phát DC chậm. Để phát sóng đều, dùng timer TRGO + DMA bảng mẫu; tính `f_out = f_trigger / N` với N mẫu/chu kỳ, rồi lọc tái tạo nếu cần. Tránh để timer chạy khi bảng DMA chưa sẵn sàng; chọn kiểu buffer/underrun theo RM0481 §30.

## 5. Ví dụ HAL: đọc PA0 và xuất mức tương ứng qua PA4

Tạo project STM32H533CE trong CubeMX/CubeIDE; cấu hình **ADC1 channel 0 trên PA0**, single-ended 12 bit, software trigger, một regular rank, sampling time đủ dài cho nguồn thử; **DAC1 channel 1 trên PA4**, software trigger, output buffer phù hợp tải. Hai chân đều analog mode. `MX_ADC1_Init()` và `MX_DAC1_Init()` do CubeMX sinh; hãy xem chúng để xác nhận rank, clock và trigger. Hàm calibration cụ thể có thể đổi theo phiên bản HAL, nên chạy bước calibration do HAL H5 của project cung cấp **trước** `HAL_ADC_Start` (RM0481 §28.5, ADC calibration).

```c
#include "main.h"

extern ADC_HandleTypeDef hadc1;
extern DAC_HandleTypeDef hdac1;

static void Error_Handler_Analog(void)
{
    __disable_irq();
    for (;;) { }
}

void Analog_Loop(void)
{
    /* Gọi sau MX_ADC1_Init(), MX_DAC1_Init() và ADC calibration. */
    if (HAL_DAC_Start(&hdac1, DAC_CHANNEL_1) != HAL_OK)
        Error_Handler_Analog();

    for (;;) {
        if (HAL_ADC_Start(&hadc1) != HAL_OK)
            Error_Handler_Analog();
        if (HAL_ADC_PollForConversion(&hadc1, 10u) != HAL_OK)
            Error_Handler_Analog();
        uint32_t raw = HAL_ADC_GetValue(&hadc1) & 0x0FFFu;
        if (HAL_ADC_Stop(&hadc1) != HAL_OK)
            Error_Handler_Analog();

        /* Chỉ hợp lệ khi ADC/DAC dùng cùng reference và cùng thang 12 bit. */
        if (HAL_DAC_SetValue(&hdac1, DAC_CHANNEL_1,
                             DAC_ALIGN_12B_R, raw) != HAL_OK)
            Error_Handler_Analog();
        HAL_Delay(10u); /* demo DC; KHÔNG dùng để tạo tần số mẫu chính xác */
    }
}
```

Code minh họa đường dữ liệu, **không phải một thiết kế lấy mẫu chính xác**. Nếu cần `f_s` cố định, chuyển sang timer trigger + DMA. Nếu đầu vào thay đổi nhanh, ADC và DAC không đồng thời; lượng trễ gồm chuyển đổi, polling, phần mềm và settling của DAC. Nối DAC ra ADC để tự kiểm tra cần chú ý biên swing/offset và không lấy kết quả loopback làm bằng chứng độ chính xác tuyệt đối vì chúng có thể chia sẻ cùng reference.

## 6. Lỗi thường gặp và cách chẩn đoán

| Triệu chứng | Kiểm tra trước |
|---|---|
| ADC luôn 0 hoặc full-scale | Chân/package, analog mode, nguồn VDDA/VSSA, VREF+, dây GND, channel/rank, điện áp quá dải. |
| Số đo đổi khi chuyển kênh | Sample time quá ngắn, trở kháng nguồn cao, tụ S/H còn điện tích kênh trước; thử tăng sample time và đo bỏ mẫu đầu. |
| Sai lệch cố định hoặc theo nhiệt | Calibration, VREF thực, offset/gain, nhiệt độ, chất lượng reference. |
| DMA mất mẫu | Overrun, tốc độ trigger quá cao, cấu hình width/length/circular, xử lý callback chậm. |
| DAC không đạt sát rail | Output buffer/headroom, tải quá nặng, VREF và dải điện áp hợp lệ. |
| Sóng DAC méo | Settling, tần số update, bus/DMA underrun, lọc đầu ra và tải tụ. |

## 7. Bài thực hành và tiêu chí đạt

1. **Đặc tuyến tĩnh:** cấp PA0 lần lượt 0.5, 1.0, 2.0 V từ nguồn đo được; ghi raw, VREF, sai số mV. Lặp với sampling time khác nhau và nguồn có điện trở nối tiếp 1 kΩ/100 kΩ.
2. **VREFINT:** dùng kênh ADC1 nội và hằng calibration đúng part để ước lượng VREF+; đối chiếu đồng hồ. Ghi rõ giả định và công thức.
3. **Timer + DMA:** lấy 1024 mẫu ở 10 kS/s; kiểm tra nhịp bằng GPIO toggle tại callback half/full và logic analyzer. Không in UART trong callback.
4. **DAC:** phát DC 25/50/75% full scale trên PA4, đo thực tế; sau đó phát sine 64 mẫu/chu kỳ qua timer + DMA. Tính tần số dự kiến và đo bằng scope.
5. **Thiết kế:** lập budget sai số gồm reference, ADC, nguồn tín hiệu, nhiệt và lượng tử hóa. Chỉ kết luận số bit hiệu dụng sau khi đo nhiễu/dải động thực tế.

**Checklist:** xác nhận 2 ADC/10 kênh/1 DAC hai kênh và không VREFBUF trên CE; chỉ đúng PA0/PA4/PA5 trên gói 48; biết vì sao calibration, sampling time, VREF và timer trigger quan trọng; xử lý được overrun/DMA buffer và giải thích giới hạn độ chính xác.

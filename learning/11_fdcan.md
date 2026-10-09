# FDCAN trên STM32H533CE: từ frame đến bus thực

> Nguồn: `doc/DS_stm32h533ce (1).pdf` (**DS14539 Rev 3**) và `doc/rm0481-...pdf` (**RM0481 Rev 5**). Số trang dẫn là số in trên PDF. Ví dụ dùng STM32CubeH5 HAL, cần đối chiếu phiên bản header, clock tree và chân của board trước khi biên dịch.

## 1. Mục tiêu và lộ trình

Học theo thứ tự: **CAN classic → arbitration/ACK/error → bit timing → transceiver và dây bus → filter/FIFO/message RAM → ngắt → CAN FD/BRS → kiểm thử bus lỗi**. Sau bài, bạn phải cấu hình được hai node trao đổi frame, giải thích vì sao transmit pending khi chỉ có một node normal mode, tính nominal/data bitrate và chẩn đoán bus-off.

## 2. H533CE có gì? Điểm cần đối chiếu

DS14539 bảng 2 tr. 15 ghi **2 FDCAN** cho H533CE; bảng chân/AF tr. 69, 73, 89–90 liệt kê cả FDCAN1 và FDCAN2 trên chân hiện diện của gói 48 chân. RM0481 §60.1 tr. 2887 mô tả hai module dùng message RAM chung, mỗi instance được cấp vùng khoảng **0,8 KB**. Tuy nhiên, câu đầu DS14539 §3.40 tr. 58 viết “one CAN module”, mâu thuẫn với bảng và pinout. Khi chọn sản phẩm, xác nhận thêm trong CubeMX/device header và thử bring-up từng instance; không giả định hai FDCAN có thể dùng chung một cặp chân.

Ví dụ chân trên gói 48 chân từ DS14539 bảng 14/16: FDCAN1 có `PA11_RX/PA12_TX` hoặc `PB8_RX/PB7_TX` (AF9); FDCAN2 có `PB12_RX/PB13_TX`, `PB5_RX/PB6_TX` hoặc `PA0_RX/PA10_TX` theo AF9 và chân thực tế. `PB9` không phải FDCAN1_TX trong bảng chân của DS14539. Chọn cặp không xung đột USB, timer, SPI hoặc chức năng board khác. Mỗi controller muốn nối bus cần **một transceiver riêng**.

## 3. CAN classic và CAN FD

| Thuộc tính | CAN classic | CAN FD |
|---|---|---|
| ID | 11 bit standard hoặc 29 bit extended | Tương tự |
| Payload | 0–8 byte | 0–64 byte theo mã DLC |
| Bitrate | Một pha nominal | Pha nominal cho arbitration, pha data có thể nhanh hơn nếu BRS=1 |
| CRC/format | Classic frame | FD frame khác định dạng; node classic không đọc được FD frame |

CAN dùng bus đa chủ. **ID nhỏ hơn có ưu tiên cao hơn** khi tranh quyền phát vì bit dominant thắng recessive; arbitration không phá hủy frame thắng. ID là nhãn thông điệp và mức ưu tiên, không phải địa chỉ node. Một frame normal mode cần node khác ACK đúng; chỉ có một transceiver/node trên bus thường không ACK nên bộ điều khiển thử lại, trừ chế độ loopback/test. RM0481 §60.1–60.3 tr. 2887–2893.

CAN FD với BRS chỉ tăng bitrate ở data phase; cả mạng phải có transceiver và node tương thích timing đó. Độ dài hợp lệ của FD gồm 0–8, 12, 16, 20, 24, 32, 48, 64 byte; DLC là mã, không phải số byte tuyến tính từ 9 trở lên. DS14539 §3.40 tr. 58–59 xác nhận hỗ trợ tối đa 64 byte.

## 4. Khung dây, xác nhận và bảo đảm ở đúng tầng

**Định nghĩa và lợi ích.** CAN là bus đa chủ ưu tiên theo nội dung: node có thể phát khi bus rỗi, ID quyết định arbitration không phá frame thắng. Truyền vi sai qua transceiver, bit stuffing, CRC và cơ chế lỗi giúp phù hợp điều khiển phân tán trong nhiễu công nghiệp/xe; nó không tự định nghĩa ý nghĩa payload. FDCAN của H533CE triển khai CAN classic và CAN FD; bộ điều khiển MCU xuất mức logic, còn transceiver chuyển thành CAN_H/CAN_L.

```text
Classic data frame: SOF | arbitration(ID, RTR/IDE) | control(DLC) |
                    DATA(0..8 byte) | CRC+delimiter | ACK slot+delimiter |
                    EOF | intermission
CAN FD: arbitration nominal | FDF, BRS, ESI, DLC | DATA(0..64 byte,
        data bitrate khi BRS) | FD CRC | ACK | EOF | intermission
```

Bit dominant ghi đè recessive. Node thua arbitration dừng phát và chờ bus rỗi; đây không phải lỗi truyền. Receiver kiểm bit stuffing, format và CRC; receiver nhận frame hợp lệ kéo ACK slot dominant. **ACK CAN chỉ cho biết ít nhất một node thấy frame hợp lệ**, không cho biết node mong muốn đã qua filter, FIFO còn chỗ hay ứng dụng đã xử lý. Nếu không ACK hoặc lỗi truyền, controller thường tự phát lại theo cấu hình; có thể xảy ra bản sao ở ứng dụng khi sender không biết receiver đã xử lý đến đâu. Bus-off dừng phát sau tích lũy lỗi; theo dõi TEC/REC và khôi phục có kiểm soát.

| Cơ chế/mode | Vấn đề giải quyết và lúc chọn | Giới hạn thực tế |
|---|---|---|
| Classic | Tương thích mạng CAN hiện hữu, payload tối đa 8 byte | Không dùng frame FD khi còn node classic không chịu được FD. |
| FD không BRS | Payload lớn hơn nhưng vẫn một bitrate | Chiếm bus lâu hơn FD BRS; mọi node phải hiểu FD. |
| FD + BRS | Pha data nhanh để giảm airtime | Timing, transceiver, topology và mọi node phải đạt tốc độ data. |
| Loopback/silent/normal | Loopback kiểm phần mềm; silent quan sát bus; normal tham gia ACK/phát | Loopback/silent không chứng minh ACK và chất lượng vật lý. |
| Auto retransmission | Tự xử lý lỗi bit/ACK tạm thời | Không thay timeout ứng dụng; bus hỏng có thể gây phát liên tục và trễ khó đoán. |

**Payload ứng dụng mẫu:** ID `0x123`, DLC 8: `[version:1][seq:1][status:1][value_le:2][reserved:1][app_crc16:2]`. Đây là *quy ước ứng dụng*, không thuộc CAN. CRC khung CAN bảo vệ dữ liệu trên dây, nhưng không chứng minh dữ liệu mới hoặc đúng ngữ nghĩa. `seq` hỗ trợ phát hiện mất/lặp bản tin, timeout giám sát phát hiện node câm; ACK ứng dụng nếu cần xác nhận xử lý thì dùng ID phản hồi riêng chứa `seq` và mã trạng thái. Với lệnh gây tác động, receiver lưu transaction ID và chỉ thực hiện một lần; sender retry có backoff và hạn mức. Không suy ra `HAL_FDCAN_AddMessageToTxFifoQ()` thành công nghĩa là frame đã ra dây; cần Tx event/completion, và ACK ứng dụng nếu yêu cầu end-to-end.

**Đồng bộ và chống nhiễu:** cạnh SOF đồng bộ bit timing, cạnh trong frame hỗ trợ tái đồng bộ theo SJW; bit stuffing duy trì chuyển mức và phát hiện lỗi stuffing. CAN CRC phát hiện frame lỗi; error flag phá frame lỗi để retry. Điện trở 120 Ω ở hai đầu, cặp xoắn, nhánh ngắn, transceiver đúng common-mode và tốc độ, bảo vệ nguồn/ESD quyết định mức nhiễu trước khi phần mềm xử lý. Đo CAN_H/CAN_L vi sai và log PSR/ECR khi tăng tải; phép đo 60 Ω chỉ thực hiện lúc bus mất nguồn và không thay kiểm tra dạng sóng.

## 5. Bit timing: tính rồi đo

Clock FDCAN có miền **APB register** và **kernel clock** riêng. Với nominal phase:

`bitrate = f_fdcan_ker / [NPrescaler × (1 + NTSEG1 + NTSEG2)]`

`sample_point = (1 + NTSEG1)/(1 + NTSEG1 + NTSEG2)`.

Data phase dùng `DataPrescaler`, `DataTimeSeg1`, `DataTimeSeg2` tương ứng. HAL nhận độ dài segment theo time quantum; register `NBTP/DBTP` mã hóa theo giá trị trừ 1. Ví dụ **giả định** `f_fdcan_ker=80 MHz`: nominal 500 kbit/s với prescaler 8, tổng 20 tq (`1+15+4`), sample point 80%; data 2 Mbit/s với prescaler 2, tổng 20 tq (`1+15+4`). SJW không vượt segment 2 và phải tính theo chênh lệch oscillator/đường truyền. Tốc độ data cao có thể cần transmitter delay compensation (TDC). RM0481 §60.3.3 tr. 2892–2894; `NBTP` §60.4.7 tr. 2924, `DBTP` §60.4.3 tr. 2920, `TDCR` §60.4.14 tr. 2931.

Không lấy `HAL_RCC_GetPCLKxFreq()` thay cho FDCAN kernel clock khi clock mux chọn nguồn khác. Cấu hình hai node phải tương thích **nominal bitrate**, mẫu lấy và sai số oscillator; data bitrate/BRS cũng phải khớp cho FD. Dùng analyzer đo thật trên CAN_H/CAN_L và kiểm register lỗi `ECR`, `PSR` nếu có lỗi.

## 6. Phần cứng: controller không nối thẳng CAN_H/CAN_L

`FDCAN_TX/RX` của MCU là tín hiệu logic, nối tới chân `TXD/RXD` của **CAN transceiver** phù hợp 3,3 V I/O. Transceiver mới nối tới `CAN_H/CAN_L`. Hai đầu **vật lý** của bus đặt điện trở 120 Ω giữa H và L; khi tắt nguồn, đo giữa H–L trên bus có hai termination sẽ xấp xỉ 60 Ω. Không đặt 120 Ω ở mọi node giữa đường. Dùng cặp xoắn, dây nhánh ngắn, chung tham chiếu GND theo thiết kế transceiver; đánh giá common-mode, bảo vệ ESD và cách ly galvanic nếu môi trường yêu cầu. Điện trở/EMC là thiết kế hệ thống, cần kiểm datasheet transceiver và tiêu chuẩn mạng thực tế.

Node đầu tiên nên thử loopback nội bộ để xác nhận cấu hình phần mềm, sau đó nối hai board có transceiver để kiểm ACK và arbitration. Loopback **không chứng minh** transceiver, termination hay dây bus hoạt động. Nếu board có chân standby/silent của transceiver, kéo về trạng thái active trước khi kết luận FDCAN lỗi.

## 7. Message RAM, filter và FIFO

Message RAM chứa standard/extended filters, Rx FIFO0/1, Tx FIFO/queue và Tx event FIFO. Trong CubeMX/HAL, đặt kích thước payload từng element, số phần tử, offset vùng của FDCAN1/2; tổng không được vượt vùng cấp. Vì hai controller dùng RAM chung, **offset của FDCAN2 phải bắt đầu sau vùng FDCAN1**, không chồng nhau. DS14539 §3.40 tr. 58–59; RM0481 §60.3.6–60.3.12 tr. 2900–2919. RM0481 §60.3.8–60.3.12 giải thích layout từng element.

Filter standard 11 bit và extended 29 bit là hai bảng riêng. Thứ tự filter và chính sách global filter quyết định frame không khớp đi đâu; cấu hình filter mà quên global filter có thể vẫn nhận mọi frame. Với traffic cao, FIFO 3 phần tử có thể overflow rất nhanh nếu callback xử lý lâu. Trong ISR, đọc header/data ra queue ứng dụng rồi xử lý ở task/main; đếm `Rx FIFO full/lost`, `Tx event`, `bus-off`, `protocol error`. `FDCAN_RXGFC` ở RM0481 §60.4.19 tr. 2937; FIFO status §60.4.22–25 tr. 2940–2942.

## 8. Ví dụ HAL: 2 node CAN classic 500 kbit/s

**Giả định:** `f_fdcan_ker=80 MHz`; FDCAN1 ở normal mode; nominal prescaler 8, seg1 15, seg2 4, SJW 4; `FrameFormat=FDCAN_FRAME_CLASSIC`; `RxFifo0ElmtsNbr ≥ 1`, `TxFifoQueueElmtsNbr ≥ 1`, data field 8 byte; đã cấu hình chân/AF, transceiver active, hai node trên bus và interrupt line 0. Các trường `MX_FDCAN1_Init()` do CubeMX sinh, nhưng phải kiểm giá trị thay vì tin mặc định.

```c
static volatile uint32_t can_rx_count;
static volatile uint32_t can_rx_error;
static uint8_t rx_payload[8];

void App_FdcanStart(void)
{
    FDCAN_FilterTypeDef filter = {0};
    filter.IdType = FDCAN_STANDARD_ID;
    filter.FilterIndex = 0;
    filter.FilterType = FDCAN_FILTER_MASK;
    filter.FilterConfig = FDCAN_FILTER_TO_RXFIFO0;
    filter.FilterID1 = 0x123;
    filter.FilterID2 = 0x7FF; /* Mask 11 bit: chỉ nhận ID 0x123. */
    if (HAL_FDCAN_ConfigFilter(&hfdcan1, &filter) != HAL_OK) Error_Handler();

    /* Chặn frame không khớp; xem chính sách remote frame của giao thức. */
    if (HAL_FDCAN_ConfigGlobalFilter(&hfdcan1,
        FDCAN_REJECT, FDCAN_REJECT,
        FDCAN_REJECT_REMOTE, FDCAN_REJECT_REMOTE) != HAL_OK) Error_Handler();

    if (HAL_FDCAN_Start(&hfdcan1) != HAL_OK) Error_Handler();
    if (HAL_FDCAN_ActivateNotification(&hfdcan1,
        FDCAN_IT_RX_FIFO0_NEW_MESSAGE | FDCAN_IT_BUS_OFF, 0) != HAL_OK)
        Error_Handler();
}

bool App_FdcanSendStatus(uint8_t status)
{
    FDCAN_TxHeaderTypeDef h = {0};
    uint8_t data[8] = {status};
    h.Identifier = 0x123;
    h.IdType = FDCAN_STANDARD_ID;
    h.TxFrameType = FDCAN_DATA_FRAME;
    h.DataLength = FDCAN_DLC_BYTES_8;
    h.ErrorStateIndicator = FDCAN_ESI_ACTIVE;
    h.BitRateSwitch = FDCAN_BRS_OFF;
    h.FDFormat = FDCAN_CLASSIC_CAN;
    h.TxEventFifoControl = FDCAN_NO_TX_EVENTS;
    return HAL_FDCAN_AddMessageToTxFifoQ(&hfdcan1, &h, data) == HAL_OK;
}

void HAL_FDCAN_RxFifo0Callback(FDCAN_HandleTypeDef *h, uint32_t flags)
{
    if (h->Instance != FDCAN1 || !(flags & FDCAN_IT_RX_FIFO0_NEW_MESSAGE)) return;
    FDCAN_RxHeaderTypeDef hdr;
    if (HAL_FDCAN_GetRxMessage(h, FDCAN_RX_FIFO0, &hdr, rx_payload) == HAL_OK) {
        if (hdr.IdType == FDCAN_STANDARD_ID && hdr.Identifier == 0x123 &&
            hdr.RxFrameType == FDCAN_DATA_FRAME && hdr.DataLength == FDCAN_DLC_BYTES_8)
            ++can_rx_count; /* Ví dụ tối giản: đẩy dữ liệu vào queue trong ứng dụng thực. */
    } else {
        ++can_rx_error;
    }
}
```

Trong ứng dụng thực, không dùng một `rx_payload` toàn cục nếu callback và main cùng đọc không đồng bộ; copy vào ring buffer và chặn overflow. Hàm phát phải kiểm còn slot Tx, có timeout ở tầng ứng dụng, và xử lý bus-off. Đừng coi `HAL_FDCAN_AddMessageToTxFifoQ == HAL_OK` là frame đã được node khác ACK; nó chỉ xác nhận được xếp hàng. Dùng Tx complete/event hoặc bus analyzer để xác nhận. Các tên macro có thể đổi theo phiên bản HAL; kiểm `stm32h5xx_hal_fdcan.h` của dự án.

## 9. CAN FD và xử lý lỗi

Sau khi CAN classic ổn định, bật `FDCAN_FRAME_FD_BRS`, cấu hình `DataPrescaler/DataTimeSeg1/DataTimeSeg2/DataSyncJumpWidth`, kích thước data field (ví dụ 64 byte) và header `FDFormat=FDCAN_FD_CAN`, `BitRateSwitch=FDCAN_BRS_ON`. Nếu Rx element vẫn cấp 8 byte nhưng nhận frame 64 byte, dữ liệu bị mất/cắt theo cấu hình; tăng payload làm giảm số phần tử khả dụng trong message RAM. Cấu hình cả hai đầu; node classic chung bus phải được đánh giá tương thích giao thức.

Theo dõi `PSR` (last error, activity, bus-off) và `ECR` (TEC/REC) cùng interrupt `IR`/`IE`/`ILS`/`ILE`; đọc RM0481 §60.4.12–18 tr. 2928–2937. Bus-off là trạng thái lỗi mạng, không phải lý do để reset MCU vô điều kiện. Chính sách khôi phục phải có giới hạn retry, log nguyên nhân, và tránh phát dồn dập làm hỏng bus chung.

## 10. Ghi chú kiểu senior và bài thực hành

1. **Thiết kế ID có ưu tiên.** Bản tin an toàn/điều khiển có ID thấp hơn telemetry; tính tải bus ở worst case, gồm bit stuffing và retry, không chỉ cộng payload.
2. **Kiểm thời gian end to end.** Bitrate đúng chưa đủ; queue Tx, arbitration, FIFO và scheduling tạo độ trễ. Đo từ sự kiện ứng dụng tới frame trên bus.
3. **Filter là tài nguyên và biên bảo vệ.** Lọc sớm trong hardware; vẫn xác thực DLC, ID, range, sequence counter và CRC tầng ứng dụng khi dữ liệu quan trọng.
4. **Dùng signal ground phù hợp.** CAN vi sai vẫn có giới hạn common-mode của transceiver; với dây dài/nguồn khác nhau, xem cách ly và bảo vệ surge.
5. **Pinout trước firmware.** FDCAN1 PA11/PA12 xung đột USB; chọn AF bằng sơ đồ board, không chỉ bằng danh sách chân của MCU.

| Bài | Tiêu chí tự kiểm |
|---|---|
| Loopback | Gửi/nhận ID 0x123; chứng minh filter chặn ID khác. |
| Bus thật | Hai board + hai transceiver + termination hai đầu; cả hai nhận, không bus-off. |
| Arbitration | Hai node phát cùng lúc ID khác; ID nhỏ hơn ra trước, frame còn lại vẫn truyền. |
| Timing | Tính và đo 500 kbit/s; đổi data phase 2 Mbit/s với FD/BRS và xác nhận bằng analyzer. |
| Lỗi vật lý | Tháo một termination hoặc đưa transceiver standby trong môi trường thử; ghi PSR/ECR và cách hồi phục. |
| Tải cao | Phát burst, đo FIFO lost/Tx pending; sửa queue/ISR để không mất frame trong tải mục tiêu. |

Một bài đạt khi có sơ đồ dây, nguồn kernel clock, bảng bit timing, cấu hình message RAM/filter, capture bus và log lỗi. Đây là dữ liệu cần giữ để người sau tái hiện được kết quả.

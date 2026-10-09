# Timer trên STM32H533CE: từ bộ đếm đến PWM và đồng bộ ngoại vi

> Phạm vi: **STM32H533CE**, datasheet **DS14539 Rev 3** và reference manual **RM0481 Rev 5**. Trang trích dẫn là số trang in trên PDF. Ví dụ C dùng STM32Cube HAL; tên chân, alternate function (AF), nguồn clock và phiên bản HAL phải được xác nhận trong dự án thực tế. Timer trong RM0481 dùng chung cho nhiều chip, vì vậy chỉ dùng các instance được datasheet xác nhận.

## 1. Kết quả cần đạt

Sau chủ đề này, bạn cần tự tính được PSC/ARR/CCR cho chu kỳ và độ phân giải; biết vì sao APB prescaler làm clock timer thay đổi; cấu hình update interrupt, PWM, input capture và trigger ADC/DAC; chọn đúng timer cho bài toán; dùng oscilloscope hoặc logic analyzer để kiểm tra thay vì chỉ tin vào giá trị thanh ghi. Thứ tự học: **time base → PWM → capture → trigger → advanced PWM → LPTIM**.

## 2. Bản đồ timer của H533CE

| Nhóm | Instance | Đặc điểm | Chọn khi |
|---|---|---|---|
| Advanced | TIM1, TIM8 | 16 bit; 4 kênh capture/compare; 3 ngõ ra bổ sung, dead time, break, repetition counter; đếm lên/xuống/center aligned | Điều khiển công suất, motor, PWM cần đầu ra bổ sung và trạng thái lỗi |
| General purpose đầy đủ | TIM2, TIM5 | 32 bit; 4 kênh, encoder, PWM, input capture | Đo khoảng thời gian dài, timestamp, encoder |
| General purpose đầy đủ | TIM3, TIM4 | 16 bit; 4 kênh, encoder, PWM, input capture | PWM nhiều kênh, đo xung thông thường |
| General purpose gọn | TIM12, TIM15 | 16 bit, đếm lên, tối đa 2 kênh; **TIM15** có ngõ ra bổ sung/dead time, TIM12 không có ngõ ra bổ sung; tính năng khác nhau theo instance | PWM/capture ít kênh, tiết kiệm timer lớn |
| Basic | TIM6, TIM7 | 16 bit, không có chân capture/PWM; update event và TRGO | Time base, kích DAC/ADC nếu sơ đồ trigger cho phép |
| Low power | LPTIM1, LPTIM2 | 16 bit, 2 kênh, nguồn clock độc lập; có thể đánh thức từ Stop khi chọn clock phù hợp | Thức dậy định kỳ, đếm xung khi tiết kiệm điện |

Nguồn: DS14539 §3.33, bảng 7, tr. 45–47; RM0481 §43–48, đặc biệt bảng 527 tr. 2128 (chỉ LPTIM1/2 được triển khai trên H533). **Không suy ra chân sử dụng được từ tên kênh:** xem pinout và bảng AF của đúng gói 48 chân trong DS14539 §4.

## 3. Mô hình thời gian: clock → prescaler → counter → sự kiện

**Định nghĩa.** Timer là bộ đếm phần cứng cập nhật theo clock, so sánh với ngưỡng rồi tạo event, interrupt, DMA request hoặc tín hiệu ra chân. Nó giải quyết timing mà vòng lặp CPU khó giữ chính xác khi có ngắt và tải biến động. Với STM32H533CE, chọn TIM thường, TIM nâng cao hay LPTIM theo độ dài bộ đếm, tín hiệu cần tạo/đo và việc có phải chạy trong Stop hay không; instance cụ thể được liệt kê ở §2. [RM0481 §15, §43–48; DS14539 bảng 2](../doc/DS_stm32h533ce%20%281%29.pdf).

| Mode/cách dùng | Vì sao có và cách hoạt động | Khi chọn; đánh đổi/lỗi |
|---|---|---|
| Upcount time base | `CNT` tăng tới `ARR`, phát update rồi quay về đầu; `PSC` giảm tốc độ tick. | Scheduler nhịp, timeout, trigger lấy mẫu; ISR chỉ đặt cờ, không làm việc dài. Đổi APB/clock tree phải tính lại chu kỳ. |
| Downcount / center-aligned | Đếm xuống hoặc lên-xuống để đặt thời điểm cạnh theo yêu cầu; center-aligned tạo PWM đối xứng. | Điều khiển motor/inverter cần phổ hài và cân đối cạnh; tần số update khác edge-aligned, tính lại công thức và trigger ADC. |
| One-pulse | Sau trigger timer tạo đúng một xung rồi dừng. | Xung kích triac, trigger cảm biến, timeout một lần; tránh jitter khởi động bằng phần mềm. Muốn lặp phải arm lại đúng trình tự. |
| Output compare/PWM | So `CNT` với `CCR` để đổi mức ra; PWM lặp có duty do `CCR` quyết định. | LED, servo, nguồn công suất, waveform; dùng preload để đổi `CCR/ARR` ở update, tránh chu kỳ méo khi ghi giữa vòng. |
| Input capture | Chụp `CNT` vào `CCR` đúng lúc cạnh ngoài đến, độc lập độ trễ ISR. | Đo chu kỳ/pulse width/encoder; xử lý overflow, mất cạnh, overcapture và filter để tránh nhiễu giả. |
| Encoder/external clock | Cạnh ở đầu vào quyết định hướng hoặc nhịp đếm thay vì clock nội. | Núm xoay, vị trí trục, đếm xung cảm biến; filter giảm bounce nhưng có thể bỏ xung nhanh. |
| Slave trigger / TRGO | Một timer phát event để timer/ADC/DAC khác nhận theo đường nội. | Lấy mẫu ADC cùng pha PWM; kiểm tra bảng liên kết ITR/TRGO của đúng instance, vì không phải mọi cặp đều nối được. |
| Complementary PWM + dead time/break | TIM1/TIM8 chèn khoảng cả hai transistor cùng tắt và có đường ngắt phần cứng. | Half-bridge; dead time giải quyết shoot-through nhưng làm méo duty hữu dụng, break phải được thử với tầng công suất ở trạng thái an toàn. |

Một timer có thể có nhiều mode nhưng không thể coi chúng là các công tắc độc lập: lựa chọn chế độ đếm, trigger, preload và clock quyết định **lúc nào** event xuất hiện. Khi thay mode, vẽ timeline `CNT`, `ARR`, `CCR`, update và cạnh pad rồi đo bằng logic analyzer. Ví dụ center-aligned đi lên rồi xuống nên hai lần chạm cùng `CCR` trong một vòng đầy đủ; nếu dùng công thức tần số PWM edge-aligned sẽ sai hệ số. [RM0481 §43–47](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf).

Với timer thông thường ở chế độ đếm lên, `f_CNT = f_TIM / (PSC + 1)`, `f_update = f_CNT / (ARR + 1)` và `T_update = (PSC + 1)(ARR + 1)/f_TIM`. `PSC` là giá trị thanh ghi, nên `PSC=249` nghĩa là chia 250. Tương tự, bộ đếm đi từ 0 đến ARR, gồm `ARR+1` tick. Với PWM mode 1, đếm lên, active high, duty gần bằng `CCR/(ARR+1)` nếu `0 ≤ CCR ≤ ARR+1`; trường hợp 0%/100% và polarity cần đối chiếu cách cấu hình output. Ở center aligned, một vòng PWM gồm hai lượt đếm; không áp công thức edge aligned một cách máy móc. RM0481 §43.3, §44.4; DS14539 bảng 7 tr. 45.

**Clock timer không nhất thiết bằng PCLK.** RM0481 §11.4.14 tr. 503: nếu APB prescaler = 1 thì `f_TIM=f_PCLK`; nếu APB prescaler khác 1 thì `f_TIM=2×f_PCLK`. Phải xác định timer nằm trên APB nào và đọc cấu hình RCC thực tế. Hàm `HAL_RCC_GetPCLKxFreq()` một mình chưa đủ khi APB bị chia. Trên HAL, trường `Prescaler` của `TIM_Base_InitTypeDef` là giá trị PSC, không phải hệ số chia.

**Ví dụ tính:** `f_TIM=200 MHz`, muốn tick 1 MHz: `PSC=199`. Muốn update mỗi 1 ms: `ARR=999`. Muốn PWM 20 kHz với độ phân giải cao hơn: chọn `PSC=0`, `ARR=9999`; duty 25% tương ứng `CCR=2500`. Hai cấu hình phục vụ mục tiêu khác nhau; chọn PSC nhỏ khi ưu tiên độ phân giải PWM, chọn tick dễ đổi đơn vị khi ưu tiên đo thời gian.

### Thanh ghi và sự kiện phải hiểu

| Thanh ghi/bit | Ý nghĩa và lỗi thường gặp |
|---|---|
| `CR1.CEN`, `DIR`, `CMS`, `ARPE` | Bật counter, chiều/center mode, preload ARR. Thay ARR khi đang chạy có thể áp ngay hoặc tại update tùy ARPE. |
| `PSC`, `ARR`, `CNT` | Tần số tick, chu kỳ, giá trị hiện tại. PSC được nạp vào phần đang dùng tại update event. |
| `CCR1..4`, `CCMRx.OCxPE`, `CCER` | So sánh/capture, preload CCR, bật kênh và polarity. Capture dùng cùng CCR nhưng với nghĩa khác PWM. |
| `EGR.UG`, `SR.UIF`, `DIER.UIE` | Tạo update bằng phần mềm, cờ update, cho phép ngắt. `UG` có thể khiến UIF được đặt: xử lý cờ trước khi bật IRQ. |
| `CR2.MMS`, `SMCR`, `TRGO` | Chọn sự kiện master và nguồn/kiểu đồng bộ slave; đường nối phải tra bảng interconnect. |
| `BDTR.MOE`, `DTG`, `BKE` | Main output enable, dead time và break trên timer advanced; thiếu MOE thì counter chạy nhưng chân không phát PWM. |

Tham khảo RM0481 §43.6 tr. 1770+, §44.5 tr. 1900+, §45.4 tr. 1952+.

## 4. Bài 1: TIM6 tạo nhịp 1 ms không chặn CPU

Giả sử đã đo/thiết lập `TIM6CLK=200 MHz`. Cấu hình CubeMX: bật TIM6, internal clock, prescaler 199, period 999, bật **TIM6 global interrupt** trong NVIC. TIM6 là lựa chọn hợp lý cho time base vì không chiếm kênh PWM. Khi HAL đã tạo `htim6`, dùng:

```c
/* TIM6: 200 MHz / 200 / 1000 = 1 kHz update. */
static volatile uint32_t tick_ms;

void App_TimerStart(void)
{
    if (HAL_TIM_Base_Start_IT(&htim6) != HAL_OK) Error_Handler();
}

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM6) {
        ++tick_ms;                 /* ISR ngắn; xử lý việc nặng ở main/task. */
    }
}

uint32_t App_GetTickMs(void)
{
    return tick_ms;                /* Đọc 32 bit aligned là atomic trên Cortex-M33. */
}
```

Không dùng callback này để `HAL_Delay()`, printf chặn hoặc xử lý dài. Nếu cần thời gian chính xác cao, ISR có jitter do NVIC và tải CPU; dùng chân output compare hoặc timer trigger phần cứng. RM0481 §45.1–45.3, tr. 1939–1951.

## 5. Bài 2: PWM và thay đổi duty an toàn

Chọn một chân `TIM3_CH1` **thực sự được đưa ra trên gói CE** từ bảng AF. Ví dụ `TIM3CLK=200 MHz`, `PSC=0`, `ARR=9999` cho PWM 20 kHz. Trong CubeMX đặt TIM3 Channel 1 = PWM Generation CH1, PWM mode 1, polarity high, pulse ban đầu 0, bật preload nếu muốn thay duty tại biên chu kỳ.

```c
void App_PwmStart(void)
{
    if (HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1) != HAL_OK) Error_Handler();
}

/* duty_permille: 0..1000; ARR=9999. */
void App_PwmSetDuty(uint16_t duty_permille)
{
    if (duty_permille > 1000u) duty_permille = 1000u;
    uint32_t period = __HAL_TIM_GET_AUTORELOAD(&htim3) + 1u;
    uint32_t ccr = (period * duty_permille) / 1000u;
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, ccr);
}
```

Nếu `CCR=ARR+1` không được chấp nhận trong cấu hình cụ thể, dùng forced active/inactive mode cho mức 100%/0%. Đo tần số và duty trên chân, không suy diễn từ `CNT` đơn lẻ. Nếu thay cả ARR và CCR lúc chạy, bật preload cho cả hai để tránh một chu kỳ có giá trị pha trộn; xác nhận thời điểm chuyển tại update. RM0481 §44.4, các mục output compare/PWM; §44.5.

## 6. Bài 3: Input capture đo chu kỳ xung

Ví dụ dùng TIM2_CH1 vì counter 32 bit ít tràn hơn. Cấu hình timer tick 1 MHz, input capture direct TI rising edge, thêm digital filter phù hợp nhiễu, bật capture interrupt. `delta` là hiệu modulo 32 bit nên đúng nếu giữa hai cạnh không quá một vòng counter; việc xử lý chu kỳ sau nhiều vòng cần thêm biến đếm overflow hoặc timer chậm hơn.

```c
static volatile uint32_t period_us;
static uint32_t last_capture;
static bool have_last;

void App_CaptureStart(void)
{
    if (HAL_TIM_IC_Start_IT(&htim2, TIM_CHANNEL_1) != HAL_OK) Error_Handler();
}

void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance != TIM2 || htim->Channel != HAL_TIM_ACTIVE_CHANNEL_1) return;
    uint32_t now = HAL_TIM_ReadCapturedValue(htim, TIM_CHANNEL_1);
    if (have_last) period_us = now - last_capture;  /* unsigned modulo 2^32 */
    last_capture = now;
    have_last = true;
}
```

`f_signal ≈ 1,000,000/period_us` chỉ khi tick thật là 1 MHz và `period_us != 0`. Dùng capture trực tiếp để timestamp cạnh; đừng timestamp bằng `HAL_GetTick()` trong EXTI nếu cần độ phân giải microsecond. Kiểm tra cờ **overcapture** khi tín hiệu quá nhanh so với thời gian phục vụ ISR; khi đó chuyển sang DMA. Đo duty bằng cả rising/falling capture hoặc PWM input mode với hai CCR và reset slave mode. RM0481 §44.4 (input capture, PWM input, encoder), tr. 1824+; §44.4.29 tr. 1899.

## 7. Trigger và đồng bộ phần cứng

Đường đi chuẩn: **TIMx update/compare → TRGO/TRGO2 → ADC hoặc DAC → DMA → buffer**. Ví dụ TIM6 có thể xuất TRGO từ update để kích DAC mà không cần CPU vào ISR mỗi mẫu. Cấu hình timer master `TRGO = UPDATE`, trong ADC/DAC chọn đúng external trigger tương ứng theo bảng trigger của ngoại vi; bắt đầu DMA và ngoại vi trước, sau đó mới bật timer. Trước khi mã hóa, tra RM0481 §15.3.1 tr. 666 (timer master/slave) và bảng trigger của **ADC/DAC**; không phải mọi TIMx_TRGO đều nối tới mọi ADC/DAC trên H533CE. TIM6/TIM7 nội bộ không xuất chân PWM (RM0481 §45.3, tr. 1940–1951).

Đối với hai timer đồng bộ, chọn master TRGO (update hoặc OC event), slave ITR đúng theo bảng interconnect, rồi chọn `SMS=reset/trigger/gated` tùy yêu cầu. `reset` canh lại phase, `trigger` khởi chạy, `gated` chỉ đếm khi cổng cho phép. Kiểm bằng hai chân quan sát được trên logic analyzer; chỉ nhìn giá trị cấu hình không chứng minh được phase. RM0481 §44.4.23, tr. 1891–1898.

## 8. TIM1/TIM8: PWM cho tầng công suất

TIM1/8 hỗ trợ center aligned, cặp CHx/CHxN, **dead time** để hai khóa cùng nhánh không bật đồng thời, **break** để đưa đầu ra về trạng thái đã định khi lỗi, repetition counter để giảm tần suất update. Trình tự kỹ thuật: (1) tính clock, tần số PWM và độ phân giải; (2) xác nhận sơ đồ chân CHx/CHxN/BKIN trên package; (3) đặt polarity, idle state, dead time theo datasheet MOSFET/gate driver; (4) kiểm `BDTR`, break và MOE; (5) chỉ sau đó cấp năng lượng tầng công suất. RM0481 §43.3, tr. 1680–1768 và §43.6, tr. 1770–1821; DS14539 §3.33.1 tr. 45–46.

```c
/* Chỉ là lệnh khởi động sau khi CubeMX đã cấu hình TIM1 PWM, dead time và break. */
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_1);
/* Kiểm chứng CH1/CH1N và phản ứng BKIN bằng oscilloscope trước khi nối tải. */
```

Đừng tự suy ra công thức dead time từ giá trị `DTG` như một prescaler tuyến tính: mã hóa DTG chia vùng và phụ thuộc clock tDTS. Tính theo mô tả `BDTR.DTG` trong RM0481 rồi đo thời gian không chồng dẫn ở cả hai cạnh. `MOE` có thể bị xóa khi break; cần chính sách khôi phục lỗi rõ ràng. Với center aligned, sự kiện update, compare và hướng đếm ảnh hưởng điểm lấy mẫu ADC; kiểm đồ thị thời gian thay vì chỉ tần số trung bình.

## 9. LPTIM1/2: thời gian khi hệ thống ngủ

LPTIM có clock kernel riêng, prescaler `1,2,4,8,16,32,64,128`, counter/ARR 16 bit, 2 kênh, filter và khả năng đánh thức từ Stop. Theo DS14539 §3.33.4 tr. 47, nó có thể chạy trong Stop nếu clock từ LSE, LSI hoặc external clock; APB/HSI không được mặc định là còn chạy trong Stop. RM0481 §48.3 bảng 527 tr. 2128 và §48.4.4 tr. 2133.

Ví dụ dùng LSE danh định 32 768 Hz, prescaler 32 ⇒ tick 1024 Hz. Muốn ngắt xấp xỉ 1 giây, đặt ARR theo cách đếm của LPTIM và đo thực tế; **không sao chép công thức `ARR+1` của TIM thường một cách mù quáng**, vì cơ chế khởi động, nạp ARR và cờ đồng bộ của LPTIM khác. Khi ghi ARR/CCR qua hai clock domain, đợi cờ `ARROK`/`CMPOK` hoặc để HAL xử lý; ghi tiếp quá sớm có thể mất cập nhật. Đọc RM0481 §48.4.4–48.4.14, tr. 2133–2141, §48.5 tr. 2150. Nếu dùng filter khi external clock, phải có clock nội cho mạch filter; nếu không, tắt filter (RM0481 §48.4.5 tr. 2134).

## 10. Ghi chú kiểu senior

1. **Clock là một phần của phép đo.** Thay PLL/APB/VOS hoặc vào low power mode làm cấu hình thời gian cần được tính lại. Ghi `f_TIM` và cách suy ra nó ngay cạnh thông số PSC/ARR trong thiết kế.
2. **Ưu tiên phần cứng cho timing chính xác.** ISR phù hợp điều phối và thống kê; PWM, capture, trigger và DMA mới giữ được thời điểm cạnh ổn định dưới tải CPU.
3. **Đặt giới hạn rõ ràng.** Với timer 16 bit, thời gian tối đa một vòng là `(PSC+1)×65536/f_TIM`. Đo chu kỳ dài hơn cần TIM2/5 32 bit hoặc overflow accounting.
4. **Tách đơn vị.** Tên biến `timer_hz`, `tick_hz`, `period_us`, `duty_permille`; dùng số nguyên 64 bit khi nhân chia có thể tràn 32 bit. Kiểm lỗi chia 0 ở tín hiệu mất xung.
5. **Đừng nhầm update event và interrupt.** Event có thể nạp preload, phát TRGO/DMA; interrupt chỉ phát khi DIER/NVIC cho phép. `EGR.UG` dùng để nạp PSC trước khi bắt đầu nhưng cũng có thể làm cờ UIF xuất hiện.
6. **Kiểm chứng trên chân vật lý.** AF, pull, tốc độ GPIO, tải điện, polarity và clock đều có thể làm sóng khác thanh ghi. Với PWM công suất, kiểm trên mạch chưa cấp tải và thử break fail safe.
7. **Khóa đồng thời.** Biến đọc/ghi từ ISR phải `volatile`; cấu trúc dữ liệu nhiều trường cần critical section hoặc snapshot nhất quán. `volatile` không thay thế cơ chế đồng bộ.

## 11. Bài tập và tiêu chí hoàn thành

| Bài | Yêu cầu | Tự kiểm |
|---|---|---|
| A. Time base | TIM6 update 1 kHz, LED đổi trạng thái mỗi 500 tick | Logic analyzer thấy chu kỳ LED 1 s; CPU không busy wait |
| B. PWM | TIM3_CH1 20 kHz, duty 10/50/90% | Sai số tần số giải thích được từ clock; duty đo đúng; đổi duty không có xung bất thường |
| C. Capture | TIM2 đo tín hiệu 100 Hz–100 kHz từ signal generator | So với nguồn chuẩn; thử mất tín hiệu và overcapture |
| D. Trigger | TIM6 TRGO kích ADC/DAC hoặc timer khác qua đường được RM xác nhận | Khoảng mẫu không phụ thuộc tải CPU |
| E. Advanced | TIM1 CH1/CH1N, dead time, BKIN | Hai xung không chồng dẫn; BKIN làm đầu ra về trạng thái an toàn |
| F. Low power | LPTIM1 với LSE đánh thức từ Stop | Đo dòng điện và chu kỳ wakeup, ghi sai số LSE/LSI |

Một bài chỉ hoàn thành khi có **cấu hình clock**, **giá trị PSC/ARR/CCR**, **ảnh/ghi chú kết quả đo**, và **giải thích sai số**. Xem thêm pinout/AF trong DS14539 §4 và sơ đồ clock/triggers trong RM0481 §11, §15 trước khi nối phần cứng.

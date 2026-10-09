# RTC, LPTIM và watchdog trên STM32H533CE

> Nguồn chính: **DS14539 Rev 3** (STM32H533xx datasheet) và **RM0481 Rev 5**. Số trang bên dưới là số in trên PDF trong `doc/`. Bài này chỉ áp dụng các instance hiện diện trên **H533CE**. Ví dụ HAL giả định STM32CubeH5 đã tạo handle và cấu hình RCC/GPIO/NVIC; kiểm tra tên hàm, macro và chân theo phiên bản HAL đang dùng.

## 1. Mục tiêu và thứ tự học

**Định nghĩa và vai trò.** RTC là đồng hồ lịch trong backup domain để giữ thời điểm qua reset/mất VDD khi có VBAT và nguồn clock phù hợp. LPTIM là bộ đếm dùng clock thấp để đo/định thời ngay cả khi CPU ngủ. Watchdog là bộ đếm buộc reset khi phần mềm không chứng minh được tiến triển đúng hạn. Ba khối này giải quyết ba vấn đề khác nhau: *thời điểm*, *khoảng thời gian* và *khả năng phục hồi*. Chúng chỉ có ích khi clock, nguồn nuôi và đường đánh thức/reset được thiết kế trọn vẹn. [RM0481 §10.8, §48–51; DS14539 bảng 2](../doc/DS_stm32h533ce%20%281%29.pdf).

| Mode/cơ chế | Tại sao tồn tại, cách hoạt động | Khi dùng, đánh đổi và lỗi điển hình |
|---|---|---|
| RTC calendar/alarm | Prescaler biến clock thấp thành nhịp lịch; alarm so ngày/giờ với mốc đã đặt. | Sự kiện theo lịch, ví dụ mở van 06:00; cần đồng bộ giờ ban đầu, xử lý múi giờ ở ứng dụng, không dùng alarm như timeout ngắn chính xác. |
| RTC wakeup timer | Bộ đếm riêng tạo IRQ định kỳ từ nguồn RTC. | Logger thức mỗi phút; ít cấu hình hơn LPTIM nhưng chức năng timing hạn chế. Phải kiểm tra cờ/EXTI và sai số oscillator. |
| LPTIM internal clock | Counter chạy từ LSE/LSI hoặc nguồn kernel thích hợp khi core dừng. | Định thời/wakeup Stop hoặc PWM chậm; chờ cờ đồng bộ khi ghi ARR/CCR qua hai clock domain. |
| LPTIM external clock/counter | Xung ở chân ngoài thành nhịp đếm thay vì clock hệ thống. | Đếm xung lưu lượng khi CPU ngủ; filter cần clock nội và xung quá nhanh/cạnh nhiễu có thể sai số. |
| Sleep | Lõi dừng, ngoại vi clock thông thường còn chạy theo cấu hình. | Chờ ngắt ngắn, wake nhanh nhưng tiết kiệm điện ít hơn Stop. SysTick có thể đánh thức liên tục. |
| Stop | Dừng clock nhanh, giữ trạng thái cần thiết và cho phép nguồn thấp đánh thức. | Chờ giây/phút; sau wake phải phục hồi SYSCLK/PLL và các ngoại vi phụ thuộc clock. |
| Standby | Tắt sâu hơn, thức dậy qua đường boot/reset với miền backup còn theo nguồn. | Chờ dài, pin; phải lưu trạng thái ứng dụng cần giữ và thiết kế boot idempotent. |
| IWDG / WWDG | IWDG dùng LSI độc lập để bắt treo/mất clock chính; WWDG kiểm tra cả refresh quá sớm và quá muộn theo cửa sổ. | IWDG làm lưới an toàn cuối, WWDG bắt lỗi nhịp scheduler; thiết kế sai timeout hoặc refresh trong ISR có thể che lỗi hoặc reset giả. |

**Tính mode bằng ngân sách hệ thống.** Với Stop/Standby, xác định tối đa được ngủ bao lâu trước deadline, nguồn nào vẫn chạy, dữ liệu nào phải giữ, thời gian và năng lượng phục hồi. Với IWDG, đặt timeout lớn hơn thời gian chạy tệ nhất *đã đo* (kể cả Flash erase, truyền lỗi, startup và nhiệt độ biên) nhưng nhỏ hơn thời gian an toàn của tải. Với WWDG, chứng minh cả cận refresh sớm và trễ. Sau reset, giữ output điều khiển tải ở trạng thái an toàn rồi mới khôi phục chức năng. [RM0481 §10.8, §49–50](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf).

Hiểu sự khác nhau giữa **giữ thời gian** (RTC), **định thời khi ngủ** (LPTIM), **phát hiện chương trình lỗi** (IWDG/WWDG), và **chế độ nguồn** (Sleep/Stop/Standby). Lộ trình: clock LSE/LSI → backup domain và RTC calendar → RTC wakeup → LPTIM → IWDG → WWDG → đo dòng và kiểm tra reset. Đầu ra thực hành là log nguyên nhân reset, thời gian thức dậy và dòng tiêu thụ đo được.

## 2. Những gì có trên H533CE

| Khối | Có trên CE | Chức năng chính |
|---|---:|---|
| RTC và backup | Có; backup SRAM 2 KB | Calendar, alarm A/B, periodic wakeup, timestamp; miền backup có thể được cấp bởi VBAT |
| LPTIM1, LPTIM2 | 2 timer 16 bit | Đếm/PWM/capture 2 kênh; đánh thức từ Stop với nguồn clock phù hợp |
| IWDG | 1 | Watchdog clock LSI độc lập; bảo vệ cả khi CPU hoặc clock chính gặp lỗi |
| WWDG | 1 | Watchdog cửa sổ dựa trên clock hệ thống/APB; phát hiện refresh quá sớm lẫn quá muộn |

DS14539 bảng 2 tr. 15–16; §3.33.4–3.34 tr. 47–49. RM0481 bảng 527 tr. 2128 xác nhận dòng H523/533 chỉ triển khai **LPTIM1/2**, dù RM mô tả thêm LPTIM3–6 cho chip khác.

## 3. Clock, nguồn và miền backup

**LSE** là nguồn 32.768 kHz ngoài, phù hợp RTC ổn định nếu board gắn thạch anh; **LSI** là RC nội danh định 32 kHz, ít linh kiện nhưng tần số sai lệch theo chip/nhiệt/điện áp. Chọn RTC clock trong `RCC_BDCR.RTCSEL`; thay lựa chọn này thường cần reset backup domain, đồng nghĩa mất dữ liệu backup và thời gian đã đặt. Khi `VDD` mất và `VBAT` còn, LSE cùng RTC có thể tiếp tục trong backup domain; LSI/HSE không được bảo đảm duy trì như LSE. RM0481 §11.4.13 tr. 503 và §10.4.7 tr. 435.

**Không reset backup domain mỗi lần boot.** Khởi tạo RTC theo marker backup register/version: nếu marker và checksum hợp lệ, giữ giờ cũ; nếu không, đặt giờ mặc định từ nguồn tin cậy. Nếu ứng dụng yêu cầu thời gian chuẩn, RTC vẫn cần đồng bộ từ GNSS/NTP/thiết bị chủ và hiệu chỉnh sai số oscillator. RTC không tự tạo ra giờ UTC chính xác sau lần cấp nguồn đầu tiên.

```c
/* Ý tưởng khởi tạo sau MX_RTC_Init(); chỉ chạy SetTime/SetDate ở lần đầu.
   Chọn một backup register được HAL của dự án hỗ trợ và kiểm tra vùng bảo mật. */
if (HAL_RTCEx_BKUPRead(&hrtc, RTC_BKP_DR0) != 0x533CE001u) {
    RTC_TimeTypeDef t = {.Hours=0, .Minutes=0, .Seconds=0,
                         .TimeFormat=RTC_HOURFORMAT12_AM};
    RTC_DateTypeDef d = {.WeekDay=RTC_WEEKDAY_MONDAY, .Month=RTC_MONTH_JANUARY,
                         .Date=1, .Year=0};
    if (HAL_RTC_SetTime(&hrtc, &t, RTC_FORMAT_BIN) != HAL_OK) Error_Handler();
    if (HAL_RTC_SetDate(&hrtc, &d, RTC_FORMAT_BIN) != HAL_OK) Error_Handler();
    HAL_RTCEx_BKUPWrite(&hrtc, RTC_BKP_DR0, 0x533CE001u);
}
```

Giờ 00:00, ngày 01/01/2000 chỉ là **giá trị báo chưa đồng bộ**, không phải giờ thực. Marker đơn lẻ không đủ cho dữ liệu quan trọng; thêm phiên bản cấu trúc và CRC. Khi đọc calendar bằng HAL, đọc **time rồi date** để giải phóng shadow lock và nhận cặp dữ liệu nhất quán; xem RM0481 §51.3.12 tr. 2218.

```c
RTC_TimeTypeDef now_t;
RTC_DateTypeDef now_d;
if (HAL_RTC_GetTime(&hrtc, &now_t, RTC_FORMAT_BIN) != HAL_OK) Error_Handler();
if (HAL_RTC_GetDate(&hrtc, &now_d, RTC_FORMAT_BIN) != HAL_OK) Error_Handler();
```

RTC có prescaler bất đồng bộ/đồng bộ; với LSE 32768 Hz cấu hình thường gặp là `(PREDIV_A+1)×(PREDIV_S+1)=32768`, ví dụ 128×256. Giá trị cụ thể phụ thuộc yêu cầu độ phân giải subsecond và năng lượng. RM0481 §51.3.6–51.3.11 tr. 2212–2217.

## 4. Alarm, wakeup và Stop/Standby

RTC alarm dành cho mốc lịch; RTC wakeup timer dành cho nhịp định kỳ. LPTIM dành cho định thời linh hoạt, PWM/capture và đánh thức khi Stop. Trong **Stop**, core dừng nhưng một số clock thấp và retention còn; trong **Standby**, phần lớn ngữ cảnh chạy mất và thức dậy giống reset. Trước khi chọn mode, lập bảng: dữ liệu nào phải giữ, nguồn wakeup nào còn chạy, thời gian khởi động và dòng mục tiêu. RM0481 §10.8, bảng 107–108 tr. 445–446, §10.8.5–10.8.6 tr. 451–456; RTC §51.3.9–51.3.10 tr. 2214.

Ví dụ HAL **định hướng** cho chu kỳ wakeup 1 giây khi RTC đã tạo nguồn 1 Hz `ck_spre`:

```c
/* Gọi sau MX_RTC_Init(); bật đúng RTC wakeup IRQ/EXTI qua CubeMX. */
if (HAL_RTCEx_SetWakeUpTimer_IT(&hrtc, 0u,
        RTC_WAKEUPCLOCK_CK_SPRE_16BITS) != HAL_OK) Error_Handler();

/* Chuẩn bị ngoại vi trước Stop; kiểm tra cờ và IRQ pending theo project. */
HAL_SuspendTick();
HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI);
/* Sau wakeup: tái cấu hình SystemClock/PLL theo mode và clock tree dự án. */
SystemClock_Config();
HAL_ResumeTick();

void HAL_RTCEx_WakeUpTimerEventCallback(RTC_HandleTypeDef *h)
{
    if (h->Instance == RTC) wakeup_pending = 1u; /* volatile flag */
}
```

Tên mode/API Stop khác giữa biến thể HAL H5; đoạn trên cần đối chiếu header dự án trước khi biên dịch. `WUTR=0` cùng `CK_SPRE` thường ứng với một chu kỳ 1 Hz, nhưng hãy xác nhận cấu hình `WUCKSEL`, công thức trong RM0481 §51.3.10 và đo trên board. `HAL_SuspendTick()` ngăn SysTick đánh thức liên tục. Đừng xóa cờ wakeup tùy tiện sau khi vào Stop. Với TrustZone, security attribution của RTC/PWR/EXTI phải cho phép code đang chạy cấu hình và nhận IRQ.

## 5. LPTIM1/2: timer trong chế độ nguồn thấp

LPTIM có 16 bit ARR, prescaler 1–128, 2 kênh PWM/capture và chọn kernel clock từ LSE, LSI, HSI hoặc APB. Để **tiếp tục chạy trong Stop**, chọn LSE/LSI hoặc external clock theo khả năng thực tế; APB/HSI không mặc nhiên sống trong Stop. DS14539 §3.33.4 tr. 47; RM0481 §48.3–48.5 tr. 2127–2150. LPTIM1/2 không phải TIM thường thu nhỏ: vùng clock APB và kernel tách nhau, ghi ARR/CCR cần chờ cờ `ARROK`/`CMPOK`, và filter đầu vào cần clock nội (RM0481 §48.4.4–48.4.5 tr. 2133–2134; §48.7 tr. 2151+).

Ví dụ ý tưởng dùng LPTIM1 với LSE 32768 Hz và prescaler 32 ⇒ counter khoảng 1024 tick/s. Chọn ARR theo mô tả continuous mode của RM, bắt đầu bằng HAL và **đo chu kỳ thực**, sau đó hiệu chỉnh 1 tick nếu cần. Không dùng công thức `ARR+1` của TIMx để áp bừa cho mọi mode LPTIM. Để đánh thức Stop, bật đúng IRQ/EXTI của LPTIM, cấu hình low-power mode, và bảo đảm LSE thực sự chạy. Đọc `RCC_BDCR.LSERDY` và kiểm tra scope/current meter khi debug. PWM LPTIM chỉ đáng dùng khi chân AF của đúng package được đưa ra ngoài; xem DS14539 §4 pinout/alternate function.

## 6. IWDG: bảo vệ khi phần mềm không còn tiến triển

IWDG dùng LSI độc lập; thời gian xấp xỉ `T=(RLR+1)×prescaler/f_LSI`. Vì `f_LSI` sai lệch, hãy tính **min/max timeout** từ giới hạn điện của datasheet và đo/hiệu chuẩn LSI nếu cần. Sau khi start, IWDG không thể dừng bằng phần mềm trong vận hành thường; debug freeze có thể làm hành vi khác chạy thật. Option bytes có thể bật watchdog phần cứng từ lúc boot. RM0481 §49.3–49.7 tr. 2178–2193; DS14539 §3.33.5 tr. 47.

```c
/* MX_IWDG_Init() cấu hình prescaler/reload bằng CubeMX.
   Chỉ refresh khi tất cả task quan trọng đã báo tiến triển. */
static bool sensors_ok, comms_ok, control_ok;

void Safety_Supervisor_100ms(void)
{
    if (sensors_ok && comms_ok && control_ok) {
        if (HAL_IWDG_Refresh(&hiwdg) != HAL_OK) Error_Handler();
        sensors_ok = comms_ok = control_ok = false;
    }
}
```

Không refresh vô điều kiện trong timer ISR, vì CPU vẫn có thể phục vụ ISR khi main loop hoặc task chính đã treo. Với hệ nhiều task, mỗi task cập nhật heartbeat/sequence sau khi hoàn tất công việc thực sự; supervisor kiểm tra mọi heartbeat trước refresh. Nếu dùng **window option** của IWDG, refresh quá sớm cũng lỗi; xem RM0481 §49.4.4 tr. 2181.

## 7. WWDG: phát hiện refresh lệch nhịp

WWDG dùng clock từ miền APB, bộ đếm 7 bit và cửa sổ `W`. Refresh khi counter còn **lớn hơn cửa sổ** là quá sớm; để counter xuống ngưỡng reset là quá muộn. Bởi vậy chọn window dựa trên **thời gian chạy tệ nhất** của vòng giám sát và clock PCLK thật, kể cả lúc thay clock/Stop. RM0481 §50.4 tr. 2195–2197. WWDG thích hợp giám sát lỗi lịch trình khi hệ đang chạy; IWDG mạnh hơn cho mất clock/chết sâu vì dùng LSI độc lập. Có thể dùng cả hai nếu chứng minh được trình tự khởi động và low-power không gây reset sai.

```c
/* Chỉ gọi tại điểm hợp lệ của vòng điều khiển; hiwwdg đã được CubeMX init. */
if (all_deadlines_met && sequence_is_valid) {
    if (HAL_WWDG_Refresh(&hwwdg) != HAL_OK) Error_Handler();
}
/* Callback EWI chỉ nên lưu trạng thái rất ngắn; thời gian trước reset hữu hạn. */
```

WWDG early wakeup interrupt (EWI) **không thay thế** thiết kế lưu log bền vững: quãng thời gian còn lại có thể quá ngắn cho ghi Flash. Dùng backup register/RAM có retention, hoặc log nguyên nhân reset ngay lần boot sau. RM0481 §50.5 tr. 2198.

## 8. Reset cause, khôi phục và kỹ thuật kiểm thử

Ngay đầu boot, đọc cờ reset trong RCC trước khi xóa; phân biệt IWDG, WWDG, BOR/POR, software reset và wake từ Standby bằng trạng thái PWR phù hợp. Sau đó ghi một bản ghi sự kiện gồm `reset_flags`, boot counter, firmware version và CRC; xóa cờ **sau khi đã chụp**. RM0481 §11 (RCC reset flags) và §10.8.6 tr. 453. Sau watchdog reset, không tự động bật lại tải nguy hiểm: đưa output về trạng thái an toàn, xác thực cảm biến và nguyên nhân lỗi trước khi chạy lại.

| Bài tập | Tiêu chí hoàn thành |
|---|---|
| RTC backup | Cấp nguồn, đặt giờ, reset MCU, tắt VDD nhưng giữ VBAT; ghi rõ điều kiện nào giữ giờ. |
| RTC wakeup | Stop 5–10 lần, đo dòng và khoảng thức dậy; so với nguồn LSE/LSI đã chọn. |
| LPTIM | LPTIM1 đánh thức Stop, đổi prescaler và đo sai số thời gian. |
| IWDG | Làm một task ngừng heartbeat; chứng minh supervisor không refresh và log đúng reset cause. |
| WWDG | Thử refresh quá sớm và quá muộn trong bài test riêng; chứng minh cửa sổ hoạt động. |
| Tích hợp | Sau watchdog reset, mọi chân điều khiển tải ở trạng thái an toàn cho tới khi hoàn tất kiểm tra. |

**Checklist senior:** xác nhận nguồn clock từng khối; tính timeout bằng biên sai số oscillator; giữ ISR ngắn; bảo vệ backup data bằng version/CRC; thử trên phần cứng ở nhiệt độ/điện áp biên nếu sản phẩm yêu cầu; đo dòng với debugger ngắt kết nối vì debug có thể giữ clock hoặc freeze watchdog; ghi rõ hành vi của từng mode Sleep/Stop/Standby theo RM0481 §10.8, không giả định mọi RAM/ngoại vi đều còn trạng thái.

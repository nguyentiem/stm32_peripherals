# GPIO, chọn chức năng chân và ngắt EXTI trên STM32H533CE

## Kết quả cần đạt

Sau chương này, bạn có thể đọc sơ đồ chân của đúng gói 48 chân, cấu hình GPIO bằng thanh ghi, chọn alternate function (AF), và theo dõi một cạnh tín hiệu từ chân qua EXTI tới NVIC. Bài kiểm tra cuối là nút PA0 điều khiển LED ngoài ở PA5 bằng ngắt, có xử lý dội phím ở tầng ứng dụng.

**Nguồn gốc:** [DS14539 Rev 3](../doc/DS_stm32h533ce%20%281%29.pdf), §4, tr. 61 trở đi; [RM0481 Rev 5](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf), §13 GPIO (tr. 615 trở đi), §18 EXTI (tr. 775 trở đi); [PM0264](../doc/pm0264-stm32-cortexm33-mcus-and-mpus-programming-manual-stmicroelectronics%20%281%29.pdf), mục NVIC và exception. Số trang ở đây là số **in trên tài liệu**.

## 1. Từ pad vật lý đến GPIO

**Định nghĩa và nguyên lý.** GPIO là mạch ghép giữa pad vật lý với bộ đọc mức logic, bộ lái đầu ra và mux chức năng ngoại vi. CPU cấu hình các mạch này qua thanh ghi; điện áp trên pad còn do nguồn, tải, điện trở và thiết bị bên ngoài quyết định. Lợi ích là cùng một chân có thể đọc nút, lái LED hoặc mang tín hiệu ngoại vi mà không cần logic rời. Giới hạn là một pad chỉ có một chức năng được chọn tại một thời điểm và không được vượt giới hạn điện áp/dòng của datasheet. [RM0481 §13.3–13.5; DS14539 §4–5](../doc/DS_stm32h533ce%20%281%29.pdf).

| Mode `MODER` | Vì sao tồn tại, mạch bên trong | Dùng khi nào; giới hạn cần nhớ |
|---|---|---|
| Input | Tắt bộ lái, đưa điện áp pad vào bộ đọc `IDR`; CPU/EXTI quan sát mức hoặc cạnh. | Nút, tín hiệu báo lỗi, dữ liệu vào. Nếu nguồn tín hiệu trở kháng cao hoặc để hở, mức có thể trôi và phát ngắt giả. |
| Output | Nối latch `ODR`/`BSRR` với bộ lái pad; CPU chủ động tạo mức. | LED, chip-select, reset cho IC ngoài. Kiểm tra dòng tải và mức an toàn ngay sau reset; đừng nối trực tiếp hai output đối kháng. |
| Alternate function | Mux cho ngoại vi điều khiển/đọc pad; cấu hình `AFR` chọn đúng chức năng. | UART/SPI/I²C/PWM khi timing do phần cứng quyết định. Chọn AF sai vẫn có thể thấy thanh ghi ngoại vi chạy nhưng pad im lặng. |
| Analog | Ngắt đường số không cần thiết, đưa pad về đường analog hoặc trạng thái giảm rò tùy chân. | ADC/DAC và chân không dùng theo khuyến nghị RM. Không áp dụng pull số để làm nguồn bias chính xác cho ADC; kiểm tra chức năng analog thật trên đúng pin. |

**Pull là định thiên đầu vào, không phải kiểu lái đầu ra.** `PUPDR` nối một điện trở yếu tới VDD (pull-up), GND (pull-down) hoặc không nối (no-pull). Nút kéo xuống GND cần pull-up để lúc thả đọc 1; nút kéo lên VDD cần pull-down để lúc thả đọc 0. No-pull hợp lý khi tín hiệu luôn được lái bởi mạch ngoài hoặc đã có điện trở ngoài. Chân để hở không có pull sẽ *floating*: nhiễu điện dung có thể làm `IDR` đổi ngẫu nhiên, tăng tiêu thụ và gây EXTI giả. Pull nội chỉ phù hợp bias nhẹ; với cáp dài, rise-time, dòng rò, mức fail-safe hoặc yêu cầu an toàn, tính điện trở ngoài theo giới hạn điện của hai phía. [RM0481 §13.3–13.4; DS14539 §5](../doc/DS_stm32h533ce%20%281%29.pdf).

| Kiểu output `OTYPER` | Dòng điện đi như thế nào | Bài toán giải quyết và ví dụ | Đánh đổi/lỗi thường gặp |
|---|---|---|---|
| Push-pull | Hai transistor chủ động nối pad lên nguồn I/O hoặc xuống GND; dòng cấp/tải qua điện trở tải. | Cạnh nhanh, lái LED hoặc SPI/USART TX trên một dây có một chủ. | Không được đấu nhiều đầu ra push-pull cùng dây: khi một bên kéo cao và bên kia kéo thấp sẽ có dòng đối kháng. |
| Open-drain | Transistor chỉ kéo thấp; khi ghi 1, transistor nhả chân, điện trở pull-up kéo đường dây lên cao. | I²C, interrupt chia sẻ kiểu active-low, wired-AND và đôi khi level shifting có kiểm soát. Nhiều thiết bị có thể kéo thấp mà không đối kháng ở mức cao. | Không có pull-up thì mức cao trôi; pull-up quá lớn làm cạnh lên chậm, quá nhỏ làm tăng dòng khi kéo thấp. Chỉ dùng điện áp pull-up tương thích pin và thiết bị. |

`OSPEEDR` chọn sức lái/tốc độ cạnh, **không** tăng tần số CPU. Chọn mức thấp nhất còn đáp ứng rise/fall time: cạnh quá nhanh tăng EMI và ringing, quá chậm vi phạm timing giao tiếp. Đo bằng oscilloscope ngay trên tải. Sau reset hoặc khi chuyển mode, kiểm tra trạng thái mặc định của đúng chân trong RM và trạng thái do mạch ngoài áp đặt; với chip-select active-low, điện trở kéo lên ngoài giúp giữ thiết bị không được chọn trước khi firmware chạy. [RM0481 §13.3–13.5; DS14539 §5](../doc/DS_stm32h533ce%20%281%29.pdf).

Một chân MCU không mặc định là GPIO dùng được. Cần kiểm tra theo thứ tự:

1. **Part và package:** H533CE chỉ có 48 chân. Bảng AF chung của họ H533 có những chân không được bond ra ngoài package này. Xem hình LQFP48 hoặc UFQFPN48 trong DS14539 §4.1, tr. 61–62, rồi đối chiếu bảng mô tả chân và bảng AF §4.2–4.3.
2. **Mạch board:** kiểm tra nút/LED nối vào chân nào, mức tác động, điện trở, nguồn điện, chân bị ST-LINK hay ngoại vi khác chiếm.
3. **Quyền sở hữu chân:** mỗi pad chỉ có một chế độ chính tại một thời điểm: input, output, AF hoặc analog. Đổi cấu hình lúc ngoại vi đang truyền có thể gây xung giả.
4. **Clock port:** bật clock GPIO trong RCC trước khi ghi cấu hình. GPIOA dùng `RCC->AHB2ENR`.
5. **Miền bảo mật:** nếu bật TrustZone, GPIO/EXTI và IRQ có thể được gán secure; mã nonsecure không tự nhiên có quyền cấu hình mọi thứ.

`MODER` có hai bit mỗi chân: `00` input, `01` output, `10` AF, `11` analog. `OTYPER` chọn push-pull/open-drain; `OSPEEDR` đặt tốc độ cạnh, không phải tần số bus; `PUPDR` chọn no pull/pull-up/pull-down. `IDR` đọc mức thực tại đầu vào, `ODR` là latch đầu ra. `BSRR` ghi 1 vào nửa thấp để set, nửa cao để reset, giúp cập nhật một chân bằng **một phép ghi nguyên tử**; tránh `ODR ^= bit` nếu ISR và main cùng thao tác port. Xem RM0481 §13.3–13.5.

### Alternate function (AF)

Đặt `MODER=10`, ghi mã AF 4 bit/chân vào `AFR[0]` cho chân 0–7 hoặc `AFR[1]` cho chân 8–15. Chỉ lấy mã AF từ **dòng đúng chân** trong bảng DS14539 §4.3; không suy ra AF từ tên ngoại vi. Sau đó xét `OTYPER`, pull và speed theo electrical/timing của giao tiếp. Ví dụ I²C cần đường open-drain và pull-up phù hợp; USART TX thường push-pull. Cấu hình AF là nối mux pad với ngoại vi, chưa bật clock hoặc khởi tạo chính ngoại vi.

## 2. Từ cạnh chân đến hàm ngắt

```text
PA0 ──> EXTI mux line 0 ──> chọn cạnh FTSR1/RTSR1 ──> FPR1/RPR1
                                  │                         │
                                  └── IMR1 mask ────────────> NVIC EXTI0_IRQn
                                                              └── EXTI0_IRQHandler()
```

EXTI line 0–15 là **số chân**, không phải số port: PA0/PB0/PC0 cạnh tranh cùng line 0; `EXTICR[0]` chọn đúng một port cho line 0. `RTSR1` chọn cạnh lên, `FTSR1` chọn cạnh xuống, `IMR1` cho phép IRQ. `RPR1` và `FPR1` là cờ pending riêng; xóa bằng cách **ghi 1** vào bit tương ứng, không dùng `&= ~bit`. `EMR1` dành cho event; event và interrupt là hai đường khác nhau. EXTI xác định nguyên nhân và tạo request; NVIC mới quyết định IRQ được CPU phục vụ. Xem RM0481 §18.4–18.6 và PM0264 phần exceptions/NVIC.

Nếu dùng các line khác, tra bảng vector để lấy IRQ đúng; một số line dùng IRQ nhóm và handler phải kiểm tra từng cờ. Đặt mức ưu tiên theo hệ thống, không dùng `NVIC_SetPriority(..., 0)` cho tất cả. Với Cortex-M33, ngắt chỉ chạy sau khi cấu hình vector table, IRQ enable và global interrupt enable đúng miền bảo mật.

## 3. Ví dụ thanh ghi: nút PA0, LED ngoài PA5

**Phần cứng giả định:** nút nối PA0 xuống GND khi nhấn; pull-up nội được bật. LED nối PA5 qua điện trở phù hợp xuống GND, mức cao bật LED. Cả PA0 và PA5 có mặt trên package 48 chân theo DS14539 §4.1, tr. 61–62. Nếu board dùng sơ đồ khác, đổi port/chân và cực tính; đừng giả định LED tích hợp nằm ở PA5.

```c
/* Build với CMSIS device header cho đúng STM32H533xx.
 * Startup file phải ánh xạ EXTI0_IRQHandler vào vector table.
 */
#include "stm32h533xx.h"
#include <stdint.h>

#define BUTTON_PIN 0u
#define LED_PIN    5u
#define BUTTON_BIT (1u << BUTTON_PIN)
#define LED_BIT    (1u << LED_PIN)

static volatile uint32_t button_events;

static void gpio_exti_init(void)
{
    RCC->AHB2ENR |= RCC_AHB2ENR_GPIOAEN;
    (void)RCC->AHB2ENR; /* bảo đảm ghi enable clock đã tới peripheral */

    /* PA5: output push-pull; đặt LED tắt trước khi đổi mode. */
    GPIOA->BSRR = LED_BIT << 16u;
    GPIOA->OTYPER &= ~LED_BIT;
    GPIOA->OSPEEDR &= ~(3u << (2u * LED_PIN)); /* low speed */
    GPIOA->PUPDR &= ~(3u << (2u * LED_PIN));
    GPIOA->MODER = (GPIOA->MODER & ~(3u << (2u * LED_PIN)))
                 |  (1u << (2u * LED_PIN));

    /* PA0: input, pull-up. */
    GPIOA->MODER &= ~(3u << (2u * BUTTON_PIN));
    GPIOA->PUPDR = (GPIOA->PUPDR & ~(3u << (2u * BUTTON_PIN)))
                 | (1u << (2u * BUTTON_PIN));

    /* EXTI0 lấy từ port A (mã 0 ở EXTICR[0] bits [3:0]). */
    EXTI->IMR1 &= ~BUTTON_BIT;
    EXTI->EXTICR[0] &= ~0xFu;
    EXTI->RTSR1 &= ~BUTTON_BIT;
    EXTI->FTSR1 |= BUTTON_BIT;
    EXTI->RPR1 = BUTTON_BIT; /* write-one-to-clear */
    EXTI->FPR1 = BUTTON_BIT;
    NVIC_ClearPendingIRQ(EXTI0_IRQn);
    NVIC_SetPriority(EXTI0_IRQn, 5u); /* chọn lại theo ưu tiên toàn hệ thống */
    EXTI->IMR1 |= BUTTON_BIT;
    NVIC_EnableIRQ(EXTI0_IRQn);
}

void EXTI0_IRQHandler(void)
{
    if ((EXTI->FPR1 & BUTTON_BIT) != 0u) {
        EXTI->FPR1 = BUTTON_BIT; /* xóa nguồn ngắt trước khi rời ISR */
        ++button_events;
    }
}

int main(void)
{
    gpio_exti_init();
    uint32_t handled = 0u;
    for (;;) {
        /* Mỗi cạnh có thể là dội phím. Đây là bộ đếm cạnh, chưa phải bộ đếm lần nhấn. */
        if (handled != button_events) {
            handled = button_events;
            GPIOA->BSRR = LED_BIT; /* LED on; tắt bằng LED_BIT << 16u */
        }
    }
}
```

Đoạn mã trên minh họa đường ngắt, **chưa có debounce**. Với sản phẩm thật, ISR chỉ ghi nhận timestamp hoặc cờ; dùng timer 10–30 ms kiểm tra trạng thái ổn định rồi mới sinh sự kiện “nhấn”. Biến `volatile` bảo đảm compiler đọc lại; nó không giải quyết tính nguyên tử cho dữ liệu phức hợp hay đồng bộ giữa nhiều luồng. Nếu main cần lấy và xóa bộ đếm, dùng vùng critical ngắn hoặc atomic thích hợp.

## 4. Ghi chú kỹ thuật kiểu senior

- **Chọn chân bằng bảng giao nhau:** pinout package ∩ AF của chân ∩ ràng buộc board ∩ điện áp/timing. Bảng peripheral tổng quát không chứng minh một chân cụ thể dùng được.
- **An toàn khởi động:** set latch output bằng `BSRR` trước khi đặt `MODER=output`, nhất là enable nguồn, chip-select, relay. Nếu tải có tác dụng ở mức thấp, đảo thứ tự giá trị theo sơ đồ.
- **Không dùng high speed mặc định:** tốc độ cạnh cao tăng EMI và ringing. Chỉ nâng `OSPEEDR` khi timing/đo tín hiệu đòi hỏi.
- **Open-drain cần đường lên:** pull-up nội thường quá yếu hoặc không đáp ứng rise time cho bus; tính điện trở ngoài theo điện dung và thông số thiết bị.
- **Nguồn ngắt lặp:** clearing NVIC pending mà chưa clear `EXTI->FPR1/RPR1` khiến IRQ trở lại. Cũng kiểm tra cả cạnh nếu đã bật hai cạnh.
- **Dội phím và nhiễu:** EXTI báo cạnh điện, không hiểu “một lần nhấn”. Dùng RC/Schmitt nếu tín hiệu ngoài nhiễu, timer debounce và tránh xử lý chậm trong ISR.
- **Debug pins:** PA13/PA14 thường dành cho SWD; đổi AF quá sớm có thể mất kết nối debug. BOOT0, NRST, nguồn, VCAP không phải GPIO thông thường.
- **Sleep/Stop:** event, interrupt và wake-up khác nhau; kiểm tra loại low-power mode, clock và khả năng wake-up theo RM trước khi dựa vào EXTI.

## 5. Bài tập và tiêu chí kiểm tra

1. **GPIO output:** tạo xung PA5 bằng `BSRR`; đo mức cao/thấp và thời gian cạnh bằng logic analyzer. So sánh với thao tác read-modify-write `ODR` khi có ISR ghi cùng port.
2. **Input:** đọc `IDR` PA0 với pull-up, sau đó tháo nút để xác nhận mức mặc định. Vẽ sơ đồ dòng qua pull-up khi nhấn.
3. **EXTI:** đếm cạnh xuống; dùng breakpoint ở handler và kiểm tra `FPR1`, `IMR1`, `EXTICR[0]`, NVIC pending. Nhấn một lần và ghi nhận nhiều cạnh do bounce.
4. **Debounce:** thêm timer định kỳ 1 ms và bộ lọc 20 ms. Tiêu chí: mỗi lần nhấn vật lý tăng đúng một sự kiện, nhấn giữ không lặp, nhả không tăng.
5. **AF:** chọn một UART/SPI/I²C thực tế trên board. Chứng minh chân tồn tại trên gói 48 chân, ghi mã AF của từng chân theo DS và giải thích điện trở/drive/speed.

**Checklist trước khi chuyển chương:** giải thích được `MODER`, `PUPDR`, `AFR`, `BSRR`; chỉ đúng đường EXTI mux → trigger → pending → mask → NVIC; biết vì sao phải xóa pending bằng ghi 1; và chứng minh được pin đã chọn có trên **H533CE 48 chân**.

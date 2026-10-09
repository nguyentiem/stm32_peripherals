# 04 — USART, UART và LPUART trên STM32H533CE

## Mục tiêu

Sau bài này bạn phải tự cấu hình được truyền nhận bất đồng bộ, giải thích baud rate và sai số clock, chọn polling/ngắt/DMA đúng tình huống, xử lý lỗi đường truyền, và thiết kế bộ đệm nhận không mất dữ liệu.

## Phần cứng của H533CE

Theo **DS14539 Rev 3, bảng 2 trang 15**, H533CE có **3 USART, 2 UART và 1 LPUART**. Datasheet mô tả USART1/2/3/6 cho cả họ chip; vì CE chỉ có ba, hãy kiểm tra instance và chân trong CubeMX/device header trước khi dùng. Ví dụ dưới dùng **USART2: PA2 = TX, PA3 = RX, AF7**, được bảng AF trang 82 xác nhận cho gói 48 chân (sơ đồ trang 61–62). USART hỗ trợ synchronous clock; UART4/5 chỉ bất đồng bộ; LPUART1 phục vụ ứng dụng tiêu thụ điện thấp. Xem **RM0481 Rev 5 §56 trang 2596, §57 trang 2686**.

## Kiến thức phải học

1. **Khung dữ liệu:** idle mức 1, start mức 0, 7/8/9 data bit, tùy chọn parity, 1/2 stop bit. Hai đầu phải thống nhất baud, parity và stop bit.
2. **Clock:** APB cấp bus truy cập thanh ghi; kernel clock tạo nhịp UART. Sau mỗi thay đổi clock phải tính/cấu hình lại baud. Kiểm tra oversampling 8/16 và sai số thực tế của cả hai phía.
3. **Luồng TX/RX:** `TXE/TXFNF` cho phép nạp dữ liệu, `TC` xác nhận bit cuối đã ra chân; `RXNE/RXFNE` báo dữ liệu đến. Đừng tắt transceiver RS-485 chỉ vì bộ đệm TX đã rỗng: chờ `TC`.
4. **Ba phương thức truyền:** polling cho thông điệp ngắn và khởi động; ngắt cho dữ liệu thưa; DMA cho luồng liên tục. FIFO giảm số lần ngắt nhưng không thay thế bộ đệm phần mềm.
5. **Lỗi:** framing, parity, noise, overrun. Overrun thường là dấu hiệu ISR hoặc tác vụ tiêu thụ dữ liệu quá chậm; xóa cờ theo trình tự RM/HAL và ghi lại bộ đếm lỗi.
6. **Dữ liệu thực tế:** UART là luồng byte, không có ranh giới gói. Định nghĩa delimiter, length hoặc framing riêng; có CRC/checksum nếu dữ liệu quan trọng. IDLE-line hữu ích khi nhận gói có khoảng nghỉ.
7. **LPUART:** có thể hoạt động với clock thấp và đánh thức khỏi Stop; phải kiểm tra giới hạn baud ứng với clock chọn (DS §3.37.2 trang 54–55).

## Giải thích giao thức từ chân TX đến bản tin ứng dụng

**Định nghĩa và ích lợi.** UART là bộ truyền nhận nối tiếp bất đồng bộ: hai đầu thống nhất tốc độ bit trước khi trao đổi, không cần dây clock. Nó hợp cho console, GPS, modem và cảm biến tốc độ vừa. USART của H533CE còn có chế độ đồng bộ dùng chân clock; chọn chế độ này khi thiết bị đối tác yêu cầu clock và phần cứng có đủ chân. LPUART ưu tiên hoạt động ở chế độ tiết kiệm điện; phải kiểm tra clock và baud được phép. UART TTL/CMOS trên chân MCU chỉ dùng trong board hoặc qua đầu nối phù hợp: RS-232 cần bộ chuyển mức, RS-485 cần transceiver và điều khiển chiều truyền.

```text
TX idle=1 ──┐ START=0 │ d0 d1 d2 d3 d4 d5 d6 d7 │ PARITY? │ STOP=1 │ idle=1
            └──────────┘  LSB phát trước; ví dụ 8N1 dài 10 bit/byte
```

Start tạo mốc thời gian, bộ nhận lấy mẫu gần tâm mỗi bit bằng clock nội và kiểm tra stop. Sai số baud tích lũy trong một ký tự; sai clock, nhiễu hoặc méo cạnh gây lỗi framing/parity/noise. Byte kế tiếp bắt đầu bằng start mới nên thường tự đồng bộ ở mức *ký tự*, nhưng UART không biết ranh giới *bản tin*. Parity chỉ phát hiện một số lỗi bit; không xác nhận dữ liệu đã đến hay đã được xử lý. 8N1 tại 115200 bit/s đạt tối đa khoảng 11520 byte/s trước overhead ứng dụng.

| Cấu hình | Vì sao có, khi chọn | Đánh đổi/lỗi cần đo |
|---|---|---|
| 8N1 | Định dạng phổ biến cho console, module số; 8 data, không parity, 1 stop | Không có phát hiện lỗi bit tích hợp. |
| Parity even/odd | Phát hiện lỗi lẻ bit khi đối tác quy định | Tốn vị trí bit/thời gian; không thay CRC gói; phải kiểm tra cách HAL biểu diễn word length kèm parity. |
| 2 stop | Tăng thời gian phục hồi cho thiết bị nhận chậm/chuẩn yêu cầu | Giảm thông lượng; không chữa sai baud lớn. |
| RTS/CTS | Flow control phần cứng khi đầu nhận có thể đầy buffer | Cần thêm dây và đối tác hỗ trợ; không bảo vệ chống mất điện. |
| Half duplex/RS-485 | Nhiều node trên cặp dây chia sẻ, đi xa hơn qua truyền vi sai | Cần transceiver, turnaround sau `TC`, giao thức phân quyền bus. |
| 16×/8× oversampling | 16× cho biên lấy mẫu tốt; 8× khi cần baud cao hơn | 8× giảm dung sai nhiễu/sai lệch clock. |

**Một gói ứng dụng minh họa, không thuộc UART:** `[SOF=0xA5][LEN:1][SEQ:1][TYPE:1][PAYLOAD:LEN][CRC16:2]`. Quy định rõ CRC tính từ LEN đến cuối PAYLOAD, thứ tự byte CRC, LEN tối đa và hành vi khi LEN sai. Parser chỉ công nhận gói nếu đủ byte trước timeout và CRC đúng; khi sai, quét tìm SOF kế tiếp. Nếu payload có thể chứa `0xA5`, dùng length + CRC và quét từng vị trí, hoặc dùng byte stuffing/COBS để ranh giới chắc chắn hơn. Gói lệnh quan trọng cần ACK **ứng dụng** chứa SEQ và trạng thái xử lý; sender timeout rồi gửi lại có giới hạn, receiver lưu SEQ đã xử lý để không thực hiện lệnh hai lần. ACK UART không tồn tại; ACK ứng dụng báo trạng thái theo định nghĩa của bạn, không chứng minh tác động vật lý hoàn tất nếu chưa đo/ghi nhận.

**Chống nhiễu và mất bản tin.** Dây ngắn: chung GND, đầu thu có idle ổn định, tránh đi cạnh đường công suất. Cáp dài hoặc môi trường công nghiệp: transceiver RS-485 đúng điện áp, termination/bias theo topology, chống ESD và kiểm tra common-mode. CRC phát hiện gói hỏng; timeout, SEQ và retry phục hồi mất gói; FIFO/DMA/ring buffer chống overrun do CPU chậm. Đây là ba vấn đề khác nhau. Đo TX/RX với `0x55`, kiểm baud, idle, thời gian turnaround và bộ đếm ORE/FE/NE; tăng tải cho đến khi xác định điểm mất dữ liệu.

**Bài kiểm chứng:** gửi gói hợp lệ, gói sai CRC, gói thiếu byte cuối và hai bản sao cùng SEQ. Ghi chính xác phản ứng parser, timeout, ACK và số lần lệnh được thực hiện.

## Trình tự cấu hình

`RCC clock` → `GPIO AF + pull/speed` → `chọn kernel clock` → `word length/parity/stop/oversampling/baud` → `enable TX/RX` → `bật ngắt/DMA nếu dùng` → `xóa cờ tồn dư` → `truyền thử bằng loopback hoặc USB-UART`.

## Ví dụ HAL: USART2 trên PA2/PA3, 115200 8N1

Ví dụ dành cho dự án STM32CubeH5 đã có `HAL_Init()`, `SystemClock_Config()` và file startup cho đúng chip. Kiểm tra xung UART từ cấu hình clock trước khi dùng.

```c
#include "stm32h5xx_hal.h"

static UART_HandleTypeDef huart2;
extern void Error_Handler(void);

void usart2_init(void)
{
    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_USART2_CLK_ENABLE();

    GPIO_InitTypeDef pin = {0};
    pin.Pin = GPIO_PIN_2 | GPIO_PIN_3;
    pin.Mode = GPIO_MODE_AF_PP;
    pin.Pull = GPIO_PULLUP;
    pin.Speed = GPIO_SPEED_FREQ_LOW;
    pin.Alternate = GPIO_AF7_USART2;
    HAL_GPIO_Init(GPIOA, &pin);

    huart2.Instance = USART2;
    huart2.Init.BaudRate = 115200;
    huart2.Init.WordLength = UART_WORDLENGTH_8B;
    huart2.Init.StopBits = UART_STOPBITS_1;
    huart2.Init.Parity = UART_PARITY_NONE;
    huart2.Init.Mode = UART_MODE_TX_RX;
    huart2.Init.HwFlowCtl = UART_HWCONTROL_NONE;
    huart2.Init.OverSampling = UART_OVERSAMPLING_16;
    if (HAL_UART_Init(&huart2) != HAL_OK) Error_Handler();
}

void usart2_hello(void)
{
    static const uint8_t msg[] = "STM32H533CE ready\r\n";
    if (HAL_UART_Transmit(&huart2, (uint8_t *)msg, sizeof(msg) - 1, 100) != HAL_OK)
        Error_Handler();
}
```

**Đi dây:** PA2 → RX adapter, PA3 ← TX adapter, nối GND chung; adapter phải ở mức logic 3,3 V. Đừng nối trực tiếp RS-232 ±V vào chân MCU.

## Ghi chú kiểu senior

- Luôn đặt timeout hữu hạn cho polling. Khi dữ liệu đến liên tục, dùng DMA vòng hoặc nhận đến IDLE và một ring buffer; ISR chỉ chuyển dữ liệu, không parse gói dài.
- Nếu muốn log từ nhiều tác vụ, gom qua một hàng đợi để tránh các bản tin bị xen byte. Không gọi hàm transmit blocking trong ISR ưu tiên cao.
- Kiểm tra pin conflict với SWD, USB, timer hoặc các AF khác trước khi cố định sơ đồ mạch.
- Test bằng logic analyzer: đo một bit để kiểm tra baud thật; gửi mẫu `0x55` để nhìn bit đều nhau; gây lỗi parity/baud có chủ đích và quan sát cờ lỗi.

## Bài tập đạt chuẩn

1. Echo byte nhận được và in bộ đếm framing/overrun mỗi 10 giây.
2. Viết parser gói `[SOF][LEN][PAYLOAD][CRC]` có timeout và khả năng đồng bộ lại sau byte hỏng.
3. Thay polling bằng DMA + IDLE; chứng minh không mất byte khi nhận liên tục trong lúc CPU làm việc khác.

## Tài liệu gốc

- [DS14539 Rev 3 — bảng 2, sơ đồ chân, bảng AF, §3.37](../doc/DS_stm32h533ce%20%281%29.pdf), trang 15, 52–55, 61–62, 82.
- [RM0481 Rev 5 — USART/UART §56, LPUART §57](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf), trang 2596–2742.

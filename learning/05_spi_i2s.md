# 05 — SPI và I²S trên STM32H533CE

## Mục tiêu và phạm vi

H533CE có **4 SPI**, trong đó SPI1/2/3 có chức năng I²S; SPI4 không có I²S. SPI1/2/3 hỗ trợ khung 4–32 bit, SPI4 4–16 bit (**DS14539 Rev 3, bảng 2 trang 15 và bảng 11 trang 57**). Học SPI trước; chỉ học I²S khi làm âm thanh số.

## Mô hình cần nắm

SPI dùng SCK, MOSI, MISO, NSS/CS. Một xung clock dịch đồng thời một bit ra và một bit vào; vì vậy muốn *đọc* slave, master vẫn phải phát dummy data. Không có địa chỉ hay ACK chung ở tầng SPI: mỗi IC định nghĩa lệnh, độ dài địa chỉ, dummy cycles và cách chọn CS riêng. CPOL/CPHA tạo bốn mode, nhưng mode đúng là mode trong datasheet của slave. Tốc độ an toàn phải thỏa cả giới hạn slave, dây/mạch và timing I/O MCU. Một transaction thường là `CS xuống → opcode → address → data/dummy → CS lên`; không nhả CS giữa chừng nếu slave yêu cầu giữ liên tục.

## Cấu trúc transaction và lý do chọn từng mode

**Định nghĩa.** SPI là liên kết đồng bộ do controller cấp SCK; một cạnh dịch bit ra, cạnh còn lại lấy mẫu. Lợi ích là full duplex, tốc độ cao, ít logic đóng gói, phù hợp Flash, ADC, màn hình trên cùng PCB. Nó không quy định địa chỉ, frame chuẩn, CRC, ACK, gửi lại hoặc phân quyền nhiều controller: datasheet *slave* định nghĩa tất cả lệnh và thời điểm CS.

```text
CS    ─────┐________________________________________┌─────
SCK        0 1 2 ... (8 × tổng số byte + dummy clocks)
MOSI       [opcode][address?][dummy?][data ghi/dummy]
MISO       [không hợp lệ][status?][data đọc.........]
```

Ví dụ giao thức thanh ghi trong code là **giả định của bài học**, không phải frame SPI chung. Mỗi transaction phải giữ CS theo yêu cầu slave; CS lên thường reset bộ giải mã lệnh và giúp đồng bộ lại sau lỗi. MISO có thể high-Z khi CS không chọn, vì thế không suy luận `0xFF` là dữ liệu hợp lệ. Nhiều slave chia sẻ SCK/MOSI/MISO nhưng mỗi slave cần CS riêng; không cho hai slave cùng kéo MISO.

| Mode/cách dùng | Cơ chế và bài toán giải quyết | Chọn khi nào; giới hạn |
|---|---|---|
| CPOL=0/1, CPHA=0/1 | CPOL chọn mức SCK idle; CPHA chọn cạnh lấy mẫu, tạo 4 mode. | Chỉ chọn theo datasheet slave; sai mode thường lệch bit. |
| MSB/LSB first | Quy định thứ tự bit trong từng word. | Theo slave; nhầm bit order tạo byte đảo bit. |
| Full duplex/2-wire receive/1-wire | Full duplex dịch hai chiều; 1-wire giảm chân khi slave hỗ trợ bidirectional. | Full duplex cho đa số IC; 1-wire cần đổi hướng đúng thời điểm và tránh tranh chấp. |
| GPIO CS/hardware NSS | GPIO cho phép giữ CS xuyên nhiều lời gọi và chèn delay; hardware NSS giảm thao tác CPU khi timing tương thích. | Kiểm datasheet slave và xung NSS thực tế; DMA complete chưa chắc bit cuối đã ra dây. |
| Polling/interrupt/DMA | DMA chuyển block lớn với ít tải CPU; polling đơn giản cho vài byte. | DMA phải quản lý buffer, cache và CS cho đến khi truyền thực sự xong. |

SPI có SCK nên đồng bộ bit trong lúc CS active; không có cơ chế sửa dữ liệu sai. Nếu slave có status/CRC riêng, kiểm theo datasheet đó. Nếu không, ứng dụng có thể đọc lại, so sánh hoặc thêm checksum trong giao thức của *hai đầu tự thiết kế*. Retry phải bắt đầu transaction mới và chỉ an toàn nếu lệnh idempotent hoặc có sequence/transaction ID. Timeout HAL chỉ báo giao dịch cục bộ bị kẹt, không chứng minh slave đã xử lý lệnh. Giảm SCK, dây ngắn, GND chung, decoupling, chọn slew phù hợp và đo setup/hold tại chân slave trước khi thêm retry phần mềm.

**I²S khác SPI dù dùng khối phần cứng chung:** BCLK đồng bộ từng bit, WS/LRCLK đánh dấu kênh trái/phải và ranh giới mẫu; SD mang các slot audio liên tiếp. Philips I²S có trễ một BCLK từ cạnh WS đến MSB; left-justified đặt MSB ngay tại chuyển WS. Chọn đúng format, độ rộng mẫu/slot và polarity theo codec, nếu không tiếng méo hoặc đổi kênh. Stream I²S không có ACK/retry từng mẫu; DMA double buffer, phát hiện underrun và chính sách mute/điền zero mới giữ audio liên tục. Clock audio/jitter ảnh hưởng chất lượng hơn việc retry mẫu đã trễ.

**Bài kiểm chứng:** đọc cùng thanh ghi ở 4 SPI mode, ghi capture CS/SCK/MOSI/MISO và chỉ ra mode đúng. Tăng SCK từng nấc, ghi mức đầu tiên lỗi CRC/status hoặc đọc sai, rồi đo setup/hold và cạnh tín hiệu tại slave.

## Trình tự cấu hình

1. Kiểm tra chân SPI và AF trong **DS §4.1–4.3, trang 61–95** cho vỏ 48 chân; đừng suy từ sơ đồ khối của cả họ.
2. Bật GPIO clock, SPI clock; cấu hình SCK/MOSI/MISO AF. CS có thể là GPIO output để kiểm soát transaction rõ ràng.
3. Chọn master/slave, mode CPOL/CPHA, chiều truyền, data size, bit order, tần số SCK và NSS management.
4. Đọc lại cấu hình bus bằng logic analyzer, sau đó mới tăng clock.
5. Với luồng lớn, dùng FIFO, interrupt hoặc DMA; xử lý overrun/underrun và kết thúc transaction theo cờ tương ứng.

## Ví dụ HAL: đọc thanh ghi của một SPI slave giả định

Đoạn sau **minh họa giao thức** `[opcode đọc = 0x80 | địa chỉ][1 byte dữ liệu]`; thay opcode và mode bằng datasheet của IC thật. Hàm giả định `hspi1` đã được CubeMX tạo đúng chân và mode; `CS` là GPIO output.

```c
#include "stm32h5xx_hal.h"

extern SPI_HandleTypeDef hspi1;

HAL_StatusTypeDef spi_read_reg(GPIO_TypeDef *cs_port, uint16_t cs_pin,
                                uint8_t reg, uint8_t *value)
{
    uint8_t tx[2] = { (uint8_t)(0x80u | reg), 0xFFu };
    uint8_t rx[2] = {0};
    HAL_GPIO_WritePin(cs_port, cs_pin, GPIO_PIN_RESET);
    HAL_StatusTypeDef result = HAL_SPI_TransmitReceive(&hspi1, tx, rx, 2, 100);
    HAL_GPIO_WritePin(cs_port, cs_pin, GPIO_PIN_SET);
    if (result == HAL_OK) *value = rx[1];
    return result;
}
```

Ví dụ chọn SPI1 vì PA5/PA6/PA7 lần lượt hỗ trợ SCK/MISO/MOSI theo bảng AF trang 82. Trước khi nối mạch, kiểm tra xem các chân đó có được board dùng cho mục đích khác không.

## Những lỗi senior thường kiểm tra đầu tiên

- **Dữ liệu lệch một bit:** CPHA/CPOL hoặc lấy mẫu ở sai cạnh; xác nhận bằng `0xAA`, `0x55` và analyzer.
- **Đọc toàn `0xFF`/`0x00`:** CS sai, MISO hở, mức logic sai, IC chưa cấp nguồn hoặc transaction thiếu dummy clocks.
- **Byte đầu đúng, byte sau sai:** CS bị nhả giữa các lời gọi HAL; đóng gói toàn transaction vào một lần truyền hoặc giữ CS thủ công.
- **Chạy chậm thì đúng:** tốc độ SCK vượt giới hạn hệ thống, rise time xấu, dây dài, thiếu decoupling hoặc timing chưa đủ.
- **DMA/cache:** khi dùng bộ đệm trong vùng cacheable, cần xem khả năng nhìn thấy dữ liệu giữa CPU và DMA theo kiến trúc bộ nhớ/cache; không giả định `volatile` giải quyết coherency.

## I²S cần học thêm khi làm audio

Phân biệt WS/LRCLK, BCLK, SD, MCLK; format Philips/left/right justified; số bit mẫu và slot; clock audio, đồng bộ TX/RX, DMA đôi buffer và xử lý underrun. I²S ở đây là chức năng ghép với SPI1/2/3; không dùng đồng thời SPI thường trên cùng instance.

## Bài tập

1. Đọc ID của một cảm biến/bộ nhớ SPI và xác minh bằng analyzer.
2. Đọc liên tục 256 byte; so sánh polling, interrupt và DMA về tải CPU.
3. Cố tình chọn sai CPHA, sai CS và tốc độ quá cao; ghi lại triệu chứng và cách chứng minh nguyên nhân.

## Tài liệu gốc

- [DS14539 Rev 3 — §3.38, bảng 11, §4.3](../doc/DS_stm32h533ce%20%281%29.pdf), trang 55–57, 82–95.
- [RM0481 Rev 5 — §58 SPI/I²S](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf), trang 2743–2818.

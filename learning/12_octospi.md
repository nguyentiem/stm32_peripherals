# 12 — OCTOSPI và bộ nhớ nối tiếp ngoài trên STM32H533CE

## Mục tiêu và giới hạn của package CE

H533CE có **OCTOSPI controller**, nhưng bản 48 chân chỉ đưa ra cấu hình **ít nhất Quad-SPI**; khả năng giao tiếp octal của peripheral không đồng nghĩa đủ tám đường dữ liệu ở package này. Datasheet DS14539 Rev 3 §2 trang 14 nêu rõ “at least one Quad-SPI available on all packages”; kiểm tra từng chân OCTOSPI theo package và bảng AF §4 trước khi thiết kế board. H533CE **không có FMC** và **không có SDMMC** theo bảng 2 trang 15.

Sau bài này bạn phải phân biệt indirect/status-polling/memory-mapped, đọc được JEDEC ID của flash ngoài, hiểu dummy cycles và kiểm tra coherency khi CPU hoặc DMA truy cập vùng memory-mapped.

## Định nghĩa, nguyên lý và giao dịch trên dây

**OCTOSPI** là bộ điều khiển nối MCU với NOR flash hoặc RAM nối tiếp ngoài; nó phát clock, chip select và các pha lệnh/địa chỉ/dữ liệu, nhờ vậy tiết kiệm chân hơn bus song song và có thể ánh xạ vùng nhớ ngoài vào không gian địa chỉ CPU. Phần cứng STM32 tạo pha và chuyển byte; **chip nhớ** quyết định opcode, số dummy cycle, cách write-enable/program/erase và trạng thái busy. Bộ điều khiển không tự tạo CRC, ACK hay giao thức lưu dữ liệu an toàn cho NOR flash thông dụng (RM0481 phần OCTOSPI; cần đối chiếu datasheet IC nhớ cụ thể).

```text
CS↓ | instruction/opcode | optional address | optional alternate bytes |
    | dummy clocks (nếu lệnh yêu cầu) | data bytes | CS↑
Ví dụ JEDEC ID 1-1-1: CS↓ | 0x9F (8 clock, IO0) |
                         manufacturer/type/capacity trên IO1 | CS↑
```

Số lane có thể khác nhau theo pha: `1-1-4` là instruction 1 lane, address 1 lane, data 4 lane. SDR lấy mẫu một cạnh clock; DTR/DDR dùng hai cạnh nếu **cả MCU, package/chân và chip nhớ** hỗ trợ mode tương ứng. Dummy cycle tạo thời gian để flash tìm dữ liệu trước pha đọc; quá ít sẽ lệch bit/byte, quá nhiều chỉ giảm throughput. `CS↑` kết thúc transaction và đặt ranh giới đồng bộ; sau reset, khởi động ở mode mặc định của *chip nhớ* rồi xác nhận ID trước khi bật quad/tốc độ cao. Không được lấy tên OCTOSPI làm bằng chứng package 48 chân có đủ 8 data lane.

NOR SPI thường không có ACK cho từng byte và không tự truyền lại khi đọc sai. `WREN` đặt write-enable latch; sau program/erase phải polling busy với timeout, đọc lại và so sánh CRC ở firmware nếu dữ liệu quan trọng. Timeout không nên khiến firmware gửi lại program một cách mù quáng: đọc trạng thái/nội dung trước, vì lệnh trước có thể đã hoàn thành. Chống nhiễu gồm giảm clock, kiểm tra biên sample/DLYB, chiều dài trace, decoupling, mức drive và kiểm tra CRC; đó là biện pháp thiết kế/ứng dụng, không phải CRC mặc định của OCTOSPI. Với RAM ngoài, cần xem giao thức riêng của chip về latency, refresh và tính toàn vẹn.

## Kiến thức cốt lõi

| Chế độ | Dùng khi nào | Điều cần kiểm tra |
|---|---|---|
| Indirect | Lệnh rời như đọc ID/status, ghi/erase; firmware kiểm soát từng transaction | Opcode, address size, số đường ở từng pha, dummy cycles, CS timing; overhead CPU lớn hơn |
| Automatic status polling | Hardware lặp lệnh đọc status để tránh CPU bận chờ flash hết busy | Mask/match, chu kỳ polling, timeout; chỉ xác nhận bit status, không xác nhận dữ liệu đã đúng |
| Memory-mapped | Hardware đổi CPU read thành lệnh đọc flash, thuận tiện asset/code/data lớn | Cấu hình read command, latency và cache; ghi/erase cần chuyển mode và bảo vệ mọi luồng đang đọc vùng này |

OCTOSPI điều khiển **giao thức truyền**, còn lệnh cụ thể (`0x9F` JEDEC ID, page program, sector erase...) phụ thuộc datasheet của chip nhớ. Nhiều flash cần bật Quad Enable trước khi dùng bốn data lines. Ghi NOR flash thường chỉ đổi bit 1→0; muốn đổi 0→1 phải erase sector. Không giả định ngắt nguồn giữa page program là an toàn.

### Các yếu tố timing

Tốc độ bus phụ thuộc clock, prescaler, sample shifting, DLYB, dummy cycles và chất lượng đường mạch. DLYB có thể cần hiệu chỉnh lại theo điện áp/nhiệt độ ở tốc độ cao (DS §3.21 trang 38). Bắt đầu ở clock thấp với single-line SPI để đọc ID, sau đó chuyển quad và tăng dần clock trong khi kiểm tra CRC dữ liệu.

## Ví dụ C: thuật toán đọc ID có timeout

Đây là **mã khung** độc lập phiên bản HAL: `ospi_cmd()` và `ospi_receive()` biểu thị lời gọi driver OCTOSPI của dự án, không phải API STM32Cube có sẵn. Tách thuật toán khỏi HAL giúp bạn thay peripheral driver mà vẫn giữ trình tự đúng.

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>

typedef enum { OSPI_OK, OSPI_TIMEOUT, OSPI_IO_ERROR } ospi_result_t;

/* Driver phải giới hạn thời gian chờ, cấu hình command 0x9F không address,
 * instruction/data một đường, và giữ CS đúng trong toàn transaction. */
extern ospi_result_t ospi_cmd(uint8_t opcode, uint32_t timeout_ms);
extern ospi_result_t ospi_receive(uint8_t *dst, size_t len, uint32_t timeout_ms);

bool flash_read_jedec_id(uint8_t id[3])
{
    if (id == NULL) return false;
    if (ospi_cmd(0x9Fu, 10u) != OSPI_OK) return false;
    if (ospi_receive(id, 3u, 10u) != OSPI_OK) return false;
    return !(id[0] == 0x00u || id[0] == 0xFFu);
}
```

Khi triển khai bằng STM32Cube HAL, `ospi_cmd` và `ospi_receive` phải thuộc **cùng transaction cấu hình OCTOSPI**, không nhả CS giữa pha lệnh và pha dữ liệu nếu bộ nhớ yêu cầu. Xem ví dụ `OSPI_RegularCmdTypeDef`, `HAL_OSPI_Command`, `HAL_OSPI_Receive` tương ứng phiên bản CubeH5 đang dùng; tra RM0481 §24/25 để chọn chương có instance implementation phù hợp H533.

## Những kỹ thuật kiểu senior

- **Dùng JEDEC ID và SFDP:** không hard-code dung lượng, sector size, dummy cycles theo một flash rồi thay IC khác mà không kiểm tra.
- **Thiết kế cập nhật an toàn:** dữ liệu quan trọng nên có version, CRC và commit marker; ghi bản mới trước khi xóa bản cũ. Tính wear budget theo datasheet flash ngoài.
- **Memory-mapped không xóa/ghi tùy tiện:** khóa các luồng CPU/ISR có thể đang fetch hoặc đọc vùng ánh xạ trước khi đổi mode; invalidate cache nếu dữ liệu thay đổi phía ngoài cache.
- **Lỗi ở tốc độ cao:** dùng pattern `0x00`, `0xFF`, `0xAA`, `0x55`, địa chỉ ranh giới trang/sector và CRC; thử ở góc điện áp/nhiệt độ nếu là sản phẩm thật.
- **DMA:** buffer phải ở vùng DMA truy cập được, có căn chỉnh thích hợp và được xử lý cache coherency khi cần. `volatile` không tự làm sạch hay invalidate cache.

## Bài tập đạt chuẩn

1. Đọc JEDEC ID ở tốc độ thấp; ghi lại dạng sóng instruction/data và xác nhận ID với datasheet chip nhớ.
2. Ghi một page, đọc lại và so CRC; thử vượt ranh giới page để thấy vì sao driver phải chia nhỏ.
3. Bật memory-mapped read; đo throughput khi cache bật/tắt. Sau khi sửa dữ liệu flash, chứng minh dữ liệu CPU đọc không bị stale.
4. Viết cơ chế lưu cấu hình hai bản với CRC và commit marker; mô phỏng mất nguồn ở từng bước.

## Tài liệu gốc

- [DS14539 Rev 3 — §2, bảng 2, §3.20–3.21, §4](../doc/DS_stm32h533ce%20%281%29.pdf), trang 14–16, 37–38, 61–95.
- [RM0481 Rev 5 — OCTOSPI §24/25, DLYB §27](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf), trang 990–1106 và 1189–1194.

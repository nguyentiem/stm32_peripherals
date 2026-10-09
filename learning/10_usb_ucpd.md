# 10 — USB full speed và USB Type-C/PD trên STM32H533CE

## Mục tiêu

H533CE có một bộ điều khiển **USB 2.0 full speed host/device** và một **UCPD** cho tín hiệu Type-C/Power Delivery (DS14539 Rev 3, bảng 2 trang 15, §3.41–3.42 trang 59–60). Sau bài này, bạn phải phân biệt rõ **USB data** và **USB Type-C/PD**, chạy được một thiết bị USB CDC, và biết cách kiểm tra clock, nguồn, dây và descriptor khi enumeration thất bại.

## Định nghĩa, cấu trúc giao dịch và độ tin cậy

**USB FS** giải quyết kết nối thiết bị ngoại vi với host qua bus được host điều phối; MCU có thể chạy vai trò host hoặc device tùy firmware và mạch nguồn. Cặp D+/D− truyền vi sai, tín hiệu được mã hóa NRZI với bit stuffing; đầu gói có SYNC để bên nhận khóa nhịp, cuối gói có EOP. Host phát token rồi thiết bị hoặc host phát data/handshake theo hướng truyền. Trên H533CE, bộ điều khiển USB xử lý gói/endpoint; firmware USB stack xử lý descriptor, class và dữ liệu ứng dụng (RM0481 §61; DS14539 §3.41).

```text
OUT transaction: host TOKEN[SYNC|PID OUT|address|endpoint|CRC5|EOP]
              -> host DATA[SYNC|PID DATA0/1|payload|CRC16|EOP]
              -> device HANDSHAKE[ACK hoặc NAK/STALL]
IN transaction:  host TOKEN IN -> device DATA hoặc NAK/STALL -> host ACK nếu nhận data hợp lệ
SETUP control:   host SETUP token -> DATA0 (8 byte request) -> ACK;
                 tiếp theo là data stage (nếu có) và status stage ngược chiều.
```

Địa chỉ/endpoint thuộc USB; ý nghĩa byte CDC hay `[length][payload][CRC]` do class/ứng dụng định nghĩa. Full speed bulk packet tối đa 64 byte; một transfer/stream CDC có thể gồm nhiều packet, vì thế phải tự đặt ranh giới bản tin ứng dụng. USB dùng CRC5 cho token và CRC16 cho data, bit stuffing giúp phục hồi clock và phát hiện một số lỗi. Với control/bulk/interrupt, ACK xác nhận *packet ở tầng USB*; NAK báo chưa sẵn sàng, host có thể thử lại theo lịch và driver. DATA0/DATA1 giúp nhận ra lần gửi lặp sau khi ACK thất lạc. Isochronous ưu tiên thời gian thực và **không retransmit**, vì gửi lại sẽ trễ mốc phát. USB ACK không xác nhận firmware ứng dụng đã xử lý lệnh; nếu cần bảo đảm thao tác, thêm ID bản tin, phản hồi ứng dụng, timeout và xử lý lệnh lặp an toàn. Khi reset/reconnect, endpoint và địa chỉ được thiết lập lại; giao thức ứng dụng phải khởi tạo lại phiên nếu cần.

| Kiểu transfer | Vì sao tồn tại, khi chọn | Đánh đổi |
|---|---|---|
| Control | Cấu hình/enumeration qua EP0, có setup/data/status. | Bắt buộc cho thiết bị, không phù hợp stream lớn. |
| Bulk | Dùng thời gian bus còn lại để chuyển dữ liệu chính xác. | Có retry nhưng latency không được bảo đảm; hợp CDC/MSC. |
| Interrupt | Host hỏi theo chu kỳ để giới hạn trễ thăm dò. | Hợp HID/sự kiện nhỏ, không phải ngắt tức thời từ device lên host. |
| Isochronous | Giữ lịch băng thông cho audio/video. | Có thể mất packet; ứng dụng xử lý gap, không chờ retry. |

Nhiễu điện được giảm nhờ truyền vi sai, định tuyến cặp dây đúng trở kháng, ESD và đường mass hợp lý; CRC/retry chỉ xử lý lỗi đã xảy ra. Đo VBUS, D+/D− và dùng USB analyzer khi lỗi enumeration; oscilloscope thông thường có thể cho thấy reset và activity nhưng không thay phân tích giao dịch.

**Type-C/UCPD là lớp riêng:** CC1/CC2 và Rp/Rd phát hiện cắm cáp, orientation, vai trò và dòng quảng cáo. USB PD dùng thông điệp trên CC với header (message type, vai trò, message ID, số data object), payload các object 32 bit và CRC; lớp PD có GoodCRC, message ID, timeout/retry theo quy tắc PD. GoodCRC chỉ xác nhận nhận thông điệp PD, không chứng minh power contract đã được thực thi. Firmware policy engine cùng mạch power path phải xử lý Accept/PS_RDY, giới hạn dòng và tháo cáp. Không áp dụng cơ chế ACK của USB data cho PD hoặc ngược lại (RM0481 §62; DS14539 §3.42).

## Kiến thức lõi

### USB data

- USB là mô hình **host–device**. Host cấp lịch giao dịch, device trả lời qua các endpoint. Không thể nối hai device rồi mong chúng tự trao đổi.
- H533 hỗ trợ full speed (12 Mbit/s), endpoint điều khiển EP0, transfer control/bulk/interrupt/isochronous và vùng SRAM USB riêng. CDC là lớp tiện để tạo cổng serial ảo; MSC/HID có đặc tả khác.
- Enumeration: reset bus → đọc device descriptor → gán địa chỉ → đọc configuration/interface/endpoint descriptors → chọn configuration. Một lỗi descriptor khiến máy tính không tạo cổng COM dù chân D+/D− vẫn có tín hiệu.
- USB cần clock đúng yêu cầu; xem **RCC/CRS** và lựa chọn clock USB trong RM. Datasheet cho biết có clock recovery. Kiểm tra nguồn USB và mạch kết nối theo pinout thực tế của gói 48 chân.
- Bulk/CDC không bảo toàn ranh giới lệnh ứng dụng. Dữ liệu có thể đến theo nhiều mảnh; thiết kế parser có length/framing, không giả định một callback là một gói hoàn chỉnh.

### Type-C và UCPD

UCPD theo dõi CC1/CC2, orientation, vai trò source/sink, các điện trở Rp/Rd và thông điệp PD. Nó **không phải** USB data peripheral và không tự tạo ra cổng CDC. Nếu sản phẩm chỉ dùng cổng Type-C để truyền USB 2.0 và nhận nguồn mặc định, phần cứng CC vẫn phải được thiết kế đúng; nếu đàm phán điện áp/dòng qua PD, cần stack PD và mạch nguồn phù hợp. Không dùng firmware để yêu cầu mức điện áp mà power path không chịu được.

## Bài thực hành CDC

Tạo project STM32CubeMX/CubeIDE cho **đúng part STM32H533CE**, bật USB device CDC, chọn clock USB hợp lệ và tạo code middleware. Đoạn dưới minh họa cách gửi và xử lý `USBD_BUSY`; tên API phụ thuộc gói middleware Cube được sinh ra.

```c
#include "usbd_cdc_if.h"

/* Gọi từ task/main loop, không gọi từ ISR ưu tiên cao. */
int usb_cdc_try_send(const uint8_t *data, uint16_t len)
{
    if (data == NULL || len == 0) return -1;
    uint8_t status = CDC_Transmit_FS((uint8_t *)data, len);
    if (status == USBD_OK)   return 0; /* Đã nhận vào USB stack */
    if (status == USBD_BUSY) return 1; /* Đưa vào hàng đợi để gửi sau */
    return -2;
}
```

`USBD_OK` ở đây không có nghĩa PC đã xử lý nội dung; nó chỉ xác nhận USB stack nhận yêu cầu. Khi truyền nhiều bản tin, cần queue và tín hiệu hoàn tất TX. Nếu nhận CDC, callback RX nên **chép dữ liệu vào ring buffer**, re-arm reception ngay và parse ở main/task.

## Chuỗi kiểm tra kiểu senior

1. Xác nhận part, chân USB và supply trong DS §4, sau đó đo điện áp cấp và tín hiệu reset.
2. Xác nhận clock USB, nguồn clock và CRS nếu dùng HSI48; sai số clock thường gây lỗi kết nối không ổn định.
3. Kiểm tra D+/D−, CC và trở bảo vệ trên schematic; tránh đảo dây hoặc thiếu CC khi dùng đầu Type-C.
4. Ghi log từng pha enumeration; so sánh descriptor sinh ra với interface mong muốn.
5. Tách lỗi vật lý, enumeration, class driver và giao thức ứng dụng thành bốn lớp khác nhau.
6. Test hot-plug, rút cáp khi đang TX, suspend/resume và nhiều gói nhỏ liên tiếp.

## Bài tập đạt chuẩn

- Tạo CDC echo nhưng giữ ranh giới gói ứng dụng bằng `[length][payload][CRC]`.
- Gửi liên tục từ PC xuống MCU và ngược lại; đo throughput và số lần `USBD_BUSY`, không mất byte.
- Rút cáp giữa lúc truyền; hệ thống tiếp tục chạy, cắm lại thì phục hồi giao tiếp.
- Nếu có bo Type-C PD, đọc trạng thái CC/orientation trước khi thử đàm phán PD.

## Tài liệu gốc

- [DS14539 Rev 3 — bảng 2, §3.41–3.42, chân §4](../doc/DS_stm32h533ce%20%281%29.pdf), trang 15, 59–60, 61–95.
- [RM0481 Rev 5 — USB §61, UCPD §62, RCC §11, CRS §12](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf), trang 2953–3038, 480–614.

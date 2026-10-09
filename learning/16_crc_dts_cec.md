# 16 — CRC, cảm biến nhiệt số và HDMI-CEC

## Phạm vi

Đây là ba ngoại vi **có trên H533CE** nhưng thường học sau GPIO, timer, giao tiếp cơ bản, ADC/DMA. Chúng phục vụ ba bài toán khác nhau: CRC kiểm tra toàn vẹn dữ liệu, DTS đo nhiệt độ chip, CEC điều khiển thiết bị HDMI. Datasheet DS14539 Rev 3 §3.18, §3.23, §3.43; RM0481 §19, §29, §65. Chọn phần phù hợp dự án, không cần dùng cả ba trong một ứng dụng.

**Cách tư duy:** với mỗi khối, phân biệt tín hiệu đầu vào, phép biến đổi phần cứng, tín hiệu/cờ đầu ra và quyết định của ứng dụng. CRC tạo một mã kiểm lỗi; DTS biến nhiệt độ die thành tần số rồi đếm theo clock tham chiếu; CEC truyền thông điệp qua một dây open-drain. Chúng không tự giải quyết yêu cầu ở tầng cao hơn: CRC không xác thực người gửi, DTS không biết nhiệt độ linh kiện ngoài, ACK của CEC không chứng minh thiết bị đã thực hiện lệnh.

## 1. CRC — kiểm tra dữ liệu truyền/lưu

CRC là phép tính phát hiện lỗi, **không phải mã hóa hay chữ ký bảo mật**. Hai đầu phải thống nhất polynomial, width, init value, reflection/bit order, XOR output, thứ tự byte và phần dữ liệu được tính. Nếu một tham số khác, CRC sẽ không khớp dù dữ liệu không hỏng. Peripheral CRC tăng tốc tính toán, nhưng phải so với vector chuẩn của giao thức trước khi dùng.

**Nguyên lý:** thanh ghi dịch chia đa thức trên luồng bit đầu vào và giữ phần dư làm CRC. Người nhận tính lại cùng cấu hình rồi so phần dư; bit lỗi hoặc mất/đổi thứ tự byte thường làm phần dư khác. Không có khóa nên kẻ chủ động sửa dữ liệu cũng có thể tính lại CRC. H533CE có polynomial mặc định CRC-32 Ethernet và polynomial lập trình được kích thước 7/8/16/32 bit; tùy chọn đảo bit đầu vào/đầu ra giải quyết khác biệt thứ tự bit giữa các giao thức. Các bit `RTYPE` hoán đổi byte/halfword trong RM chỉ có trên H543/553, **không dùng cho CE** (RM0481 §19.2–19.4, tr. 825–827).

| Cấu hình | Vì sao tồn tại; khi dùng | Bẫy thực tế |
|---|---|---|
| CRC-32 mặc định | Kiểm tra dữ liệu/firmware theo cấu hình phù hợp; phần cứng tính nhanh. | “CRC-32” chưa đủ để khẳng định cùng kết quả: init, reflection, final XOR có thể khác. |
| Polynomial và width khác | Khớp đúng giao thức hoặc định dạng lưu sử dụng CRC-8/16/32. | Không đổi polynomial giữa một phép tính; đối chiếu vector chuẩn. |
| Input/output reversal | Khớp cách giao thức dịch MSB/LSB và thứ tự byte. | Word memory order khác thứ tự byte trên dây; test độ dài không chia hết bốn. |

### Ví dụ HAL minh họa

Giả định `hcrc` được CubeMX cấu hình với đúng polynomial, init và input/output format của giao thức. Chỉ tính trên word array khi bạn đã xác định rõ thứ tự byte và padding; không ép kiểu buffer byte tùy tiện.

```c
#include "stm32h5xx_hal.h"
#include <stdint.h>

extern CRC_HandleTypeDef hcrc;

uint32_t frame_crc_words(const uint32_t *words, uint32_t word_count)
{
    /* Kết quả chỉ có nghĩa khi cấu hình hcrc khớp vector chuẩn của giao thức. */
    return HAL_CRC_Calculate(&hcrc, (uint32_t *)words, word_count);
}
```

**Bài kiểm chứng:** tính CRC bằng phần mềm với chuỗi ASCII `123456789` và một buffer có byte 0x00 ở giữa, rồi so với phần cứng. Kiểm tra cả buffer không chia hết 4 byte theo API HAL đang dùng. Với thông điệp rỗng, đối chiếu giá trị khởi tạo/final XOR theo giao thức; không gọi `HAL_CRC_Calculate` với `word_count = 0` nếu API HAL không cho phép.

## 2. DTS — theo dõi nhiệt độ chip

DTS là cảm biến nhiệt độ **bên trong die**, không đại diện chính xác cho nhiệt độ không khí hoặc một linh kiện ở xa. Học cách bật clock, chọn sampling/trigger, đọc kết quả, áp hệ số hiệu chuẩn theo tài liệu và đặt ngưỡng cảnh báo có hysteresis. Đo khi CPU tải nặng và khi ở chế độ nghỉ để thấy nhiệt độ die thay đổi. Không dùng DTS để bảo vệ một MOSFET ngoài nếu không chứng minh được tương quan nhiệt.

**Nguyên lý và mode:** mạch analog tạo sóng vuông có tần số phụ thuộc nhiệt độ; DTS đo tần số đó với PCLK hoặc LSE, rồi chuyển số đếm thành nhiệt độ theo hệ số hiệu chuẩn. PCLK cho đo khi hệ thống chạy, LSE cho phép đo/đánh thức khi PCLK tắt; sampling dài giúp kết quả ít nhiễu hơn nhưng tốn thời gian/năng lượng. Trigger phần mềm dành cho kiểm tra theo yêu cầu; trigger phần cứng/watchdog dành cho giám sát định kỳ hoặc ngưỡng độc lập luồng ứng dụng. Quick measurement giảm độ trễ nhưng bỏ bước calibration trước phép đo, nên phải cân nhắc sai số. RM0481 §29.1–29.3.5, tr. 1317–1319.

**Ví dụ thuật toán xử lý mẫu:**

```c
#include <stdint.h>

typedef struct { int32_t alarm_mC; int32_t clear_mC; } temp_limit_t;

/* temp_mC đã được đổi từ raw DTS theo hệ số hiệu chuẩn của chip. */
static int temp_alarm_update(int was_active, int32_t temp_mC,
                             temp_limit_t limit)
{
    if (was_active) return temp_mC > limit.clear_mC;
    return temp_mC >= limit.alarm_mC;
}
```

Hysteresis ngăn cảnh báo bật/tắt liên tục quanh ngưỡng. Phần chuyển raw → °C phải lấy từ RM/dữ liệu calibration đúng revision; đừng tự dùng công thức của dòng STM32 khác.

## 3. HDMI-CEC — giao tiếp điều khiển tốc độ thấp

CEC là đường điều khiển một dây trong hệ HDMI để các thiết bị gửi lệnh như bật/tắt/chọn nguồn. Đây **không phải** giao diện video HDMI. Cần học logical address, arbitration, ACK, timeout, retry và lệnh của thiết bị; nối phần cứng theo điện áp/điều kiện CEC trong datasheet. CEC có clock độc lập với CPU và có thể đánh thức MCU khỏi Stop (DS §3.43 trang 60).

**Khung CEC trên dây:** `START → HEADER(source 4 bit | destination 4 bit) → [OPCODE] → [0–14 OPERAND]`. Mỗi block gồm **8 data bit MSB trước + EOM + ACK**. `EOM=1` ở block cuối; header có EOM=1 tạo bản tin ping. Sau start, độ dài xung thấp/cao mã hóa bit và điểm bắt đầu bit tiếp theo, nên bên nhận tự đồng bộ theo cạnh trên dây; không có clock SCK riêng. Đây là cấu trúc CEC, còn ý nghĩa opcode/operand theo lệnh HDMI-CEC tương ứng. [RM0481 §65.3.3–65.3.4, tr. 3589–3590](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf)

**Điện và tin cậy:** dây CEC là open-drain/wired-AND 3,3 V với điện trở kéo lên **27 kΩ bên ngoài**; cần cách ly khi thiết bị mất nguồn theo RM §65.3.1 tr. 3588. Trong unicast, follower nhận đúng địa chỉ kéo ACK xuống thấp; broadcast có quy ước ACK khác để follower báo lỗi, vì vậy không dùng cùng một điều kiện “ACK thấp là thành công” cho cả hai. Bên phát phải đợi signal-free time và nhường bus khi thua arbitration; phần cứng có phát hiện bit timing, line error và tự retry sau arbitration lost. Bản tin direct không ACK hoặc broadcast bị negative ACK có thể cần gửi lại; đặt giới hạn retry và quyết định idempotency ở ứng dụng, vì ACK trên dây **không xác nhận TV đã thi hành lệnh**. CEC không có trường CRC trong frame; kiểm lỗi dựa vào timing, error bit, ACK và trạng thái bus. [RM0481 §65.2–65.5, tr. 3587–3593](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf)

| Mode | Mục đích và lúc dùng | Đánh đổi |
|---|---|---|
| Listen | Nhận/quan sát cả message không gửi đến OAR mà không can thiệp bus. | Dùng để debug/monitor; không thay thế xử lý địa chỉ ở ứng dụng. |
| Signal-free time tự động | Phần cứng chọn thời gian nhường bus theo lịch sử truyền. | Phù hợp phần lớn hệ nhiều initiator; fixed SFT chỉ khi ứng dụng cần thời gian xác định và đã kiểm tra chuẩn. |
| Standard/extended RX tolerance | Điều chỉnh biên chấp nhận timing tín hiệu nhận. | Extended giúp bus lệch timing nhưng tăng nguy cơ nhận nhầm tín hiệu xấu; đo trên bus thật. |

**Bài kiểm chứng CEC:** capture một frame direct và một frame broadcast; ghi source/destination, EOM và ACK từng block. Ngắt kết nối follower để xem cờ thiếu ACK và số lần retry; sau đó gây lỗi timing trong môi trường thử, ghi BRE/SBPE/LBPE cùng kết quả ứng dụng. Với lệnh bật/tắt, phản hồi/trạng thái thiết bị phải được quan sát riêng vì ACK dây chỉ nói frame đã được nhận.

**Bài kiểm chứng DTS:** đo cùng board ở PCLK và LSE, ở tải CPU thấp/cao; so thời gian đo và dao động kết quả. Đặt hai ngưỡng có hysteresis, xác nhận không báo động lặp liên tục gần ngưỡng. Không dùng phép thử này để suy ra nhiệt độ môi trường nếu chưa có hiệu chuẩn riêng.

Một state machine tối thiểu của ứng dụng nên là `IDLE → TX_WAIT → ACK/RETRY → IDLE`; RX IRQ chỉ đẩy frame đã kiểm tra định dạng/địa chỉ vào queue. CEC không tự xác thực danh tính thiết bị. Đừng chạy lệnh điều khiển tải trực tiếp trong ISR. Trên H533CE 48 chân, kiểm tra bảng AF §4.3 cho tín hiệu CEC và xung đột với chức năng khác trước khi chọn chân.

## Ghi chú senior chung

- Định nghĩa **observable test** cho từng ngoại vi: CRC đối chiếu vector, DTS đối chiếu nhiệt độ tại nhiều mức tải, CEC dùng analyzer/thiết bị HDMI thật.
- Những khối này vẫn cần clock, reset, IRQ và quyền TrustZone đúng như ngoại vi lớn. Nếu thanh ghi đọc được nhưng không có output, kiểm tra đường clock kernel và pin mux.
- Sai số và giới hạn điện lấy từ datasheet, còn trình tự lập trình/thanh ghi lấy từ reference manual. RM0481 dùng chung nhiều dòng H5: luôn kiểm tra instance của CE trước.

## Tài liệu gốc

- [DS14539 Rev 3 — §3.18, §3.23, §3.43, §4](../doc/DS_stm32h533ce%20%281%29.pdf), trang 36, 39–40, 60–95.
- [RM0481 Rev 5 — CRC §19, DTS §29, CEC §65](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf), trang 825–831, 1317–1334, 3587–3604.

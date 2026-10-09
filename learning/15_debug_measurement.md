# 15 — SWD, trace và đo kiểm firmware trên STM32H533CE

## Mục tiêu

Làm chủ breakpoint, watchpoint, xem thanh ghi ngoại vi, đo thời gian chạy và phân biệt lỗi firmware với lỗi điện/timing. H533CE hỗ trợ SWD/JTAG và các khối debug/trace Cortex-M33; TrustZone và cơ chế secure debug có thể giới hạn quyền truy cập tùy trạng thái thiết bị (DS14539 Rev 3 §3.44 trang 60, RM0481 §66 trang 3605).

**Định nghĩa và nguyên lý:** SWD/JTAG là cổng cho probe đọc/ghi thanh ghi, bộ nhớ và điều khiển trạng thái CPU. Breakpoint dừng CPU tại một lệnh, watchpoint theo dõi truy cập dữ liệu, DWT đếm chu kỳ/sự kiện, còn trace phát thông tin thực thi ra công cụ ngoài. Các cơ chế này giúp trả lời *trạng thái sai ở đâu* và *sai vào thời điểm nào*. Dừng CPU hữu ích cho lỗi logic nhưng làm thay đổi timing; khi lỗi phụ thuộc thời gian, đo bằng GPIO marker, logic analyzer hoặc trace sẽ sát điều kiện chạy thật hơn.

## Các lớp quan sát

| Công cụ | Nhìn được gì | Hạn chế cần nhớ |
|---|---|---|
| SWD debugger | CPU halt, thanh ghi, bộ nhớ, breakpoint | Halt CPU thay đổi timing, watchdog/ngoại vi có thể vẫn chạy tùy DBGMCU |
| GPIO marker + scope | Thời gian thật của một đoạn code/ISR | Chiếm chân và thêm vài lệnh ghi GPIO |
| Logic analyzer | UART/SPI/I²C/PWM và tương quan nhiều tín hiệu | Không nhìn trạng thái nội bộ CPU |
| DWT cycle counter | Số chu kỳ CPU giữa hai điểm | Clock CPU thay đổi làm quy đổi sang thời gian thay đổi; kiểm tra tính năng/cấu hình debug |
| Trace/ITM/ETM | Dòng sự kiện hoặc lệnh với ít ảnh hưởng hơn halt | Cần đường truyền và probe hỗ trợ; package 48 chân có thể không đưa đủ trace pins |

### Vì sao có nhiều cách debug?

| Cách dùng | Vấn đề giải quyết và cơ chế | Khi chọn; đánh đổi |
|---|---|---|
| Breakpoint/halt | Dừng CPU để đọc stack, biến và thanh ghi tại một điểm. | Lỗi logic tái hiện ổn định; có thể làm watchdog, DMA và bus giao tiếp lệch thời gian. |
| Watchpoint | DWT bắt truy cập tới địa chỉ dữ liệu quan trọng. | Tìm chỗ ghi hỏng buffer; số lượng phần cứng ít, tối ưu compiler có thể làm biểu hiện khó đọc. |
| GPIO marker + scope/analyzer | Kéo chân lên/xuống quanh đoạn code, quan sát thời gian thực cùng tín hiệu ngoại vi. | ISR latency, CS/PWM/IRQ; tốn chân và một ít chu kỳ lệnh. |
| Cycle counter | DWT tích lũy số chu kỳ CPU, phép trừ cho chi phí thuật toán. | Benchmark đoạn ngắn; phải biết clock hiện tại và xem ngắt có chen vào không. |
| Trace/log buffer | Ghi sự kiện ngắn khi chạy, giải mã sau; ETM/ITM có thể xuất luồng trace qua probe. | Lỗi chỉ xuất hiện ở tốc độ thật; cần buffer/băng thông, trace chân ngoài phụ thuộc package. |

## Trình tự debug kiểu senior

1. **Tái hiện:** ghi bản firmware, VDD, clock, nhiệt độ, đầu vào và thời điểm lỗi. Lỗi chỉ xuất hiện khi có debugger khác lỗi chạy độc lập.
2. **Khoanh lớp:** nguồn/reset → clock → pin mux → peripheral → DMA/IRQ → giao thức ứng dụng. Kiểm tra lần lượt, không sửa nhiều tầng cùng lúc.
3. **Chụp trạng thái trước khi xóa:** reset cause, peripheral status/error flags, địa chỉ DMA, counter và một số byte dữ liệu mẫu.
4. **Đo ngoài chip:** dùng scope cho xung clock/PWM/CS và analyzer cho transaction. Nếu thanh ghi nói đã gửi nhưng chân không đổi, xét AF, clock, output enable và nguồn chân.
5. **Định lượng:** đo latency tối đa ISR, thời gian xử lý buffer, margin so với tốc độ dữ liệu; kết luận bằng số đo thay vì cảm giác.

## Ví dụ C: đo chu kỳ bằng DWT

Đây là ví dụ CMSIS; kiểm tra device header và quyền truy cập khi bật TrustZone. Không dùng khi tính năng cycle counter bị khóa/không hiện diện.

```c
#include "stm32h5xx.h"
#include <stdint.h>

static int cycle_counter_start(void)
{
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
    DWT->CYCCNT = 0u;
    DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk;
    return (DWT->CTRL & DWT_CTRL_CYCCNTENA_Msk) != 0u;
}

static uint32_t measure_cycles(void (*work)(void))
{
    uint32_t begin = DWT->CYCCNT;
    work();
    return DWT->CYCCNT - begin; /* uint32_t wrap được xử lý theo modulo 2^32 */
}
```

Đo nhiều lần, có cả phân bố và giá trị xấu nhất; ngắt có thể chen vào giữa hai lần đọc. Muốn đo *wall time* của một transaction DMA/USB, dùng timer hoặc GPIO marker vì DWT chỉ đếm chu kỳ CPU. Khi tối ưu code, đo cả bản release và điều kiện cache giống production.

## Các lỗi thực tế

- **Mất SWD:** cấu hình PA13/PA14 thành GPIO/AF khác quá sớm; cần sửa firmware hoặc kết nối under reset.
- **Bị treo khi breakpoint:** watchdog không được freeze, USB/CAN tiếp tục có deadline, DMA ghi tràn buffer. Dùng breakpoint cho logic, scope/trace cho timing.
- **Khó tái hiện:** log thay đổi scheduling; thay vì `printf` trong ISR, ghi event ngắn vào ring buffer RAM rồi dump sau.
- **Đọc biến thấy sai:** tối ưu compiler, cache hoặc biến được ISR/DMA sửa. Kiểm tra memory barrier/volatile/critical section theo ngữ cảnh; `volatile` không phải cơ chế đồng bộ đầy đủ.
- **Không truy cập vùng secure:** xem trạng thái TrustZone/lifecycle/debug authentication trước khi kết luận chip hỏng.

## Bài tập đạt chuẩn

1. Đo thời gian ISR EXTI bằng GPIO marker và bằng DWT; giải thích chênh lệch.
2. Tạo lỗi UART overrun có chủ ý; lưu snapshot cờ lỗi và khôi phục mà không reset chip.
3. Ghi nguyên nhân reset vào backup RAM/register rồi in ở lần boot sau; phân biệt watchdog reset và power-on reset.
4. Thử build debug/release, đo tác động của cache và compiler optimization lên một thuật toán ngắn.

## Tài liệu gốc

- [DS14539 Rev 3 — §3.44, §4 pinout/AF](../doc/DS_stm32h533ce%20%281%29.pdf), trang 60–95.
- [RM0481 Rev 5 — §66 Debug, §17 NVIC](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf), trang 3605–3759 và 758–774.
- [PM0264 — Cortex-M33 programming manual](../doc/pm0264-stm32-cortexm33-mcus-and-mpus-programming-manual-stmicroelectronics%20%281%29.pdf), phần debug/exception theo mục lục của bản PDF.

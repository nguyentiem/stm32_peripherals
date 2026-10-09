# Lộ trình học ngoại vi STM32H533CE

Bộ ghi chú này dành riêng cho **STM32H533CE (48 chân, Flash 512 KiB)**. Mỗi chủ đề có một file Markdown: điều cần hiểu, cách cấu hình, ví dụ C, lỗi hay gặp, kỹ thuật kiểm chứng trên phần cứng và bài tập. Các ví dụ HAL giả định project STM32CubeH5 đã chọn **đúng part/package**; những ví dụ ghi *mã khung* cần được nối với driver của project trước khi build.

Mỗi bài giải thích **định nghĩa → nguyên lý/cách hoạt động → lợi ích và ứng dụng → lý do tồn tại của từng mode → khi nào chọn và đánh đổi**. Các bài giao tiếp đi thêm từ **tín hiệu trên dây và cấu trúc frame** đến đồng bộ, ACK, phát hiện lỗi, gửi lại và phục hồi. Nếu một giao thức không cung cấp cơ chế đó, bài viết chỉ rõ phần ứng dụng phải tự thiết kế. Đây là quy ước của skill `embedded-topic-research`.

## Bắt đầu ở đâu?

| Thứ tự | Chủ đề | Kết quả nên đạt |
|---|---|---|
| 01 | [Clock, nguồn và boot](01_clock_power.md) | Tính và đo clock, chọn VOS/Flash latency an toàn, chụp reset cause. |
| 02 | [GPIO, AF, EXTI và NVIC](02_gpio_exti.md) | Chọn chân có thật trên CE; cấu hình output/input và ngắt không lặp vô hạn. |
| 03 | [Timer và PWM](03_timers.md) | Tính PSC/ARR/CCR, đo PWM/capture, dùng trigger. |
| 04 | [USART/UART/LPUART](04_uart.md) | Truyền nhận ổn định, hiểu lỗi và đóng khung dữ liệu. |
| 05 | [SPI/I²S](05_spi_i2s.md) | Đọc ID của SPI slave, xác nhận CPOL/CPHA và CS bằng analyzer. |
| 06 | [I²C/I3C](06_i2c_i3c.md) | Đọc cảm biến, giải thích ACK/NACK, timing và phục hồi bus. |
| 07 | [ADC/DAC](07_adc_dac.md) | Đo tín hiệu analog đúng VREF/sample time, tạo tín hiệu DAC. |
| 08 | [RTC, LPTIM và watchdog](08_rtc_lptim_watchdog.md) | Đánh thức từ low power và phục hồi sau watchdog reset. |
| 09 | [GPDMA](09_gpdma.md) | Chuyển mẫu ADC/UART không chiếm CPU, quản lý buffer/IRQ. |
| 10 | [USB FS và Type-C/PD](10_usb_ucpd.md) | Phân biệt USB data với PD, chạy và debug USB CDC. |
| 11 | [FDCAN](11_fdcan.md) | Nối transceiver và bus đúng, gửi/nhận, lọc ID, xử lý bus-off. |
| 12 | [OCTOSPI](12_octospi.md) | Đọc/ghi flash ngoài, hiểu indirect và memory-mapped mode. |
| 13 | [Flash, SRAM và cache](13_flash_cache_memory.md) | Lưu dữ liệu an toàn, hiểu ECC, bank, cache và option bytes. |
| 14 | [TrustZone và mật mã](14_security_crypto.md) | Phân vùng secure, dùng RNG/crypto và quản lý khóa đúng nguyên tắc. |
| 15 | [Debug và đo kiểm](15_debug_measurement.md) | Dùng SWD, DWT, analyzer để chứng minh nguyên nhân lỗi. |
| 16 | [CRC, DTS và HDMI-CEC](16_crc_dts_cec.md) | Học các khối tiện ích theo nhu cầu dự án. |

**Lộ trình tối thiểu:** 01 → 02 → 03 → 04 → 05/06 → 07 → 09. Các chương 08, 10–16 có thể chọn theo dự án. Debug ở chương 15 nên thực hành ngay từ chương 02.

## Cách học một chương

1. Đọc mục tiêu và vẽ lại đường đi **clock → pin → peripheral → IRQ/DMA → application**.
2. Mở đúng mục trong datasheet để xác nhận **H533CE có ngoại vi và chân cần dùng**; mở RM0481 để đọc trình tự/thanh ghi.
3. Chạy ví dụ với mức clock thấp và timeout hữu hạn; chụp thanh ghi status và dạng sóng.
4. Làm bài tập lỗi có chủ đích, ví dụ sai AF, sai baud, NACK hoặc DMA overrun; ghi lại bằng chứng.
5. Chỉ tăng tốc độ/tối ưu khi đã đo được bản đơn giản chạy đúng.

## Ba tài liệu gốc trong thư mục `doc/`

- [DS14539 Rev 3 — STM32H533xx datasheet](../doc/DS_stm32h533ce%20%281%29.pdf): bảng 2 trang **15–16** xác nhận tính năng từng mã; pinout trang **61–62** và alternate functions trang **82–95**; đặc tính điện từ trang **96**.
- [RM0481 Rev 5 — reference manual](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf): hoạt động và thanh ghi của các ngoại vi; **dùng chung nhiều biến thể H5**, nên một chương hoặc instance trong RM chưa chứng minh CE có phần cứng đó.
- [PM0264 — Cortex-M33 programming manual](../doc/pm0264-stm32-cortexm33-mcus-and-mpus-programming-manual-stmicroelectronics%20%281%29.pdf): exception, NVIC, SysTick, MPU và cơ chế của lõi CPU.

## Ranh giới phần cứng H533CE

Theo **DS14539 bảng 2**, CE có 35 GPIO, 2 ADC với 10 kênh, DAC 2 kênh, 3 USART + 2 UART + 1 LPUART, 4 SPI/3 I²S, 3 I²C, 2 I3C, 2 FDCAN, USB FS, UCPD và OCTOSPI. CE **không có** FMC, SDMMC, DCMI/PSSI hay VREFBUF. Một số khả năng của OCTOSPI controller không đưa đủ chân trên package 48 chân; kiểm tra bảng AF trước khi chọn bộ nhớ ngoài. Danh sách này là khả năng của chip, không bảo đảm board đã nối các chân hoặc có linh kiện ngoài cần thiết.

## Quy ước cho code minh họa

- Ví dụ dùng STM32Cube HAL hoặc CMSIS, **không trộn hai kiểu khởi tạo trên cùng peripheral** nếu không hiểu side effects.
- Tên `hxxx`, `Error_Handler`, cấu hình clock, startup và IRQ handler có thể đến từ project CubeMX; các file nêu rõ giả định cạnh code. Kiểm tra API theo phiên bản STM32CubeH5 đang dùng.
- `HAL_OK`/cờ hoàn tất driver không luôn đồng nghĩa dữ liệu đã tới ứng dụng ở đầu bên kia. Test bằng thiết bị ngoài, scope hoặc analyzer.
- Với pin, điện áp, timing, option bytes và bảo mật, datasheet/errata của đúng mã chip và revision trên board là nguồn quyết định cuối cùng.

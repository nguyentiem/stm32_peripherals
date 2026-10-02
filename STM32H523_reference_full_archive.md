# Giáo trình STM32H523 từ nền tảng đến thiết kế firmware chuyên nghiệp

## Bản mở rộng ngày 01 tháng 10 năm 2026

Giữ nội dung nền tảng 52 bài và bổ sung 36 bài về mode ngoại vi cùng các đặc tính H7 U5 L5. Tổng lịch là 20 chủ đề 88 bài 259 buổi. Phụ lục I có ma trận mode và nguồn mở rộng; Phụ lục J có sổ thực hành tính toán timing DMA cache và protocol. Đây là giáo trình học và quy trình lab, chưa phải bộ firmware đã được chạy trên mọi board hoặc bản diễn giải từng bit của toàn bộ reference manual.


**Đối tượng:** người bắt đầu học vi điều khiển, lập trình viên firmware cần hệ thống hóa kiến thức, kỹ sư muốn tiến tới tự thiết kế và kiểm chứng một sản phẩm.

**Bản biên soạn:** 01 tháng 10 năm 2026. Ngôn ngữ thực hành chính: C11. Đích chính: STM32H523; lấy STM32H523CC48 chân, 256 KiB flash làm cấu hình tối thiểu khi có thể. Các bài cần package lớn được đánh dấu riêng.

Giáo trình đi từ điện tử cơ bản, C và cách chương trình chạy trên CPU đến quản lý nguồn, clock, ngoại vi, thời gian thực, bảo vệ bộ nhớ và chuỗi khởi động tin cậy. Mục tiêu cuối cùng là có thể giải thích một thiết kế, triển khai nó, đo hành vi thực tế, tìm nguyên nhân lỗi và đưa ra bằng chứng rằng hệ thống đáp ứng yêu cầu. “Senior” ở đây là mức năng lực cần rèn luyện qua dự án, không phải chứng nhận đạt được chỉ bằng đọc hết sách.

Đây là tài liệu bài học và hướng dẫn thực nghiệm, không phải bản dịch từng thanh ghi của reference manual. Các thanh ghi trọng tâm được giải thích; bảng bit đầy đủ và giới hạn điện phải tra bản gốc. Các phần về kiến trúc phần mềm, toán học tín hiệu, quy trình kiểm thử và bài tập là phần biên soạn bổ sung. Không có kết quả đo phần cứng giả định được trình bày như kết quả đã thực hiện.

## Cách dùng bốn tài liệu gốc

| Ký hiệu | Tài liệu đã dùng | Vai trò |
|---|---|---|
| D1 | DS14540 Rev 3, tháng 3 năm 2026, STM32H523xx, 226 trang | Xác định tài nguyên có thật, package, chân và điều kiện điện |
| D2 | DS14539 Rev 3, tháng 3 năm 2026, STM32H533xx, 231 trang | So sánh phần mật mã; không dùng để gán tính năng H533 cho H523 |
| R | RM0481 Rev 5, tháng 7 năm 2026, 3766 trang | Nguyên lý khối ngoại vi, chuỗi thao tác và thanh ghi |
| P | PM0264 Rev 8, tháng 3 năm 2026 | Lõi Cortex-M33, instruction, exception, MPU, SAU và quyền CPU |

Thứ tự tra cứu: D1 để biết có khối đó không; R để biết điều khiển thế nào; P để hiểu phần thuộc CPU. Bổ sung errata ES0621 đúng silicon revision trước khi kết luận hành vi bất thường là lỗi chương trình. Số chương R trong sách chỉ áp dụng cho Rev 5. Hai chương có cùng tên trong R có thể thuộc hai nhóm chip khác nhau: H523 dùng OCTOSPI chương 24, RNG chương 36 và I3C chương 55.

## Ranh giới phần cứng phải nhớ

| Nội dung | STM32H523 thực tế | Ý nghĩa khi học |
|---|---|---|
| CPU | Cortex-M33, tối đa 250 MHz, TrustZone, MPU, FPU đơn chính xác | Một lõi; Secure và Non-secure không chạy đồng thời như hai CPU |
| Flash | H523xC 256 KiB; H523xE512 KiB | H523CC không có 512 KiB chỉ vì trang bìa ghi “up to 512” |
| SRAM | SRAM1 128 KiB, SRAM2 80 KiB, SRAM3 64 KiB; backup SRAM 2 KiB riêng | Lập linker và DMA theo từng vùng |
| Cache | ICACHE 8 KiB; DCACHE4 KiB cho bộ nhớ ngoài | Không áp dụng máy móc cách cache SRAM của Cortex-M7 |
| GPDMA | Hai controller, mỗi controller 8 channel | Channel không đồng nhất về FIFO và 2D |
| ADC và DAC | Hai ADC 12-bit; một DAC hai kênh 12-bit | Có thể học cả thu nhận lẫn phát tín hiệu |
| OPAMP và COMP | Không có OPAMP hay comparator COMP tích hợp được liệt kê cho H523 | Dùng linh kiện ngoài; không viết driver OPAMP nội trên H523 |
| Timer | TIM1/8, TIM2/3/4/5/12/15, TIM6/7, LPTIM1/2 | Không suy diễn TIM13/14/16/17 chỉ vì có trong R |
| Giao tiếp | USART/UART/LPUART, SPI/I2S, I2C, I3C, FDCAN, USB FS, UCPD, CEC | Số instance ra chân phụ thuộc package |
| Mật mã | RNG, HASH SHA-1/SHA-2; PKA hỗ trợ xác minh chữ ký ECDSA | Không gán AES, SAES, OTFDEC và mọi chế độ PKA của H533 cho H523 |
| Các khối khác | Không có Ethernet MAC, SAI riêng, CORDIC, FMAC, PLAY được liệt kê cho H523 | Học khái niệm/mở rộng bằng phần mềm hoặc linh kiện ngoài khi cần |
| Nguồn lõi | LDO; không có SMPS nội như một số H563/573 | Sơ đồ nguồn phải theo H523 |

Nguồn: D1 mục 2, bảng 2, mục 3 và sơ đồ khối; D2 mục 2–3; R các mục “applies only” đầu chương. Tên file DS_stm32h523cc.pdf chứa datasheet cho cả nhóm STM32H523xx.

### Package quyết định khả năng làm bài lab

| Tính năng | H523Hx 39 chân | H523Cx 48 chân | H523Rx 64 chân | H523Vx 100 chân | H523Zx 144 chân |
|---|---:|---:|---:|---:|---:|
| GPIO tối đa | 26 | 35 | 49 | 80 | 112 |
| SPI và I2S | 3 và 2 | 4 và 3 | 4 và 3 | 4 và 3 | 4 và 3 |
| I2C | 2 | 3 | 3 | 3 | 3 |
| USART ngoài hai UART và một LPUART | 2 | 3 | 4 | 4 | 4 |
| SDMMC và DCMI/PSSI | Không | Không | Có | Có | Có |
| FMC | Không | Không | Không | Có | Có |
| VREFBUF | Không | Không | Không | Có | Có |
| UCPD | Không | Có | Có | Có | Có |

“Có” không đồng nghĩa mọi tín hiệu đều dùng đồng thời được. Pin multiplexing, chức năng đang nối trên PCB và số chân cần thiết vẫn phải kiểm tra. H523 có FMC trên package phù hợp nhưng không có chức năng FMC SDRAM của một số chip khác trong R. [D1 bảng 2, mục 4; R mục 3.6.4]

## Lộ trình và phương pháp học

| Chặng | Bài | Kết quả phải tạo được | Thời lượng gợi ý |
|---|---|---|---|
| Nền tảng | 01–06 | Build, nạp, debug và giải thích startup, bộ nhớ, quyền CPU | 35–55 giờ |
| Điều khiển phần cứng | 07–16 | Clock đúng, I/O đúng điện, timer chính xác, watchdog có ý nghĩa | 55–85 giờ |
| Dữ liệu và analog | 17–22 | Thu/phát tín hiệu có DMA, có tính sai số và deadline | 55–85 giờ |
| Giao tiếp và lưu trữ | 23–35 | Protocol có phục hồi, lưu dữ liệu bền vững, hiểu USB/CAN/I3C | 90–140 giờ |
| Firmware hệ thống | 36–40 | Low power, RTOS, kiến trúc, chẩn đoán lỗi | 55–90 giờ |
| Bảo mật và sản phẩm | 41–48 | Cô lập, secure boot/update, kiểm thử và đưa vào sản xuất | 70–110 giờ |
| Dự án tổng hợp | 49–52 | Hồ sơ thiết kế, firmware và báo cáo nghiệm thu | 80–140 giờ |

Đây là ước lượng học tập, không phải cam kết thời gian. Có thể học 8–12 giờ/tuần, ưu tiên đo được một bài rồi mới chuyển bài. Các bài nâng cao cần quay lại làm khi thiết kế dự án.

Mỗi lab có mục tiêu đo được, cấu hình, trình tự thao tác, dấu hiệu đạt và lỗi thường gặp. Mã ghi “mẫu C” là đoạn để tích hợp sau khi tạo đúng cấu hình; mã ghi “giả mã” biểu diễn thuật toán, không phải HAL API có sẵn. Các ví dụ không phải bộ firmware đã build và chạy trên board của bạn. Pin LED, nút nhấn và cổng COM phải khớp sơ đồ board thực tế.

## Bàn thực hành và quy ước chung

Tối thiểu cần board STM32H523 có SWD, nguồn 3,3 V, ST-LINK hoặc debug probe tương thích, USB-UART mức logic 3,3 V, breadboard/dây, LED và điện trở 1 kΩ, nút nhấn, biến trở 10 kΩ. Oscilloscope và logic analyzer giúp kiểm chứng timing; bài dòng thấp cần đồng hồ đo dòng hoặc thiết bị đo năng lượng. FDCAN cần transceiver và bus có termination; analog nâng cao cần op-amp rail-to-rail phù hợp nguồn và tải; không cần mua tất cả để bắt đầu.

Chuẩn nền của các lab đầu: TrustZone tắt hoặc một cấu hình development đã biết; user code chạy privileged; HSI 64 MHz, bus và timer được kiểm tra thực tế; không tự thay option bytes chỉ để khớp sách. Nếu board đã bật TrustZone, dùng project Secure/Non-secure tương ứng và cấp ngoại vi cho đúng miền ngay từ đầu.

Ánh xạ gợi ý cho board H523C tự nối: PA5 cho LED ở bài đầu; PB0 cho LED khi PA5 chuyển sang SPI/DAC; PB1 cho nút active-low; PA9/PA10 cho USART1 khi AF phù hợp; PA0 là ADC12_INP0; PA4 là DAC1_OUT1; PA5 là DAC1_OUT2. Chỉ dùng từng cấu hình lab, không nối tất cả các chức năng đồng thời. SWDIO PA13, SWCLK PA14 giữ lại cho debug. Số chân vật lý phải tra D1 bảng 13; không lấy số chân LQFP48 áp sang UFQFPN hay BGA.

Trước mỗi lab tạo một nhánh Git, lưu file cấu hình pin/clock, phiên bản toolchain, log build, map file và ảnh đo. Lập bảng gồm: yêu cầu, phép thử, kết quả dự kiến, kết quả đo, đạt/không đạt và giải thích sai lệch. Đây là thói quen xuyên suốt từ bài đầu đến cấp senior.




# Lịch học theo chủ đề và buổi

Bản mở rộng gồm 20 chủ đề, 88 bài và 259 buổi gợi ý, mỗi buổi 3 giờ, tổng 777 giờ. Bài 01–52 là nền tảng H523; Bài 53–68 đào sâu mode; Bài 69–88 mở rộng H7 U5 L5. 108 buổi mới không bắt buộc thực hành trên mọi dòng cùng lúc: chọn board tương ứng, ghi rõ lab chưa đo nếu thiếu phần cứng. Thời gian tự học và phát triển dự án có thể nhiều hơn lịch này.

Một buổi lý thuyết hoàn thành bằng bảng mode và tính toán; một buổi lab bằng capture hoặc dữ liệu đo; một buổi phân tích bằng fault recovery report. Mỗi bài mới có ba buổi; các ngoại vi phức tạp có thể cần chia thêm buổi thực hành.

## Chủ đề 1 Nền tảng điện tử C và công cụ

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 1 | Bài 01: Điện áp dòng điện logic và RC | Tính mạch LED và nhận biết mức logic |
| 2 | Bài 01: Đấu nối đo nguồn và quan sát bounce | Sơ đồ nối cùng bảng điện áp và ảnh bounce |
| 3 | Bài 02: Kiểu dữ liệu con trỏ bit và register | Bài tập bitmask và phân loại register access |
| 4 | Bài 02: Buffer endian overflow và bài test C | Host test đọc số 16 bit và kiểm wrap thời gian |
| 5 | Bài 03: Toolchain CMSIS HAL LL và tạo project | Project đúng part cùng bảng phiên bản công cụ |
| 6 | Bài 03: Nạp debug và chạy blinky độc lập | ELF map file và trace LED |
| 7 | Bài 04: Register CPU stack privilege và exception | Bảng trạng thái Thread Handler Secure Non-secure |
| 8 | Bài 04: Quan sát exception và disassembly | Ảnh debugger giải thích stack frame và instruction |

## Chủ đề 2 Khởi động bộ nhớ nguồn và clock

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 9 | Bài 05: Reset boot startup và linker section | Sơ đồ khởi động và bảng section |
| 10 | Bài 05: Đọc map quan sát data bss và reset cause | Log reset và ngân sách flash RAM |
| 11 | Bài 06: Bus memory map RAMCFG và ECC | Bảng các master và vùng nhớ H523 |
| 12 | Bài 06: Đặt DMA buffer và kiểm alignment retention | Map file và phép kiểm pattern RAM |
| 13 | Bài 07: Power tree LDO reset BOR PVD và VOS | Sơ đồ nguồn đúng package cùng thứ tự đổi VOS |
| 14 | Bài 07: Đo nguồn reset và xây phản ứng sụt áp | Reset log và mô hình năng lượng dự trữ |
| 15 | Bài 08: Clock tree PLL bus và kernel clock | Bảng clock cho từng peripheral |
| 16 | Bài 08: MCO CRS CSS và chuyển clock có timeout | Ảnh đo clock cùng test fallback |

## Chủ đề 3 GPIO ngắt và timer điều khiển

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 17 | Bài 09: Cấu tạo GPIO và các mode | Bảng MODER OTYPER AFR BSRR cho lab |
| 18 | Bài 09: LED nút nhấn debounce không chặn | Code debounce và test nhấn giữ bounce |
| 19 | Bài 10: EXTI NVIC priority và pending | Luồng sự kiện từ chân đến handler |
| 20 | Bài 10: Đo latency và thiết kế ISR ngắn | Trace latency và thống kê sự kiện mất |
| 21 | Bài 11: Counter PSC ARR update và SysTick | Bài tính tần số cùng cấu hình timebase |
| 22 | Bài 11: Timer 1 kHz và đo microsecond | Scope marker và bảng thời gian thực thi |
| 23 | Bài 12: PWM output compare và preload | Tính duty độ phân giải và hành vi update |
| 24 | Bài 12: Đo PWM one pulse và các giá trị biên | Trace PWM 25 50 75 phần trăm |
| 25 | Bài 13: Capture đo period width và overflow | Thuật toán tính delta có wrap |
| 26 | Bài 13: Encoder quadrature và đo tốc độ | Bảng count mỗi vòng và test đổi chiều |
| 27 | Bài 14: TIM1 TIM8 dead time break và synchronization | Sơ đồ điều khiển và bảng trạng thái fault |
| 28 | Bài 14: Đo complementary output và trigger chain | Trace dead time break và kế hoạch trigger ADC |

## Chủ đề 4 Thời gian dài backup và watchdog

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 29 | Bài 15: LPTIM clock mode và wakeup | Bảng nguồn clock và power mode hợp lệ |
| 30 | Bài 15: RTC calendar alarm và calibration | Clock report và test qua nửa đêm |
| 31 | Bài 15: TAMP backup registers và backup SRAM | Lab pattern backup và chính sách tamper |
| 32 | Bài 16: IWDG WWDG và heartbeat theo tiến độ | Tính dải timeout và refresh window |
| 33 | Bài 16: Chèn lỗi và kiểm watchdog recovery | Log nguyên nhân reset cùng chính sách boot lại |

## Chủ đề 5 DMA cache và hệ thống analog

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 34 | Bài 17: GPDMA request trigger FIFO và direct transfer | Descriptor transfer và kiểm byte count |
| 35 | Bài 17: Linked list circular và 2D addressing | Sơ đồ node cùng lab hai buffer |
| 36 | Bài 17: Ownership deadline và recovery DMA | Overrun test và bảng lỗi DMA |
| 37 | Bài 18: ICACHE DCACHE và đặc điểm SRAM H523 | Sơ đồ đường truy cập cùng cache policy |
| 38 | Bài 18: Cache coherency và benchmark bộ nhớ ngoài | Cold warm timing và phép kiểm DMA buffer |
| 39 | Bài 19: ADC lượng tử sample hold và sai số | Ngân sách sai số ADC và thời gian conversion |
| 40 | Bài 19: Các mode calibration scan và watchdog analog | Bảng mode và cấu hình đo biến trở |
| 41 | Bài 19: Đo sai số theo trở kháng nguồn | Bảng mean noise offset và ảnh tín hiệu |
| 42 | Bài 20: Timer ADC DMA và chống alias | Sơ đồ pipeline và ngân sách deadline |
| 43 | Bài 20: Ping pong nhiều channel và kiểm overrun | Log sample count và sequence |
| 44 | Bài 20: VREFINT nhiệt analog VBAT và DTS | Bảng VDDA ước lượng và nhiệt độ die |
| 45 | Bài 21: DAC data alignment buffer trigger và reference | Bảng code điện áp cùng giới hạn tải |
| 46 | Bài 21: LUT timer DMA và phát dạng sóng | Trace sin và underrun test |
| 47 | Bài 21: Sample and hold VREFBUF và loopback ADC | So sánh mode và báo cáo loopback |
| 48 | Bài 22: OPAMP comparator và sai số mạch ngoài | Sơ đồ gain buffer cùng ngân sách sai số |
| 49 | Bài 22: Đo analog front end và đường fault ngoài | Trace gain bandwidth và comparator hysteresis |

## Chủ đề 6 Giao tiếp nối tiếp và thiết kế protocol

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 50 | Bài 23: UART USART LPUART và frame ký tự | Trace 0x55 và tính baud throughput |
| 51 | Bài 23: FIFO interrupt DMA IDLE và ring buffer | RX stress test và thống kê lỗi |
| 52 | Bài 23: RS485 DE turnaround và request response | Trace DE cùng bit cuối |
| 53 | Bài 23: LIN schedule PID data và checksum | Frame LIN có PID 92 checksum 6A |
| 54 | Bài 23: Smartcard IrDA và synchronous USART | Bảng character APDU và mode theo instance |
| 55 | Bài 24: Định nghĩa schema wire format và CRC | Protocol specification có offset và endian |
| 56 | Bài 24: Encoder parser state machine và timeout | Host codec cùng bộ test biên |
| 57 | Bài 24: ACK retry duplicate backpressure và fuzz | Test split coalesced corrupted và overload |
| 58 | Bài 25: I2C electrical timing frame và address | Trace write read repeated START |
| 59 | Bài 25: Driver transaction timeout và bus recovery | NACK stuck bus và recovery report |
| 60 | Bài 25: SMBus PEC PMBus và mô hình command | Schema command và test vector PEC theo thiết bị |
| 61 | Bài 26: I3C SDR T bit và địa chỉ động | Bảng PID BCR DCR dynamic address |
| 62 | Bài 26: Private transfer CCC DAA và IBI | Trace SDR cùng log discovery |
| 63 | Bài 26: Mixed bus hot join và recovery | Bảng khả năng target và test reset target |
| 64 | Bài 27: SPI CPOL CPHA NSS và transaction | Trace bốn mode cùng command layout |
| 65 | Bài 27: FIFO DMA packing và lifetime | Test độ dài biên và callback ownership |
| 66 | Bài 27: Timing nhiều device và lỗi giao dịch | Bus profile từng device và trace CS cuối |
| 67 | Bài 28: I2S frame slot PCM và sample format | Bảng BCLK WS MCLK và DMA layout |
| 68 | Bài 28: Audio DMA và xử lý underrun | Sin 1 kHz cùng deadline đo được |

## Chủ đề 7 CAN USB và quản lý nguồn Type C

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 69 | Bài 29: CAN điện arbitration và Classical frame | Sơ đồ bus và trace ID 11 ID 29 |
| 70 | Bài 29: CAN FD DLC bit timing và BRS | Bảng DLC cùng bài tính nominal data phase |
| 71 | Bài 29: FDCAN filter FIFO message RAM và hai node | Test filter sequence và ACK |
| 72 | Bài 29: Bus off overload và giao thức tầng trên | Recovery report và lựa chọn profile ứng dụng |
| 73 | Bài 30: USB role packet endpoint và transfer type | Sơ đồ transaction cùng bảng endpoint |
| 74 | Bài 30: Enumeration SETUP và descriptors | Capture enumeration và decode descriptor |
| 75 | Bài 30: USB CDC packet ứng dụng và backpressure | Test 63 64 65 128 byte cùng throughput |
| 76 | Bài 30: Host suspend resume disconnect và recovery | Ma trận trạng thái và test reconnect |
| 77 | Bài 31: Type C CC role và nguồn ngoài | Power role policy và pin schematic |
| 78 | Bài 31: PD frame header PDO RDO và state machine | Decode một phiên negotiation |
| 79 | Bài 31: Sink 5 V timeout reset và attach detach | PD log cùng kiểm giới hạn nguồn |

## Chủ đề 8 Lưu trữ bộ nhớ ngoài và luồng song song

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 80 | Bài 32: Flash bank sector program ECC OTP | Bảng vùng thử và đơn vị erase program |
| 81 | Bài 32: Journal wear leveling và schema bền vững | Codec record và thuật toán chọn generation |
| 82 | Bài 32: Reset power cut và garbage collection | Ma trận recovery sau mọi bước ghi |
| 83 | Bài 33: OCTOSPI command indirect và auto polling | Bảng lệnh serial memory |
| 84 | Bài 33: Memory mapped DTR DQS và DLYB | Pattern test và cửa sampling |
| 85 | Bài 33: FMC parallel memory và LCD bus cycle | Timing address data strobes và byte lanes |
| 86 | Bài 33: Cache DMA XIP và lỗi bộ nhớ ngoài | Benchmark cùng timeout recovery |
| 87 | Bài 34: SDMMC command response và card init | Capture command cùng capacity addressing |
| 88 | Bài 34: Block driver filesystem và buffering | Record file kiểm CRC sequence |
| 89 | Bài 34: Latency mất nguồn và card removal | Histogram write latency và recovery report |
| 90 | Bài 35: DCMI frame line pixel và memory budget | Tính framebuffer và schema ảnh |
| 91 | Bài 35: Camera capture DMA và overrun | Một frame đã giải mã màu stride đúng |
| 92 | Bài 35: PSSI word packing clock DE và RDY | Spec luồng song song |
| 93 | Bài 35: PSSI flow control và lỗi đường truyền | Counter pattern và test stall |
| 94 | Bài 35: CEC header opcode EOM ACK và địa chỉ | Spec message và bảng byte |
| 95 | Bài 35: CEC polling arbitration và wakeup | Trace message và error recovery |

## Chủ đề 9 Low power RTOS kiến trúc và debug

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 96 | Bài 36: Sleep Stop Standby retention và wakeup | Bảng mode theo yêu cầu ứng dụng |
| 97 | Bài 36: Đo dòng năng lượng và latency | Đường dòng cùng năng lượng mỗi chu kỳ |
| 98 | Bài 36: Race trước ngủ và khôi phục clock | Test 1000 chu kỳ cùng dữ liệu giữ lại |
| 99 | Bài 37: Race volatile atomic và memory ordering | Bảng invariant và lựa chọn primitive |
| 100 | Bài 37: SPSC queue và snapshot nhất quán | Host test wrap full empty cùng stress ISR |
| 101 | Bài 38: RTOS task priority stack và primitive | Task table và ngân sách bộ nhớ |
| 102 | Bài 38: Response time inversion mutex và deadline | Phép tính response time và đo blocking |
| 103 | Bài 38: Stress scheduler và supervisor | Deadline report cùng queue stack stats |
| 104 | Bài 39: Phân lớp BSP driver service app | Dependency và ownership table |
| 105 | Bài 39: Một image hai image và ranh giới API | So sánh kiến trúc cùng request snapshot schema |
| 106 | Bài 39: Fault containment và recovery app | Ma trận app fault đối chiếu control timing |
| 107 | Bài 40: Debug SWD trace DWT và phép đo | Kế hoạch quan sát triệu chứng |
| 108 | Bài 40: Fault frame CFSR HFSR và crash log | Fault record giải mã bằng đúng ELF |
| 109 | Bài 40: Tối ưu benchmark và kiểm hồi quy | Báo cáo trước sau có correctness tests |

## Chủ đề 10 Cô lập bộ nhớ và ngoại vi

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 110 | Bài 41: Privilege MPU base limit MAIR và XN | Memory permission map không overlap |
| 111 | Bài 41: SVC ABI bad pointer và validation | Service contract cùng negative tests |
| 112 | Bài 41: Cô lập app với MPU và fault report | CPU allowed denied matrix |
| 113 | Bài 42: TrustZone SAU IDAU NSC và context | Secure Non-secure memory map |
| 114 | Bài 42: Build hai image và gọi gateway | Artifact version và service chạy đúng |
| 115 | Bài 42: TOCTOU bad entry và fault isolation | Boundary test cùng recovery policy |
| 116 | Bài 43: GTZC SBS RCC GPIO IRQ và ownership | Peripheral security matrix |
| 117 | Bài 43: DMA source destination link và bus access | CPU DMA allowed denied test |
| 118 | Bài 43: Availability bus contention và budget | Control jitter khi NS tạo tải |

## Chủ đề 11 Mật mã boot bảo mật kiểm thử và sản xuất

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 119 | Bài 44: CRC hardware và chuẩn hóa test vector | Known-answer CRC nhiều độ dài |
| 120 | Bài 44: RNG HASH stream và health status | Hash vectors và error handling |
| 121 | Bài 44: PKA ECDSA verification đúng H523 | Valid invalid signature report |
| 122 | Bài 44: Khóa UID entropy và trust anchor | Key lifecycle model và kế hoạch lưu trữ |
| 123 | Bài 45: Secure boot trust chain và image manifest | Manifest byte specification |
| 124 | Bài 45: Update staging A B trial confirm rollback | State machine và memory budget |
| 125 | Bài 45: Ký xác minh version và chống rollback | Bộ negative tests image |
| 126 | Bài 45: Power cut update và recovery boot | Ma trận reset giữa từng bước |
| 127 | Bài 46: PRODUCT_STATE HDP và debug policy | Lifecycle state machine có hậu quả dữ liệu |
| 128 | Bài 46: Provisioning SFI và quản lý credential | Checklist provisioning mô phỏng |
| 129 | Bài 46: Authenticated recovery và hậu mãi | Kế hoạch service không cần thử Locked |
| 130 | Bài 47: Unit integration HIL và traceability | Test plan gắn yêu cầu |
| 131 | Bài 47: CI fuzzing static analysis và memory gate | Pipeline host tests cùng map budget |
| 132 | Bài 47: Stress fault injection và release gate | Báo cáo pass fail và giới hạn chưa kiểm |
| 133 | Bài 48: Pin package power PCB và signal integrity | Pin allocation và schematic review |
| 134 | Bài 48: Bring up điện clock debug ngoại vi | Trình tự bring up có điểm đo |
| 135 | Bài 48: EOL calibration UID và manufacturing data | Bộ hồ sơ bàn giao sản xuất |

## Chủ đề 12 Dự án tổng hợp và đánh giá cuối khóa

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 136 | Bài 49: Thiết kế hệ thu phát analog | Clock bandwidth memory và frame plan |
| 137 | Bài 49: Tích hợp timer ADC DAC GPDMA | Waveform và sample count |
| 138 | Bài 49: Telemetry command watchdog và overload | Parser stress cùng deadline report |
| 139 | Bài 49: Nghiệm thu dự án analog | Source trace và báo cáo sai số |
| 140 | Bài 50: Định nghĩa control service và fault model | API state machine và timing contract |
| 141 | Bài 50: Cài ownership MPU TrustZone và GTZC | Access control được thử |
| 142 | Bài 50: Gây lỗi app CPU DMA và flood | Control timing và output fault trace |
| 143 | Bài 50: Nghiệm thu cô lập và giới hạn | Báo cáo kiến trúc cùng rủi ro còn lại |
| 144 | Bài 51: Thiết kế record và pipeline gateway | Schema cùng ngân sách bus RAM |
| 145 | Bài 51: Tích hợp sensor CAN ADC storage USB | Log sequence từ các nguồn |
| 146 | Bài 51: Storage stall reconnect và power cut | Recovery và loss window đo được |
| 147 | Bài 51: Nghiệm thu datalogger | Host decoder và system test report |
| 148 | Bài 52: Lập specification và release baseline | Requirement matrix cùng memory budget |
| 149 | Bài 52: Tích hợp signed update và rollback | V1 V2 manifest và negative tests |
| 150 | Bài 52: System fault injection và isolation gate | Bằng chứng các gate bắt buộc |
| 151 | Bài 52: Bảo vệ thiết kế và bàn giao cuối khóa | Hồ sơ source build test production hoàn chỉnh |

## Chủ đề 13 Nguồn clock và đường tín hiệu

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 152 | Bài 53: Phân tích nguồn reset và chuyển trạng thái hoạt động — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 153 | Bài 53: Phân tích nguồn reset và chuyển trạng thái hoạt động — Cấu hình và thực hành đo kiểm | Sơ đồ nguồn và bảng retention reset |
| 154 | Bài 53: Phân tích nguồn reset và chuyển trạng thái hoạt động — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 155 | Bài 54: Clock mux PLL CRS và thay đổi clock khi đang chạy — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 156 | Bài 54: Clock mux PLL CRS và thay đổi clock khi đang chạy — Cấu hình và thực hành đo kiểm | Cây clock và đo MCO baud |
| 157 | Bài 54: Clock mux PLL CRS và thay đổi clock khi đang chạy — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 158 | Bài 55: GPIO EXTI và định tuyến tín hiệu nội bộ — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 159 | Bài 55: GPIO EXTI và định tuyến tín hiệu nội bộ — Cấu hình và thực hành đo kiểm | Pin map và capture cạnh GPIO EXTI |
| 160 | Bài 55: GPIO EXTI và định tuyến tín hiệu nội bộ — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |

## Chủ đề 14 Timer từ counter đến điều khiển động cơ

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 161 | Bài 56: Counter đếm tăng đếm giảm và center aligned — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 162 | Bài 56: Counter đếm tăng đếm giảm và center aligned — Cấu hình và thực hành đo kiểm | Waveform up down center và số compare update |
| 163 | Bài 56: Counter đếm tăng đếm giảm và center aligned — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 164 | Bài 57: Output compare PWM và tạo xung theo sự kiện — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 165 | Bài 57: Output compare PWM và tạo xung theo sự kiện — Cấu hình và thực hành đo kiểm | Waveform từng output mode và endpoint duty |
| 166 | Bài 57: Output compare PWM và tạo xung theo sự kiện — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 167 | Bài 58: Input capture PWM input encoder và clock ngoài — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 168 | Bài 58: Input capture PWM input encoder và clock ngoài — Cấu hình và thực hành đo kiểm | Capture PWM encoder và sai số tốc độ |
| 169 | Bài 58: Input capture PWM input encoder và clock ngoài — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 170 | Bài 59: Đồng bộ timer dead time break và ADC cho động cơ — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 171 | Bài 59: Đồng bộ timer dead time break và ADC cho động cơ — Cấu hình và thực hành đo kiểm | PWM complementary break và cửa ADC |
| 172 | Bài 59: Đồng bộ timer dead time break và ADC cho động cơ — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 173 | Bài 60: LPTIM RTC TAMP và watchdog theo từng chế độ — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 174 | Bài 60: LPTIM RTC TAMP và watchdog theo từng chế độ — Cấu hình và thực hành đo kiểm | Log RTC wakeup và ba thử nghiệm watchdog |
| 175 | Bài 60: LPTIM RTC TAMP và watchdog theo từng chế độ — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |

## Chủ đề 15 DMA và analog theo từng cơ chế

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 176 | Bài 61: GPDMA normal circular burst FIFO request và trigger — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 177 | Bài 61: GPDMA normal circular burst FIFO request và trigger — Cấu hình và thực hành đo kiểm | Bảng FIFO burst request trigger và loss counters |
| 178 | Bài 61: GPDMA normal circular burst FIFO request và trigger — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 179 | Bài 62: Linked list truyền 2D và thiết kế buffer DMA — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 180 | Bài 62: Linked list truyền 2D và thiết kế buffer DMA — Cấu hình và thực hành đo kiểm | Node list ownership và 2D sentinel |
| 181 | Bài 62: Linked list truyền 2D và thiết kế buffer DMA — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 182 | Bài 63: ADC theo nhóm chuyển đổi trigger và oversampling — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 183 | Bài 63: ADC theo nhóm chuyển đổi trigger và oversampling — Cấu hình và thực hành đo kiểm | Timing ADC rank mapping và histogram |
| 184 | Bài 63: ADC theo nhóm chuyển đổi trigger và oversampling — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 185 | Bài 64: DAC VREFBUF và đo tín hiệu nội — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 186 | Bài 64: DAC VREFBUF và đo tín hiệu nội — Cấu hình và thực hành đo kiểm | Đo DAC buffer trigger underrun và calibration |
| 187 | Bài 64: DAC VREFBUF và đo tín hiệu nội — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |

## Chủ đề 16 Giao tiếp lưu trữ và bảo mật theo mode

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 188 | Bài 65: USART UART SPI I2S theo cấu hình và kết thúc giao dịch — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 189 | Bài 65: USART UART SPI I2S theo cấu hình và kết thúc giao dịch — Cấu hình và thực hành đo kiểm | Capture UART SPI I2S và parser negative tests |
| 190 | Bài 65: USART UART SPI I2S theo cấu hình và kết thúc giao dịch — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 191 | Bài 66: I2C SMBus PMBus và I3C theo transaction — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 192 | Bài 66: I2C SMBus PMBus và I3C theo transaction — Cấu hình và thực hành đo kiểm | Trace I2C PEC DAA CCC và IBI |
| 193 | Bài 66: I2C SMBus PMBus và I3C theo transaction — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 194 | Bài 67: FDCAN USB UCPD và các giao tiếp song song — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 195 | Bài 67: FDCAN USB UCPD và các giao tiếp song song — Cấu hình và thực hành đo kiểm | Frame CAN USB PD camera và fault report |
| 196 | Bài 67: FDCAN USB UCPD và các giao tiếp song song — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 197 | Bài 68: Flash bộ nhớ ngoài cache và khối bảo mật — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 198 | Bài 68: Flash bộ nhớ ngoài cache và khối bảo mật — Cấu hình và thực hành đo kiểm | Storage power loss và crypto access negative tests |
| 199 | Bài 68: Flash bộ nhớ ngoài cache và khối bảo mật — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |

## Chủ đề 17 Kiến trúc hiệu năng và truyền dữ liệu STM32H7

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 200 | Bài 69: Cortex M7 TCM AXI SRAM cache và MPU — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 201 | Bài 69: Cortex M7 TCM AXI SRAM cache và MPU — Cấu hình và thực hành đo kiểm | Map TCM SRAM và tái hiện sửa cache DMA |
| 202 | Bài 69: Cortex M7 TCM AXI SRAM cache và MPU — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 203 | Bài 70: H7 hai lõi HSEM IPC và cách ly điều khiển — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 204 | Bài 70: H7 hai lõi HSEM IPC và cách ly điều khiển — Cấu hình và thực hành đo kiểm | Hai image IPC ring generation và app crash |
| 205 | Bài 70: H7 hai lõi HSEM IPC và cách ly điều khiển — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 206 | Bài 71: DMA1 DMA2 BDMA MDMA và DMAMUX trên H7 cổ điển — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 207 | Bài 71: DMA1 DMA2 BDMA MDMA và DMAMUX trên H7 cổ điển — Cấu hình và thực hành đo kiểm | DMA RAM matrix và đo burst FIFO MDMA |
| 208 | Bài 71: DMA1 DMA2 BDMA MDMA và DMAMUX trên H7 cổ điển — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 209 | Bài 72: FMC SDRAM và băng thông hệ thống — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 210 | Bài 72: FMC SDRAM và băng thông hệ thống — Cấu hình và thực hành đo kiểm | SDRAM timing worksheet và memory stress |
| 211 | Bài 72: FMC SDRAM và băng thông hệ thống — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |

## Chủ đề 18 Giao tiếp và multimedia chưa có trên H523

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 212 | Bài 73: Ethernet MAC PHY RMII MII và descriptor DMA — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 213 | Bài 73: Ethernet MAC PHY RMII MII và descriptor DMA — Cấu hình và thực hành đo kiểm | Packet capture và Ethernet descriptor ownership |
| 214 | Bài 73: Ethernet MAC PHY RMII MII và descriptor DMA — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 215 | Bài 74: USB high speed ULPI và PHY tích hợp — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 216 | Bài 74: USB high speed ULPI và PHY tích hợp — Cấu hình và thực hành đo kiểm | USB negotiated speed throughput và disconnect |
| 217 | Bài 74: USB high speed ULPI và PHY tích hợp — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 218 | Bài 75: SAI SPDIFRX và stream audio — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 219 | Bài 75: SAI SPDIFRX và stream audio — Cấu hình và thực hành đo kiểm | Audio frame slot và underrun clock loss |
| 220 | Bài 75: SAI SPDIFRX và stream audio — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 221 | Bài 76: DFSDM MDF ADF và phát hiện hoạt động âm thanh — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 222 | Bài 76: DFSDM MDF ADF và phát hiện hoạt động âm thanh — Cấu hình và thực hành đo kiểm | PCM filter delay và activity wake statistics |
| 223 | Bài 76: DFSDM MDF ADF và phát hiện hoạt động âm thanh — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 224 | Bài 77: OPAMP COMP ADC nâng cao và DAC sample and hold — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 225 | Bài 77: OPAMP COMP ADC nâng cao và DAC sample and hold — Cấu hình và thực hành đo kiểm | Analog gain offset hysteresis và S H droop |
| 226 | Bài 77: OPAMP COMP ADC nâng cao và DAC sample and hold — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |

## Chủ đề 19 Đồ họa điều khiển nhanh và giao tiếp chuyên dụng

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 227 | Bài 78: LTDC DMA2D và pipeline framebuffer — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 228 | Bài 78: LTDC DMA2D và pipeline framebuffer — Cấu hình và thực hành đo kiểm | Framebuffer budget và load tearing tests |
| 229 | Bài 78: LTDC DMA2D và pipeline framebuffer — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 230 | Bài 79: MIPI DSI GFXMMU NeoChrom và JPEG — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 231 | Bài 79: MIPI DSI GFXMMU NeoChrom và JPEG — Cấu hình và thực hành đo kiểm | DSI packet assets validation và GPU pipeline |
| 232 | Bài 79: MIPI DSI GFXMMU NeoChrom và JPEG — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 233 | Bài 80: HRTIM chuyển đổi công suất và định thời độ phân giải cao — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 234 | Bài 80: HRTIM chuyển đổi công suất và định thời độ phân giải cao — Cấu hình và thực hành đo kiểm | HRTIM event map dead time và fault latency |
| 235 | Bài 80: HRTIM chuyển đổi công suất và định thời độ phân giải cao — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 236 | Bài 81: SWPMI MDIO slave và time triggered CAN — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 237 | Bài 81: SWPMI MDIO slave và time triggered CAN — Cấu hình và thực hành đo kiểm | MDIO SWP TT CAN schema và capability report |
| 238 | Bài 81: SWPMI MDIO slave và time triggered CAN — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |

## Chủ đề 20 Ultra low power tính toán và security U5 L5

| Buổi | Bài và nội dung | Sản phẩm cần có |
|---:|---|---|
| 239 | Bài 82: SMPS MSI retention LPBAM và batch acquisition — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 240 | Bài 82: SMPS MSI retention LPBAM và batch acquisition — Cấu hình và thực hành đo kiểm | LPBAM BAM retention và energy budget |
| 241 | Bài 82: SMPS MSI retention LPBAM và batch acquisition — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 242 | Bài 83: CORDIC FMAC và tính toán fixed point — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 243 | Bài 83: CORDIC FMAC và tính toán fixed point — Cấu hình và thực hành đo kiểm | CORDIC FMAC error latency và saturation tests |
| 244 | Bài 83: CORDIC FMAC và tính toán fixed point — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 245 | Bài 84: TSC cảm ứng điện dung và thiết kế điện cực — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 246 | Bài 84: TSC cảm ứng điện dung và thiết kế điện cực — Cấu hình và thực hành đo kiểm | TSC baseline threshold và noise environment |
| 247 | Bài 84: TSC cảm ứng điện dung và thiết kế điện cực — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 248 | Bài 85: AES SAES PKA OTFDEC và vòng đời khóa — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 249 | Bài 85: AES SAES PKA OTFDEC và vòng đời khóa — Cấu hình và thực hành đo kiểm | Crypto KAT nonce key lifecycle và negative tests |
| 250 | Bài 85: AES SAES PKA OTFDEC và vòng đời khóa — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 251 | Bài 86: H7R S bootflash XSPI HPDMA và bảo mật M7 — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 252 | Bài 86: H7R S bootflash XSPI HPDMA và bảo mật M7 — Cấu hình và thực hành đo kiểm | Bootflash XSPI memory margin và recovery |
| 253 | Bài 86: H7R S bootflash XSPI HPDMA và bảo mật M7 — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 254 | Bài 87: Port firmware giữa H523 H7 U5 và L5 — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 255 | Bài 87: Port firmware giữa H523 H7 U5 và L5 — Cấu hình và thực hành đo kiểm | Port capability build và functional parity |
| 256 | Bài 87: Port firmware giữa H523 H7 U5 và L5 — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |
| 257 | Bài 88: Dự án senior đa ngoại vi và hồ sơ nghiệm thu — Lý thuyết cấu tạo mode và giới hạn | Bảng mode timing routing và căn cứ datasheet |
| 258 | Bài 88: Dự án senior đa ngoại vi và hồ sơ nghiệm thu — Cấu hình và thực hành đo kiểm | Hồ sơ nghiệm thu một dự án senior |
| 259 | Bài 88: Dự án senior đa ngoại vi và hồ sơ nghiệm thu — Phân tích lỗi phục hồi và ứng dụng dự án | Báo cáo lỗi recovery và tiêu chí nghiệm thu của bài |


# Chủ đề 1 Nền tảng điện tử C và công cụ

Bài 01 đến Bài 04. Đọc lịch buổi học để chọn sản phẩm cần hoàn thành trước khi chuyển bài.


## Bài 01 Điện tử nền tảng cho vi điều khiển

### Mục tiêu và khái niệm

Điện áp là chênh lệch thế điện giữa hai điểm; dòng điện là tốc độ dịch chuyển điện tích. GPIO không tạo ra “số 1” trừu tượng: nó tạo hoặc nhận một mức điện áp tương đối với GND. Hai board không nối chung tham chiếu có thể đọc sai dù UART được đặt cùng baud rate. Điện trở hạn dòng, tụ tích trữ điện tích và transistor dùng tín hiệu nhỏ để điều khiển dòng tải lớn.

Ba quan hệ cần dùng ngay: I = V/R; công suất P = VI; thời hằng mạch RC bằng R×C. Với LED có sụt áp xấp xỉ 2 V, nguồn 3,3 V và điện trở 1 kΩ, dòng dự kiến khoảng 1,3 mA. Giá trị thực phụ thuộc LED và điện áp đầu ra GPIO dưới tải. Đây là mức thuận tiện cho bài học, không phải giới hạn dòng cho phép của chân.

**Digital** là cách diễn giải điện áp thành các miền logic. VIH/VIL quy định ngưỡng vào; VOH/VOL mô tả đầu ra ở điều kiện tải. Khoảng giữa các ngưỡng không được xem là logic ổn định. **Analog** giữ ý nghĩa liên tục của điện áp; ADC biến miền điện áp thành các khoảng mã số hữu hạn.

### Cấu tạo và nguyên lý liên quan đến MCU

Đường nguồn có điện cảm ký sinh; khi hàng nghìn transistor đổi trạng thái, nhu cầu dòng biến thiên nhanh. Tụ decoupling gần chân VDD cấp dòng tức thời và giảm dao động nguồn. Tụ bulk xử lý biến thiên chậm hơn. Một tụ lớn ở xa chip không thay thế hoàn toàn tụ nhỏ đặt đúng vị trí.

Tín hiệu logic đi trên dây cũng có thời gian lan truyền, điện dung và phản xạ. Xung có tần số lặp thấp nhưng sườn rất nhanh vẫn có thể gây ringing. Vì vậy chọn tốc độ cạnh GPIO vừa đủ và giữ đường trở về GND tốt có giá trị hơn việc luôn đặt “very high speed”.

Ngõ vào có thể có điện trở kéo nội yếu. Pull-up tạo trạng thái mặc định cao, pull-down tạo trạng thái thấp. Một ngõ vào để nổi dễ bắt nhiễu và tiêu thụ dòng bất thường. Khả năng 5 V tolerant là thuộc tính có điều kiện của từng chân, không cho phép cấp 5 V tùy ý vào chân analog hoặc khi MCU mất nguồn. Tra D1 mục 4 và 5 trước khi nối.

### Lab 01 LED và nút nhấn

1. Tắt nguồn; nối GPIO LED qua điện trở 1 kΩ tới anode, cathode tới GND. Nếu board đã có LED, đọc sơ đồ để biết active-high hay active-low.
2. Nối một nút giữa PB1 và GND. Dùng pull-up nội khi lập trình ở bài 09; có thể dùng điện trở kéo ngoài 10 kΩ để thử mạch trước.
3. Cấp nguồn giới hạn dòng phù hợp board, đo VDD–GND. Không đặt đồng hồ ở thang dòng rồi chập hai que lên nguồn.
4. Khi có firmware bài 03, đặt đầu ra cao/thấp, đo điện áp hai đầu điện trở và suy ra dòng. So sánh LED sáng với giá trị ODR.
5. Với oscilloscope, đo chân nút khi nhấn một lần. Lưu số lần chuyển mức trong vài mili giây đầu. Đây là bounce cơ học, chưa phải lỗi EXTI.

**Đạt:** dự đoán đúng chiều dòng, LED sáng đúng trạng thái, giải thích được vì sao nút nhấn một lần tạo nhiều cạnh. **Lỗi điển hình:** LED ngược, thiếu chung GND, lấy nguồn 5 V USB-UART cấp vào chân 3,3 V, nhầm thang đo dòng.

**Bài tập:** nếu dùng 330 Ω thay 1 kΩ thì dòng lý tưởng là bao nhiêu? Đáp án khoảng 3,94 mA với giả thiết LED 2 V. Trước khi dùng, vẫn kiểm tra tổng dòng từng port và toàn chip trong datasheet. Ứng dụng dự án: thiết kế đầu vào công tắc, đèn báo, mạch reset và chống nhiễu nguồn. [D1 mục 4, 5]


## Bài 02 C nền tảng và thao tác bit

### Kiểu dữ liệu và bộ nhớ

Một byte là đơn vị địa chỉ cơ bản; trên STM32 là 8 bit. `uint8_t`, `uint16_t`, `uint32_t` biểu diễn số nguyên không dấu có độ rộng xác định. Một biến có giá trị, địa chỉ và thời gian tồn tại. Con trỏ chứa địa chỉ; giải tham chiếu yêu cầu địa chỉ hợp lệ, đúng quyền và đúng cách truy cập.

Số hex giúp nhìn bit: 0x20 tương ứng bit 5. Mặt nạ `1u << n` bật bit n khi n nhỏ hơn độ rộng kiểu. Dùng `uint32_t` cho thanh ghi 32-bit; không dịch bởi số âm hoặc lớn hơn/equal độ rộng. Tràn số nguyên không dấu theo modulo, còn tràn số nguyên có dấu là hành vi không được ngôn ngữ C định nghĩa.

```c
#include <stdint.h>
#include <stdbool.h>

static uint32_t field_replace(uint32_t reg,
                              uint32_t mask,
                              uint32_t encoded_value)
{
    return (reg & ~mask) | (encoded_value & mask);
}

static bool elapsed_ms(uint32_t now,
                       uint32_t start,
                       uint32_t interval)
{
    return (uint32_t)(now - start) >= interval;
}
```

Hàm thay field dùng khi `encoded_value` đã dịch đúng vị trí. Biểu thức tính thời gian xử lý được một lần wrap nếu không bỏ lỡ thời gian quá một vòng bộ đếm; quy ước thực tế giữ deadline nhỏ hơn nửa miền đếm để tránh nhầm thứ tự thời gian.

### volatile không phải cơ chế đồng bộ hoàn chỉnh

Thanh ghi ngoại vi được ánh xạ vào không gian địa chỉ. Trình biên dịch phải thực sự phát lệnh đọc/ghi, nên định nghĩa CMSIS thường dùng qualifier tương đương `volatile`. Nhưng `volatile` không tạo mutex, không biến `count++` thành nguyên tử, không bảo đảm thứ tự mọi truy cập RAM và không thay thế cache maintenance.

`const` ngăn sửa qua biểu thức C đó; không tự cấu hình MPU và không ngăn DMA hoặc mã lỗi ghi vào địa chỉ RAM. `static` ở cấp file giới hạn tên trong translation unit; đây là đóng gói phần mềm, không phải rào chắn phần cứng.

Thanh ghi không giống biến RAM: bit có thể read-only, write-one-to-clear, write-zero-to-clear, self-clearing hoặc có tác dụng phụ khi đọc. Với W1C, dùng phép ghi đúng mask; một phép `reg |= mask` có thể đọc các cờ khác đang bằng 1 và vô tình xóa chúng. Mỗi loại phải theo mô tả access trong R.

### Lab 02 Thực hành C trên máy tính

1. Viết chương trình có buffer 16 byte và hàm parse một trường 16-bit little-endian từ hai byte bằng shift và OR.
2. Bật cảnh báo compiler, ví dụ `-Wall -Wextra -Wconversion`; xử lý từng cảnh báo bằng hiểu kiểu, không tắt hàng loạt.
3. Thử `elapsed_ms(5, 0xFFFFFFF0u, 20)`: khoảng cách modulo là 21 nên kết quả đúng.
4. Viết hai hàm bật bit: một đúng với unsigned, một cố tình dịch quá 31. Chạy sanitizer trên máy tính để thấy lỗi ngôn ngữ; không nạp phiên bản lỗi vào hệ thống điều khiển.
5. Viết phép kiểm tra biên `length <= capacity - offset` sau khi đã kiểm tra `offset <= capacity`. So sánh với `offset + length <= capacity`, vốn có thể bị tràn.

**Đạt:** đọc được địa chỉ/con trỏ, hiểu storage lifetime, không trả con trỏ đến biến local, biết vì sao ép một buffer byte sang `uint32_t*` có thể vi phạm alignment/aliasing. **Bài tập:** tại sao cờ volatile giúp vòng polling quan sát ISR nhưng vẫn có thể mất sự kiện? Vì cờ chỉ chứa trạng thái, nhiều sự kiện có thể bị gộp và phép read-clear có thể đua với ISR. [R chương 1; P mục 2–3]


## Bài 03 Công cụ và chương trình đầu tiên

### Chuỗi từ mã nguồn đến máy chạy

Preprocessor xử lý include/macro; compiler sinh mã máy hoặc assembly; assembler tạo object; linker ghép section, giải quyết symbol và đặt địa chỉ; programmer ghi image vào flash. ELF giữ symbol/debug metadata; BIN chỉ là dãy byte; HEX chứa bản ghi địa chỉ. Nạp BIN mà không có địa chỉ đúng có thể ghi sai vùng.

CMSIS mô tả lõi và register device. HAL cung cấp thao tác cấp cao có trạng thái; LL gần thanh ghi hơn; BSP mô tả phần nối trên board. CubeMX tạo cấu hình ban đầu nhưng không tự chứng minh timing, khả năng phục hồi, độ chính xác analog hay mức an toàn.

Dùng STM32CubeIDE/CubeMX và STM32CubeH5 có hỗ trợ đúng STM32H523. Lưu phiên bản thực tế thay vì luôn nâng lên bản mới giữa khóa học. Kho chính thức STM32CubeH5 dùng submodule; tải ZIP riêng có thể thiếu thành phần. Ví dụ của board H563/H533 chỉ là tài liệu tham khảo cho thuật toán, phải port pin, clock, bộ nhớ và tính năng sang H523. [SW]

### Lab 03 Tạo và debug blinky

1. Ghi đầy đủ mã in trên chip và tên board. Tạo project bằng MCU selector đúng mã đó, không chọn một board gần giống chỉ để có ví dụ.
2. Chọn debug Serial Wire. Giữ PA13/PA14. Đặt một GPIO đang trống làm `LED`, output push-pull, low speed, mặc định off.
3. Dùng HSI trước; kiểm tra system clock do cấu hình sinh ra. Chưa bật PLL 250 MHz hay thay option bytes.
4. Generate code, build; đọc phần memory usage. Kiểm tra linker không cho flash vượt 256 KiB nếu chip là H523CC.
5. Nối SWDIO, SWCLK, GND, VTref và NRST theo probe/board. VTref thường dùng để nhận mức điện áp mục tiêu; không mặc định nó là nguồn cấp.
6. Nạp, đặt breakpoint đầu `main`, single-step qua khởi tạo. Quan sát PC, SP, register GPIO và biến `HAL_GetTick()`.
7. Trong user-code region, toggle LED rồi delay 500 ms. Chu kỳ sáng-tắt hoàn chỉnh dự kiến 1 giây, không phải 500 ms.
8. Reset bằng nút và ngắt probe, xác nhận chương trình vẫn chạy độc lập. Lưu `.ioc`, ELF, map, log build và ảnh chân LED.

```c
/* Mẫu đặt trong vòng while(1), sau HAL và MX_GPIO_Init. */
HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
HAL_Delay(500);
```

`LED_GPIO_Port` và `LED_Pin` phải do project định nghĩa. Delay thuận tiện để học nhưng chặn luồng gọi; bài sau sẽ thay bằng deadline hoặc timer.

**Chẩn đoán:** không kết nối được thì kiểm nguồn/NRST/SWD trước; thử connect-under-reset khi cấu hình hợp lệ nhưng chương trình làm mất debug. LED không sáng dù chạy tới dòng toggle thì kiểm active level/pin/AF. Tick không tăng thì xem SysTick handler và masking interrupt. Không kết luận chip hỏng chỉ từ một thông báo programmer.

**Đạt:** build sạch, giải thích chức năng startup và linker, biết đọc map để tìm `.text`, `.data`, `.bss`. Bài tập: đặt delay 100 ms, đo 5 Hz ở chân LED; lý giải sai số do thời gian thực thi và clock. [D1 mục 3.42; R chương 66; SW]


## Bài 04 Lõi Cortex M33 và cơ chế thực thi

### CPU làm gì

CPU đọc instruction, giải mã và thao tác register/memory. R0–R12 là các register đa dụng; SP trỏ stack; LR giữ địa chỉ quay về hoặc EXC_RETURN; PC chỉ dòng lệnh; xPSR chứa trạng thái như các cờ và exception number. Kiến trúc Thumb dùng instruction 16 hoặc 32 bit. Không phải mọi lệnh mất cùng số cycle vì bộ nhớ, pipeline và bus có thể chờ.

Thread mode dùng cho mã ứng dụng. Handler mode dùng khi xử lý exception/interrupt và chạy privileged. Trong Thread mode có privileged/unprivileged. Khi bật TrustZone, Secure/Non-secure là một trục khác với privilege: Non-secure privileged vẫn không có quyền tùy ý sửa Secure memory.

MSP thường dùng cho handler/khởi động; PSP thường dùng cho task. Với TrustZone có các bank trạng thái tương ứng. M33 có stack limit registers hỗ trợ phát hiện một số dạng vượt stack; compiler/RTOS phải cấu hình đúng, và cơ chế này không thay tất cả biện pháp kiểm tra stack.

### Exception và ngữ cảnh

Khi nhận interrupt, phần cứng lưu frame cơ bản gồm R0–R3, R12, LR, PC, xPSR vào stack thích hợp, rồi chuyển tới vector handler. FPU có thể tạo frame mở rộng; alignment và lazy stacking làm phân tích frame phức tạp hơn. Handler kết thúc bằng exception return để khôi phục trạng thái, không phải một hàm C bình thường gọi `main` trở lại.

Tail-chaining giảm một số thao tác lưu/khôi phục khi chuyển giữa interrupt; late arrival cho phép xử lý ưu tiên cao đến muộn trong chuỗi nhận ngắt. Những tối ưu này không làm mọi ISR có latency cố định. Tranh chấp bus, flash miss, critical section và mã đang thực thi đều cần đo.

FPU hỗ trợ single precision. Dùng literal `1.0f` khi muốn float; `double` thường đắt hơn và không tự được tăng tốc như phép float đơn. ABI hard/soft-float phải nhất quán giữa thư viện và ứng dụng. DSP instruction có thể giúp MAC/saturation nhưng không có nghĩa H523 chứa FMAC/CORDIC riêng.

### Lab 04 Nhìn exception bằng debugger

1. Ở blinky, đặt breakpoint trong SysTick handler và trong vòng chính.
2. Khi dừng trong handler, xem IPSR/xPSR, MSP/PSP, LR. Ghi exception number và giá trị EXC_RETURN.
3. Xem disassembly của một hàm cộng số nguyên và một hàm float. Build với `-Og` rồi `-O2`, quan sát instruction thay đổi.
4. Đặt local array nhỏ trong hàm, nhìn stack trước/sau. Không dùng breakpoint làm phép đo timing vì nó thay đổi hoàn toàn thời gian thực.
5. Trả về chương trình chạy tự do; dùng GPIO đánh dấu ISR ở bài 10 để đo latency thật.

**Đạt:** phân biệt function call với exception entry, Thread/Handler với Secure/Non-secure và privilege. **Bài tập:** hai image Secure/Non-secure có chạy song song không? Không; chúng chia sẻ cùng CPU, bus và thời gian xử lý. Điều này quyết định thiết kế cô lập lỗi ở bài 39–43. [P mục 2, 3, 4; D1 mục 3.1]


# Chủ đề 2 Khởi động bộ nhớ nguồn và clock

Bài 05 đến Bài 08. Đọc lịch buổi học để chọn sản phẩm cần hoàn thành trước khi chuyển bài.


## Bài 05 Reset boot startup và linker

### Từ cấp nguồn đến main

Reset đưa hệ thống về trạng thái xác định theo từng miền reset. Sau lựa chọn boot, CPU đọc vector table để lấy stack ban đầu và Reset_Handler. Startup chuẩn thường gọi khởi tạo hệ thống, sao chép `.data` từ flash sang RAM, xóa `.bss`, chạy khởi tạo runtime rồi gọi `main`. Thứ tự chi tiết phải đọc startup file và runtime của toolchain đang dùng.

`.text` chứa mã, `.rodata` chứa dữ liệu chỉ đọc, `.data` là biến đã khởi tạo có bản nạp trong flash và bản chạy trong RAM, `.bss` là biến zero-initialized. `.noinit` là quy ước linker cho vùng không tự xóa; nó chỉ hữu ích khi phần cứng còn giữ RAM và chương trình kiểm tra tính hợp lệ của nội dung. Stack phục vụ lời gọi hàm/ngắt; heap phục vụ cấp phát động.

Linker script là hợp đồng đặt bộ nhớ. Hai file firmware đặt ở hai vùng flash không tự có cơ chế lập lịch hay cô lập. Muốn một image gọi image khác cần ABI, vector/entry, ownership, version và cơ chế bảo vệ rõ ràng. Trên một lõi, vẫn chỉ có một dòng instruction được thực thi tại một thời điểm.

### Chế độ boot

User flash chứa ứng dụng; system memory chứa mã ROM/system do ST cung cấp; các tùy chọn boot và security quyết định đường khởi động hợp lệ. Không áp bảng BOOT 0 của STM32F1 cho H523. H523 có BOOT_UBE, trạng thái sản phẩm và các ràng buộc bảo mật liên quan. Bootloader tích hợp hỗ trợ giao tiếp nào phải tra AN2606 đúng dòng và revision; việc chip có USB/CAN không chứng minh ROM bootloader hỗ trợ mọi giao tiếp đó.

Chuyển ứng dụng từ bootloader không chỉ là gọi con trỏ hàm: cần kiểm tra image, stack, vector, tắt nguồn interrupt không còn dùng, xử lý DMA đang chạy, thống nhất clock, đặt VTOR đúng miền và chuyển stack/trạng thái đúng quy tắc. Với TrustZone, boot Secure→Non-secure dùng cơ chế chuyển miền, không dùng nhánh bình thường.

### Lab 05 Quan sát các section

1. Tạo `static uint32_t count;`, `static uint32_t seed = 123;` và `static const uint32_t table[4] = {1,2,3,4};`.
2. Build, tìm ba symbol trong map file. Ghi run address và load address khi có.
3. Dừng tại Reset_Handler. Step qua đoạn copy/zero; quan sát RAM trước và sau. Không dùng giá trị RAM trước khởi tạo như dữ liệu hợp lệ.
4. Tạo mảng global lớn hơn ngân sách SRAM còn lại, build để quan sát linker báo overflow; hoàn nguyên sau thử nghiệm.
5. Tính flash dùng thực tế: mã + dữ liệu chỉ đọc + bản khởi tạo `.data` + metadata liên quan. RAM dùng: `.data` + `.bss` + stack + heap + vùng DMA/RTOS.
6. Ghi lý do reset sớm khi khởi động rồi mới clear cờ reset. Kiểm tra power reset, nút NRST, software reset; thấy rằng một số cờ có thể cùng tồn tại.

**Đạt:** giải thích vì sao thêm một mảng initialized có thể tăng cả RAM lẫn flash. **Bài tập:** `const` luôn nằm flash không? Không nhất thiết; vị trí phụ thuộc storage, compiler, linker và cách dùng. Muốn bảo vệ phải kiểm tra map và cơ chế phần cứng. [R chương 2, 4, 7; P mục 2]


## Bài 06 Bus bộ nhớ RAMCFG và tính toàn vẹn dữ liệu

### Memory mapped I O và bus

CPU và GPDMA đều là bus master. SRAM, flash và ngoại vi cung cấp các vùng địa chỉ để master truy cập. AHB phục vụ nhiều đường truyền tốc độ cao; APB nối các register ngoại vi qua bridge. Address decoder chọn slave theo địa chỉ. Một thao tác C trông đơn giản vẫn có thể phải chờ arbitration, bridge hoặc peripheral ready.

H523 có SRAM1 128 KiB, SRAM2 80 KiB và SRAM3 64 KiB. Ba khối cho phép phân bố dữ liệu và giảm tranh chấp khi topology cho phép. Chúng không tự tạo ba vùng bảo mật. Non-secure/Secure aliases có thể trỏ cùng tài nguyên vật lý với kiểm tra quyền khác nhau; không được cộng dung lượng hai alias thành RAM gấp đôi.

CPU có thể truy cập một vùng không có nghĩa mọi DMA/ngoại vi đều truy cập được theo cùng đường hoặc quyền. Trước khi đặt buffer, xác nhận địa chỉ, bus connectivity, quyền Secure/privilege, alignment và tình trạng nguồn/clock.

### RAMCFG và ECC

RAMCFG quản lý một số chức năng SRAM như kiểm tra lỗi, xóa SRAM và retention/power-down theo khối được triển khai. ECC là mã dư để phát hiện/sửa một số lỗi bit. Trên H523, SRAM2 và backup SRAM hỗ trợ ECC khi cấu hình option tương ứng; không suy diễn SRAM1/3 cũng có cùng cơ chế.

Với SECDED, một lỗi bit có thể được sửa, hai lỗi bit có thể được phát hiện trong phạm vi mã bảo vệ. Điều này không có nghĩa sửa được mọi lỗi đa bit hoặc mọi lỗi phần mềm. DMA ghi nhầm một giá trị hợp lệ sẽ có ECC hợp lệ; cần MPU/GTZC và kiểm tra nghiệp vụ để ngăn.

Khi bật ECC hay sử dụng vùng chưa khởi tạo, phải tuân thủ yêu cầu khởi tạo của phần cứng. Đọc dữ liệu có ECC chưa hợp lệ có thể tạo fault. Không tự xóa SRAM chứa stack/code đang sử dụng. Các option SRAM reset cũng phải đối chiếu errata và được thực hiện có kiểm soát.

### Alignment endianness và barrier

Little-endian nghĩa byte thấp của số được đặt tại địa chỉ thấp. Giao thức trên dây có thể dùng big-endian; hãy serialize từng trường, không gửi thẳng C struct vì padding và ABI có thể khác. Alignment giúp truy cập hợp lệ/hiệu quả và có thể là yêu cầu bắt buộc của DMA hoặc descriptor.

DMB sắp thứ tự memory access theo kiến trúc; DSB chờ hoàn tất các thao tác liên quan; ISB đồng bộ luồng instruction sau thay đổi trạng thái thực thi. Chúng không làm việc thay cache clean/invalidate và không biến hai thao tác thành nguyên tử.

### Lab 06 Lập bản đồ bộ nhớ

1. Từ R mục 2.3 lập bảng SRAM1/2/3, backup SRAM, user flash, register và memory ngoài. Ghi cả aliases cần dùng cho cấu hình TrustZone của project.
2. Tạo một section `.dma_buffer` đặt trong SRAM1 bằng linker. Dùng attribute compiler để đặt buffer vào section và alignment 4 byte cho mẫu half-word/word ban đầu.
3. Trong map và debugger, xác nhận địa chỉ thật nằm đúng vùng. Không chỉ tin tên section.
4. Chạy memcpy CPU giữa hai buffer 1 KiB, kiểm bằng pattern tăng dần và checksum. Bài 17 sẽ thay bằng DMA và đo thời gian.
5. Liệt kê vùng RAM nào cần giữ khi Stop, vùng nào có thể tắt; ước lượng dữ liệu khôi phục. Thử retention trên một vùng thử độc lập, không thay option byte ECC tùy tiện.

**Đạt:** giải thích được ranh giới giữa lỗi vật lý, lỗi truy cập trái phép và lỗi dữ liệu nghiệp vụ. **Bài tập:** bảo vệ CPU bằng MPU có chặn DMA không? MPU lõi không trực tiếp giám sát mọi master; cần quyền bus/GTZC và cấu hình DMA. [D1 mục 3.3–3.5, 3.15; R chương 2, 5, 6; P mục 2, 4]


## Bài 07 Nguồn reset và voltage scaling

### Các miền cấp nguồn

VDD cấp I/O và các khối nguồn số; LDO nội tạo VCORE cho logic. VCAP là nút liên quan đến nguồn lõi và tụ ổn định theo sơ đồ datasheet, không phải chân cấp tải ngoài. VDDA/VSSA phục vụ analog; VREF+ là tham chiếu chuyển đổi ở package có chân riêng. VDDUSB phục vụ USB transceiver khi có chân/sơ đồ tương ứng. VBAT giữ backup domain như RTC và backup registers/SRAM theo cấu hình. VDDIO 2 chỉ có ở package được liệt kê, không có trên H523C.

Tụ VCAP, decoupling VDD/VDDA, liên kết VREF và trình tự nguồn phải theo D1 mục 5.3, hình power supply scheme và package thực. Không sao chép sơ đồ H563 có SMPS vào H523. Miền không dùng vẫn có yêu cầu nối nguồn; không để nổi chỉ vì firmware chưa dùng ngoại vi.

### Giám sát điện áp và reset

POR/PDR xử lý power-on/power-down; BOR phát hiện tụt nguồn dưới ngưỡng cấu hình để reset; PVD tạo thông báo khi điện áp đi qua ngưỡng, cho phép phần mềm phản ứng nếu còn thời gian và năng lượng. AVD giám sát analog supply theo khả năng triển khai. Reset ngoại vi không giống system reset; reset backup domain có thể làm mất RTC/backup state.

PVD không bảo đảm ghi flash kịp khi rút nguồn. Phải tính năng lượng dự trữ và thời gian xấu nhất. Với tụ C từ V1 xuống V2, năng lượng khả dụng lý tưởng bằng C×(V1²−V2²)/2; năng lượng thực còn bị hao hụt và giới hạn regulator. Thiết kế journal không mất bản dữ liệu cũ thường đáng tin cậy hơn trông chờ một lần ghi khẩn cấp cuối cùng.

### Voltage scaling

| Thang | Giới hạn tần số nêu trong D1 | Điều cần kiểm tra |
|---|---:|---|
| VOS0 | 250 MHz | Nhiệt độ junction tối đa ở thang này và flash timing |
| VOS1 | 200 MHz | Điện áp, tần số bus và clock kernel |
| VOS2 | 150 MHz | Độ trễ flash tương ứng |
| VOS3 | 100 MHz | Năng lượng và thời gian hoàn thành công việc |

Đây là trần theo thang, không phải mọi tổ hợp nguồn/nhiệt độ/ngoại vi đều chạy ở trần. Khi tăng tần số: chuẩn bị VOS, đợi VOSRDY, tăng flash latency theo bảng, chuẩn bị bus prescaler, rồi chuyển clock và kiểm tra trạng thái. Khi giảm công suất: giảm clock trước, rồi hạ VOS và điều chỉnh latency phù hợp. Mã phải có timeout hữu hạn và phương án giữ nguồn clock an toàn.

### Lab 07 Reset có bằng chứng

1. Chạy project ổn định ở HSI; log reset cause ngay lúc boot trước khi clear.
2. Thử NRST, software reset và power cycle; lập bảng các cờ quan sát được.
3. Đọc cấu hình VOS và clock; đối chiếu D1. Không tăng clock trước khi làm bài 08.
4. Nếu có nguồn chỉnh áp và hiểu sơ đồ board, giảm từ từ trong dải cho phép để quan sát PVD/BOR. Ngắt tải động cơ và nguồn ngoài có thể back-power I/O.
5. Ghi hành vi mong đợi: firmware không tiếp tục điều khiển sai khi nguồn ngoài điều kiện; dữ liệu cấu hình trước đó còn hợp lệ sau reset.

**Đạt:** vẽ được các miền nguồn của chính board, biết giá trị reset cause được đọc/clear ở đâu và vì sao. **Bài tập:** vì sao giảm MHz không luôn giảm năng lượng mỗi tác vụ? Dòng nền kéo dài lâu hơn; phải đo tích phân công suất theo thời gian, không chỉ dòng tại một thời điểm. [D1 mục 3.10, 5.3; R chương 10, 11]


## Bài 08 RCC PLL clock kernel và CRS

### Cấu tạo cây clock

Nguồn dao động đi vào mux/prescaler/PLL; đầu ra cấp SYSCLK, HCLK, PCLK và các kernel clock. Bus clock cho phép đọc/ghi register; kernel clock vận hành khối chức năng. Một UART có thể truy cập register bình thường nhưng baud sai nếu kernel clock được chọn khác PCLK mà phần mềm tính nhầm.

| Nguồn | Giá trị danh định hoặc dải D1 | Công dụng |
|---|---|---|
| HSI | 64 MHz, có chia | Khởi động, clock nội chính |
| CSI | 4 MHz | Hoạt động với nguồn clock nội thấp hơn |
| HSI48 | 48 MHz | USB và các mục đích phù hợp, có thể hiệu chỉnh bằng CRS |
| LSI | Khoảng 32 kHz | Watchdog/clock thấp; sai số cần tính |
| HSE | 4–50 MHz theo điều kiện datasheet | Nguồn ngoài cho độ chính xác hoặc PLL |
| LSE | 32,768 kHz | RTC, đo thời gian dài |

HSE crystal mode dùng tinh thể và mạch dao động nội; bypass nhận clock ngoài. Hai chế độ không thể thay tùy tiện. Tụ tải tinh thể dựa trên CL của tinh thể và điện dung ký sinh; chọn “hai tụ bất kỳ” có thể làm khởi động kém hoặc sai tần số.

PLL dùng chia đầu vào M, nhân N và có chia đầu ra P/Q/R; khi hỗ trợ fractional thì phần nhân chứa phân số. Công thức lý tưởng: fVCO = fin/M × (N + FRACN/8192); fout = fVCO/divider. Mọi lựa chọn phải đồng thời nằm trong dải PLL input, VCO, output, VOS và peripheral. Không lấy một bộ M/N/P cho chip khác rồi chỉ sửa SystemCoreClock.

### Clock timer và CRS

Timer clock không luôn bằng PCLK. Ở chế độ TIMPRE=0, R quy định khi APB chia 1 hoặc 2 thì timer nhận HCLK; các chia lớn hơn dùng 2×PCLK. TIMPRE=1 có quy tắc khác, phải tra RCC_CFGR1 và kiểm tra trần timer. Không dùng câu “APB chia thì cứ nhân hai” cho mọi cấu hình.

CRS so tần số HSI48 với xung tham chiếu như USB SOF/LSE/ngoại vi theo mux cho phép, đo sai lệch và điều chỉnh trim. Crystal-less không có nghĩa clock tự chính xác ở mọi trạng thái. Khi mất nguồn đồng bộ, phần mềm phải biết chất lượng clock có còn đáp ứng giao tiếp không.

### Trình tự cấu hình và lỗi

Bật oscillator, đợi ready có timeout; cấu hình PLL khi điều kiện cho phép; bật PLL và chờ lock; chuẩn bị voltage/flash/bus; chuyển SYSCLK và xác nhận SWS; cập nhật biến clock của phần mềm và timebase. Clock Security System có thể phát hiện lỗi nguồn clock được hỗ trợ nhưng ứng dụng vẫn phải thiết kế phản ứng và khôi phục.

### Lab 08 Đo clock thật

1. Chạy ở HSI với HSIDIV đã xác nhận; đọc SystemCoreClock và các tần số mà HAL báo.
2. Chọn MCO trên chân được D1 cho phép, đặt prescaler đủ lớn để oscilloscope đo được và không vượt điện/tốc độ chân.
3. Đo MCO; đối chiếu giá trị tính toán. `SystemCoreClock` chỉ là biến phần mềm, không phải thiết bị đo clock.
4. Tạo cấu hình PLL bằng CubeMX cho một tần số hợp lệ thấp hơn trần. In bảng SYSCLK/HCLK/PCLK/timer/UART/ADC/USB trước và sau.
5. Kiểm tra baud UART và chu kỳ timer sau đổi clock. Nếu tick 1 ms sai, sửa timebase; không chữa bằng cách đổi hằng delay tùy ý.
6. Tạo tình huống HSE không khởi động bằng cấu hình thử trên phần cứng phù hợp; xác nhận timeout dẫn về chế độ chẩn đoán/clock dự phòng, không kẹt vô hạn.

**Đạt:** giải thích đường clock từ oscillator tới từng ngoại vi và đo được ít nhất hai nhánh. **Bài tập:** HCLK=64 MHz, APB=32 MHz, TIMPRE=0 thì timer là 64 MHz. Với PSC=63, một tick counter là 1 µs. [D1 mục 3.12–3.13, 5.3.7–5.3.9; R chương 7, 11, 12]


# Chủ đề 3 GPIO ngắt và timer điều khiển

Bài 09 đến Bài 14. Đọc lịch buổi học để chọn sản phẩm cần hoàn thành trước khi chuyển bài.


## Bài 09 GPIO từ mạch chân đến thanh ghi

### Cấu tạo một chân

Một GPIO gồm pad ngoài, bảo vệ điện, đường input, output driver, điện trở kéo và mux nối ngoại vi. MODER chọn input/output/alternate/analog; OTYPER chọn push-pull/open-drain; OSPEEDR chọn đặc tính cạnh; PUPDR chọn kéo lên/kéo xuống; AFR chọn alternate function; IDR đọc mức chân; ODR chứa latch đầu ra; BSRR cho phép set/reset bit bằng một lần ghi. LCKR có cơ chế khóa cấu hình theo trình tự được mô tả, không phải bảo mật dữ liệu tổng quát.

| Mode | Đường hoạt động chính | Ứng dụng |
|---|---|---|
| Input | Chân đi vào mạch nhận số | Nút, tín hiệu trạng thái |
| Output push-pull | Chủ động kéo cao hoặc thấp | LED, chip select |
| Output open-drain | Kéo thấp hoặc thả nổi | Bus chia sẻ, nối logic wired-AND |
| Alternate function | Ngoại vi điều khiển/nhận chân | UART, SPI, timer |
| Analog | Đường analog được dùng, input số bị vô hiệu theo cấu hình | ADC/DAC, giảm dòng chân không dùng |

Open-drain cần pull-up để tạo mức cao. Hai push-pull nối với nhau mà một bên cao, một bên thấp tạo xung đột dòng. Một output open-drain viết 1 chỉ nhả chân, không bảo đảm điện áp đã lên mức cao.

BSRR tránh read-modify-write trên cả port khi cần set/reset bit. Nó không biến chuỗi nhiều lần toggle thành phép nguyên tử; `ODR ^= mask` vẫn là đọc, XOR, ghi và có thể đua với ISR.

### Lab 09 Điều khiển không chặn

1. Dùng PB0 làm LED và PB1 làm input pull-up active-low. Bật clock GPIOB trước khi cấu hình.
2. Đặt mức đầu ra off trước khi chuyển chân sang output để hạn chế glitch.
3. Viết LED qua BSRR, đọc nút qua IDR. Giữ các bit khác bằng mask khi sửa register cấu hình.
4. Thay HAL_Delay bằng kiểm deadline để vừa nháy LED vừa đọc nút.
5. Debounce bằng yêu cầu trạng thái ứng viên ổn định 20 ms. 20 ms là giá trị thử nghiệm, cần điều chỉnh theo switch và yêu cầu latency.

```c
/* Mẫu C trong vòng chính, giả sử tick_ms tăng mỗi 1 ms. */
uint32_t now = HAL_GetTick();
bool raw = (HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_1)
            == GPIO_PIN_RESET);
if (raw != candidate) {
    candidate = raw;
    candidate_since = now;
}
if ((uint32_t)(now - candidate_since) >= 20u &&
    stable != candidate) {
    stable = candidate;
    if (stable) {
        press_count++;
    }
}
```

`candidate`, `stable`, `candidate_since`, `press_count` phải được khai báo với lifetime kéo dài qua các vòng lặp và khởi tạo từ mức chân. Nếu dùng local tự động khởi tạo lại mỗi vòng thì thuật toán không còn đúng.

**Đạt:** một lần nhấn hợp lệ tăng một count, giữ nút không tăng liên tục, LED vẫn chạy. Thử cả bounce bằng dây/nút và nhấn nhanh dưới ngưỡng. **Bài tập:** chuyển PB0 sang open-drain, tháo LED rồi đo với/không pull-up; giải thích khác biệt. [D1 mục 3.14, 4, 5.3.14; R chương 13]


## Bài 10 NVIC EXTI và thiết kế ISR

### Từ tín hiệu đến handler

EXTI phát hiện sự kiện từ GPIO hoặc nguồn nội, dùng các mask và cạnh cấu hình để tạo interrupt/event. GPIO cùng số bit có thể cạnh tranh đường EXTI tương ứng; cần chọn nguồn port trong cấu hình EXTI. NVIC nhận IRQ, kiểm enable, pending, priority và trạng thái CPU rồi quyết định preemption. Interrupt peripheral flag, EXTI pending và NVIC pending là các tầng khác nhau; clear nhầm tầng có thể làm ngắt lặp ngay.

Số ưu tiên nhỏ hơn thường biểu thị độ ưu tiên cao hơn trong CMSIS. Chỉ một số bit priority được triển khai; tra device header `__NVIC_PRIO_BITS`. Không truyền một giá trị đã shift vào API mà API sẽ tự shift lần nữa. Priority grouping phân chia preemption/subpriority theo khả năng; subpriority không cho phép tự preempt khi preemption priority bằng nhau.

PRIMASK che phần lớn configurable exceptions; BASEPRI đặt ngưỡng ưu tiên; FAULTMASK có phạm vi mạnh hơn. Giữ masking ngắn. `volatile` không thay thế critical section. Khi dùng RTOS, interrupt gọi API RTOS phải nằm trong miền priority mà port cho phép.

### ISR cần làm gì

ISR đọc trạng thái, xử lý nguồn gây ngắt theo thứ tự RM, ghi dữ liệu tối thiểu rồi chuyển công việc nặng cho main/task. Không chờ UART blocking, không format log dài, không cấp phát heap tùy tiện. Nếu nhiều sự kiện quan trọng, dùng counter/queue thay một cờ boolean. Nếu cần timestamp chính xác, dùng timer capture thay thời điểm CPU vào ISR.

“Event” có thể dùng đánh thức WFE mà không chạy handler tương ứng; “interrupt” dùng exception flow. Sự khác biệt này hữu ích khi học low power, nhưng cần clear trạng thái tồn đọng đúng cách để không thức ngay lập tức.

### Lab 10 Đo độ trễ ngắt

1. Cấu hình PB1 EXTI cạnh xuống, bật NVIC, dùng handler sinh bởi project làm điểm xuất phát.
2. Handler gọi đường xử lý HAL hoặc LL phù hợp; callback chỉ tăng counter và đặt một GPIO marker, rồi thoát. Không debounce bằng delay trong ISR.
3. Dùng một nguồn xung có mức 3,3 V vào PB1; oscilloscope kênh 1 đo cạnh vào, kênh 2 đo marker.
4. Đo latency tối thiểu/tối đa ở 1000 sự kiện. Bật thêm một interrupt ưu tiên cao với thời gian xử lý xác định, quan sát latency tăng.
5. Thử mask ngắt 10 µs trong đoạn thử; quan sát pending không đồng nghĩa mọi cạnh đã được đếm. EXTI không phải FIFO đếm xung.
6. Debounce ngoài ISR, hoặc dùng timer ghi nhận trạng thái ổn định. Đếm xung nhanh dùng timer external/capture.

**Đạt:** phân biệt latency với thời gian xử lý ISR, biết đo cả hai. **Lỗi điển hình:** pending không clear; callback sai tên cho phiên bản HAL; GPIO đang ở AF khác; priority quá cao làm RTOS assertion; debug breakpoint tạo ra hành vi không lặp khi chạy thật. [R chương 17–18; P mục 2 về exception và mục 4 NVIC]


## Bài 11 Timer cơ bản timebase và SysTick

### Khối đếm

Một timer thông thường có clock/prescaler, counter CNT, auto-reload ARR, compare/capture CCR, logic update và output/trigger. Prescaler PSC chia clock theo PSC+1. Trong đếm lên từ 0 tới ARR, một chu kỳ có ARR+1 tick. Update event nạp các thanh ghi shadow và có thể tạo interrupt, DMA hoặc TRGO.

Với edge-aligned upcounter và repetition không dùng: f_update = f_TIM / ((PSC+1)×(ARR+1)). Tại f_TIM=64 MHz, PSC=63 cho tick 1 MHz; ARR=999 cho update 1 kHz. Đây là tần số sự kiện. Nếu ISR toggle chân mỗi sự kiện, tần số sóng vuông chỉ là 500 Hz.

TIM6/TIM7 là basic timer thuận tiện làm timebase/TRGO, không có output PWM ở chân như TIM1/2. TIM2/TIM5 có counter 32-bit; phần lớn timer còn lại 16-bit. TIM15 không có đầy đủ kênh như TIM1. SysTick là bộ đếm 24-bit thuộc core; khi bật TrustZone có các bank liên quan, không nên coi là hai timer ứng dụng hoàn toàn tự do không phụ thuộc trạng thái bảo mật.

### Shadow register và trình tự start

PSC thường được nạp vào bộ chia hoạt động qua update event. ARR/CCR có thể preload để thay đổi đúng ranh giới chu kỳ. Nếu ghi PSC rồi start mà không tạo update khi cần, chu kỳ đầu có thể dùng giá trị cũ. UG có thể tạo tác dụng phụ như flag/update request/TRGO tùy cấu hình; phải chuẩn bị downstream trước hoặc ngăn trigger ngoài ý muốn.

Chuỗi lab: tắt counter; chọn clock; ghi PSC/ARR; chọn preload; tạo update nếu cần; clear flag theo access rule; đặt CNT; cấu hình interrupt/trigger; cuối cùng enable counter. Không clear cờ rc_w0/W1C bằng một mẫu bit chung cho mọi peripheral.

### Lab 11 Timer 1 kHz và bộ đo microsecond

1. Lấy f_TIM từ bài 08; không mặc định bằng core clock. Cấu hình TIM6 update 1 kHz.
2. Trong IRQ chỉ toggle marker và tăng tick. Đo marker 500 Hz.
3. Dùng TIM2 free-running, tick 1 MHz, ARR tối đa 32-bit. Đọc CNT trước/sau một hàm và tính hiệu modulo.
4. Chạy khi không có interrupt rồi khi bật UART/GPDMA interrupt. So sánh thời gian CPU quan sát với thời gian cố định phần cứng.
5. Đổi PSC và ARR nhưng giữ tích chia; xác nhận cùng tần số update nhưng độ phân giải của CCR/capture khác.

**Đạt:** sai số giải thích được từ clock và phép đo; không gọi delay trong timer IRQ. **Bài tập:** 64 MHz, PSC=6399, ARR=9999 tạo 1 Hz update; toggle tạo 0,5 Hz. Vì sao không dùng NOP làm đồng hồ? Tối ưu compiler, flash/bus và ngắt thay đổi thời gian quan sát; peripheral timer là cơ sở thời gian phù hợp hơn. [D1 mục 3.31; R chương 43–48; P mục 4 SysTick]


## Bài 12 PWM output compare và one pulse

### Nguyên lý PWM

PWM điều khiển tỷ lệ thời gian bật trong chu kỳ. Trong PWM mode 1 đếm lên, với polarity active-high và cấu hình thông thường, output active khi CNT < CCR. Duty lý tưởng trong miền hợp lệ là CCR/(ARR+1). Mode 2 đảo quan hệ logic của mode 1; polarity output có thể đảo thêm lần nữa. Không đồng nhất PWM mode với active level.

Output compare tạo sự kiện khi CNT bằng CCR và có thể toggle/set/reset chân. PWM dùng quan hệ compare liên tục để tạo duty. One-pulse phát một chuỗi theo điều kiện start/trigger rồi dừng, phù hợp tạo xung điều khiển có chiều rộng xác định.

Edge-aligned đếm một chiều; center-aligned đếm lên/xuống, giúp phân bố switching đối xứng và có lợi cho lấy mẫu dòng động cơ. Công thức chu kỳ phải theo dãy đếm của mode; không áp ARR+1 của upcounter vào mọi kiểu. PWM ở tần số càng cao thì số mức duty khả dụng với cùng timer clock càng giảm.

### Cấu hình và ứng dụng

Ví dụ f_TIM=64 MHz, PSC=0, ARR=3199 tạo PWM 20 kHz; CCR=800 cho 25%. Một servo dân dụng thường dùng độ rộng xung chứ không chỉ duty, nên cần đúng chu kỳ/biên độ theo thiết bị. LED dimming có cảm nhận độ sáng phi tuyến; có thể ánh xạ gamma trước khi ghi CCR. Điều khiển motor cần power stage và xử lý fault riêng, không nối tải trực tiếp vào GPIO.

Bật CCR preload giúp thay duty vào ranh giới chu kỳ thay vì đổi ngay giữa xung. ARR preload giúp đổi period đồng bộ. Muốn cập nhật nhiều kênh cùng lúc, nghiên cứu update/COM/DMA burst đúng timer thay vì ghi tuần tự và hy vọng CPU đủ nhanh.

### Lab 12 PWM với scope

1. Chọn một chân TIM3_CH1, chẳng hạn PA6 nếu AF trong D1 phù hợp board. Tạm tháo mọi thiết bị đang dùng chân đó.
2. Chọn PWM edge-aligned, 20 kHz theo timer clock thật. Đặt duty 25%, preload enable.
3. Đo period khoảng 50 µs và high time khoảng 12,5 µs. Đặt 50%, 75% và quan sát.
4. Thay đổi duty theo nút đã debounce; dùng scope kiểm tra không có xung quá ngắn ngoài ý muốn.
5. Thử output compare toggle với cùng ARR/CCR để thấy nó khác PWM. Thử one-pulse theo một trigger hợp lệ.
6. Thử duty biên 0% và gần 100%; đọc quy tắc output ở CCR/ARR biên để triển khai “off hoàn toàn” và “on hoàn toàn” đúng yêu cầu.

**Đạt:** báo cáo có tần số, duty, polarity, jitter và hành vi khi update. **Bài tập:** tăng PWM từ 20 kHz lên 100 kHz với 64 MHz, PSC=0 thì ARR=639, còn 640 khoảng thời gian mỗi chu kỳ; độ phân giải xấp xỉ 9,32 bit. [R chương 43–44, 46–47]


## Bài 13 Input capture đo xung và encoder

### Capture và đo khoảng thời gian

Input capture chốt giá trị CNT vào CCR khi xuất hiện cạnh đã chọn. Thời điểm được giữ bằng phần cứng, nên không bị dịch đi bởi latency vào ISR như cách đọc CNT trong EXTI. Bộ lọc input lấy nhiều mẫu để loại xung ngắn; nó cũng thêm delay và giới hạn tần số tín hiệu hợp lệ. Input prescaler có thể chỉ capture mỗi một số cạnh để giảm tải.

Đo period bằng hiệu hai capture trên cùng loại cạnh. Nếu timer N-bit và chỉ có tối đa một vòng wrap giữa hai capture, `delta = (new-old) mod 2^N`. Tần số vào bằng f_counter/delta. Với counter 1 MHz, delta=1000 tương ứng 1 kHz. Đo pulse width cần rising/falling capture hoặc PWM input mode với kết nối channel/trigger phù hợp.

Overcapture báo dữ liệu mới đến trước khi dữ liệu cũ được xử lý; phải phát hiện và thống kê. Đo tần số thấp cần counter rộng hoặc mở rộng overflow bằng phần mềm có xử lý race. Đo rất cao có thể đếm số cạnh trong một cửa thời gian thay vì timestamp từng cạnh.

### Encoder quadrature

Encoder A/B lệch pha khoảng 90° cho biết chiều và số bước. Encoder interface của timer dùng quan hệ hai tín hiệu để tăng/giảm CNT; mode quy định đếm trên tín hiệu/cạnh nào. Với encoder 1024 chu kỳ/vòng và giải mã x 4, một vòng cho 4096 count. Nhà sản xuất có thể gọi PPR/CPR khác nhau; phải định nghĩa bằng một phép đo quay đúng một vòng.

Tốc độ RPM = delta_count × 60 / (counts_per_rev × delta_time_s). Tại tốc độ thấp, count trong cửa nhỏ rất ít nên lượng tử lớn; đo period giữa cạnh hoặc dùng bộ ước lượng lai có thể tốt hơn. Bộ lọc quá mạnh làm mất cạnh; dây dài cần giao tiếp điện phù hợp, không chỉ tăng debounce phần mềm.

### Lab 13 Nối timer với timer

1. Dùng một timer phát PWM 1 kHz, 30%. Nối qua dây ngắn sang input capture của timer khác và chung GND.
2. Cấu hình counter capture 1 MHz, nhận cả period và high time qua mode/đường capture hợp lệ.
3. Tính ra period gần 1000 tick, high gần 300 tick. Đổi PWM 2 kHz rồi 100 Hz.
4. Cố ý trì hoãn việc đọc capture; kiểm tra overcapture tăng. Không báo “đo đúng” nếu đã mất mẫu.
5. Với encoder, đưa A/B vào hai channel, quay một vòng thuận/ngược; kiểm tra sign và số count. Nếu không có encoder, dùng nguồn tín hiệu hai pha phù hợp, không mô phỏng bằng hai nút bounce rồi kết luận timer sai.

**Đạt:** sai số có cận theo tick và clock, báo invalid khi không có cạnh quá timeout. **Bài tập:** counter 16-bit tại 1 MHz wrap sau 65,536 ms; không thể dùng một hiệu đơn để phân biệt period 100 ms và 34,464 ms nếu không đếm overflow. [R chương 43–44, 46–47]


## Bài 14 Timer nâng cao và ma trận kết nối ngoại vi

### Complementary output dead time và break

TIM1/TIM8 có các chức năng dành cho điều khiển như complementary outputs, dead-time, break, repetition counter và đồng bộ. Hai transistor cùng nhánh cầu dẫn đồng thời có thể gây shoot-through; dead-time tạo khoảng cả hai tắt giữa hai lần chuyển. Mã DTG có các miền mã hóa phi tuyến; không lấy “số register × một tick” cho mọi giá trị. Cần tính tDTS và tra bảng DTG của timer.

Break là đường đưa đầu ra về trạng thái định nghĩa khi phát hiện fault; logic này có thể phản ứng mà không đợi task. Trạng thái idle, polarity, OSSI/OSSR, MOE, automatic output enable và khả năng tái bật phải được thiết kế cùng nhau. Chỉ clear cờ break rồi chạy tiếp có thể tái cấp điện vào lỗi còn tồn tại.

Repetition counter điều tiết thời điểm update trong các timer có hỗ trợ. Master/slave timer dùng TRGO/ITR/trigger mode để reset/gate/start timer khác. Không phải mọi timer đều nối tới mọi timer; phải tra bảng interconnect và mux cụ thể.

### Lấy mẫu không cần CPU canh giờ

Một chuỗi có ích là timer tạo TRGO, ADC lấy mẫu đúng thời điểm, GPDMA chuyển kết quả vào RAM, CPU xử lý mỗi block. Một chuỗi khác là timer trigger DAC, DMA nạp sample tiếp theo. Đây là đồng bộ phần cứng, giảm jitter do scheduler.

Phải phân biệt ba thời điểm: cạnh trigger, khoảng sample-and-hold và lúc conversion hoàn thành. Muốn lấy mẫu ở giữa vùng PWM ít nhiễu, cần đặt cửa sample vào đúng vùng, không chỉ đặt interrupt ADC ở giữa chu kỳ. Timer output và ADC trigger có thể khác nhau dù dùng cùng một counter.

### Lab 14 Đồng bộ và thử fault không có công suất

1. Chưa nối motor/power stage. Cấu hình TIM1 hai output bổ sung trên các chân package cho phép.
2. Đặt PWM tần số thấp đủ dễ quan sát; tính dead-time theo clock và mã DTG. Đo khoảng cả hai đầu ra inactive bằng hai kênh scope.
3. Tạo tín hiệu break mức phù hợp qua mạch thử; xác nhận đầu ra chuyển trạng thái ngay cả khi main loop đang bận.
4. Giữ fault active và thử lệnh start; thiết kế phải từ chối hoặc vẫn giữ trạng thái an toàn.
5. Nhả fault, yêu cầu một thao tác acknowledge riêng rồi mới re-arm. Ghi nguyên nhân và số lần fault.
6. Cấu hình TRGO sang ADC ở bài 20; kiểm bảng kết nối thay vì thử ngẫu nhiên các trigger enum.

**Đạt:** có bảng state IDLE/RUN/FAULT/REARM, đồ thị đo dead-time và thời gian ngắt đầu ra. **Bài tập:** TrustZone bảo vệ code điều khiển có đủ để motor an toàn không? Không; nguồn, output state, giới hạn vật lý, lịch chạy và cơ chế hardware shutdown đều ảnh hưởng. [D1 mục 3.11, 3.31; R chương 15, 43]


# Chủ đề 4 Thời gian dài backup và watchdog

Bài 15 đến Bài 16. Đọc lịch buổi học để chọn sản phẩm cần hoàn thành trước khi chuyển bài.


## Bài 15 LPTIM RTC TAMP và backup domain

### Hai loại thời gian

LPTIM phục vụ đếm/xung và các tác vụ trong điều kiện low-power theo clock được phép. RTC phục vụ lịch và thời gian dài, có calendar, alarm, wake-up, timestamp và calibration. LPTIM không phải mọi mode đều tiếp tục chạy ở mọi power mode; phải kiểm bảng clock/wakeup trong R.

RTC thường chia LSE 32.768 kHz qua prescaler asynchronous/synchronous để tạo nhịp calendar. Với cấu hình phù hợp, (PREDIV_A+1)×(PREDIV_S+1)=32768 cho một giây. LSI có thể dùng khi không có crystal, nhưng độ chính xác thời gian dài bị giới hạn bởi sai số và nhiệt độ.

Calendar thường mã hóa BCD ở register; không nhầm 0x23 với số thập phân 23 khi xử lý thô. Khi dùng shadow register, cần tuân theo thứ tự đọc để có snapshot nhất quán; với HAL thường đọc time rồi date trong cùng thao tác ứng dụng. Không giữ riêng time đọc trước nửa đêm và date đọc sau nửa đêm mà coi là một timestamp.

### Backup và tamper

H523 có 32 backup register 32-bit và backup SRAM 2 KiB. Chúng hữu ích cho reset reason, checkpoint và thông tin cần giữ qua mất VDD khi VBAT hợp lệ, nhưng không thay bộ nhớ không bay hơi trong mọi tình huống. Backup reset, tamper policy và điều kiện nguồn có thể xóa nội dung.

TAMP hỗ trợ phát hiện can thiệp qua input và active tamper theo package. Active tamper phát tín hiệu và kiểm hồi đáp nhằm phát hiện đứt/chập hoặc sửa đường bảo vệ. Chính sách phản ứng có thể xóa dữ liệu nhạy cảm; phải phân biệt một cờ sự kiện bình thường với cấu hình có tác dụng xóa. Tránh dùng backup region nhạy cảm cho lab tamper khi có khóa thật.

### Lab 15 Đồng hồ còn chạy sau reset

1. Chọn LSE nếu board lắp crystal phù hợp; nếu không, dùng LSI và ghi rõ giới hạn đo. Không giả định HSE của debugger vẫn tồn tại khi rút cáp.
2. Đặt lịch thử, lưu magic/version vào backup register để không set lại lịch mỗi lần reset.
3. Đọc time/date thành một cấu trúc ứng dụng nhất quán, chuyển sang số nguyên thời gian khi cần protocol.
4. Tạo alarm hoặc wake-up 1 giây, đếm event; thử Sleep rồi Stop ở bài 36.
5. Reset MCU, xác nhận lịch không quay về giá trị ban đầu nếu backup domain được giữ.
6. Đo sai số sau một khoảng dài với nguồn thời gian tham chiếu. ppm = sai số giây / thời gian quan sát giây × 10^6. Ghi hướng và giới hạn bù calibration.
7. Lab tamper riêng: chỉ dùng pattern không nhạy cảm, kích một input, ghi cờ và kiểm chính sách xóa đã cấu hình.

**Đạt:** có thử qua nửa đêm, năm nhuận và reset; không coi reset và mất VBAT là cùng trường hợp. **Bài tập:** 20 ppm tương đương khoảng 1,728 giây/ngày. [D1 mục 3.31–3.32; R chương 48, 51–52]


## Bài 16 IWDG WWDG và giám sát sức khỏe hệ thống

### Watchdog không chỉ là reset định kỳ

IWDG sử dụng clock độc lập kiểu LSI, giúp phát hiện phần mềm không hoàn tất công việc trong thời gian đặt. Timeout xấp xỉ bằng (reload+1)×prescaler/f_LSI, nhưng phải dùng dải f_LSI để tính timeout sớm nhất và muộn nhất. Một watchdog “2 giây” danh định có thể không phải đúng 2 giây.

WWDG dựa trên clock hệ thống tương ứng, yêu cầu refresh trong một cửa: quá muộn hoặc quá sớm đều có thể là lỗi. Cửa này giúp phát hiện phần mềm lặp sai nhanh, điều mà watchdog chỉ có timeout trên không phát hiện được. Early wakeup interrupt hỗ trợ xử lý trước timeout trong điều kiện cho phép; không nên dùng nó để tự cứu vô hạn một hệ thống đã mất chức năng chính.

Refresh watchdog từ một timer ISR chạy đều có thể che việc mọi task ứng dụng đã chết. Một supervisor tốt nhận heartbeat gắn với tiến độ thật: hoàn thành block ADC, hoàn thành chu kỳ control, parser còn xử lý và storage chưa mắc kẹt. “Task vẫn được schedule” chưa chứng minh tác vụ đã làm đúng.

### Lab 16 Watchdog có fault injection

1. Tạo hai tác vụ logic trong superloop: thu mẫu và truyền status. Mỗi tác vụ cập nhật sequence khi hoàn thành một vòng công việc hợp lệ.
2. Supervisor chỉ refresh IWDG khi cả hai sequence tiến và không có lỗi deadline nghiêm trọng.
3. Chèn tùy chọn thử để làm parser kẹt; xác nhận reset xảy ra trong dải timeout tính bằng LSI.
4. Boot sau reset ghi cờ IWDG, tăng reset counter và vào chế độ chẩn đoán nếu lặp quá ngưỡng. Không tự kích tải công suất ngay sau reset liên tục.
5. Với WWDG, thử refresh đúng cửa, quá sớm và quá muộn. Đo khoảng thời gian bằng timer/logic analyzer.
6. Thử khi không gắn debugger. Nếu debugger freeze watchdog, một bài chạy thành công trong debug chưa chứng minh firmware ngoài thực tế hoạt động.

**Đạt:** reset xảy ra khi thành phần trọng yếu mất tiến độ, không reset với tải hợp lệ xấu nhất; sau reset dữ liệu và đầu ra ở trạng thái xác định. **Bài tập:** watchdog có đảm bảo giữ motor chạy không gián đoạn khi app fault không? Không; reset thường ảnh hưởng toàn chip. Muốn duy trì điều khiển phải thiết kế cô lập và phục hồi trước cấp watchdog, hoặc tách phần cứng khi yêu cầu bắt buộc. [R chương 49–50; D1 mục 3.31]


# Chủ đề 5 DMA cache và hệ thống analog

Bài 17 đến Bài 22. Đọc lịch buổi học để chọn sản phẩm cần hoàn thành trước khi chuyển bài.


## Bài 17 GPDMA direct transfer linked list và ownership

### DMA vận hành thế nào

DMA là bus master chuyển dữ liệu theo cấu hình mà không cần CPU thực hiện từng phép load/store. Một channel có địa chỉ nguồn, đích, độ rộng, increment, số byte, request, trigger, FIFO và thuộc tính truy cập. Request thường biểu thị ngoại vi sẵn sàng trao đổi dữ liệu; trigger điều khiển khi nào một tác vụ/block được phép tiến theo cấu hình. Hai khái niệm không đồng nghĩa.

Ví dụ ADC→RAM: nguồn là data register cố định, đích tăng sau mỗi sample, độ rộng dữ liệu phù hợp ADC và buffer. RAM→UART: nguồn tăng, đích TDR cố định. Memory→memory: cả hai thường tăng. Chọn sai increment có thể ghi đè một ô RAM liên tục hoặc ghi sang các register ngoại vi kế bên.

| Channel mỗi controller | FIFO theo R | Đặc điểm |
|---|---:|---|
| 0–3 | 8 byte | Address cố định/tăng tuyến tính; phù hợp nhiều luồng peripheral–SRAM |
| 4–5 | 32 byte | Tuyến tính, lưu lượng lớn hơn |
| 6–7 | 32 byte | Hỗ trợ thêm 2D/repeated block |

Priority DMA điều tiết arbitration của DMA, không phải NVIC priority. Bật cả hai ở mức cao nhất không làm mọi luồng nhanh hơn; một luồng ít khẩn cấp có thể gây tranh chấp với thu mẫu.

### Direct và linked list

Direct transfer dùng bộ register channel cho một cấu hình. Linked list dùng descriptor trong RAM để phần cứng nạp cấu hình kế tiếp. Node có thể cập nhật nguồn/đích/độ dài và các tham số được chọn; liên kết vòng cho phép chạy lặp. GPDMA H5 không nên được cấu hình theo khái niệm DMA stream/channel cũ của F4. “Circular” trong ví dụ thường được tạo bằng linked list vòng.

Descriptor là dữ liệu mà DMA sẽ đọc. Nó phải có alignment, địa chỉ/link format, vùng nhớ và quyền phù hợp. Không đặt node trên stack của hàm rồi return; không sửa node đang được DMA fetch mà không có quy trình suspend/ownership. R quy định base/link address và các giới hạn liên kết; dùng builder của driver đúng phiên bản để tránh tự đoán bit layout.

### CPU và DMA cùng sở hữu buffer

Buffer có trạng thái FREE→DMA_OWNED→READY→CPU_OWNED→FREE. Dùng hai nửa buffer không tự đảm bảo an toàn: nếu CPU chưa xử lý xong nửa đầu mà DMA vòng lại, dữ liệu bị ghi đè. Với N sample mỗi nửa ở fs, deadline xử lý xấp xỉ N/fs trừ các độ trễ và biên an toàn.

Mẫu 256 sample chia hai nửa, fs=10 ksample/s: mỗi nửa có 128 sample, thời gian xử lý cho một nửa khoảng 12,8 ms. Callback nên đưa descriptor hoặc sequence cho task, không xử lý FFT và printf trong IRQ. Một boolean “ready” có thể mất nhiều block; dùng counter/queue và phát hiện khoảng nhảy sequence.

### Lab 17 Từ copy RAM đến luồng liên tục

1. Chuẩn bị hai buffer 1024 byte và vùng canary trước/sau; điền nguồn bằng pattern phụ thuộc index, đích bằng 0.
2. Chạy memory-to-memory direct transfer với width/increment và số byte đúng. Nếu API lấy số transfer thay vì byte, chuyển đổi theo tài liệu API; không giả định mọi `Length` đều cùng đơn vị.
3. Sau complete, so từng byte và canary. Đo CPU busy so với memcpy; DMA có overhead nên buffer nhỏ chưa chắc nhanh hơn.
4. Tạo linked list hai node chuyển hai block vào hai vùng đích. Kiểm transfer-complete event phát theo block hay toàn list theo setting.
5. Chuyển sang vòng hai node với nguồn ngoại vi ở bài 20. Theo dõi HT/TC, transfer error, setting error và link error riêng.
6. Cố ý giảm khả năng xử lý CPU để tạo overrun ứng dụng. Báo lỗi rõ và chọn drop-old/drop-new/stop-acquisition theo yêu cầu; không báo dữ liệu vẫn liên tục.

**Đạt:** chứng minh đúng byte count, không vượt buffer, nhận và khôi phục được lỗi. **Bài tập:** channel “secure” có đủ không? Còn source/destination/link attributes, vùng RAM và quyền peripheral phải thống nhất. [D1 mục 3.16; R chương 16]


## Bài 18 Cache coherency và đo hiệu năng bộ nhớ

### Đúng kiến trúc H523

ICACHE 8 KiB nằm trên đường code, có thể cache instruction và các truy cập qua C-bus theo memory mapping. DCACHE4 KiB nằm trên đường dữ liệu ra bộ nhớ ngoài. Theo R mục 9.3, truy cập vùng SRAM nội/peripheral qua S-bus bỏ qua DCACHE này; cache line DCACHE là **16 byte**. Không lấy ví dụ Cortex-M7 “SRAM DMA phải clean line 32 byte” áp nguyên vào H523.

Cache lưu bản sao các dòng bộ nhớ. Hit lấy bản có sẵn, miss nạp từ bộ nhớ. Write-through cập nhật bộ nhớ theo policy; write-back có thể giữ dữ liệu mới trong cache với dirty bit đến khi eviction/clean. DMA thường không tham gia snooping cache của CPU, nên có nguy cơ hai bên nhìn hai phiên bản dữ liệu.

### Quy tắc trao quyền

Nếu CPU tạo dữ liệu trong vùng external cacheable rồi DMA đọc: hoàn tất ghi CPU, clean vùng dirty tới bộ nhớ, dùng barrier theo driver/kiến trúc, rồi trao ownership cho DMA. Nếu DMA ghi dữ liệu để CPU đọc: bảo đảm không còn dirty line CPU có thể ghi đè buffer, chờ DMA hoàn tất, invalidate vùng cần đọc trước khi CPU sử dụng. Thao tác trước/sau cụ thể phụ thuộc policy và trạng thái buffer; chỉ “invalidate sau DMA” chưa đủ khi còn dirty data trước đó.

Làm tròn vùng maintenance theo cache line có thể chạm dữ liệu kế bên. Nếu buffer không độc lập một line, invalidate có thể làm mất dữ liệu dirty của biến khác; clean có thể ghi nội dung cũ vào vùng DMA đang ghi. Vì vậy cả alignment lẫn kích thước/lifetime/ownership đều quan trọng.

Trên H523 dùng API của khối ICACHE/DCACHE ST tương ứng và status completion; không giả định các hàm cache của một lõi M7 có cùng ý nghĩa. Với code vừa cập nhật trong flash hoặc bộ nhớ ngoài, cần quy trình đảm bảo instruction fetch không tiếp tục dùng mã cũ.

### Lab 18 Chứng minh phạm vi cache

1. Vẽ đường CPU→SRAM nội và CPU→OCTOSPI/FMC, đánh dấu DCACHE có/không trên đường.
2. Với SRAM nội, lặp lab DMA và kiểm dữ liệu; không thêm cache maintenance vô căn cứ.
3. Nếu board có bộ nhớ ngoài: đặt một buffer riêng, chọn policy qua MPU/thuộc tính đúng, chạy CPU-write/DMA-read và DMA-write/CPU-read.
4. Tạo hai phiên bản có/không maintenance trong môi trường thử. Không mặc định mỗi lần thiếu maintenance đều tái hiện lỗi: cache có thể miss/evict tình cờ.
5. Đọc hit/miss counter, đo thời gian cold-cache và warm-cache. Ghi worst observed, không chỉ trung bình.
6. Tách code realtime và buffer lưu lượng lớn để xem contention. Không dùng cache để che một thiết kế không đáp ứng deadline khi cold boot.

**Đạt:** giải thích được vì sao dữ liệu “đúng khi debug, sai khi release” có thể liên quan timing/cache nhưng cần bằng chứng. **Bài tập:** `aligned(32)` có sai không khi line 16? Nó có thể vẫn đáp ứng alignment, nhưng không chứng minh chọn đúng policy và quy trình. [D1 mục 3.2; R chương 8–9, 2]


## Bài 19 ADC và cơ sở đo lường analog

### Cấu tạo và quá trình chuyển đổi

ADC lấy điện áp qua mux đầu vào, nạp sample capacitor trong thời gian sampling rồi chuyển thành mã số. Với SAR ADC, quá trình xấp xỉ liên tiếp quyết định các bit bằng so sánh điện áp tương ứng với mức tham chiếu. Hai giai đoạn lấy mẫu và chuyển đổi có thời gian hữu hạn.

Với 12-bit có 4096 mã 0…4095. LSB lý tưởng bằng VREF/4096. Nếu VREF=3,3 V thì khoảng 0,806 mV/LSB. Ước lượng điện áp thực dụng có thể dùng `raw*VREF/4095` khi muốn ánh xạ endpoint; mô hình lượng tử và vị trí tâm bin dùng hệ số 4096. Phải nêu quy ước, tránh gọi một trong hai là phép đo tuyệt đối chính xác.

Trở kháng nguồn cùng switch/sample capacitor tạo mạch nạp RC. Nguồn trở kháng cao cần sampling dài hơn hoặc buffer. Nếu đổi channel từ điện áp cao sang thấp mà sample chưa ổn định, có thể thấy phụ thuộc channel trước. Bộ lọc RC ở ngoài giúp chống alias nhưng cũng tăng tải động cần tính.

### Các chế độ

Regular group dùng cho chuỗi đo thường; injected group phục vụ chuỗi ưu tiên/trigger riêng theo cấu hình. Single conversion đo một lần; continuous lặp liên tục; scan thực hiện nhiều rank; discontinuous chia nhỏ chuỗi; external trigger đồng bộ timer. Single-ended đo so với mốc tham chiếu; differential dùng cặp input được phần cứng hỗ trợ và cần calibration/giới hạn khác.

Hai ADC có thể chạy độc lập hoặc phối hợp theo multimode có trong R. Analog watchdog so mã với ngưỡng mà không cần CPU kiểm từng sample; nó không thay mạch comparator ngoài khi cần latency/độ độc lập khác. Oversampling cộng nhiều mẫu để giảm nhiễu ngẫu nhiên và tăng số bit hữu dụng trong điều kiện phù hợp; không tự sửa offset, reference error hoặc alias.

Ở 12-bit, Tconv = (sampling_cycles + 12,5)/f_ADC theo R. Ví dụ f_ADC=32 MHz và sample 47,5 thì Tconv=1,875 µs. Đây là thời gian chuyển đổi mô hình, chưa tính mọi đồng bộ và mode. Muốn quảng cáo 5 Msps phải đồng thời đáp ứng clock và điều kiện điện cụ thể D1, không chỉ đặt continuous.

### Lab 19 Đo biến trở có kiểm sai số

1. Nối hai đầu biến trở 10 kΩ giữa 3,3 V analog và GND; cần gạt tới PA0/ADC1_INP0. Đảm bảo chân không vượt dải analog.
2. Chọn GPIO analog, ADC1 single-ended, resolution 12, một rank, software trigger, sampling 47,5 hoặc dài hơn khi cần; chọn ADC kernel clock hợp lệ.
3. Thực hiện chuỗi nguồn ADC, regulator, calibration và enable đúng driver/R. Chỉ start khi các cờ ready cho phép.
4. Đọc ba điểm gần 0, giữa và gần VREF; đồng thời đo điện áp bằng đồng hồ. Thu ít nhất 1000 sample mỗi điểm để tính mean/min/max/standard deviation.
5. Tăng trở kháng nguồn thử và giảm sampling, quan sát sai lệch. Khôi phục sampling rồi kiểm lại.
6. Thử calibration và bù offset/gain hai điểm; lưu hệ số với version/CRC ở bài 32.

**Đạt:** có bảng sai số đo được và giải thích reference/tải/nhiễu. **Bài tập:** ADC 12-bit không đảm bảo 12-bit accuracy; phân biệt resolution, ENOB, offset, gain error, INL, DNL và noise. [D1 mục 3.22, 5.3 phần ADC; R chương 28]


## Bài 20 ADC timer DMA VREFINT và DTS

### Thiết kế luồng lấy mẫu

Timer quyết định fs; ADC đổi mẫu; GPDMA đưa dữ liệu vào RAM; CPU xử lý block. Với nhiều rank, trigger có thể khởi động một sequence; thời gian hoàn thành cả sequence phải nhỏ hơn khoảng giữa trigger với biên an toàn. Hai channel trong scan được đo nối tiếp nên có độ lệch thời điểm; ứng dụng đo pha cần tính lệch này hoặc dùng hai ADC đồng bộ phù hợp.

Nyquist yêu cầu fs lớn hơn hai lần tần số cao nhất muốn biểu diễn, nhưng thực tế cần anti-alias filter và vùng chuyển tiếp. Lấy 10 ksample/s không có nghĩa mọi tín hiệu dưới 5 kHz được đo chính xác hoàn hảo. Tín hiệu ngoài dải có thể gập vào dải hữu ích và không thể sửa hoàn toàn bằng lọc sau ADC.

Với hai channel, `uint16_t sample[256][2]` là cấu trúc ứng dụng hợp lý nếu DMA ghi tuần tự rank 0, rank 1. Phải kiểm layout và độ rộng DMA; timestamp thường ghi cho block/trigger chứ không giả định mỗi phần tử chứa timestamp phần cứng. Gắn sequence để phát hiện mất block.

### Các kênh nội và nguồn tham chiếu

VREFINT là nguồn tham chiếu nội để ước lượng điện áp tham chiếu/cấp analog theo hệ số factory calibration và điều kiện D1. Nó không phải VREFBUF. VREFBUF là buffer tạo tham chiếu ra chân ở package hỗ trợ; H523C/R không có, nên bài này vẫn thực hiện được qua VREFINT mà không bật VREFBUF.

Một công thức dạng chung: VDDA_est = Vcal × VREFINT_CAL / VREFINT_RAW, sau khi quy đổi cùng resolution. Vcal, địa chỉ calibration, thời gian ổn định và sampling tối thiểu phải lấy từ D1/header đúng chip, không chép hằng 3,0 V/3,3 V từ dòng khác.

ADC còn có đường đo sensor nhiệt analog và VBAT theo cấu hình. **DTS** là một khối digital temperature sensor riêng, dùng thông tin đo/tần số và hệ số calibration theo R để suy ra nhiệt độ, có mode đo và threshold/interrupt. Nhiệt độ die không bằng nhiệt độ không khí; CPU/tải ngoại vi làm chip tự nóng.

### Lab 20 Thu mẫu đều và giám sát nhiệt

1. Chọn timer có TRGO nối được ADC1. Với f_TIM64 MHz, PSC 63, ARR 99 tạo trigger 10 kHz.
2. Tắt continuous để một trigger cho một chuỗi mong muốn. Cấu hình hai rank PA0 và một đầu vào hợp lệ khác; tổng conversion nhỏ hơn 100 µs.
3. Chuẩn bị buffer ping-pong/linked list trong SRAM nội; width half-word, nguồn cố định, đích tăng. Enable DMA và ADC trước khi start timer để không mất trigger đầu; tránh UG tạo sample thừa ngoài dự kiến.
4. Callback HT/TC chỉ đưa `{sequence, half_index, sample_count}` cho xử lý. Tính mean/RMS hoặc bộ lọc đơn ở main/task, đo thời gian phải dưới deadline.
5. Phát tín hiệu 1 kHz từ nguồn ngoài/DAC đã biết, kiểm phổ/chu kỳ và count. Giảm tốc consumer để kiểm overrun detection.
6. Trong một phép thử riêng, đọc VREFINT với sample time tối thiểu phù hợp kênh nội; so VDDA_est với đồng hồ.
7. Bật DTS, đọc calibration theo driver/R, theo dõi die temperature khi đổi clock/tải CPU. Thử ngưỡng trong điều kiện nhiệt bình thường bằng ngưỡng phần mềm thấp, không cố làm quá nhiệt chip.

**Đạt:** log xác nhận số sample theo thời gian, sequence liên tục, deadline và sai số. **Bài tập:** với 128 sample/channel mỗi nửa, hai channel 10 ksample/s/channel thì deadline vẫn 12,8 ms, không phải 6,4 ms; bus data rate là 40 kB/s. [D1 mục 3.22–3.23; R chương 15–16, 28–29]


## Bài 21 DAC VREFBUF và phát dạng sóng

### Từ mã số đến điện áp

DAC H523 có hai channel 12-bit, mỗi channel có data holding register, data output register, trigger logic và đường output/buffer. DHR giữ giá trị chờ; trigger chuyển dữ liệu theo cơ chế DAC sang DOR để tạo điện áp. Có các cách căn dữ liệu 8-bit, 12-bit trái/phải và ghi hai kênh theo khả năng register.

Điện áp lý tưởng có dạng VOUT = VREF+ × DOR/4096 cho DAC 12-bit khi mốc thấp bằng 0. Mã 4095 vì vậy không tạo chính xác VREF+ theo mô hình lý tưởng. Offset, gain, settling time, output impedance, tải và headroom buffer tạo sai lệch thực. DAC là nguồn tín hiệu, không phải power amplifier để kéo loa hay motor trực tiếp.

Mode normal dùng chuyển đổi thông thường; sample-and-hold giảm tiêu thụ trong ứng dụng phù hợp nhưng đòi hỏi cấu hình thời gian sample/hold/refresh và điều kiện ngoài. Buffer on/off đổi trở kháng đầu ra và dải swing/tải; self calibration của buffer nếu dùng phải theo trình tự. Noise/triangle generator tạo dạng đơn giản; arbitrary waveform dùng LUT và timer/DMA.

### Tần số dạng sóng và chất lượng

Nếu LUT có N mẫu, timer trigger ở fs thì một vòng LUT có f_wave=fs/N. N=100, fs=100 kHz tạo sin 1 kHz. Có thể dùng N=128, fs=128 kHz để xử lý thuận tiện. Tần số waveform không bằng DMA interrupt rate. DAC cần settle giữa các mẫu; tăng fs quá cao có thể méo dù timer và DMA không báo lỗi.

LUT sin 12-bit đặt quanh midpoint: code ≈ 2048 + A×sin(phase), với A giữ đủ headroom. Sau DAC thường cần reconstruction low-pass filter; dạng bậc thang có ảnh phổ quanh bội fs. Nếu chỉnh tần số bằng thay timer, kiểm thời gian settle và glitch; nếu dùng phase accumulator, hiểu sai số lượng tử pha/biên độ.

VREFBUF chỉ có trên H523 V/Z theo D1 bảng 2. Nó có enable/high-impedance và lựa chọn mức tham chiếu theo điện áp analog. Không bật VREFBUF khi chân tham chiếu đang bị nguồn ngoài ép vào mà không kiểm sơ đồ. Với H523CC, dùng VDDA/tham chiếu theo kết nối package, không gọi VREFBUF như một peripheral có sẵn.

### Lab 21 Loopback DAC tới ADC

1. Dùng PA4/DAC1_OUT1, tháo chức năng SPI/thiết bị khác khỏi chân. Đầu tiên không nối tải nặng; đo bằng đồng hồ/scope trở kháng cao.
2. Cấu hình DAC normal, software trigger hoặc không trigger theo driver; xuất 0,1024,2048,3072,4095. Ghi điện áp thực và headroom ở hai đầu.
3. Nối PA4 sang PA0 ADC qua mạch bảo vệ/lọc phù hợp; cùng tham chiếu. Không dùng đầu ra analog vượt range ADC.
4. Tạo LUT 100 mẫu, timer 100 kHz, DMA vòng; nạp mẫu đầu và start theo thứ tự trong R để không phát một mẫu rác ngoài ý muốn.
5. Quan sát sin 1 kHz, đo biên độ/DC offset. Tăng/giảm tốc và tải, phân biệt lỗi underrun với giới hạn settling analog.
6. Bật ADC timer 10 kHz đọc lại; tính mean/RMS và sai số. Hai converter dùng chung reference có thể che sai số reference, nên cần đồng hồ độc lập để đánh giá tuyệt đối.

**Đạt:** có bảng code→Vout, waveform, underrun counter và phản ứng khi DMA dừng. **Bài tập:** đặt 100 mẫu ở 200 kHz thì 2 kHz, nhưng chất lượng không tự giữ nguyên; phải đo lại. Trước lab đọc ES0621 mục 2.2.16 cho DAC clock. [D1 mục 3.24–3.25; R chương 30–31]


## Bài 22 OPAMP comparator và analog front end bên ngoài

### Phạm vi bài

STM32H523 không có OPAMP/COMP tích hợp trong danh mục D1. Bài này dạy thiết kế mạch ngoài để hoàn thiện chuỗi đo/phát, không tạo một ngoại vi hư cấu. OPAMP khuếch đại sai biệt điện áp; comparator quyết định bên nào cao hơn và tạo output digital. Hai loại có cấu trúc và giới hạn sử dụng khác nhau; không mặc định dùng op-amp làm comparator tốc độ cao.

Với negative feedback, op-amp cố điều chỉnh output để hai input gần bằng nhau trong miền hoạt động tuyến tính. Mạch voltage follower có gain gần 1 và output impedance thấp, giúp lái sample capacitor của ADC. Mạch non-inverting có gain lý tưởng 1+Rf/Rg; inverting có gain−Rf/Rin nhưng cần bias/mức nguồn phù hợp khi dùng một nguồn 3,3 V.

### Các giới hạn phải tính

Rail-to-rail input và rail-to-rail output là hai thuộc tính khác nhau, đều có điều kiện tải. Input common-mode phải nằm trong dải; output cần headroom. Offset input nhân gain gây sai số DC; input bias qua điện trở tạo offset; noise spectral density tích phân theo bandwidth tạo nhiễu tổng. Gain-bandwidth product giới hạn băng thông ở gain đã chọn; slew rate giới hạn tốc độ thay đổi điện áp lớn.

Với sin biên độ đỉnh A, tần số f, yêu cầu slew rate lý tưởng ít nhất 2πf A, cộng biên an toàn. A=1 V, f=10 kHz cần khoảng 0,063 V/µs. Đạt slew rate vẫn chưa đảm bảo ổn định với tải tụ; cần kiểm phase margin theo datasheet op-amp và có thể thêm điện trở cách ly/RC.

Comparator ngoài có hysteresis dùng hai ngưỡng để tránh nhấp nháy khi tín hiệu quanh ngưỡng. Có thể nối output phù hợp tới TIM break hoặc GPIO/EXTI. ADC analog watchdog cũng so ngưỡng nhưng nằm trong chu kỳ lấy mẫu, nên latency và tính độc lập khác comparator liên tục.

### Lab 22 Chuỗi cảm biến nhỏ

1. Chọn một op-amp ngoài theo nguồn 3,3 V, common-mode, output swing, GBW, offset, noise và tải ADC; đọc datasheet linh kiện cụ thể trước khi lắp.
2. Dùng nguồn tín hiệu an toàn 0,1–0,5 V và mạch gain 4 để tạo 0,4–2,0 V. Với non-inverting, chọn tỷ số Rf/Rg=3; giá trị tuyệt đối cần cân đối noise/bias/drive.
3. Đo input/output DC bằng đồng hồ; kiểm không saturation. Đo sin ở 100 Hz,1 kHz,10 kHz để thấy băng thông.
4. Nối output vào ADC với sampling phù hợp. So sai số khi có buffer và khi nguồn trở kháng cao nối trực tiếp.
5. Với comparator ngoài, đặt một ngưỡng thấp để mô phỏng fault, nối tới input số/TIM break qua mức điện hợp lệ. Quan sát hysteresis và thời gian phản ứng.

**Đạt:** có sơ đồ, tính gain, ngân sách sai số, đo waveform và giải thích giới hạn. **Bài tập:** “ADC 12-bit + op-amp gain 100” không tự đo được chính xác tín hiệu nhỏ hơn LSB/100, vì offset/noise/reference/saturation cũng được đưa vào hệ thống. Ứng dụng: đo dòng shunt, cảm biến áp lực, điều hòa tín hiệu DAC. [D1 bảng 2, mục 3.22–3.25, mục 5; lý thuyết mạch ngoài là phần bổ sung]


# Chủ đề 6 Giao tiếp nối tiếp và thiết kế protocol

Bài 23 đến Bài 28. Đọc lịch buổi học để chọn sản phẩm cần hoàn thành trước khi chuyển bài.


## Bài 23 USART UART LPUART RS 485 LIN và các chế độ đặc biệt

### Đường truyền và cấu tạo UART

UART là giao tiếp nối tiếp bất đồng bộ: hai đầu thống nhất baud/format nhưng không chia sẻ dây clock. USART bổ sung synchronous mode ở instance hỗ trợ. LPUART có cơ chế clock/baud và wakeup phù hợp tiêu thụ thấp, không phải bản UART giống hệt chỉ thêm tên “low power”. Bên trong có baud generator, shift register TX/RX, FIFO, control/status/error, interrupt và DMA request.

Logic UART mức 3,3 V khác RS-232, RS-485 và LIN ở tầng điện. RS-232 thường cần bộ chuyển mức và có đảo cực logic; RS-485 cần differential transceiver; LIN cần LIN transceiver. Không nối trực tiếp chân MCU vào dây bus 12 V hoặc cổng RS-232.

### Frame UART trên dây

| Trường theo thứ tự thời gian | Độ dài điển hình | Ý nghĩa |
|---|---:|---|
| Idle | Không cố định, mức 1 | Chưa truyền |
| Start | 1 bit mức 0 | Đồng bộ điểm bắt đầu |
| Data | 7/8/9 bit tùy mode | Thông thường LSB-first; kiểm cấu hình |
| Parity | 0 hoặc 1 bit | Even/odd kiểm lỗi đơn giản |
| Stop | Theo mode, thường 1 hoặc 2 bit mức 1 | Hoàn tất ký tự |

8N1 gồm 1 start+8 data+1 stop=10 bit/byte. 115200 baud tương ứng lý tưởng 11520 byte/s trước overhead gói. Gửi byte 0x55 tạo data bit theo thời gian 1,0,1,0,1,0,1,0, tiện đo bit width. Khi bật parity, lựa chọn word length của STM32 có thể bao gồm vị trí parity; kiểm số bit payload thật thay vì chọn “8-bit + parity” bằng trực giác.

USART hỗ trợ oversampling/FIFO theo instance, flow control RTS/CTS, IDLE/receiver timeout và các chế độ đặc biệt. IDLE chỉ nói đường dây đã rỗi theo quy tắc hardware, không chứng minh một packet ứng dụng đã đủ length/CRC. TXE/TXFNF/DMA complete báo khả năng nạp dữ liệu hoặc DMA đã chuyển vào peripheral; TC báo shift register đã truyền xong ở đường dây.

### Cấu trúc dữ liệu phần mềm

```c
typedef struct {
    uint32_t baud;
    uint8_t data_bits;
    uint8_t stop_bits;
    uint8_t parity;       /* enum ứng dụng, không phải giá trị HAL */
} uart_format_t;

typedef struct {
    uint32_t rx_bytes;
    uint32_t overrun;
    uint32_t framing;
    uint32_t parity_error;
    uint32_t ring_overflow;
} uart_stats_t;
```

Ring buffer cần head/tail và quy tắc đồng bộ giữa ISR/task. Không để parser giữ con trỏ vào vùng DMA sẽ sớm ghi đè. Khi dùng Receive-to-IDLE DMA, phải hiểu callback size là vị trí hay số byte mới của API; tính delta modulo và xử lý cả wrap, HT/TC và IDLE mà không đếm đôi.

### RS 485 và giao thức phía trên

RS-485 mô tả tầng điện, không định nghĩa packet ứng dụng. Half-duplex cần DE để bật driver. Bật DE trước ký tự đầu theo settling time, chỉ nhả sau TC và guard time cần thiết, không nhả ngay khi DMA complete. Modbus RTU là một giao thức có thể chạy trên RS-485, gồm address, function, data, CRC16; timing silence thuộc Modbus, không thuộc UART8N1. Muốn triển khai phải chọn chuẩn/profile và kiểm bằng test vector chính thức.

### LIN frame và scheduler

LIN thường dùng một commander/controller phát header theo schedule; node được chỉ định phát response. USART LIN mode hỗ trợ các chức năng break theo R, còn lịch, signal database, PID/checksum và timeout vẫn là trách nhiệm phần mềm hoặc stack.

| Phần LIN | Trường | Nội dung |
|---|---|---|
| Header | Break và delimiter | Khoảng dominant dài để đánh dấu đầu frame, rồi recessive |
| Header | Sync | Byte 0x55 để đồng bộ baud |
| Header | PID | ID6-bit +2 parity bit |
| Response | Data | 1–8 byte theo định nghĩa frame |
| Response | Checksum | Classic hoặc enhanced theo loại frame |

PID: P0=ID0 XOR ID1 XOR ID2 XOR ID4; P1=NOT(ID1 XOR ID3 XOR ID4 XOR ID5), lấy 1 bit. PID=ID|(P0<<6)|(P1<<7). ID0x12 cho PID0x92. Classic checksum là bù một của tổng 8-bit có end-around carry trên data; enhanced thêm PID vào tổng. ID0x3C/0x3D dùng classic. Ví dụ data 01 02: classic FC; enhanced với PID92 là 6A. Không thay end-around carry bằng lấy tổng modulo 256 đơn giản. [LIN]

```c
typedef struct {
    uint8_t id;          /* 0..63, chưa có parity */
    uint8_t length;      /* 1..8 do cấu hình frame quy định */
    bool enhanced;
    uint16_t period_ms;
    uint8_t payload[8];
} lin_frame_t;
```

LIN không có trường length trong header kiểu packet tự thiết kế; hai bên cần biết cấu trúc qua cấu hình/LDF. Multibyte signal packing, scale, offset và publisher/subscriber phải ghi thành tài liệu; không gửi nguyên struct.

### Smartcard IrDA và synchronous USART

Smartcard mode phục vụ giao tiếp ISO7816 ở mức UART phù hợp: character có start, data, parity, guard/retry/error handling theo chế độ. APDU ở tầng ứng dụng có CLA, INS, P1, P2 và các trường length/data/Le tùy case; response có data và SW1/SW2. USART không tự tạo toàn bộ ATR/PPS/T=0/T=1 stack. Dùng `smartcard_apdu_t` chứa header 4 byte, pointer/length data, Le và status thay vì một mảng không kiểm biên.

IrDA SIR dùng cách mã hóa xung quang cho bit UART và cần transceiver IR; payload bên trên vẫn phải có packet/profile. Half-duplex single-wire cần xử lý đổi chiều và bus ownership. Synchronous USART xuất/nhận clock cùng data ở mode hỗ trợ; phải kiểm CPOL/CPHA/last-bit timing và giới hạn so với SPI thật.

### Lab 23 Kiểm frame và phục hồi RX

1. Nối TX↔RX giữa hai đầu 3,3 V, chung GND. Cấu hình 115200 8N1, gửi 0x55 và đo bit khoảng 8,68 µs, ký tự khoảng 86,8 µs.
2. Gửi chuỗi có counter; thử polling, interrupt rồi DMA. Đếm loss/error với tải CPU tăng dần.
3. Cố ý đặt sai parity/baud ở đầu phát; quan sát FE/PE/ORE, clear/rearm đúng driver rồi khôi phục không cần reset cả MCU.
4. Với RS 485, đo DE cùng TX, xác nhận bit stop cuối không bị cắt. Thử hai node và timeout, không cho hai driver cùng phát.
5. Với LIN, dùng hai node/transceiver, schedule một frame ID12/data 0102. Bắt break,55,92,01,02,6A trong enhanced mode; sửa checksum để kiểm reject.
6. Nếu có smartcard/IR module, thêm lab giao dịch nhỏ, ghi cấu trúc request/response, timeout và lỗi parity. Không giả định mọi USART instance hỗ trợ mọi mode; chọn theo bảng D1 mục 3.35.

**Đạt:** bắt được frame, tính throughput, phát hiện dữ liệu lỗi/mất, có kế hoạch recovery. [D1 mục 3.35; R chương 56–57; LIN]


## Bài 24 Thiết kế data frame parser và version của giao thức

### Phân biệt ba cấu trúc

Frame đường dây như UART8N1 thuộc ngoại vi. Packet ứng dụng định nghĩa một thông điệp có ý nghĩa. C struct là cách biểu diễn trong RAM. Ba thứ không nhất thiết có cùng layout. Một DMA callback cũng không tương đương một packet: packet có thể bị chia thành nhiều callback, nhiều packet có thể nằm trong một callback.

Thiết kế mẫu dưới đây là **giao thức tự xây cho khóa học**, không phải chuẩn UART hay chuẩn ST. Giới hạn payload 240 byte để dễ quản lý bộ nhớ.

| Offset | Trường | Kích thước | Mã hóa |
|---:|---|---:|---|
| 0 | SOF | 2 | A5 5A |
| 2 | Version | 1 | 01 |
| 3 | Type | 1 | 01 sample,02 status,10 command |
| 4 | Sequence | 2 | Little-endian |
| 6 | Length | 2 | Little-endian,0…240 |
| 8 | Payload | Length | Schema theo Type/Version |
| 8+Length | CRC | 2 | CRC-16/CCITT-FALSE, little-endian |

CRC tính trên Version…Payload, không gồm SOF. Quy định poly 0x1021, init 0xFFFF, refin=false, refout=false, xorout 0; test vector ASCII“123456789” phải ra 0x29B1. Chỉ ghi “CRC16” không đủ vì có nhiều biến thể.

```c
typedef struct {
    uint8_t type;
    uint16_t sequence;
    uint16_t length;
    uint8_t payload[240];
} app_packet_t;

static void put_u16le(uint8_t *p, uint16_t v)
{
    p[0] = (uint8_t)v;
    p[1] = (uint8_t)(v >> 8);
}

static uint16_t get_u16le(const uint8_t *p)
{
    return (uint16_t)((uint16_t)p[0] |
                     ((uint16_t)p[1] << 8));
}
```

Không dùng `memcpy(wire,&packet,sizeof packet)` vì padding, endianness và `length` không được kiểm. Payload sample có thể định nghĩa `{timestamp_us:u32le, channel:u8, raw:u16le, flags:u8}`, tổng 8 byte. Version mới có thể thêm schema mới, không tự thay length mà giữ cùng ý nghĩa.

### State machine nhận và độ tin cậy

Parser đi qua SEARCH_SOF→HEADER→PAYLOAD→CRC→DISPATCH. Trước khi cấp buffer/chép, kiểm version, type và length. Timeout khi packet dở dang để tránh giữ state mãi. Sau CRC sai, tìm lại SOF với cơ chế giữ byte có thể là đầu SOF mới; không bỏ một vùng không giới hạn. Payload có thể chứa A55A vì parser đang theo length; SOF không phải ký tự escape trong thiết kế này.

CRC chỉ phát hiện lỗi ngẫu nhiên, không xác thực người gửi. Sequence phát hiện gap/duplicate trong cửa modulo được quy định; cần session ID hoặc reset handshake nếu sender reboot. Command có tác dụng phụ cần request ID, ACK/NACK, timeout và retry policy. “Tăng tốc motor thêm 10%” lặp hai lần nguy hiểm hơn “đặt tốc độ 1000 rpm”; ưu tiên command idempotent hoặc deduplication.

Backpressure phải được thiết kế: nếu producer nhanh hơn consumer lâu dài, mọi queue hữu hạn đều sẽ đầy. Chọn flow control, giảm sampling, drop có thống kê hoặc từ chối request; không giấu mất dữ liệu. Logs nên có reason code, sequence và timestamp, tránh in từng byte trong IRQ.

### Lab 24 Kiểm parser trên máy tính trước MCU

1. Viết encoder/decoder độc lập HAL. Kiểm vector CRC đã cho.
2. Encode một packet không payload và một packet 240 byte. Decode lại, so từng trường.
3. Cấp dữ liệu cho parser từng byte, theo block ngẫu nhiên và hai packet liền nhau; kết quả phải giống nhau.
4. Sửa một bit, cắt cuối packet, chèn byte rác, đặt length 241/65535, chèn SOF trong payload; parser không vượt buffer và không dispatch frame sai.
5. Thử sequence 65535→0, duplicate và sender restart. Ghi policy rõ ràng.
6. Tích hợp ring UART/DMA; chạy 10000 packet có sequence, đo mất/CRCfail/timeout. Gửi tải vượt bandwidth để kiểm backpressure.

**Đạt:** có bộ test tự động và spec đủ để người khác viết đầu nhận bằng Python. **Bài tập:** nếu đối phương chủ động sửa payload và tính lại CRC thì nhận có biết không? Không; muốn authenticity cần MAC/chữ ký và quản lý khóa ở bài 44. Đây là nền cho protocol trên UART, USB CDC, SPI, RS 485 và một số payload CAN. [R chương 19,56; cấu trúc packet do giáo trình thiết kế]


## Bài 25 I2C SMBus PMBus và cấu trúc giao dịch

### Tầng điện và nguyên lý

I2C dùng SDA và SCL chia sẻ, thông thường open-drain với pull-up. Bên nào kéo thấp thì mức bus thấp; mức cao cần điện trở kéo nạp điện dung bus. Rise time xấp xỉ 0,8473×Rp×Cb trong mô hình RC tương ứng mức 30–70%. Với R p 4,7 kΩ và Cb 100p F, rise khoảng 398 ns; phải so với giới hạn mode đã chọn, không chỉ nhìn tần số clock.

Controller phát clock và bắt đầu giao dịch, target trả lời khi được gọi. Multi-controller dùng arbitration; một bên phát mức 1 nhưng đọc 0 có thể thua quyền bus. Clock stretching cho phép target giữ SCL thấp theo mode/thiết bị. Pull-up quá lớn làm sườn lên chậm; quá nhỏ làm tăng dòng sink vượt khả năng thiết bị.

Khối I2C có timing generator, filter analog/digital, address match, shift register, TX/RX data, byte counter, reload/autoend và error flags. TIMINGR phụ thuộc kernel clock, rise/fall time và filter; không tái dùng giá trị hex khi thay clock/bus.

### Frame và byte order

| Bước write 7-bit | Ai điều khiển | Nội dung |
|---|---|---|
| START | Controller | SDA xuống khi SCL đang cao |
| Address | Controller | 7 bit địa chỉ, rồi R/W=0; MSB-first |
| ACK/NACK | Target | Bit thứ 9: kéo thấp ACK, nhả cao NACK |
| Data byte và ACK | Sender rồi receiver | Mỗi byte 8 bit +1 bit phản hồi |
| STOP | Controller | SDA lên khi SCL đang cao |

Ví dụ địa chỉ 7-bit 0x52: address byte write A4, read A5. Trong code lưu `addr7=0x52`; chỉ shift khi API yêu cầu địa chỉ đã dịch. Logic analyzer có thể hiển thị 7-bit 52 hoặc 8-bit A4; phải biết chế độ hiển thị trước khi kết luận sai địa chỉ.

Đọc register giả định 0x0F gồm: START, A4, ACK, 0F, ACK, repeated START, A5, ACK, byte 0, ACK, byte 1, NACK, STOP. Controller NACK byte đọc cuối để báo kết thúc; đây không phải lỗi. “Địa chỉ register” và endian của dữ liệu là quy ước thiết bị, không do I2C định nghĩa. Thiết bị có thể dùng register 8 bit,16 bit hoặc command khác.

```c
typedef struct {
    uint8_t addr7;
    uint8_t reg_addr_bytes;   /* 0,1,2 theo thiết bị */
    uint16_t reg;
    uint8_t *data;
    uint16_t length;
    uint32_t deadline_ms;
} i2c_transaction_t;
```

SMBus thêm quy tắc timeout, transaction và PEC ở các mode phù hợp; PEC dùng CRC-8 theo chuẩn SMBus. PMBus dùng cơ chế truyền SMBus để chuẩn hóa command quản lý nguồn, gồm các cách mã hóa giá trị như LINEAR 11/LINEAR 16 theo thiết bị. Có I2C hỗ trợ SMBus không có nghĩa tự hiểu command PMBus hay đơn vị engineering.

### Lab 25 Đọc cảm biến và phục hồi bus

1. Chọn cặp SCL/SDA đúng AF; dùng điện trở kéo đã tính. Bắt đầu 100 kHz và một target đã biết địa chỉ.
2. Bắt một write và repeated-start read bằng analyzer, ghi từng ACK và bit R/W. Đối chiếu register map của target.
3. Viết driver trả các lỗi riêng: NACK, address mismatch, arbitration lost, bus error, timeout. Mọi wait có deadline.
4. Tăng 400 kHz khi cả hai đầu hỗ trợ; đo rise time. Thay pull-up nếu cần trong giới hạn dòng, không chỉ tăng OSPEEDR.
5. Thử target không trả lời và reset target giữa giao dịch. Bus manager chỉ recovery khi có quyền độc chiếm bus và đã dừng controller hiện tại.
6. Nếu SDA bị giữ thấp bởi target mắc giữa byte, quy trình bus-clear có thể phát tối đa 9 xung SCL rồi STOP theo điều kiện thiết bị; nếu SCL vẫn thấp hoặc đường lỗi điện, không cố đẩy push-pull. Reset/power-cycle target chỉ khi thiết kế cho phép.

**Đạt:** đọc đúng endian/scale, parser không treo khi rút sensor, recovery không phá giao dịch của controller khác. **Bài tập:** cảm biến trả 12 34 theo big-endian 16 bit thì giá trị 0x1234, không phải 0x3412. [D1 mục 3.33; R chương 53]


## Bài 26 I3C SDR địa chỉ động CCC và IBI

### I3C khác I2C ở đâu

I3C dùng hai dây nhưng bổ sung dynamic address assignment, in-band interrupt, common command codes và cơ chế truyền hiệu quả hơn. Các pha có thể sử dụng open-drain hoặc push-pull theo chuẩn; không thể xem đây là I2C chỉ tăng clock. H523 triển khai I3C SDR theo khả năng nêu ở D1, không hỗ trợ HDR-DDR/HDR-TSL/HDR-TSP/HDR-BT chỉ vì chúng thuộc họ I3C.

Khối I3C quản lý bus controller/target, timing, control/status FIFO, TX/RX FIFO, thông tin device và interrupt. Một transfer có thể gồm nhiều descriptor/pha. CPU cần chuẩn bị đủ dữ liệu/response trước thời điểm phần cứng yêu cầu; không mặc định target có thể kéo dài mọi pha như I2C stretching.

### Địa chỉ và cấu trúc dữ liệu

Dynamic address là địa chỉ dùng trong phiên bus sau DAA, không phải danh tính bất biến. PID48-bit của I3C là Provisional ID, khác hoàn toàn Protected Identifier của LIN. BCR mô tả khả năng bus, DCR mô tả loại thiết bị. Phần mềm nên quản lý bảng thiết bị theo PID cùng địa chỉ hiện tại.

```c
typedef struct {
    uint64_t pid48;       /* chỉ48bit thấp hợp lệ */
    uint8_t bcr;
    uint8_t dcr;
    uint8_t dynamic_addr;
    bool online;
} i3c_device_t;

typedef struct {
    uint8_t dynamic_addr;
    bool read;
    uint8_t *buffer;
    uint16_t length;
    bool repeated_start;
} i3c_transfer_t;
```

DAA có quá trình arbitration theo thông tin định danh, rồi controller cấp địa chỉ hợp lệ. Không cấp trùng địa chỉ với target I2C hoặc các địa chỉ dành riêng. Khi reset/hot-join, bảng địa chỉ có thể phải cập nhật; lưu cứng dynamic_addr vào flash như số serial là sai.

### Cấu trúc frame SDR

| Thành phần | Ý nghĩa |
|---|---|
| START hoặc repeated START | Bắt đầu một pha/giao dịch |
| Address 7 bit và R/W | Chọn target và chiều |
| Address ACK/NACK | Phản hồi ở pha địa chỉ |
| Data 8 bit và T-bit | Byte dữ liệu với ý nghĩa T theo chiều |
| STOP hoặc pha tiếp | Kết thúc hoặc tiếp tục transfer |

SDR private write dùng T-bit kiểm odd parity của byte. SDR private read dùng T-bit cho cơ chế kết thúc/tiếp tục truyền và early termination, không phải ACK/NACK sau từng byte kiểu I2C. Các pha chuyển quyền lái SDA phải theo chuẩn; không bit-bang dựa trên bảng rút gọn này để thay IP.

CCC broadcast dùng địa chỉ broadcast 0x7E và command code, có thể có payload. CCC direct nhắm target qua các pha địa chỉ/command phù hợp. Ví dụ ENTDAA khởi động quy trình cấp địa chỉ; RSTDAA xóa địa chỉ động theo phạm vi chuẩn. IBI cho target yêu cầu phục vụ ngay trên bus; có thể có MDB và payload tùy BCR/cấu hình. Hot-join là cơ chế một target mới xin tham gia, không chỉ là một interrupt GPIO.

I3C controller có thể giao tiếp một số legacy I2C target nhưng cần kiểm hạn chế: D1 nêu các target legacy trong trường hợp hỗ trợ không thực hiện clock stretching. H523 I3C target không được mặc định có thể dùng như I2C target tĩnh. [D1 bảng 8; R chương 55]

### Lab 26 Khám phá và truyền dữ liệu

1. Cần target I3C thật hoặc một MCU hỗ trợ I3C target; module chỉ I2C không làm được DAA/IBI. Thiết kế điện và pull-up theo bus I3C, tránh level-shifter I2C cũ không hỗ trợ push-pull.
2. Cấu hình controller SDR, khởi tạo bus ở trạng thái hợp lệ, chạy DAA. Log PID48/BCR/DCR/dynamic address.
3. Gửi private write/read với payload có sequence, bắt address/data/T bằng analyzer có decoder I3C. Decoder I2C có thể hiển thị sai T-bit.
4. Dùng một CCC đọc thông tin được target hỗ trợ; kiểm unsupported command và timeout.
5. Cho target phát IBI; handler ghi descriptor ngắn rồi task xử lý payload. Giới hạn queue để IBI liên tục không chiếm mọi thời gian CPU.
6. Reset target và chạy lại khám phá theo chính sách bus. Không sử dụng địa chỉ cũ nếu chưa xác nhận.
7. Thử mixed bus chỉ với legacy target đáp ứng điều kiện; ghi rõ giới hạn speed và stretching.

**Đạt:** bảng device được quản lý theo danh tính, không treo khi target biến mất, phân biệt IBI với dữ liệu thông thường. **Bài tập:** vì sao giữ nguyên driver I2C và đổi baud không tạo ra I3C? Vì cách cấp địa chỉ, quyền lái dây, framing và sự kiện khác nhau. [D1 mục 3.34; R chương 55; AN5879 bổ sung ví dụ]


## Bài 27 SPI cấu trúc transaction FIFO và DMA

### Nguyên lý và các mode

SPI dùng SCK, một hoặc hai đường data và NSS/CS. Controller thường phát clock; target được chọn bằng CS. Trong full-duplex, mỗi bit clock vừa dịch một bit đi vừa nhận một bit về. Muốn đọc N byte, controller thường phải phát N byte dummy sau command/address theo giao thức thiết bị.

| Mode | CPOL | CPHA | Cạnh lấy mẫu tương đối |
|---:|---:|---:|---|
| 0 | 0 | 0 | Cạnh đầu, thường cạnh lên |
| 1 | 0 | 1 | Cạnh thứ hai, thường cạnh xuống |
| 2 | 1 | 0 | Cạnh đầu, thường cạnh xuống |
| 3 | 1 | 1 | Cạnh thứ hai, thường cạnh lên |

CPOL xác định SCK khi idle; CPHA xác định pha sample/shift. Khi CPHA=0, dữ liệu đầu phải hợp lệ trước cạnh lấy mẫu đầu. Thứ tự MSB/LSB là cấu hình riêng, không suy ra từ mode. H523 SPI có FIFO, frame size, transfer size và các cờ kết thúc theo IP; phải hiểu TXP/RXP/EOT và trạng thái truyền tương ứng thay vì dùng cờ của SPI đời khác.

Cấu hình liên quan gồm controller/target, full-duplex/simplex/half-duplex, hardware/software NSS, data size, baud prescaler, bit order và FIFO threshold. FIFO threshold phải phù hợp độ rộng truy cập và lượng dữ liệu còn lại. Nếu cấu hình DMA word mà buffer chỉ chứa byte theo một layout khác, có thể gặp packing hoặc byte order bất ngờ.

### SPI không có một packet chung

SPI không chuẩn hóa address, ACK hay CRC cho mọi chip. Một sensor có thể dùng bit 7 của command để chọn read; serial NOR có opcode riêng; display có chân D/C ngoài. Ví dụ sau là transaction mẫu cho thiết bị nhớ **được quy định** nhận opcode 03 và address 24-bit; chỉ dùng khi datasheet thiết bị xác nhận.

| Thứ tự khi CS active | MOSI | MISO |
|---|---|---|
| 1 | Opcode 03 | Không có nghĩa hoặc status |
| 2–4 | Address A23…A0, byte cao trước | Thường không dùng |
| 5 trở đi | Dummy byte 00 để tạo SCK | Data tại address và các byte tiếp |
| Kết thúc | CS inactive sau bit cuối | Target thả MISO theo quy định |

Không được toggle CS giữa opcode/address/data nếu đó phải là một transaction. Một DMA complete chỉ báo dữ liệu đã chuyển đến SPI, không luôn báo bit cuối đã ra dây; chờ đúng cờ hoàn thành theo R/errata trước khi nhả CS. Nếu mode của thiết bị có dummy **cycle**, phải phân biệt số cycle với số byte dummy do CPU phát.

```c
typedef struct {
    const uint8_t *tx;
    uint8_t *rx;
    uint16_t length;
} spi_segment_t;

typedef struct {
    uint32_t max_hz;
    uint8_t mode;
    bool lsb_first;
    uint8_t word_bits;
    uint32_t cs_setup_ns;
    uint32_t cs_hold_ns;
} spi_device_profile_t;
```

Các cấu trúc trên là descriptor ứng dụng, không phải byte trên dây. Bus manager khóa toàn transaction, chọn mode/clock từng device, giữ CS đúng phạm vi các segment rồi mới nhả bus. Tránh mutex chỉ quanh mỗi lệnh Transmit khiến thread khác xen giữa header và data.

### Lab 27 Từ loopback đến giao dịch thật

1. Chọn SPI1 với SCK/MISO/MOSI đúng AF. Loopback MOSI→MISO khi không có target và không gây xung đột; truyền pattern 00, FF, 55, AA.
2. Đo bốn mode để thấy sample edge. Loopback đúng chưa chứng minh mode đúng cho chip ngoài.
3. Nối một thiết bị đã có register map; đọc ID theo opcode/CS/timing của nó. Ghi lại từng byte TX/RX kể cả dummy.
4. Dùng interrupt và DMA; đặt buffer có lifetime đủ dài. Thử độ dài không chia đều FIFO threshold, một byte và lớn hơn FIFO.
5. Tăng clock trong giới hạn thiết bị/PCB; đo setup/hold/ringing. Nếu đọc sai ở tốc độ cao, giảm tốc và đổi đường dây là phép chẩn đoán, không phải bằng chứng chip SPI hỏng.
6. Reset target, mô phỏng timeout và xử lý overrun/underrun, đưa CS về inactive trước khi retry.

**Đạt:** hiểu toàn transaction và có trace khớp register map. **Bài tập:** đọc 16 byte với opcode 1 byte, address 3 byte và dummy 1 byte cần ít nhất 21 byte clock nếu giao thức đúng như mô tả, không phải 16. [D1 mục 3.36; R chương 58]


## Bài 28 I2S và luồng âm thanh số

### Cấu trúc tín hiệu

I2S dùng bit clock BCLK/SCK, word-select WS và data; có thể cần master clock MCLK cho codec. Trong chuẩn Philips I2S, MSB của word bắt đầu trễ một bit clock sau khi chuyển WS. Left-justified và right-justified khác điểm đặt dữ liệu; PCM short/long frame lại dùng frame sync khác. Hai đầu cùng sample rate nhưng khác format có thể phát ra âm thanh méo hoặc giá trị dịch bit.

H523 có I2S multiplex với một số SPI, không có SAI riêng trong D1. Khi SPI instance đã chuyển sang I2S, không dùng nó đồng thời như SPI thường. Hỗ trợ TX/RX/full-duplex, master/slave và format phải tra từng instance.

### Frame slot và data structure

| Khái niệm | Ví dụ stereo 48 kHz | Phân biệt |
|---|---|---|
| Sample frame | Một mẫu trái và một mẫu phải | 48000 frame/s |
| Slot | Một vị trí kênh trên bus | Có thể 32 bit/slot |
| Valid sample bits | Ví dụ 24 bit có dấu | 8 bit còn lại không nhất thiết là âm thanh |
| BCLK | 48000×2×32=3,072 MHz | Không phải 48 kHz |
| MCLK | Phụ thuộc codec, chẳng hạn tỷ số 256fs | Không mặc định bắt buộc/cố định |

```c
typedef struct {
    int32_t left;
    int32_t right;
} pcm_stereo_t;
```

Đây là biểu diễn logic. Buffer DMA có thể được đóng gói khác cấu trúc này tùy độ rộng register và driver. Với 24-bit two's complement, phải sign-extend đúng vào 32 bit và biết 24 bit hợp lệ nằm ở vị trí nào trong slot. Không ép mọi word 32 bit về int16 mà không scale.

Một buffer 128 frame stereo với 32 bit/kênh chiếm 1024 byte; ở 48 kHz deadline mỗi 128 frame khoảng 2,667 ms. Ping-pong và queue phải duy trì deadline này. Codec clock drift có thể làm buffer tràn/rỗng khi đồng bộ với USB hoặc nguồn mẫu khác; chỉ tăng buffer không giải quyết drift dài hạn. Cần dùng một clock chung, feedback hoặc rate matching theo giao thức ứng dụng.

### Lab 28 Phát và thu tín hiệu âm thanh

1. Chọn codec/DAC âm thanh số phù hợp 3,3 V và format. Không nối loa trực tiếp vào chân I2S.
2. Cấu hình 48 kHz, hai slot 32 bit, dữ liệu sin 1 kHz biên độ nhỏ. Đo nhịp WS/BCLK và xác nhận khoảng dịch MSB.
3. Dùng DMA ping-pong, đếm underrun/overrun. Tạo công việc CPU chậm để kiểm đường báo lỗi.
4. Thu âm hoặc loopback digital qua thiết bị phù hợp; kiểm left/right không bị đảo, không sai bit shift, không clipping.
5. Xuất dữ liệu PCM cho PC kèm metadata sample_rate, channels, bits, endianness. WAV header là format file, không phải phần bus I2S.

**Đạt:** phổ có đỉnh 1 kHz đúng, không underrun trong khoảng thử đặt trước, biết ngân sách RAM và deadline. **Bài tập:**48 kHz stereo 16-bit payload là 192 kB/s, vượt UART115200 8N1 rất xa; cần USB hoặc kênh phù hợp. [D1 mục 3.36; R chương 58]


# Chủ đề 7 CAN USB và quản lý nguồn Type C

Bài 29 đến Bài 31. Đọc lịch buổi học để chọn sản phẩm cần hoàn thành trước khi chuyển bài.


## Bài 29 FDCAN CAN Classical và CAN FD

### Tầng điện và arbitration

CAN dùng bus vi sai CAN_H/CAN_L qua transceiver. Controller FDCAN trong MCU xử lý bit/frame/filter/error; transceiver xử lý điện. Bus tuyến tính thường có termination 120Ω ở mỗi đầu; đo khi tắt nguồn giữa hai dây thường gần 60Ω khi có đúng hai điện trở và không có mạch khác ảnh hưởng. Không gắn 120Ω ở mọi node. Topology, stub, common-mode và propagation delay quyết định bitrate khả dụng.

CAN arbitration dựa trên dominant/recessive và identifier. Node phát recessive nhưng quan sát dominant trong arbitration sẽ nhường bus mà không phá frame đang thắng. Identifier nhỏ hơn thường có ưu tiên cao hơn trong so sánh arbitration tương ứng; ID là định danh/độ ưu tiên thông điệp, không bắt buộc là địa chỉ node.

Một node nhận đúng frame tham gia ACK theo chế độ. ACK không chứng minh task ứng dụng đã xử lý lệnh. Chỉ có một node thật trên bus có thể gặp ACK error dù dạng sóng TX đẹp. Internal loopback kiểm logic controller nhưng không kiểm transceiver, termination hay clock của node thứ hai.

### Frame Classical CAN

| Phần | Nội dung |
|---|---|
| SOF | Bắt đầu frame bằng dominant bit |
| Arbitration | ID11-bit hoặc định dạng extended 29-bit và các bit phân biệt/liên quan |
| Control | Định dạng, DLC và các bit điều khiển |
| Data |0–8 byte với data frame Classical |
| CRC | CRC sequence 15-bit và delimiter |
| ACK | ACK slot và delimiter |
| EOF |7 recessive bit; sau đó intermission theo quy tắc bus |

Remote frame của Classical yêu cầu dữ liệu theo ID, khác data frame và không có data payload. CAN FD không dùng remote frame kiểu này. Bit stuffing và error frame tạo overhead; không tính bus load chỉ bằng 8 byte payload.

### CAN FD và DLC

CAN FD có FDF, BRS cho phép đổi tốc độ data phase, ESI phản ánh trạng thái lỗi và payload tối đa 64 byte. Nominal/arbitration bitrate vẫn phải phù hợp mọi node. CAN Classical controller không tự đọc được FD frame trên cùng bus chỉ vì tốc độ arbitration bằng nhau. ISO CAN FD sử dụng CRC17 cho payload tới 16 byte và CRC21 cho payload lớn hơn, với quy tắc stuffing/check bổ sung; không thay bằng một CRC ứng dụng chung.

| DLC |0–8 |9 |10 |11 |12 |13 |14 |15 |
|---|---|---|---|---|---|---|---|---|
| Payload CAN FD byte | Bằng DLC |12 |16 |20 |24 |32 |48 |64 |

```c
typedef struct {
    uint32_t id;
    bool extended;
    bool fd;
    bool brs;
    uint8_t length;       /* byte thực, không phải mã DLC */
    uint8_t data[64];
    uint32_t timestamp;
} can_message_t;
```

Codec đổi length↔DLC, kiểm ID<=0x7FF hoặc<=0x1FFFFFFF và không cho BRS/length 64 trong frame Classical. Nếu payload logic 10 byte cần DLC9=12 byte, phải định nghĩa padding hoặc trường length riêng; receiver không tự biết chỉ 10 byte có nghĩa.

### Bit timing và message RAM

Bitrate = f_kernel / (prescaler×(1+TSEG1+TSEG2)), khi dùng các giá trị có ý nghĩa vật lý. Register thường mã hóa các trường bằng giá trị trừ 1; driver có thể nhận giá trị khác. Sample point = (1+TSEG1)/(1+TSEG1+TSEG2). Ví dụ 64 MHz, prescaler 8, TSEG1=13, TSEG2=2 cho 500 kbit/s và 87,5%. SJW phải hợp giới hạn và propagation. Đây là ví dụ tính, chưa phải cấu hình tối ưu cho mọi mạng.

FDCAN dùng filter, Rx FIFO0/1, dedicated buffers, TxFIFO/queue và Tx event FIFO trong message RAM. Dung lượng/cách chia theo instance phải lấy đúng R/HAL. Một filter rộng làm CPU chịu mọi frame; một filter sai làm bus có dữ liệu nhưng ứng dụng không nhận. Bus-off cần recovery policy có thời gian, trạng thái đầu ra và log; không reset/retry dồn dập vào bus lỗi.

### Lab 29 Từ loopback tới hai node

1. Internal loopback Classical 500k, ID0x123, payload `{sequence:u16le, raw:u16le}`. Xác nhận filter chỉ nhận dải mong muốn.
2. Nối hai node qua hai transceiver và termination đúng; đo CAN_H/CAN_L bằng cách đo phù hợp, không kẹp ground scope gây chập một dây bus.
3. Gửi sequence ở 100 Hz, đo frame, ACK và số mất. So data ID11 với extended 29.
4. Chuyển cả hai sang FD, thử payload 8,12,64 byte; kiểm DLC mapping, BRS và data timing bằng analyzer hỗ trợ FD.
5. Tắt node nhận để tạo ACK error; quan sát error counter/error-passive/bus-off theo điều kiện. Khôi phục node và kiểm recovery policy.
6. Làm Rx consumer chậm để tạo FIFO lost; log mất message, không coi FIFO overwrite là hoạt động bình thường.

**Đạt:** có bus-load estimate, worst latency dưới tải, lỗi được thống kê và khôi phục. CANopen/J1939/ISO-TP/UDS là tầng trên, phải chọn một profile khi dự án yêu cầu; FDCAN không tự cung cấp những stack đó. [D1 mục 3.38; R chương 60]


## Bài 30 USB full speed packet endpoint descriptor và class

### Kiến trúc host và device

USB FS truyền tín hiệu danh định 12 Mbit/s trên D+/D− nhưng payload ứng dụng thấp hơn do token, handshake, scheduling và stack. Host lên lịch transaction; device không tự gửi bất kỳ lúc nào theo kiểu UART. Endpoint là đầu mút logic trong device; pipe là liên kết logic do host quản lý. Endpoint number và direction cùng xác định địa chỉ endpoint, ngoại trừ cách điều khiển đặc biệt của EP0.

H523 có USB FS host/device theo D1. Không suy ra có USB High Speed 480 Mbit/s, external ULPI hoặc Ethernet. Board phải có routing, ESD, supply và VBUS sensing/power phù hợp. Khi làm host cần mạch cấp VBUS và bảo vệ; đổi mode firmware không tự tạo nguồn 5 V an toàn.

### Packet và transaction

| Loại packet FS | Trường chính | Vai trò |
|---|---|---|
| Token | SYNC, PID, ADDR 7, ENDP 4, CRC5, EOP | Host chọn địa chỉ/endpoint và loại IN/OUT/SETUP |
| Data | SYNC, PID DATA0/1, payload, CRC16, EOP | Mang byte dữ liệu |
| Handshake | SYNC, PID ACK/NAK/STALL…, EOP | Phản hồi kết quả/khả năng nhận |
| SOF | Frame number và CRC5 trong định dạng SOF | Mốc lập lịch FS, danh định 1 ms |

PID có 4 bit loại và 4 bit bù để kiểm tra. USB sử dụng mã hóa/bit stuffing ở tầng vật lý; byte ứng dụng không được nối trực tiếp thành waveform bằng GPIO. Một OUT transaction thường có OUT token, DATA và handshake; một IN transaction có IN token, DATA hoặc NAK/STALL rồi ACK khi thích hợp. Isochronous có quy tắc khác, không có cơ chế retry/ACK như bulk.

| Transfer type | Đặc tính | Ví dụ |
|---|---|---|
| Control | Cấu hình/điều khiển qua setup, data, status | Enumeration, class request |
| Bulk | Tin cậy theo USB, tận dụng bandwidth còn lại | CDC data, mass storage |
| Interrupt | Host polling theo interval | HID, thông báo trạng thái |
| Isochronous | Băng thông/timing ưu tiên, không retry như bulk | Audio/video thời gian thực |

USB transfer ứng dụng có thể gồm nhiều packet. Một callback 64 byte không phải ranh giới message của giao thức ở bài 24. Zero-length packet và short packet có thể ảnh hưởng kết thúc transfer tùy class/stack.

### Data structure của control và descriptor

SETUP payload luôn 8 byte, các trường 16 bit dùng little-endian:

| Offset | Trường | Byte |
|---:|---|---:|
|0 | bmRequestType |1 |
|1 | bRequest |1 |
|2 | wValue |2 |
|4 | wIndex |2 |
|6 | wLength |2 |

Device descriptor, configuration, interface, endpoint và string descriptors mô tả device cho host. Endpoint descriptor có bEndpointAddress, bmAttributes, wMaxPacketSize và bInterval. Một CDC function thường cần các interface và class-specific descriptors đúng liên kết; chỉ đổi tên thiết bị không tạo được một CDC hợp lệ.

```c
typedef struct {
    uint8_t request_type;
    uint8_t request;
    uint16_t value;
    uint16_t index;
    uint16_t length;
} usb_setup_view_t;
```

Đây là view sau khi parse 8 byte, không bảo đảm`sizeof` bằng 8. Stack thường đã giải mã setup; ứng dụng phải kiểm direction, length và trạng thái trước khi truy cập buffer. Với device control transfer, status stage là một pha riêng, không phải một byte status tự thêm vào response.

### Lab 30 USB CDC có protocol

1. Cấu hình USB device FS đúng nguồn, clock 48 MHz, pin D+/D−, VBUS và stack hỗ trợ đúng H523. Kiểm HSI48/CRS nếu dùng crystal-less.
2. Nạp ví dụ CDC được port đúng chip. Trên PC xem device/configuration/interface/endpoint descriptors, lưu một capture enumeration.
3. Gửi packet bài 24 qua CDC; đầu nhận phải xử lý packet bị chia nhỏ hoặc nhiều packet nối nhau. Thử length 63,64,65,128 byte để lộ lỗi giả định ranh giới callback.
4. Rút/cắm cáp, suspend/resume, thử khi host chưa mở port hoặc ngừng đọc. Callback không được chờ vô hạn hoặc giữ lock khóa toàn ứng dụng.
5. Đo throughput payload và CPU load. Kiểm NAK là backpressure bình thường, STALL là trạng thái khác cần xử lý đúng.
6. Bài host mở rộng dùng thiết bị được hỗ trợ, nguồn VBUS giới hạn dòng và class driver phù hợp; kiểm ngắt kết nối giữa giao dịch.

**Đạt:** enumeration thành công lặp lại, descriptors hợp lệ, protocol không phụ thuộc packet size và có reconnect. [D1 mục 3.39; R chương 61]


## Bài 31 UCPD USB Type C và Power Delivery

### USB dữ liệu và USB nguồn

Type-C mô tả connector và các vai trò/kết nối; USB data dùng D+/D− ở FS; USB Power Delivery trao đổi thông điệp trên CC. Có cổng Type-C không có nghĩa thiết bị đã hỗ trợ PD, và USB PD không phải một class packet trên USB FS. H523C/R/V/Z có UCPD, H523H không có theo D1.

UCPD cung cấp logic CC, phát/nhận PD và các sự kiện liên quan. Phần mềm Type-C state machine, protocol layer và policy engine phối hợp với mạch nguồn ngoài. Source cấp nguồn, sink nhận nguồn; data role DFP/UFP là trục khác, không đồng nhất vĩnh viễn với power role.

Rp/Rd giúp phát hiện attach và xác định khả năng dòng theo cơ chế Type-C. Cáp đảo chiều dẫn đến CC 1 hoặc CC 2 hoạt động. Dead-battery pin/pull-down có thể ảnh hưởng chân multifunction; khi không dùng PD vẫn phải kiểm cấu hình pin, không kết luận GPIO hỏng.

### Frame PD và cấu trúc dữ liệu

| Phần trên CC | Nội dung |
|---|---|
| Preamble | Đồng bộ thu |
| SOP ordered set | Phân loại đích giao tiếp |
| Header 16 bit | Loại message, roles, revision, message ID, số data object, extended flag |
| Data |0–7 data object 32 bit ở message không extended |
| CRC32 | Kiểm lỗi packet |
| EOP | Kết thúc packet |

Tầng vật lý dùng BMC và quy tắc mã hóa PD, không phải UART. Extended message có extended header/chunking theo revision; không coi mọi message là tối đa 28 byte payload ứng dụng đơn giản. GoodCRC phản ánh packet đã được nhận hợp lệ ở protocol layer, không chứng minh yêu cầu nguồn đã được chấp nhận.

PDO mô tả khả năng nguồn; RDO là yêu cầu được chọn theo khả năng đã quảng bá. Bitfield và đơn vị phụ thuộc type fixed/variable/battery/PPS và revision. Phải parse type trước, kiểm range và công suất cáp/mạch trước khi điều khiển nguồn.

```c
typedef struct {
    uint16_t header;
    uint8_t object_count;
    uint32_t object[7];
} pd_message_t;

typedef struct {
    uint32_t max_millivolts;
    uint32_t max_milliamps;
    bool allow_role_swap;
} pd_policy_t;
```

Struct là dữ liệu sau parse; codec serialize object theo byte order chuẩn PD. Policy ứng dụng phải hạn chế theo thiết kế phần cứng, không theo mọi PDO mà đối tác có thể gửi. State machine tối thiểu phải phân biệt unattached, attached, negotiating, ready, error recovery; chuyển state gắn với deadline, sự kiện và hành động nguồn cụ thể.

### Lab 31 Sink 5 V có quan sát negotiation

1. Dùng board Type-C/PD có mạch nguồn đã được xác nhận, giới hạn lab ở sink 5 V. Không yêu cầu điện áp cao khi chưa có power path đúng.
2. Cấu hình CC, dead-battery và role; bắt sự kiện attach/detach ở cả hai chiều cắm.
3. Nhận Source_Capabilities, parse PDO và ghi giá trị có đơn vị mV/mA. Chọn yêu cầu trong giới hạn phần cứng.
4. Log chuỗi capabilities/request/accept/PS_RDY theo trạng thái. Chỉ cho phép ứng dụng dùng công suất sau khi hợp đồng và nguồn hợp lệ.
5. Thử detach, retry, soft reset/hard reset bằng thiết bị test hoặc stack phù hợp. Xác nhận đầu ra về trạng thái đúng, buffer không bị giữ vĩnh viễn.
6. Kiểm duplicate Message ID, CRC error và timeout. Dùng PD analyzer; decoder UART không giải mã được đường CC.

**Đạt:** policy không yêu cầu vượt phần cứng, state machine phục hồi được, log phân biệt GoodCRC/Accept/PS_RDY. [D1 mục 3.40; R chương 62]


# Chủ đề 8 Lưu trữ bộ nhớ ngoài và luồng song song

Bài 32 đến Bài 35. Đọc lịch buổi học để chọn sản phẩm cần hoàn thành trước khi chuyển bài.


## Bài 32 Flash EEPROM emulation OTP và dữ liệu bền vững

### Cấu tạo và các phép thao tác

Flash là bộ nhớ không bay hơi nhưng không hoạt động như RAM. Program thường đưa các bit từ trạng thái đã erase sang trạng thái lập trình; muốn tái sử dụng cần erase theo sector. Trên vùng user flash thông thường H523, R mô tả program 128-bit và sector 8 KiB. OTP/high-cycle data có đơn vị riêng. Không áp đơn vị program 16-bit của vùng đặc biệt vào user flash.

Hai bank hỗ trợ read-while-write theo điều kiện, nhưng không cho phép mọi dạng đọc/ghi đồng thời tùy ý. Code, vector, constant và ISR nằm trong bank đang erase có thể bị ảnh hưởng. Từ khóa “dual bank” cũng không cho biết địa chỉ bank 2 hoặc dung lượng từng bank của mọi mã chip; tra memory map đúng xC/xE, bank swap và option hiện tại.

High-cycle data area phục vụ dữ liệu cần cập nhật nhiều hơn nhưng là cấu hình sử dụng vùng flash với cơ chế riêng, không phải RAM hay dung lượng thêm miễn phí. Datasheet nêu tối đa 48 KiB mỗi bank có khả năng 100kcycles trong điều kiện quy định; endurance còn liên quan nhiệt độ/retention. OTP chỉ được lập trình theo quy tắc một lần; không dùng OTP thật để thử một vòng ghi/xóa.

ECC sửa/phát hiện lỗi vật lý trong đơn vị bảo vệ nhưng không bảo đảm transaction ứng dụng đã hoàn tất. Ghi mất nguồn có thể để record dở dang; CRC, version và commit giúp nhận biết. Không ghi lại một phần của cùng flash word đã program nếu R không cho phép, kể cả trường commit nhỏ.

### Data structure cho journal cấu hình

| Trường logic | Kích thước gợi ý | Vai trò |
|---|---:|---|
| Magic |4 byte | Nhận diện format |
| Format version |2 byte | Chọn decoder |
| Payload length |2 byte | Kiểm giới hạn |
| Generation |4 byte | Chọn record mới |
| Payload CRC32 |4 byte | Phát hiện hỏng dữ liệu |
| Payload | Theo schema, làm tròn đơn vị ghi | Nội dung cấu hình |
| Commit record | Một đơn vị program riêng | Đánh dấu hoàn tất sau verify |

Dùng codec byte rõ ràng, không lưu thẳng struct có padding. Record mới được ghi vào vùng còn trống, verify, rồi ghi commit riêng. Boot chỉ chọn record có header, length, CRC và commit hợp lệ. Giữ record cũ đến khi record mới đã được xác nhận. Khi sector đầy, copy record còn sống sang sector dự phòng trước khi erase sector cũ. Chính sách generation wrap và lỗi hai bản đều invalid phải được định nghĩa.

Wear leveling phân tán số lần erase; ghi mỗi 1 ms vào cùng sector sẽ làm hỏng thiết kế dù thời gian write ban đầu ngắn. Chỉ lưu khi có thay đổi có ý nghĩa, gom cập nhật và đặt tuổi thọ dự kiến theo số lần erase xấu nhất.

### Lab 32 Cấu hình sống qua mất nguồn

1. Dành hai sector thử trong linker, bảo đảm không chứa code, image boot hoặc dữ liệu người dùng. Dùng địa chỉ lấy từ map thực, không sao chép một địa chỉ bất kỳ.
2. Implement read/append/scan bằng record format trên, có timeout, kiểm alignment, return code và error flags.
3. Ghi cấu hình A, reset, đọc lại. Ghi B và kiểm boot chọn B.
4. Dùng fault injection phần mềm dừng quy trình sau từng giai đoạn: erase, header, payload, verify, commit. Reset rồi kiểm phải nhận A hoặc B hợp lệ, không nhận dữ liệu nửa chừng.
5. Sau khi test logic, dùng fixture ngắt nguồn ở các thời điểm trên board thử riêng. Xử lý giới hạn flash/brownout và đọc errata trước khi kết luận.
6. Kiểm đầy journal, garbage collection, CRC sai, format không biết và generation wrap.

**Đạt:** không mất bản hợp lệ cuối cùng trong mô hình lỗi đã thử, có thống kê endurance và thời gian erase xấu nhất. **Bài tập:** chỉ ghi một biến `valid=1` vào cuối cùng có đủ không? Không nếu nằm trong flash word đã ghi, bị mất nguồn hoặc không có kiểm tra tính toàn vẹn của toàn record. [D1 mục 3.4,5.3.10; R chương 7]


## Bài 33 OCTOSPI FMC DLYB và bộ nhớ ngoài

### OCTOSPI và các mode

OCTOSPI giao tiếp serial memory bằng 1/2/4/8 đường data theo loại lệnh, hỗ trợ regular-command và HyperBus theo thiết bị. Indirect mode thực hiện command qua register/FIFO; auto-polling đọc status đến khi khớp mask; memory-mapped làm vùng ngoài xuất hiện trong address space để CPU/bus master truy cập. Memory-mapped không tự biến NOR flash thành RAM ghi từng byte tùy ý.

| Pha regular-command | Nội dung cần cấu hình |
|---|---|
| Instruction | Opcode và số đường truyền |
| Address | Địa chỉ, số bit và số đường |
| Alternate bytes | Mode bits nếu thiết bị cần |
| Dummy cycles | Thời gian chờ trước data, đơn vị cycle |
| Data | Chiều, độ dài, số đường, SDR/DTR |

Từng pha có thể dùng bus width khác nhau. Một cấu hình “1-4-4” nghĩa instruction/address/data dùng số đường tương ứng, không phải ba byte đặc biệt. SDR truyền một lần mỗi chu kỳ theo cạnh chọn; DTR dùng hai cạnh và yêu cầu timing chặt hơn. DQS hỗ trợ strobe khi có; HyperBus có command/address phase riêng, không dùng cùng opcode structure với NOR thông thường.

```c
typedef struct {
    uint32_t opcode;
    uint32_t address;
    uint32_t data_bytes;
    uint8_t instruction_lines;
    uint8_t address_lines;
    uint8_t data_lines;
    uint8_t address_bits;
    uint8_t dummy_cycles;
    bool dtr;
} serial_memory_command_t;
```

Đây là mô hình ứng dụng. Driver phải chuyển sang các field cụ thể của OCTOSPI và kiểm giới hạn. Serial NOR cần write-enable, page boundaries, busy polling, erase granularity và endurance theo chip ngoài; NAND cần ECC/bad-block management theo loại, không chỉ đổi opcode.

### DLYB và cửa lấy mẫu

DLYB điều chỉnh delay trên đường clock/data tương ứng để đưa điểm lấy mẫu vào giữa vùng dữ liệu ổn định. Calibration có thể quét tap, đọc pattern và tìm vùng pass liên tục, chọn điểm giữa với margin. Một tap chạy ở nhiệt độ phòng chưa chứng minh chạy ở nhiệt độ/điện áp khác. DLYB không sửa routing quá tệ hoặc clock vượt thông số.

### FMC và bus cycle song song

FMC trên H523 V/Z hỗ trợ loại bộ nhớ ngoài nêu trong D1 như SRAM/PSRAM/FRAM/NOR/NAND; không có SDRAM mode của các dòng khác trong R. Một giao dịch asynchronous có address, chip enable, read/write strobe, data và wait khi dùng. Timing gồm address setup/hold, data setup, bus turnaround; không có một header byte chung như UART.

Một access 16-bit có thể tương ứng một word trên bus; byte lane/NBL và CPU byte order cần được hiểu để đọc đúng thanh ghi LCD hoặc bộ nhớ. LCD dùng giao diện 8080 có thể nối qua FMC, với một address line chọn command/data. Đây là memory-mapped bus protocol của phần cứng ngoài, không phải bộ điều khiển LCD framebuffer chuyên dụng tự tạo hình.

### Lab 33 Kiểm bộ nhớ trước khi tăng tốc

1. H523C có thể dùng OCTOSPI với pin phù hợp; FMC cần package V/Z. Lập pin budget trước khi nối.
2. Dùng indirect mode tốc độ thấp để đọc ID/status. Không bắt đầu bằng XIP/memory-mapped khi chưa chứng minh command đúng.
3. Ghi/đọc pattern walking 1, walking 0, address-dependent và pseudorandom trong vùng thử. Giữ dữ liệu người dùng ngoài vùng test.
4. Kiểm page crossing, erase boundary, timeout khi thiết bị mất nguồn và write-protect.
5. Chuyển memory-mapped read; cấu hình MPU/cache và đo cold/warm throughput. Áp ownership/cache bài 18 khi DMA tham gia.
6. Tăng tốc từng bước, quét DLYB nếu áp dụng, lưu cửa pass và thử sau reset/nhiệt độ theo khả năng.
7. Với FMC, đo strobes và data setup/hold, thêm bus turnaround khi đổi chiều/device.

**Đạt:** không lỗi pattern ở toàn vùng thử, recovery từ timeout, biết giới hạn timing và nguồn. XIP nâng cao phải xét việc image update đang thay chính bộ nhớ chứa code đang chạy. [D1 mục 3.19–3.21; R chương 23–24,27,8–9]


## Bài 34 SDMMC SDIO và lưu file

### Phạm vi phần cứng

SDMMC có trên H523R/V/Z, không có trên H523C/H. Một board 48 chân vẫn có thể dùng thẻ SD ở SPI mode nếu thẻ/driver hỗ trợ, nhưng đó là bài SPI với protocol riêng, không phải bật SDMMC bằng phần mềm. H523 có một SDMMC theo D1, không gán SDMMC2 từ R cho chip này.

Khối SDMMC gồm command state machine, data path state machine, FIFO, clock generation, error/CRC/timeout và cơ chế DMA được triển khai. Bus native dùng CLK, CMD, DAT; card bắt đầu ở trạng thái/tốc độ khởi tạo phù hợp rồi mới chuyển wide bus/tốc độ. Clock nhanh chỉ được bật sau khi card và host thống nhất chế độ.

### Command response và data block

| Phần native SD/MMC | Cấu trúc |
|---|---|
| Command |48 bit: start 1, transmission 1, index 6, argument 32, CRC7, end 1 |
| Response | Loại ngắn hoặc dài theo command; không phải luôn cùng status layout |
| Data | Block trên DAT theo width 1/4/8 khi chuẩn/thiết bị cho phép |
| Data integrity | CRC theo đường data, kết thúc và busy theo giao dịch |

Byte/block addressing phụ thuộc loại card và trạng thái; không luôn gửi `LBA*512` cho mọi card. Nhiều thẻ hiện đại dùng logical block 512 byte nhưng ứng dụng vẫn phải lấy capacity và đặc tính qua quy trình init. Lệnh read/write block không tương đương file: filesystem ánh xạ cluster, directory và metadata lên block.

```c
typedef struct {
    uint32_t lba;
    uint32_t block_count;
    uint8_t *buffer;
    bool write;
    uint32_t deadline_ms;
} block_request_t;
```

API phải kiểm tích block_count×block_size không tràn, buffer đủ lớn/aligned, card present và lifetime. DMA complete không chắc card đã hoàn thành erase/program nội; cần kiểm trạng thái card và busy trước khi kết thúc giao dịch.

### Filesystem và mất nguồn

Filesystem như FAT/FileX/FatFs thêm cấu trúc metadata và caching. Ghi file thành công ở API có thể chỉ cập nhật buffer; cần hiểu flush/sync và card busy. Thẻ có controller riêng và wear leveling, nên latency write có đuôi dài. Tốc độ trung bình cao không bảo đảm không nghẽn 100 ms hoặc hơn trên thẻ cụ thể.

Datalogger nên dùng buffer RAM đủ cho stall đã đo, block write lớn, record có sequence/timestamp/CRC và chính sách flush. File preallocation có thể giảm biến thiên metadata nhưng không loại bỏ hết stall. Tránh mount cùng file system cho host USB MSC và MCU cùng ghi mà không cơ chế ownership.

### Lab 34 Datalogger có kiểm mất nguồn

1. Init card ở tốc độ phù hợp, đọc ID/capacity, kiểm loại addressing. Thử read block trước write.
2. Dành file thử, ghi record cố định với sequence và CRC. Đọc lại bằng PC, kiểm count/lỗi.
3. Đo phân bố latency write của ít nhất 1000 block, gồm max; dùng kết quả để chọn buffer, không chỉ lấy MB/s.
4. Thử card full, write-protect khi có, tháo card giữa operation và timeout. Driver trả lỗi, không vòng chờ vô hạn.
5. Test power cut trên card thử, kiểm mức mất data chấp nhận và khả năng mount lại. Giữ nguyên dữ liệu gốc ngoài vùng lab.
6. Với 48 chân, làm nhánh SPI-SD riêng và ghi rõ khác biệt command/response của SPI mode; không dùng trace native SD để đọc trace SPI.

**Đạt:** có ngân sách buffer, bounded timeout, recover và chính sách mất data. [D1 mục 3.37; R chương 26]


## Bài 35 DCMI PSSI và HDMI CEC

### DCMI và frame ảnh

DCMI nhận dữ liệu camera song song theo PIXCLK, HSYNC/VSYNC hoặc các mã đồng bộ được hỗ trợ. Nó có capture continuous/snapshot, width 8–14 bit theo cấu hình, cropping và FIFO/DMA. Có DCMI không có nghĩa có ISP/JPEG encoder; sensor phải xuất format mà ứng dụng hiểu. DCMI/PSSI chỉ có trên package R/V/Z.

Frame ảnh gồm nhiều line; mỗi line chứa pixel/byte theo format. RGB565 có 16 bit/pixel; thứ tự hai byte và thứ tự màu theo sensor. YUV422 có cách ghép chroma và luma; Bayer cần xử lý màu; JPEG là byte stream độ dài biến đổi. VSYNC đánh dấu frame, HSYNC/embedded code xác định line; chúng không phải magic byte ứng dụng tự chèn.

```c
typedef struct {
    uint16_t width;
    uint16_t height;
    uint32_t stride_bytes;
    uint32_t valid_bytes;
    uint32_t frame_id;
    uint32_t timestamp;
    uint8_t *pixels;
} image_buffer_t;
```

Frame QVGA 320×240 RGB565 chiếm 153600 byte; hai buffer chiếm 307200 byte, lớn hơn 272 KiB SRAM chính của H523 trước cả stack/ứng dụng. Cần giảm resolution/crop, xử lý theo line, dùng bộ nhớ ngoài hoặc thay strategy. DMA không tạo thêm RAM. FIFO overrun báo bus/buffer không theo kịp, phải đếm và bỏ frame có kiểm soát.

### PSSI và cấu trúc luồng song song

PSSI dùng chung circuitry, alternate functions và interrupt với DCMI; không dùng hai khối đồng thời. Nó là slave interface 8/16 bit input/output, external clock, FIFO 32 byte và DE/RDY tùy chọn. Edge capture/transmit, packing word 32 bit và handshake phải thống nhất hai đầu.

PSSI không quy định packet ứng dụng. Có thể định nghĩa luồng `{magic:u32, type:u16, count:u16, sequence:u32, payload, CRC32}` bằng byte order đã thống nhất rồi chuyển qua 8/16 bit bus. DE biểu thị data hợp lệ, RDY biểu thị receiver có thể nhận trong mode hỗ trợ. Phải kiểm errata trước chọn đồng thời DE/RDY ở output mode; không suy diễn mọi tổ hợp tín hiệu đều dùng được.

### HDMI CEC và frame điều khiển

CEC là đường điều khiển chậm một dây, open-drain, có arbitration và địa chỉ logic. Nó không truyền hình HDMI. Mạch ngoài cần pull-up/isolation theo R, đặc biệt khi thiết bị mất nguồn; không nối vào cổng HDMI mà bỏ qua bảo vệ điện.

| Trường CEC | Nội dung |
|---|---|
| Start | Xung start theo timing CEC |
| Header | Nibble initiator 4 bit +destination 4 bit |
| Opcode | Tùy chọn, mô tả lệnh |
| Operands |0–14 byte theo opcode |
| Mỗi block |8 bit MSB-first +EOM +ACK |

Tối đa 16 byte gồm header/opcode/operands. EOM đánh dấu block cuối. Unicast ACK và broadcast feedback có ý nghĩa khác; broadcast address 0xF không được kiểm theo cùng quy tắc ACK unicast. Logical address 4 bit khác physical address mô tả vị trí thiết bị trong topology HDMI. CEC không có CRC payload chung để ứng dụng tự tính như CAN.

```c
typedef struct {
    uint8_t initiator;
    uint8_t destination;
    uint8_t opcode;
    uint8_t operand_count;
    uint8_t operand[14];
    bool polling_only;
} cec_message_t;
```

Polling message có thể chỉ header, nên không ép mọi frame phải có opcode. Driver kiểm maximum length và state address allocation, xử lý arbitration lost, retry, error và wakeup theo R.

### Lab 35 Ba nhánh thực hành

1. **Camera:** cấu hình sensor qua I2C, đọc ID, chọn resolution nhỏ; đo PIXCLK/VSYNC/HSYNC; capture một frame, xuất file ảnh để kiểm stride, màu, line count. Chạy capture liên tục và đếm overrun.
2. **PSSI:** dùng FPGA/MCU nguồn clock, truyền counter tăng dần bằng 8 bit rồi 16 bit; xác minh packing và DE/RDY; ngừng consumer để test flow control. Ghi spec payload và CRC nếu ứng dụng cần.
3. **CEC:** dùng node/adapter CEC thử, cấp địa chỉ đúng, phát polling và một message chuẩn được thiết bị test hỗ trợ. Bắt header, EOM, ACK; thử hai initiator để quan sát arbitration. Không gửi lệnh ảnh hưởng thiết bị người khác trên bus đang dùng.

**Đạt:** biết framing của từng giao tiếp, memory budget và error path. [D1 mục 3.26–3.27,3.41; R chương 33–34,65]


# Chủ đề 9 Low power RTOS kiến trúc và debug

Bài 36 đến Bài 40. Đọc lịch buổi học để chọn sản phẩm cần hoàn thành trước khi chuyển bài.


## Bài 36 Low power và thiết kế năng lượng

### Chọn trạng thái theo công việc

Run thực thi bình thường. Sleep dừng clock CPU nhưng nhiều ngoại vi có thể tiếp tục; interrupt/event đánh thức CPU. Stop dừng nhiều clock của core domain, giữ trạng thái theo cấu hình retention; một số nguồn và ngoại vi thấp năng lượng có thể hoạt động/đánh thức theo bảng R. Standby tắt core domain, mất phần lớn SRAM/register ngoài vùng được duy trì và đi qua quá trình boot khi thức lại.

Không nhập danh sách Stop 0/1/2/3 từ một dòng STM32 khác vào H523. Tên mode, điều kiện regulator, SRAM retention và wakeup phải theo chương PWR của chip. Ngoại vi tồn tại trong chip không có nghĩa nó hoạt động trong mọi mode.

| Câu hỏi trước khi ngủ | Ví dụ quyết định |
|---|---|
| Cần giữ dữ liệu nào | Buffer đo, trạng thái protocol, RTC |
| Nguồn nào đánh thức | EXTI, RTC, LPTIM, UART/I2C trong mode hỗ trợ |
| Cần clock nào tiếp tục | LSE/LSI hoặc kernel clock được cho phép |
| Deadline sau thức | Khôi phục PLL, sensor warmup, giao tiếp |
| Đầu ra phải thế nào | Giữ LED, ngắt motor, điều khiển power enable |

### Đo năng lượng và tránh wakeup giả

Năng lượng một chu kỳ bằng tích phân V×I theo thời gian. Dòng trung bình có thể ước lượng bằng tổng I_i×t_i chia tổng thời gian khi điện áp gần cố định. Nên đo cả board và tách phần MCU khi có điểm đo: LED nguồn, USB-UART, debugger, regulator và pull-up đều tiêu thụ điện.

`WFI` chờ interrupt; `WFE` liên quan event register. Pending interrupt hoặc event cũ có thể làm CPU thức ngay. Có race giữa kiểm “không có việc” và đi ngủ; dùng quy trình được RTOS/kiến trúc hỗ trợ, không tự chèn WFI sau một if mà bỏ qua công việc vừa đến.

Trước Stop cần quiesce luồng không thể duy trì, bảo đảm DMA không còn dùng vùng sẽ tắt và thống nhất ownership. Sau Stop, SYSCLK có thể về HSI/CSI theo cấu hình; phải khôi phục clock/timebase và ngoại vi phụ thuộc trước khi báo hệ thống READY.

### Lab 36 Ngủ giữa hai lần đo

1. Đặt yêu cầu: mỗi giây đo cảm biến một lần, gửi một packet, thời gian phản hồi nút dưới 50 ms.
2. Đo baseline Run bằng nguồn đo dòng; tháo các LED/debugger phụ khi muốn đánh giá riêng MCU.
3. Dùng Sleep chờ interrupt, giữ timer thích hợp. Đo lại năng lượng mỗi chu kỳ và latency.
4. Chuyển sang Stop với RTC/LPTIM wakeup được hỗ trợ; dừng tick gây thức định kỳ nếu không cần, phục hồi clock sau wakeup.
5. Thử nhiều lần ngủ/thức, pending EXTI, UART dữ liệu đến ngay trước khi ngủ và sensor mất nguồn.
6. Thử Standby trong một project riêng với checkpoint backup và xác nhận boot path, không mong mọi biến RAM còn nguyên.

**Đạt:** có biểu đồ dòng theo thời gian, wakeup reason, latency và dữ liệu không hỏng sau ít nhất 1000 chu kỳ thử. **Bài tập:** dòng Sleep thấp hơn Run nhưng thời gian hoạt động gấp ba có thể làm năng lượng cao hơn; hãy tính bằng số đo. [D1 mục 3.10,5.3.6; R chương 10–12,48,51]


## Bài 37 Đồng bộ dữ liệu và mô hình bộ nhớ của C

### Race condition trên một lõi

Một CPU vẫn có concurrency vì ISR ngắt task, scheduler chuyển task và DMA chạy độc lập. `counter++` có thể gồm load/add/store; ISR chen vào giữa làm mất increment. Cấu trúc nhiều field có thể bị đọc trong lúc writer mới cập nhật một phần. `volatile` buộc một số access diễn ra nhưng không tạo transaction dữ liệu.

Critical section giữ đoạn cập nhật ngắn không bị tác nhân liên quan chen vào; phải lưu và khôi phục trạng thái mask cũ, không unconditionally enable interrupt khi caller đã mask từ trước. RTOS mutex phục vụ task và có thể block; không dùng mutex blocking trong ISR. Atomic operation phù hợp khi thao tác nhỏ và implementation lock-free ở môi trường đó.

### SPSC queue mẫu

SPSC nghĩa đúng một producer và một consumer. Ring dùng một ô trống để phân biệt full/empty. C11 acquire/release công bố dữ liệu theo thứ tự hợp lệ, nhưng phải xác nhận atomics trên target/ABI là lock-free trước khi dùng từ ISR. Nếu không, chọn primitive của RTOS hoặc critical section ngắn phù hợp.

```c
#include <stdatomic.h>
#include <stdbool.h>
#include <stdint.h>
#define Q_SIZE 16u

typedef struct {
    uint32_t item[Q_SIZE];
    _Atomic unsigned head;
    _Atomic unsigned tail;
} spsc_u32_t;

static bool q_push(spsc_u32_t *q, uint32_t v)
{
    unsigned h = atomic_load_explicit(&q->head,
                                      memory_order_relaxed);
    unsigned next = (h + 1u) % Q_SIZE;
    unsigned t = atomic_load_explicit(&q->tail,
                                      memory_order_acquire);
    if (next == t) return false;
    q->item[h] = v;
    atomic_store_explicit(&q->head, next,
                          memory_order_release);
    return true;
}

static bool q_pop(spsc_u32_t *q, uint32_t *v)
{
    unsigned t = atomic_load_explicit(&q->tail,
                                      memory_order_relaxed);
    unsigned h = atomic_load_explicit(&q->head,
                                      memory_order_acquire);
    if (t == h) return false;
    *v = q->item[t];
    atomic_store_explicit(&q->tail, (t + 1u) % Q_SIZE,
                          memory_order_release);
    return true;
}
```

Khởi tạo atomics về 0 trước khi enable producer. Queue trên chứa giá trị, không đảm bảo lifetime của một pointer được ép thành số. Hai ISR cùng push biến nó thành multi-producer và phá giả thiết. Queue full phải trả về chính sách có thống kê; không spin chờ trong ISR vì consumer có thể không được chạy.

### Lab 37 Phát hiện lỗi mất sự kiện

1. Viết producer tăng sequence trong ISR và consumer trong main. Bản đầu dùng boolean, cho producer nhanh hơn consumer và quan sát gap.
2. Thay bằng counter hoặc SPSC đúng mô hình; kiểm full và tổng produced/consumed/dropped.
3. Kiểm `atomic_is_lock_free` cho kiểu sử dụng trên build đích, xem disassembly để hiểu primitive compiler sinh ra.
4. Thử wrap head/tail nhiều lần. Thêm timestamp để đo latency queue.
5. Tạo một cấu trúc `{sequence,value,crc}` cập nhật không khóa, quan sát snapshot không nhất quán; sửa bằng ownership hoặc snapshot có cơ chế đồng bộ.

**Đạt:** có invariant kiểm chứng, không dùng delay để “chữa race”. **Bài tập:** thêm `volatile` vào toàn struct không bảo đảm ba field thuộc cùng một lần cập nhật. [P mục 2–3 về memory access/exclusive/barrier; phần C11 là kiến thức ngôn ngữ bổ sung]


## Bài 38 RTOS scheduling deadline và priority inversion

### Task và scheduler

RTOS quản lý các task có stack/context riêng, trạng thái ready/running/blocked/suspended và priority. Context switch lưu/khôi phục trạng thái để chia sẻ một CPU. Task độc lập về stack không tự độc lập về quyền bộ nhớ. RTOS thông thường chạy mọi task privileged vẫn có thể bị một con trỏ lỗi phá toàn hệ thống.

Preemptive scheduling cho task ưu tiên cao chạy khi ready. Time slicing phân chia giữa task cùng priority theo cấu hình; không thay thiết kế deadline. Delay tương đối lặp có thể trôi chu kỳ do cộng thêm thời gian xử lý; delay-until theo mốc tuyệt đối phù hợp tác vụ định kỳ hơn. Tick không phải độ chính xác cao nhất của chip; capture/trigger timer vẫn có vai trò riêng.

Semaphore thông báo số lượng/sự kiện; mutex bảo vệ tài nguyên và có owner; queue truyền bản sao message; event flags gộp trạng thái; task notification thường nhẹ cho một số pattern. Chọn theo ý nghĩa dữ liệu, không dùng một primitive duy nhất cho mọi bài toán.

### Tính thời gian đáp ứng

Đối với mô hình task định kỳ fixed-priority đơn giản, response time có thể ước lượng lặp:

`R_i = C_i + B_i + sum(ceil(R_i/T_j) * C_j)` với j là task ưu tiên cao hơn.

C là worst-case execution time đã có cơ sở; B là blocking; T là period. Công thức có giả thiết và không tự bao phủ mọi DMA/bus/cache/security interference. Nếu chỉ lấy thời gian trung bình làm C, kết quả có thể lạc quan sai.

Priority inversion xảy ra khi task cao chờ mutex do task thấp giữ, trong lúc task trung bình chiếm CPU. Priority inheritance giảm một số trường hợp nhưng không sửa deadlock hoặc critical section quá dài. Quy định thứ tự lấy lock, không giữ lock qua I/O blocking dài và không gọi callback ngoài khi đang giữ lock nội nếu chưa có hợp đồng rõ.

### Lab 38 Ba task có deadline

1. Tạo task acquisition xử lý block mỗi 10 ms, protocol xử lý packet, storage ghi nền. Ghi rõ priority, stack và thời gian xử lý dự kiến.
2. ISR chỉ giao descriptor; task acquisition xử lý và trả buffer trước deadline. Dùng primitive FromISR của RTOS khi được phép, với IRQ priority hợp quy định port.
3. Đo stack high-water mark và response time khi protocol bị flood và storage chậm.
4. Tạo priority inversion có kiểm soát bằng một mutex, đo có/không inheritance trong cấu hình cho phép.
5. Thử task không yield, queue full và allocation failure. Supervisor phải phát hiện mất tiến độ thật.
6. Dùng memory pool/buffer cố định cho đường realtime để tránh biến thiên heap nếu yêu cầu chặt.

**Đạt:** tất cả deadline có phép đo dưới tải xấu nhất đã định nghĩa; không tăng mọi task lên priority tối đa. **Bài tập:** một task ưu tiên cao chạy vô hạn có thể làm watchdog task thấp không được chạy, nhưng IRQ điều khiển cao hơn vẫn có thể hoạt động; cần phân tích từng đường điều khiển. [P exception/SysTick/PendSV; R chương 17; RTOS là lớp phần mềm bổ sung]


## Bài 39 Kiến trúc firmware phân lớp và cô lập lỗi

### Trách nhiệm và giao diện

BSP biết pin và board revision; driver biết peripheral; service quản lý một chức năng có state/ownership; application diễn đạt nghiệp vụ; supervisor theo dõi sức khỏe và recovery. Một service vẫn có API để gọi; “service” và “API” không loại trừ nhau. Khác biệt có ích là ai sở hữu trạng thái, ai quyết định thời điểm làm việc và caller được phép tác động tới đâu.

Ví dụ service motor sở hữu timer PWM, sensor ADC/DMA, tham số đã kiểm tra và state machine. App gửi yêu cầu tốc độ có giới hạn; service quyết định chấp nhận, ramp và xử lý fault. App không nhận pointer tới cấu trúc mutable nội bộ của service.

```c
typedef struct {
    uint32_t request_id;
    int32_t target_rpm;
    uint32_t deadline_ms;
} motor_request_t;

typedef struct {
    uint32_t sequence;
    int32_t actual_rpm;
    uint32_t fault_flags;
    uint8_t state;
} motor_snapshot_t;
```

API trả snapshot copy hoặc immutable view có lifetime rõ. Request phải kiểm range, state, deadline, version và caller permission ở boundary; không tin chỉ vì request đến từ module cùng firmware.

### Một image hay hai image

| Phương án | Lợi ích | Hạn chế |
|---|---|---|
| Một image, module rõ | Build/debug đơn giản, ABI thống nhất | Không có cô lập phần cứng nếu cùng quyền |
| Một image, task unprivileged và MPU | Giới hạn CPU access giữa task/vùng | Cần RTOS port, SVC và kiểm DMA/peripheral |
| Secure và Non-secure image | Phân miền bằng TrustZone, cập nhật theo hợp đồng | Chung CPU/bus/reset; phải quản lý veneer/version |
| Hai MCU | Tách nhiều tài nguyên và fault domain hơn | Tăng chi phí, protocol và đồng bộ |

Hai partition flash chỉ là bố trí lưu trữ. Muốn app lỗi không phá dữ liệu control cần quyền phần cứng; muốn app không làm trễ control còn cần ưu tiên, budget, bus ownership và hạn chế thời gian service call. Muốn app lỗi không gây reset control phải xét HardFault/lockup/watchdog/power và yêu cầu vật lý. Không có cấu hình MPU/TrustZone nào biến mọi lỗi trên một chip thành vô hại.

### Lab 39 Service điều khiển giả lập

1. Chưa nối động cơ; service phát PWM vào scope, sensor lấy từ DAC/nguồn thử. App chỉ gửi setpoint qua queue/API.
2. Tách header public khỏi dữ liệu private, mọi mutable state của service nằm trong module riêng; kiểm map để chuẩn bị MPU.
3. Kiểm request âm/quá lớn/hết hạn/trùng ID/state sai; xác nhận không tạo duty ngoài giới hạn.
4. App cố viết vùng state qua con trỏ sai trong build thử có MPU/TrustZone ở bài 41–43; kỳ vọng fault bị ghi nhận.
5. Flood request và giữ app bận; đo control period, jitter, queue pressure. Áp rate limit và budget.
6. Thiết kế recovery app: dừng request mới, thu hồi tài nguyên app, khởi tạo lại context hoặc reset có kiểm soát theo lỗi. Không chỉ nhảy về đầu task hỏng mà giữ lock/DMA cũ.

**Đạt:** có dependency diagram, hợp đồng API, ownership table và bằng chứng fault injection. [R chương 3,5,16; P privilege/MPU/SAU; kiến trúc biên soạn]


## Bài 40 Debug fault trace và tối ưu có bằng chứng

### Quan sát từ ngoài vào trong

Bắt đầu bằng triệu chứng đo được, điều kiện tái hiện và khoảng thời gian lỗi. Kiểm nguồn/clock/reset/pin trước khi đoán driver. Dùng log có timestamp và sequence; logic analyzer cho protocol, scope cho điện/timing, debugger cho state và trace cho đường thực thi. Printf có thể làm mất lỗi timing hoặc tạo lỗi mới.

SWD/JTAG truy cập debug theo quyền; breakpoint phần cứng hữu hạn. Watchpoint giúp bắt ghi vào địa chỉ; trace SWO/ITM/ETM phụ thuộc pin, probe, quyền và cấu hình. Không hứa mọi board 48 chân có đủ trace parallel chỉ vì CPU có ETM. Debug freeze làm watchdog/timer hành xử khác chế độ standalone.

DWT cycle counter nếu có và được cấp quyền hỗ trợ đo cycle. Phải tính wrap, tần số và overhead; công việc có thể bị compiler bỏ nếu kết quả không dùng. Với thời gian thật qua đổi clock/low-power, timer/RTC thích hợp hơn việc nhân cycle với một tần số cố định.

### Chẩn đoán exception

HardFault có thể là escalation từ fault khác. Đọc CFSR/HFSR, MMFAR/BFAR khi valid, SecureFault status khi được quyền, stacked PC/LR/xPSR và trạng thái stack. EXC_RETURN giúp xác định stack/frame, nhưng FPU, security state và lỗi stacking làm một handler cố đọc 8word cố định có thể tiếp tục fault.

Fault record nên tối thiểu: version, reset/fault reason, timestamp/boot count, PC/LR/SP, status registers, CRC. Đặt vào vùng an toàn còn truy cập được; không format chuỗi dài hoặc gọi filesystem khi trạng thái kernel/heap không tin cậy. Thu log sau reset. Không tự tăng PC để bỏ qua mọi instruction fault rồi tiếp tục điều khiển như bình thường.

### Lab 40 Ba lỗi có thể tái hiện

1. Tạo build thử riêng với MemManage enabled; truy cập vùng MPU cấm để thu fault có chủ đích.
2. Tạo stack overflow có kiểm soát trong task thử, dùng guard/stack limit theo port. Không để thực nghiệm tác động power stage.
3. Tạo UART/DMA overrun rồi ghi reason code, so với fault CPU để thấy đây là hai loại lỗi khác nhau.
4. Dùng map/addr 2line của đúng ELF để giải mã PC. ELF khác build có thể cho source line sai.
5. Benchmark một hàm xử lý mẫu với compiler flags, cache cold/warm và interrupt load; chỉ tối ưu phần có đóng góp đáng kể.
6. Giữ test vector và đo trước/sau để bảo đảm tối ưu không đổi kết quả ngoài sai số đã cho phép.

**Đạt:** báo cáo nguyên nhân gốc, cách tái hiện, bản sửa và phép thử ngăn tái phát. **Bài tập:** mất log cuối cùng không chứng minh code chưa tới đó; buffer log có thể chưa được truyền khi reset. [D1 mục 3.42; R chương 66; P exception/fault/debug-related core registers]


# Chủ đề 10 Cô lập bộ nhớ và ngoại vi

Bài 41 đến Bài 43. Đọc lịch buổi học để chọn sản phẩm cần hoàn thành trước khi chuyển bài.


## Bài 41 Privilege MPU và SVC

### Ba vấn đề khác nhau

Privilege xác định mã có quyền thực hiện thao tác hệ thống nào. MPU kiểm CPU access theo vùng và thuộc tính. SVC là exception có chủ đích để mã yêu cầu dịch vụ đặc quyền. Chia module C chỉ hạn chế cách lập trình, còn ba cơ chế này giúp thực thi một phần ranh giới bằng phần cứng và kernel.

Thread privileged có thể tự hạ quyền qua CONTROL theo quy tắc; Thread unprivileged không tự ghi lại một bit để trở thành privileged. Nó phải đi qua exception/service được cho phép. Handler mode chạy privileged, vì vậy mọi handler có lỗi đều là một phần của trusted computing base cần được xem xét.

MPU Armv8-M trên M33 dùng base/limit và memory attributes qua MAIR, khác MPU region power-of-two/subregion của một số Cortex-M cũ. Base/limit có granularity 32 byte. Vùng chồng lấn có thể tạo fault khi access chạm phần overlap, không áp quy tắc “region số lớn thắng” từ kiến trúc khác. Kiểm số region triển khai từ device/MPU_TYPE; D1 mô tả khả năng MPU cho Secure và Non-secure.

### Thiết kế vùng

| Vùng | Quyền ứng dụng gợi ý | Thuộc tính |
|---|---|---|
| Code app | Read/execute | Không write |
| Const | Read-only | Execute-never khi không có code |
| Stack/heap app | Read/write | Execute-never |
| Service private | Không truy cập hoặc read snapshot riêng | Không cho app write |
| MMIO ngoại vi | Chỉ phần được cấp, hoặc qua service | Device memory, execute-never |
| Guard | Không access | Bắt một số lỗi vượt vùng |

MPU không tự kiểm giá trị nghiệp vụ: app có quyền ghi setpoint buffer vẫn có thể ghi giá trị nguy hiểm. Nó cũng không trực tiếp chặn mọi DMA. Background map cho privileged và các exception policy phải được tính trong threat model; một region đẹp trên giấy có thể bị bỏ qua nếu app vẫn privileged và tự cấu hình lại MPU.

### SVC là một cổng dịch vụ

Wrapper đặt service ID và tham số theo ABI, thực hiện SVC; handler xác định caller/context, kiểm tham số rồi thực hiện hoặc giao việc. Trong Non-secure, SVC không tự biến thành gọi Secure. Secure entry qua TrustZone có cơ chế riêng ở bài 42.

Không tin pointer do caller đưa. Kiểm range, overflow, quyền đọc/ghi và lifetime. Với buffer mutable, sao chép vào vùng tin cậy sau khi kiểm hoặc dùng hợp đồng ownership để tránh TOCTOU. Không giữ con trỏ stack caller sau khi trả về nếu dịch vụ chạy bất đồng bộ.

### Lab 41 App không sửa được trạng thái service

1. Tạo vùng RAM riêng cho app và service bằng linker. Xác nhận alignment 32 byte và không overlap ngoài ý muốn.
2. Cấu hình MPU trước khi hạ app xuống unprivileged; vẫn giữ đường handler/kernel hợp lệ và stack đúng.
3. App gọi dịch vụ đọc snapshot qua SVC, nhận bản sao dữ liệu. Kiểm service ID không hợp lệ, buffer quá dài và pointer ngoài vùng.
4. Trong build thử, app ghi trực tiếp service state: kỳ vọng MemManage/fault được ghi nhận, không thay đổi giá trị service.
5. Cho DMA app thử truy cập cùng vùng để chứng minh MPU CPU chưa đủ; bài 43 sẽ bổ sung quyền bus.
6. Đo thời gian SVC và hạn chế lượng công việc chạy privileged; không thực hiện ghi file dài trong một syscall giữ interrupt mask.

**Đạt:** có ma trận allowed/denied và phép thử cả hai phía. **Bài tập:** `static const` và MPU read-only giống nhau không? Không; một cái là quy tắc/ngữ nghĩa phần mềm, một cái là kiểm truy cập CPU ở runtime. [P mục 2 và 4.5; D1 mục 3.3]


## Bài 42 TrustZone SAU IDAU và Secure gateway

### Hai miền bảo mật trên một lõi

TrustZone-M phân chia tài nguyên và trạng thái thực thi thành Secure/Non-secure. SAU do phần mềm Secure cấu hình; IDAU do implementation quy định. Kết quả attribution còn phối hợp với bảo vệ bộ nhớ/ngoại vi phía hệ thống, không chỉ là một bảng địa chỉ trong CPU. H523 hỗ trợ tối đa 8 SAU region theo D1.

Non-secure Callable là vùng Secure cho phép vào qua Secure Gateway hợp lệ. Veneer có instruction SG và chuyển điều khiển tới dịch vụ Secure. Không đánh dấu toàn bộ Secure flash là NSC để “gọi cho tiện”; làm vậy mở rộng mặt tấn công và phá mục đích ranh giới.

Secure có register/stack/vector theo quy tắc kiến trúc; Non-secure có context tương ứng. Secure gọi sang Non-secure dùng cơ chế kiểm/chuyển miền đúng compiler/CMSIS. Con trỏ hàm giữa miền không thể xử lý như mọi pointer C thông thường. FPU context và dữ liệu trong register cũng cần được quản lý khi crossing boundary.

### Giao diện Secure nhỏ và hữu hạn

Một API tốt có input rõ, giới hạn length, operation ID và mã lỗi; không cho Non-secure chọn địa chỉ peripheral tùy ý hoặc thực thi function pointer tùy ý bên Secure. CMSE/compiler hỗ trợ kiểm thuộc tính pointer nhưng developer vẫn phải kiểm overflow, quyền, lifetime và tính nhất quán dữ liệu.

Ví dụ service `control_set_target(request)` nhận bản request nhỏ, kiểm min/max/rate/state, sao chép rồi trả request ID. Không gọi control loop dài ngay trong gateway. Nếu Non-secure flood gateway, Secure vẫn tốn CPU; cần rate limit/budget và thiết kế preemption. TrustZone bảo vệ quyền, không tự đảm bảo availability hay deadline.

### Lab 42 Hai image với một service

1. Tạo project TrustZone đúng H523 theo tool hỗ trợ. Ghi boot state/option hiện tại; không thay lifecycle bằng các giá trị chép từ video dòng khác.
2. Lập map Secure flash/RAM, NSC, Non-secure flash/RAM và các vùng shared nếu có. Tính ngân sách cho chip 256 KiB trước khi thêm middleware.
3. Tạo một Secure service trả kết quả tính nhỏ từ input đã kiểm. Build Secure, sinh import/veneer artifact và build Non-secure với đúng version.
4. Start từ boot Secure, cấu hình attribution/stack/vector, kiểm entry Non-secure rồi chuyển bằng cách chuẩn của toolchain.
5. Từ Non-secure gọi gateway hợp lệ; thử nhảy vào một địa chỉ Secure không phải gateway trong build fault-test và xác nhận bị chặn.
6. Thử buffer NS sai range/length/tràn địa chỉ; service trả lỗi, không đọc tràn vùng Secure. Thử NS sửa buffer trong khi service xử lý để đánh giá TOCTOU.
7. Log fault ở Secure và phản ứng hệ thống; chỉ reset/restart miền ứng dụng khi có thiết kế phục hồi tài nguyên đầy đủ.

**Đạt:** hai image có version tương thích, mọi vùng được giải thích và boundary được thử. **Bài tập:** “Secure luôn chạy trước mọi Non-secure interrupt” là phát biểu không đủ; cần xét priority, security configuration, masking và cơ chế exception của core. [R chương 3–5; P security extension/SAU; D1 mục 3.6–3.9]


## Bài 43 GTZC SBS quyền ngoại vi và DMA

### CPU attribution chưa phải toàn hệ thống

GTZC phân loại tài nguyên theo Secure/Non-secure và privilege ở các khối được hỗ trợ. TZSC quản lý nhiều thuộc tính peripheral; MPCBB bảo vệ các block SRAM; MPCWM hoặc cơ chế watermark bảo vệ các vùng phù hợp. TZIC báo illegal access theo khả năng. Một số peripheral TrustZone-aware có security register nội; cấu hình không chỉ nằm tại một thanh ghi GTZC chung.

SBS quản lý các chức năng system configuration, boot và security liên quan như hide protection/debug/option interface được mô tả trong R. RCC, PWR, GPIO, EXTI và GPDMA có các khía cạnh quyền riêng. Clock/reset của peripheral được bảo vệ cũng phải thuộc miền đúng; nếu app không sửa register timer nhưng vẫn tắt clock/reset timer thì mục tiêu cô lập chưa đạt.

### Ma trận ownership mẫu

| Tài nguyên | Owner | App được làm gì |
|---|---|---|
| TIM1 PWM và break config | Control Secure | Gửi setpoint đã kiểm |
| ADC1 và DMA lấy mẫu control | Control Secure | Nhận snapshot/copy |
| Buffer control | Secure RAM | Không direct access |
| UART telemetry | Non-secure hoặc service broker | Truyền packet trong budget |
| Shared mailbox | Quyền được thiết kế rõ | Chỉ request/response đã serialize |
| RCC/PWR liên quan | Secure quản lý | Yêu cầu mode, không tự thay |
| Watchdog policy | Supervisor | Heartbeat tiến độ |

GPDMA có channel security và các thuộc tính source/destination/link; descriptor ở Secure nhưng channel Non-secure hoặc link attribute sai vẫn có thể lỗi. Ngược lại, cho app cấu hình một Secure DMA mạnh có thể tạo đường vượt ranh giới dù app không trực tiếp đọc Secure RAM. Quyền điều khiển master phải được bảo vệ cùng quyền vùng đích.

Ngoại vi có message RAM/FIFO nội như FDCAN cũng cần xét riêng, không mặc định classification register đã bảo vệ mọi vùng phụ. Interrupt target state và handler phải cùng thiết kế ownership; không để một Non-secure ISR quản lý flags của resource mà nó không được truy cập.

### Lab 43 Kiểm quyền theo đường truy cập

1. Vẽ tất cả master: CPU-S, CPU-NS, GPDMA1/2 và các bus master ngoại vi thực sự dùng. Vẽ các vùng đích từ linker/memory map.
2. Cấp một peripheral cho Secure, một cho Non-secure; cấu hình GPIO, clock, IRQ và buffer tương ứng. Chạy phép truy cập hợp lệ để tránh “bảo vệ” bằng cách làm mọi thứ ngừng hoạt động.
3. NS thử đọc/ghi register Secure, thay clock/reset, sửa DMA descriptor và ghi Secure RAM. Ghi fault/illegal access flag cùng dữ liệu trước/sau.
4. NS DMA thử sao chép từ/vào vùng Secure theo cấu hình thử; chứng minh policy chặn chứ không chỉ CPU bị MPU chặn.
5. Cấp quyền shared mailbox nhỏ, kiểm NS không vượt length để đụng vùng bên cạnh. Không dùng buffer chung làm nơi chứa secret.
6. Đo jitter control khi NS chạy DMA tải lớn. Nếu bus contention làm trễ deadline, điều chỉnh budget/arbitration/bố trí RAM hoặc tách phần cứng theo yêu cầu.

**Đạt:** có bảng từng access allowed/denied và kết quả thử; kiểm cả data integrity lẫn timing. **Bài tập:** vì sao cấp quyền read-only snapshot qua CPU vẫn chưa đủ nếu NS có DMA ghi được vào snapshot? Vì access control phải áp dụng cho mọi master liên quan. [D1 mục 3.8–3.9; R chương 3,5,13–18]


# Chủ đề 11 Mật mã boot bảo mật kiểm thử và sản xuất

Bài 44 đến Bài 48. Đọc lịch buổi học để chọn sản phẩm cần hoàn thành trước khi chuyển bài.


## Bài 44 CRC RNG HASH PKA và quản lý khóa

### Phân biệt các mục đích

| Cơ chế | Giải quyết | Không giải quyết |
|---|---|---|
| CRC | Phát hiện nhiều kiểu lỗi truyền/lưu ngẫu nhiên | Chống người chủ động sửa và tính lại CRC |
| Hash | Tóm tắt dữ liệu, phát hiện thay đổi khi digest tin cậy | Xác thực nguồn nếu digest cũng bị thay |
| MAC | Xác thực/integrity bằng khóa bí mật chung | Phân phối khóa tự động |
| Chữ ký số | Xác minh bằng public key, ký bằng private key | Giữ bí mật payload |
| Mã hóa có xác thực | Bí mật và integrity theo scheme | Cứu khóa/nonce quản lý sai |
| RNG | Entropy cho khóa/nonce/challenge phù hợp | Tự bảo vệ nơi lưu khóa |

CRC hardware H523 có polynomial/độ rộng và các tùy chọn đảo dữ liệu theo R. Phải khớp init, reflection, xorout, byte order và cách feed byte cuối với chuẩn. Chạy vector đã biết trước khi tăng tốc DMA. CRC hardware của USB/CAN ở giao thức riêng không đồng nghĩa CRC unit chung luôn tự truy cập/kiểm mọi frame.

RNG có nguồn entropy, conditioning và health status. Khi báo lỗi entropy/clock, không tiếp tục dùng output như số ngẫu nhiên tin cậy. Không seed khóa bằng UID/tick/time. UID 96-bit là định danh, không phải bí mật.

HASH của H523 tăng tốc SHA-1 và SHA-2 theo bảng triển khai của R mục 40.3, bảng 397. H523 không có tăng tốc SHA-3; trong bảng này SHA-3 thuộc H543/H553. Trang tổng quan D1 có dòng SHA-3 nhưng không nhất quán với phần HASH chi tiết và bảng triển khai R; khi chọn thuật toán phần cứng, dùng phạm vi H523 được xác định trong bảng triển khai. Thiết kế mới dùng thuật toán còn phù hợp với mục đích; không chọn SHA-1 cho chữ ký mới chỉ vì phần cứng hỗ trợ. Xử lý stream cần giữ state, total length, padding và byte order đúng. HMAC là cấu trúc dùng hash với khóa, phải kiểm driver/library và mode cụ thể thay vì tự ghép `hash(key||data)`.

### Giới hạn PKA của H523

D1 xác định PKA trên H523 hỗ trợ **xác minh chữ ký ECDSA**. Các phép RSA, DH, ECC signing hoặc toàn bộ primitive ở chương PKA chung không được tự gán cho H523. D2 H533 có các khả năng mật mã bổ sung như AES/SAES/OTFDEC và PKA rộng hơn. H523 muốn AES/TLS hoặc operation ngoài hardware phải chọn thư viện phần mềm được hỗ trợ và tính CPU/RAM/side-channel theo yêu cầu.

Xác minh ECDSA cần public key hợp lệ, curve/domain đúng, digest và signature đúng encoding. DER và raw r||s là hai định dạng khác. Lỗi endianness, độ dài field hoặc dẫn zero có thể làm verify fail. Không “sửa” bằng bỏ qua return code hoặc chấp nhận image khi PKA báo lỗi.

### Lab 44 Known answer và negative tests

1. CRC unit: tính vector“123456789” với đúng variant, so với implementation phần mềm; thử length 1,3,4,5 để kiểm byte cuối.
2. HASH: chạy kết quả đã biết cho empty, ASCII abc và buffer lớn chia theo nhiều chunk. Hash cùng dữ liệu phải giống nhau dù chunk boundary khác.
3. RNG: đọc một lượng mẫu, kiểm return/status và đường xử lý lỗi. Thống kê byte/histogram chỉ là smoke test, không chứng minh entropy cryptographic.
4. Trên PC dùng thư viện chuẩn tạo ECDSA key thử, ký manifest. Trên H523 verify với PKA/driver hỗ trợ.
5. Đổi một bit image/signature/public key, đổi độ dài và encoding; tất cả trường hợp không hợp lệ phải reject.
6. Không đặt khóa private dùng ký firmware vào device. Lab dùng khóa thử; production giữ private key ở hệ thống ký được kiểm soát, device giữ trust anchor/public key theo chuỗi tin cậy.

**Đạt:** test vector pass, negative cases fail, timeout/error không dẫn đến accept. [D1 mục 3.18,3.28–3.30; D2 mục security; R chương 19,36,40–41,67]


## Bài 45 Secure boot cập nhật firmware và chống rollback

### Chuỗi tin cậy

Secure boot chỉ chuyển quyền cho image đã được xác minh theo một trust anchor không thể bị attacker thay bằng cùng đường cập nhật không tin cậy. Nếu attacker thay được cả image và public key, chữ ký không còn giá trị bảo vệ. Chuỗi thường bắt đầu từ root of trust, xác minh boot stage tiếp theo rồi ứng dụng và cấu hình cần bảo vệ.

H523 và H533 khác nhau về các thành phần cung cấp sẵn. D1 không liệt kê ST-iROT như tính năng preconfigured immutable root of trust của H533 trong D2. Không lấy một project STiRoT cho H533 rồi coi có thể nạp nguyên lên H523. Kiến trúc OEMiROT/OEMuROT, thư viện RSS/SFI và ví dụ Cube phải kiểm hỗ trợ đúng part, memory size và release.

### Image manifest và ranh giới xác minh

| Trường được xác thực | Mục đích |
|---|---|
| Magic và format version | Chọn decoder đúng |
| Target chip/board compatibility | Không nạp image sai phần cứng |
| Image size và vùng đích | Chặn tràn vùng, ghi boot/key |
| Firmware version/security counter | Chính sách chống rollback |
| Hash algorithm và digest | Ràng buộc payload image |
| Entry/vector information theo format | Kiểm khả năng khởi động |
| Signing key ID | Chọn trust anchor được phép |
| Signature | Xác thực manifest và payload qua digest |

Định nghĩa chính xác byte được ký, byte order và canonical encoding. Không cho parser chấp nhận hai cách diễn giải cùng một manifest. Kiểm số học `offset+size`, overlap và địa chỉ trước khi copy hoặc flash. Chữ ký hợp lệ chưa chứng minh image dành cho board này hoặc có version được phép; đó là các kiểm tra bổ sung.

CRC của gói truyền giúp phát hiện hỏng trong transport; hash giúp kiểm payload; signature xác thực nhà phát hành. Mã hóa image là nhu cầu confidentiality khác. Anti-rollback cần trạng thái version/counter được bảo vệ khỏi việc attacker khôi phục cùng image cũ, không chỉ một biến flash app ghi tùy ý.

### Thiết kế cập nhật chịu mất nguồn

A/B giữ image đang chạy và image ứng viên. Ghi ứng viên, verify đầy đủ, đánh dấu pending bằng metadata chịu mất nguồn; boot thử ứng viên; ứng viên phải tự kiểm tra rồi confirm. Nếu không confirm trong chính sách cho phép, rollback về image cũ hợp lệ. Không nâng irreversible rollback counter quá sớm làm image cũ không còn boot được trước khi ứng viên được xác nhận.

Trên H523CC chỉ 256 KiB flash, ngân sách phải tính boot, Secure services, app, staging, metadata và dữ liệu. Hai image lớn có thể không chứa đủ. External staging hoặc single-slot overwrite có journal là phương án khác nhưng tăng phức tạp. “Có hai bank” không đồng nghĩa luôn có đủ dung lượng cho hai bản đầy đủ.

Khi thay image cần quản lý cache, interrupt, DMA và flash đang được thực thi. Quyền updater chỉ được viết vùng ứng viên/metadata cần thiết; không để command UART chỉ định địa chỉ flash tùy ý. Sau verify, dữ liệu sắp boot không được bị thay qua một master khác trước lúc sử dụng.

### Lab 45 Cập nhật có xác thực trong môi trường thử

1. Tạo hai image nhỏ V1/V2 với cùng ABI, khác thông tin version. Dùng khóa thử và manifest được mô tả bằng byte cụ thể.
2. Viết verifier trên PC trước: hợp lệ, sai chữ ký, sai target, length quá lớn, version cũ, digest sai và unknown algorithm.
3. Tích hợp verifier vào boot stage đúng nền H523. Ban đầu giữ môi trường development có khả năng phục hồi, không khóa debug.
4. Truyền V2 theo protocol bài 24, chỉ ghi vùng staging được phép. Reset ở mọi bước nhận/erase/program/verify/pending/confirm.
5. V2 cố tình không confirm để kiểm rollback; V2 hợp lệ confirm để kiểm commit. Sau mỗi reset phải boot một image hợp lệ hoặc vào recovery được định nghĩa.
6. Thử update lại cùng request ID và mất kết nối; resume/retry phải không phá image tốt.

**Đạt:** ma trận power-cut, signature và rollback đều có kết quả; không có nhánh “verify lỗi nhưng vẫn boot”. [R chương 3–4,7–9; D1 mục 3.6–3.7; D2 dùng để phân biệt tính năng]


## Bài 46 Lifecycle debug authentication HDP và provisioning

### Bảo vệ trong các giai đoạn sản phẩm

Device lifecycle xác định quyền debug và một số cơ chế bảo vệ theo giai đoạn. H523 dùng PRODUCT_STATE cùng các option/quy tắc tương ứng; không thay toàn bộ bằng bảng RDP 0/1/2 của STM32 cũ.

| Trạng thái chính trong R | Ý nghĩa học tập |
|---|---|
| Open | Phát triển, quyền debug rộng theo cấu hình |
| Provisioning | Đang cài đặt nền tin cậy/chính sách |
| iRoT-Provisioned | Nền tin cậy đã được provision theo quy trình |
| TZ-Closed | Debug bị giới hạn về miền theo chính sách |
| Closed | Bảo vệ sản phẩm, regression phụ thuộc xác thực/chính sách |
| Locked | Không được coi là trạng thái có thể quay lại bằng reset thông thường |

Bảng là mô tả định hướng; chuyển trạng thái chỉ hợp lệ theo bảng R mục 7.6.11 với điều kiện cụ thể. Một số regression dẫn tới xóa flash/khóa hoặc thay epoch. Không học bằng cách thử các giá trị option byte trên board duy nhất đang chứa công việc cần giữ.

HDP cung cấp temporal isolation: vùng boot/secret có thể bị ẩn sau khi hide protection level tiến lên, và chỉ trở lại theo chu kỳ reset/quy tắc phần cứng. Nó khác MPU phân quyền theo region trong thời gian chạy. Write protection ngăn sửa/erase theo vùng; không tự đồng nghĩa read confidentiality. Debug authentication cho phép một số hoạt động debug/regression có ủy quyền, không phải “mật khẩu UART” do app tự đặt.

### Provisioning là một giao dịch hệ thống

Quy trình phải xác định danh tính device, image gốc, key/certificate, quyền người vận hành, chính sách debug/recovery, log kiểm tra và điều kiện chuyển trạng thái. SFI hỗ trợ secure firmware installation trong hệ sinh thái tương ứng; phải hiểu ai giữ khóa và ai được phép thấy code tại nhà sản xuất.

Private signing key của nhà phát hành không đặt vào firmware hay repo. Public key/trust anchor trên device vẫn phải được bảo vệ khỏi thay thế. UID hỗ trợ liên kết hồ sơ sản xuất nhưng không thay chứng thư xác thực. TAMP/backup erase cần được xét cùng quy trình giữ/xóa bí mật và service sau bán hàng.

### Lab 46 Mô phỏng trước khi ghi option

1. Viết bảng state machine trên giấy và phần mềm mô phỏng: state hiện tại, action, điều kiện, hậu quả dữ liệu, state kế tiếp.
2. Tạo checklist provisioning cho device thử: identity, option snapshot, image hash, test functional, verify credentials, recovery test, rồi mới bước đóng trạng thái theo quy trình chính thức.
3. Chỉ đọc PRODUCT_STATE/option trên board học và lưu bản mô tả. Bài bắt buộc không yêu cầu ghi OTP, Locked hoặc option không đảo ngược.
4. Nếu có board thử chuyên dụng và quy trình ST phù hợp, thực hiện từng chuyển trạng thái được tài liệu hỗ trợ; kiểm cả trường hợp credential hợp lệ/không hợp lệ bằng công cụ chính thức.
5. Lập quy trình trả hàng: khi nào được debug, dữ liệu nào bị xóa khi regression, làm sao xác thực phiên bản sửa chữa.

**Đạt:** hiểu hậu quả từng bước, có phương án sản xuất và hậu mãi, không chỉ “tắt SWD”. **Bài tập:** khóa debug có bảo vệ khỏi bug parser ghi vào vùng app được phép không? Không; runtime isolation và kiểm dữ liệu vẫn cần. [R chương 3,7,14,52,66; D1 security]


## Bài 47 Kiểm thử CI fault injection và đo mức đáp ứng

### Phân tầng kiểm thử

Unit test kiểm logic độc lập phần cứng: codec, parser, state machine, tính timer, conversion đơn vị. Integration test kiểm liên kết driver/service/protocol. Hardware-in-the-loop kiểm tín hiệu và lỗi thật trên board. System test kiểm yêu cầu người dùng, thời gian dài, môi trường, power cycle và cập nhật. Một bài blinky chạy không thay các cấp còn lại.

Mỗi yêu cầu phải có đơn vị và điều kiện. “UART ổn định” không đo được; “nhận 10000 packet, payload 64 byte, 115200 8N1, CPU load 60%, không mất packet, recovery dưới 200 ms sau reconnect” là yêu cầu có thể kiểm. Giá trị cụ thể phải dựa dự án, không coi các con số bài học là chuẩn công nghiệp chung.

### Các nhóm negative test

| Nhóm | Ca thử |
|---|---|
| Biên dữ liệu |0,1, max, max+1; signed/unsigned; wrap |
| Timing | Quá sớm, quá muộn, timeout, jitter |
| Transport | Split/coalesced, duplicate, CRC sai, mất kết nối |
| Tài nguyên | Queue full, heap fail, stack gần giới hạn, FIFO overrun |
| Nguồn | Reset giữa write/update, wakeup liên tục |
| Quyền | CPU/DMA access sai miền, bad pointer gateway |
| Compatibility | Board revision, image version, schema version |

Fuzzing đưa nhiều chuỗi bất thường vào parser và tìm crash/invariant violation. Sanitizer trên host giúp bắt out-of-bounds/undefined behavior nhưng không thay kiểm HAL/register trên MCU. Static analysis phát hiện một số mẫu lỗi, không chứng minh firmware an toàn tuyệt đối. Coverage cao nhưng assertion vô nghĩa vẫn bỏ lọt lỗi thiết kế.

CI nên build đúng flags, chạy host tests, phân tích warning, lưu ELF/map/hash và báo tăng flash/RAM. Pin toolchain/library version cho build tái lập. Việc “build pass” không chứng minh image được ký đúng hoặc chạy đúng chip; release gate có các bước riêng.

### Lab 47 Tạo một gate phát hành

1. Viết 10–20 ca có ý nghĩa cho parser bài 24, journal bài 32 và manifest bài 45. Bao gồm ca hợp lệ lẫn ca bị từ chối.
2. Build debug/release với warning; kiểm map không vượt flash/RAM và reserve stack hợp lý.
3. Chạy stress ADC-DMA cộng UART/CAN/storage, ghi max latency, lost counters và reset reason.
4. Tự động power-cycle fixture thử nếu có, hoặc thực hiện thủ công có log; không ghi kết quả “pass” khi chưa chạy.
5. Tạo báo cáo một trang: yêu cầu, cấu hình, số lần thử, kết quả, giới hạn chưa kiểm.
6. Chỉ release khi các gate bắt buộc đạt; ghi rõ test nào không áp dụng do package hoặc thiếu fixture.

**Đạt:** người khác có thể chạy lại test từ commit và manifest môi trường. **Bài tập:** lỗi tái hiện 1/10000 lần cần log và stress theo điều kiện nguyên nhân, không chỉ chạy lại một lần rồi kết luận đã hết. [R/P cho hành vi phần cứng; quy trình kiểm thử là phần biên soạn]


## Bài 48 Thiết kế PCB bring up và sản xuất

### Từ pin mux đến sơ đồ mạch

Bắt đầu bằng requirement và resource budget: số ADC, timer/PWM/break, giao tiếp, RAM/flash, điện năng và package. Pin count không đồng nghĩa đủ AF cho mọi chức năng cùng lúc. Lập bảng từng net, pin, AF, domain nguồn, trạng thái reset, owner và test point. Ưu tiên SWD/NRST, nguồn, oscillator, analog reference trước khi lấp hết chân bằng chức năng phụ.

Nguồn cần tụ decoupling/VCAP đúng datasheet và đường trở về tốt. Analog cần tham chiếu sạch, bố trí dòng return và trở kháng nguồn phù hợp; không đơn giản cắt ground plane tùy ý. USB, clock cao, OCTOSPI/SDMMC cần kiểm routing/timing/return path; CAN/RS 485/LIN cần transceiver, termination/bias, ESD và điều kiện common-mode.

Unused pin chọn trạng thái phù hợp để tránh floating/back-power. Reset/boot pins không được kéo theo một thiết bị ngoài khiến MCU vào nhầm boot. USB-C dead-battery, VBAT và VDDIO 2 nếu có phải được review theo package. Mạch nguồn motor/tải nhiễu cần tách đường dòng lớn khỏi tham chiếu analog theo thiết kế thực.

### Bring up theo từng lớp

1. Kiểm trở kháng nguồn–GND khi tắt nguồn và hướng linh kiện; dùng nguồn giới hạn dòng cho lần cấp đầu.
2. Đo từng rail, NRST và clock cơ bản trước khi bật toàn ngoại vi.
3. Kết nối SWD, đọc chip ID/flash size, nạp chương trình tối thiểu HSI và GPIO.
4. Kiểm clock ngoài, UART log, analog reference và các interface từng cái một.
5. Tăng clock/tải và đo reset/brownout/temperature. Lỗi nguồn chỉ xuất hiện khi bật nhiều peripheral cần trace nguồn cùng trigger lỗi.
6. Chạy hệ thống tổng hợp, không dừng ở kiểm từng khối riêng.

### Manufacturing data và kiểm thử cuối dây chuyền

Thông tin điện tử gồm UID 96-bit, device/revision ID và flash size phục vụ nhận dạng, chọn errata và ghi hồ sơ. Firmware phải đối chiếu board revision hoặc cấu hình sản phẩm theo cách được thiết kế. Một tên chip trong project không chứng minh PCB được lắp đúng silicon.

End-of-line test kiểm pin quan trọng, nguồn, giao tiếp, ADC/DAC loopback, clock/RTC và thông tin version. Ghi kết quả cùng serial/UID/firmware hash; không log bí mật sản xuất. Calibration phải có version, đơn vị, CRC và khoảng giá trị cho phép; image update không được vô tình xóa hệ số đã hiệu chuẩn.

### Lab 48 Hồ sơ thiết kế có thể bàn giao

Tạo pin allocation, power tree, clock tree, memory budget, bus timing budget, threat model, fault matrix, programming/provisioning checklist và bộ acceptance tests. Review mỗi mục bằng một ví dụ lỗi cụ thể: thiếu VCAP, sai crystal mode, ADC quá trở kháng, SPI nhả CS sớm, option lifecycle sai, firmware vượt flash.

**Đạt:** người khác có thể lắp, nạp, kiểm tra và chẩn đoán board bằng hồ sơ này. **Bài tập:** thay H523CC sang H523CE có thể thêm flash nhưng không tự thêm chân hoặc SDMMC; thay sang H523RE đổi cả pin/package và cần review PCB/AF. [D1 mục 4–6 và device signature; R chương 67]


# Chủ đề 12 Dự án tổng hợp và đánh giá cuối khóa

Bài 49 đến Bài 52. Đọc lịch buổi học để chọn sản phẩm cần hoàn thành trước khi chuyển bài.


## Bài 49 Dự án thu phát tín hiệu analog

### Yêu cầu và kiến trúc

Xây một thiết bị tạo sóng DAC 1 kHz, lấy mẫu hai kênh ADC 10 ksample/s/kênh, tính giá trị trung bình/RMS theo block, gửi telemetry 100 Hz qua UART và thay tham số bằng packet có CRC. Đây là dự án lõi làm được trên H523C nếu các chân cần thiết đã được đưa ra board.

| Khối | Cấu hình mục tiêu | Dữ liệu |
|---|---|---|
| DAC1 | LUT 100 mẫu, trigger 100 kHz | Sin 1 kHz có offset |
| ADC1 | Hai rank, trigger 10 kHz | Interleaved uint16 |
| GPDMA | Buffer 256 sample/kênh, chia hai nửa |128 sample/kênh mỗi callback |
| CPU | Mean/RMS, phát hiện mất block | Snapshot có sequence |
| UART |115200 8N1 | Packet mẫu ở bài 24, telemetry 100 Hz |

Mỗi nửa ADC có deadline 12,8 ms. Hai kênh raw tạo 40 kB/s nên không thể gửi toàn bộ qua 115200 8N1. Packet telemetry payload 8 byte cộng overhead 10 byte tạo 18 byte; 100 packet/s là 1800 byte/s hay 18000 bit/s với 8N1, khoảng 15,6% line rate trước các khoảng nghỉ. Việc tính bandwidth này quyết định thiết kế downsample, không đợi mất dữ liệu rồi tăng queue.

### Trình tự triển khai

1. Hoàn thành độc lập GPIO/UART, ADC và DAC với các lab trước. Lưu các kết quả đã đo làm baseline.
2. Nối DAC PA4 sang ADC PA0 trong dải điện hợp lệ; kênh còn lại dùng biến trở. Chuyển LED sang PB0 nếu cần để không xung đột PA5.
3. Cấu hình timer trigger đúng bảng interconnect, ADC kernel clock hợp lệ và sampling đủ dài. Start các consumer/DMA trước nguồn trigger.
4. Trong callback chỉ đưa half-index/sequence; task xử lý kiểm sequence và deadline trước khi đọc buffer. Nếu nửa đã bị ghi đè, đánh dấu mất block và không xuất số liệu giả.
5. Tính mean bằng bộ tích lũy đủ rộng; RMS sau khi bỏ DC nếu muốn AC RMS. Với raw 12-bit, bình phương gần 16,8 triệu; tổng nhiều mẫu cần uint64_t hoặc float có phân tích sai số.
6. Encode telemetry bằng codec byte. Parser nhận lệnh đổi biên độ/tần số, kiểm giới hạn DAC/ADC/bandwidth và chỉ cập nhật tại ranh giới an toàn.
7. Thêm watchdog supervisor theo tiến độ block và parser; lưu reset reason.

### Kịch bản nghiệm thu

Đo sin 1 kHz, sample count theo thời gian, tần số gửi 100 Hz và jitter callback. Gửi command hợp lệ/sai CRC/quá length/trùng request. Flood UART khi DMA chạy; làm consumer chậm để kiểm overrun; ngắt kết nối PC; reset giữa lúc đổi tham số. Chạy một phiên dài do người học chọn, ghi thời lượng thật cùng lost counters, không ghi “ổn định tuyệt đối”.

**Sản phẩm nộp:** sơ đồ nối, bảng clock/timer, source/`.ioc`, spec packet, host decoder, ảnh scope/analyzer, báo cáo CPU/RAM/deadline và lỗi đã thử. **Mức nâng cao:** thêm CRC unit, calibration, low-power giữa các phiên và USB CDC để truyền raw data ở rate phù hợp.


## Bài 50 Dự án tầng điều khiển độc lập với ứng dụng

### Mục tiêu có thể kiểm chứng

Thiết kế tầng control chạy chu kỳ 1 ms, app gửi setpoint và đọc status. App không được ghi state control, cấu hình PWM/ADC hoặc DMA control. Khi app nhận dữ liệu sai, ghi nhầm vùng nhớ hoặc chiếm CPU, hệ thống phải có hành vi đã định nghĩa và lưu bằng chứng về deadline/đầu ra.

Bắt đầu bằng plant giả lập hoặc PWM ra oscilloscope. Chỉ gắn power stage/motor sau khi có review điện, giới hạn dòng và đường shutdown phần cứng. Giáo trình không tự chứng nhận một thiết kế điều khiển công suất là sản phẩm an toàn chức năng.

### Thiết kế điều khiển và state

State gồm INIT, IDLE, RUN, FAULT và REARM. INIT tự kiểm tra nguồn/sensor/config; IDLE giữ output inactive; RUN áp setpoint và giới hạn; FAULT latch nguyên nhân và đưa ra trạng thái đã thiết kế; REARM chỉ cho phép khi fault đã nhả và có request hợp lệ.

PI rời rạc đơn giản: `e = target - measured`; `integral += Ki * e * dt`; `u = clamp(Kp*e + integral, umin, umax)`. Khi output bão hòa, cần anti-windup để integral không tăng vô hạn. Khi input invalid, không tiếp tục dùng giá trị cũ mà không giới hạn; ghi age và timeout. Control algorithm cũng cần kiểm NaN, overflow và giới hạn rate setpoint.

### Hai cấu hình cần so sánh

Cấu hình A dùng RTOS với app unprivileged, MPU và service/SVC; cấu hình B đặt control resource trong Secure, app ở Non-secure và gateway nhỏ. Cả hai phải bảo vệ clock/reset/DMA/IRQ của control. Đo latency crossing boundary và bus contention, không chọn phương án chỉ vì sơ đồ có hai ô.

### Thực hành theo từng mốc

1. Chạy control giả lập 1 kHz, đo GPIO marker để xác định period và execution time. Chọn deadline/jitter chấp nhận dựa trên plant, ví dụ mục tiêu bài học jitter dưới 10 µs nhưng chỉ đánh giá bằng số đo.
2. Tách setpoint request khỏi state control; không export mutable pointer. Bắt buộc validate range, state, deadline và request ID.
3. Cấu hình MPU/TrustZone/GTZC tương ứng. Test CPU app ghi RAM control và app DMA ghi RAM control đều bị chặn.
4. App flood queue, gọi gateway liên tục, giữ mutex app, xử lý packet lỗi. Control không được phụ thuộc mutex do app nắm vô hạn.
5. Kích break ngoài để kiểm output shutdown không chờ task. Thử fault vẫn active khi app gửi lệnh start.
6. Inject fault trong app và ghi policy: restart app, degraded mode hay reset toàn chip. Phân biệt trường hợp control tiếp tục được với lỗi cần đưa output về safe state.

**Sản phẩm nộp:** threat/fault model, memory/peripheral ownership, API/ABI, logic recovery, worst observed timing và ma trận allowed/denied. **Mức senior:** chỉ ra giả thiết mà thiết kế chưa đảm bảo, như lỗi hardfault secure, bus lockup, nguồn mất hoặc reset toàn chip; đề xuất hai MCU nếu yêu cầu đòi hỏi fault domain độc lập hơn.


## Bài 51 Dự án datalogger và gateway nhiều giao tiếp

### Kiến trúc

Thiết bị đọc một cảm biến I2C/I3C, một luồng ADC và một luồng CAN, chuẩn hóa thành record có timestamp/sequence/source, ghi lên storage và gửi PC qua USB CDC. Board H523R/V/Z dùng SDMMC; H523C chọn SPI-SD/serial flash với ngân sách tốc độ phù hợp. Không yêu cầu một package 48 chân bỗng có camera/SDMMC.

Schema record: magic/version, record_length, source_id, timestamp 64 hoặc 32 bit có quy ước wrap, sequence, payload_type, payload_length, payload, CRC. Timestamp cần nói rõ monotonic hay UTC; RTC update không được khiến thời gian đo đột ngột quay lùi mà không cờ báo. Nếu đồng bộ nguồn ngoài, lưu chất lượng/thời gian từ lần sync gần nhất.

### Tách pipeline

Driver nhận dữ liệu thành buffer owned; normalizer serialize theo schema; queue chuyển record; storage writer ghi theo block; USB sender gửi theo nhu cầu host. Hai consumer dùng ref count/pool hoặc copy có ngân sách, không cho storage và USB cùng giải buffer. Không giữ bus lock I2C khi đang đợi file system.

Tính RAM buffer dựa theo stall storage: nếu ingress 20 kB/s và stall đã thiết kế 250 ms thì ít nhất 5 kB cho phần stall đó, cộng metadata, burst và margin. Nếu stall không có cận thì không thể hứa không mất data với RAM hữu hạn; cần drop/rate-limit/degraded policy.

### Lab theo giai đoạn

1. Tạo adapter cho từng nguồn và test riêng với known pattern. Chốt unit, scale, endianness và timestamp.
2. Ghép từng nguồn một, đo bus load/CPU load/RAM. Không bật cùng lúc mọi interface rồi debug một lỗi không rõ nguồn.
3. Ghi record có sequence/CRC, đọc lại bằng host tool và so sánh count.
4. Cho SD/flash stall, USB host ngừng đọc, sensor nack và CAN bus-off; từng nhánh lỗi không được khóa toàn pipeline.
5. Rút/cắm storage trong điều kiện thiết kế cho phép, power cut trên thiết bị thử, kiểm recover và mất data trong cửa chấp nhận.
6. Thêm low power cho trạng thái chưa ghi, giữ RTC/wakeup theo clock bảng R.

**Sản phẩm nộp:** architecture, protocol schema, pin budget, bandwidth/RAM budget, latency histogram, parser PC và test report. **Mức nâng cao:** thêm OCTOSPI staging, camera frame nhỏ hoặc PSSI input trên package phù hợp; dùng bảng tài nguyên để quyết định phần nào đủ chân/bandwidth.


## Bài 52 Dự án thiết bị có cập nhật và hồ sơ phát hành

### Mục tiêu cuối khóa

Ghép thiết bị bài 49 hoặc 51 với boot/update bài 45, isolation bài 41–43 và quy trình release bài 47–48. Kết quả là một sản phẩm thử có tài liệu thiết kế, image được xác minh, dữ liệu cấu hình bền vững và khả năng chẩn đoán lỗi.

Trước code, phân bổ flash thực tế theo part. Một ngân sách minh họa 256 KiB có thể gồm 48 KiB boot/security,96 KiB slot A,96 KiB slot B và 16 KiB metadata/data; tổng 256 KiB. Đây chỉ là bài toán ngân sách, không phải linker script: địa chỉ bank, granularity, security watermark và kích thước thư viện thực tế có thể khiến phương án không khả thi. Nếu vượt, phải đổi bố trí/staging/part hoặc giảm phạm vi, không sửa dung lượng chip trong linker để build pass.

### Các mốc phải hoàn thành

1. **Specification:** pin, clock, power, memory, protocol, security và deadline được viết thành các yêu cầu có thể test.
2. **Baseline:** firmware V1 chạy chức năng chính, journal config đã qua reset test, fault log giải mã được bằng ELF đúng build.
3. **Update:** nhận V2, verify target/version/hash/signature, boot trial và confirm; rollback khi V2không đạt.
4. **Isolation:** app CPU/DMA không sửa Secure/control resource; gateway xử lý bad pointer và length; flood không làm vỡ deadline đã cam kết trong test envelope.
5. **Resilience:** ngắt kết nối, queue full, watchdog, reset giữa flash write và power cut update được thử.
6. **Release:** tạo image, manifest, hash, version, source commit, map file và báo cáo test. Provisioning production chỉ tiến hành theo quy trình đã review, không trộn vào lab phát triển.

### Thang đánh giá năng lực

| Nhóm | Điểm | Bằng chứng |
|---|---:|---|
| Hiểu phần cứng và giới hạn part |15 | Pin/clock/power/memory đúng datasheet |
| Driver và protocol |20 | Frame/schema/timeout/recovery và trace |
| Timing và hiệu năng |15 | Deadline, WCET estimate, jitter, bandwidth |
| Robustness và persistence |15 | Fault injection, journal, watchdog |
| Isolation và security |20 | Access matrix, verify/rollback, gateway tests |
| Chất lượng bàn giao |15 | Build tái lập, test report, hồ sơ production |

Các lỗi chặn nghiệm thu dù tổng điểm cao: chấp nhận image sai chữ ký, ghi vượt buffer, vô hiệu hóa bảo vệ để chạy, mất toàn bộ image hợp lệ trong power cut đã đặt yêu cầu, hoặc đầu ra công suất vượt giới hạn do input không hợp lệ. Điểm số là công cụ tự đánh giá, không thay chứng nhận nghề nghiệp.

**Dấu hiệu đạt mức làm việc độc lập:** có thể giải thích vì sao chọn cấu hình, chỉ ra giới hạn chưa kiểm, đọc errata để điều chỉnh thiết kế và sửa lỗi bằng bằng chứng. **Dấu hiệu tiến tới senior:** xây được giao diện và cơ chế kiểm chứng để người khác phát triển an toàn trong hệ thống, dự đoán failure mode trước khi nó xuất hiện ở khách hàng và cân đối rõ chi phí, độ phức tạp, độ tin cậy.


# Phụ lục A Ma trận bao phủ toàn bộ chức năng trong datasheet

Bảng này đối chiếu các mục 3.1–3.42 của D1 với bài học. Nó bao gồm ngoại vi, khối lõi, bus, nguồn và bảo mật; những chức năng có chung phần cứng vẫn được chỉ rõ. Tra bảng package ở đầu giáo trình trước khi chọn lab.

| Mục D1 | Khối hoặc chức năng | Bài | Nội dung thực hành chính |
|---|---|---|---|
|3.1 | Cortex-M33, FPU, TrustZone |04,37,41,42 | Context, exception, quyền, chuyển miền |
|3.2 | ART, ICACHE, DCACHE |06,18,40 | Đường truy cập, cache maintenance, đo hit/miss |
|3.3 | MPU |41 | Allowed/denied và service boundary |
|3.4 | Flash, ECC, protection, privilege |05,32,45,46 | Program/erase, journal, image update |
|3.5 | SRAM và quyền |06,17,18,43 | Linker, RAMCFG, ECC, ownership |
|3.6 | Security overview |41–46 | Threat model và chuỗi tin cậy |
|3.7 | Boot modes |05,45,46 | Startup, entry, boot policy |
|3.8 | GTZC |43 | CPU/DMA/peripheral access matrix |
|3.9 | TrustZone architecture |42,43 | SAU/IDAU/GTZC và NSC |
|3.10 | Power, reset, VBAT |07,15,36 | Rail, VOS, reset cause, low power |
|3.11 | Peripheral interconnect |14,20,21 | Timer trigger ADC/DAC và break |
|3.12 | RCC |08 | Cây clock, PLL, kernel/bus, CSS |
|3.13 | CRS |08,30 | HSI48 và nguồn đồng bộ |
|3.14 | GPIO |09,10 | Input/output/AF/analog, EXTI |
|3.15 | Multi-AHB bus matrix |06,17,18,43 | Master/slave, arbitration, contention |
|3.16 | GPDMA1/2 |17,20,21,43 | Direct, linked list, 2D, quyền và lỗi |
|3.17 | NVIC và EXTI |10,38,40 | Priority, latency, pending, fault |
|3.18 | CRC |24,44 | Tham số CRC và known-answer test |
|3.19 | FMC và LCD parallel |33 | Bus cycle, setup/hold, memory map |
|3.20 | OCTOSPI |33 | Indirect, polling, memory-mapped, frame lệnh |
|3.21 | DLYB |33 | Quét cửa timing và calibration |
|3.22 | ADC1/2, nhiệt analog, VREFINT, VBAT |19,20 | Single/scan/trigger/DMA, calibration |
|3.23 | DTS |20 | Đo nhiệt die và threshold |
|3.24 | DAC |21 | Code-to-voltage, trigger/DMA, sample-and-hold |
|3.25 | VREFBUF |21 | Tham chiếu, package V/Z, điều kiện nguồn |
|3.26 | DCMI |35 | Frame ảnh, line/pixel packing, DMA |
|3.27 | PSSI |35 | Parallel 8/16 bit, DE/RDY, app frame |
|3.28 | RNG |44 | Entropy status và error path |
|3.29 | HASH |44,45 | SHA-1/SHA-2, hash stream và verify image |
|3.30 | PKA |44,45 | ECDSA verification theo giới hạn H523 |
|3.31 | TIM1/8, TIM2/3/4/5/12/15, TIM6/7 |11–14 | Counter, PWM, capture, encoder, break |
|3.31 | LPTIM1/2, IWDG, WWDG, SysTick |11,15,16,36 | Low-power timing, watchdog, timebase |
|3.32 | RTC, TAMP, backup registers |15,46 | Calendar, alarm, tamper, retention |
|3.33 | I2C, SMBus, PMBus |25 | Frame, PEC/command tầng trên, recovery |
|3.34 | I3C1/2 |26 | SDR, DAA, CCC, IBI, mixed bus |
|3.35 | USART/UART/LPUART |23,24 | Frame, FIFO/DMA, protocol và wakeup |
|3.35 | LIN, IrDA, ISO7816, synchronous, RS 485 support |23 | Format/stack boundary và mode theo instance |
|3.36 | SPI và I2S |27,28 | Transaction, CPOL/CPHA, audio slots |
|3.37 | SDMMC/SDIO/MMC |34 | Command/response/block/filesystem |
|3.38 | FDCAN1/2 |29 | Classical/FD, DLC, filter, bus-off |
|3.39 | USB FS host/device |30 | Packet, descriptors, transfer/class |
|3.40 | UCPD |31 | CC, PD frame, PDO/RDO, policy |
|3.41 | HDMI-CEC |35 | Header/opcode/operand/EOM/ACK |
|3.42 | SWD/JTAG/ETM debug |03,40,46 | Debug, trace, authentication |
|R chương 6,14,67 | RAMCFG, SBS, electronic signature |06,43,48 | RAM control, system security, UID/revision |

OPAMP/COMP, Ethernet MAC, SAI riêng, CORDIC, FMAC, PLAY, AES/SAES/OTFDEC không được đưa vào danh mục ngoại vi tích hợp H523. Bài 22 về OPAMP/comparator là mạch ngoài. Các chế độ/giao thức tầng trên có tên trong sách không tự trở thành hardware engine riêng.

# Phụ lục B Danh mục frame và cấu trúc dữ liệu giao tiếp

| Giao tiếp | Frame hoặc chu kỳ trên dây | Cấu trúc ứng dụng trong sách | Bài |
|---|---|---|---:|
| UART/LPUART | Start, data, parity, stop | uart_format_t, uart_stats_t, app_packet_t |23,24 |
| RS 485 | Dùng ký tự UART; DE và turnaround | Request/response và packet đã chọn |23,24 |
| LIN | Break, sync, PID, data, checksum | lin_frame_t |23 |
| Smartcard | Character ISO7816; protocol T=0/T=1 tùy stack | Header APDU, data length, Le, SW1/SW2 |23 |
| IrDA SIR | Mã hóa xung cho bit UART | Packet/profile của ứng dụng |23 |
| Synchronous USART | Clock cùng dữ liệu theo mode | Descriptor giao dịch theo thiết bị |23 |
| I2C | START, address/RW, ACK, data, STOP | i 2c_transaction_t |25 |
| SMBus/PMBus | I2C transaction cùng command/PEC khi dùng | Command ID, đơn vị và schema thiết bị |25 |
| I3C | Address/ACK, data/T, CCC/DAA/IBI | i 3c_device_t, i 3c_transfer_t |26 |
| SPI | CS và chuỗi word; command tùy device | spi_segment_t, spi_device_profile_t |27 |
| I2S | WS, slots, sample bits, BCLK | pcm_stereo_t và DMA layout |28 |
| CAN/FDCAN | Arbitration/control/data/CRC/ACK/EOF | can_message_t và DLC mapping |29 |
| USB FS | Token/data/handshake/SOF | usb_setup_view_t và descriptors |30 |
| USB PD | Preamble/SOP/header/objects/CRC/EOP | pd_message_t, pd_policy_t |31 |
| OCTOSPI/HyperBus | Các pha command/address/dummy/data hoặc HyperBus CA | serial_memory_command_t |33 |
| FMC | Address/control/data/wait bus cycle | Memory layout, width, byte lanes |33 |
| SDMMC/SDIO | Command, response, data block, CRC | block_request_t |34 |
| DCMI | Frame/line/pixel đồng bộ | image_buffer_t |35 |
| PSSI | Word song song, clock, DE/RDY | Header/length/sequence/payload/CRC tự thiết kế |35 |
| CEC | Start, header, opcode/operand, EOM/ACK | cec_message_t |35 |

Một struct C trong bảng là mô hình dữ liệu để học. Khi đi qua bus, lưu file hoặc gọi giữa image, luôn xác định codec/ABI và kiểm biên; không suy ra wire format từ `sizeof(struct)`.

# Phụ lục C Quy trình đọc một ngoại vi mới

1. **Kiểm sự tồn tại:** part, package, instance, pin và AF trong D1.
2. **Đọc sơ đồ khối:** input, output, clock, FIFO/buffer, state machine, IRQ/DMA.
3. **Đọc chuỗi hoạt động:** trạng thái reset, enable, ready, start, busy, complete, stop/abort.
4. **Liệt kê mode:** mode nào dùng trong dự án, mode nào loại trừ nhau, điều kiện chuyển mode.
5. **Chốt timing:** clock kernel, prescaler, external setup/hold, timeout và clock accuracy.
6. **Chốt dữ liệu:** độ rộng, alignment, endian, framing, length, CRC, ownership/lifetime.
7. **Chốt quyền:** CPU privilege/security, peripheral/clock/IRQ/DMA/buffer attribution.
8. **Chốt lỗi:** flag nào cần ghi 0/ghi 1 để clear, FIFO có mất dữ liệu không, phục hồi cần reset khối nào.
9. **Làm một giao dịch nhỏ:** polling giúp nhìn trình tự; sau đó chuyển interrupt/DMA khi có lý do.
10. **Đo và gây lỗi:** ghi kết quả và errata áp dụng; chỉ tăng tốc sau khi baseline đúng.

Một checklist thanh ghi hữu ích cho mọi driver: clock enable; reset peripheral; pin/AF; kernel clock; mode; timing; data width; buffer; clear stale flag; IRQ/DMA; security; enable; start; timeout; complete; abort; shutdown. Thứ tự chi tiết phải theo peripheral, không coi danh sách là một trình tự duy nhất đúng cho mọi IP.

# Phụ lục D Mẫu báo cáo cho mỗi buổi thực hành

**Buổi và bài:** ghi số buổi, chủ đề và mục tiêu đo được.

**Cấu hình:** part/package/revision; board revision; nguồn; sơ đồ nối; clock; pin; phiên bản compiler/Cube/driver; Git commit; TrustZone/lifecycle hiện tại.

**Dự đoán trước phép đo:** giá trị register trọng tâm, công thức, frame byte dự kiến, tần số/latency/bandwidth và giới hạn sai số.

| Ca thử | Đầu vào hoặc thao tác | Kết quả dự kiến | Kết quả đo | Kết luận |
|---|---|---|---|---|
| Bình thường | Một giao dịch hợp lệ | Theo spec | Điền sau khi đo | Chưa thử/Đạt/Không đạt |
| Biên | Min/max hoặc wrap | Không tràn, đúng định dạng | Điền sau khi đo | Chưa thử/Đạt/Không đạt |
| Lỗi | Timeout/CRC/access sai | Bị từ chối và phục hồi | Điền sau khi đo | Chưa thử/Đạt/Không đạt |
| Tải | Producer nhanh/consumer chậm | Chính sách quá tải rõ | Điền sau khi đo | Chưa thử/Đạt/Không đạt |

**Phân tích:** lỗi nằm ở điện, clock, pin, protocol, ownership, timing hay quyền; bằng chứng loại trừ các giả thuyết khác.

**Bàn giao:** source, cấu hình, map/ELF phù hợp, trace ảnh hoặc file capture, host test và giới hạn chưa kiểm. Không thay cột “kết quả đo” bằng giá trị dự kiến.

# Phụ lục E Từ điển thuật ngữ

| Thuật ngữ | Ý nghĩa thực dụng |
|---|---|
| Peripheral | Khối phần cứng chuyên trách ngoài lõi CPU |
| Register | Giao diện điều khiển/trạng thái được ánh xạ địa chỉ |
| Bus master | Thành phần khởi tạo giao dịch bus |
| Kernel clock | Clock chạy logic chức năng của ngoại vi |
| Bus clock | Clock cho giao diện register/bus |
| Prescaler | Bộ chia tần số |
| Jitter | Biến thiên thời điểm so với lịch mong muốn |
| Latency | Thời gian từ sự kiện đến phản ứng |
| Throughput | Lượng dữ liệu hữu ích xử lý mỗi đơn vị thời gian |
| Bandwidth | Khả năng truyền hoặc ngân sách dung lượng kênh |
| FIFO | Bộ đệm vào trước ra trước |
| DMA request | Yêu cầu trao đổi dữ liệu của ngoại vi |
| DMA trigger | Điều kiện kích hoạt tiến trình/chuyển giao được cấu hình |
| Descriptor | Cấu trúc mô tả một giao dịch hoặc buffer |
| Ownership | Ai được đọc/ghi/giải phóng tài nguyên ở thời điểm đó |
| Lifetime | Khoảng thời gian dữ liệu/con trỏ còn hợp lệ |
| Alignment | Ràng buộc địa chỉ theo bội số byte |
| Endianness | Thứ tự byte của số nhiều byte |
| Race condition | Kết quả phụ thuộc thứ tự xen kẽ không được kiểm soát |
| Atomic | Thao tác không bị quan sát ở trạng thái trung gian theo phạm vi bảo đảm |
| Reentrant | Có thể được gọi xen kẽ mà không phá trạng thái chung |
| Idempotent | Thực hiện lại vẫn cho tác dụng cuối như một lần |
| Deadline | Hạn chót hoàn thành để kết quả còn có ích/hợp lệ |
| WCET | Thời gian thực thi xấu nhất trong mô hình đã xác định |
| Fault | Lỗi được phần cứng/hệ thống phát hiện theo loại |
| Recovery | Khôi phục về trạng thái đã biết và kiểm soát được |
| Privileged | Có quyền hệ thống cao hơn trong miền CPU tương ứng |
| Secure | Thuộc miền bảo mật TrustZone, độc lập với trục privilege |
| NSC | Vùng Secure cho phép entry Non-secure qua gateway hợp lệ |
| Root of trust | Nền tin cậy dùng để kiểm chứng các thành phần tiếp theo |
| Provisioning | Cài đặt danh tính, khóa/chứng thư và chính sách ban đầu |
| Anti-rollback | Ngăn quay lại phiên bản không còn được phép |
| Errata | Mô tả giới hạn/lỗi silicon hoặc tài liệu và workaround |

# Phụ lục F Tài liệu tham chiếu và trạng thái kiểm chứng

**Tài liệu chính do người học cung cấp:**

- D1: `DS_stm32h523cc.pdf`, DS14540 Rev 3, tháng 3/2026. Đọc bảng 2 trang 15–16, functional overview trang 18–57, pin/AF trang 58–92 và electrical characteristics từ trang 93.
- D2: `DS_stm32h533ce.pdf`, DS14539 Rev 3, tháng 3/2026. Dùng so sánh tính năng, đặc biệt cryptography; không thay D1.
- R: `rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics.pdf`, RM0481 Rev 5, tháng 7/2026. Đây là manual dùng chung nhiều dòng; luôn đọc applicability đầu chương.
- P: `pm0264-stm32-cortexm33-mcus-and-mpus-programming-manual-stmicroelectronics.pdf`, PM0264 Rev 8, tháng 3/2026. Tập trung mục 2 processor, mục 3 instruction và mục 4 core peripherals.

**Tài liệu chính thức bổ sung đã đối chiếu:**

- E: [ES0621 STM32H523xx và STM32H533xx device errata](https://www.st.com/resource/en/errata_sheet/es0621-stm32h523xx-and-stm32h533xx-device-errata-stmicroelectronics.pdf), Rev 3, tháng 1/2026. Bài thực hành cần ghi silicon revision và những mục áp dụng; chú ý các mục về DAC clock, flash operation, peripheral protection và PSSI mode khi sử dụng.
- SW: [STM32CubeH5 chính thức](https://github.com/STMicroelectronics/STM32CubeH5). Chọn release hỗ trợ đúng part, ghi version thực tế và lấy đầy đủ submodule; không giả định mọi ví dụ của board khác chạy nguyên trên H523.
- LIN: [Microchip LIN Data Link Layer](https://developerhelp.microchip.com/xwiki/bin/view/applications/lin/data-link-layer/), dùng bổ sung cấu trúc PID và checksum của LIN. Thanh ghi triển khai H523 vẫn theo R chương 56.
- [AN5879 Introduction to I3C for STM32 MCUs](https://www.st.com/resource/en/application_note/an5879-introduction-to-i3c-for-stm32-mcus-stmicroelectronics.pdf), tài liệu đọc thêm cho bài 26; đối chiếu đúng implementation H523, không lấy tính năng dòng khác.
- [Trang tài liệu STM32H523 và STM32H533](https://www.st.com/en/microcontrollers-microprocessors/stm32h523-533/documentation.html), nơi tìm AN2606, AN2834, các application note và bản errata phù hợp khi bắt đầu dự án.

**Giới hạn xác minh của bản biên soạn:** tài nguyên, package và các cơ chế trọng tâm đã được đối chiếu các tài liệu trên; bảng frame, công thức và test vector được kiểm tra về logic. Không có board STM32H523 nối vào phiên biên soạn này, nên các lab là quy trình để người học thực hiện, không phải kết quả phần cứng đã đo. Những đoạn HAL/CMSIS cần đặt vào project đúng phiên bản và được build/test trên board. Bộ codec đi kèm là ví dụ chạy trên máy tính để kiểm wire format; nó không thay firmware driver hoàn chỉnh.

# Phụ lục G Bộ codec thực hành và frame mẫu

Bộ mã đi kèm gồm C11 encoder/decoder cùng Python reference và stream parser. Nó thuộc bài 24, dùng cho cả UART hoặc USB CDC sau khi tách transport. Python chỉ dùng standard library; C không phụ thuộc HAL.

Frame mẫu đã được đối chiếu giữa hai implementation:

```text
A5 5A01 02 34 12 02 00 01 02 E1 81
```

Giải nghĩa: SOF=A5 5A, version=1, type=2, sequence=0x1234, length=2, payload=0102. CRC trên 8 byte từ Version đến Payload bằng 0x81E1, nên hai byte cuối là E1 81. Test vector ASCII 123456789 cho CRC 0x29B1.

Bộ test kiểm mọi payload length 0–240, frame thiếu byte,CRC sai,buffer nhỏ, split/coalesced stream, payload chứa SOF, deadline reset và nhiễu giả ngẫu nhiên có seed cố định. Đây là test host cho logic protocol; chưa phải kiểm toàn firmware hoặc đường điện trên board. Các file và lệnh chạy nằm trong thư mục `STM32H523_practice` của gói tải về.


# Phụ lục H Sổ thực hành triển khai từng bước

Phụ lục này cụ thể hóa các bài nền tảng thành một chuỗi có thể làm tại bàn. Các đoạn C dùng handle do project STM32Cube sinh ra; đây là mã tích hợp, không phải project chạy độc lập trên mọi board. Lưu từng lab thành project hoặc cấu hình riêng. Trước khi nạp, kiểm tra đúng part, sơ đồ board, chân, clock và quyền Secure/Non-secure. Khi một tên API khác phiên bản, đối chiếu prototype trong bộ STM32CubeH5 đang dùng thay vì tự đoán tham số.

## H1 Chuẩn bị project và xác nhận đường debug

Mục tiêu là tạo một đường kiểm chứng từ source đến firmware đang chạy. Bắt đầu bằng project trống đúng mã chip in trên linh kiện. Với STM32H523CC, flash là 256 KiB. Chọn cấu hình TrustZone phù hợp trạng thái chip đang có; một project Non-secure không thể mặc nhiên cấu hình mọi ngoại vi nếu chip đang chạy boot Secure.

1. Ghi tên board, part, package, nguồn cấp và chân SWD vào `board_notes.md`. Đọc sơ đồ để xác định LED có active-high hay active-low, điện trở hạn dòng có sẵn hay chưa.
2. Kết nối SWDIO, SWCLK, GND, reference voltage và NRST theo probe. Reference voltage của probe dùng để biết mức điện I/O; không mặc định nó là đầu ra cấp nguồn cho board.
3. Tạo project C, giữ clock HSI với cấu hình được tool tạo hợp lệ. Build trước khi thêm mã. Lưu toolchain version, STM32CubeH5 version và file `.ioc`.
4. Nạp rồi đặt breakpoint ở `main`. Kiểm PC nằm trong flash dự kiến; xem SP có nằm trong SRAM hợp lệ; đối chiếu map file. Nếu không tới main, xem Reset_Handler và fault thay vì thêm delay tùy ý.
5. Thêm một biến `volatile uint32_t boot_marker = 0x12345678u;`, xem biến trong debugger. Sửa giá trị trong source, build/nạp lại để chứng minh ELF và image đang chạy cùng phiên bản.
6. Thử reset bằng NRST rồi chạy lại không giữ breakpoint. Một chương trình chỉ chạy khi debugger gắn có thể phụ thuộc timing, semihosting, nguồn từ probe hoặc cấu hình debug freeze.

Sản phẩm cần lưu: log build không lỗi, map file, ảnh breakpoint ở main, sơ đồ dây và ghi chú nguồn. Khi mất kết nối, kiểm nguồn và GND trước, rồi tốc độ SWD, NRST và chế độ connect-under-reset nếu tool hỗ trợ. Không chuyển product state chỉ để sửa lỗi kết nối chưa rõ nguyên nhân.

## H2 GPIO với thanh ghi và một nút chống dội

Giả sử PB0 nối qua điện trở tới LED xuống GND, mức 1 làm sáng; PB1 là nút nối xuống GND, dùng pull-up. Nếu board có cách nối khác thì đổi polarity. Mã sau đặt PB0 về thấp trước khi bật output và không sửa cấu hình các chân khác.

```c
/* Chạy sau HAL_Init và cấu hình clock hợp lệ. */
__HAL_RCC_GPIOB_CLK_ENABLE();
GPIOB->BSRR = (1u << (0u + 16u));
GPIOB->OTYPER &= ~(1u << 0u);
GPIOB->OSPEEDR &= ~(3u << (0u * 2u));
GPIOB->PUPDR &= ~(3u << (0u * 2u));
GPIOB->MODER = (GPIOB->MODER & ~(3u << 0u)) | (1u << 0u);

/* PB1 input, pull-up. */
GPIOB->MODER &= ~(3u << 2u);
GPIOB->PUPDR = (GPIOB->PUPDR & ~(3u << 2u)) | (1u << 2u);
```

Đọc lại MODER/PUPDR trong debugger để xác nhận các bit vừa ghi. Dùng đồng hồ đo PB1: khi thả gần VDD, khi nhấn gần GND. Nếu mức không đúng, tháo dây tải và kiểm cấu hình/điện trước khi sửa thuật toán debounce.

```c
static uint32_t last_blink;
static bool led_on;

void led_step(uint32_t now_ms)
{
    if ((uint32_t)(now_ms - last_blink) >= 500u) {
        last_blink = now_ms;
        led_on = !led_on;
        GPIOB->BSRR = led_on ? (1u << 0u) : (1u << 16u);
    }
}
```

Gọi `led_step(HAL_GetTick())` trong vòng chính và ghép debounce ở Bài 09. Cách trên bỏ qua các nhịp đã lỡ khi main bị chậm; nó phù hợp LED báo trạng thái. Nếu cần bám pha thời gian, dùng lịch deadline có chính sách xử lý overrun rõ ràng hoặc timer phần cứng. Không đổi sang vòng while “bù hết nhịp” nếu điều đó có thể khóa main quá lâu.

Nghiệm thu: LED có chu kỳ đầy đủ khoảng 1 giây, nút được đếm một lần mỗi lần nhấn ổn định, LED không dừng khi giữ nút. Giải thích được vì sao set/reset qua BSRR không có race read-modify-write như `ODR ^= mask`.

## H3 Timer tự chạy và PWM có phép tính kiểm chứng

Tạo một project riêng với timer kernel clock đã xác nhận là 64 MHz. Trong CubeMX chọn TIM2 làm Internal Clock, prescaler 63, counter period 0xFFFFFFFF, upcounter; không bật IRQ cho phép đo ban đầu. Sau `MX_TIM2_Init()`, gọi `HAL_TIM_Base_Start(&htim2)`. Counter tăng một lần mỗi microsecond theo clock danh định.

```c
uint32_t begin = __HAL_TIM_GET_COUNTER(&htim2);
work_under_test();
uint32_t end = __HAL_TIM_GET_COUNTER(&htim2);
uint32_t elapsed_us = (uint32_t)(end - begin);
```

Hiệu modulo đúng nếu khoảng đo ngắn hơn một vòng counter. Nếu có ngắt xen vào, elapsed là thời gian trôi qua gồm thời gian ngắt, không chỉ thời gian instruction của hàm. Đặt GPIO marker quanh hàm, dùng scope để so sánh kết quả và kiểm ảnh hưởng của log/debugger.

Để tạo PWM, dùng TIM3_CH1 trên chân AF được datasheet xác nhận, ví dụ PA6 khi board cho phép. Đặt PSC=0, ARR=3199, CCR1=800, PWM mode 1, polarity high, bật preload. Tại 64 MHz, chu kỳ là 50 µs và thời gian high 12,5 µs.

1. Khởi tạo GPIO AF và timer bằng mã sinh của project. Gọi `HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1)`.
2. Đo ở chân thật; counter chạy trong debugger chưa chứng minh mux GPIO đúng.
3. Đổi CCR1 thành 1600 bằng macro driver; kiểm duty 50% và không xuất hiện xung bất thường khi preload hoạt động.
4. Tắt PWM bằng API dừng, quan sát mức chân sau dừng. Mức sau dừng phải được thiết kế theo idle state/polarity/pin mode; không suy ra “dừng timer” luôn nghĩa đầu ra bằng 0.
5. Đổi APB prescaler, tính lại timer clock bằng RCC/TIMPRE. So tần số đo với bảng clock để tìm lỗi hiểu sai clock timer.

Báo cáo ghi PSC, ARR, CCR, kernel clock, period/high-time dự kiến, giá trị đo và sai lệch phần trăm. Dùng công thức sai lệch `100 × (đo − dự kiến)/dự kiến`, đồng thời ghi độ chính xác của dụng cụ đo.

## H4 ADC đơn trước khi ghép DMA

Dùng PA0 làm analog input khi D1 xác nhận ADC12_INP0 cho package. Nối con chạy biến trở 10 kΩ vào input, hai đầu biến trở vào GND và nguồn analog phù hợp board. Không đưa điện áp ngoài dải vào input. Bắt đầu ở sampling time dài để giảm ảnh hưởng trở kháng nguồn; sau đó mới tối ưu tốc độ.

1. Cấu hình ADC1, độ phân giải 12 bit, single-ended, right alignment, regular rank 1, software trigger, continuous conversion tắt. Chọn kernel clock trong giới hạn datasheet.
2. Thực hiện calibration theo driver/RM khi ADC ở trạng thái cho phép. Kiểm return code, không bỏ qua chỉ vì bản release muốn chạy tiếp.
3. Start conversion, poll với timeout hữu hạn, đọc kết quả và stop nếu mode yêu cầu. Giữ thao tác này ngoài ISR.
4. Đặt biến trở ở khoảng 0%, 25%, 50%, 75% và gần toàn thang. Mỗi điểm lấy ít nhất 100 mẫu, lưu min/max/mean; đo input bằng đồng hồ độc lập.
5. Với 12 bit, code nằm trong 0…4095. Độ rộng một LSB lý tưởng là VREF+/4096. Cách đổi code về điện áp phải nhất quán mô hình và calibration; không dùng một hằng 3,3 V như thể reference luôn chính xác tuyệt đối.
6. Chọn sampling ngắn dần rồi so mean/noise với baseline. Một kết quả tăng tốc nhưng sai đáng kể cho thấy chưa đủ thời gian nạp tụ lấy mẫu hoặc có nhiễu/nguồn không phù hợp.

Nếu mọi mẫu bằng 0, kiểm pin, analog switch, rank, trigger và cấp clock. Nếu bão hòa 4095, đo điện áp input/reference và kiểm đường nối. Nếu số đọc đúng khi debugger dừng nhưng sai khi chạy nhanh, xem settling, trigger rate, overrun và mạch nguồn trước khi quy lỗi cho compiler.

## H5 ADC trigger timer và GPDMA dạng vòng

Mục tiêu là hai kênh ADC, mỗi kênh 10 ksample/s; xử lý theo nửa buffer. Chọn timer/TRGO nằm trong bảng external trigger của ADC1 trong RM0481. Không chỉ chọn một timer bất kỳ vì nó có nút “TRGO” trong tool.

Bảng cấu hình cần ghi vào báo cáo:

| Thành phần | Giá trị bài thực hành | Lý do |
|---|---|---|
| Timer TRGO | 10 kHz, update event | Mỗi event khởi động một chuỗi regular |
| ADC regular | Hai rank, continuous tắt | Hai mẫu cho mỗi trigger |
| ADC data | 12-bit right aligned | Lưu được trong uint16_t |
| DMA direction | Peripheral to memory | Đọc ADC data register |
| Address | Peripheral fixed, memory increment | Ghi các mẫu liên tiếp |
| Data width | Halfword theo cấu hình ADC/driver | Tránh nhầm byte, halfword, word |
| Circular behavior | Linked list vòng của GPDMA | Lặp lại buffer theo cơ chế H5 |
| Buffer | 512 phần tử uint16_t | 256 sample cho mỗi kênh |
| Half event | 256 phần tử | 128 sample/kênh, deadline 12,8 ms |

Layout dự kiến là CH0[0], CH1[0], CH0[1], CH1[1]… nếu rank/data format được cấu hình tương ứng. Kiểm bằng hai điện áp khác nhau: kênh 0 gần thấp, kênh 1 gần cao. Nếu dữ liệu không xen kẽ như dự kiến, dừng để xem rank, request, burst và width; không sửa parser bằng cách bỏ ngẫu nhiên byte.

Khởi tạo GPIO/ADC/timer/GPDMA, xây và liên kết node DMA đúng driver, chuẩn bị buffer, calibration ADC, arm DMA/ADC rồi mới bật timer. Với API nhận byte count, 512 halfword cần 1024 byte; với API nhận số phần tử, truyền 512. Tên tham số `Length` không đủ để kết luận đơn vị; đọc prototype và implementation của driver đang dùng.

ISR half/full chỉ ghi sự kiện chứa half-index và sequence. Task có thể đọc trực tiếp nửa vừa hoàn thành nếu bảo đảm xử lý xong trước khi DMA quay lại. Nếu không có bảo đảm đó, cần copy kịp sang buffer owned hoặc dùng pool nhiều buffer/descriptor. Một sequence check giúp phát hiện dữ liệu đã quá hạn, nhưng không tự ngăn DMA ghi đè giữa lúc task đang đọc.

```c
/* Xử lý một half đã được cấp quyền đọc và còn hiệu lực. */
uint32_t sum0 = 0u, sum1 = 0u;
for (size_t n = 0; n < 128u; ++n) {
    sum0 += half[2u * n];
    sum1 += half[2u * n + 1u];
}
float mean0 = (float)sum0 / 128.0f;
float mean1 = (float)sum1 / 128.0f;
```

Giới hạn sum mỗi kênh là 128×4095=524160, vừa uint32_t. Khi tính tổng bình phương hoặc tăng block size, tính lại giới hạn; đừng cho rằng kiểu đủ ở bài trước sẽ luôn đủ. Thu thập thời gian xử lý lớn nhất, block sequence gap và ADC/GPDMA error counter. Thêm tải CPU đến khi deadline bị vi phạm để chứng minh cơ chế báo mất block có hoạt động.

## H6 DAC phát sin và kiểm tra loopback

Tạo LUT 100 mẫu 12-bit theo `2048 + 1500 × sin(2πn/100)`, làm tròn và giới hạn 0…4095. Đây là dạng sóng một chu kỳ, không lặp thêm điểm n=100 vì điểm đó trùng n=0. Nạp LUT trước khi DMA đọc; không sửa LUT đang được DMA sử dụng nếu chưa có cơ chế đổi buffer.

Đặt timer trigger 100 kHz, DAC channel 1, normal mode, buffer theo tải, GPDMA/driver transfer đúng data register và alignment. Phát dạng vòng bằng cơ chế được hỗ trợ. Khởi động DAC/DMA và chuẩn bị mẫu đầu theo RM trước khi bật trigger. Với 100 mẫu mỗi chu kỳ, output có tần số 1 kHz.

Đo PA4 khi chưa nối ADC: DC offset, Vpp và tần số. Sau đó nối loopback qua đường phù hợp và lấy ADC ở 10 ksample/s. Tỷ lệ này cho khoảng 10 mẫu mỗi chu kỳ; đủ minh họa, chưa phải phép đo chất lượng DAC toàn diện. Hai converter cùng tham chiếu có thể làm một số sai số reference triệt tiêu trong tỷ số, nên cần phép đo ngoài để đánh giá điện áp tuyệt đối.

Dừng DMA có chủ đích trong bản thử và ghi DAC underrun/trạng thái output. Yêu cầu ứng dụng phải định nghĩa giữ mẫu cuối, chuyển về mức an toàn hay dừng hệ thống khi mất stream. Timer vẫn chạy không chứng minh waveform vẫn đúng.

## H7 UART từ byte đến packet đã kiểm CRC

Giữ project UART1 115200 baud, 8 data bit, no parity, 1 stop bit, TX/RX. Chọn AF/pin theo board. USB-UART phải có logic 3,3 V và GND chung. TX của một bên nối RX bên kia; không nối nhầm TX với TX.

```c
/* Ví dụ phát đồng bộ ngắn trong main, không đặt trong ISR. */
const uint8_t probe[] = {0x55u, 0x00u, 0xFFu};
HAL_StatusTypeDef st = HAL_UART_Transmit(
    &huart1, (uint8_t *)probe, sizeof probe, 100u);
if (st != HAL_OK) {
    /* Ghi lỗi hoặc chuyển sang trạng thái chẩn đoán. */
}
```

Đo byte 0x55: start thấp, data LSB-first là 1,0,1,0,1,0,1,0, stop cao. Một bit khoảng 8,68 µs và một ký tự 8N1 khoảng 86,8 µs. Nếu analyzer cần invert signal mới đọc được, kiểm tầng điện và cấu hình invert của UART; không coi đó là lý do để sửa payload.

Tiếp theo build và chạy codec C/Python đi kèm trên máy tính. Frame mẫu là `A5 5A01 02 34 12 02 00 01 02 E1 81`. Nó có 12 byte, chiếm 120 bit đường dây ở 8N1, thời gian lý tưởng khoảng 1,042 ms tại 115200 baud. Phần CRC được lưu little-endian; không đổi thứ tự byte chỉ vì hex viewer hiển thị số 0x81E1.

Khi ghép RX DMA, gọi parser bằng đúng số byte mới nhận. Lưu vị trí DMA lần trước và tính delta qua wrap; xử lý khả năng HT/TC/IDLE báo cùng một đoạn. Không để parser hoặc tác vụ khác giữ pointer vào ring sau khi DMA có thể ghi đè. Khi queue đầy, tăng counter và áp chính sách drop/backpressure đã viết trong spec.

Nghiệm thu bằng các phép thử: gửi từng byte, chia frame tại mọi vị trí, nối nhiều frame trong một write, payload chứa A5 5A, length quá giới hạn, CRC sai, timeout giữa frame và sender reboot. UART IDLE chỉ là sự kiện vận chuyển; parser vẫn phải kiểm version/length/CRC/schema trước khi phát command.

## H8 Đọc trace I2C SPI LIN và CAN theo một mẫu thống nhất

Mỗi trace phải có ảnh tín hiệu, bản decode, byte raw và diễn giải semantic. Analyzer decode thành công không chứng minh command đúng cho thiết bị.

Với I2C target địa chỉ 7-bit 0x52, giao dịch đọc hai byte ở register 0x0F được viết thành: START → A4 → ACK → 0F → ACK → repeated START → A5 → ACK → D0 → ACK → D1 → NACK → STOP. Khoanh từng ACK/NACK, xác định bên đang lái SDA ở nhịp thứ 9. NACK cuối do controller tạo là kết thúc bình thường. Đo rise time riêng; decoder có thể vẫn đọc được một bus đã vi phạm timing margin.

Với SPI mode 0, SCK idle thấp, sample ở cạnh lên. Với thiết bị hỗ trợ lệnh read 03 và địa chỉ 24-bit, đọc tại 0x001234 gồm MOSI `03 00 12 34` rồi các dummy byte để nhận data. Giữ CS active qua toàn giao dịch. Đây là ví dụ profile thiết bị nhớ, không phải mọi SPI chip đều hiểu opcode 03. Kiểm bit order, byte order, dummy cycles và thời điểm nhả CS bằng datasheet target.

Với LIN ID 0x12, tính PID 0x92; data 01 02 có enhanced checksum 0x6A. Trace cần có break, delimiter, sync 55, PID 92, data 01 02 và checksum 6A. Một frame “UART decode đúng” nhưng không có break/sync/schedule hợp lệ vẫn chưa chứng minh node LIN hoạt động đúng. Đổi một bit PID hoặc checksum và kiểm receiver từ chối.

Với Classical CAN standard data frame, nhận diện SOF 1 bit, ID 11 bit, RTR 1 bit, IDE 1 bit, r0 1 bit, DLC4 bit, data theo DLC, CRC sequence 15 bit, CRC delimiter 1 bit, ACK slot 1 bit, ACK delimiter 1 bit và EOF 7 bit. Đây là cấu trúc trước khi cộng bit stuffing và intermission. Đừng đếm số bit trên dây bằng cách chỉ cộng các trường nếu analyzer đã chèn/ẩn stuff bit. Trong CAN FD, dùng cấu trúc FD/DLC tương ứng ở Bài 29.

Ở mỗi bus, cố ý tạo một lỗi được kiểm soát rồi ghi: lỗi điện hoặc protocol nào, hardware flag nào, timeout nào, bước recovery nào và dữ liệu nào có thể mất. Báo cáo “đã giao tiếp được” mà không có thử lỗi chưa đáp ứng mục tiêu bài học.

## H9 Ghi cấu hình vào flash theo giao dịch

Bắt đầu trên vùng flash thử đã loại khỏi linker ứng dụng. Ghi rõ granularity erase/program và alignment theo RM, voltage/clock và khả năng đọc trong khi ghi. Không chọn một địa chỉ “trông trống” từ debugger vì lần build sau code có thể lớn tới đó.

Thiết kế ít nhất hai vùng erase độc lập. Mỗi record có magic, schema version, sequence, payload length, payload CRC và commit marker. Ghi body trước, kiểm lại, ghi commit cuối theo quy tắc program flash. Khi boot, chỉ chọn record có commit hợp lệ, length/range hợp lệ và CRC đúng. CRC không được dùng làm chữ ký chống giả mạo.

Trình tự kiểm: ghi A hợp lệ; bắt đầu ghi B; reset lần lượt trước erase, trong erase, sau header, sau payload và trước/sau commit. Sau mỗi lần, firmware phải chọn được A hoặc B theo policy, không dùng nửa record. Không erase bản hợp lệ cuối trước khi bản mới được xác nhận. Khi sắp hết chỗ, garbage collection cũng là một giao dịch cần power-cut test.

Test trên board nguồn điều khiển được. Software reset không tái hiện đầy đủ sự cố mất nguồn giữa program pulse; báo cáo phải phân biệt hai loại thử. Log số chu kỳ erase và dự tính tuổi thọ theo tần suất ghi, tránh lưu cấu hình mỗi vòng main.

## H10 Kiểm chứng cô lập thay vì chỉ vẽ hai vùng flash

Tạo build thử với service control và app. Lập bảng CPU privilege, Secure attribution, vùng flash/RAM, peripheral, GPDMA, IRQ, clock/reset và ai có quyền đổi các cấu hình đó. Mỗi quyền phải có phép thử cho phép và từ chối.

1. Trước hết chạy hợp lệ: app gửi request, service kiểm giá trị và trả snapshot. Nếu đường hợp lệ không chạy, một access fault chưa chứng minh policy đúng.
2. Trong build thử, app ghi trực tiếp control state; xác nhận fault và giá trị không bị đổi. Ghi stacked PC và fault status bằng ELF đúng build.
3. App dùng DMA để ghi cùng vùng; xác nhận GTZC/channel security chặn đường này. Kết quả CPU bị chặn không thay phép thử DMA.
4. App thử thay RCC/reset hoặc GPIO của control; kiểm ownership của cả đường phụ trợ.
5. App gửi pointer ngoài vùng, length gây overflow, service ID sai, request trùng và flood liên tục. Service không được đi tới địa chỉ tùy ý do app chọn.
6. Đo control jitter khi app chạy hợp lệ tải lớn và khi app lỗi. Ghi policy cho fault: restart app, reset toàn chip hoặc giữ trạng thái suy giảm. Khả năng restart app cần thiết kế dọn tài nguyên; TrustZone không tự cung cấp một hệ điều hành nhiều process.

Tiêu chí kết thúc: ma trận quyền đã thử đủ, không có thay đổi control state trái phép, timing trong cận đã đặt ở bài thử và có hành vi xác định cho mọi lỗi đã liệt kê. Không kết luận “mọi lỗi app không thể ảnh hưởng control” trên một MCU đơn lõi; reset, nguồn, bus và code đặc quyền còn là tài nguyên chung.


# Phần bổ sung các chế độ ngoại vi STM32H523

Phần này mở rộng Bài 07 đến Bài 48. Học cùng bài gốc, không dùng riêng như tài liệu cấu hình cho mọi STM32. Một mode chỉ áp dụng khi instance, package và revision silicon hỗ trợ. Các ví dụ số học dùng clock giả định đã ghi rõ; phải thay bằng clock đo được trên board. Các lab là quy trình thực hành, chưa phải firmware đã được xác nhận trên phần cứng của người học.

# Chủ đề 13 Nguồn clock và đường tín hiệu

## Bài 53 Phân tích nguồn reset và chuyển trạng thái hoạt động

### Cấu tạo và nguyên lý

PWR quản lý regulator, voltage scaling và trạng thái nguồn; RCC quản lý clock và reset. CPU ngủ không đồng nghĩa mọi ngoại vi dừng. Phải xét riêng nguồn của khối, kernel clock, bus clock, trạng thái RAM và nguồn đánh thức. VBAT cấp miền backup; không dùng VBAT để vận hành toàn bộ MCU. VDDA/VREF+ quyết định chất lượng analog; tụ VCAP ổn định regulator và không phải chân cấp nguồn tải ngoài. H523 dùng LDO, không gán cấu hình SMPS của H7/U5/L5 vào chip này.

Run thực thi lệnh; Sleep dừng hoạt động CPU trong khi các khối được cấp clock có thể tiếp tục; Stop giảm mạnh clock/nguồn nhưng giữ các tài nguyên được quy định; Standby đưa nhiều miền về trạng thái mất nội dung và thoát bằng quy trình giống reset. Retention là giữ dữ liệu, không bảo đảm ngoại vi tiếp tục vận hành. Bảng low-power trong datasheet là căn cứ cho từng nguồn wakeup, không suy diễn từ tên mode.

VOS chọn mức điện áp nội bộ để đáp ứng tần số. Tăng hiệu năng: yêu cầu mức điện áp phù hợp, chờ ready, tăng flash latency, rồi tăng clock. Giảm hiệu năng: giảm clock trước, sau đó giảm điện áp/latency theo trình tự RM. Không đổi đồng thời nhiều tham số rồi chỉ kiểm tra xem chương trình có chạy hay không.

Reset do nguồn, chân NRST, watchdog, phần mềm và miền backup có phạm vi khác nhau. Đọc và lưu reset flags trước khi xóa. BOR phát hiện điện áp thấp để reset; PVD cung cấp thông báo mức điện áp, không bảo đảm đủ thời gian ghi flash. Chỉ ghi khi dự trữ năng lượng và thời gian program đã được tính.

### Thiết kế dự án và thực hành

1. Tạo bảng từng tài nguyên: miền nguồn, clock, retention, wakeup, trạng thái sau reset. Điền GPIO, SRAM, RTC, UART và DMA của board.
2. Chạy LED và timer; đo dòng Run và Sleep sau khi loại tải LED, ST-LINK, USB và regulator của board khỏi phần đo MCU.
3. Vào Stop với một nguồn wakeup đã xác nhận. Lưu timestamp RTC, kiểm tra clock sau wakeup và khởi tạo lại những driver cần thiết.
4. Đo dòng trung bình của chu kỳ làm việc 10 ms trong mỗi giây. Tính Iavg = (Irun×trun + Isleep×tsleep)/T; thêm năng lượng chuyển trạng thái và cảm biến.
5. Thử watchdog reset và software reset; log nguyên nhân đúng một lần. Thử standby với dữ liệu có checksum trong backup và SRAM thông thường để phân biệt retention.

**Đạt:** có sơ đồ nguồn, trace wakeup, bảng RAM giữ/mất và phép đo dòng kèm điều kiện. **Lỗi thường gặp:** debug giữ clock làm sai dòng; cờ wakeup cũ khiến vừa ngủ đã thức; GPIO floating; clock UART chưa phục hồi; watchdog vẫn chạy trong lúc ngủ. [H523 DS14540 mục Power supply; RM0481 PWR/RCC]

## Bài 54 Clock mux PLL CRS và thay đổi clock khi đang chạy

### Các chế độ và quan hệ

Oscillator tạo clock; PLL nhân/chia để tạo tần số; mux chọn nguồn; prescaler chia cho bus; kernel clock cấp logic ngoại vi. Bật bus clock giúp truy cập thanh ghi, không thay thế kernel clock. Timer clock có thể khác APB clock do quy tắc prescaler/TIMPRE. Baud UART, SPI SCK, ADC sampling, USB và I3C đều cần tính từ nguồn thực tế của chính khối.

HSE/LSE crystal mode dùng mạch dao động với tinh thể và tụ tải; bypass nhận clock ngoài. Không cấu hình bypass cho crystal. HSI/CSI/HSI48/LSI là RC nội; sai số thay đổi theo nhiệt độ và điện áp. LSE phù hợp RTC chính xác; LSI phù hợp chi phí thấp khi chấp nhận sai số. PLL có input range, VCO range, divider và fractional mode; fractional hữu ích tạo clock audio nhưng phải xét jitter.

CSS giám sát lỗi oscillator theo khả năng của nguồn. CRS hiệu chỉnh HSI48 bằng nguồn đồng bộ hỗ trợ; clock recovery không có nghĩa mọi RC khác cũng được hiệu chỉnh. MCO đưa một nguồn clock ra chân để đo, có giới hạn tốc độ chân và hệ số chia.

### Trình tự và lab

1. Lập cây clock từ oscillator đến CPU, AHB, APB và từng kernel. Ghi cả tần số và giới hạn theo VOS.
2. Đưa HSI hoặc clock đã chia ra MCO; đo bằng scope/counter. Kiểm chứng SystemCoreClock bằng timebase độc lập.
3. Tạo hai profile clock. Dừng giao dịch đang chạy, đổi clock theo trình tự VOS/flash/PLL, cập nhật timebase và cấu hình timing của driver.
4. Đo UART trước/sau chuyển profile. Nếu peripheral có kernel độc lập, chứng minh baud giữ nguyên; nếu cùng nguồn thay đổi, tính và nạp divider mới.
5. Lập fault policy khi nguồn clock lỗi: ngắt phát công suất, chuyển nguồn dự phòng, ghi trạng thái, khôi phục có kiểm soát. Chỉ thử ngắt nguồn oscillator trên board có thiết kế cho phép.

**Đạt:** bảng clock có đường nguồn đầy đủ; tần số đo khớp sai số dự kiến. **Lỗi:** áp công thức timer APB cho mọi cấu hình; đổi PLL đang làm nguồn SYSCLK; UART đang gửi bị đổi baud; dùng LSI đo thời gian với yêu cầu ppm. [RM0481 RCC/CRS]

## Bài 55 GPIO EXTI và định tuyến tín hiệu nội bộ

### Mode GPIO và đặc tính điện

Input đọc mức logic; output điều khiển chân; alternate function nối chân với ngoại vi; analog ngắt đường digital liên quan theo cấu trúc chân để phục vụ analog/giảm tiêu thụ. Push-pull chủ động kéo lên/kéo xuống. Open-drain chỉ kéo xuống hoặc thả nổi, cần pull-up; phù hợp I2C và wired-AND. Pull-up/down nội yếu và không thay điện trở ngoài khi bus có yêu cầu rise time.

Output speed chọn đặc tính chuyển cạnh, không tạo tần số output. Cạnh quá nhanh gây ringing/EMI; quá chậm có thể vi phạm timing. Chân 5 V tolerant chỉ được dùng trong điều kiện datasheet quy định, không mặc định tolerant trong analog mode hoặc khi mất nguồn. IDR đọc trạng thái chân; ODR giữ latch đầu ra. BSRR set/reset theo bit giúp tránh read-modify-write làm mất thay đổi từ ISR/task khác.

EXTI phân biệt interrupt và event. Interrupt chuyển CPU vào handler qua NVIC; event có thể phục vụ cơ chế đánh thức mà không tương đương handler. Chọn cạnh rising/falling/hai cạnh theo tín hiệu; đường GPIO EXTI cùng số pin cần mux chọn port. NVIC priority quyết định preemption; số nhỏ thường ưu tiên cao hơn, nhưng số bit implement phải lấy từ CMSIS.

### Lab theo mode

1. Đặt latch về mức an toàn trước khi chuyển MODER sang output; kiểm tra có glitch trên chân enable động cơ không.
2. Với LED/tải nhỏ phù hợp, so sánh push-pull và open-drain có pull-up; đo mức cao và thời gian lên.
3. Đổi speed trong cùng PWM, đo cạnh và ringing. Dùng probe ground spring để tránh kết luận sai do dây mass dài.
4. Đưa nút vào EXTI hai cạnh; timestamp và debounce ngoài ISR. Chứng minh bounce tạo nhiều cạnh nhưng chỉ một sự kiện ứng dụng.
5. Cấu hình AF UART và cố ý sai AF; phân biệt lỗi pin routing với lỗi baud bằng đo chân. Thử GPIO lock trên cấu hình thử nghiệm và xác nhận phạm vi khóa.

**Đạt:** bảng pin bao gồm AF, điện áp, pull, safe state, tốc độ cạnh và quyền secure. **Lỗi:** dùng analog pin như chân 5 V tolerant; hai task sửa ODR; quên clear pending; EXTI nhầm port; external pull giữ MCU bị back-power khi tắt. [RM0481 GPIO/EXTI/SBS; DS14540 pin descriptions]

# Chủ đề 14 Timer từ counter đến điều khiển động cơ

## Bài 56 Counter đếm tăng đếm giảm và center aligned

### Cấu tạo và công thức

Clock qua PSC cấp tick cho CNT. ARR đặt biên; update event thực hiện việc nạp shadow và có thể tạo IRQ/DMA/TRGO. PSC là giá trị chia trừ một: ftick = fTIM/(PSC+1). Với edge-aligned chạy tự do, không repetition: fperiod = ftick/(ARR+1). Up đếm 0 đến ARR; down đếm ARR đến 0. Hai mode có cùng độ dài chu kỳ nhưng quan hệ compare và vị trí cạnh khác nhau.

Center-aligned chạy hai chiều giữa 0 và ARR. Với ARR>0, một vòng ổn định có 2×ARR tick; fPWM = ftick/(2×ARR). Không dùng công thức ARR+1 của edge-aligned. Chọn fTIM=64 MHz và PSC=0: edge-aligned 20 kHz cần ARR=3199; center-aligned 20 kHz cần ARR=1600. Cùng tần số nhưng thang CCR và các thời điểm sự kiện khác nhau.

| Cấu hình | Cách hoạt động | Lợi ích và hạn chế |
|---|---|---|
| Up | CNT tăng rồi quay về 0 | Timebase dễ hiểu; thuận tiện timestamp và PWM thông thường |
| Down | CNT giảm rồi nạp ARR | Biểu diễn thời gian còn lại; không mặc nhiên giảm nhiễu hay tăng chính xác |
| Center mode 1 | CNT lên/xuống; cờ compare output chỉ theo chiều xuống | Chọn một thời điểm xử lý trên nhánh xuống |
| Center mode 2 | CNT lên/xuống; cờ compare output chỉ theo chiều lên | Chọn một thời điểm xử lý trên nhánh lên |
| Center mode 3 | CNT lên/xuống; cờ compare output theo cả hai chiều | Có thể xử lý hai compare mỗi vòng; phải tính tải IRQ/DMA |

CMS1/2/3 khác cách tạo cờ compare của channel output, không phải ba dạng sóng tam giác riêng. Các điều kiện tạo ADC trigger phải đối chiếu đường TRGO/TRGO2/OC thực tế; không suy luận từ cờ interrupt. “Đếm xuống rồi lên” không phải một mode độc lập chỉ vì chọn CNT ban đầu gần ARR. TIM6/TIM7 không hỗ trợ toàn bộ kiểu đếm và channel của TIM1/TIM8.

### Update preload và repetition

ARR/CCR preload tách giá trị CPU ghi với giá trị đang điều khiển waveform; update chuyển giá trị ở mốc xác định để tránh chu kỳ bị cắt. PSC cũng có cơ chế nạp riêng. UG có thể gây update và tác động prescaler/counter theo RM; URS/UDIS điều khiển các khía cạnh update, không dùng để thay việc đọc quy tắc flag. Repetition counter của advanced timer điều tiết update; trong center-aligned cần xét overflow lẫn underflow. Không lấy fIRQ=fPWM một cách mặc định.

### Lab so sánh kiểu đếm

1. Dùng TIM1 hoặc timer đã xác nhận hỗ trợ CMS. Chọn clock 64 MHz hoặc tính lại toàn bộ ví dụ theo clock thật.
2. Tạo edge-up 20 kHz duty 25%; ghi CNT/ARR/CCR và đo waveform.
3. Chuyển sang down khi counter đã dừng. Giữ tần số và duty mục tiêu bằng cấu hình phù hợp; đo vị trí cạnh so với update marker.
4. Chuyển center-aligned ARR=1600, CCR=400 cho PWM1 active-high duty xấp xỉ 25% trong cấu hình chuẩn. Đo hai cạnh đối xứng; không giữ ARR=3199 rồi gọi đó là cùng tần số.
5. Lần lượt CMS1/2/3; đếm compare IRQ bằng counter phần mềm và tạo GPIO marker ngắn. So sánh 1 hoặc 2 compare mỗi vòng khi CCR nằm trong miền nội.
6. Thử cập nhật duty giữa chu kỳ với preload tắt/bật. Lưu waveform có glitch và waveform chuyển sạch.

**Đạt:** bảng thời điểm compare/update và dạng sóng cho từng mode. **Ứng dụng motor:** center-aligned hỗ trợ bố trí switching và cửa lấy mẫu đối xứng; không tự bảo đảm dòng sạch hoặc giảm mọi loại nhiễu. [RM0481 TIM1/TIM8 CR1 CMS; TIM2–TIM5]

## Bài 57 Output compare PWM và tạo xung theo sự kiện

### Mode của channel output

Compare kiểm tra CNT với CCR để tạo hành động trên OCxREF. OCxREF là tín hiệu nội; polarity, output enable, complementary, dead-time và MOE có thể tiếp tục thay đổi tín hiệu ra chân. Vì vậy cần phân tích từng tầng thay vì chỉ nhìn CCR.

| Mode | Nguyên lý | Ứng dụng |
|---|---|---|
| Frozen | Compare không đổi OCxREF | Scheduling sự kiện mà không tạo cạnh output |
| Active on match | Match đặt reference active | Bắt đầu một cửa tín hiệu |
| Inactive on match | Match đặt reference inactive | Kết thúc cửa tín hiệu |
| Toggle | Mỗi match đảo reference | Tạo clock hoặc cạnh định thời; tần số chân phụ thuộc số match |
| Forced active/inactive | Ép reference theo cấu hình | Kiểm tra output path hoặc trạng thái điều khiển được thiết kế |
| PWM1/PWM2 | Trạng thái phụ thuộc quan hệ CNT–CCR và hướng đếm | Điều khiển duty; PWM2 đảo quan hệ reference của PWM1 |
| Combined/asymmetric PWM | Ghép quan hệ compare của các channel hỗ trợ | Tạo cửa/xung có vị trí và độ rộng linh hoạt |

Combined/asymmetric, retriggerable one-pulse và các mở rộng chỉ dùng trên instance có trong RM. Forced inactive không thay thế break phần cứng khi CPU đã treo. PWM1/PWM2 khác output polarity; chọn cả PWM2 và polarity đảo có thể đảo hai lần.

One-pulse dừng counter theo điều kiện one-pulse; tạo xung trễ bằng CCR và biên kết thúc bằng ARR với trigger hợp lệ. Repetition có thể thay số chu kỳ trên timer hỗ trợ. Khi retrigger xảy ra, timer có thể khởi động lại/kéo dài theo mode cụ thể; phải thử với khoảng trigger ngắn hơn xung. Không dùng delay phần mềm thay thế mạch định thời này.

### Lab

1. Tạo tick 1 MHz. Với toggle, đặt compare và đo số cạnh thực tế trước khi tính tần số output.
2. Với PWM1/PWM2 và hai polarity, lập bảng bốn cấu hình; đo duty của mức cao và duty active riêng.
3. Kiểm tra CCR=0, CCR gần ARR và yêu cầu 100%. Quy tắc endpoint theo mode; không mặc định CCR=ARR là 100% ở upcounting.
4. Cấu hình một trigger vào để tạo xung trễ 100 µs, rộng 50 µs. Tính và kiểm chứng endpoint bằng scope, xét tick đầu/cuối.
5. Gửi trigger hai lần sát nhau; ghi bị bỏ qua, retrigger hay kéo dài. Đổi ARR/CCR bằng preload để không làm méo xung.
6. Nếu timer hỗ trợ combined/asymmetric, dùng hai compare tạo cửa ADC; đo cửa theo từng nhánh center-aligned.

**Đạt:** waveform đầy đủ các mode áp dụng, cấu hình enable/polarity và bảng endpoint. **Lỗi:** timer đang chạy nhưng MOE chưa bật; CHxN thiếu chân AF; compare IRQ đúng nhưng output bị break giữ inactive. [RM0481 timer CCMR/CCER/BDTR]

## Bài 58 Input capture PWM input encoder và clock ngoài

### Đường input và các lựa chọn

Input qua mux, polarity, filter và prescaler trước khi capture. Capture chốt CNT vào CCR bằng phần cứng; ISR đến trễ vẫn đọc timestamp đã chốt, trừ khi overcapture. Filter yêu cầu tín hiệu ổn định qua nhiều mẫu; nó giảm glitch nhưng thêm delay và giới hạn xung ngắn. Input prescaler bỏ bớt các sự kiện capture, khác PSC chia counter clock.

Direct input nối channel với TI tương ứng; indirect input cho channel khác quan sát cùng TI; TRC nối với trigger controller khi hỗ trợ. PWM input thường ghép hai channel để đo period và high-time, kết hợp slave reset. Channel dùng capture không đồng thời tạo PWM output trong cùng cấu hình.

Encoder TI1/TI2/cả hai lựa chọn cạnh được đếm để suy ra hướng từ quan hệ pha. Mức giải mã x2/x4 cần đối chiếu cấu hình và định nghĩa PPR/CPR của encoder. Encoder count không tự đổi thành tốc độ: tốc độ cần chênh count theo thời gian hoặc đo period ở tốc độ thấp. Counter wrap và đổi chiều cần xử lý rõ.

External clock mode 1 đếm sự kiện qua đường trigger/slave controller; external clock mode 2 dùng ETR theo khả năng timer. Gated chỉ cho counter chạy khi gate cho phép; trigger mode khởi động theo trigger; reset mode đưa counter về mốc khi trigger xuất hiện. Một tín hiệu clock ngoài cần thỏa timing/filter/prescaler, không thể đếm tùy ý đến tần số CPU.

### Tính toán và lab

1. Phát PWM 1 kHz, 30% từ timer khác; nối vào TI hợp lệ. Capture tick 1 MHz sẽ cho period khoảng 1000 và high khoảng 300 tick.
2. Thử direct/indirect + reset để đo PWM input. So sánh với capture hai cạnh độc lập; đánh dấu sample invalid khi overcapture.
3. Giảm xuống 1 Hz; chọn counter/PSC để không wrap nhiều lần giữa hai capture. Nếu mở rộng bằng overflow IRQ, kiểm tra race giữa CCR và overflow flag.
4. Chèn glitch ngắn bằng nguồn phát có kiểm soát. Tăng filter rồi đo delay và tần số tối đa còn nhận đúng.
5. Encoder: quay một vòng thuận/ngược, ghi count. Tính rpm = delta_count×60/(counts_per_rev×delta_t). Thử đổi chiều gần wrap.
6. External clock: đếm số xung trong cửa 100 ms; so sánh gated với CPU đọc CNT đầu/cuối để thấy ảnh hưởng latency.

**Đạt:** sai số được phân tích theo lượng tử tick, filter và nguồn tín hiệu; không trả kết quả hợp lệ sau mất capture. [RM0481 timer input capture/slave controller]

## Bài 59 Đồng bộ timer dead time break và ADC cho động cơ

### Nguyên lý hệ thống

Master phát TRGO/TRGO2; slave nhận qua ITR hoặc input khác. Bảng routing quy định timer nào nối được với timer nào. Đồng bộ phần cứng giảm lệch khởi động và jitter so với lần lượt gọi HAL_Start. Reset slave tạo lại mốc; gated tạo cửa đếm; trigger start và external clock phục vụ mục tiêu khác nhau.

Advanced timer tạo CHx/CHxN qua dead-time generator để tránh hai transistor cùng dẫn. Dead-time DTG không phải luôn là số tick tuyến tính; có các miền mã hóa với hệ số khác nhau. Tính từ tDTS và mã DTG, rồi đo. Dead-time quá nhỏ gây shoot-through; quá lớn làm méo điện áp/dòng và tăng tổn hao. Polarity và driver gate có thể đảo mức; kiểm tra trạng thái cả hai transistor thực sự tắt.

Break nhận fault phần cứng, đưa output về trạng thái quy định mà không chờ ISR. Cần cấu hình polarity, filter, nguồn break, OSSI/OSSR, idle state và MOE. AOE/re-arm tự động chỉ dùng khi yêu cầu hệ thống cho phép. Lock bảo vệ cấu hình quan trọng theo mức và reset; không phải quyền truy cập memory hay TrustZone.

Trong inverter ba pha, timer tạo switching; ADC đo dòng tại cửa hợp lệ; DMA chuyển mẫu; control tính duty kế tiếp. Cửa đo phải đủ rộng cho settling + acquisition, trước conversion. Với một shunt, có duty khiến cửa đo rất hẹp; center-aligned không giải quyết toàn bộ bài toán này.

### Lab tiến triển

1. Dùng ba channel cùng timer tạo PWM 20 kHz center-aligned; cập nhật CCR qua preload tại mốc chung.
2. Quan sát CHx và CHxN ở mức logic, chưa nối công suất. Chọn dead-time mục tiêu 500 ns, tính mã theo clock và đo thực tế.
3. Đưa fault vào break; đo thời gian từ fault đến cả output ở safe state. ISR chỉ ghi nguyên nhân; cấm phụ thuộc ISR để tắt gate.
4. Dùng compare nội tạo ADC trigger. Đặt marker trigger và marker end-of-conversion; đo để phân biệt acquisition với ISR latency.
5. Thay duty từ 10% đến 90%; xác định vùng không đủ cửa lấy mẫu cho topology giả định. Đánh dấu mẫu invalid hoặc điều chỉnh chiến lược.
6. Cho app treo hoặc trễ control; watchdog/deadline monitor đưa hệ thống về trạng thái thiết kế. Nhả fault cần acknowledge trước khi re-arm.

**Đạt:** timing budget gồm PWM, ADC, DMA và control; chứng minh fault tắt output độc lập CPU. [RM0481 TIM1/TIM8 synchronization/BDTR/ADC trigger]

## Bài 60 LPTIM RTC TAMP và watchdog theo từng chế độ

LPTIM dùng clock và cơ chế đồng bộ riêng, có thể phục vụ tác vụ khi CPU ngủ trong mode được cho phép. Các chức năng gồm timebase, PWM/one-shot, counter ngoài và encoder trên instance hỗ trợ. ARR/CMP có handshake đồng bộ; phải chờ cờ xác nhận trước khi ghi tiếp theo quy tắc RM. Clock LSE/LSI cho tiết kiệm điện nhưng độ phân giải và sai số khác timer nhanh.

RTC gồm prescaler, calendar, subsecond, alarm và wakeup timer. Alarm so khớp các trường với mask; wakeup timer tạo chu kỳ khác cơ chế alarm. Timestamp ghi thời điểm một sự kiện; tamper phát hiện can thiệp và có thể xóa backup theo policy. Calibration điều chỉnh sai số oscillator, không sửa timestamp theo múi giờ. Giữ UTC trong thiết bị; trình bày giờ địa phương ở tầng ứng dụng.

IWDG dựa nguồn clock độc lập để phát hiện CPU/clock ứng dụng lỗi; WWDG ràng buộc cửa refresh, phát hiện refresh quá sớm lẫn quá muộn theo cấu hình. Timeout phụ thuộc clock thực tế và dung sai. Feed từ một task đơn lẻ chỉ chứng minh task đó chạy; supervisor phải kiểm tra heartbeat của các chức năng quan trọng. Debug freeze thay đổi hành vi đo, không dùng kết quả debug để khẳng định watchdog production.

### Lab

1. LPTIM: phát chu kỳ 1 s từ LSE nếu có; đo Run/Stop và thời gian xác nhận ARR/CMP. Thử external counter đúng instance.
2. RTC: đặt calendar, alarm có mask, wakeup định kỳ; log loại nguồn riêng. Đọc calendar theo quy tắc shadow/đồng bộ để tránh ngày và giờ không cùng snapshot.
3. Đo RTC với nguồn thời gian độc lập trong khoảng đủ dài; tính ppm = delta_t/T×10^6 rồi hiệu chỉnh trong giới hạn calibration.
4. TAMP: dùng chân thử nghiệm, backup data không nhạy cảm; kiểm chứng timestamp và hành vi erase đã chọn.
5. IWDG: tạo lỗi một task; supervisor ngừng feed; xác nhận reset flags. WWDG: thử feed trước cửa, trong cửa và sau cửa.

**Đạt:** bảng mode nguồn/clock/wakeup của LPTIM và RTC; ba thử nghiệm watchdog có log. [RM0481 LPTIM/RTC/TAMP/IWDG/WWDG]

# Chủ đề 15 DMA và analog theo từng cơ chế

## Bài 61 GPDMA normal circular burst FIFO request và trigger

### Định nghĩa và cấu tạo

DMA là bus master đọc nguồn và ghi đích theo cấu hình, giảm việc CPU chuyển từng phần tử. Một channel GPDMA có cấu hình địa chỉ, độ rộng, burst, block length, request/trigger, FIFO và linked-list. FIFO nằm trong DMA, khác RX/TX FIFO của UART/SPI. Request thường biểu thị peripheral cần được phục vụ; trigger điều kiện hóa thời điểm hoặc phạm vi truyền. Không dùng hai từ này thay thế nhau.

H523 có hai GPDMA, mỗi bộ tám channel. Channel 0–3 FIFO 8 byte; 4–5 FIFO 32 byte; 6–7 FIFO 32 byte và có truyền 2D/repeated block. Đây không phải DMA stream của H743. Không gán thanh ghi DBM hay FIFO threshold của DMA cổ điển vào GPDMA.

| Cơ chế | Cách hoạt động | Lựa chọn dự án |
|---|---|---|
| Normal | Hoàn thành block hữu hạn rồi kết thúc theo cấu hình | Command TX, đọc sensor có kích thước biết trước |
| Circular linked-list | Node cuối liên kết về node đầu hoặc một node trước | ADC/audio liên tục, luân phiên buffer |
| Source/destination fixed | Không tăng địa chỉ phía đó | Thanh ghi dữ liệu peripheral |
| Increment | Địa chỉ tiến theo độ rộng/offset quy định | Buffer tuyến tính |
| Burst | Nhóm beat đọc/ghi với độ dài đã lập trình | Tăng hiệu quả bus khi endpoint cho phép |
| FIFO | Đệm giữa các giao dịch source và destination | Ghép độ rộng, tách nhịp truyền; bị giới hạn dung lượng |
| Trigger block | Trigger mở đầu một block | Chuyển một block theo sự kiện |
| Trigger repeated block | Điều kiện hóa nhóm 2D trên channel hỗ trợ | Tác vụ nhiều hàng theo một sự kiện |
| Trigger link | Điều kiện hóa tải liên kết | Chuyển cấu hình theo sự kiện |
| Trigger burst | Điều kiện hóa từng burst lập trình | Tiến từng nhóm dữ liệu theo trigger |

Burst length đo bằng beat; block length và width phải có đơn vị rõ. Source/destination có thể có độ rộng hoặc burst khác nhau trong cấu hình hợp lệ; packing/unpacking, padding, exchange phải tuân thủ RM. Một thanh ghi UART không cho phép tự ý dùng burst dài như SRAM. FIFO không biến một peripheral đơn beat thành endpoint burst tùy ý.

### Trigger request và ownership

Ví dụ timer TRGO kích ADC; ADC hoàn tất conversion tạo DMA request; GPDMA đọc ADC_DR vào RAM. Timer không nhất thiết được chọn làm trigger của GPDMA. Nếu thêm trigger GPDMA, cả điều kiện request và trigger có thể phải thỏa theo mode. TRIGPOL chọn rising/falling; TRIGSEL chọn nguồn hợp lệ; TRIGM chọn granularity. Trigger arriving quá nhanh có thể gây overrun; DMA không có queue vô hạn.

Circular không bảo đảm không mất dữ liệu. CPU chỉ được đọc vùng DMA đã bàn giao; khi DMA quay lại, CPU phải hoàn thành hoặc bỏ block có thống kê. Half-transfer event phụ thuộc transfer-event mode và cấu trúc linked-list; không mặc định mọi node đều phát half callback như normal.

### Lab định lượng

1. Memory-to-memory normal 1024 byte, source/destination tăng; verify mọi byte, không chỉ checksum. Thử địa chỉ và width hợp lệ.
2. ADC timer-trigger 10 ksample/s vào hai buffer 128 mẫu bằng circular linked-list. Deadline mỗi buffer 12,8 ms; xử lý quá deadline phải tăng overrun counter.
3. UART TX normal: DMA complete là đã chuyển dữ liệu tới peripheral; chờ UART TC mới coi đường dây gửi xong hoặc hạ RS485 DE.
4. Memory-to-memory: so sánh burst 1 và các burst hợp lệ lớn hơn trên channel FIFO khác nhau. Đo throughput đồng thời đo latency của DMA channel quan trọng.
5. Chọn trigger block, sau đó trigger burst; phát số trigger hữu hạn và đo byte đã truyền. Tạo trigger quá nhanh, kiểm tra trigger overrun flag và recovery.
6. Dừng bằng suspend/abort theo driver/RM; chờ trạng thái dừng, ghi lượng dữ liệu hợp lệ rồi mới tái sử dụng buffer. Không sửa descriptor đang bị DMA đọc.

**Đạt:** log request/trigger riêng, bảng burst-width-FIFO hợp lệ, ownership diagram và thống kê mất block. [RM0481 GPDMA mục channel features, transfers, triggers, TR2]

## Bài 62 Linked list truyền 2D và thiết kế buffer DMA

Linked-list lưu cấu hình node trong RAM; DMA tải node thay CPU nạp lại thanh ghi sau từng lần. Static node cung cấp cấu hình đầy đủ; dynamic node cập nhật phần cần đổi theo cơ chế phần cứng. Tối ưu node chỉ sau khi chứng minh node đầy đủ chạy đúng. Địa chỉ link, alignment, phạm vi địa chỉ và bit update là cấu trúc phần cứng, không phải con trỏ C tùy ý.

Normal chain kết thúc bằng link terminal; circular chain quay lại. Ping-pong là chiến lược hai buffer A/B; trên GPDMA có thể thực hiện bằng hai node vòng. Trên DMA cổ điển có DBM, cơ chế khác. Repeated block/2D dùng số block, length và offset để đọc/ghi dữ liệu theo hàng. Offset tính từ vị trí đã tiến qua block theo định nghĩa RM, không nhất thiết bằng stride toàn bộ hàng.

Ví dụ ảnh mỗi hàng 320 byte hữu ích nằm trong stride 336 byte. DMA copy 320 byte mỗi hàng rồi nhảy qua padding 16 byte phía nguồn; phía đích liên tục. Chỉ channel hỗ trợ 2D được dùng. Chiều tăng/giảm và giới hạn offset phải kiểm tra; không dùng phép trừ unsigned làm wrap địa chỉ.

### Data structure ứng dụng

```c
typedef enum { BUF_FREE, BUF_DMA, BUF_READY, BUF_CPU } owner_t;
typedef struct {
    void *address;
    size_t capacity_bytes;
    size_t valid_bytes;
    uint32_t sequence;
    owner_t owner;
} app_buffer_t;
```

Đây là metadata phần mềm, không phải hardware descriptor để nạp thẳng vào DMA. Owner thay đổi qua primitive đồng bộ phù hợp; volatile không thay memory ordering hoặc cache maintenance. Descriptor HAL cũng không được cast từ cấu trúc trên. Buffer phải ở RAM DMA truy cập được và phù hợp security.

### Lab

1. Tạo ba node copy ba vùng có pattern khác nhau; verify thứ tự và termination. Ghi dump cấu hình node để đối chiếu driver.
2. Tạo A→B→A, interrupt đúng điểm bàn giao; CPU kiểm sequence và valid length. Cố ý xử lý chậm để phát hiện block bị ghi đè.
3. 2D: copy ma trận 8 hàng, payload 32 byte/hàng, stride nguồn 40 byte. Sentinel ở padding và ngoài buffer phải không đổi.
4. Thử suspend ở giữa chuỗi rồi resume; phân biệt pause giữ trạng thái với abort cần khởi tạo lại.
5. Tiêm node sai địa chỉ trên vùng thử nghiệm được kiểm soát; ghi transfer error và khôi phục. Không dùng vùng chứa firmware hoặc descriptor đang vận hành.

**Đạt:** hardware node đúng alignment, buffer có ownership, không ghi ngoài vùng và không dùng data sau lỗi. **Bài toán senior:** khi app sở hữu RAM không tin cậy, secure service phải kiểm tra descriptor/address/length; DMA có thể vượt giới hạn CPU MPU nếu bus security chưa chặn. [RM0481 GPDMA linked-list/2D/GTZC]

## Bài 63 ADC theo nhóm chuyển đổi trigger và oversampling

ADC gồm mux analog, sample-and-hold, bộ chuyển đổi và đường kết quả. Resolution chọn số bit; sampling time quyết định thời gian nạp tụ lấy mẫu. Clock nhanh không cứu được source impedance quá cao. Calibration bù một phần sai số nội; không thay calibration toàn đường đo gồm sensor, OPAMP, divider và VREF.

Regular group phục vụ chuỗi đo thông thường; injected group cung cấp đường chuyển đổi riêng theo khả năng IP, hữu ích lấy mẫu điều khiển đồng bộ PWM. Single conversion thực hiện lần hữu hạn; continuous tự lặp; scan đi qua ranks; discontinuous chia chuỗi thành nhóm theo trigger. Phải kiểm tra tương thích giữa continuous, discontinuous, injected và oversampling của ADC cụ thể.

Software start và external trigger khác thời điểm bắt đầu. Trigger polarity, nguồn timer và enable thuộc bảng routing. EOC báo conversion; EOS báo cuối chuỗi; overrun báo kết quả không được tiêu thụ kịp. Overrun overwrite giữ mẫu mới nhưng mất mẫu cũ; preserve có hành vi giữ dữ liệu theo cấu hình. Chọn policy theo yêu cầu thay vì mặc định callback sẽ nhận mọi mẫu.

Oversampling tích lũy nhiều conversion rồi shift để giảm nhiễu ngẫu nhiên/tăng độ phân giải biểu diễn. Nó tăng thời gian và không tạo thêm độ chính xác tuyệt đối tương ứng số bit xuất ra. Analog watchdog so giá trị với ngưỡng trên các channel được chọn, phát hiện tín hiệu ngoài cửa mà không cần CPU đọc mọi mẫu. Dual-ADC simultaneous/interleaved chỉ dùng nếu instance/IP hỗ trợ trong RM; kiểm tra độ lệch và sai số giữa ADC.

### Lab

1. Single channel DC: đo 0 V, mức giữa và gần VREF; tính V = code×VREF/(2^N−1) với VREF đã đo và điều kiện lý tưởng.
2. Tăng sampling time khi nguồn qua điện trở lớn; quan sát code ổn định hơn. Tính budget mỗi rank = acquisition + conversion theo resolution/clock thật.
3. Scan ba ranks, timer trigger 1 kHz và DMA; ghi mapping index→rank. Tổng thời gian chuỗi phải nhỏ hơn 1 ms.
4. Thử continuous và external-triggered single sequence; đo khác biệt khoảng giữa chuỗi. Đừng để continuous vô tình bỏ qua nhịp timer mong muốn.
5. Oversampling: ghi mean/stddev trên ít nhất 1000 mẫu ở cấu hình không oversample và oversample; phân biệt averaging với ENOB đo thực tế.
6. Watchdog: sweep điện áp chậm qua hai ngưỡng; log event và hysteresis phần mềm nếu cần. Cố ý dừng DMA để kiểm chứng overrun policy.

**Đạt:** bảng từng mode và tổ hợp hợp lệ, timing budget, histogram, rank mapping và chính sách mất mẫu. [RM0481 ADC; DS14540 analog characteristics]

## Bài 64 DAC VREFBUF và đo tín hiệu nội

DAC biến code thành điện áp qua mạch chuyển đổi và buffer. Software write có thể cập nhật trực tiếp hoặc vào holding register chờ trigger. External trigger + DMA tạo waveform đều theo timer. Dual-channel có thể cập nhật đồng thời theo đường dữ liệu hỗ trợ. Noise/triangle generator của DAC là nguồn test tích hợp, không thay LUT tùy ý. Buffer enable giảm yêu cầu drive bên ngoài nhưng có output swing/settling/load riêng; disable buffer cần mạch tải phù hợp.

### Sample and hold sawtooth và các đường output

H523 hỗ trợ sample-and-hold theo DS14540 mục3.24. Chu kỳ sample nạp tụ tới điện áp mục tiêu; hold giữ điện áp khi một phần mạch được tắt; refresh nạp lại để hạn chế droop. Các khoảng sampling/holding/refresh cần clock và điều kiện mode đúng RM, không đơn giản là dừng DMA. Droop phụ thuộc tụ, leakage và tải; tải scope/probe cũng làm thay đổi phép đo. Cấu hình giữ điện áp DC cần tính sai số trong toàn hold interval. Tính năng này phải học cùng nguồn/clock Stop; không suy từ câu giới thiệu rằng mọi đường GPDMA/timer đều chạy trong mọi Stop mode.

Sawtooth dùng giá trị khởi đầu/reset và bước tăng/giảm theo trigger tương ứng, tạo ramp có kiểm soát; khác triangle tự đổi chiều theo amplitude và khác LUT waveform do DMA cấp. Chọn trigger reset/step, step size và giới hạn theo DAC implementation. Double-data DMA cấp nhiều mẫu qua giao dịch dữ liệu được hỗ trợ để giảm hoạt động bus; không nhầm với dual-channel simultaneous update hoặc double buffer DMA.

Output có thể nối ra pad, nối peripheral nội hoặc kết hợp theo mode; buffer enable/disable và normal/sample-and-hold tạo các cấu hình khác nhau. Phải kiểm pin analog routing, loading và peripheral connection. Buffer offset calibration cần trình tự enable/calibration/status của channel; không tùy ý hiệu chỉnh khi đang phát waveform cho tải điều khiển. DAC dữ liệu8bit,12bit right/left-aligned là format ghi, không phải ba DAC vật lý khác nhau. HFSEL có điều kiện clock/update rate trong RM; phải cấu hình khi clock DAC vượt miền tương ứng, không chỉ bật clock nhanh rồi giữ default.

Giá trị lý tưởng liên quan VREF; settling time giới hạn tần số waveform hữu ích. DAC underrun xảy ra khi trigger đòi mẫu nhưng nguồn dữ liệu không cấp kịp; phải chọn dừng, giữ mức hoặc phục hồi an toàn. Không đưa DAC trực tiếp vào loa, động cơ hoặc tải dòng lớn.

VREFBUF cung cấp reference theo các mức và điều kiện phần cứng hỗ trợ; high-impedance mode khác enable output. H523 chỉ có VREFBUF trên package V/Z theo bảng feature. VREFINT là nguồn tham chiếu nội để suy ra VDDA với calibration, không phải chân VREFBUF. VBAT measurement cần đường chia và điều kiện enable; DTS/temperature có calibration và thời gian ổn định riêng.

### Lab

1. DAC static ở 0%, 25%, 50%, 75%, gần full-scale; đo offset/gain và output swing có buffer. So sánh tải khác nhau trong giới hạn.
2. LUT sine 64 điểm, timer 64 kHz → sine 1 kHz. Dùng DMA vòng; đo bước mẫu, amplitude, THD nếu có thiết bị phù hợp.
3. Thử trigger phần mềm và trigger timer; đo jitter. Cố ý ngừng cấp DMA và ghi underrun/recovery.
4. Thử noise/triangle trong mode hỗ trợ; thu bằng ADC để phân tích phân bố hoặc ramp.
5. Đo VREFINT/VBAT/temperature sau thời gian ổn định; áp calibration theo nhiệt độ/điện áp được ghi trong datasheet, không hardcode mọi STM32 giống nhau.
6. Sample-and-hold: chọn tụ/tải theo datasheet, đặt sample/hold/refresh, đo droop và settling; so năng lượng với normal. Ghi mode nguồn/clock thực sự hoạt động.
7. Sawtooth: tạo ramp bằng reset/step trigger, đổi step và chiều; capture từng bước. Double-data DMA: verify thứ tự mẫu và alignment bằng pattern trước waveform.

**Đạt:** bảng trigger/buffer/generator/DMA/sample-and-hold, đo settling/droop và fault policy. **Phân biệt dòng:** H523, U5 và L5 có sample-and-hold nhưng clock, routing và điều kiện low-power phải kiểm riêng. OPAMP/COMP tích hợp không có trên H523, dùng Bài 22 ngoại vi ngoài và Bài 77 mở rộng. [DS14540 mục3.24; RM0481 DAC/VREFBUF/DTS]

# Chủ đề 16 Giao tiếp lưu trữ và bảo mật theo mode

## Bài 65 USART UART SPI I2S theo cấu hình và kết thúc giao dịch

### UART USART và giao thức trên đường dây

Asynchronous UART có start, data LSB-first, parity tùy chọn, stop; baud và oversampling đặt timing. USART synchronous bổ sung clock; half-duplex dùng chung đường truyền theo chế độ hỗ trợ; RS485 dùng DE và transceiver ngoài. FIFO enable/threshold giảm số IRQ nhưng thêm latency. RTS/CTS flow control không giống ACK của giao thức ứng dụng. DMA normal/circular là mode DMA phục vụ UART, không thay frame UART.

LIN dùng break, delimiter, sync 0x55, protected identifier, data và checksum; publisher/subscriber theo schedule. USART hỗ trợ phần framing/break không tự cung cấp LIN stack. Smartcard ISO7816 dùng timing, parity và xử lý lỗi riêng; IrDA dùng modulation theo mode; multiprocessor mute/address wakeup giảm CPU nhận frame không dành cho mình. Receiver timeout và IDLE giúp tách burst, không bảo đảm đó là ranh giới message nếu sender có khoảng dừng bất kỳ.

Application frame cần version/type/length/sequence/payload/integrity và giới hạn parser. Không serialize nguyên struct C vì padding, endian và ABI. RX DMA circular phải xử lý wrap, lỗi framing/parity/overrun, dữ liệu đến trong lúc CPU đọc vị trí DMA. DMA TC khác UART TC.

### SPI và I2S

SPI master/slave; full-duplex hai line; simplex TX/RX; half-duplex bidirectional theo instance. CPOL đặt mức idle; CPHA đặt cạnh sample; word size và bit order xác định chuỗi bit. NSS software/hardware/pulse, TI mode nếu hỗ trợ, CRC và FIFO threshold cần học riêng. SPI không có địa chỉ, ACK hoặc frame ứng dụng chung; CS và command của thiết bị định nghĩa giao dịch. Trong full-duplex, gửi để tạo clock đồng thời nhận; phải xử lý RX khi chỉ quan tâm TX để tránh overrun.

I2S Philips có độ trễ một bit sau đổi WS; left/right-justified và PCM có framing khác. Tách sample width với slot width: mẫu 24 bit có thể đặt trong slot 32 bit. Fs, BCLK và MCLK phải phù hợp codec; PLL audio giảm sai lệch nhưng không tự giải quyết drift giữa hai thiết bị clock độc lập.

### Lab và sản phẩm

1. UART: đo frame 8N1, parity và stop khác; đưa lỗi baud để log FE/PE/ORE. Kiểm parser bằng input bị cắt, nhiễu và length vượt giới hạn.
2. RS485: đo DE tới bit stop cuối; dùng TC thay DMA TC. LIN: decode PID parity và cả checksum classic/enhanced theo frame type.
3. SPI: loopback trước, sau đó sensor; thử bốn CPOL/CPHA với nguồn test đúng, kiểm bit order, word size và CS suốt command+address+data.
4. DMA TX/RX SPI: khởi động theo driver, chờ peripheral end-of-transfer/busy điều kiện hợp lệ trước thả CS; ghi timeout và recovery.
5. I2S: phát tone, kiểm WS/BCLK/slot với logic analyzer; chạy hai buffer và tạo underrun để kiểm policy im lặng/restart.

**Đạt:** capture mỗi mode áp dụng, schema message rõ endian/length và bảng thời điểm hoàn tất bus. [RM0481 USART/SPI/I2S; Bài 23–24 và 27–28]

## Bài 66 I2C SMBus PMBus và I3C theo transaction

I2C open-drain gồm START, address+R/W, ACK/NACK, data, repeated START và STOP. Master/controller phát clock; slave/target phản hồi địa chỉ. 7-bit/10-bit address có framing khác; HAL có thể yêu cầu address đã shift, application nên lưu 7-bit canonical và chuyển ở boundary. Repeated START giữ transaction liên tục để đọc register. Clock stretching và arbitration có điều kiện hỗ trợ, khác timeout phần mềm.

Timing gồm prescaler, SCL high/low, data setup/hold và bộ lọc; điện trở pull-up và capacitance ảnh hưởng rise time. SMBus bổ sung timeout, alert, PEC và các transaction chuẩn; PMBus định nghĩa command/payload quản lý nguồn trên nền SMBus. PEC không phải CRC tùy ý của application. Decode Linear11 phải sign-extend exponent 5 bit và mantissa 11 bit trước V=Y×2^N; không cast unsigned rồi nhân.

I3C dùng address động, CCC và private transfer. DAA gán dynamic address dựa nhận dạng target; static address và dynamic address không hoán đổi tự do. IBI cho target yêu cầu attention; controller phải cấu hình accept/reject, payload và IRQ handling. Hot-join, controller-role request và target/controller mode phải theo khả năng IP. H523 I3C hỗ trợ SDR; không đưa HDR vào lab H523. Legacy I2C trên bus I3C có giới hạn, đặc biệt không giả định clock stretching được chấp nhận.

### Data structure và lab

Lưu device record gồm bus_id, protocol, static_address, dynamic_address, PID/BCR/DCR khi có, max_transfer, timeout và capabilities. Đây là database ứng dụng; không gửi nguyên record lên bus. Transaction gồm address, direction, segments, stop/restart policy, timeout và completion status.

1. I2C đọc register bằng write pointer + repeated START + read. Capture address, ACK từng byte và NACK byte cuối.
2. Thử NACK address, NACK data, timeout và stuck SDA. Bus recovery chỉ clock SCL khi thiết kế cho phép và không có master khác đang giao dịch; giới hạn số lần rồi báo lỗi.
3. SMBus: kiểm PEC bằng vector biết trước; PMBus decode số dương/âm và exponent âm; dùng dữ liệu giả lập trước khi đổi cấu hình nguồn thật.
4. I3C: enumerate bằng DAA với target hỗ trợ, lưu PID và address; reset target rồi kiểm tái enumerate.
5. Gửi CCC hợp lệ, private read/write và IBI nếu board hỗ trợ. Kiểm capture khác biệt address phase và data phase; log IBI overflow/length.

**Đạt:** state machine transaction, trace lỗi và schema dữ liệu không phụ thuộc ABI. [RM0481 I2C/I3C; Bài 25–26]

## Bài 67 FDCAN USB UCPD và các giao tiếp song song

### Mode và frame cần phân biệt

FDCAN có normal, bus monitoring, internal/external loopback và restricted operation theo IP. Classical CAN payload 0–8 byte; CAN FD dùng DLC ánh xạ tới 0–8,12,16,20,24,32,48,64 byte. BRS đổi tốc độ data phase; arbitration vẫn dùng nominal timing. Standard ID 11 bit và extended 29 bit có arbitration khác; filter chọn FIFO/buffer/reject, không phải cơ chế xác thực. Tx FIFO/queue, Rx FIFO overwrite/blocking, watermark, timestamp, error counters và bus-off cần policy rõ. Remote frame không dùng như CAN FD data frame.

USB controller có host/device/OTG theo khả năng chip/board. Control, bulk, interrupt, isochronous là loại transfer; endpoint number+direction định danh endpoint. Enumeration dùng descriptor device/config/interface/endpoint; SETUP packet gồm bmRequestType, bRequest, wValue, wIndex, wLength, các trường nhiều byte theo little-endian. CDC byte stream không bảo đảm giữ ranh giới application frame. Isochronous ưu tiên thời gian, không retry như bulk. H523 USB FS; không dùng ULPI/HS của H7.

UCPD quản lý CC và PHY USB PD, không tự thay policy engine. Source/sink/DRP và orientation thuộc Type-C; PD header/data object/CRC thuộc giao thức PD. PDO/APDO mô tả khả năng, RDO yêu cầu; contract phải hoàn tất trước cấp mức điện áp tương ứng. GoodCRC không có nghĩa đã chấp nhận power request. Không thực hành đổi điện áp trên tải khi chưa có power path, protection và stack phù hợp.

DCMI nhận pixel theo PCLK, HSYNC/VSYNC hoặc embedded sync; continuous/snapshot và crop theo hỗ trợ. PSSI nhận/phát bus song song đồng bộ theo cấu hình; dùng chung tài nguyên với DCMI trên H523 nên không chạy đồng thời. HDMI-CEC open-drain gửi message có start, initiator/destination, opcode/operands và ACK theo protocol; logical address allocation khác pin GPIO addressing.

### Lab

1. FDCAN loopback: kiểm standard/extended, DLC FD, filter accept/reject; sau đó hai node có transceiver và termination. Ngắt node ACK để quan sát error/bus-off rồi phục hồi theo policy.
2. USB CDC: capture enumeration, descriptor và SETUP; thử frame ứng dụng bị chia/gộp qua nhiều packet. Host: thử thiết bị đúng class đã hỗ trợ.
3. UCPD: trước hết đọc CC attach/orientation ở board an toàn; với PD analyzer, decode header/PDO/RDO và state transition, không chỉ báo “cắm có điện”.
4. DCMI: lấy test pattern nhỏ, tính bytes/line và bytes/frame; DMA buffer có length/stride/format/sequence. Thử snapshot trước continuous và log overrun.
5. PSSI/CEC: chỉ khi package/board có chân; ghi giao dịch có timing và ACK, tạo timeout khi đối tác vắng mặt.

**Đạt:** frame/schema cho mỗi interface, đầy đủ mode lỗi và phép thử phục hồi. [RM0481 FDCAN/USB/UCPD/DCMI/PSSI/CEC; Bài 29–31,35]

## Bài 68 Flash bộ nhớ ngoài cache và khối bảo mật

### Mode lưu trữ và bus

Flash read/program/erase có granularity riêng; bank read-while-write không bảo đảm code chạy ở mọi bank trong khi erase. ECC, error flags, write protection, hide protection và OTP khác nhau; OTP chỉ ghi khi đã chốt quy trình provisioning. Journal dùng header version/length/sequence/CRC/commit; commit cuối để chống mất điện. Không coi CRC là xác thực firmware.

OCTOSPI có indirect read/write, automatic polling và memory-mapped theo IP. Command gồm instruction, address, alternate bytes nếu dùng, dummy cycles và data; số line/SDR/DTR có thể khác ở từng phase. Memory-mapped không biến NOR thành SRAM ghi tùy ý; program/erase vẫn theo command. DLYB điều chỉnh sampling theo timing, phải kiểm ở nhiệt độ/điện áp cần thiết.

FMC H523 phục vụ SRAM/PSRAM/FRAM/NOR/NAND theo hỗ trợ, không có SDRAM. Asynchronous timing gồm address setup, data setup, hold/turnaround; NAND cần command/address/data, ECC và bad-block management. SDMMC có command/data state machine, bus width, card identification, clock chuyển giai đoạn, CRC và timeout. CMD/response/data framing khác SPI SD. Cache H523 DCACHE phục vụ đường external memory được quy định; không áp cách xử lý SRAM cache M7 một cách máy móc.

### Mode tính toán và security

CRC polynomial/init/reflection/final convention phải thống nhất với phần mềm; cùng tên CRC32 chưa đủ. HASH algorithm, input datatype/byte ordering, DMA và finalize/padding phải kiểm bằng vector. H523 dùng SHA1/SHA2 theo phần chi tiết DS/RM; không gán SHA3 vì dòng tiêu đề tài liệu có thông tin xung đột. RNG có health/error flags; không dùng dữ liệu sau lỗi entropy. PKA H523 phục vụ ECDSA verification theo phạm vi chip; không suy ra RSA/ECDH/signing từ chương RM dùng chung. H523 không có AES/SAES/OTFDEC tích hợp như một số chip khác.

MPU giới hạn truy cập CPU theo vùng và quyền; TrustZone tách security state; GTZC/SBS và attribution quản lý tài nguyên/bus; flash protections và lifecycle bảo vệ đọc/ghi/debug theo cơ chế riêng. Secure service phải validate address/length, race và DMA ownership. Timer break vẫn cần cho an toàn công suất, dù TrustZone đã cấu hình.

### Lab và tiêu chí

1. Flash journal: cắt nguồn mô phỏng giữa erase/program/commit; boot luôn chọn bản đã commit hợp lệ. Không thử vào vùng firmware đang chạy.
2. OCTOSPI: cùng pattern đọc indirect rồi mapped; thay dummy/sample delay trong miền hỗ trợ và ghi lỗi. Cache invalidate đúng khi dữ liệu ngoài đã đổi.
3. SDMMC: ghi/read file có checksum, tháo card ở thao tác được kiểm soát; timeout và mount lại không phá dữ liệu đã commit.
4. CRC/HASH: thử empty, một byte, dữ liệu không bội word và vector chuẩn; CPU/DMA phải khớp. RNG: log health flags và fail-closed khi lỗi.
5. Security: app thử đọc secure RAM, sửa secure GPIO và cung cấp DMA descriptor vượt phạm vi; mỗi trường hợp phải bị chặn đúng lớp. Kiểm signature hợp lệ/sai và rollback trong secure update.

**Đạt:** từng mode có kết quả đo hoặc được đánh dấu chưa thử, không chỉ screenshot CubeMX. [RM0481 FLASH/OCTOSPI/FMC/SDMMC/ICACHE/DCACHE/CRC/HASH/RNG/PKA/GTZC; Bài 32–34,41–46]


# Phần mở rộng STM32H7 STM32U5 và STM32L5

Phần mở rộng học các khối chưa có trên H523 và các implementation khác đáng kể. Phạm vi đối chiếu gồm H743/H753, H747/H757, H7A3/H7B3, H7R3/R7/S3/S7; U575/U585, U595/U599/U5A5/U5A9, U5F/U5G; L552/L562. Các bảng ghi mã đại diện, không khẳng định tất cả package hoặc SKU trong dòng đều có một khối. Chip không xuất hiện trong bảng phải được xác nhận bằng datasheet riêng trước lab.

# Chủ đề 17 Kiến trúc hiệu năng và truyền dữ liệu STM32H7

## Bài 69 Cortex M7 TCM AXI SRAM cache và MPU

### Nguyên lý và khác H523

Cortex-M7 sử dụng pipeline, cache và đường truy cập instruction/data để tăng hiệu năng. TCM cho truy cập cục bộ với tính xác định cao; AXI SRAM kết nối bus có nhiều master, chịu arbitration. ITCM phục vụ code, DTCM phục vụ data theo memory map. Linker quyết định nơi đặt section; việc khai báo một biến có tên fast_buffer không đưa nó vào DTCM.

Cache lưu bản sao theo line. Write-back có thể giữ dữ liệu CPU mới trong cache chưa ghi RAM; DMA đọc RAM sẽ nhận bản cũ. DMA ghi RAM nhưng CPU còn line cache cũ sẽ đọc bản cũ. Đây là coherency, không được sửa bằng volatile. H743 dùng D-cache line 32 byte; bộ đệm DMA phải xử lý cả alignment và khoảng địa chỉ được làm tròn. H523 DCACHE có phạm vi khác, không dùng cùng quy tắc cho mọi SRAM.

MPU chọn quyền truy cập và thuộc tính memory như normal/device/cacheable, ảnh hưởng ordering và cache. Device register không được ánh xạ như RAM cacheable. Vùng DMA non-cacheable giảm chi phí maintenance nhưng có thể giảm hiệu năng CPU. Write-through giảm một số tình huống TX dirty data, vẫn cần xử lý dữ liệu RX và thuộc tính đúng.

### Quy trình ownership và lab

CPU→DMA TX: CPU ghi buffer, clean vùng cache thích hợp, barrier theo API/kiến trúc, bàn giao DMA; CPU không sửa cho đến completion. DMA→CPU RX: chuẩn bị vùng để dirty line cũ không ghi đè dữ liệu DMA, bắt đầu truyền; khi xong invalidate đúng vùng trước CPU đọc. Buffer và metadata không nên dùng chung cache line nếu vòng đời khác nhau. Invalidate một line chứa biến dirty không liên quan có thể làm mất thay đổi của biến đó.

1. Đọc memory map H743; lập bảng CPU/DMA1/DMA2/BDMA/MDMA truy cập vùng nào. DMA1/2 không dùng DTCM như buffer thông thường; không suy ra từ CPU đọc được.
2. Link một hàm thử vào ITCM, data vào DTCM và buffer DMA vào SRAM hợp lệ. Kiểm map file và địa chỉ runtime.
3. Memory-to-memory DMA với cache bật, cố ý bỏ maintenance; tái hiện mismatch, sau đó sửa và verify toàn buffer.
4. Tạo section DMA non-cacheable bằng linker+MPU theo alignment/size phù hợp. Đo throughput so với cacheable có maintenance.
5. Đo thời gian hàm bằng DWT trong warm/cold cache và có DMA contention. Báo min/max, không chỉ trung bình.

**Đạt:** map file, quyền bus, ownership và bằng chứng lỗi/sửa cache. [H743 DS12110; RM0433; AN4839]

## Bài 70 H7 hai lõi HSEM IPC và cách ly điều khiển

### Cấu trúc và mode hoạt động

H745/H747/H755/H757 có Cortex-M7 và Cortex-M4; đây là hai lõi chạy song song, khác hai task trên một core và khác secure/non-secure M33. Mỗi core có vector table, startup, interrupt và firmware riêng theo thiết kế. Boot có thể cho một core khởi động trước rồi đánh thức core còn lại; reset/domain power có phạm vi cần đọc RM0399.

HSEM cung cấp semaphore phần cứng để phối hợp truy cập và notification. Nó không chuyển payload; payload ở shared RAM. Một core lấy semaphore rồi treo có thể gây deadlock nếu protocol không có timeout/recovery. HSEM không tự tạo cache coherency M7↔M4. Shared buffer cần non-cacheable hoặc maintenance, memory barriers và ownership.

Thiết kế motor core và app core: core điều khiển sở hữu timer/ADC/driver; app chỉ gửi command có giới hạn. Shared RAM không chứa con trỏ trực tiếp vào private data. MPU mỗi core và cơ chế bus/resource ownership phải được đánh giá; dual-core không tự bảo đảm core app lỗi không ảnh hưởng nguồn/clock/bus chung.

### Cấu trúc message

```c
typedef struct {
    uint32_t version;
    uint32_t sequence;
    uint32_t opcode;
    uint32_t payload_bytes;
    uint8_t payload[48];
} ipc_message_t;
```

Đây là schema gợi ý. Chốt endian, layout/alignment, range và memory ordering; nếu dùng giữa ABI khác phải encode field riêng. Ring dùng producer/consumer index, capacity, generation và reset handshake. Notification HSEM chỉ báo có việc; receiver phải kiểm queue, xử lý coalescing và missed notification.

### Lab

1. Build hai image đúng địa chỉ, chớp hai LED ở hai core. Log core ID và firmware version.
2. Shared ring một producer/một consumer; dùng HSEM notification; verify sequence không mất khi notification bị gộp.
3. M7 bật cache; kiểm shared data sau khi chọn policy phù hợp. Ghi latency IPC min/max.
4. Core app gửi command vượt giới hạn; motor core reject và giữ safe state. Cho app treo để control tiếp tục trong budget thiết kế.
5. Reset một core trong khi queue còn message; generation mới làm message cũ invalid. Thử timeout khi semaphore bị giữ quá lâu.

**Đạt:** protocol reboot/recovery, resource ownership và chứng cứ không phụ thuộc app heartbeat để xử lý fault motor. [H747 DS12930; RM0399; ST AN5617 interprocessor communication]

## Bài 71 DMA1 DMA2 BDMA MDMA và DMAMUX trên H7 cổ điển

DMA1/2 của H743 dùng stream, normal/circular, double-buffer, direct/FIFO, memory/peripheral widths và burst theo IP. BDMA là DMA đơn giản hơn cho đường bus/miền tương ứng; MDMA xử lý truyền khối, linked-list, address increment/offset và biến đổi dữ liệu theo hỗ trợ. DMAMUX định tuyến request, có synchronization và request generator; không phải mọi DMA của H7 đều dùng cùng mux hay cùng bus.

Direct mode truyền với ít đệm; FIFO mode đệm để kết hợp burst/width và chọn threshold. Memory burst INCR4/8/16 nếu hỗ trợ có yêu cầu alignment/length/FIFO; peripheral burst phải phù hợp endpoint. Double-buffer mode đổi M0/M1 và current target; CPU chỉ xử lý buffer inactive và không sửa địa chỉ active. Circular đơn buffer không mặc nhiên thành double buffer.

MDMA có trigger theo software/hardware và granularity buffer/block/repeated block/whole transfer tùy cấu hình. Có thể dùng DMA peripheral đưa data vào SRAM rồi MDMA chuyển hoặc sắp xếp khối ở điểm hoàn tất, nhưng phải kiểm route trigger và không sửa vùng DMA đang ghi. Arbitration/priority ảnh hưởng latency; ưu tiên cao không bảo đảm tuyệt đối nếu bus bị master khác giữ.

### Lab

1. DMA1/2: peripheral RX normal, sau đó circular; log half/full và overrun. Tạo waveform marker đo latency IRQ.
2. Double buffer: A/B 256 mẫu; kiểm bit current target, sequence và deadline. Thử app trễ để thống kê mất dữ liệu.
3. Direct/FIFO: copy pattern với width và burst hợp lệ; đo hiệu năng ở threshold/burst khác nhau. Khi lỗi, ghi stream flags trước clear.
4. DMAMUX synchronization: gate số request theo sự kiện; request generator: tạo số request hữu hạn từ nguồn hỗ trợ. Không gán timer trigger của ADC thành request generator nếu routing không có.
5. MDMA: copy nhiều block có stride; thử linked-list; verify sentinel. Đặt buffer ở từng SRAM hợp lệ để đo bus contention.
6. Lặp lại cache ownership Bài 69; không chấp nhận test chỉ pass khi D-cache tắt.

**Đạt:** ma trận DMA×RAM×request, mode table từng engine, measurements throughput/deadline. **Phân biệt:** H7R/S dùng GPDMA/HPDMA, không copy cấu hình stream này. [RM0433 DMA/BDMA/MDMA/DMAMUX]

## Bài 72 FMC SDRAM và băng thông hệ thống

SDRAM lưu bit bằng điện tích cần refresh. Controller gửi activate/read/write/precharge/refresh, truy cập theo bank/row/column. CAS latency, tRCD, tRP, tRAS, tRC, tWR và tXSR là timing bộ nhớ; chuyển nanosecond sang cycle bằng ceil(t×fSDRAM), rồi áp quy tắc thanh ghi có thể mã hóa cycles−1. Không lấy nguyên giá trị timing từ board khác.

Khởi tạo thường gồm ổn định clock/nguồn, clock enable, precharge all, auto-refresh, load mode register và refresh counter theo datasheet memory/RM. Refresh quá thưa mất dữ liệu; quá dày giảm bandwidth. Burst length và CAS phải nhất quán controller/memory. SDRAM không có trên FMC H523 nhưng có trên H743 và các H7 phù hợp.

Memory-mapped SDRAM cần MPU/cache attributes đúng, startup không truy cập trước init, linker không đặt stack cần dùng sớm vào RAM ngoài. Băng thông lý tưởng bus 16 bit tại 100 MHz là 200 MB/s với một transfer/cycle; băng thông hữu ích thấp hơn do refresh, row miss và arbitration. LTDC đọc framebuffer, camera ghi frame, CPU và DMA2D cùng sử dụng sẽ tranh chấp.

### Lab

1. Đọc part SDRAM trên schematic; lập bảng timing với đơn vị ns, cycle, giá trị register. Chọn clock trong giới hạn memory và MCU.
2. Init SDRAM trước test; chạy walking-one/zero, address pattern và pseudorandom trên vùng thử không chứa code/stack.
3. Kiểm alias bằng ghi các địa chỉ tương ứng row/column/bank. Sai geometry có thể pass vùng nhỏ nhưng fail vùng lớn.
4. Thử nhiều clock và load CPU/DMA; tính CRC pattern sau thời gian dài. Không chủ động dùng timing ngoài giới hạn trong sản phẩm.
5. Đặt framebuffer thử, chạy LTDC+DMA2D; log underrun khi tải tăng. Tính bandwidth budget và thêm margin.

**Đạt:** init sequence, timing worksheet, memory test và load test. [RM0433 FMC; datasheet SDRAM của board; H743 DS12110]

# Chủ đề 18 Giao tiếp và multimedia chưa có trên H523

## Bài 73 Ethernet MAC PHY RMII MII và descriptor DMA

### Cấu trúc và frame

MAC đóng/mở frame Ethernet và điều phối DMA; PHY xử lý tín hiệu đường dây. H743 cần PHY/transceiver và magnetics thích hợp; MAC không phải cổng RJ45 hoàn chỉnh. MII/RMII là giao tiếp MAC–PHY; MDIO/MDC quản lý register PHY. Link up chỉ chứng minh lớp liên kết, không chứng minh IP/TCP hoạt động.

| Trường Ethernet | Độ dài | Vai trò |
|---|---:|---|
| Preamble và SFD | 7+1 byte | Đồng bộ và bắt đầu frame trên đường dây |
| Destination MAC | 6 byte | Đích unicast/multicast/broadcast |
| Source MAC | 6 byte | Nguồn frame |
| EtherType hoặc length | 2 byte | Phân loại payload hoặc độ dài theo framing |
| VLAN tag tùy chọn | 4 byte | TCI và EtherType liên quan |
| Payload và padding | Phụ thuộc frame | Packet tầng trên và padding tối thiểu |
| FCS | 4 byte | CRC Ethernet do MAC xử lý theo cấu hình |

Frame buffer DMA thường không chứa preamble/SFD và có thể không giữ FCS tùy cấu hình. Inter-frame gap là khoảng thời gian trên đường dây, không field trong payload. IP/UDP/TCP có header và checksum riêng; Ethernet CRC không thay authentication.

Descriptor chứa địa chỉ buffer, length, ownership và control/status theo layout IP. CPU bàn giao OWN sau khi hoàn tất nội dung và maintenance; DMA trả OWN rồi CPU mới đọc. Descriptor ring và packet pbuf cần lifetime rõ, đặc biệt zero-copy. Checksum offload phải khớp cấu hình lwIP và loại packet; cache lỗi có thể biểu hiện như packet checksum sai ngẫu nhiên.

### Lab

1. Reset/init PHY, đọc ID qua MDIO; kiểm strap address, link speed/duplex và clock RMII theo schematic.
2. Dùng MAC address locally administered hợp lệ cho lab; chạy ARP rồi ping với mạng kiểm thử. Capture Wireshark và chỉ rõ Ethernet/IP/ICMP headers.
3. UDP echo có sequence/length; kiểm loss và latency. TCP echo kiểm data stream bị chia/gộp, không dùng recv length làm message boundary.
4. Đo throughput đồng thời control timer/ADC hoạt động; log RX descriptor starvation, TX queue full và cache errors.
5. Rút/cắm cable, đổi link; stack phục hồi không leak pbuf hoặc giữ descriptor OWN sai. Broadcast storm thử trong mạng cô lập có giới hạn.

**Đạt:** descriptor ownership, packet capture và load/recovery report. [H743 DS12110; RM0433 ETH; ST AN4839]

## Bài 74 USB high speed ULPI và PHY tích hợp

USB FS là 12 Mbit/s; HS là 480 Mbit/s tốc độ signaling, không phải tốc độ application đảm bảo. Một controller có tên OTG_HS có thể chỉ chạy FS khi dùng PHY FS nội; HS trên H743 thường cần PHY ULPI ngoài. Một số U5/H7R/S có PHY HS tích hợp, nên không yêu cầu ULPI ngoài như H743. Chọn theo datasheet và schematic.

ULPI là bus parallel giữa controller và PHY, có clock/data/control như DIR/NXT/STP theo interface. Nó không phải USB packet gửi tới host. USB ở tầng bus gồm token/data/handshake; controller/stack xử lý phần lớn. Application phải hiểu endpoint, max packet size, FIFO allocation, DMA alignment, short packet/ZLP và buffer ownership.

Bulk retry có độ tin cậy theo bus nhưng latency không cố định. Interrupt transfer được host polling theo interval, không phải device ngắt trực tiếp CPU host. Isochronous dùng cho audio/video theo deadline, mất packet phải có concealment. Control transfer dùng setup/data/status. HS enumeration và high-speed negotiation phải đo/capture, không dựa tên driver.

### Lab

1. Chọn board HS đúng PHY; kiểm clock, nguồn và route D+/D−. Dùng USB analyzer hoặc host log xác nhận negotiated speed.
2. CDC/vendor bulk loopback; đo throughput với buffer 64,512 và nhiều packet hợp lệ, phân biệt max packet size theo speed.
3. Gửi message vượt một USB packet; dùng application length parser. Kiểm kết thúc transfer ở biên max-packet và ZLP theo stack/class.
4. Thử USB DMA/cache với ownership đúng. Rút cáp khi đang transfer; release buffer và state không bị kẹt.
5. Nếu học audio isochronous: tính samples/interval, test clock drift, underrun/overrun và policy bù; ghi format descriptor.

**Đạt:** tốc độ được xác nhận, descriptor đúng, teardown sạch. [H743 DS12110 USB; U59 DS13633; H7R DS14360]

## Bài 75 SAI SPDIFRX và stream audio

SAI tách frame, slot và sample linh hoạt hơn I2S đơn giản. Block có master/slave transmitter/receiver, đồng bộ nội/ngoài theo routing. I2S, left/right-justified, DSP/PCM/TDM và AC97 hoặc SPDIF transmission chỉ chọn khi IP hỗ trợ. Frame length, active frame length, polarity/offset, slot size, slot number và slot active mask cần nhất quán với codec.

Ví dụ stereo 48 kHz, 2 slot×32 bit: BCLK=48,000×64=3.072 MHz. Sample 24 bit trong slot32 cần sign extension/packing đúng; DMA word 32 bit không có nghĩa sample có 32 bit hữu ích. TDM 8×32 bit cần BCLK=12.288 MHz ở cùng Fs. MCLK ratio lấy theo codec, không cố định mọi thiết bị 256×Fs.

SPDIFRX nhận chuỗi biphase-mark có preamble/subframe và bits validity/user/channel-status/parity. Một subframe 32 time slots; hai subframe tạo một frame stereo, block gồm 192 frame cho channel-status organization. Preamble là mẫu đặc biệt, không bit data thường. Dữ liệu PCM, validity và status có ý nghĩa khác; sample invalid phải được xử lý chứ không phát như âm thanh hợp lệ.

### Data structure và lab

Audio block metadata cần sample_rate, channels, sample_bits, slot_bits, sample_count, timestamp, sequence và error flags. Sample buffer format như signed PCM little-endian phải ghi rõ; WAV file header là container ứng dụng, không frame SAI.

1. SAI phát tone 1 kHz tại Fs48k; đo FS/BCLK/MCLK. Decode I2S và left-justified để thấy lệch một bit.
2. TDM: bật slot mask và verify codec/receiver nhận đúng kênh. Thử đổi sample width, giữ slot width, không đổi mọi clock một cách vô ý.
3. DMA hai buffer; đo deadline block 128 frame ≈2.667 ms. Khi thiếu data phát silence có kiểm soát, đếm underrun.
4. SPDIFRX từ nguồn phù hợp: đọc PCM, validity, parity/status và lock state. Mất clock nguồn → mute/relock, không giữ sample rác.

**Đạt:** format worksheet và capture frame/slot, buffer lifetime và clock-loss recovery. [RM0433 SAI/SPDIFRX; RM0456 SAI; RM0438 SAI]

## Bài 76 DFSDM MDF ADF và phát hiện hoạt động âm thanh

Sigma-delta modulator tạo bitstream chứa thông tin analog; digital filter tích lũy/decimate thành mẫu có tốc độ thấp hơn. PDM microphone cần clock và filter; bitstream không phải PCM byte. DFSDM trên các H7/L5 có channel input, filter, regular/injected conversion và watchdog theo IP. MDF của U5 cung cấp filtering đa chức năng; ADF ưu tiên audio và sound-activity detection; không dùng nguyên register DFSDM cho MDF.

Sinc filter loại nhiễu nhưng có group delay; oversampling ratio/decimation tăng thường giảm output rate và thay response. Scaling/offset và độ rộng accumulator cần tránh saturation. Fs output liên quan bit clock và tổng decimation theo các stage; không lấy một divider duy nhất nếu có nhiều stage. Filter startup có transient, phải bỏ/đánh dấu mẫu chưa ổn định.

MDF hỗ trợ đường input/filter/trigger theo cấu trúc từng chip; ADF activity detection có threshold, cửa đánh giá và điều kiện wakeup. VAD/SAD chỉ phát hiện hoạt động theo thuật toán phần cứng hỗ trợ, không nhận dạng tiếng nói hoặc từ khóa. PDM frame thường là chuỗi bit clocked, không có start/length/CRC như UART application frame; PCM block cần metadata riêng.

### Lab

1. Cấp clock PDM đúng mic và routing; capture bitstream, kiểm channel theo cạnh clock nếu microphone stereo sharing hỗ trợ.
2. Chọn decimation để output 16 kHz hoặc 48 kHz; tính cả stage, đo Fs từ DMA timestamp.
3. Ghi tone và silence, phân tích noise floor và clipping. Thử filter/ratio khác, đo group delay với tín hiệu test.
4. DFSDM injected/regular hoặc MDF trigger theo instance: kiểm synchronization và không lẫn buffer của hai group.
5. ADF low-power: cho CPU ngủ ở mode hỗ trợ, sound activity đánh thức; sweep threshold và đo false wakeups trong nhiễu nền.

**Đạt:** filter rate/delay worksheet, PCM format và wakeup statistics. [H743 DS12110; L552 DS12737; U575 DS13737; RM0456; ST AN5795]

## Bài 77 OPAMP COMP ADC nâng cao và DAC sample and hold

OPAMP tích hợp gồm input mux, amplifier, feedback route và calibration theo IP. Standalone dùng feedback ngoài; follower gain1; PGA dùng mạng feedback nội với gain hỗ trợ. Common-mode range, output swing, offset, bandwidth, slew rate và tải giới hạn đo được; không được coi opamp lý tưởng vì nằm trong MCU. High-speed/low-power mode nếu có đổi tiêu thụ và settling. H523 không có OPAMP nội; H743/U575/L552 là các ví dụ có khối này.

Comparator so hai input, output logic; chọn reference DAC/VREF/internal/external theo mux. Hysteresis giảm chatter; blanking che một cửa nhiễu theo routing; output có thể đi EXTI/timer break/capture. Comparator propagation delay và filter thêm latency; fault protection cần budget end-to-end. Comparator không thay ADC khi cần giá trị biên độ.

H743 có ADC tối đa16 bit; U575 có ADC14 bit và ADC12 bit low-power; L552 ADC12 bit với oversampling. Bit danh định khác ENOB. Dual ADC/multimode, injected, differential và calibration phải kiểm IP cụ thể; không port HAL_ADCEx từ dòng khác rồi coi mode tương đương. DAC sample-and-hold trên U5/L5 giữ điện áp bằng tụ theo chu kỳ sampling/hold/refresh để giảm năng lượng, có droop/settling và timing riêng.

### Lab theo đường đo

1. OPAMP follower: đưa ramp trong common-mode range; đo input/output, offset và clipping. PGA: chọn hai gain được hỗ trợ, tính headroom ở ADC.
2. Đổi power mode nếu hỗ trợ; đo settling sau enable và đáp ứng step. Calibration đúng điều kiện và chờ thời gian yêu cầu.
3. COMP: sweep qua threshold, đo hysteresis; nối output vào break hợp lệ, đo thời gian fault đến safe output.
4. ADC: cùng nguồn DC, đo noise/code distribution ở resolution và sampling time khác. Chỉ báo ENOB khi có phương pháp đo đúng, không lấy số bit output.
5. DAC sample-and-hold: so continuous với S/H; đo droop theo hold interval, tải và tụ theo datasheet. Tính sai số cho sensor bias/reference dự án.

**Đạt:** analog error budget, route map và latency measurements. [DS12110/DS13737/DS12737; RM0433/RM0456/RM0438 analog]

# Chủ đề 19 Đồ họa điều khiển nhanh và giao tiếp chuyên dụng

## Bài 78 LTDC DMA2D và pipeline framebuffer

LTDC đọc framebuffer và phát pixel theo pixel clock, sync và porch. RGB interface cần HSYNC/VSYNC/DE/pixel data theo panel. Pixel clock = total_horizontal×total_vertical×refresh_rate; total gồm active + sync + back/front porch. DMA2D là engine đồ họa copy/fill/pixel-format conversion/blending theo hỗ trợ, không phải DMA peripheral thông thường.

RGB565 có 2 byte/pixel; ARGB8888 có4 byte/pixel. 800×480 RGB565 cần768,000 byte/frame; double buffer cần1,536,000 byte chưa tính metadata. Scan60Hz đọc active pixel khoảng46.08 MB/s; bus còn overhead và cạnh tranh DMA2D/CPU. Không chọn độ phân giải chỉ vì RAM chứa vừa một frame.

LTDC layer có window, format, pitch, alpha và blending. Pitch có thể lớn hơn width×bytes_per_pixel. Reload immediate có thể thay giữa frame gây tearing; vertical-blank reload hỗ trợ đổi buffer ở mốc thích hợp. DMA2D completion chỉ nói rendering xong, không có nghĩa panel đã scan xong; front buffer chỉ release theo ownership pipeline.

### Lab

1. Đọc timing panel, tạo màu solid rồi color bars; kiểm polarity và porch trước khi dùng GUI.
2. DMA2D fill/copy RGB565, PFC sang ARGB8888 và alpha blend; verify vùng ngoài rectangle không đổi bằng sentinel/checksum.
3. Double buffer: render back, chờ DMA2D complete, schedule LTDC address reload tại blank, sau xác nhận mới đổi ownership.
4. Cố ý immediate swap giữa frame để quan sát tearing; sau đó sửa. Log FIFO underrun khi tăng memory load.
5. Đo frame time, dirty rectangle và CPU load; tính margin cho tác vụ control. MPU/cache maintenance áp dụng cả DMA2D và LTDC.

**Đạt:** timing panel, memory/bandwidth budget và không tearing/underrun ở tải mục tiêu. [RM0433 LTDC/DMA2D; DS13633 graphic features]

## Bài 79 MIPI DSI GFXMMU NeoChrom và JPEG

DSI host truyền pixel/command tới panel qua lane vi sai. Video mode stream theo timing; command mode ghi nội dung theo command và cơ chế refresh panel. LP/HS là trạng thái truyền vật lý, không đồng nghĩa MCU Sleep/Run. DSI short packet có header với data type, payload ngắn và ECC; long packet có word count, payload và CRC. DCS command và generic packet khác nhau; TE hỗ trợ đồng bộ cập nhật nếu panel cung cấp.

GFXMMU ánh xạ framebuffer logic tới vùng vật lý để giảm lưu trữ phần không hiển thị, ví dụ màn tròn. Đây không phải MMU virtual memory để chạy process Linux. Mapping/LUT sai gây pixel sai hoặc truy cập vùng sai. NeoChrom GPU hỗ trợ rotate/scale/texture operations trên chip phù hợp; NeoChromVG thêm vector graphics trên U5F/U5G. DMA2D, GPU và GFXMMU là ba vai trò khác nhau.

JPEG codec encode/decode stream; pipeline cần xử lý MCU block, màu YCbCr, subsampling và chuyển format tới RGB. JPEG file là chuỗi marker/segments và entropy-coded data, khác ảnh raw liên tục. Kiểm SOI/EOI, dimensions, subsampling và unsupported variants trước cấp buffer; hardware JPEG không bảo đảm giải được mọi biến thể JPEG.

### Lab

1. DSI panel hỗ trợ: gửi init DCS từ tài liệu panel, đọc ID/status nếu hỗ trợ; bật test pattern và xác nhận lane/timing.
2. So command/video mode nếu panel hỗ trợ cả hai; capture hoặc host log packet type, word count và TE handling.
3. GFXMMU: tạo mapping màn tròn/rectangle thử; tính bytes tiết kiệm, kiểm edge pixels và out-of-range access.
4. GPU: rotate texture rồi scale; so thời gian CPU reference, image checksum hoặc pixel tolerance. Quản lý command list/texture/destination ownership.
5. JPEG: decode ảnh nhỏ known-good tới RGB; đo codec và conversion time riêng. Thử file truncate, kích thước quá lớn và format không hỗ trợ; reject an toàn.

**Đạt:** packet/data structure, memory map, validation và fault report. **Áp dụng:** H747 có DSI; H743 không được gán DSI vì cùng họ H7. U599/H7R7 có GPU theo feature table; U5F/U5G có VG/JPEG theo mã; không gán toàn bộ cho U575 hoặc H7R3. [DS12930/DS13633/DS14395/DS14360]

## Bài 80 HRTIM chuyển đổi công suất và định thời độ phân giải cao

HRTIM có master và các timing unit, nhiều compare, set/reset output, dead-time, external event, fault và ADC trigger theo IP. Độ phân giải cao dùng cơ chế nội bộ bổ sung, không chỉ tăng PSC. Calibration/DLL ready nếu IP yêu cầu phải hoàn tất trước dùng high-resolution path. HRTIM có trên H743 theo datasheet; không mặc định mọi H7/U5/L5 đều có.

Output được set/reset bởi các nguồn sự kiện được chọn, linh hoạt hơn PWM một CCR. Continuous/one-shot, retriggerable, push-pull, burst operation và chopper chỉ chọn khi hỗ trợ. Burst mode của HRTIM tạo cửa nhóm chu kỳ để giảm công suất; khác DMA burst. Fault/blanking/filter có latency và phạm vi khác; dead-time không thay bảo vệ quá dòng.

Ứng dụng converter buck/boost, phase-shift bridge hoặc nhiều pha cần compare/ADC trigger định thời chặt. Minimum on/off time, driver delay, dead-time và sensing window đặt giới hạn thực tế; độ phân giải counter cao không bảo đảm control loop ổn định.

### Lab

1. Bắt đầu logic output, chưa nối power stage. Chọn clock/calibration, tạo hai PWM lệch pha và đo phase/dead-time.
2. Dùng set/reset event tạo xung có hai mốc compare; update preload để thay tham số đồng bộ.
3. Trigger ADC tại cửa sample hợp lệ; đo tổng delay từ compare đến acquisition/EOC.
4. Burst gating theo nhiều chu kỳ; đo tần số envelope và switching bên trong. Phân biệt với DMA transfer burst bằng trace.
5. Fault qua pin/COMP route hợp lệ; đo tắt output, kiểm re-arm và filter. Cố ý CPU treo ở lab logic để chứng minh đường fault độc lập.

**Đạt:** timing/event matrix, calibration handling và minimum pulse/fault measurements. [RM0433 HRTIM; STM32H7 online training HRTIM]

## Bài 81 SWPMI MDIO slave và time triggered CAN

SWPMI là single-wire protocol master interface chuyên dụng, không phải UART một dây hay Dallas 1-Wire. Cần đối tác và electrical layer phù hợp. Active/deactivated, buffering/DMA và frame/control handling theo IP; application không gửi struct C tùy ý rồi gọi là SWP frame. PHY và framing/control giao thức cần được học từ mục SWPMI trong RM cùng tiêu chuẩn đối tác.

MDIO master của Ethernet quản lý PHY; MDIOS là MDIO slave để hệ thống ngoài đọc/ghi register MCU qua giao diện quản lý. Clause22 frame gồm preamble, start, opcode, PHY address, register address, turnaround và16-bit data. Preamble có thể bị suppress theo khả năng; turnaround khác read/write. Clause45 có cơ chế address/data khác, không mặc định MDIOS hỗ trợ chỉ vì MDIO tiêu chuẩn có.

Time-triggered CAN trên controller hỗ trợ bổ sung reference message, time base và scheduling lên CAN; không thay arbitration/error detection. Message schema phải có cycle/time identity và freshness theo protocol dự án. CAN FD và TT-CAN là hai khả năng khác nhau; H523 FDCAN không tự được gán TT-CAN từ H743.

### Lab có điều kiện phần cứng

1. MDIOS: tạo register bank ID/status/command; master ngoài đọc/ghi, capture từng field Clause22. Kiểm invalid register và write permissions.
2. DMA cho register/frame handling nếu IP hỗ trợ; log read/write events với sequence. Không để master ngoài ghi trực tiếp cấu hình motor.
3. SWPMI: với đối tác tương thích, activation, transfer và deactivation; capture/log frame status, payload và lỗi. Nếu không có đối tác, phân tích state/register và đánh dấu chưa kiểm physical frame.
4. TT-CAN: trên node hỗ trợ, lập cycle schedule, mất reference message và late-message policy; đo deadline thay vì chỉ thấy frame có trên bus.

**Đạt:** schema/frame có căn cứ, capability rõ và timeout/freshness. **Giới hạn thực hành:** ba interface này cần thiết bị/stack chuyên dụng; không tuyên bố lab loopback phần mềm thay xác minh vật lý. [RM0433 SWPMI/MDIOS/FDCAN; H743 DS12110]

# Chủ đề 20 Ultra low power tính toán và security U5 L5

## Bài 82 SMPS MSI retention LPBAM và batch acquisition

SMPS chuyển đổi bằng switching để giảm tổn hao trong điều kiện phù hợp; LDO đơn giản hơn nhưng tổn hao xấp xỉ (Vin−Vout)×I. Cuộn cảm/tụ/layout/nguồn mode theo schematic MCU; không bật SMPS trên board không lắp mạch. Ripple/EMI có thể ảnh hưởng analog. Mode on-the-fly chỉ dùng khi chip hỗ trợ và làm đúng ready/voltage sequence.

MSI chọn nhiều dải tần để phù hợp năng lượng; U5 có MSIS/MSIK theo implementation, tách vai trò system/kernel. L5 MSI có range và autotrim theo LSE. Retention SRAM chọn bank/page phù hợp từng dòng, ECC/parity đổi usable capacity hoặc điều kiện init. Stop2, Stop3, Standby và Shutdown có nguồn wake/retention khác; bảng datasheet từng chip là chuẩn.

LPBAM U5 cho ngoại vi và DMA thực hiện chuỗi nền khi CPU ngủ trong mode hỗ trợ. Cần autonomous clock, trigger, buffer/descriptor ở RAM giữ được và engine còn hoạt động. LPDMA phục vụ miền low-power; GPDMA có khả năng lớn hơn nhưng không được giả định tương đương ở mọi Stop mode. Stop3 không mặc định tiếp tục chuỗi LPBAM như Stop2.

L5 có BAM batch acquisition: gom công việc/mẫu để giảm số lần CPU wake theo các đường phần cứng hỗ trợ. Đây không phải toàn bộ LPBAM descriptor framework của U5. Race-to-sleep chạy nhanh rồi ngủ có thể tiết kiệm hơn chạy chậm lâu; phải tính năng lượng bao gồm oscillator startup, memory retention và sensor.

### Lab

1. Tạo power profiles LDO/SMPS nếu board cho phép; đo năng lượng một workload, không chỉ current một thời điểm.
2. U5: LPTIM trigger ADC4 low-power → LPDMA lưu N mẫu ở vùng RAM thích hợp, CPU chỉ wake khi batch xong. Kiểm clock/retention theo RM0456.
3. Chạy trong Stop2, log sample count/timestamps và đo wake frequency. So với interrupt mỗi mẫu.
4. L5: dựng batch acquisition bằng ngoại vi/DMA/trigger thực sự hỗ trợ; không dùng API LPBAM U5. Đo cùng workload để so năng lượng.
5. Tắt retention vùng thử và xác nhận dữ liệu invalid sau wake; dùng magic/version/checksum cho retained context.

**Đạt:** autonomy map, energy budget, bằng chứng sample không mất khi CPU ngủ. [DS13737/DS12737; RM0456/RM0438; AN5645]

## Bài 83 CORDIC FMAC và tính toán fixed point

CORDIC dùng chuỗi phép quay/shift-add để tính các hàm được IP hỗ trợ như sine/cosine, phase/modulus hoặc các hàm khác. Chọn function, precision, scaling, input/output format và số word. Góc thường dùng chuẩn hóa fixed-point theo quy ước IP, không đưa radian float trực tiếp vào register. Precision tăng số vòng/latency; accuracy chịu lượng tử hóa và miền đầu vào.

FMAC hỗ trợ phép lọc FIR/IIR theo cấu hình phần cứng, sử dụng coefficient/input/output memory và flow control. FIR y[n]=Σb[k]x[n−k]; IIR thêm feedback coefficients, phải thống nhất dấu theo API/IP. Fixed-point có saturation/overflow, headroom và scaling; IIR còn cần ổn định cực trong miền rời rạc. DMA giảm CPU chuyển word nhưng không loại deadline và ownership.

U575 có CORDIC/FMAC theo datasheet; H523 không có hai khối này. H7R/S có CORDIC theo feature table; không suy ra có FMAC nếu không ghi. Đo total cost gồm setup, data movement và completion, không chỉ vài cycle engine.

### Lab

1. Tạo reference trên host cho sine/cosine; chuyển góc sang fixed-point đúng miền. Test0,±π/2, gần±π và random points; báo max absolute error.
2. Sweep precision/scaling và đo latency. So CPU library với engine cho một mẫu và block lớn.
3. FMAC FIR low-pass: thiết kế coefficients, normalize, quantize; impulse response phải khớp reference trong tolerance đã tính.
4. IIR: step/impulse/saturation tests; dùng coefficient stable và kiểm output bounds. Test input lớn để phát hiện overflow.
5. DMA pipeline ADC→filter→buffer theo trigger khả thi; tổng latency phải nằm trong control budget. Khi input/output buffer starvation, giữ state hoặc reset theo policy rõ.

**Đạt:** numerical error, speed và overflow report; không đổi tên hàm float thành hardware acceleration rồi bỏ validation. [DS13737; RM0456 CORDIC/FMAC; DS14360 CORDIC]

## Bài 84 TSC cảm ứng điện dung và thiết kế điện cực

TSC đo điện dung bằng charge transfer: điện tích từ điện cực sensor chuyển tới tụ sampling đến ngưỡng, counter biểu diễn số chu kỳ. Điện dung đổi khi ngón tay chạm; giá trị còn chịu PCB, cover, ground, độ ẩm, EMI và cáp. TSC có trên L5 và một số U5 như U575, không có trên H523. Không gán TSC cho toàn bộ U5 vì các mã đồ họa khác có feature table khác.

Channels được tổ chức theo group với sampling IO và channel IO; acquisition đồng thời/luân phiên tùy nhóm hợp lệ. Charge/discharge timing, pulse prescaler, spread-spectrum nếu có và maximum count đặt tradeoff noise/speed. Threshold detection cần baseline tracking, hysteresis, debounce và stuck-touch handling ở phần mềm. Slider/wheel nội suy vị trí từ nhiều điện cực, không chỉ đọc pin digital.

### Lab

1. Dùng board/electrode và tụ sampling theo hướng dẫn TSC; ghi schematic, area điện cực và cover thickness.
2. Thu raw counts khi không chạm/chạm; tính mean/noise và khoảng phân biệt. Reject acquisition vượt max count.
3. Thêm baseline tracking chậm khi không touch; khi touch không kéo baseline theo ngón tay quá nhanh. Hysteresis tách ngưỡng press/release.
4. Thử tay ướt, thay nguồn USB/pin, motor switching gần board; đo false touch và missed touch.
5. Slider ba điện cực: normalize từng channel, tính vị trí và giữ valid flag. Thử đa chạm hoặc saturation để không trả vị trí giả.

**Đạt:** raw data, thresholds có căn cứ và environmental test. [DS12737/DS13737; RM0438/RM0456 TSC; ST touch sensing design guidance]

## Bài 85 AES SAES PKA OTFDEC và vòng đời khóa

AES là mã khối; mode ECB/CBC/CTR/GCM/CCM hoặc mode khác chỉ sử dụng nếu hardware/software hỗ trợ. ECB lộ cấu trúc; CBC cần IV/padding/authentication riêng; CTR cần nonce/counter không lặp; GCM/CCM cung cấp authenticated encryption khi dùng đúng nonce/AAD/tag. DMA accelerates data movement, không quản lý nonce hoặc policy khóa.

SAES tăng bảo vệ xử lý/nguồn khóa trên chip phù hợp; khác AES engine thường và không bảo đảm mọi secret application an toàn nếu software leak. PKA hỗ trợ các phép toán public-key theo từng chip: kiểm operation list cho sign/verify/ECDH/RSA, miền số và operand format. H523 verification-only không đại diện cho PKA U585/L562.

OTFDEC giải mã đọc từ bộ nhớ ngoài qua vùng được cấu hình, phục vụ XiP/data confidentiality theo mode hỗ trợ. Không mặc định cung cấp integrity, authentication hoặc chống replay. Boot phải verify image và phiên bản bằng policy riêng. L562 có crypto/OTFDEC mà L552 không được gán; U585 có crypto bổ sung so U575. H7S có tăng cường security so H7R; Cortex-M7 không có TrustZone-M như M33.

### Data structure an toàn

Encrypted record nên có format_version, algorithm_id, key_id, nonce, ciphertext_length, ciphertext, tag và AAD binding device/context/sequence khi cần. Key_id là handle, không phải key bytes trong message. Serialize endian/length rõ; authenticate header cần bảo vệ trước khi tin dữ liệu. Không reuse nonce sau reboot: lưu counter chống mất điện hoặc chọn cơ chế uniqueness đã được thiết kế.

### Lab

1. Known-answer AES theo mode hardware hỗ trợ; test length, endian, alignment và CPU/DMA. Với AEAD, sửa từng field AAD/nonce/ciphertext/tag và yêu cầu reject.
2. Key service secure trên M33 hoặc vùng bảo vệ theo kiến trúc M7; app chỉ gọi handle. Test invalid handle và buffer overlap/race.
3. PKA: verify vector hợp lệ/sai, key/curve sai và operand length sai; đo total latency.
4. OTFDEC: provision khóa thử vào quy trình được hỗ trợ, đọc image encrypted ở đúng vùng; ngoài vùng không được coi là plaintext. Boot vẫn kiểm signature và rollback.
5. Tamper/debug lifecycle: làm trên board lab riêng với khóa thử; các bước không đảo ngược chỉ thực hiện khi đã chốt procedure và recovery khả thi.

**Đạt:** KAT, negative tests, nonce persistence và security threat model. [DS12736 L562; U575/585 product; AN5371; RM0456/RM0438/RM0477]

## Bài 86 H7R S bootflash XSPI HPDMA và bảo mật M7

H7R/S dùng bootflash nhỏ để khởi tạo và chạy ứng dụng từ bộ nhớ ngoài hoặc load vào RAM theo thiết kế. External-memory init trở thành một phần chuỗi boot tin cậy: pin, clock, command, dummy cycles, delay calibration và mapped region phải sẵn sàng trước nhảy code. Startup/linker không được giả định toàn image nằm trong flash nội2MB như H743.

XSPI có octal/hexa theo instance và package; serial NOR/PSRAM/HyperBus có frame khác. HyperBus transaction có command/address và latency/data phases theo memory. SDR/DTR thay cạnh truyền; DQS/delay calibration và board signal integrity quyết định margin. XiP khi erase/program memory chứa code có thể khiến instruction fetch stall/fault; critical code phải đặt nơi truy cập được trong thao tác update.

GPDMA/HPDMA ở đây có FIFO/linked-list và bus width/khả năng riêng; không dùng bảng DMA1/2/MDMA H743. ECC/TCM remap thay usable RAM và placement; kiểm memory map sau configuration. Graphic timer và DCMI pixel conversion/crop trên IP mới cần xác nhận mode từ RM0477 thay vì dùng register H523.

Security H7R/S dùng lifecycle, debug authentication, HDP/RSS và crypto theo mã; không có TrustZone-M chỉ vì tài liệu nhắc secure. H7R PKA có verification limitation theo datasheet; H7S bổ sung crypto cần xác nhận đúng operation. Policy boot/update vẫn phải xử lý signature, version, power loss và recovery image.

### Lab

1. Tạo boot stage trong flash nội, init XSPI và verify header/hash image trước chuyển execution. Log boot phase và fallback reason.
2. XiP và copy-to-RAM cùng workload; đo cold/warm cache và deadline. Critical interrupt ở TCM/RAM có placement rõ.
3. Test external-memory pattern ở clock hợp lệ với calibrated delay; đo margin ở điều kiện dự án, không chỉ nhiệt độ phòng.
4. HPDMA/GPDMA linked-list theo RM0477; verify bus access, descriptor alignment và ownership.
5. Mô phỏng image thiếu/truncate/sai signature và update mất điện; boot vào recovery hợp lệ. Không erase bootflash tin cậy trong thử nghiệm thông thường.

**Đạt:** boot memory map, failure recovery và measured memory margin. [DS14359/DS14360; RM0477; STM32H7RS online training]

## Bài 87 Port firmware giữa H523 H7 U5 và L5

Porting cần giữ yêu cầu chức năng rồi thay implementation theo capability, không thay tên HAL prefix là xong. Peripheral version khác có register, FIFO, flags, trigger và startup sequence khác. GPDMA linked-list, stream DMA DBM và DMA channel cổ điển cần backend riêng. L5 DMA/DMAMUX không có cùng FIFO/burst/2D của H523; không giả lập tên mode rồi coi hardware hỗ trợ.

Tạo capability record từ datasheet gồm timer modes/channels, DMA engines/requests, RAM reachability, cache scope, ADC trigger/group, USB PHY, analog blocks, power retention và crypto operations. Application gọi interface theo semantics: start_stream, acquire_block, release_block, send_message, get_timestamp. Backend phải trả unsupported khi mode không có; không âm thầm đổi thành software delay.

L5 parity SRAM không đồng nghĩa ECC SRAM của H523/U5. U5 có thể đổi usable SRAM khi ECC bật. H7A3/B3 không dùng nguyên ba domain D1/D2/D3 của H743; hai nhánh có topology khác. USART/FDCAN/USB tương tự chức năng vẫn cần kiểm clock/kernel/reset và SRAM message/descriptor riêng.

### Lab và checklist port

1. Chọn một lab ADC→DMA→UART từ H523. Giữ sample rate, frame schema và loss policy; dựng backend H743/U575/L552 theo board có sẵn.
2. Lập pin/clock/security/memory map cho từng target. Build bằng Cube package/version được pin và không reuse startup/linker sai core.
3. Test functional parity bằng cùng pattern/frame vectors; chạy timeout/loss/reconnect.
4. Đo latency, current và memory pressure theo cùng workload. Explain khác biệt bằng clock/bus/cache/autonomy, không chỉ CPU MHz.
5. Liệt kê unsupported modes; ví dụ H523 không SDRAM/OPAMP, H743 không TrustZone-M, L552 không được gán AES/OTFDEC của L562.

**Đạt:** port report, capability table và CI build mỗi target. **Giới hạn:** chưa chọn board cụ thể thì lab là spec thực hành; không gọi đó là firmware nạp chạy được. [AN5371; datasheet/RM từng target]

## Bài 88 Dự án senior đa ngoại vi và hồ sơ nghiệm thu

### Dự án A Điều khiển motor và app hai lõi

H747: core control sở hữu TIM/ADC/fault; app core phục vụ Ethernet/USB/HMI. Dữ liệu IPC dùng schema Bài70, commands có range/freshness, timeout và generation. Tính control budget, bus contention và cửa đo dòng. Nghiệm thu gồm app crash, IPC flood, invalid command, cache stress, break hardware và core reboot. Không coi dual-core là chứng minh cách ly hoàn chỉnh trước các thử nghiệm này.

### Dự án B Cảm biến pin U5

U575/U585: LPTIM→ADC4/ADF→LPDMA/buffer ở Stop2; CPU wake theo batch, xử lý bằng CORDIC/FMAC khi phù hợp, gửi communication. Secure service giữ khóa trên chip có crypto; update có rollback protection. Nghiệm thu bằng joule mỗi batch, sample loss, false activity wakeup, nonce uniqueness và wakeup recovery. Battery life tính cả sensor, radio và regulator của sản phẩm.

### Dự án C HMI có đồ họa

U599/U5F9 hoặc H7R7 trên board phù hợp: framebuffer pipeline, DMA2D/GPU/GFXMMU, panel LTDC/DSI theo chip. Image JPEG phải validate dimensions/format; load dưới stress không gây underrun. Nghiệm thu gồm frame time percentile, memory bandwidth, cache/ownership, tearing, malformed assets và mất nguồn update.

### Dự án D Logger bảo mật L5

L552/L562: ADC/DFSDM batch acquisition, RTC timestamp, SDMMC hoặc OCTOSPI, UART/CAN message; crypto hardware chỉ dùng L562 khi có. Record gồm sequence/timestamp/length/payload/integrity và commit transaction. Nghiệm thu cắt nguồn, card absent, retention invalid, timeout, watchdog và secure/non-secure access violations.

### Hồ sơ phải giao

Schematic/pin map; exact part/revision; datasheet/RM/errata revisions; clock/power tree; memory map và linker; DMA ownership; protocol schemas và golden captures; timing/energy budgets; test logs; fault/recovery state machine; threat model; build version và release procedure. Mỗi phép đo có thiết bị, cấu hình, nhiệt độ/nguồn và điều kiện tải. Phần chưa đo phải ghi chưa đo, không thay bằng ước lượng.

**Đạt:** có thể giải thích vì sao chọn mode, chứng minh điều kiện vận hành và phục hồi, và để người khác tái tạo test. Senior không được xác định bằng việc bật nhiều ngoại vi; năng lực nằm ở lựa chọn, kiểm chứng và kiểm soát tương tác của chúng.


# Phụ lục I Ma trận kiểm tra các mode và nguồn mở rộng

## I1 Cách xác định một ngoại vi đã học đủ

Không đánh dấu “đã thành thạo” chỉ vì mode có tên trong bảng. Với mỗi mode áp dụng, người học cần giải thích điều kiện vào/ra, đường dữ liệu, timing, flags và IRQ/DMA; thực hiện cấu hình tối thiểu; đo hoạt động bình thường; tạo một lỗi có kiểm soát; chứng minh recovery. Mode không có trên chip ghi N/A; có nhưng chưa đo ghi chưa thực hành. Đây là checklist năng lực và khả năng phần cứng, không phải tuyên bố mọi lab đã chạy trên board.

| Khối | Mode và cơ chế cần kiểm | Bài gốc và bổ sung |
|---|---|---|
| PWR reset | Run/Sleep/Stop/Standby, VOS, retention, BOR/PVD, reset flags, VBAT | 07,36,53 |
| RCC CRS | RC/crystal/bypass, mux/divider, PLL/fractional, CSS, CRS, MCO, clock switching | 08,54 |
| GPIO | Input/output/AF/analog, push-pull/open-drain, pull, speed, lock, atomic set/reset | 09,55 |
| EXTI NVIC | Rising/falling/both, interrupt/event, routing, pending, priority/preemption, secure attribution | 10,55 |
| Core CPU | Thread/Handler, privileged/unprivileged, MSP/PSP, exceptions, SVC, faults, FPU context | 04,39–42 |
| TIM counter | Up/down/center theo instance, PSC/ARR, update, preload, repetition, clock selection | 11,56 |
| TIM output | Frozen/set/reset/toggle/forced, PWM1/2, polarity, complementary, one-pulse, mở rộng theo instance | 12,57 |
| TIM input | Direct/indirect/TRC, capture edges/filter/prescaler, PWM input, encoder, external clock | 13,58 |
| TIM control | Master/slave/reset/gated/trigger, TRGO/TRGO2, DMA burst, dead-time/break/lock | 14,59 |
| LPTIM | Timer/counter/PWM/one-shot/encoder theo instance, asynchronous clock/register sync, wakeup | 15,60 |
| RTC TAMP | Calendar/subsecond, alarms/masks, wakeup, timestamp, calibration, tamper/erase | 15,60 |
| IWDG WWDG | Timeout/window, refresh, early warning nếu có, low-power/debug behavior | 16,60 |
| GPDMA | Normal/circular chain, width/inc, burst/FIFO, request/trigger, 2D, link-step, suspend/abort, security | 17,61–62 |
| ICACHE DCACHE | Enable/disable, hit/miss, invalidate/clean theo IP, mapped scope, memory change | 18,68–69 |
| RAMCFG SRAM | ECC/parity theo vùng, init, error detection, retention, security/privilege | 06,43,68,87 |
| ADC | Single/continuous/scan/discontinuous, regular/injected, external trigger, watchdog, oversampling, dual mode | 19–20,63,77 |
| DAC | Static/trigger/DMA, data format, buffer/calibration/routing, dual update, noise/triangle/sawtooth, double-data DMA, S/H, underrun | 21,64,77 |
| VREF DTS | VREFINT/VREFBUF/VBAT/temperature, enable/stabilization/calibration/threshold | 20–21,64 |
| USART UART | Async/sync/half-duplex, FIFO, flow control, RS485, LIN, smartcard, IrDA, mute, IDLE/timeout | 23–24,65 |
| SPI I2S | Master/slave, duplex/simplex/bidirectional, CPOL/CPHA, word/bit order, NSS/TI/CRC, framing audio | 27–28,65 |
| I2C SMBus PMBus | Controller/target, 7/10-bit, restart, stretching/arbitration, filter/timing, PEC/alert/commands | 25,66 |
| I3C | Controller/target theo IP, SDR, DAA, CCC/private, IBI/hot-join/role request theo hỗ trợ | 26,66 |
| FDCAN | Normal/monitor/loopback/restricted, classical/FD/BRS, filters/FIFO/queue, errors/bus-off | 29,67 |
| USB | Device/host/OTG theo IP, control/bulk/interrupt/isochronous, FIFO/DMA, suspend/resume, PHY/speed | 30,67,74 |
| UCPD | Source/sink/DRP, attach/orientation, PD RX/TX, contract/reset, fault policy | 31,67 |
| FLASH OTP | Read/program/erase/banks, ECC/errors, WRP/HDP/RDP, journal, provisioning | 32,46,68 |
| OCTOSPI DLYB | Indirect/polling/mapped, phase widths/SDR/DTR, HyperBus, sample delay, timeout | 33,68,86 |
| FMC | Async/sync theo memory, NOR/PSRAM/NAND, LCD8080/6800; SDRAM riêng H7 | 33,68,72 |
| SDMMC | Identify/transfer, bus width/speed, command/response/data, DMA, SDIO, errors | 34,68 |
| DCMI PSSI | Snapshot/continuous, sync/crop, parallel input/output theo IP, DMA/overrun, shared resources | 35,67 |
| CEC | Address allocation, initiator/follower, arbitration/ACK/error/retry | 35,67 |
| CRC HASH | Polynomial/init/reflection, algorithm/input format, DMA, finalize, known-answer | 44,68 |
| RNG PKA | Health/errors, operand format, operation capability, verification/negative tests | 44,68,85 |
| SAU GTZC SBS MPU | Attribution, region/permission, secure/non-secure, privilege, illegal access/DMA, locking | 41–43,68,85 |
| Boot debug trace | Boot source/vector, ROM interfaces, HDP/RSS, debug authentication, ETM/SWV/fault tracing | 05,40,45–46,86 |

## I2 Các điểm mode cần tra chính xác theo instance

TIM1/TIM8 và TIM2–TIM5 hỗ trợ counter hai chiều/center theo RM; TIM6/TIM7 basic không có capture/PWM channel. TIM12 và TIM15 là các timer ít channel với khả năng riêng, không dùng ma trận TIM1 cho chúng. PWM advanced cần MOE/BDTR nếu instance có. Bảng routing ITR/ADC/DMA request/trigger cần lập lại cho từng chip, kể cả cùng tên TIM1.

| Timer H523 | Counter | Channel và chức năng trọng tâm |
|---|---|---|
| TIM1 TIM8 | 16-bit up/down/center | Advanced PWM, capture/compare, complementary, dead-time, break, repetition |
| TIM2 TIM5 | 32-bit up/down/center | General-purpose capture/compare/PWM/encoder và synchronization |
| TIM3 TIM4 | 16-bit up/down/center | General-purpose capture/compare/PWM/encoder và synchronization |
| TIM6 TIM7 | 16-bit up | Basic timebase/TRGO; không có capture/PWM pin |
| TIM12 | 16-bit up | Hai channel capture/compare/PWM edge-aligned/one-pulse; không center-aligned |
| TIM15 | 16-bit up | Hai channel, một complementary output, dead-time/break/repetition theo RM; không center-aligned |
| LPTIM1 LPTIM2 | 16-bit low-power counter | Chức năng/routing riêng; không sử dụng CMS của TIM1 |

TIM13/14/16/17 xuất hiện trong RM chung nhưng không được gán cho H523. Mode encoder và combined/asymmetric phải kiểm chapter của đúng timer; không bật chỉ vì macro có trong HAL chung.

DMA timer burst dùng DCR/DMAR nếu timer hỗ trợ để cập nhật nhiều register qua một chuỗi request. Đây là cơ chế timer, khác source/destination burst của DMA và burst gating của HRTIM. Lab bổ sung: tạo mảng ARR/CCR hoặc các register hợp lệ theo map, dùng timer DMA burst để thay waveform nhiều bước; verify update/preload và không ghi register ngoài danh sách. Chỉ cập nhật khi DMA đã sở hữu bảng dữ liệu; không dùng cấu trúc C có padding làm register sequence.

ADC low-power auto-wait/auto-delay giữ nhịp theo việc tiêu thụ data trên mode hỗ trợ; giảm overrun nhưng làm sample interval không còn cố định nếu CPU đọc chậm. Nếu cần lấy mẫu đều theo timer, kiểm tương thích và chọn DMA/overrun policy thay vì bật auto-wait để che mất dữ liệu. Differential input, offset/saturation, injected queue và oversampling restart/resume cần thực hành với ADC hỗ trợ. Các bài analog phải luôn ghi single-ended/differential và calibration tương ứng.

H523 ADC1/2: SAR12bit, single-ended/differential, injected, dual mode, offset calibration, oversampling tối đa256 theo DS14540 bảng5; không có linearity calibration/gain compensation được liệt kê ở bảng này. Register result16bit không biến conversion thành ADC16bit. FIFO kết quả ba stage theo DS cần được xét khi phân tích overrun/data handling, không nhầm với GPDMA FIFO.

I3C controller-role/hot-join/IBI acceptance và HDR không suy từ tên I3C; H523 chỉ học SDR. USB host/device phụ thuộc nguồn VBUS/ID/Type-C hardware ngoài, không chỉ bật middleware. UCPD “dead battery”/CC termination có electrical startup riêng theo board; không ép mức GPIO lên CC.

## I3 Bảng tính năng bổ sung theo mã đại diện

| Nhóm tính năng | Mã đại diện đã đối chiếu | Điều kiện cần nhớ |
|---|---|---|
| M7 TCM và SRAM cache | H743, H747, H7A3, H7R | Cache size, bus và memory map khác nhau |
| Hai lõi M7/M4 | H745/H747/H755/H757 | H743/H7A3/H7R không mặc định có M4 |
| Ethernet | H743, H747, H7R/S | H7A3/B3 không gán Ethernet MAC |
| SDRAM | H743 và H7 có FMC SDRAM | H523/U575/L552 không gán SDRAM từ FMC chung |
| USB HS | H743/H747 với PHY HS phù hợp; U59/H7R/S có PHY HS tích hợp theo DS | Tên controller không xác nhận speed thực tế |
| SAI | H743, U575, L552 | Số block, protocol và synchronisation khác |
| DFSDM | H743, L552 | Không thay bằng MDF register |
| MDF ADF | U575; ADF trên H7R/S | Filter count, SAD/VAD và autonomous capability theo chip |
| OPAMP COMP | H743, U575, L552 | H523 không có nội |
| ADC14/16 bit | U575 ADC14; H743 ADC16 | Không đồng nghĩa ENOB14/16 |
| DAC sample and hold | H523, U575, L552 | Timing/tụ/load và low-power routing cần kiểm riêng |
| LTDC DMA2D JPEG | H743; U599/U5F/U5G và H7R7 có tổ hợp đồ họa theo bảng part | Không mọi chip có cả ba |
| MIPI DSI | H747; U599/U5F9 theo DS | H743 không có DSI |
| GPU GFXMMU | U599/U5A9 và H7R7/S7 theo feature table | R3/S3 khác R7/S7; U595 khác U599 |
| NeoChromVG | U5F/U5G mã hỗ trợ | Không gọi GPU U575 là VG |
| HRTIM | H743 theo DS | Không mặc định toàn bộ H7 |
| SWPMI MDIOS TT-CAN | H743 theo DS/RM | Chuyên dụng; cần đúng controller và đối tác |
| SMPS MSI | U575/L552 có SMPS và MSI; các H7 có SMPS theo part | H523 LDO; H743 không gán SMPS từ H747 |
| LPBAM | U5 theo peripheral/domain table | GPDMA và LPDMA có Stop capability khác |
| BAM | L5 | Không đồng nhất với LPBAM U5 |
| CORDIC FMAC | U575 có cả hai; H7R có CORDIC | H523 không có; H7R không tự có FMAC |
| TSC | U575, L552 | Chỉ package/chip có bảng touch sensing |
| AES PKA OTFDEC mở rộng | U585, L562 và các mã crypto khác | U575/L552 không nhận mọi tính năng của mã crypto |
| Bootflash XiP XSPI HPDMA | H7R/S | Kiến trúc khác H743; M7 không có TrustZone-M |

Phạm vi này là các nhánh tiêu biểu có tài liệu đối chiếu. U535/U545/U575/U585/U595/U599/U5A/U5F/U5G có tài nguyên khác nhau; part ngoài bảng cần hồ sơ capability riêng. Không dùng bảng này như danh sách chân hoặc BOM.

## I4 Data structure và frame của các giao tiếp bổ sung

| Interface | Framing vật lý hoặc giao thức | Dữ liệu phần mềm cần định nghĩa |
|---|---|---|
| Ethernet | Preamble/SFD, MAC addresses, EtherType/length, VLAN nếu có, payload/pad, FCS | Descriptor ownership, buffer length, packet metadata, application message |
| MII RMII | Bus MAC–PHY, timing/control theo mode | PHY configuration/link state; không phải application packet |
| MDIO MDIOS | Clause22 preamble/start/opcode/address/turnaround/data; Clause45 chỉ khi hỗ trợ | Register bank, access policy, event status |
| USB HS ULPI | USB token/data/handshake; ULPI bus register/data riêng | Descriptors, setup fields, endpoint buffers, application framing |
| SAI TDM | Frame sync, slots, bits/sample, offset/polarity | PCM layout, channel order, block metadata |
| SPDIF | Preamble, subframe, validity/user/status/parity; block192frames | PCM samples, lock/errors/channel status |
| PDM DFSDM MDF ADF | Bitstream clocked, channel/edge theo sensor | Decimated samples, rate/scaling/delay, activity events |
| LTDC RGB | Pixel clock, sync/DE, active/porch | Framebuffer format, dimensions, pitch, ownership |
| MIPI DSI | Short/long packet, data type, ECC, word count, CRC theo loại | DCS commands, pixel format, update rectangles |
| SWPMI | SOF0x7E, payload, CRC16, EOF0x7F, bit stuffing | NSB/SSB/SMB buffer layout và frame status |
| TT-CAN | CAN framing cộng scheduling/reference theo controller/protocol | Cycle/time/sequence/freshness của message |
| XSPI HyperBus | Command/address/latency/data theo memory protocol | Image header, memory geometry, integrity/boot metadata |
| IPC hai lõi | Shared memory, không có wire frame bắt buộc | Version/sequence/opcode/length/payload/generation/ownership |

## I5 SWPMI framing và các mode buffer

SWP truyền hai chiều trên một dây: master dùng miền điện áp, slave dùng miền dòng điện. SOF0x7E và EOF0x7F bao payload1–30byte và CRC16; bit stuffing phân biệt marker với data. Khối tự xử lý marker/stuffing/CRC.

NSB yêu cầu phần mềm phục vụ register theo từng nhóm data. SSB dùng DMA với buffer32byte cho một frame. SMB dùng nhiều buffer32byte và DMA circular; TX có trường length đầu buffer, RX có length/status ở phần cuối theo layout RM. Không dùng cùng layout TX cho RX. Lab phải decode length/status, thử CRC error/overrun và bàn giao ownership từng buffer. Điện áp class, clock và pin phải xác nhận trên datasheet chip. [ST SWPMI training; RM0433]

## I6 DMA U5 và L5 không tương đương H523

U575 GPDMA16channel: channel0–11 FIFO8byte,12–15 FIFO32byte và2D. LPDMA4channel dùng single transfers và linear addressing, không có FIFO/burst như GPDMA; vùng RAM truy cập cần xác nhận, thường SRAM4 cho đường LPDMA U575. GPDMA autonomous trong Stop0/1, LPDMA trong Stop0/1/2 theo cấu hình hỗ trợ. L5 DMA14channel qua DMAMUX là kiến trúc khác, không gán linked-list2D hoặc FIFO của U5.

Lab bổ sung: cùng một batch ADC, chọn GPDMA rồi LPDMA trên U575, đổi mode nguồn đúng khả năng; ghi sample count, completion và energy. Đặt thử buffer ngoài vùng LPDMA hỗ trợ để kiểm validate cấu hình trước enable, không cố tình DMA vào vùng quan trọng. [ST U5 DMA overview; RM0456; RM0438]

## I7 Nguồn chính thức và cách quản lý revision

Các liên kết dưới là điểm tra cứu. Khi dùng trong dự án, ghi revision/ngày tải và silicon revision vào báo cáo; ST có thể cập nhật PDF. Bài 53–68 dùng tài liệu H523 đã cung cấp; các bài mở rộng dùng datasheet và RM tương ứng, không dùng nguồn bán hàng bên thứ ba.

- [STM32H743 datasheet DS12110](https://www.st.com/resource/en/datasheet/stm32h743xi.pdf)
- [STM32H747 datasheet DS12930](https://www.st.com/resource/en/datasheet/stm32h747xi.pdf)
- [STM32H7A3 datasheet DS13195](https://www.st.com/resource/en/datasheet/stm32h7a3zi.pdf)
- [STM32H7R datasheet DS14360](https://www.st.com/resource/en/datasheet/stm32h7r7l8.pdf)
- [STM32H7R7 S7 documentation RM0477 và datasheet](https://www.st.com/en/microcontrollers-microprocessors/stm32h7r7-7s7/documentation.html)
- [STM32H743 H753 documentation RM0433](https://www.st.com/en/microcontrollers-microprocessors/stm32h743-753/documentation.html)
- [STM32H747 H757 documentation RM0399](https://www.st.com/en/microcontrollers-microprocessors/stm32h747-757/documentation.html)
- [STM32U575 datasheet DS13737](https://www.st.com/resource/en/datasheet/stm32u575zi.pdf)
- [STM32U595 U599 datasheet DS13633](https://www.st.com/resource/en/datasheet/stm32u599nj.pdf)
- [STM32U5F datasheet DS14395](https://www.st.com/resource/en/datasheet/stm32u5f9vj.pdf)
- [STM32U5 documentation RM0456 và errata](https://www.st.com/en/microcontrollers-microprocessors/stm32u5-series/documentation.html)
- [STM32L552 datasheet DS12737](https://www.st.com/resource/en/datasheet/stm32l552ze.pdf)
- [STM32L562 datasheet DS12736](https://www.st.com/resource/en/datasheet/stm32l562ze.pdf)
- [STM32L5 RM0438](https://www.st.com/resource/en/reference_manual/dm00346336-stm32l552xx-and-stm32l562xx-advanced-arm-based-32-bit-mcus-stmicroelectronics.pdf)
- [AN4839 cache M7](https://www.st.com/resource/en/application_note/an4839-level-1-cache-on-stm32f7-series-and-stm32h7-series-stmicroelectronics.pdf)
- [AN5371 migration L5 U5](https://www.st.com/resource/en/application_note/an5371-migration-from-stm32l5-series-to-stm32u5-series-microcontrollers-stmicroelectronics.pdf)
- [AN5645 LPBAM](https://www.st.com/resource/en/application_note/an5645-stm32u5-series-power-optimization-using-lpbam-stmicroelectronics.pdf)
- [STM32U5 LPBAM guide](https://wiki.st.com/stm32mcu/wiki/Getting_started_with_LPBAM)
- [STM32H7 peripheral training](https://www.st.com/content/st_com/en/support/learning/stm32-education/stm32-online-training/stm32h7-online-training.html)
- [STM32H7RS training](https://www.st.com/content/st_com/en/support/learning/stm32-online-training/stm32h7rs-online-training.html)

Một số URL RM dài không được dịch vụ tra cứu mở thành công trong phiên này; các khẳng định cụ thể về số khối/khả năng chip được đối chiếu datasheet mở được, tài liệu H523 cục bộ và training/application note chính thức. Chưa rà soát từng bit của toàn bộ RM H7/U5/L5 và mọi errata silicon. Các bài là nội dung học mở rộng, không thay reference manual hoặc hồ sơ validation của sản phẩm.


# Phụ lục J Sổ thực hành mode và lỗi liên ngoại vi

## J1 Bảng cấu hình bắt buộc trước khi viết driver

Mỗi lab tạo một configuration record gồm exact part/package/revision, board, Cube package version, peripheral instance, pin AF, power domain, bus clock, kernel clock, mode, IRQ priority, DMA engine/channel/request/trigger, memory region, buffer ownership và fault policy. Thiếu clock hoặc memory region thì chưa bật DMA. Thông tin này đặt trong báo cáo, không gửi nguyên record lên wire.

Các bước triển khai chung: xác nhận capability → cấp nguồn/clock và reset khối → cấu hình khi khối đang dừng → clear flags theo RM → chuẩn bị buffer/descriptor → arm đường nhận/trigger → bắt đầu nguồn phát → đo → stop nguồn phát → chờ dừng peripheral/DMA → bàn giao/release buffer. Một số ngoại vi có thứ tự riêng; sơ đồ chung phải được thay bằng thứ tự đúng IP.

## J2 Timer counter và PWM với kết quả dự kiến

Giả sử timer clock đo được64MHz, PSC0. Edge-up20kHz: ARR3199, PWM1 active-high CCR800 cho duty25%. Center20kHz: ARR1600, CCR400 cho duty25% trong mode chuẩn. Toggle marker ở update được điều khiển CPU sẽ có jitter ISR, không dùng marker này làm bằng chứng cạnh PWM bị jitter tương tự.

1. Tạo ba profile up/down/center trên TIM1. Stop CEN trước đổi CMS; đặt CNT và direction theo quy tắc khởi tạo của driver/RM.
2. Với down mode, lấy CCR theo quan hệ output của mode để giữ duty mục tiêu, sau đó kiểm bằng scope. Không lấy công thức up rồi bỏ qua endpoint.
3. Ghi waveform reference25%,50%,75%,0% và yêu cầu100%; mô tả endpoint theo mode.
4. CMS1/2/3: CCR ở miền nội; kiểm flags và direction. Không ghi CNT/DIR liên tục trong debugger rồi kết luận hướng không ổn định.
5. Giữ cùng duty nhưng đổi fPWM100kHz: edge ARR639, center ARR320. Độ phân giải khác trước; ghi bước duty nhỏ nhất từ phép đo/config.

**Báo cáo:** profile, waveform, công thức, compare/update count và ví dụ ứng dụng. Nếu clock thực tế khác64MHz, mọi con số phải tính lại; không cố ép ARR ví dụ.

## J3 Capture encoder và đo tốc độ motor

Nguồn PWM1kHz30% → period1000tick/high300tick ở counter1MHz. Với hai rising capture, delta theo modulo2^N chỉ đúng khi không có nhiều wrap không được ghi nhận. Capture falling và rising cần logic ghép pair cùng chu kỳ; input bounce/glitch có thể tạo pair giả.

Encoder1024cycle/rev, giải mãx4 →4096counts/rev khi đúng định nghĩa. Trong cửa10ms, delta≈41 cho tốc độ khoảng60rpm; lượng tử hóa đáng kể. Dùng cửa dài hơn giảm noise nhưng thêm latency. Tốc độ thấp có thể dùng period, tốc độ cao dùng delta-count/window; chuyển thuật toán cần hysteresis và valid flag.

Thực hành sweep tốc độ và đổi chiều. Ghi position count, timestamp, speed, validity và reason nếu invalid. Không dùng một biến unsigned delta cho cả tốc độ âm. Stall detection cần threshold/time window; encoder unplug không đồng nghĩa motor đứng nếu tín hiệu floating còn nhiễu.

## J4 ADC DMA circular và deadline buffer

Chọn sample rate10kframe/s,2ranks, mỗi sample uint16. Data rate40kB/s. Một buffer128frame chứa256sample=512byte; deadline xử lý12.8ms khi luân phiên hai buffer. Phân biệt frame của chuỗi ADC với một conversion và với byte.

1. Timer tạo trigger regular sequence. Tính acquisition+conversion của cả hai ranks nhỏ hơn100µs; DMA chuyển mỗi result theo request.
2. Hai node GPDMA ghi A và B rồi quay lại. Driver completion/event mode phải được cấu hình để ứng dụng nhận đúng buffer boundary.
3. Callback chỉ publish block metadata/index, timestamp, sequence; xử lý/format gửi ở task/main. Không cấp phát heap hoặc UART blocking trong callback.
4. CPU xử lý block rồi release; nếu buffer vẫn BUF_CPU lúc DMA sắp quay lại, thống kê overrun và chọn drop/stop theo yêu cầu.
5. Chèn workload20ms để vượt deadline12.8ms. Hệ thống phải báo mất data, không gửi sequence liên tục giả như không lỗi.

```c
/* Mã mô tả luồng; cần primitive đồng bộ và driver theo target. */
on_dma_block_done(index) {
    /* publish valid length, sequence, timestamp, then notify */
}
worker_loop() {
    /* acquire READY block; process before reuse deadline; release */
}
on_transfer_error(status) {
    /* stop trigger; stop DMA by IP procedure; mark data invalid */
}
```

Đo time budget gồm ISR, scheduling, processing và transport backpressure. Nếu UART bandwidth không đủ, queue vô hạn không giải quyết; giảm sample rate, batch/compress hoặc chọn transport thích hợp. Với UART1152008N1, raw capacity lý tưởng khoảng11,520byte/s, thấp hơn40kB/s trước overhead frame.

## J5 Burst FIFO trigger và 2D với pattern kiểm chứng

Memory-copy source128byte có pattern byte i→i, destination sentinel0xCC. Với width1/burst1/block128, expected toàn128byte copy. Đổi width/burst chỉ sau kiểm alignment, block unit và endpoint. Tốc độ đo gồm setup và throughput steady-state riêng.

Trigger block cho phép bắt đầu một block/hit theo cấu hình; trigger burst chia tiến độ theo burst lập trình. Count bytes hoàn tất không chỉ dựa callback total. Khi source/destination burst hoặc FIFO buộc chia transaction, phân biệt logical burst gating với bus burst vật lý theo RM.

2D:8hàng×32byte hữu ích, stride40byte → source payload256byte nằm trong span312byte từ byte đầu tới byte cuối payload. Destination256byte contiguous. Sentinel8byte padding mỗi hàng không được copy. Offset sau block là8byte trong ví dụ source contiguous increment trước đó; kiểm đúng semantics TR3/BR2 của channel6/7 H523.

Tiêm trigger quá nhanh để kiểm TOF; channel không nhất thiết tự disable. Recovery phải xóa/ghi nhận lỗi, quyết định sample validity và khởi động lại từ mốc biết trước. Không chỉ clear flag rồi tiếp tục gán timestamp đều cho dữ liệu đã mất trigger.

## J6 Giao tiếp phải thử cả frame và lỗi

Với UART/I2C/SPI/CAN/USB, tạo golden trace tối thiểu và fault trace. Golden trace chỉ ra field, byte order, bit order, frame boundary, ACK/CRC và timestamp. Fault trace gồm đối tác vắng, data truncate, invalid length, timeout, overrun hoặc disconnect phù hợp.

SPI command-read: CS active → opcode → address → dummy nếu thiết bị yêu cầu → data → peripheral hoàn tất → CS inactive. DMA complete chưa đủ điều kiện thả CS ở mọi IP. I2C register-read: START→addressW→register→reSTART→addressR→data→NACK cuối→STOP. CANFD length12byte phải dùng DLC9 theo mapping, không đặt DLC12 với ý nghĩa12byte. USB CDC message có thể chia/gộp theo endpoint transactions; application parser dựa header length.

Protocol data structure không lưu pointer hoặc compiler padding trên wire. Header field nhiều byte cần encode/decode; range validate trước buffer allocation/copy. Các capture này bổ sung cho codec host đã có, không thay test thiết bị thật.

## J7 Cache M7 và lỗi chia sẻ cache line

H743 RX buffer cần alignment32byte và kích thước/region không làm maintenance ảnh hưởng data khác. Đặt guard và metadata ngoài line chứa DMA payload. Test CPU dirty data, DMA writes rồi CPU reads; sau sửa ownership phải khớp mọi byte và guards.

Lặp cùng test với non-cacheable MPU region đúng size/alignment. So throughput và CPU processing time; không mặc định non-cacheable luôn tốt hơn. Khi clean/invalidate cần làm tròn vùng theo cache line và dùng API phù hợp; invalidation của dirty unrelated bytes có thể mất dữ liệu, nên layout là một phần correctness.

Dual-core shared ring: notification và shared payload không đủ nếu M7 còn line cache cũ. Dùng acquire/release ordering hoặc primitive phù hợp cùng policy cache; sequence/generation phát hiện queue reset. M4 không có D-cache M7 không làm shared RAM tự coherent.

## J8 Low power và đồ họa có bằng chứng nghiệm thu

U5 Stop2 LPBAM: chọn batch128mẫu, capture completion, kiểm RAM/descriptor giữ được; đo charge tích phân một chu kỳ. Nếu board regulator/LED tiêu thụ hàng mA, không dùng số dòng board để kết luận MCU không đạtµA. Ghi phạm vi phép đo và phần load loại khỏi measurement.

HMI framebuffer800×480RGB565: một frame768,000byte, hai frame1,536,000byte. LTDC60Hz đọc46.08MB/s active pixel; DMA2D full-frame copy thêm đọc+ghi tùy mode, chưa tính refresh/arbitration. Stress SDMMC/Ethernet/CPU và log underrun/frame time. Chọn dirty rectangles hoặc memory placement để giảm tải, rồi đo lại cùng điều kiện.

**Hồ sơ lab cuối:** configuration record, source/version, schematic route, tính toán, measurement captures, negative tests, faults/recovery, phần chưa kiểm. Không đánh dấu hoàn thành peripheral chỉ vì ứng dụng demo chạy một lần.


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

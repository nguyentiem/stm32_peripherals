# Lịch lab ngoại vi STM32H523

Đi theo [10 bài ngoại vi](STM32H523_reference.md), mỗi bài làm ít nhất một phép đo và một phép thử lỗi. [Lịch 259 buổi cũ](STM32H523_schedule_archive.md) được giữ để tra mode sâu; không còn là thứ tự học chính.

| Tuần gợi ý | Ngoại vi | Lab trọng tâm | Bằng chứng để qua bài |
|---:|---|---|---|
| 1 | Board/BSP, RCC, GPIO, EXTI | Pin/clock map; LED hai board revision; nút EXTI | Clock đo được, polarity đúng, số IRQ khi bounce |
| 2–3 | Timer cơ bản và advanced | 1 kHz, PWM 20 kHz, capture, encoder, break | Tần số/duty/sai số; fault đưa output về safe state |
| 4–5 | GPDMA, RAMCFG/ECC, cache | RAM copy, UART RX vòng, ADC hai buffer | Ownership timeline, overrun counter, map buffer |
| 6–7 | ADC, DAC, analog ngoài | ADC timer-DMA, DAC loopback, op-amp gain/offset | Error/timing budget và waveform đo được |
| 8 | UART/RS485/LIN | RX stress, frame parser, đo DE | FE/ORE/timeout và khôi phục frame |
| 9 | SPI/I2S/I2C/SMBus/PMBus/I3C | Sensor transaction, PEC, I3C nếu có target | Trace CS/START/ACK, thử NACK/stuck bus |
| 10–11 | FDCAN, USB FS, UCPD | CAN hai node, USB CDC reconnect, CC/PD nếu có | Bus-off/reconnect, descriptor và frame |
| 12–13 | Flash, OCTOSPI, SDMMC/FMC theo board | Journal, memory pattern, logger | Dữ liệu hợp lệ sau ngắt ghi/reboot |
| 14 | Power, LPTIM/RTC/TAMP, watchdog | Wakeup, đo dòng, lỗi khiến watchdog reset | Năng lượng, reset cause và recovery |
| 15–17 | Dự án tích hợp | Chọn logger, gateway hoặc BLDC | Báo cáo timing, lỗi, giới hạn và nghiệm thu |
| Sau đó | H7/U5/L5 nâng cao | Chọn đúng ngoại vi cho yêu cầu dự án | So sánh datasheet/RM và lab trên chip tương ứng |

## Bài tập tích hợp ưu tiên

| Dự án | Chuỗi ngoại vi | Thử lỗi bắt buộc |
|---|---|---|
| Thu tín hiệu | Timer → ADC → DMA → UART/USB | Consumer chậm, ADC overrun, mất kết nối |
| Logger | Timer/RTC → ADC/I2C/SPI → DMA → Flash/SDMMC | Mất nguồn giữa ghi, card vắng mặt |
| Gateway | FDCAN/RS485 → buffer → USB/UART | Bus-off, frame sai, RX quá tải |
| BLDC mức logic | TIM1/8 → ADC trigger → DMA → CCR preload; fault → break | Mất mẫu, trễ deadline, break và re-arm |

## Điều kiện chọn phần cứng

- H523 không có OPAMP/COMP nội. Lab BLDC cần mạch khuếch đại dòng và comparator ngoài khi topology yêu cầu; bắt đầu bằng tín hiệu mức logic và dòng giả lập.
- SDMMC/DCMI/PSSI, FMC và VREFBUF phụ thuộc package/board. Kiểm pin mux trước khi lên lịch hoặc mua module.
- Không có scope/logic analyzer thì bài timing, dead time và break latency phải ghi **chưa đo**. Không có board công suất thì nghiệm thu BLDC ở mức logic và control giả lập.
- Chỉ mở nhánh H7/U5/L5 khi dự án cần ngoại vi H523 không có hoặc topology bộ nhớ/cache khác.

**Mẫu nộp mỗi tuần:** sơ đồ đường tín hiệu → bảng cấu hình → phép tính → ảnh/log đo → lỗi chủ động → cách phục hồi. Không cần lặp lại bài học C, startup hay RTOS trong lịch này.

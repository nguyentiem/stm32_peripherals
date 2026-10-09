# Bài 13 — Flash, SRAM và cache trên STM32H533CE

> Phạm vi: **STM32H533CE**, 512 KiB user Flash, 272 KiB SRAM hệ thống và 2 KiB backup SRAM. Ví dụ ghi bền dùng pseudocode độc lập driver vì API HAL STM32CubeH5 phụ thuộc phiên bản và trạng thái TrustZone/option bytes của project.

## Mục tiêu và thứ tự học

Sau bài này, bạn phải đọc được memory map và linker script, biết sector nào chứa firmware/dữ liệu, thiết kế một bản ghi chịu mất điện khi ghi, giải thích ECC và sự khác nhau giữa ICACHE/DCACHE, và dự trù đường SWD khôi phục khi đổi option bytes. Lộ trình: **bản đồ nhớ → thao tác Flash cơ bản → bản ghi bền → ECC/cache → option bytes/boot → cập nhật hai bank**. Mỗi bước cần có phép kiểm chứng trên board trước khi sang bước sau.

**Định nghĩa và nguyên lý:** Flash là bộ nhớ không mất dữ liệu khi mất nguồn; các ô nhớ được đọc như địa chỉ thông thường nhưng phải xóa theo sector rồi mới lập trình theo đơn vị flash word. SRAM đọc/ghi nhanh từng byte/word nhưng mất nội dung khi mất nguồn nuôi tương ứng. Cache giữ bản sao của những dữ liệu vừa truy cập để giảm độ trễ; chính vì có bản sao, phần mềm phải xử lý tính nhất quán khi Flash hoặc bộ nhớ ngoài bị sửa bởi đường khác. Lợi ích của hai bank Flash là CPU có thể đọc một bank trong lúc bank kia được ghi/xóa và xây dựng quy trình cập nhật A/B; đổi lại, phân vùng firmware và recovery phức tạp hơn.

| Cách dùng/chế độ | Vì sao có và cơ chế | Chọn khi nào; đánh đổi |
|---|---|---|
| Flash thường, ghi cấu hình A/B | Erase theo sector rồi program từng word; bản commit ghi sau cùng giúp nhận diện giao dịch hoàn tất. | Cấu hình thay đổi ít, cần bền khi mất điện; tốn ít nhất hai sector và phải quản lý endurance. |
| High-cycle EDATA | Vùng dữ liệu Flash cấu hình qua option bytes có độ bền ghi cao hơn. | Counter/log ghi nhiều; phải kiểm tra vùng đã bật và layout trước khi truy cập. |
| Dual-bank read-while-write và bank swap | Hai bank vật lý cho phép đọc bank này khi sửa bank kia; swap đổi ánh xạ boot. | Cập nhật firmware A/B; cần image, linker, xác minh và rollback phù hợp. |
| ICACHE/DCACHE bật | Giữ line đã đọc để giảm chờ Flash/bộ nhớ ngoài. | Code/data ngoài được truy cập lặp lại; phải đo hiệu quả và xử lý invalidate/clean đúng đường dữ liệu. |
| BKPSRAM | Miền backup giữ ít dữ liệu khi nguồn backup còn. | Reset cause hoặc trạng thái ngắn hạn; không dùng thay Flash nếu cả VDD/VBAT đều mất. |

## 1. Bản đồ nhớ đúng cho H533CE

| Vùng | Dung lượng/đặc tính | Dùng cho | Điều cần nhớ |
|---|---:|---|---|
| User Flash bank 1 | 256 KiB, 32 sector × 8 KiB; thường `0x0800_0000–0x0803_FFFF` khi chưa swap | Code/const hoặc bản firmware A | Program theo flash word 128 bit; erase cả sector. |
| User Flash bank 2 | 256 KiB, 32 sector × 8 KiB; thường `0x0804_0000–0x0807_FFFF` | Firmware B, dữ liệu hoặc vùng dự phòng | `SWAP_BANK` đổi ánh xạ logic, không đổi bank vật lý. |
| SRAM1 / SRAM2 / SRAM3 | 128 / 80 / 64 KiB | Stack, heap, DMA buffers, dữ liệu tạm | SRAM2 có ECC; SRAM3 của H533 là 64 KiB, đừng dùng số của H543/H563. |
| BKPSRAM | 2 KiB, ECC | Ít dữ liệu cần giữ qua một số reset/Standby khi nguồn backup phù hợp | Không thay thế Flash để lưu dữ liệu khi mất cả VDD và VBAT. |
| System memory / option bytes / OTP | Vùng thông tin tách user Flash | Bootloader ST, cấu hình boot/bảo vệ, dữ liệu một lần | Không dùng làm vùng dữ liệu ứng dụng thông thường. |

Địa chỉ bank trong bảng là bản đồ **chưa swap, nonsecure alias**. Với TrustZone, còn secure alias và quyền truy cập; linker script, `FLASH_OPTSR_CUR` và map file của build là nguồn kiểm tra cuối cùng. Đừng lấy số “2 MiB Flash” trong mở đầu RM áp cho mã CE: đó là tối đa của cả họ. [DS14539 Rev 3 bảng 2, tr. 15–16](../doc/DS_stm32h533ce%20%281%29.pdf); [RM0481 Rev 5 §2.3.3–2.3.4, tr. 147; bảng 53, tr. 289](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf)

**Bài kiểm tra đầu tiên:** mở `.map` và linker script, xác nhận `.text/.rodata`, `.data`, `.bss`, stack và heap rơi đúng các vùng. Kiểm tra linker `FLASH` length là 512 KiB hoặc phân hoạch thực tế, không để vùng dữ liệu trùng firmware. Nếu có bootloader, địa chỉ bắt đầu ứng dụng phải trùng vector table/boot address.

## 2. Flash không phải RAM bền

Flash user có hai thao tác khác nhau: **erase** một sector 8 KiB đưa các byte về `0xFF`; **program** một flash word 128 bit (16 byte), thẳng hàng 16 byte, chỉ đổi bit theo khả năng lập trình Flash. Ghi lại cùng địa chỉ khi chưa erase có thể gây lỗi chương trình hoặc ECC dù nội dung trông như chỉ thay vài bit. Mỗi lần chỉ có một thao tác sửa Flash toàn thiết bị; hai bank hỗ trợ **read-while-write**, nhưng không write-while-write. [RM0481 §7.2–7.3.6, tr. 269–280](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf)

Quy trình erase/program ở mức thanh ghi:

1. Xác minh sector thuộc vùng dữ liệu đã **dành riêng trong linker** và không bị WRP/secure/HDPL bảo vệ. Đừng erase sector chứa code đang chạy hoặc vector table.
2. Chờ `BSY=0`, write buffer trống (`WBNE=0`, `DBNE=0`); đọc rồi xóa các error flag cũ trước giao dịch mới.
3. Unlock `FLASH_NSCR` hoặc `FLASH_SECCR` nếu đang khóa. Đừng gửi chuỗi unlock khi thanh ghi đã mở: RM cảnh báo có thể khóa lại đến reset.
4. Erase: chọn bank vật lý (`BKSEL`), số sector (`SNB`), bật `SER`, kích `STRT`; chờ hoàn thành với timeout, kiểm tra `WRPERR/PGSERR` và các cờ liên quan.
5. Program: bật `PG`, ghi **đủ 16 byte** vào địa chỉ căn 16 byte; chờ `BSY=0`, kiểm tra lỗi, tắt `PG`, khóa lại khi xong.
6. Read back và kiểm CRC/sequence trước khi coi bản ghi là hợp lệ; nếu code hoặc dữ liệu cacheable vừa đổi, xử lý ICACHE/MPU theo chính sách đã chọn.

Các bước trên tóm lược trình tự của RM, không thay cho bảng điện áp/timing và các điều kiện bảo vệ. `FLASH_ACR.LATENCY` và `WRHIGHFREQ` phụ thuộc clock; **không đổi `WRHIGHFREQ` khi Flash đang program/erase**. [RM0481 §7.3.5–7.3.6, tr. 276–280; §7.9, tr. 330–339](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf)

## 3. Thiết kế lưu dữ liệu an toàn khi mất điện

Một sector chứa một cấu hình duy nhất sẽ hỏng nếu mất điện sau erase nhưng trước program. Dùng **hai sector A/B**, mỗi bản ghi có `magic`, `schema_version`, `sequence`, `payload_length`, `payload_crc`, `payload`, và một dấu `commit` được ghi ở **flash word riêng cuối cùng**. Chỉ bản ghi có header hợp lệ, length trong giới hạn, CRC đúng và commit hợp lệ mới được nhận. Khi boot, chọn bản ghi có sequence mới hơn. Đảm bảo mỗi lần ghi đi vào vùng erased mới, không sửa ngược nội dung đã program.

```text
persist(new_payload):                  # Pseudocode độc lập HAL
    old = scan_valid_records(A, B)
    target = sector_khac(old.sector)
    ensure(target không chứa code/option/bootloader)
    erase_sector(target, timeout)
    assert(all_bytes_FF(target))
    header = {magic, schema, seq=old.seq+1, len, crc32(new_payload)}
    program_16_byte_words(target, header + new_payload + padding_FF)
    verify_readback_crc(target)
    program_16_byte_commit_word(target.commit_address, COMMIT_MAGIC)
    verify_record(target)
    # Chỉ sau khi target hợp lệ mới được cân nhắc xóa old.

load():
    records = [validate(A), validate(B)]
    return newest_valid_sequence(records)  # Có default khi cả hai lỗi.
```

Pseudocode cần hiện thực thêm: timeout, xử lý wrap của sequence, kiểm tra địa chỉ/độ dài, CRC của cả header quan trọng, và lỗi readback. Với cập nhật thường xuyên, dùng log nhiều record/sector để giảm erase; đặt giới hạn số lần ghi và đo endurance thực tế theo datasheet. Dữ liệu cần ghi rất thường xuyên có thể dùng **Flash high-cycle data**: H533 có tới 48 KiB mỗi bank, khả năng tới 100 kcycles theo RM; phải bật `EDATAx` qua option bytes, vùng địa chỉ riêng bắt đầu `0x0900_0000`, sector dữ liệu còn 6 KiB và write granularity 16 bit. Khi chưa cấu hình EDATA, đọc vùng này có thể bus error; đừng tự động coi nó đã sẵn sàng. [RM0481 §7.3.10, tr. 285–287; §7.6.9, tr. 315](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf)

**Ví dụ C độc lập HAL: kiểm tra bản ghi trước khi dùng.** `crc32_payload` là hàm phần mềm hoặc driver CRC đã cấu hình đúng polynomial/byte order của định dạng lưu; `commit` nằm ở flash word riêng. Hàm này không thực hiện erase/program.

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>

typedef struct {
    uint32_t magic, schema, sequence, length, payload_crc;
} record_header_t;

extern uint32_t crc32_payload(const uint8_t *data, size_t len);

bool record_is_valid(const record_header_t *h, const uint8_t *payload,
                     size_t capacity, uint32_t commit)
{
    const uint32_t RECORD_MAGIC = 0x52454344u;
    const uint32_t COMMIT_MAGIC = 0xC0DEC0DEu;
    if (h == NULL || payload == NULL || h->magic != RECORD_MAGIC ||
        h->schema != 1u || commit != COMMIT_MAGIC ||
        h->length > capacity)
        return false;
    return crc32_payload(payload, h->length) == h->payload_crc;
}
```

Trong sản phẩm, thêm CRC/header authentication và kiểm tra vùng địa chỉ trước khi dereference; `capacity` phải được tính từ sector/linker, không lấy từ header trong Flash.

**Kỹ thuật senior:** giữ một sector test riêng và script có thể xác minh địa chỉ trước khi erase. Trong production, cấm lệnh erase nếu địa chỉ không thuộc whitelist compile-time/linker. Test mất điện ở nhiều điểm của giao dịch bằng nguồn có thể ngắt lập trình được; xác nhận boot luôn tìm được bản ghi cũ hoặc mới, không bao giờ đọc cấu hình nửa chừng.

## 4. ECC: phát hiện lỗi chứ không thay CRC ứng dụng

User Flash dùng ECC cho mỗi flash word 128 bit: sửa lỗi một bit và phát hiện lỗi hai bit. `ECCC` báo dữ liệu đã được sửa; `ECCD` báo lỗi không sửa được và có thể phát NMI. Các thanh ghi `FLASH_ECCCORR`/`FLASH_ECCDETR` giữ thông tin địa chỉ và bank của lỗi; lưu snapshot **trước khi xóa cờ**. Theo RM, khi ECCD phát trong NMI cần invalidate ICACHE để tránh fetch từ line không hợp lệ. CRC/sequence vẫn cần cho tính nhất quán giao dịch và lỗi logic, vì ECC không chứng minh bản ghi đã ghi xong. [RM0481 §7.2, tr. 269; §7.9.10, tr. 336–337](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf)

SRAM2 và BKPSRAM có ECC. Khi bật bảo vệ hoặc thay đổi chế độ ECC trong RAMCFG, xem quy trình khởi tạo/erase vùng RAM để tránh đọc các word chưa được khởi tạo ECC; không suy rằng mọi SRAM của H533 đều có ECC. [RM0481 §2.3.3, tr. 147; §6 RAMCFG, tr. 241–268](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf)

## 5. ICACHE và DCACHE khác nhau thế nào?

| | ICACHE | DCACHE |
|---|---|---|
| Đường CPU | C-AHB/code bus | S-AHB/system bus |
| Kích thước | 8 KiB, line 16 byte | 4 KiB, line 16 byte |
| Nội dung | Lệnh và read từ vùng code, gồm Flash nội/ngoài | Chủ yếu dữ liệu vùng **RAM ngoài** qua OCTOSPI/FMC |
| Sau tự sửa bộ nhớ | Invalidate nếu code đã đổi trước khi execute | Clean trước DMA đọc từ RAM ngoài; invalidate sau DMA ghi RAM ngoài |
| Boot | Mặc định tắt | Mặc định tắt |

Trên H533CE, DCACHE1 chỉ cache vùng RAM ngoài; **SRAM nội đi vòng qua DCACHE**, nên buffer DMA ở SRAM1/2/3 không cần DCACHE maintenance. Điều này khác một số STM32H7. Nếu lưu record Flash rồi đọc bằng CPU, kiểm tra đường cache/MPU và invalidate ICACHE khi nội dung vùng code đã thay đổi. ICACHE không hỗ trợ write cacheable; sửa code trong vùng ICACHE đang hoạt động cần quy trình invalidate rõ ràng. [RM0481 §8.3–8.4, tr. 390–400; §9.3–9.4.8, tr. 408–417](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf)

**Giới hạn CE:** datasheet không có FMC trên mã CE; RAM ngoài nếu thiết kế có phải kiểm tra qua OCTOSPI và phần cứng board. Đừng bê ví dụ FMC từ RM dùng chung. [DS14539 bảng 2, tr. 15](../doc/DS_stm32h533ce%20%281%29.pdf)

## 6. Option bytes, boot và hai bank

Option bytes quyết định `NSBOOTADD/SECBOOTADD`, TrustZone, bảo vệ ghi, BOR, boot policy, bank swap và EDATA. Chúng có hai nhóm thanh ghi: `_CUR` đang có hiệu lực và `_PRG` giá trị chuẩn bị ghi. Đổi option bytes là thao tác bền, có thể làm thiết bị không boot hoặc hạn chế SWD; đọc snapshot hiện tại và lưu cấu hình khôi phục trước khi đổi. Với H533, bảng boot đúng là **bảng 27–28** của RM; ở trạng thái Open và TrustZone tắt, BOOT0=0 mặc định boot user Flash tại `0x0800_0000`, BOOT0=1 vào bootloader. Khi TrustZone bật, có thêm `BOOT_UBE` và secure boot address, không thể dùng quy tắc BOOT0 đơn giản cho mọi product state. [RM0481 §4.2, tr. 191; §7.4, tr. 292–299](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf)

Bank swap hỗ trợ A/B firmware: ghi và xác minh image ở bank không chạy, kiểm signature/CRC/vector, lập trình `SWAP_BANK` option, rồi reset để ánh xạ mới có hiệu lực. `BKSEL` khi erase vẫn chỉ **bank vật lý**, không phải địa chỉ logic sau swap. Một image A/B phải có linker/vector/boot plan phù hợp cả hai ánh xạ; đừng đổi option byte chỉ vì một CRC khớp mà chưa kiểm thử khả năng rollback. [RM0481 §7.3.11, tr. 287–290](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf)

## 7. SWD và khôi phục khi lỗi

- Giữ chân SWD và NRST tiếp cận được trên PCB; tránh dùng PA13/PA14 cho chức năng khác trong giai đoạn phát triển.
- Khi firmware crash ngay sau reset, dùng **connect under reset** trong ST-LINK/CubeProgrammer rồi nạp lại image hoặc sửa option bytes. Điều kiện dùng được tùy product state và chính sách debug authentication.
- Trước thay đổi WRP/TZEN/PRODUCT_STATE/SECBOOT_LOCK, lưu bản chụp option bytes, bản firmware có thể boot, và quy trình recovery đã thử trên board lab. Một số trạng thái bảo mật hạn chế hoặc yêu cầu xác thực debug; không giả định mass erase sẽ luôn mở khóa.
- Khi Flash erase/program, breakpoint có thể thay timing và watchdog vẫn chạy. Test cả khi không gắn debugger. Nếu code đang fetch cùng bank đang erase, có thể stall; chọn other-bank RWW hoặc chạy phần điều khiển tối quan trọng từ RAM theo thiết kế.

Các hành vi debug/security/boot được mô tả tại [RM0481 §4, tr. 189–191; §7.3.7, tr. 282; §7.6, tr. 305–321](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf).

## 8. Bài tập và checkpoint

| Bài tập | Tiêu chí đạt |
|---|---|
| Vẽ memory map từ linker và RM | Xác nhận chính xác bank1/2, SRAM1/2/3, BKPSRAM; chỉ ra vùng không được erase. |
| Ghi thử một sector dành riêng | Erase → đọc `0xFF` → program 16 byte → readback/CRC; xử lý timeout và error flag. |
| Hai sector A/B | Ngắt nguồn sau erase, giữa các flash word, trước/sau commit; luôn load được bản cũ hoặc mới. |
| ICACHE | Đo thời gian một đoạn code chạy lặp với cache bật/tắt và ghi hit/miss counter; giải thích sai khác. |
| Bank swap trên board thử nghiệm | Xác nhận image dự phòng boot, phiên bản mới tự kiểm, và rollback khi image lỗi. |

**Tự hỏi:** 512 KiB chia mấy bank, mấy sector mỗi bank? Vì sao erase là 8 KiB nhưng program là 16 byte? Vì sao một cờ commit trong cùng flash word payload chưa đủ? `ECCC` khác `ECCD` ra sao? Tại sao DMA buffer SRAM nội không cần DCACHE invalidate trên H533? `BKSEL` chọn bank nào sau swap? Nếu trả lời được và đo được trên board, bạn đã nắm nền tảng Flash/memory của CE.

## Nguồn chính

- [DS14539 Rev 3 — STM32H533xx](../doc/DS_stm32h533ce%20%281%29.pdf): bảng 2 biến thể CE, tr. 15–16; đặc tính điện và bộ nhớ trong §5.
- [RM0481 Rev 5](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf): §2.3 memory map, §4 boot, §6 RAMCFG, §7 Flash, §8 ICACHE, §9 DCACHE.

# Bảo mật và gia tốc mật mã trên STM32H533CE

## Mục tiêu học

Thiết kế được ranh giới secure/nonsecure, dùng đúng vai trò của RNG, AES/SAES, HASH, PKA, OTFDEC và TAMP, rồi kiểm chứng cấu hình trên **H533CE** mà không tự khóa đường nạp/debug. Thứ tự học theo phụ thuộc: mô hình đe dọa → phân vùng và boot → quản lý khóa → thuật toán → phản ứng tamper → thử nghiệm lỗi.

**Định nghĩa và nguyên lý:** bảo mật trên MCU là tập hợp cơ chế kiểm soát *ai* được chạy code, đọc bộ nhớ, dùng ngoại vi và chấp nhận firmware mới. TrustZone gắn trạng thái secure/nonsecure cho CPU và giao dịch bus; GTZC/bảo vệ bộ nhớ chặn truy cập không được cấp quyền. Các bộ gia tốc RNG/AES/HASH/PKA thực hiện phép toán mật mã nhanh hơn phần mềm, nhưng quyết định tin ai, giữ khóa ở đâu và xử lý lỗi vẫn thuộc firmware. Lợi ích thực tế là tách khóa/boot khỏi ứng dụng không tin cậy và giảm tải CPU; chi phí là linker, boot, DMA và debug phải được thiết kế nhất quán.

**Tài liệu:** [DS14539 Rev 3](../doc/DS_stm32h533ce%20%281%29.pdf), Table 2 tr. 15–16 và §3 security; [RM0481 Rev 5](../doc/rm0481-stm32h523533xx-stm32h543553xx-stm32h562xx-and-stm32h563573xx-armbased-32bit-mcus-stmicroelectronics%20%281%29.pdf), §2.2 và §3.6–3.10 (tr. 124–179), §5 GTZC (tr. 192), §7 FLASH (tr. 269), §36 RNG cho H523/533 (tr. 1456), §38 SAES (tr. 1493), §39 AES (tr. 1548), §40 HASH (tr. 1593), §41 PKA (tr. 1617), §42 OTFDEC (tr. 1656), §52 TAMP (tr. 2259). Đây là số trang in trong PDF.

## 1. H533CE có gì, và từng khối giải quyết việc gì?

DS14539 Table 2 xác nhận H533CE có RNG, SAES, AES, PKA, HASH và OTFDEC cho OCTOSPI; 4 tamper pins, 3 active tampers dạng chia sẻ output. RM0481 bao quát nhiều chip: RNG ở **§36 áp dụng H523/533**, §37 dành H543/553; SAES §38 và OTFDEC §42 có ghi rõ áp dụng H533. Đừng lấy bảng khả năng của part khác để cấu hình CE.

| Khối | Vai trò | Không nên hiểu nhầm |
|---|---|---|
| Cortex-M33 TrustZone + GTZC | Phân vùng thực thi, memory/peripheral secure hoặc nonsecure, phát hiện truy cập trái phép. | Phân vùng không tự tạo ra mã an toàn hoặc kênh gọi an toàn. |
| FLASH/option bytes | Chính sách truy cập, boot, bảo vệ nội dung Flash và trạng thái sản phẩm. | Bit bảo vệ là trạng thái lâu dài; ghi sai có thể làm mất debug hoặc yêu cầu mass erase. |
| RNG | Nguồn entropy phần cứng để cấp seed/nonce/key theo thiết kế thích hợp. | Một số ngẫu nhiên không thay thế giao thức quản lý khóa; lỗi health test phải được xử lý. |
| AES/SAES | Mã hóa/xác thực đối xứng; SAES thêm cơ chế bảo vệ khóa/side channel theo RM. | AES-ECB không bảo vệ mẫu dữ liệu; mã hóa CTR đơn thuần không xác thực. |
| HASH | SHA/HMAC cho digest/MAC. | Hash không phải chữ ký số và không chứng minh nguồn gốc nếu không có khóa/chữ ký. |
| PKA | Tăng tốc toán RSA/ECC/DH. | PKA không sinh khóa, không kiểm tra chứng chỉ hoặc thiết kế giao thức thay bạn. |
| OTFDEC | Giải mã dữ liệu/firmware từ vùng OCTOSPI ngoài trên đường đọc. | Giải mã không đồng nghĩa với xác thực tính toàn vẹn hoặc chống rollback. |
| TAMP | Phát hiện sự kiện xâm nhập và phản ứng trên dữ liệu/backup domain. | TAMP không phải cảm biến vật lý hoàn chỉnh nếu PCB và chính sách phản ứng chưa thiết kế. |

### Chọn cơ chế theo vấn đề, không theo tên ngoại vi

| Chế độ/cơ chế | Vì sao nó tồn tại và tác dụng | Trường hợp dùng; giới hạn |
|---|---|---|
| TrustZone tắt/bật | Tắt để có một không gian firmware đơn giản; bật để CPU và bus thực thi hai miền quyền và chuyển qua NSC gateway. | Bật khi phải cô lập khóa, boot hoặc dịch vụ tin cậy; chi phí là linker, giao tiếp hai miền và xử lý secure fault. Không bật chỉ vì ứng dụng có AES. |
| AES/SAES với chế độ mã hóa phù hợp | AES là phép biến đổi khối; mode như CTR hoặc GCM biến nó thành luồng/bản tin. SAES hỗ trợ chính sách khóa và chống side-channel theo RM. | Dùng AEAD như GCM/CCM khi cần cả bí mật và toàn vẹn; CTR đơn thuần cần MAC riêng. Nonce reuse có thể phá bảo mật dù phần cứng hoạt động đúng. |
| HASH/HMAC so với chữ ký PKA | HASH tạo digest không khóa; HMAC thêm khóa chung; chữ ký dùng khóa bí mật công khai để bên nhận xác minh nguồn gốc. | CRC/HASH cho lỗi vô ý, HMAC cho hai phía cùng secret, chữ ký cho firmware update phân phối rộng. Không coi digest đơn thuần là xác thực. |
| OTFDEC và secure boot | OTFDEC giải mã trên đường đọc bộ nhớ ngoài; secure boot kiểm tính xác thực trước khi chạy. | Dùng cùng nhau khi firmware ở flash ngoài vừa cần bí mật vừa cần chống sửa/rollback; OTFDEC một mình không đủ. |
| TAMP passive/active và phản ứng | Passive quan sát thay đổi tại chân; active phát mẫu và kiểm đáp ứng để khó giả mạo tín hiệu. | Bảo vệ thiết bị có vùng khóa/backup; phải thiết kế tín hiệu PCB, lọc nhiễu và chính sách phục hồi để tránh false alarm hoặc tự khóa thiết bị. |

## 2. Ranh giới secure/nonsecure

TrustZone phân biệt **trạng thái CPU**; GTZC và cơ chế bảo vệ bộ nhớ/peripheral quy định tài nguyên nào được trạng thái nào truy cập. Secure firmware sở hữu các bí mật và cấp một API nhỏ qua NSC veneer; nonsecure firmware chỉ nhận kết quả cần thiết. Xem RM0481 §2.2, §3.6 và §5. Viết “secure” trong tên biến không tạo isolation.

Thiết kế một dịch vụ ví dụ `secure_verify_firmware(image, len, signature)`:

1. Nonsecure chỉ truyền buffer/length; secure kiểm tra pointer nằm hoàn toàn trong vùng nonsecure được phép, `len` chống overflow và TOCTOU.
2. Secure tính HASH của image, PKA kiểm chữ ký bằng public key tin cậy nằm trong secure Flash hoặc được bảo vệ theo chính sách.
3. Kết quả là mã lỗi/đúng sai, không trả khóa, intermediate state hay địa chỉ secure ra ngoài.
4. Luồng update dùng version counter/chống rollback và quy tắc recovery. Xác minh trước khi thực thi, không chỉ khi tải về.

**Điểm quan trọng:** Crypto peripheral có thể phục vụ cả hai thế giới tùy cấu hình security/privilege; định nghĩa rõ chủ sở hữu, DMA master và địa chỉ buffer. Kiểm tra GTZC/TZIC fault trong cả hai miền. Nếu cấu hình sai để DMA nonsecure đọc được bí mật trong tài nguyên secure, đó là lỗ hổng; cấu hình đúng phải chặn cả CPU lẫn DMA nonsecure.

## 3. FLASH, boot và vòng đời thiết bị

Đọc RM0481 §7 và các bảng boot theo **H533/553/573** trong §4 trước khi đổi option bytes. Các chủ đề cần học: `TZEN`, vùng secure Flash, WRP, HDP, `PRODUCT_STATE`, debug authentication và rollback/recovery. RM0481 dùng mô hình product state/HDP của H5; không giả định có option byte `PCROP` hoặc `RDP` giống các dòng STM32 khác. Lập sơ đồ bộ nhớ gồm secure boot, secure services, NSC veneers, nonsecure app, dữ liệu khóa và vùng update. Xem kích thước thực tế 512 KB Flash của CE để tính vùng, không sao chép linker script từ H533 mã dung lượng khác.

**Quy tắc lab:** học và thử cơ chế khóa trên board có thể phục hồi và có bản nạp dự phòng. Đọc trạng thái hiện tại, lưu cấu hình và kiểm chứng công cụ unlock/mass erase **trước** khi ghi option bytes. Đừng bật protection cuối cùng khi chưa có bootloader/update/recovery và quy trình debug authentication đã thử. Các thay đổi option bytes không nằm trong ví dụ mã của chương này.

## 4. Chuỗi mật mã thực tế

### Sinh ngẫu nhiên

RNG §36 là entropy source phần cứng có health/status logic. Đọc cờ lỗi clock/seed trước khi sử dụng; lỗi không được biến thành “dùng số 0 rồi tiếp tục”. Nếu giao thức cần nhiều byte ngẫu nhiên, dùng RNG theo hướng dẫn của primitive/DRBG được chọn. Nonce của AES-GCM/CTR phải **duy nhất theo cùng khóa**; không thể chữa nonce reuse bằng khóa dài hơn.

### Mã hóa và xác thực

Chọn AEAD như AES-GCM hoặc CCM khi cần bí mật **và** toàn vẹn; kiểm tra tag trước khi dùng plaintext. HASH SHA-256/512 dùng cho digest, HMAC dùng khi hai phía cùng chia sẻ secret; chữ ký số cần PKA và lớp xác minh định dạng/đường tin cậy ở phần mềm. So sánh MAC/tag theo thời gian cố định phù hợp, không `memcmp` nếu kẻ tấn công đo được thời gian.

SAES có chức năng bảo vệ khóa/side channel riêng; chỉ dùng đúng key source và policy đã xác nhận trong RM0481 §38. Không đọc khóa bí mật ra RAM nonsecure để “nạp nhanh”. Việc chuyển khóa giữa AES/SAES, sao lưu hoặc bọc khóa phải dựa trên chức năng được RM cho phép; không tự coi key register là vùng lưu trữ bí mật bền vững.

### Bộ nhớ ngoài OTFDEC

OTFDEC §42 giải mã vùng OCTOSPI ngoài theo địa chỉ khi đọc. Tách ba yêu cầu: **confidentiality** (OTFDEC), **integrity/authenticity** (chữ ký/MAC/secure boot), **freshness** (version chống rollback). Firmware đã giải mã vẫn có thể bị sửa hoặc thay bằng bản cũ nếu hai yêu cầu sau thiếu. Phân vùng OTFDEC cần không chồng lấn và căn chỉnh/khóa theo RM; cache và đường DMA phải được thử với cả secure/nonsecure. CE có OCTOSPI, nhưng mạch board phải thực sự gắn bộ nhớ ngoài phù hợp.

## 5. Mã minh họa: API secure kiểm tra và chuyển nonce

Đây là **giả mã kiến trúc**, không phải đoạn mã HAL có thể biên dịch. Tên `crypto_*`, `tz_*` thể hiện việc phải nối với HAL/secure firmware đúng phiên bản và kiểm chứng theo RM. Ví dụ cố ý không ghi option bytes hay khóa thiết bị.

```c
/* Secure domain; public API đi qua NSC veneer do project TrustZone sinh. */
secure_status_t secure_encrypt_message(const uint8_t *ns_input,
                                       size_t len,
                                       uint8_t *ns_output,
                                       uint8_t *ns_nonce,
                                       uint8_t *ns_tag)
{
    if (len == 0 || len > MAX_MESSAGE ||
        !tz_nonsecure_readable(ns_input, len) ||
        !tz_nonsecure_writable(ns_output, len) ||
        !tz_nonsecure_writable(ns_nonce, NONCE_LEN) ||
        !tz_nonsecure_writable(ns_tag, TAG_LEN))
        return SEC_BAD_INPUT;

    uint8_t local[MAX_MESSAGE];
    memcpy(local, ns_input, len);       /* snapshot: tránh nonsecure đổi buffer khi xử lý */
    uint8_t nonce[NONCE_LEN];
    if (crypto_unique_nonce_from_rng_and_counter(nonce) != CRYPTO_OK)
        return SEC_RNG_ERROR;          /* fail closed */

    uint8_t tag[TAG_LEN];
    if (crypto_aes_gcm_encrypt(PROTECTED_KEY_SLOT, nonce,
                               local, len, ns_output, tag) != CRYPTO_OK)
        return SEC_CRYPTO_ERROR;
    memcpy(ns_nonce, nonce, NONCE_LEN);
    memcpy(ns_tag, tag, TAG_LEN);
    secure_zero(local, sizeof local);
    secure_zero(tag, sizeof tag);
    return SEC_OK;
}
```

Trong sản phẩm thật, giới hạn stack `MAX_MESSAGE` bằng buffer theo chunk hoặc vùng secure tĩnh kiểm soát chặt. Xác thực vùng `ns_output` không được trỏ chồng lên secret hoặc vùng secure; tốt nhất ghi ciphertext vào secure buffer rồi copy ra sau khi thành công. Hàm nonce phải kết hợp bộ đếm bền vững/quy tắc chống reuse sau reset; RNG ngẫu nhiên đơn thuần cũng cần phân tích xác suất va chạm theo số bản tin. Nhiều bản HAL không có primitive AEAD hoàn chỉnh dưới tên này: dùng thư viện crypto đã kiểm chứng và test vector chuẩn.

## 6. TAMP và phản ứng lỗi

TAMP §52 có backup registers và phát hiện nguồn tamper nội/ngoại. Xác định rõ tín hiệu board nào thật sự nối vào tamper pin của gói 48 chân, nguồn VBAT có/không, mức active, lọc nhiễu và trạng thái khi mất điện. Chính sách phản ứng có thể gồm xóa secret/backup, khóa dịch vụ hoặc reset; phải thử trong điều kiện bình thường, rung/nhiễu, mất nguồn và phục hồi. “Xóa RAM” không đủ nếu khóa còn ở Flash/backup hoặc log/debug lộ bí mật. Log sự kiện chỉ lưu mã nguyên nhân, không lưu khóa hay plaintext.

## 7. Lộ trình lab và tiêu chí đánh giá

| Bước | Thực hành | Chứng cứ đạt |
|---|---|---|
| 1 | Lập bản đồ Flash/SRAM/peripheral secure/nonsecure và sơ đồ gọi NSC. | Đường dữ liệu khóa không đi qua nonsecure; thử truy cập sai gây fault dự kiến. |
| 2 | RNG: thu mẫu, kiểm cờ lỗi và inject lỗi cấu hình clock trong môi trường lab. | API trả lỗi, không tái sử dụng nonce cũ. |
| 3 | HASH/AES: chạy test vectors được công bố cho SHA và AES-GCM; sửa 1 bit ciphertext/tag. | Kết quả đúng vector; dữ liệu bị sửa bị từ chối trước khi dùng. |
| 4 | PKA: xác minh chữ ký update hợp lệ/sai và version cũ. | Chỉ image đúng chữ ký + version được phép khởi chạy. |
| 5 | TrustZone/GTZC: gọi API secure từ nonsecure và thử pointer sai/length overflow. | Trả lỗi có kiểm soát; không lộ secure memory. |
| 6 | OTFDEC/TAMP nếu board hỗ trợ phần cứng. | Đọc ngoài giải mã đúng; tamper kích phản ứng; khởi động lại theo recovery plan. |

**Senior checklist:** threat model và tài sản cần bảo vệ đã viết ra; test vector có nguồn đáng tin; khóa có vòng đời cấp/phá/hủy; nonce không lặp qua reset; mọi lỗi RNG/tag/PKA đều fail closed; DMA và cache nằm trong bản đồ bảo vệ; cấu hình option bytes có bản sao/recovery; kiểm thử sau mất điện và firmware update; không lưu secret trong log, core dump hoặc nonsecure RAM.

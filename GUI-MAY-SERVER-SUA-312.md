# GỬI MÁY SERVER — rà soát đăng nhập và nhận thưởng bản 312

Ngày 21/10/2026. Đây là một nội dung bàn giao độc lập để máy server đọc và sửa mã; chưa phải lệnh cho phép triển khai hay chỉnh database thật.

## Baseline và yêu cầu giữ nguyên

Nguồn đã nhận từ `https://github.com/anhthe10a9/a`, nhánh `server-sync-312-20261021`, commit `dfd1903ec2580bd0dfa813f8662e91608b9d7d07`, gói `HL-SERVER-SYNC-20261021020114.zip`, SHA256 `0FC7649ECDD54E1DA4030D53147DB6D9981CA1F056387459F522EA7ADF9ED9DA`.

Máy Android đã đồng bộ `api/admin/public-web`: 205/205 file khớp gói, chưa sửa mã server khác với baseline này. Các phát hiện bên dưới được tái hiện cục bộ trên HTTP API thật chạy với PGlite/SQLite tạm; không gọi server sản xuất, không dùng tài khoản thật. Có 6 kết quả audit xác nhận hành vi hiện tại trong `verification/server-risks-312.test.cjs`; các test này xác nhận vấn đề, không phải test chứng nhận an toàn.

Giữ nguyên package ID, khóa ký/keystore, database/uploads, liên kết tài khoản thật và quy trình xác minh khóa hiện có. Không dùng bản `server-v2/` legacy làm nguồn để ghi đè. Ngoại lệ chỉ dành cho danh tính review cấu hình: `kiểm thử / 099999999 / 1998-08-18`. Không mở ngoại lệ cho provider/tag/header do client tự khai. Không gộp thay đổi giao diện/cấu hình quảng cáo hoặc iOS vào patch này.

## Những phần đang tốt — phải giữ và kiểm thử hồi quy

- Refresh đã bắt buộc deviceId, kiểm tra token còn hạn/chưa thu hồi, khóa hàng phiên và tạo token mới trong transaction. Test cũ không thể refresh lại token đã rotate.
- Android device-key proof xác minh nonce, hạn, hash khóa, chữ ký P-256 trên nguyên challenge, purpose login/transfer, khóa/user đã pin và ticket một lần. Legacy enrollment cần bằng chứng phiên còn hiệu lực đúng user/device; không chỉ tin Android ID.
- Thưởng AdMob: client báo completed không được tự cộng. Callback cần chữ ký SSV, user/unit/visit/timestamp đúng; transaction ID có unique index, credit khóa user/visit và cùng transaction. Callback trùng không cộng tiền lần hai.
- Điểm danh khóa user và một receipt/ngày; farm/care có receipt và outbox, PostgreSQL credit có idempotency; video có receipt/cooldown/cap. Test đã xác nhận gửi lại quãng đường không cộng lần hai.
- Review rút thưởng/giftcode/referral được mô phỏng, không tạo thanh toán/rút tiền thật trong các route được review-access chặn.
- API hiện chạy đủ 246/246 test tại máy dự án; admin/web 21/21. Cần giữ các test iOS/APNs/proof/policy/cách ly tài khoản hiện có.

## P1 — S1. Ràng buộc đúng session/device cho MỌI API tính năng của review

**Đã tái hiện lỗi.** `review-access.js` kiểm tra tốt device khi gọi `/v1/device/verify`, nhưng `features/addon.js` và `assertTrusted()` miễn xác minh khi user nằm trong `reviewUsers`. `authUserId()` chỉ kiểm token và user, không đối chiếu X-Device-ID. Vì thế việc verify đúng không bảo đảm mọi API sau đó cũng bị ràng buộc installation.

Cách tái hiện trong fixture:

1. Login đúng hồ sơ review với hai installation UUID khác nhau: nhận cùng user, hai deviceId A/B và hai phiên.
2. Dùng token A gọi POST `/v1/device/verify` với body `{deviceId:B}`: hiện trả 403, đúng.
3. Dùng vẫn token A gọi GET `/v1/farm` với `X-Device-ID:B`, hoặc bỏ header: hiện trả 200, sai với quy tắc phiên đúng installation.
4. Token A gọi POST `/v1/widget/register` với header `android:audit-other-device`: hiện trả 200 và tạo credential widget cho header tự chọn.

Phạm vi đã chứng minh là tài khoản review; chưa chứng minh bypass khóa cho tài khoản thường. Tuy vậy, không được mô tả hiện trạng là mọi request review đã được kiểm device đúng phiên.

**Yêu cầu sửa:** xác minh token đang còn hiệu lực + user canonical review do cấu hình/salt xác định + device của chính auth_sessions và review_devices trước mọi protected route dùng device header. Device phải thuộc chính token đó, không chỉ có trong danh sách các installation cùng user. Thiếu/sai header trả 403 `REVIEW_DEVICE_MISMATCH`; token hết hạn/thu hồi trả 401. Credential widget/push phát sinh cũng phải gắn đúng session device.

Không áp yêu cầu header bừa lên login, refresh (device trong body), clock/public endpoint hoặc auth/me đang có hợp đồng riêng. Đặc biệt `assertTrusted` đang được gọi đồng bộ ở nhiều nơi: **không đổi thành async mà quên await**, vì sẽ tạo bypass quyền nghiêm trọng. Có thể thêm guard async đã await trước dispatch protected route hoặc truyền session/device đã xác minh vào context request.

**Test phải thêm:** A/A thành công; A/B, A/header trống, A/device Android bất kỳ đều thất bại trên farm, memorials, restaurant, rewards, widget/register và đăng ký credential thông báo; token cũ/thu hồi thất bại; normal Android và iOS vẫn giữ proof như cũ. Không tạo binding vật lý hoặc ticket login/transfer cho review.

## P1 — S2. Thưởng vận động chưa chứng minh nguồn số liệu, nhất là distance

**Đã tái hiện.** Một tài khoản thường trong fixture có phiên hợp lệ và trusted device đã được xác minh gửi POST `/v1/rewards/distance/sync` với ngày server hiện tại và `distanceMeters:200000`: nhận 400 linh thạch mà không cần bằng chứng nào về quãng đường ngoài số JSON. Gửi lại nhận 0, nên chống cộng trùng đang tốt; vấn đề là tính xác thực của lần nhận đầu.

Nguồn: `economy.js` nhánh distance/walking/fitness, `activity-guard.js`. Distance chỉ kiểm khoảng 0–200.000, chưa áp guard tốc độ tương đương. Walking/running có một số timing anomaly chỉ ghi log với `blocked:false`, vẫn được cộng trong giới hạn. App có lọc bản ghi Health Connect nhập tay ở một số luồng, nhưng sửa APK/gọi API có thể bỏ qua client. Chữ ký do chính khóa client ký cũng không tự chứng minh số liệu là thật.

**Yêu cầu xử lý:**

- Thống nhất mức tin cậy của thưởng vận động trước khi coi số dư là đủ điều kiện rút. Đánh dấu nguồn/phiên bản claim, thời gian đo, nonce/requestId và tổng monotonic theo user/ngày; kiểm độ tăng theo thời gian có thể xảy ra, timestamp/day, nguồn số liệu và replay trên server.
- Với claim đáng ngờ, áp chính sách từ chối/quarantine hoặc chờ đối soát đã thống nhất; không chỉ ghi log rồi coi như dữ liệu đã xác minh. Gắn cờ rủi ro vào màn duyệt rút, không tự tịch thu số dư cũ.
- Device/App Integrity giảm giả mạo ứng dụng nhưng không biến dữ liệu sức khỏe client thành chứng cứ tuyệt đối. Nếu không có nguồn xác minh độc lập, phải ghi rõ mô hình tin cậy và giới hạn rủi ro, không quảng bá chống gian lận hoàn toàn.
- Thay đổi schema/hợp đồng phải additive/versioned; không bắt iOS gửi trường Android-only và không làm mất thưởng hợp lệ của người đang dùng. Không thu/log dữ liệu sức khỏe thô, token hoặc thông tin riêng tư ngoài nhu cầu đã được cho phép.

**Test:** số âm/NaN/ngoài giới hạn; sai ngày; tăng 200 km tức thì; gửi giảm tổng; replay/đồng thời; offline upload hợp lệ; đổi ngày Việt Nam; Android/iOS; trạng thái rủi ro khi gửi và duyệt yêu cầu rút.

## P2 — S3. Refresh cache khóa SSV khi Google đưa key_id mới

**Đã tái hiện.** `admob-ssv.js` cache khóa 6 giờ. Khi callback có key_id mới chưa nằm trong cache, hàm trả 403 ngay và không refresh. Fixture dùng hai chữ ký ECDSA hợp lệ: key 1 được cache, fetcher đã cung cấp thêm key 2, nhưng callback key 2 vẫn bị từ chối và số lần fetch còn 1.

Đây là rủi ro từ chối callback hợp lệ khi xoay khóa, không phải bypass chữ ký. Chưa khẳng định người dùng đã mất thưởng hay chính sách retry của AdMob ngoài thực tế.

**Yêu cầu:** khi gặp key chưa biết, cho phép refresh sớm có giới hạn tần suất, dùng single-flight và negative cache/anti-abuse để key_id rác không gây tải Google vô hạn. Sau refresh vẫn phải verify chữ ký trên nguyên raw query và giữ mọi kiểm tra/unique receipt hiện tại. Lỗi tải khóa dùng lỗi tạm phù hợp, không đánh đồng với chữ ký giả.

**Test:** rotate key sau cache; nhiều callback đồng thời chỉ fetch một lượt; key rác bị giới hạn; fetch lỗi không cộng; chữ ký sai/tamper/duplicate parameter/SSV replay không được nới lỏng; callback muộn hợp lệ không mất credit.

## P2 — S4. Tạo phiên trực tiếp cần nguyên tử như refresh

**Đã tái hiện bằng fault injection.** `createSession()` UPDATE thu hồi phiên cũ rồi INSERT phiên mới. Refresh/review login truyền query của transaction nên an toàn hơn; một số caller login/register gọi mặc định `query` ngoài transaction. Nếu INSERT lỗi sau UPDATE, phiên cũ đã bị thu hồi dù không cấp được phiên mới.

Nguồn: `server.js` hàm createSession và các caller login/register trực tiếp.

**Yêu cầu:** đưa thu hồi/cấp session của các caller trực tiếp vào transaction có khóa theo device/user phù hợp; rollback khi INSERT lỗi; không mở transaction lồng mù quáng khi caller đã truyền dbQuery của transaction. Không đổi thời hạn/quy tắc binding chỉ để sửa lỗi này.

**Test:** lỗi INSERT giữ phiên cũ; hai login cùng device không để lại hai phiên active ngoài chính sách; refresh một lần; review hai installation không revoke chéo; Android/iOS/legacy key phục hồi đúng.

## P2 — S5. Chốt đúng mức bảo đảm cho device-key và video ngoài app

- Route device-key pin khóa lần đầu xác minh quyền sở hữu P-256 và chống replay; **không xác minh hardware attestation, package/certificate, bootloader hoặc tính trung thực của Android ID ở nhánh đó**. Không thể coi nó là bảo đảm một máy vật lý chỉ tạo được một tài khoản khi đối thủ kiểm soát client. Nếu yêu cầu chống nhiều tài khoản/thiết bị giả, cần thiết kế xác minh lúc enroll mới bằng cơ chế server kiểm được, vẫn giữ đường di trú/khóa đã pin cho tài khoản thật và không làm iOS hỏng.
- Video ngoài app: server kiểm elapsed time, receipt, ngày/cap/cooldown; hiện policy tự ghi `rewardVerification: elapsed_return`. Đây là thưởng mở link/chờ quay lại, không phải chứng nhận đã xem video. Nếu yêu cầu thưởng cho view được xác minh, cần nguồn callback/đo lường đáng tin phù hợp; nếu không có thì chốt lại chính sách và cách ghi UI. Không sửa bằng cách tin thêm một boolean `watched` từ client.
- Không tự bật chặn root/hardware hoặc tắt thưởng của toàn bộ người đang dùng trên production trong đợt sửa này. Mọi chuyển đổi phải có phạm vi/flag, migration và test tương thích.

## Các điểm cần kiểm tra/chốt thêm trên máy server

1. **Rate limit sau reverse proxy:** `limitRequest()` lấy req.socket.remoteAddress. Xác minh cấu hình đang chạy có khiến mọi người cùng IP proxy, bị dùng chung bucket 120/phút/route hay không. Chưa tái hiện trên deployment thật. Nếu dùng forwarded IP phải chỉ tin proxy đã cấu hình, không tin header Internet tùy ý.
2. **Quyền tắt review:** hiện accountLocked bỏ qua provider review và login review gỡ is_locked. Nếu cần ngắt tài khoản test khi lộ thông tin hoặc ngừng kiểm thử, thêm cờ review enabled/phạm vi truy cập và thu hồi đúng phiên review; không tác động account thường. Đây là quyết định vận hành, không tự tắt hồ sơ người dùng đang yêu cầu test.
3. **Điểm tiện ích đổi miễn quảng cáo:** ad/start hiện luôn tạo reward_mode=cultivation, còn điểm tiện ích chỉ cộng khi receipt mode=points. Đối chiếu còn nguồn points thực tế nào khác không; tránh UI yêu cầu 1.000 điểm nhưng tài khoản mới không có đường kiếm điểm. Giữ cấu hình/SDK iOS riêng, không đổi đơn vị thưởng âm thầm.
4. **VNeID/rút thưởng:** hiện là khai báo client + Admin đối soát, không phải API xác minh VNeID tự động. Giữ kiểm tra số dư/idempotency/khóa khi duyệt; gắn thông tin rủi ro vận động cho Admin trước khi thanh toán thật. Review tuyệt đối không đi vào redemptions/thanh toán thật.
5. **Dependency:** npm audit production của API tại máy dự án báo 0 mục ở thời điểm rà soát. Không suy ra toàn bộ dịch vụ hoặc ảnh Docker hết lỗ hổng; kiểm tra cả OS/image/runtime và cấu hình ở máy server.

## Máy Android đã làm — không ghi đè lại bằng bản cũ

- Giữ cleanup nhắc lịch đến khi AuthProvider tải xong, qua ServerSession.android.tsx.
- Giữ lượt quảng cáo chờ khi API trả lỗi chung; chỉ loại đúng visit khi có receipt/mã lỗi nghiệp vụ rõ ràng.
- Thêm auth-context.android.tsx: review refresh bị từ chối quay về đăng nhập đúng hồ sơ, không thử khóa vật lý hoặc relink; mất phiên xóa device ID tổng hợp cũ. Tài khoản thật giữ fallback proof.
- Banner/video Android chỉ mở HTTPS không có credentials; bỏ microphone/overlay khỏi manifest Android; giữ các quyền/chức năng sức khỏe và chọn ảnh hiện có.
- Shared/iOS source giữ nguyên hash; không yêu cầu server thay đổi policy iOS vì các sửa client này.

## Cách bàn giao patch server và điều kiện trước triển khai

Làm trên baseline mã đang chạy đã xác minh, ưu tiên S1 trước; S2/S5 cần chốt chính sách tin cậy, không thể chữa bằng UI client. Trả về danh sách file/diff, request/response/error code, test mới và test Android/iOS cũ, hash gói patch. Không gửi `.env`, khóa, service account, token/session hoặc dữ liệu người thật qua chat/GitHub.

Chạy test trên PGlite/SQLite/DB test tách biệt, không dùng DATABASE_URL thật. Tối thiểu giữ các nhóm: auth-update-286, review-access, legacy-session-key-312, device-key-307, device-key-cleanup-307, ios-restore-296, fcm-287, SSV và reward limits/idempotency; sau đó chạy toàn bộ test API/admin. Giữ checksum mã iOS và thử luồng iOS có liên quan khi sửa server dùng chung.

Trước triển khai về sau: sao lưu nhất quán PostgreSQL, SQLite/uploads và cấu hình kín; thử khả năng phục hồi; trình bày đúng diff và kế hoạch rollback để chủ dự án duyệt. Không chạy migration hoặc restart dịch vụ thật chỉ vì đọc nội dung này. Không xóa/gộp dữ liệu tài khoản cũ để làm test đạt.

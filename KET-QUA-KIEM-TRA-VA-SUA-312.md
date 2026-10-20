# Kết quả kiểm tra và sửa dự án 312 — 21/10/2026

## Phạm vi

Rà mã ứng dụng, API/admin/web vừa đồng bộ, manifest/quyền Android, build/test/backup; kiểm tra sâu đăng nhập/refresh/proof, review isolation, vận động/video/AdMob, farm/điểm danh/receipt và rút thưởng. Không đọc/in `.env`, không gọi đăng nhập sản xuất, không đổi database/service đang chạy. Mã server vẫn khớp 205 file của gói bàn giao; các sửa server cần triển khai được gom riêng trong `GUI-MAY-SERVER-SUA-312.md` để gửi máy server.

## Đã sửa trực tiếp trong dự án

| Phần | Thay đổi | Bảo vệ iOS |
|---|---|---|
| Nhắc lịch lúc cold start | Chỉ cleanup sau khi biết kết quả tải phiên; vẫn cleanup khi mất phiên thật | ServerSession.android.tsx, file shared không đổi |
| Lượt quảng cáo chờ | Không xóa visit với lỗi HTTP chung; giữ cách ly tài khoản/receipt đúng visit | ad-reward-service.android.ts |
| Phiên review hết hạn | Không chuyển sang proof của thiết bị vật lý; về form kiểm thử; offline vẫn giữ phiên; thu hồi xóa ID cũ | auth-context.android.tsx mới; tài khoản thật vẫn dùng proof |
| Banner/video | Chỉ HTTPS không credentials hoặc ký tự điều khiển | RemoteBanner.android.tsx, external-url.android.ts và video-rewards.android.tsx |
| Quyền Android | tools:node=remove cho RECORD_AUDIO và SYSTEM_ALERT_WINDOW | Chỉ manifest Android; chưa loại camera/storage vì cần đối chiếu chọn ảnh trên các đời Android |
| Test lịch sử | Bổ sung helper thật cho fixture; timer giả trong fixture không leak; mock URL mới | Chỉ scripts/test, không đổi runtime iOS |
| Backup | Script BACKUP-DU-AN-312.ps1 lưu đủ mã/modules/API/admin/web và verification, loại dữ liệu riêng; entry point backup cũ chuyển sang script này | Không đổi mã app; backup ghi rõ source-only |
| Git | Ignore private, uploads, SQLite, signing/local properties backup, package tải về và archive backup | Không xóa tệp/khóa hay thay dữ liệu |

Backup trước thay đổi: `patch-backups/review-final312-20261021/`. Hash bảo vệ file shared/iOS lấy từ mốc đồng bộ trong `verification/server-sync312-result.json`; test bỏ qua chỉ các file rõ ràng dành riêng Android và bản sao lịch sử `.bak`, không bỏ qua file shared/iOS dùng thật. app.json, app.config.js, package/lockfile root, Swift và mã iOS dùng chung không đổi.

Backup mới là **backup mã nguồn**, không phải bản khôi phục toàn bộ hệ thống. Khóa gốc, cấu hình riêng, database/uploads phải sao lưu kín riêng; không chuyển ZIP mã lên server như một bộ triển khai. Không tự mã hóa/di chuyển khóa hoặc sửa đường dẫn keystore trong đợt này.

## Nhận xét logic server

Các cơ chế chống replay/cộng trùng/transaction đang tốt hơn bản API cũ, đặc biệt refresh và AdMob SSV. Tuy nhiên, test pass không đồng nghĩa mọi số liệu thưởng đều được chứng minh trung thực. Đã tái hiện ràng buộc installation review thiếu ở API tính năng, đầu vào quãng đường không có chứng cứ độc lập, SSV cache từ chối key mới trong TTL và lỗi tạo session ngoài transaction khi INSERT thất bại. Video elapsed-return và device-key không có attestation là các giới hạn thiết kế cần chốt rõ.

Chi tiết, mức ưu tiên và acceptance test nằm trọn trong `GUI-MAY-SERVER-SUA-312.md`. Chưa sửa/triển khai mã server khác baseline vì cần máy server xử lý các route dùng chung và giữ iOS tương thích.

## Kiểm thử

- Toàn bộ test app `node --test --test-concurrency=4 --test-timeout=15000 scripts/test-*.cjs`: **295/295 đạt**, các file cũ nay tự kết thúc.
- Bộ release Android 312: **203/203 đạt** trong script build.
- API: **246/246 đạt** với fixture tạm, DATABASE_URL thật không được dùng.
- Admin/web: **21/21 đạt**.
- Test audit rủi ro server: **6/6 xác nhận hành vi cần cảnh báo**. Các test này cố ý assert hành vi hiện tại để tái hiện, không phải chứng nhận đã sửa server.
- TypeScript đạt. Test hash xác nhận nguồn shared/iOS dùng thật và 205 file server/web không đổi ngoài các variant Android đã nêu.

## Advisory dependency

`npm audit --omit=dev` hiện báo API 0 mục; app 20 mục high tính theo cây phụ thuộc, bắt nguồn từ hai advisory gốc:

- [braces — GHSA-vfj7-8cjw-p6xm](https://github.com/advisories/GHSA-vfj7-8cjw-p6xm): lồng pattern quá sâu có thể làm cạn stack.
- [node-forge — GHSA-86w9-cpqp-85rv](https://github.com/advisories/GHSA-86w9-cpqp-85rv): kiểm tra cấu trúc DigestAlgorithm chưa đủ chặt khi verify chữ ký RSA.

Đã kiểm hash hai file đã patch trong node_modules khớp scripts/security-patches.json; cả 4 test bảo mật hiện tại đạt. npm ls chỉ thấy một bản cài braces và node-forge trong cây app. Đây là mitigation cục bộ, không phải bản upstream được npm đánh dấu đã vá; npm audit vẫn báo theo version. Chưa có patched version trong hai advisory tại thời điểm kiểm tra. Không chạy audit fix --force: đề xuất tự động đang kéo Expo/React Native về major không phù hợp. Cần tiếp tục giữ postinstall patch/hash gate và lên kế hoạch thay dependency khi có bản phù hợp, kiểm thử riêng Android/iOS.

## Những mục chưa áp dụng đại trà

1. Các sửa server/phương án chống gian lận vận động/video trong nội dung gửi máy server.
2. Loại thêm camera/storage hoặc thay quy trình chọn ảnh: chưa làm để tránh hỏng chức năng trên Android cũ.
3. Bản phục hồi kín của khóa/DB và diễn tập rollback dịch vụ: cần ở máy sở hữu dữ liệu, không thực hiện trong repo Android.
4. Nâng Expo/React Native/dependency major và sửa runtime iOS: không nằm trong đợt Android này.
5. Chưa kiểm thử trực tiếp iPhone; bảo vệ iOS trong đợt này dựa trên hash mã dùng chung không đổi và các test server/client liên quan đạt. Không kết luận toàn bộ sản phẩm chống gian lận tuyệt đối.

## Build/cài Android

BUILD-LM-ANDROID-TEST.ps1 -Serial R58W418802V đã build thành công, kiểm chữ ký gốc và cài qua `install -r` báo **Success**; MainActivity đã mở. Không gỡ app hoặc tạo AAB. APK: `C:\Users\Admin\Downloads\HuyenLich-Android-1.0.9-312.apk`; log: `C:\Users\Admin\Downloads\HL-Android-312-Build-20261021-024304.log`.

Source map xác nhận dùng đủ auth-context.android.tsx, RemoteBanner.android.tsx, ServerSession.android.tsx và ad-reward-service.android.ts. Kiểm tra aapt trực tiếp APK không thấy RECORD_AUDIO/SYSTEM_ALERT_WINDOW. Chưa thực hiện toàn bộ thao tác thưởng trên thiết bị với server thật; không dùng kết quả cài thành công để khẳng định server đã hết các rủi ro bàn giao.

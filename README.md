# Huyền Lịch — gói kiểm tra đồng bộ server 312

[Tải gói ZIP (20,3 MiB)](handoff/HL-SERVER-SYNC-20261021020114.zip?raw=true)

Gói được lấy từ server ngày 21/10/2026 (Asia/Bangkok), gồm mã API/admin/web, test, cấu hình tham khảo đã lọc bí mật, báo cáo và bảng SHA-256.

## Máy dự án cần làm

1. Tải và giải nén ZIP; đọc `README-GUI-MAY-DU-AN.md` trong gói.
2. Dùng Node.js 24 trở lên chạy `node tools/verify-package.cjs`.
3. Chạy `node tools/compare-project.cjs "C:\DUONG-DAN\DU-AN-SERVER"` để so sánh thư mục có `api` và `admin`. Có thể truyền thêm thư mục web công khai làm đối số thứ hai. Công cụ chỉ đọc, không ghi đè mã.
4. Đối chiếu luồng tài khoản review và challenge/proof với bản Android trên máy dự án. Gói này không chứa dự án Android.

## Trạng thái đã xác minh

- Bản sửa tài khoản review đã triển khai trên server: tài khoản mẫu mở được chức năng mà không bị yêu cầu gắn khóa thiết bị; tài khoản thường giữ nguyên kiểm tra bảo vệ.
- 17/17 test xác thực liên quan đã đạt. Chưa khẳng định toàn bộ test trong gói đều đã chạy.
- 89 tệp mã/cấu hình API-admin trong phạm vi đối chiếu trùng container đang chạy.
- Hai tệp web `index.html` và `app-ads.txt` khác giữa host và container: bản đang chạy nằm trong `public-web`, bản host nằm riêng trong `reference/host-public-web`.

## Nội dung không kèm

Không có database, uploads/dữ liệu người dùng, phiên đăng nhập thật, log, bản sao lưu, Docker image, node_modules, `.env` sản xuất, service-account hoặc khóa ký. Gói phục vụ đối chiếu mã, không phải bộ khôi phục hay bộ triển khai tự động. Không chạy SQL migration hoặc Compose lên server thật chỉ để kiểm tra đồng bộ.

## Kiểm tra tệp tải về

Tên: `HL-SERVER-SYNC-20261021020114.zip`

Dung lượng: 21.327.249 byte

SHA-256: `0FC7649ECDD54E1DA4030D53147DB6D9981CA1F056387459F522EA7ADF9ED9DA`

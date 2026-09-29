# Viết lại nền tảng web DoctoTek

> Đối tượng: **web DoctoTek** (`edocjo-mobile` Flutter web + backend `edocjo-api`). App mobile vẫn giữ Flutter.
> Thư mục `_nham-bayer/` là bộ tài liệu viết nhầm cho khách Bayer; chỉ để tham khảo phần kỹ thuật chung.

| File | Nội dung |
|---|---|
| [01-hien-trang.md](01-hien-trang.md) | Kiến trúc Flutter web hiện tại, realtime/media, hạ tầng serve, 17 tính năng đang hỏng do lệch backend, rủi ro bảo mật backend, những gì nên giữ |
| [02-spec-tinh-nang-va-api.md](02-spec-tinh-nang-va-api.md) | **Tiêu chí nghiệm thu:** tính năng theo màn, API theo nhóm, cách xử lý tính năng đang hỏng, URL và nội dung phải giữ |
| [03-huong-code-moi.md](03-huong-code-moi.md) | Stack (React + Vite, có SDK GetStream chính chủ), repo riêng, route, **tối ưu PC/mobile**, video/live/phụ đề, auth, deploy dev/prod, lộ trình, câu hỏi mở |
| [04-ke-hoach-thuc-thi.md](04-ke-hoach-thuc-thi.md) | Kế hoạch cho Cherry: luật, **an toàn prod**, môi trường, các cổng do người làm, task M0–M4 |
| [05-style-direction-va-ke-hoach-ui.md](05-style-direction-va-ke-hoach-ui.md) | **Style direction** (giữ gì, nguyên tắc, 12 mẫu bố cục) và **kế hoạch tối ưu giao diện, UI/UX** chia 3 mức: tinh chỉnh (tự làm), đổi bố cục (cần duyệt), nội dung mới (chờ nội dung). Bổ sung cho `doctotek-web/design/STYLE.md`, không thay thế |
| [tham-khao-ui/index.html](tham-khao-ui/index.html) | Tham khảo bố cục/UX từ 8 nền tảng học y khoa (ảnh chụp toàn trang 28/09/2026) và gợi ý từng khối trang chủ. **Chỉ học bố cục, giữ tone màu hiện tại** (cam `#D93300`, nền trắng/`#F5F5F5`). Các khối cần thêm dữ liệu chưa chốt, chưa thuộc phạm vi |

## Tóm tắt
- **Hiện trạng:**
  - Web là Flutter CanvasKit, tải khoảng 19 MB JS+WASM.
  - Desktop điều hướng bằng chuỗi state, nên URL, back và F5 chạy sai.
  - HLS private không phát được. Đóng tab là mất sự kiện tiến độ.
  - Nhiều tính năng hỏng lặng lẽ vì client lệch backend: consent chương trình, nút chia sẻ, tracking, tráo lượng giá/khảo sát, xoá tài khoản…
  - Web prod được build tay rồi deploy kèm image backend.
- **Đề xuất:**
  - React 19 + Vite SPA trong repo riêng. GetStream React SDK cho live và chat, hls.js cho VOD, socket.io-client.
  - Một codebase với layout thích ứng: bottom nav (mobile), rail (tablet), top bar + sidebar (PC).
  - Giữ nguyên path cũ.
  - Dev chạy trên `.192`; prod dùng thư mục riêng, người làm cắt chuyển.
- **Đã chốt (28/09/2026):**
  - Cherry code. Làm giống web hiện tại, không làm giống được thì chọn rẻ nhất. App giữ Flutter.
  - Demo corporate và game giữ giống bản cũ.
  - Repo `eDocJo/doctotek-web`.
  - Dev: ghi đè `dev.doctotek.com`. Prod: `doctotek.com` và `www` (đã điều tra).
- **Trạng thái (28/09/2026):**
  - Cổng G0 đã xong: DNS `dev.doctotek.com` đã trả về `.192`; repo `eDocJo/doctotek-web` đã tạo; runner `doctotek-web-dev` chạy trên `.192`; đã có tài khoản test dev; bản Flutter dev đã backup; workflow `build-web-dev.yml` cũ đã tắt.
  - Còn chờ người làm: kiểm tra origin `dev.doctotek.com` trong Google Console và Apple Service ID.
  - Cherry đang làm M0 (task `GENERAL-3` M0-deploy, `GENERAL-2` M0 nền móng).

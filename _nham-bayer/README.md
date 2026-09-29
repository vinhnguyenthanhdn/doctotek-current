# Viết lại Web học viên Bayer (bayer.doctotek.com)

| File | Nội dung |
|---|---|
| [01-phan-tich-hien-trang.md](01-phan-tich-hien-trang.md) | Phân tích bản Flutter web hiện tại: kiến trúc, video/HLS, tiến độ, bảo mật, hiệu năng, danh sách bug và nợ kỹ thuật, những gì nên giữ |
| [02-spec-tinh-nang-va-api.md](02-spec-tinh-nang-va-api.md) | Checklist tính năng theo màn hình, bảng API đang dùng, API có sẵn nhưng chưa dùng, URL cần giữ tương thích, nội dung pháp lý phải chép nguyên văn |
| [03-huong-code-moi.md](03-huong-code-moi.md) | Hướng code mới: chọn stack, cấu trúc code, bản đồ route, **tối ưu cho PC và mobile**, đặc tả trình phát video, tiến độ, bảo mật, cache/CI, thay đổi backend, kiểm thử, lộ trình, câu hỏi cần chốt |
| [04-ke-hoach-thuc-thi.md](04-ke-hoach-thuc-thi.md) | Kế hoạch cho Cherry: luật làm việc, các task M0–M4 kèm tiêu chí xong, các cổng kiểm tra do người thực hiện |

## Quyết định đã chốt (28/09/2026)
- **⚠️ Không được ảnh hưởng prod.** Prod đang có học viên thật. Cherry không được đụng `master`, server prod, DB prod, và không chạy test trên prod. Thay đổi backend phải nằm sau cờ mặc định tắt. Chỉ người làm beta và cắt chuyển, theo runbook. Chi tiết ở mục "An toàn prod" của tài liệu 04.
- **Chỉ làm web**: không phát hành app native. Sau khi chuyển đổi, repo Flutter ngừng phát triển.
- **Người code: Cherry** (AI agent). Tiến độ phụ thuộc chủ yếu vào các cổng do người làm: test iPad thật, deploy, pilot.
- **Không thêm tính năng mới.** PPTX, ngôn ngữ (chỉ tiếng Việt) và mục "Giới thiệu" giữ giống bản hiện tại.
- **Không làm giống được thì chọn phương án rẻ nhất**, ghi lại trong PR (không áp dụng cho C1–C10).
- **Tuân thủ bắt buộc**: chặn tua, chặn chuyển tab và **mọi tính năng hiện tại** phải có trên web mới. Tài liệu 02 là tiêu chí nghiệm thu; tiêu chí tuân thủ C1–C10 nằm ở mục 7.4 tài liệu 03. Phía backend chỉ cần **bật lại đoạn anti-cheat đang bị comment** trong `progress.service.ts`, bật cùng lúc với lúc chuyển sang bản mới.

## Tóm tắt một trang

**Hiện trạng:**
- Flutter web (CanvasKit), 27.5k dòng Dart.
- Tải lần đầu 11–13 MB, và do nginx để `no-store` nên **lần nào mở cũng tải lại chừng đó**.
- Có hai hệ điều hướng song song (URL, back, F5 chạy sai).
- Video trên iPad phải hack nhiều. Bản vá native HLS nằm ở nhánh chưa merge.
- Lưu mật khẩu dạng rõ trong localStorage.
- Mất tiến độ khi đóng tab.

**Đề xuất:**
- **React 19 + Vite + TypeScript**, đặt tại `bayer-api/apps/learner`, cùng monorepo với admin.
- TanStack Query, React Router v7, Tailwind + Radix, client API sinh từ Swagger.
- hls.js và pdf.js lấy từ npm, chỉ tải khi cần.
- **Một codebase, layout thích ứng:**
  - Mobile dùng bottom nav, bottom sheet, video dính trên cùng.
  - iPad dùng navigation rail.
  - PC dùng sidebar, cột lọc cố định, bảng, panel học bài 2 cột và phím tắt.
- **Video:**
  - iOS/iPadOS dùng HLS native với `?t=`.
  - Trình duyệt khác dùng hls.js, chỉ gửi Bearer tới API.
  - Heartbeat có hàng đợi và gửi bằng `fetch keepalive`.
- **Hạ tầng:** sửa cache nginx (asset có hash, cache 1 năm), CI deploy tự động, không dùng service worker.

**Lộ trình:** 5 mốc M0–M4. Thời gian lịch ước lượng thô khoảng 3–5 tuần, phần lớn là chờ test thiết bị và pilot. Làm màn **học bài** trước vì rủi ro cao nhất. Chạy beta song song rồi mới cắt chuyển.

**Việc có thể làm ngay, trước cả khi viết lại:**
1. Sửa cache nginx cho bản Flutter hiện tại: đổi `no-store` thành `no-cache` cho `.js`/`.wasm` và bật gzip cho wasm. Khi đó trình duyệt kiểm tra lại qua ETag và nhận 304 thay vì tải lại 11–13 MB. Không được đặt `immutable`, vì tên file của Flutter không có hash.
2. Merge nhánh `fix/ios-native-hls` vào `main` của `bayer-doctotek-app`.
3. Sửa `PATCH` thành `POST` cho `/notifications/read-all`.

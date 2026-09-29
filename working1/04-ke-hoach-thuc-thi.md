# 04. Kế hoạch thực thi cho Cherry

> **Người đọc:** Cherry, AI coding agent viết web DoctoTek mới.
> **Đọc trước:** [01](01-hien-trang.md) (hiện trạng), [02](02-spec-tinh-nang-va-api.md) (**tiêu chí nghiệm thu**), [03](03-huong-code-moi.md) (kiến trúc).
> **Tham chiếu hành vi:** `edocjo-mobile` nhánh `main`. **Không phải** `bayer-doctotek-app`: đó là bản clone riêng cho khách Bayer.

## Luật làm việc
1. **Làm giống web hiện tại.** Không làm giống được thì chọn phương án rẻ nhất, ghi một dòng "Lệch so với bản cũ: … vì …" trong PR. Riêng các tính năng đang hỏng thì xử lý theo bảng ở 02 §3.
2. **Không thêm tính năng mới.** Mục 1.12 của 02 làm theo mặc định đề xuất, trừ khi người chốt khác đi.
3. **Mỗi PR:** `typecheck`, `lint`, `test` xanh; E2E liên quan xanh; nếu đổi UI thì kèm ảnh chụp ở 390×844, 820×1180, 1440×900.
   - Với repo `doctotek-web`: làm trên nhánh `feat/*` hoặc `fix/*`. CI xanh thì **Cherry tự merge PR vào `main`**, workflow tự deploy lên `dev.doctotek.com`, rồi Cherry tự chạy smoke/E2E trên dev. Không cần chờ người review. Không push thẳng `main`.
4. **Chuỗi hiển thị và nội dung pháp lý:** chép từ `M/lib/l10n/app_vi.arb` và `privacy_policy_screen.dart`. Không tự viết lại câu chữ, trừ các lỗi chính tả đã liệt kê ở 02 §4.

## An toàn prod (BẮT BUỘC, ưu tiên cao nhất)

Prod DoctoTek (`backend.doctotek.com`, web trên `112.213.88.123`) **có người dùng thật** (nhân viên y tế), kèm dữ liệu y khoa, thanh toán và chứng chỉ CME.

**Cherry tuyệt đối không được:**
1. **Push, merge hay mở PR vào `main` của `edocjo-api` hoặc `edocjo-mobile`.** Push `main` của `edocjo-api` là **tự build, deploy prod và chạy migration DB**. Không sửa `edocjo-api/.github/workflows/ci-cd.yml`.
2. SSH hoặc thao tác trên server prod `112.213.88.123`. Không sửa `/var/www/web` hay nginx prod.
3. Gọi API prod (`backend.doctotek.com`) từ local, CI hay test. **Mọi môi trường của Cherry đều trỏ vào dev** (`dev.doctotek.com`). CI phải fail nếu bundle dev chứa chuỗi `backend.doctotek.com`.
4. Dùng tài khoản thật để test. Chỉ dùng tài khoản test trên dev do người cấp.
5. Tạo order thanh toán thật, redeem mã AI thật, hay gửi nội dung (bài viết, bình luận, câu hỏi AI) bằng tài khoản thật.

**Server dev `.192` còn chạy dịch vụ prod** (`media-processing`, `hls.doctotek.com`, dùng cho prod của nhiều khách). Vì vậy:
- Chỉ ghi vào thư mục web dev được cấp: `/var/www/web-dev` (runner `doctotek-web-dev` chỉ có quyền ghi thư mục này).
- **Không restart docker, không restart hay reload nginx trên `.192`.** Sửa nginx trên `.192` là việc của người (`nginx -t` rồi reload; kiểm tra ngay `hls.doctotek.com` vẫn chạy).
- Không upload video hàng loạt lên dev, vì sẽ đi qua media-processing dùng chung với prod.

**Thay đổi backend** (nếu có, ví dụ B1 HLS token): PR riêng vào `edocjo-api`, ghi `⚠️ CHẠM PROD` ở dòng đầu mô tả, tương thích ngược với app Flutter, **không có migration**, **người duyệt và merge**.

## Môi trường

| Môi trường | Dùng cho | Ghi chú |
|---|---|---|
| Local: `vite dev` + proxy về `https://dev.doctotek.com` | Code hằng ngày | Không dựng backend local (cần GetStream, S3, Blaze, Firebase thật) |
| MSW | Unit/component test, mọi trạng thái lỗi | Mock dựng theo response thật ở 02 §2 |
| Dev `.192`: **`dev.doctotek.com`**, web mới ghi đè `/var/www/web-dev` | E2E, test tích hợp, test thiết bị thật | Backend dev `:3001`, DB `edocjo_dev`, MinIO dev, **mailpit** (email không gửi ra ngoài). GetStream và Blaze có thể là dịch vụ thật dùng key dev |
| Prod | **Không dùng** | |

## Cổng (người làm)

| Cổng | Khi nào | Việc |
|---|---|---|
| G0 | Trước M0 | **Đã xong ngày 28/09:**<br>✅ DNS `dev.doctotek.com` trỏ về `.192` (DNS only).<br>✅ Repo `eDocJo/doctotek-web`.<br>✅ Runner self-hosted `dev-192-doctotek-web` (label `doctotek-web-dev`) trên `.192`. Cherry (chạy trên OVH) SSH tới `.192:22` bị timeout, nên deploy dev chỉ đi được qua runner này.<br>✅ Tài khoản test dev: 1 tester, 1 thường, lưu trong GitHub Secrets `E2E_TESTER_*`, `E2E_USER_*`, đã có hồ sơ.<br>✅ Backup bản Flutter dev ở `/var/www/web-dev.flutter-20260928`.<br>✅ Tắt workflow `Build web (dev)` của `edocjo-mobile`.<br>⏳ **Còn lại:** kiểm tra origin `dev.doctotek.com` đã có trong Google Console và Apple Service ID chưa |
| G1 | Sau M0 | Đăng nhập thử trên dev ở 3 kích thước màn |
| G2 | Sau M1 | **Thiết bị thật:** HLS private trên iPhone/iPad, livestream kèm phụ đề, thanh toán QR |
| G3 | Sau M3 | QA đối chiếu 02 |
| G4 | M4 | Duyệt runbook; người tự làm beta và cắt chuyển prod |

## Task

### M0. Nền móng
| # | Task | Xong khi |
|---|---|---|
| T0.1 | Khởi tạo repo: Vite + React 19 + TS strict + Tailwind v4; proxy dev `/api`, `/ws`, `/socket.io` về dev | `dev` chạy được; `build` ra asset có hash |
| T0.2 | Sinh client từ `https://backend-dev.doctotek.com/api/docs-json`; wrapper: Bearer, `ApiError`, timeout 20s, retry GET, refresh single-flight (`/auth/refresh-token`) | Unit test: 3 request cùng 401 thì chỉ 1 lần refresh |
| T0.3 | Auth store (access token trong bộ nhớ, refresh token trong localStorage); đăng xuất xoá sạch | F5 vẫn giữ đăng nhập |
| T0.4 | Router + `RequireAuth` (chưa đăng nhập thì về `/about` kèm `returnTo`); deep link `/course/:id`, `/speakerProfile/:id`, `/speakerPost/:id`, `/drugDetail/:id`, `/post/:id[/:ref]`, `/register?refId&utm_*` (lưu lại, mở sau khi đăng nhập, gọi `POST /referrals/track/:refCode`) | Unit test từng deep link |
| T0.5 | Token thiết kế (màu, font Inter tự host), UI primitives (Button, Dialog/Sheet thích ứng, Tabs, Toast, Skeleton, EmptyState) | |
| T0.6 | AppShell: bottom nav / rail / top bar + sidebar Profile; chuông kèm unread-count | E2E ở 3 kích thước; đổi tab thì URL đổi; back/forward đúng |
| T0.7 | `/signIn`, `/signUp` + OTP, `/forgotPassword`, `/resetPassword` (≥ 8), Google GSI, Apple JS | E2E trên dev bằng tài khoản test |
| T0.8 | Hồ sơ lần đầu: ép sang `/editProfile` + `PrivacyPolicyModal` (chỉ bật được nút khi đã cuộn tới cuối) | |
| T0.9 | CI: lint/test/build + guard `backend.doctotek.com` + deploy dev. Build trên `ubuntu-latest`; job deploy chạy `[self-hosted, doctotek-web-dev]`, tải artifact rồi `rsync -rlt --delete --chmod=D2775,F664 dist/ /var/www/web-dev/` (không dùng `-a` vì runner không chown được). Backup Flutter và tắt workflow cũ **đã làm**, không cần làm lại | Workflow xanh; `dev.doctotek.com` chạy bản mới; smoke đạt: `/` trả 200, `GET /api/users/profile` không token trả 401, `https://hls.doctotek.com/docs` trả 200 |

### M1. Khoá học
| # | Task |
|---|---|
| T1.1 | Trang chủ: banner (click tracking), mục tiêu học tập (dialog ẩn 3 ngày), popup home/AI, các rail (Tiếp tục học, CME sắp diễn ra, Chương trình mới, Dành cho bạn) |
| T1.2 | Tab Khoá học và các trang danh sách (02 §1.4), bộ lọc trên query, phân trang 20/10 |
| T1.3 | Mở khoá: consent HTML (tích được khi đã cuộn tới cuối), enroll, `FeatureGuard`, rẽ nhánh payment / multi-session / live / VOD |
| T1.4 | `VideoPlayer` + `chooseSource` (03 §7.1) + hls.js lazy + Bearer chỉ cho origin API + fullscreen iOS |
| T1.5 | SeekGuard theo `enableDragDropDuringPlayback` (chặn cả hai chiều; video xem thử thì tự do) |
| T1.6 | Tiến độ: `POST /course-progress/event` giống bản cũ, cộng thêm `stop` qua keepalive ở `pagehide`; resume dialog; completion-rate; incomplete sessions |
| T1.7 | Phụ đề VOD (`vod.vtt`, reflow 56 ký tự, `line:88%`, CC 3 chế độ) |
| T1.8 | Lượng giá/khảo sát **đúng category**, khoá lượng giá theo `evaluationThresholdPercent`; thẻ Chứng nhận/CME khoá tĩnh |
| T1.9 | Đánh giá khoá học (mỗi user một bản ghi; không gửi `rating: 0`), like, lưu, chia sẻ (link có `?ref_userid=`; referral link theo kênh gửi `courseId`) |
| T1.10 | AI tóm tắt (`/ai/proxy`, `Course_Summary_Assistant_AI`, HANDSHAKE, bộ đếm) |
| T1.11 | Livestream: GetStream Video + Chat (lazy), backstage/ended, nút bật âm thanh, phụ đề live (port `live_webvtt_client.dart`, có unit test cho phần chọn cue) |
| T1.12 | Thanh toán: `/course/:id/pay`, tạo order, QR (`order.qrCode` hoặc QR tĩnh), copy thông tin, socket `/ws/payments` (tạo lại khi đổi token), đủ 4 trạng thái |

### M2. Speaker, AI, thuốc
| # | Task |
|---|---|
| T2.1 | Speaker home (3 cột, lọc chuyên khoa, gợi ý top-speakers), trang kênh 4 tab, follow (toggle), kênh của tôi + dialog tạo/sửa, bài viết speaker (HTML đã sanitize), danh sách theo dõi, mã truy cập video-only (giống bản cũ) |
| T2.2 | AI hub (tester thấy bản Test), chat Generic/Presentation (drawer hội thoại, tìm kiếm, xoá, markdown CHOICE/IMAGE_CHOICE/OVERVIEW/OPENER, vote, copy, 403/429 hiện dialog nâng cấp), guideline, gói AI, redeem, lịch sử gói, thư viện link |
| T2.3 | Tra cứu thuốc (debounce, facet, phân trang), chi tiết (Google Docs Viewer, refresh URL đã ký khi lỗi, nhà tài trợ, chia sẻ) |

### M3. Hồ sơ và phần còn lại
| # | Task |
|---|---|
| T3.1 | Profile sidebar, ProfileOverview desktop (7 thẻ + panel), MyProfile, EditProfile (sửa lỗi ghi đè ngày sinh/workplace; layout 2–3 cột), chứng chỉ hành nghề, CV (ẩn nút xoá nếu backend không có route) |
| T3.2 | CME (danh sách, lọc, wizard 4 bước, xoá), điểm (lịch sử + bảng nhận điểm), khoá đã lưu / đã đăng ký |
| T3.3 | Cài đặt (đổi mục tiêu, đổi mật khẩu, liên hệ, chính sách, giới thiệu, đăng xuất, **xoá tài khoản chạy được**), thông báo (đọc, đọc tất cả, xoá) |
| T3.4 | Bài viết: chi tiết `/post/:id`, bình luận/trả lời, like, tạo/sửa (upload bằng FormData) |
| T3.5 | Landing `/about`, `/privacy`, `/contact`, banner mở app, trang 404 |
| T3.6 | Theo 02 §1.12: demo corporate thành **một** module dùng chung cho 3 brand (giữ hành vi, kể cả tự chuyển theo workplace); `/synbiotic` là bản build Flutter riêng chỉ chứa game (PR vào `edocjo-mobile` thêm `lib/main_synbiotic.dart`, **không merge `main`**, người duyệt), serve tĩnh ở `/synbiotic/`; lab live-translate không đưa vào |
| T3.7 | Đưa `/.well-known/apple-app-site-association` và `assetlinks.json` (lấy từ `A/.well-known/`) vào `public/.well-known/`, đúng content-type | `curl` trên dev trả JSON giống hệt prod |

### M4. Cứng hoá và cắt chuyển
| # | Task |
|---|---|
| T4.1 | Bảng đối chiếu 02 §1 từng dòng (✅/❌), kèm ghi chú "lệch so với bản cũ" |
| T4.2 | Lighthouse mobile ≥ 90 Performance cho `/home`, `/course/:id`; axe không có lỗi serious |
| T4.3 | `version.json` + banner cập nhật; `ChunkLoadError` thì reload |
| T4.4 | **Chỉ soạn, không thực thi:** runbook prod (03 §9.4), gồm lệnh kiểm tra vhost, block beta, đổi root, rollback, PR gỡ bước deploy web khỏi `ci-cd.yml` |

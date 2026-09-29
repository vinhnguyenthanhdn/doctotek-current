# 03. Hướng code mới: nền tảng web DoctoTek tối ưu cho PC và mobile

> Đầu vào: [01-hien-trang.md](01-hien-trang.md), [02-spec-tinh-nang-va-api.md](02-spec-tinh-nang-va-api.md).
> Phạm vi: **chỉ viết lại web**. App Android/iOS/Windows vẫn là Flutter (`edocjo-mobile`). Backend `edocjo-api` giữ nguyên; bản đầu **không bắt buộc sửa backend**.

## 0. Quyết định

| # | Quyết định | Trạng thái |
|---|---|---|
| 1 | Chỉ viết lại web; app mobile giữ Flutter, nên sẽ có **2 client dùng chung API** | **Đã chốt** 28/09/2026 |
| 2 | Người code: **Cherry** (AI agent tương đương Claude) | **Đã chốt** |
| 3 | Tương đương đầy đủ tính năng web hiện tại, không thêm tính năng mới ở bản đầu | **Đã chốt** |
| 4 | Không làm giống được thì chọn phương án rẻ nhất, ghi lý do trong PR | **Đã chốt** |
| 5 | **Không được ảnh hưởng prod** (có user thật) | **Đã chốt** |
| 6 | Repo riêng **`eDocJo/doctotek-web`** (private) | **Đã tạo** 28/09/2026 |
| 7 | Demo corporate, game `/synbiotic`, lab live-translate: **làm giống bản cũ** (chi tiết ở 02 §1.12) | **Đã chốt** |
| 8 | **Dev: web mới thay luôn `dev.doctotek.com`**, ghi đè bản Flutter dev hiện tại | **Đã chốt** |
| 9 | **Prod: `doctotek.com` và `www.doctotek.com`** (mình đã điều tra, xem §9.2) | Đã xác minh |

## 1. Mục tiêu đo được

| Chỉ số | Hiện tại | Mục tiêu |
|---|---|---|
| JS tải lần đầu | 11.6 MB JS + 7.3 MB WASM (chưa nén) | Shell + trang đầu **≤ 200 KB gzip**. SDK GetStream chỉ tải ở route live |
| Lần mở thứ hai | Phụ thuộc cấu hình cache trên host (chưa rõ) | Gần 0 (asset có hash, cache 1 năm) |
| LCP trên 4G, máy tầm trung | Chậm (phải chờ WASM khởi động) | ≤ 2.5 s |
| URL, back/forward, F5 | Sai trong shell desktop | Mọi màn có URL thật; bộ lọc và phân trang nằm trên query |
| Chọn chữ, Ctrl+F, a11y | Hạn chế | DOM thật, WCAG 2.2 AA cho luồng chính |
| HLS private | Không phát được | Phát được trên Chrome, Edge, Firefox, Safari macOS và iOS |

## 2. Stack

| Mảng | Chọn | Lý do |
|---|---|---|
| Ngôn ngữ, build | TypeScript strict, **Vite**, SPA | File tĩnh, deploy như hiện tại (nginx). Nội dung đều cần đăng nhập; trang public (`/about`, `/privacy`) render phía client như bản cũ |
| UI | **React 19** | **GetStream chỉ có SDK chính chủ cho React** (`@stream-io/video-react-sdk`, `stream-chat-react`); Vue không có Video SDK chính chủ. Đây là lý do chính để không dùng Vue như admin |
| Router | React Router v7 (data router, lazy route) | Guard, code-split theo route |
| Server state | TanStack Query v5 | Cache, phân trang, infinite scroll |
| Client state | Zustand (auth, prefs UI) | Ít, gọn |
| API client | Sinh từ Swagger (`/api/docs-json` của **backend dev**) bằng `openapi-typescript` + `openapi-fetch` | Tránh lệch method/path/DTO như 01 §4 |
| Style | Tailwind CSS v4 + CSS variables (token) | Mobile-first, nhẹ |
| Component | Radix UI primitives | a11y chuẩn, không kéo CSS nặng |
| Form | react-hook-form + zod | |
| Realtime | `socket.io-client` v4 | Khớp với gateway NestJS |
| Livestream, chat | `@stream-io/video-react-sdk`, `stream-chat-react`, **lazy load** | |
| Video VOD | **hls.js** (npm, lazy) + `<video>` | Phát HLS private bằng Bearer |
| Markdown (AI) | `react-markdown` + `remark-gfm`, sanitize | |
| HTML (consent, bài speaker) | DOMPurify | |
| Chuỗi | Chỉ tiếng Việt, gom vào `src/shared/i18n/vi.ts` | Bản cũ cố định `vi` |
| Google / Apple login | Google Identity Services (script), Apple JS SDK | Giống bản cũ |
| Test | Vitest + Testing Library + MSW · Playwright (chromium, **webkit**, emulate iPhone/iPad) | |

## 3. Repo và cấu trúc

- **Repo riêng `eDocJo/doctotek-web`** (private). Lý do:
  - Push `main` của `edocjo-api` là **tự deploy prod kèm migration DB**. Tách repo giúp Cherry không cần quyền ghi vào repo đó.
  - CI/CD của web độc lập, không đụng `ci-cd.yml`.
  - Không kéo theo 120k dòng backend và 100k dòng admin.
- Không dùng chung package với `edocjo-api`. Kiểu dữ liệu sinh từ Swagger là đủ.

```
src/
  app/            router.tsx, providers.tsx, AppShell.tsx (header/sidebar/bottom nav), guards
  features/
    auth/         SignIn, SignUp(+OTP), Forgot, ResetPassword, Google, Apple
    landing/      About
    home/         HomeFeed, Banners, LearningGoalsDialog, Popups
    courses/      CourseHome, lists, CourseCard, filters
    course-view/  CourseView (VOD), SessionList, Evaluation/Survey, Ratings, AiSummary
      player/     VideoPlayer, chooseSource.ts, useHls.ts, SeekGuard.ts, VodCaptions.ts
      progress/   progressEvents.ts (play/pause/seek/heartbeat/stop + keepalive)
    live/         LiveStream (GetStream Video), LiveChat (Stream Chat), LiveCaptions (poll live.vtt)
    payment/      CoursePayment, PaymentQr, usePaymentSocket
    speakers/     SpeakerHome, SpeakerProfile, MySpeaker, SpeakerPost, Follows
    ai/           AiHub, ChatAi, Guideline, Plans, Redeem, Library(links)
    drugs/        DrugSearch, DrugDetail, DocViewer (Google Docs Viewer)
    profile/      Profile, ProfileOverview(desktop), EditProfile, CME, Points, Settings, Notifications
    posts/        PostDetail, CreateEditPost, Comments
    legal/        Privacy, Contact (nội dung trong content/legal/*.md)
  shared/         api/(client, generated), ui/, hooks/, lib/(platform, format, share), i18n/vi.ts
```

**Quy tắc code:**
- Mỗi file tối đa khoảng 400 dòng.
- Page chỉ ghép component lại; logic nằm trong hook hoặc hàm thuần.
- Không có state global mutable.
- Không gọi dialog hay điều hướng trong lớp API.

## 4. Route

**Giữ nguyên path của bản cũ để khỏi phải làm bảng redirect (rẻ nhất).** Những màn trước đây chỉ là key `mainWebPage` giờ có URL riêng:

| Path | Màn |
|---|---|
| `/about`, `/privacy`, `/contact`, `/signIn`, `/signUp`, `/forgotPassword`, `/resetPassword?token=`, `/register?refId=` | Public (giữ nguyên) |
| `/home`, `/homeCourse`, `/homeSpeaker`, `/homeAi`, `/homeProfile` | 5 tab (giữ nguyên) |
| `/course/:id` | Mở khoá học: consent, rồi payment/multi-session/live/VOD (thay cho key `course_view`) |
| `/course/:id/session/:sid` | Xem session (mới, để F5 và chia sẻ được) |
| `/course/:id/pay` | Thanh toán (thay cho `course_payment_view`, `/coursePayment`, `/paymentCourse`) |
| `/suggestCourses`, `/newCourses`, `/mostPopularCourses`, `/learingPrograms`, `/coursePrograms`, `/incompleteSessions`, `/academicCourses`, `/organizationCourses`, `/rehearsalCourses`, `/trainingHubs`, `/courseSearch?type=` | Danh sách (giữ path, tham số chuyển sang query thay cho `extra`) |
| `/speakerProfile/:id`, `/mySpeakerProfile`, `/speakerFollows`, `/speakerPost/:id` | Speaker |
| `/chatGenericAi`, `/chatPresentationAi`, `/guideline*Ai`, `/plan*Ai`, `/history*PlanAi`, `/myLibary` | AI |
| `/drugSearch`, `/drugDetail/:id` | Thuốc |
| `/profile`, `/myProfile`, `/editProfile`, `/setting`, `/changePassword`, `/notifications`, `/pointHistory`, `/pointReward`, `/mySavedCourses`, `/myEnrollCourses` | Hồ sơ |
| `/post/:id` (và `/post/:id/:ref_userid` cũ), `/createPost`, `/editPost/:id` | Bài viết |

- **Guard:** route private bọc `RequireAuth`. Chưa đăng nhập thì chuyển sang `/about` (giống bản cũ trên web), lưu `returnTo`. Deep link và UTM/refCode giữ đúng luồng cũ (lưu lại, đăng nhập xong mở tiếp).
- **Hồ sơ lần đầu:** chưa có `profileInfo` thì ép sang `/editProfile` kèm modal đồng ý chính sách, giống bản cũ.

## 5. Tối ưu PC và mobile: một codebase, layout thích ứng

### 5.1 Nguyên tắc
1. **Mobile-first CSS**; mở rộng dần bằng `min-width`. Không rẽ nhánh theo "có phải web không".
2. **Thích ứng theo khả năng thiết bị:** hover và tooltip chỉ khi `(hover:hover) and (pointer:fine)`; vùng chạm ≥ 44px khi `(pointer:coarse)`.
3. **Chung component, khác bố cục.** Không làm hai cây màn hình như `HomeWebPage`/`HomeScreen`. Đổi kích thước cửa sổ chỉ đổi CSS, không re-mount, nên **livestream và video không bị ngắt**.
4. **Container query** cho các thẻ `CourseCard`, `SpeakerCard`, `DrugCard`.

### 5.2 Breakpoint

| Tên | Min | Khung |
|---|---|---|
| base | 0 | Header gọn + **bottom nav 5 tab** (có safe-area) |
| sm | 640 | Như base, lưới 2 cột |
| md | 768 | **Navigation rail** bên trái (iPad dọc) |
| lg | 1024 | **Top bar 5 tab + sidebar Profile** (thu gọn được 76px, giống bản cũ) |
| xl | 1280 | Nội dung `max-width` khoảng 1440 |
| 2xl | 1536 | Tăng số cột, không kéo giãn chữ |

> Ngưỡng 800 của bản cũ được thay bằng `lg` 1024. Tablet có layout riêng (rail) thay vì dùng layout mobile.

### 5.3 Mẫu thích ứng

| Thành phần | Mobile | PC |
|---|---|---|
| Điều hướng | Bottom nav | Top bar tab + sidebar Profile |
| Dialog (consent, mục tiêu học, xác nhận) | Bottom sheet | Modal |
| Bộ lọc (khoá học theo tổ chức/chuyên khoa, speaker) | Sheet toàn màn kèm "Áp dụng" | Cột hoặc thanh lọc cố định, áp dụng ngay |
| Rail khoá học trang chủ | Cuộn ngang `scroll-snap` | Lưới 3–5 cột, có mũi tên |
| Màn VOD | Video dính trên cùng; các tab Nội dung / Lượng giá / Khảo sát / Đánh giá / AI bên dưới | 2 cột: video và thông tin bên trái; **panel phải** gồm danh sách session, lượng giá/khảo sát, AI tóm tắt (giống panel AI bản cũ) |
| Livestream | Video trên, chat dưới (tab) | Video trái, chat phải cố định |
| Chat AI | Toàn màn, drawer trượt | 2 cột: danh sách hội thoại trái, chat phải |
| Tra cứu thuốc | Tab facet ngang, danh sách thẻ | Facet cột trái, kết quả dạng lưới |
| Sửa hồ sơ | 1 cột | 2 cột (≥ lg) / 3 cột (≥ 1500), giống bản cũ |
| Thanh toán QR | QR to ở giữa, nút "Lưu mã QR" | Chi tiết khoá bên trái, QR và thông tin chuyển khoản bên phải |

### 5.4 Mobile web (iOS/Android)
- **Viewport và bố cục:** `viewport-fit=cover`, không dùng `user-scalable=no`; chiều cao `100dvh`, padding `env(safe-area-inset-*)`.
- **Input:** font-size ≥ 16px để iOS không tự zoom.
- **Video:** `<video playsinline>`. Fullscreen trên iPhone dùng `video.webkitEnterFullscreen()`.
- **Autoplay:** chỉ phát khi có thao tác người dùng. Có nút "Bật âm thanh" cho live, giống bản cũ.
- **Ảnh:** `loading="lazy"`, có width/height.
- **Upload:** gửi `FormData` với đối tượng `File`, **không đọc cả file vào RAM**.
- **Banner mở app** giữ nguyên hành vi cũ (intent Android, App Store iOS).

### 5.5 PC
- **Focus và bàn phím:** có trạng thái focus hiển thị và "Skip to content"; Tab điều hướng được.
- **Link thật:** mọi thẻ đều là `<a href>`, nên Ctrl+click mở được tab mới.
- **Prefetch khi hover:** thẻ khoá học prefetch cả query lẫn chunk route.
- **Phím tắt player:** Space/K, ←/→ (tôn trọng cờ chặn tua), M, F.
- **Đọc nội dung:** chữ thân bài 15–16px; độ dài dòng ≤ 80 ký tự cho mô tả và bài viết.

## 6. Auth và phiên đăng nhập
- **Token:** access token giữ trong bộ nhớ; refresh token lưu localStorage. Hành vi này tương đương bản cũ nhưng ít lộ hơn, và không cần sửa backend.
- **Refresh:** single-flight: nhiều request cùng 401 thì chỉ gọi `/auth/refresh-token` một lần. Refresh lỗi thì đăng xuất.
- **Không bao giờ lưu mật khẩu.** Đổi mật khẩu để server tự kiểm tra `currentPassword`.
- **Socket dùng token mới nhất:** khi token đổi thì tạo lại kết nối socket (sửa lỗi socket static của bản cũ).
- **Google / Apple:** origin web mới phải được đăng ký trong Google Console (Authorized JavaScript origins) và Apple Service ID `doctotek.app.signin` (domain + return URL). **Người làm.**

## 7. Video, livestream, phụ đề

### 7.1 VOD
| Nguồn backend trả về | Cách phát |
|---|---|
| URL public `.m3u8` | hls.js (Chrome, Edge, Firefox, Android); `<video src>` native trên Safari |
| Đường dẫn tương đối `/api/courses/:c/sessions/:s/hls-manifest` (private, cần Bearer) | Ghép origin API, dùng **hls.js**, `xhrSetup` gắn Bearer **chỉ cho request tới origin API**; segment đã ký sẵn thì không gắn header |
| MP4 đã ký | `<video src>` |

- **iOS Safari:** native HLS không gắn được header. Cách rẻ nhất là dùng **hls.js qua ManagedMediaSource** (iOS 17.1+).
  - **Rủi ro:** dự án Bayer đã đo được hls.js/MMS trên iPad bị stall nhiều.
  - Nếu gặp stall: port cơ chế `?t=` HLS token từ `bayer-api/apps/api/src/hls-token/`. Việc này là thay đổi backend, cần duyệt riêng.
- **Chặn tua:** theo `enableDragDropDuringPlayback`, chặn cả tới lẫn lui như bản cũ.
  - Chặn trên thanh tiến độ và phím tắt.
  - Nghe sự kiện `seeking` để kéo lại vị trí (khi dùng controls native trên iOS fullscreen).
  - Video xem thử thì tua tự do.
- **Resume:** hỏi "Bạn đã xem đến …"; chỉ seek sau `canplay`.
- **Tiến độ:**
  - Giữ đúng giao thức `POST /course-progress/event` (play/pause/seek/heartbeat 30s/stop, `userSessionId` theo mỗi lần mở).
  - **Thêm gửi `stop` bằng `fetch keepalive` ở `pagehide`/`visibilitychange=hidden`.** Đây là cách rẻ để khỏi mất dữ liệu; không đổi API.
- **Phụ đề VOD:** tải `vod.vtt`, reflow cue (56 ký tự, `line:88%`), gắn `<track>` (Blob URL). Nút CC xoay vòng tắt / VI / EN+VI.

### 7.2 Livestream
- `@stream-io/video-react-sdk`: join `livestream:<livestreamId>` với camera và mic tắt. Có trạng thái backstage/ended, số người xem, nút bật âm thanh.
- `stream-chat-react`: channel `livestream:<id>`, `extraData.role` (tester thì `admin`). Token lấy từ `/courses/:id/stream-token`.
- **Phụ đề live:** poll `live.vtt` mỗi 750ms (kèm `If-None-Match`); parse `NOTE timeline-origin-ms/server-now-ms/playout-delay-ms`; chọn cue theo đồng hồ server; overlay tự vẽ; 3 chế độ. **Port nguyên thuật toán** từ `M/lib/core/webvtt/live_webvtt_client.dart`.
- GetStream API key lấy theo môi trường qua env build-time (`VITE_GETSTREAM_API_KEY`). Dev và prod dùng key khác nhau.

### 7.3 Thanh toán
- Tạo order, rồi kết nối socket `/ws/payments` (auth token), nghe `payment:approved|rejected|partial_payment|overpayment_credited`.
- Hiện `order.qrCode` nếu có, nếu không thì dùng QR tĩnh; `paymentInstructions` copy được.

## 8. Tài liệu, upload, nội dung
- **Tài liệu thuốc:** Google Docs Viewer trong iframe (sandbox), giống bản cũ.
- **Bài giảng đính kèm:** tải bằng `<a href download>` với URL đã ký, giữ đúng đuôi file.
- **Upload** (`/upload/media`, chứng chỉ, CV, CME): `FormData` + `File`. Validate phía client giống bản cũ (CME: pdf/jpg/png ≤ 10MB). Resize avatar bằng canvas (1024px, chất lượng 0.6), gửi base64 như cũ.

## 9. Build, cache, deploy

### 9.1 Cache
- Vite sinh `assets/*.[hash].js|css` → `Cache-Control: public, max-age=31536000, immutable`, và `try_files $uri =404` (thiếu file thì 404, không trả `index.html`).
- `index.html` và các route SPA → `no-cache`. Bật gzip (và brotli nếu có module).
- **Không dùng service worker.** Có `version.json`; khi phát hiện bản mới thì hiện "Tải lại để cập nhật"; gặp `ChunkLoadError` thì tự reload một lần.

### 9.2 Môi trường

| Môi trường | Serve | API | Ghi chú |
|---|---|---|---|
| Local | `vite dev` | **Proxy `/api`, `/ws`, `/socket.io` về `https://dev.doctotek.com`** | Không phải dựng backend local (vốn cần GetStream, S3, Blaze, Firebase thật) |
| Dev | **`dev.doctotek.com`** trên `.192`, **ghi đè `/var/www/web-dev`** (backup bản Flutter dev trước lần đầu) | Cùng origin, nginx đã proxy `/api/`, `/ws/`, `/socket.io/` sang backend dev `:3001` (DB `edocjo_dev`, MinIO dev, mailpit) | **Không cần sửa nginx dev.** Workflow `build-web-dev.yml` của `edocjo-mobile` (vốn `rsync --delete` bản Flutter vào đúng thư mục này) **đã bị tắt** ngày 28/09; không bật lại. Bản Flutter dev đã backup ở `/var/www/web-dev.flutter-20260928` |
| Prod | **`doctotek.com` + `www.doctotek.com`** (Cloudflare proxy) trên `.123`. Web mới đặt ở **thư mục mới** `/var/www/web-next/releases/<sha>` + symlink `current` | Web prod cũ gọi `backend.doctotek.com` (khác origin; CORS của Nest cho phép `*.doctotek.com`). Web mới làm giống: gọi `https://backend.doctotek.com/api` | Không ghi vào `/var/www/web`: job CI backend `docker cp` bản Flutter vào đó mỗi lần push `main` |

**Kết quả điều tra prod (28/09/2026, chỉ kiểm tra từ bên ngoài, không SSH vào prod):**
- `doctotek.com` và `www.doctotek.com` trả bản Flutter 0.0.88+296. `last-modified` là 26/09, khớp `A/web/version.json`. SPA fallback chạy (`/course/abc` trả 200).
- `web.doctotek.com` trả **520**: Cloudflare không kết nối được origin, nên đây là vhost chết. Header CORS `web.doctotek.com` trong `A/nginx.conf` đã vô dụng.
- `sharing.doctotek.com` **không có DNS**: `SHARE_BASE_URL` mặc định của app trỏ vào domain chết (app cũng không dùng).
- `backend.doctotek.com` serve admin Vue ("Doctotek Admin Panel") kèm `/api`.
- **Mọi file trên `doctotek.com` đều `Cache-Control: no-cache, no-store, must-revalidate`**, kể cả `main.dart.js` 11.6 MB và wasm. Mỗi lượt mở trang tải lại khoảng 19 MB.
- `/.well-known/apple-app-site-association` và `/.well-known/assetlinks.json` là file tĩnh (bản gốc ở `A/.well-known/`). **Bắt buộc có trong web mới hoặc giữ ở vhost**, nếu không Universal Link và App Link trên điện thoại sẽ hỏng.
- Tên file cấu hình nginx trên `.123` chưa đọc; bước 1 của runbook §9.4 sẽ làm việc này.

### 9.3 CI (repo web)
- Mỗi PR: lint, typecheck, test, build, và kiểm tra ngân sách bundle.
- **Guard:** bundle dev **không được chứa `backend.doctotek.com`** (giống `build-web-dev.yml:56-65`).
- **Deploy dev:** build trên `ubuntu-latest`, rồi job deploy chạy trên runner self-hosted **`[self-hosted, doctotek-web-dev]`** trên `.192` (user `webdev-runner`, chỉ ghi được `/var/www/web-dev`, không có docker, không sudo) để tải artifact và `rsync --delete`.
  - Runner được đăng ký riêng cho repo này ngày 28/09.
  - Trên `.192` **không hề có runner `dev-192`**: label trong `build-web-dev.yml` cũ không có máy nào nhận job. Runner duy nhất có sẵn là `vps-hls-runner` của `media-processing-service` (prod), **không được dùng**.
- Deploy prod: **chỉ chạy tay** (`workflow_dispatch` + GitHub Environment có người duyệt), sau khi cắt chuyển được duyệt.

### 9.4 Cắt chuyển prod (chỉ người làm, theo runbook)
1. Đọc (chỉ đọc) server block của `doctotek.com`/`www.doctotek.com` trên `.123`: `root`, `location /.well-known`, header cache, SPA fallback.
2. Beta: thêm **server block mới** `beta.doctotek.com` trỏ vào `/var/www/web-next/current` (có `.well-known`). Không sửa block đang chạy. Thêm origin beta vào Google Console và Apple Service ID.
3. Cắt chuyển: đổi `root` của vhost prod sang `/var/www/web-next/current`, `nginx -t`, reload. Rollback bằng cách đổi `root` về `/var/www/web` rồi reload.
4. Sau 2 tuần ổn định: bỏ bước `COPY web` và "Deploy web" trong `edocjo-api/ci-cd.yml`, rồi xoá `A/web/`. Đây là một PR riêng vào `edocjo-api`, **người duyệt và merge**.

## 10. Thay đổi backend (không bắt buộc cho bản đầu)

| # | Thay đổi | Khi nào |
|---|---|---|
| B1 | HLS token `?t=` cho iOS (port từ bayer-api) | Chỉ khi hls.js/MMS trên iOS bị stall |
| B2 | Merge `feature/tracking-user-event` (tracking, session ratings) | Do team backend quyết định; web bật lại tính năng khi có |
| B3 | Sửa các lỗ bảo mật ở 01 §5 | Báo người phụ trách; độc lập với web |
| B4 | ~~CORS~~ **Không cần.** Web mới dùng đúng origin của web cũ (`doctotek.com`, `www.doctotek.com`), và Nest đã cho phép `^https?://(www\.)?doctotek\.com$` (`A/src/main.ts:111`). Header `web.doctotek.com` trong `A/nginx.conf` thuộc vhost đã chết | Chỉ xem lại nếu đổi sang origin ngoài `*.doctotek.com` |

**Mọi thay đổi backend phải tương thích ngược với app Flutter đang chạy trên điện thoại người dùng.**

## 11. Kiểm thử
- **Unit:** `chooseSource`, SeekGuard, progress events (keepalive), thuật toán phụ đề live (chọn cue theo đồng hồ server), reflow VTT, refresh single-flight, socket tái tạo khi đổi token, build link chia sẻ.
- **Component (MSW):** Auth (kèm OTP), EditProfile (không ghi đè ngày sinh/workplace), lượng giá/khảo sát (đúng category), Payment theo từng trạng thái socket, AI chat (CHOICE/IMAGE_CHOICE).
- **E2E (Playwright, chromium + webkit + iPhone/iPad emulation) trên dev:**
  - Đăng nhập → trang chủ → mở khoá → xem VOD → lượng giá → đánh giá.
  - Livestream (session test).
  - Thanh toán QR, dùng webhook giả ở dev nếu có.
  - Deep link khi chưa đăng nhập.
  - Back/forward/F5 ở mọi màn.
- **Thiết bị thật trước khi cắt chuyển:** iPhone Safari, iPad Safari/Chrome, Android Chrome, Windows Chrome/Edge, macOS Safari. Kiểm tra: HLS private, live kèm phụ đề, fullscreen, autoplay, upload, Google/Apple login.

## 12. Lộ trình (Cherry code; thời gian bị chặn chủ yếu bởi các cổng do người làm)

| Mốc | Nội dung | Cổng (người) |
|---|---|---|
| M0 Nền móng | Repo, Vite/TS/Tailwind/Router/Query, client Swagger, AppShell responsive (bottom nav / rail / top bar + sidebar), auth (email, OTP, Google, Apple), guard, deep link/UTM, CI + deploy dev | ✅ Repo, runner, tài khoản test (28/09). ⏳ Kiểm tra origin Google/Apple |
| M1 Khoá học (rủi ro cao nhất) | Trang chủ, danh sách, mở khoá (consent/payment/multi/live/VOD), player HLS, chặn tua, tiến độ, phụ đề VOD, lượng giá/khảo sát, đánh giá, AI tóm tắt, livestream + chat + phụ đề live, thanh toán QR + socket | **Test thiết bị thật** (iOS HLS private, live) |
| M2 Speaker, AI, thuốc | Speaker (4 tab, follow, tạo kênh), AI (hub, chat, guideline, gói, redeem, thư viện), tra cứu thuốc | QA |
| M3 Hồ sơ và phần còn lại | Profile/ProfileOverview, EditProfile, CME, điểm, cài đặt, thông báo, bài viết, landing, pháp lý | QA theo 02 |
| M4 Cứng hoá và cắt chuyển | Đối chiếu 02 từng dòng, Lighthouse, a11y, runbook, beta, cắt chuyển | Người duyệt runbook và **tự thực hiện** |

Quy mô gấp khoảng 3 lần bản Bayer (75 route, livestream, AI, thanh toán). Ước lượng thô: khoảng **5–8 tuần** lịch, phần lớn là chờ các cổng.

## 13. Câu hỏi mở

Đã chốt hết (mục 0). Gặp chỗ chưa rõ thì làm giống bản cũ; không làm giống được thì chọn phương án rẻ nhất và ghi lại trong PR.

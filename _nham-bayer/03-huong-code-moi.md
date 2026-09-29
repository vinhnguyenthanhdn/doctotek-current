# 03. Hướng code mới: Web học viên tối ưu cho PC và mobile

> Đầu vào: [01-phan-tich-hien-trang.md](01-phan-tich-hien-trang.md), [02-spec-tinh-nang-va-api.md](02-spec-tinh-nang-va-api.md).
> Phạm vi: viết lại **web học viên** (`bayer.doctotek.com`). Backend NestJS giữ nguyên, chỉ bổ sung một số điểm liệt kê ở mục 11.

## 0. Quyết định đã chốt (28/09/2026)

| # | Quyết định | Hệ quả |
|---|---|---|
| 1 | **Chỉ làm web.** App Android/iOS native không phát hành | Sau khi chuyển đổi xong, repo `bayer-doctotek-app` (Flutter) chỉ còn để tham chiếu và ngừng phát triển. Không cần giữ cùng lúc hai client |
| 2 | **Người code: Cherry**, là một AI coding agent có năng lực tương đương Claude | Thời gian viết code không còn là nút cổ chai. Tiến độ phụ thuộc vào việc review, test trên thiết bị thật và deploy backend (mục 13) |
| 3 | **Chặn tua và chặn chuyển tab là yêu cầu tuân thủ bắt buộc.** **Mọi tính năng của web hiện tại phải có trên web mới** | Mục 1 của tài liệu 02 là tiêu chí nghiệm thu: thiếu bất kỳ dòng nào thì chưa được phát hành. Phía server phải kiểm tra tiến độ (backend #2) **trước khi phát hành**. Xem thêm mục 7.4 |
| 4 | **Không thêm tính năng mới ở bản đầu.** Hành vi PPTX, ngôn ngữ và mục "Giới thiệu" giữ giống bản hiện tại | PPTX: hiện PDF đã convert (giống prod; nhánh Office Online chưa từng lên prod). Chỉ tiếng Việt. "Giới thiệu" giữ nguyên, bấm vào không làm gì. Các API ở mục 2.1 tài liệu 02 để dành cho sau |
| 5 | **Chỗ nào không làm giống được thì chọn phương án rẻ nhất** (ít code, ít phụ thuộc, không phải đổi backend nếu tránh được) | Người code tự quyết, không cần hỏi lại; ghi lại lựa chọn trong PR. Không được áp dụng cho các tiêu chí tuân thủ C1–C10 |

## 1. Mục tiêu đo được

| Chỉ số | Hiện tại (Flutter) | Mục tiêu |
|---|---|---|
| JS tải lần đầu (gzip) | khoảng 5.9 MB JS + 5–7 MB WASM, chưa nén | **≤ 200 KB** cho shell + trang đầu |
| Lần mở thứ hai | Tải lại toàn bộ do `no-store` | Gần 0 KB (asset có hash, cache 1 năm) |
| LCP trên 4G, máy tầm trung | Vài giây, phải chờ WASM khởi động | **≤ 2.5 s** |
| INP | – | ≤ 200 ms |
| Tỉ lệ phiên video bị stall trên iPad | khoảng 70% (trước fix), đang đo lại sau fix | ≤ máy khác (khoảng 11%), theo dõi qua `system_logs` |
| Mất tiến độ khi đóng tab | Tới 30 s | 0 (gửi heartbeat cuối bằng keepalive) |
| Back/forward, F5, chia sẻ link | Sai ở nhiều trang | Mọi trang có URL thật, kể cả bộ lọc và phân trang |
| Chọn chữ, Ctrl+F, trình đọc màn hình | Không có | Có sẵn (DOM thật), WCAG 2.2 AA cho luồng chính |

## 2. Chọn công nghệ

### 2.1 So sánh

| | **A. React + Vite (SPA)** (đề xuất) | B. Next.js (SSR) | C. Viết lại vẫn bằng Flutter web |
|---|---|---|---|
| Tải lần đầu | Nhỏ, code-split theo route | Nhỏ, có SSR | Vẫn phải tải CanvasKit/skwasm vài MB (Flutter đã bỏ HTML renderer) |
| `<video>`, iframe, PDF | Phần tử DOM thật, không cần hack platform view | Như A | Vẫn phải dùng platform view và hack CSS |
| Chọn chữ, a11y, Ctrl+F | Có sẵn | Có sẵn | Hạn chế |
| Hạ tầng | File tĩnh trên nginx như hiện tại, cùng origin `/api` | Cần thêm Node server hoặc container | Như hiện tại |
| Dùng lại code | Cùng stack với `apps/web` (admin), dùng chung `@bayer/shared` và cách viết test | Như A nhưng khác mô hình | Không dùng chung được gì |
| SEO | Không cần: gần hết trang nằm sau đăng nhập | Thừa | – |
| Rủi ro nhân sự | Team mobile cần học React | Như A, học nhiều hơn | Ít nhất |

**Chọn A.** Web học viên gần như toàn bộ nằm sau đăng nhập, nên SSR không đem lại lợi ích gì mà còn thêm một server phải vận hành. Vite SPA deploy y như cách hiện tại (file tĩnh + nginx), cùng monorepo và cùng stack với admin.

> Đã chốt: không phát hành app native, nên không có lý do nào để giữ Flutter. Toàn bộ client học viên chuyển sang React.

### 2.2 Stack cụ thể

| Mảng | Chọn | Lý do |
|---|---|---|
| Ngôn ngữ/build | TypeScript strict, **Vite** | Giống `apps/web` |
| UI | **React 19** | |
| Router | **React Router v7 (data router)**, lazy route | Loader/guard, code-split theo route |
| Server state | **TanStack Query v5** | Cache, retry, phân trang, infinite scroll; admin cũng đang dùng |
| Client state | **Zustand** (rất ít: auth, UI prefs) | Admin cũng đang dùng |
| API client | **Sinh từ OpenAPI** (`/api/docs-json`) bằng `orval` hoặc `openapi-typescript` + `openapi-fetch` | Không lệch method/path, loại bỏ các bug như `read-all` |
| Style | **Tailwind CSS v4** + design token CSS variables | Mobile-first, bundle nhỏ |
| Component | **Radix UI primitives** (hoặc shadcn/ui) | a11y chuẩn (Dialog, Popover, Tabs, Select), không kéo CSS nặng. **Không dùng antd** cho app học viên: bundle nặng và mang phong cách admin desktop |
| Form | react-hook-form + **zod** | |
| Chuỗi hiển thị | Chỉ **tiếng Việt**, gom vào `shared/i18n/vi.ts` (object hằng có kiểu). Không dùng thư viện i18n | Giống bản hiện tại. Rẻ nhất, sau này vẫn chuyển sang i18next được |
| Video | **hls.js** (npm, lazy-load) + `<video>` native | Không fork, không lấy từ CDN |
| PDF | **pdfjs-dist** (npm, tự host worker) | Admin đang dùng bản 6.x |
| Ngày giờ | dayjs | |
| Test | **Vitest** + Testing Library + **MSW** · **Playwright** (Chromium, **WebKit**, emulate iPhone/iPad) | Admin đã có MSW và Playwright |

## 3. Đặt code ở đâu

Thêm workspace mới trong monorepo `bayer-api`:

```
bayer-api/
  apps/
    api/          (có sẵn)
    web/          (có sẵn, admin)
    learner/      ← MỚI: web học viên
  packages/
    shared/       (có sẵn: enums, constants)
    api-client/   ← MỚI (tuỳ chọn): client sinh từ OpenAPI, dùng chung cho learner và admin
  web/            (bản build Flutter hiện tại, xoá sau khi cắt chuyển)
```

- **Lợi ích:** dùng chung `@bayer/shared` (SystemRole, enum trạng thái...). Sửa API và web trong cùng một PR, CI build cùng lúc.
- **Bỏ việc commit bản build vào git.** CI build `apps/learner` rồi deploy thẳng lên server (xem mục 10).

## 4. Cấu trúc code `apps/learner`

```
src/
  main.tsx
  app/
    router.tsx              định nghĩa route, lazy import, guard
    providers.tsx           QueryClient, i18n, Theme, Toaster
    AppShell.tsx            khung responsive (header, sidebar hoặc bottom nav)
    legacyRedirects.tsx     map URL Flutter cũ sang URL mới
  features/
    auth/                   LoginPage, ForgotPage, SetPasswordPage (activate|reset), authStore, refresh
    dashboard/              DashboardPage, StatCards, ProgramCard, LessonRail
    catalog/                CoursesPage, ProgramsPage, ProgramDetailPage, LessonListPage, FilterPanel
    lesson/
      LessonPage.tsx        chỉ ghép layout, không chứa logic
      player/
        VideoPlayer.tsx
        chooseSource.ts     chọn native HLS / hls.js / MP4 (hàm thuần, có unit test)
        useHls.ts           vòng đời hls.js, phục hồi lỗi theo tầng
        useStallDetector.ts
        usePlaybackGuard.ts tab ẩn / mất focus / khuất viewport
        SeekLimiter.ts      giới hạn tua
        PlayerControls.tsx
      pdf/PdfViewer.tsx
      progress/
        heartbeatQueue.ts   hàng đợi + retry + keepalive
        useLessonProgress.ts
      quiz/QuizPanel.tsx
      notes/NotesPanel.tsx
      completion/           dialog đánh giá / hoàn thành / kết quả quiz
    library/                LibraryPage, DocumentTable (desktop), DocumentCardList (mobile), PreviewDialog
    qa/                     QaHomePage, FaqPage, AskQuestionPage
    profile/                ProfilePage, CertificatesPage, CertificateViewer
    notifications/          NotificationsPage
    settings/               SettingsPage, ChangePasswordPage
    legal/                  PrivacyPage, ContactPage (nội dung lấy từ content/legal/*.md)
  shared/
    api/                    client.ts (fetch wrapper), generated/, queryKeys.ts
    ui/                     Button, Sheet, Dialog, ResponsiveDialog, Tabs, Badge, EmptyState, Skeleton...
    hooks/                  useBreakpoint, useMediaQuery, useInfiniteScroll
    lib/                    telemetry.ts, platform.ts (isAppleMobileWeb), format.ts
    i18n/                   vi.ts (chỉ tiếng Việt)
  content/legal/            disclaimer.md, send-question-consent.md, privacy.md
```

**Quy tắc:**
- Mỗi file tối đa khoảng 400 dòng. Page chỉ ghép component lại; logic nằm trong hook hoặc hàm thuần.
- Không có biến global mutable. Server state để trong TanStack Query, UI state để trong component hoặc URL.
- Dialog, toast và điều hướng chỉ được gọi ở tầng component, không gọi trong service hay hook dữ liệu.

## 5. Bản đồ route mới

| Route mới | Trang | Redirect từ URL cũ |
|---|---|---|
| `/login` | Đăng nhập (`?returnTo=`) | `/signIn` |
| `/forgot-password` | Quên mật khẩu | `/forgotPassword` |
| `/activate?token=` | Kích hoạt | **giữ** |
| `/account-recovery?token=` | Đặt lại mật khẩu | **giữ**; `/reset-password`, `/new-credentials` dùng cùng trang |
| `/` | Tổng quan | `/home`, `/init` |
| `/courses?program=&module=&q=&ta=&brand=&status=` | Khoá học (bộ lọc nằm trên URL) | `/homeCourse`, `/courseMain` |
| `/programs` · `/programs/:id` | Chương trình · Module của chương trình | `/allProgram` · `/allModule` |
| `/lessons/:list` (`saved\|upcoming\|newest\|mandatory\|optional`) | Danh sách bài | `/allLesson` |
| `/lesson/:id` | Học bài | **giữ** |
| `/library?type=&sort=&page=&q=` · `/library/:docId` | Thư viện · Xem trước (modal route) | `/homeLibary` |
| `/qa` · `/qa/faq` · `/qa/ask` | Hỏi đáp | `/homeQAHub`, `/qaHome` · `/qaHub` · `/sendQuestion` |
| `/profile` | Hồ sơ | `/homeProfile`, `/myProfile` |
| `/certificates` | Chứng chỉ (chỉ danh sách, giống bản hiện tại) | `/allCertificate` |
| `/notifications` | Thông báo | giữ |
| `/settings` · `/settings/password` | Cài đặt · Đổi mật khẩu | `/setting` · `/changePassword` |
| `/privacy` · `/contact` | Công khai | giữ |

- **Guard:** các route private dùng một layout `RequireAuth`. Chưa đăng nhập thì chuyển sang `/login?returnTo=<url>`. Thay cho biến static `courseIdWaitingLoad`.
- **Bảng redirect:** giữ trong `legacyRedirects.tsx` ít nhất 6 tháng. Có thể đặt thêm ở nginx dưới dạng `return 301`.

## 6. Tối ưu cho PC và mobile: một codebase, layout thích ứng

### 6.1 Nguyên tắc
1. **Mobile-first CSS**: viết cho màn hẹp trước, rồi mở rộng dần bằng `min-width`. Không rẽ nhánh theo "là web hay không".
2. **Thích ứng theo khả năng thiết bị**, không đoán theo tên máy:
   - `@media (hover: hover) and (pointer: fine)`: chỉ lúc này mới bật hover và tooltip.
   - `@media (pointer: coarse)`: vùng chạm tối thiểu 44×44 px.
   - Chỉ dùng user agent cho đúng một việc: **chọn cách phát HLS trên iOS** (mục 7).
3. **Chung component, khác cách bố trí**: dữ liệu và logic dùng chung, chỉ layout thay đổi. Không làm hai cây màn hình như `HomeWebPage` và `HomeScreen` hiện nay.
4. **Container query** (`@container`) cho các thẻ như `LessonCard` và `ProgramCard`, để thẻ tự co giãn theo khung chứa chứ không theo cả màn hình.

### 6.2 Breakpoint

| Tên | Min width | Thiết bị | Khung |
|---|---|---|---|
| `base` | 0 | Điện thoại dọc | Header gọn + **bottom nav 5 mục** |
| `sm` | 640 | Điện thoại ngang / tablet nhỏ | Như base, lưới 2 cột |
| `md` | 768 | iPad dọc | **Navigation rail** (thanh icon bên trái), không bottom nav |
| `lg` | 1024 | iPad ngang / laptop nhỏ | **Sidebar đầy đủ** (icon + chữ, thu gọn được) + header |
| `xl` | 1280 | Desktop | Sidebar + nội dung giới hạn `max-width` khoảng 1440 px |
| `2xl` | 1536 | Màn lớn | Tăng số cột, không kéo giãn chữ |

> Mốc 1100 px hiện tại được thay bằng `lg` (1024). Kéo cửa sổ qua ngưỡng chỉ làm thay đổi CSS, **không re-mount cây component**, nên video đang phát không bị mất.

### 6.3 Mẫu component thích ứng

| Thành phần | Mobile (< md) | PC (≥ lg) |
|---|---|---|
| Điều hướng chính | Bottom nav 5 mục, tôn trọng `safe-area-inset-bottom` | Sidebar trái, có trạng thái active và phím tắt |
| Dialog xác nhận / form | **Bottom sheet** (vuốt xuống để đóng) | Modal giữa màn hình |
| Bộ lọc | Nút "Lọc (n)" mở sheet toàn màn hình, có nút "Áp dụng" | **Cột lọc cố định bên trái**, áp dụng ngay khi chọn |
| Hàng bài học (rail) | Cuộn ngang bằng `scroll-snap`, vuốt được | Lưới 3–5 cột, có nút mũi tên khi cuộn ngang |
| Thư viện | Danh sách thẻ, **cuộn vô hạn** | **Bảng** (Tên / Loại / Brand / Cập nhật / Tải) sắp xếp được, phân trang có số |
| Thông báo | Trang riêng `/notifications` | Bấm chuông mở trang `/notifications` trong khung nội dung (giống bản hiện tại) |
| Hồ sơ nhanh | Tab Hồ sơ | Menu avatar ở header |
| Xem trước tài liệu | Toàn màn hình | Modal lớn, hoặc panel bên phải |
| Tooltip | Không có (dùng nhãn chữ) | Có, trên thiết bị có hover |

### 6.4 Màn học bài: màn quan trọng nhất

**PC (≥ lg):**
```
┌────────────────────────────────────────────┬──────────────────────┐
│  VIDEO / PDF (16:9, max-height 70vh)       │ [Tóm tắt][Quiz][Ghi chú]│
│                                            │  panel dạng tab,       │
├────────────────────────────────────────────┤  sticky, cuộn riêng    │
│  Tiêu đề · Lưu · Tải · Đánh giá · Metadata │                        │
│  Mô tả (Xem thêm)                          │                        │
└────────────────────────────────────────────┴──────────────────────┘
```
- Cột phải rộng 360–420 px, thu gọn được. Quiz nằm trong tab, nên khi đủ ngưỡng chỉ cần chuyển sang tab Quiz, không cần cuộn trang.
- **Phím tắt:**
  - Space/K: phát/dừng.
  - ←/→: lùi/tiến 5 s, vẫn bị giới hạn bởi seek limiter.
  - M: tắt tiếng. F: toàn màn hình. `,`/`.`: đổi tốc độ.
  - PDF: ←/→ để lật trang.
- Hover vào thanh tiến độ sẽ hiện thời điểm. Controls tự ẩn sau 3 s khi không động chuột.

**Mobile (< md):**
```
┌──────────────────────┐
│ VIDEO 16:9 (sticky)  │  ← dính trên cùng khi cuộn xuống
├──────────────────────┤
│ Tiêu đề · metadata   │
│ [Tóm tắt|Quiz|Ghi chú] tab ngang
│ nội dung tab         │
└──────────────────────┘
```
- Controls to (≥ 44 px), chạm một lần để hiện hoặc ẩn. Nhấn đúp nửa trái/phải để lùi/tiến 10 s (vẫn bị giới hạn tua).
- Nút toàn màn hình:
  - iPhone Safari không có `Element.requestFullscreen`, nên dùng `video.webkitEnterFullscreen()` (trình phát native của iOS).
  - Android/Chrome dùng `requestFullscreen`, sau đó `screen.orientation.lock('landscape')` nếu có.
- PDF trên mobile: chụm hai ngón để zoom, vuốt ngang để lật trang, hiện trang dạng "fit width".

### 6.5 Chi tiết riêng cho mobile web (iOS/Android)
- `<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">`. **Không** dùng `user-scalable=no`.
- Chiều cao dùng `100dvh` (không dùng `100vh`); padding dùng `env(safe-area-inset-*)`.
- Font-size của input tối thiểu **16px**, để iOS không tự zoom khi focus.
- `<video playsinline>` (thiếu thuộc tính này iPhone sẽ tự bật toàn màn hình).
- `touch-action: manipulation` cho nút, để bỏ độ trễ double-tap.
- Ảnh:
  - `loading="lazy"`, có `width/height` để tránh CLS.
  - `srcset` nếu backend có nhiều cỡ thumbnail; nếu chưa có thì đề xuất backend thêm preset.
- **Không khoá màn hình dọc.** Layout ngang trên điện thoại chỉ là `sm`.
- Tránh `position: fixed` bên trong phần tử cuộn trên iOS. Bottom nav đặt ở cấp `body`.

### 6.6 Chi tiết riêng cho PC
- Nội dung giới hạn `max-width`, chữ thân bài 15–16 px, độ dài dòng ≤ 80 ký tự cho phần mô tả và tóm tắt.
- Có trạng thái focus hiển thị (`:focus-visible`), điều hướng được bằng Tab, có "Skip to content".
- Link thật (`<a href>`) cho mọi thẻ bài, chương trình, tài liệu, nên chuột giữa hoặc Ctrl+click mở được tab mới.
- **Prefetch khi hover** (`queryClient.prefetchQuery` và prefetch chunk route) cho thẻ bài học, giúp mở bài gần như tức thì.
- Bảng Thư viện có header dính (sticky), sắp xếp bằng click, và nhớ bộ lọc trên URL.

## 7. Trình phát video: đặc tả bắt buộc

### 7.1 Chọn nguồn phát (`chooseSource.ts`, hàm thuần, phải có unit test)

| Điều kiện | Nguồn | Cách phát | Header |
|---|---|---|---|
| HLS + iOS/iPadOS WebKit (UA `iPhone\|iPad\|iPod`, hoặc `Macintosh` với `maxTouchPoints > 1`) + có `hlsPlaybackUrl` | `hlsPlaybackUrl` (`?t=`) | `<video src>` **native** | Không |
| HLS + trình duyệt có MSE (Chrome, Edge, Firefox, Android, Safari macOS) | `hlsUrl` (proxy) | **hls.js** | Bearer **chỉ gửi tới host API**; request tới storage (signed URL) không gửi header, `withCredentials=false` |
| HLS + native lỗi `MEDIA_ERR_SRC_NOT_SUPPORTED` | Fallback sang hls.js | | |
| MP4 | `fileUrl` | `<video src>` native | Không |

- **Lọc header theo host** (`new URL(url).origin === API_ORIGIN`), không lọc bằng cách tìm chuỗi `X-Amz-Signature` trong URL.
- **Token hết hạn giữa phiên:** hls.js lấy token qua callback `xhrSetup` mỗi lần gửi request, luôn đọc access token mới nhất sau khi refresh. Với native: lỗi 401/403, hoặc gần hết hạn 12 h, thì gọi lại `GET /progress/material/:id` để lấy `hlsPlaybackUrl` mới rồi gán lại `src`, **giữ nguyên `currentTime`**.
- hls.js được `import()` động, **chỉ tải khi thật sự cần**. iPhone/iPad sẽ không bao giờ tải file này.

### 7.2 Hành vi phát
- **Autoplay:** chỉ khi `navigator.userActivation.isActive`. Nếu gặp `NotAllowedError` thì chuyển sang trạng thái "Bấm để phát", không coi là lỗi.
- **Resume:** hiện dialog "Tiếp tục từ hh:mm:ss?". Seek **sau `canplay`**; không seek khi buffer còn rỗng.
- **Stall:** nếu vị trí phát không tiến quá 1.8 s trong khi đang phát thì coi là stall. Bỏ qua `waiting`/`isBuffering`.
- **Phục hồi theo tầng:**
  1. `hls.startLoad()`.
  2. `hls.recoverMediaError()`.
  3. `swapAudioCodec` + recover.
  4. Huỷ và tạo lại player tại vị trí cũ.
  5. Hiện nút "Tải lại video". Nút này **không** reload cả trang.
- **Tạm dừng:** khi tab ẩn, khi video khuất viewport (IntersectionObserver), hoặc khi mất focus quá 2 s. Chỉ tự phát lại nếu chính app là bên đã tạm dừng.
- **Giới hạn tua:**
  - Bài chưa hoàn thành: `maxSeekable = max(serverLastPosition, localFurthest)`.
  - Chặn qua cả thanh tiến độ, phím tắt và double-tap.
  - Lắng nghe sự kiện `seeking`: nếu vượt ngưỡng thì kéo lại `currentTime`, vì trên iOS native người dùng có thể tua bằng controls hệ thống.
- **Chất lượng:** menu Tự động/{h}p và badge băng thông chỉ hiện khi dùng hls.js. State gắn với từng instance player, không để global.
- **Khung hình:** container `aspect-ratio: 16/9`, `object-fit: contain`. Không dựa vào `videoWidth` của MSE.

### 7.3 Tiến độ (heartbeat)
```ts
// heartbeatQueue.ts (phác thảo)
type Beat = { materialId: string; positionSeconds: number; durationWatched: number; // thời gian xem thật
              at: number };  // chỉ gửi field API đang nhận; sessionId/seq để trong telemetry, không thêm vào body heartbeat
// - Đẩy mỗi 15–30 s khi đang phát, khi seek qua ngưỡng, khi ended, khi unmount.
// - Lỗi → giữ trong hàng (localStorage), retry backoff 2s→4s→8s…, gộp bản ghi cùng materialId.
// - pagehide / visibilitychange=hidden → fetch(url, { method:'POST', keepalive:true,
//     headers:{ Authorization, 'Content-Type':'application/json' }, body })
```
- Field gửi lên vẫn tên là `durationWatched` (giữ API), nhưng giá trị là **thời gian thực sự đã xem, cộng dồn** (tính theo `timeupdate` khi đang phát, trừ các lần tua). Tách riêng với `positionSeconds`.
- PDF gửi `timeSpentSeconds` thật (thời gian trang đó được hiển thị khi tab đang mở) và `totalPages` lấy từ pdf.js.
- **Client không tự quyết "hoàn thành"**: chỉ hiện theo `isCompleted`/`progressPercent` mà server trả về.

### 7.4 Yêu cầu tuân thủ: tiêu chí nghiệm thu (bắt buộc)

Phải giữ đầy đủ mọi hành vi của bản hiện tại, cộng thêm bước kiểm tra phía server để không lách được bằng DevTools.

| # | Hành vi | Tiêu chí đạt |
|---|---|---|
| C1 | Không tua vượt vị trí xa nhất đã xem khi bài chưa hoàn thành | Chặn trên thanh tiến độ, phím ←/→, double-tap mobile, **controls native iOS khi toàn màn hình** (`webkitEnterFullscreen`) và tua qua DevTools (`seeking` bị kéo lại). Tua lùi thì được. Bài đã hoàn thành thì tua tự do |
| C2 | Server không cho hoàn thành nếu xem không đủ thời gian | Bật lại `meetsDuration` và `meetsRealTime` (backend #2). Client gửi `durationWatched` thật. Test: gửi heartbeat `positionSeconds = duration` ngay sau khi mở bài thì `viewed` vẫn là `false` |
| C3 | Tốc độ phát | Giữ 1.0/1.3/1.5/1.8/2.0 như bản cũ. Không cho vượt 2.0 (chặn `playbackRate` bị đổi từ bên ngoài bằng listener `ratechange`) |
| C4 | Chuyển tab / ẩn trang khi chưa đủ ngưỡng | Tạm dừng ngay. Khi quay lại hiện dialog "Bạn vừa chuyển sang tab khác khi chưa hoàn thành bài học." kèm OK rồi mới phát tiếp. Có ghi `lifecycle_transition` |
| C5 | Mất focus ngắn (thông báo hệ thống, popup) | Debounce 2 s, sau đó tạm dừng. Nếu app là bên đã dừng thì tự phát lại |
| C6 | Đóng tab / reload | Prompt `beforeunload` khi chưa đủ ngưỡng, và **gửi heartbeat cuối bằng keepalive** |
| C7 | Back trình duyệt / back trong app / đổi mục menu | Hỏi xác nhận "Bạn chưa hoàn thành bài học..." (dùng `useBlocker` của React Router và `popstate`) |
| C8 | Video khuất viewport | Tạm dừng (IntersectionObserver) |
| C9 | Quiz bị khoá tới khi đủ `completionRulePercent` | UI khoá như bản cũ. **Server đã đạt**: `submitAttempt` không chặn nộp sớm, nhưng bài chỉ hoàn thành khi `canComplete` có `meetsProgress` (`progress.service.ts`), nên nộp sớm không làm bài hoàn thành. Không cần sửa |
| C10 | Tải tài liệu bài học chỉ khi đã hoàn thành | Ẩn nút ở UI như bản cũ. Server **chưa** kiểm tra (`content.service.ts` `getDownloadUrl`), bản hiện tại cũng vậy, nên theo quyết định #5 giữ nguyên. Nếu muốn chặt hơn: thêm một điều kiện `viewed` vào `getDownloadUrl` (khoảng 5 dòng) |

Các mục C1–C8 phải có E2E Playwright trên cả `chromium` và `webkit`, và nằm trong checklist thiết bị thật ở mục 12.

### 7.5 Telemetry
Giữ endpoint và tên event hiện có (`stall_detected`, `stall_recovered`, `video_error`, `heartbeat_failure`, `lifecycle_transition`) để các câu query `system_logs` cũ vẫn chạy.

**Mọi event bắt buộc có thêm:**
- `lessonId`, `playbackMode` (`native`/`hlsjs`/`mp4`), `level`, `bandwidthKbps`.
- `appVersion` (lấy từ git SHA lúc build), `route`.
- Với lỗi: `hlsErrorType`/`details`, `mediaErrorCode`.

**Event mới đề xuất:**
- `playback_start`, gồm `ttffMs` (thời gian tới frame đầu tiên).
- `source_selected`.
- `token_refreshed_midplay`.
- `web_vitals` (LCP/INP/CLS, lấy mẫu 10%).

Flush khi tab ẩn bằng `fetch keepalive`. Giữ lệnh `exportTelemetryLogs()` để hỗ trợ lấy log trên iPad.

## 8. Tài liệu (PDF/PPTX/ảnh/HTML)
- **PDF:** pdfjs-dist tự host (worker nằm cùng origin).
  - Chỉ render trang đang xem ±1 (lazy), trên canvas có `devicePixelRatio` giới hạn ở mức 2 để tránh tràn RAM trên iPad.
  - Lớp text layer bật được, nên người dùng chọn và copy chữ được.
- **PPTX:** giống prod, **hiển thị bản PDF đã convert** bằng chính PdfViewer. Nếu backend trả file chưa convert và pdf.js không mở được thì hiện "Không thể mở file xem trước" (chuỗi có sẵn trong app cũ). Không dùng Office Online.
- **Tải xuống:** dùng `<a href={downloadUrl} download>` trỏ thẳng tới presigned URL. Backend đặt `Content-Disposition: attachment; filename=...`. **Không** kéo cả file vào RAM.
- **HTML** (tóm tắt, câu trả lời FAQ): sanitize bằng **DOMPurify** trước khi render.

## 9. Auth, bảo mật, xử lý lỗi

### 9.1 Phiên đăng nhập
- **Giai đoạn 1** (không phải đổi backend):
  - `accessToken` chỉ giữ trong bộ nhớ (Zustand); `refreshToken` lưu localStorage.
  - Refresh theo kiểu **single-flight**: nhiều request cùng nhận 401 thì chỉ gọi refresh một lần, các request còn lại chờ kết quả đó.
  - Khi khởi động: gọi refresh để lấy access token rồi mới `GET /users/me`.
  - Đồng bộ đăng xuất giữa các tab qua `BroadcastChannel`.
- **Giai đoạn 2** (đổi backend, khuyến nghị): refresh token nằm trong cookie `HttpOnly; Secure; SameSite=Strict; Path=/api/auth`. Cách này an toàn vì web và API **cùng origin** (`bayer.doctotek.com/api`).
- **Không bao giờ lưu mật khẩu.** Đổi mật khẩu gửi `currentPassword` lên server (server đã có `bcrypt.compare`).
- Thống nhất quy tắc mật khẩu: **≥ 8 ký tự, có chữ và số**, dùng một schema zod chung, và nên đặt vào `@bayer/shared` để backend cũng dùng.
- **Đăng xuất:** xoá token, xoá cache Query, xoá telemetry local và hàng heartbeat của user đó.

### 9.2 Bảo mật frontend
- Không có secret nào trong bundle. Chỉ còn `VITE_API_BASE` (mặc định `/api`) và `VITE_APP_ENV`.
- CSP qua nginx: `default-src 'self'`; `media-src 'self' blob: https://<storage-host>`; `img-src 'self' data: https://<storage-host>`; `connect-src 'self' https://<storage-host>`; `frame-ancestors 'none'`.
- Toàn bộ thư viện bên thứ ba được đóng gói vào bundle, không lấy CDN lúc chạy. Font Inter tự host, subset `latin` + `vietnamese`, `font-display: swap`.

### 9.3 Xử lý lỗi
- `ApiError { status, code, message }`. Không đổi lỗi thành `null`.
- Timeout **20 s** cho API thường. Upload/tải file không đi qua lớp này.
- **Retry:** tối đa 2 lần, có backoff, **chỉ cho GET**.
- **UI:**
  - Lỗi theo trang: skeleton khi tải, EmptyState khi rỗng, ErrorState kèm nút "Thử lại" khi lỗi.
  - 5xx/502: banner "Hệ thống đang bảo trì". **Không** chuyển trang sang nơi khác.
- **Offline:** `navigator.onLine` kèm sự kiện `online`/`offline`, hiện banner. Hàng heartbeat tự gửi lại khi có mạng.

## 10. Hiệu năng, build, deploy

### 10.1 Ngân sách bundle
- **Shell** (router, query, UI cơ bản, i18n vi): ≤ 120 KB gzip.
- Mỗi route lazy: ≤ 60 KB gzip.
- **Tải riêng khi cần:** hls.js (khoảng 150 KB gzip), pdf.js (vài trăm KB kèm worker), DOMPurify.
- CI chạy `vite build`, kèm `rollup-plugin-visualizer` và một bước fail khi vượt ngân sách (size-limit).

### 10.2 Cache: sửa nginx (đang là nguyên nhân tải lại 11–13 MB)
```nginx
# Asset có hash trong tên (Vite: /assets/*.[hash].js|css|woff2)
location /assets/ {
    add_header Cache-Control "public, max-age=31536000, immutable";
    try_files $uri =404;               # thiếu file → 404, KHÔNG trả index.html
}
# Entry
location = /index.html { add_header Cache-Control "no-cache"; }
location / { try_files $uri /index.html; add_header Cache-Control "no-cache"; }
# Nén
gzip on; gzip_types text/css application/javascript application/json image/svg+xml;
# (bật brotli nếu image nginx có module)
```
- **Không dùng service worker** ở bản đầu, để tránh chuyện người dùng phải hard-refresh. Nếu sau này cần PWA hay offline, dùng `vite-plugin-pwa` với `registerType: 'autoUpdate'` và thêm banner "Có phiên bản mới".
- **Phát hiện phiên bản mới:** build ra `/version.json`. App kiểm tra khi tab được focus lại; nếu khác thì hiện "Tải lại để cập nhật". Khi load chunk thất bại (`ChunkLoadError`) thì tự reload một lần.
- **Version tự động** lấy từ `package.json` + git SHA. Bỏ hoàn toàn việc sửa tay 3 chỗ như hiện nay.

### 10.3 CI/CD
- **GitHub Actions** (dùng lại runner self-hosted `bayer-staging-vps` / `bayer-vps`):
  - `develop`: lint, typecheck, test, build `apps/learner`, rsync vào `/opt/bayer/flutter-web` trên staging. Nên đổi tên thư mục này.
  - `master`: tương tự cho prod (`/opt/bayer/web`).
- Giữ **2 bản build gần nhất** trên server (`releases/<sha>`, symlink `current`), rollback bằng cách đổi symlink.
- **Playwright chạy trên CI** với các project `chromium`, `webkit`, `Mobile Safari` (emulate iPhone 14), `iPad (gen 7)`.

## 11. Thay đổi backend cần đi kèm

| # | Thay đổi | Lý do | Ưu tiên |
|---|---|---|---|
| 1 | Thêm `hlsPlaybackUrl` (`?t=`) cho video trong **Thư viện** (`/documents/:id/hls/*`) | iPad đang phải đi hls.js khi xem trước. Bản hiện tại cũng chưa có, nên theo quyết định #5 bản đầu **giữ hls.js** cho xem trước | Tuỳ chọn (làm sau) |
| 2 | **Bật lại đoạn anti-cheat đang bị comment** trong `progress.service.ts` (`meetsDuration` dựa trên `durationWatched ≥ duration × MIN_VIDEO_WATCH_RATIO`, `meetsRealTime` dựa trên `firstActiveAt`). Client mới gửi `durationWatched` là **thời gian xem thật cộng dồn**, giữ nguyên tên field nên không phải đổi API | Tuân thủ bắt buộc (mục 7.4 C2). Đoạn này bị tắt vì app cũ gửi `durationWatched = position`. Đây là phương án rẻ nhất: bật lại code có sẵn, cập nhật test `progress.service.spec.ts` | **Bắt buộc trước phát hành** |
| 3 | Heartbeat chấp nhận request `keepalive` (không cần sửa code, chỉ cần body ≤ 64 KB) | Gửi được heartbeat cuối khi đóng tab | – |
| 4 | Tải xuống đặt `Content-Disposition` trên presigned URL (`ResponseContentDisposition`) | Tải trực tiếp, không kéo file vào RAM | TB |
| 5 | CORS của storage (MinIO) cho origin `bayer.doctotek.com` với các method GET/HEAD, header Range | hls.js, pdf.js đọc trực tiếp từ storage | TB (kiểm tra lại) |
| 6 | `unread-count` trả JSON `{count}` | Hiện client phải parse chuỗi | Thấp |
| 7 | Thumbnail có nhiều cỡ (ví dụ 320/640/1280) | `srcset` trên mobile | Thấp |
| 8 | Refresh token qua cookie HttpOnly (giai đoạn 2) | Bảo mật | TB |
| 9 | ~~Bật `GET /certificates/verify/:code`~~ | Ngoài phạm vi bản đầu (không thêm tính năng) | – |

## 12. Kiểm thử

| Tầng | Công cụ | Cần phủ |
|---|---|---|
| Unit | Vitest | `chooseSource` (bảng mục 7.1, gồm iPad giả UA Mac), `SeekLimiter`, `heartbeatQueue` (retry, keepalive, gộp bản ghi), stall detector, quiz (Set đáp án, single/multi), refresh single-flight, map redirect cũ sang mới |
| Component | Testing Library + MSW | Login/activate/reset, FilterPanel ↔ URL, QuizPanel, NotesPanel, LibraryTable/CardList |
| E2E | Playwright (chromium + webkit + mobile emulation) | Đăng nhập → dashboard → mở bài → xem đủ ngưỡng → quiz → hoàn thành → chứng chỉ; link sâu `/lesson/:id` khi chưa đăng nhập; tất cả URL cũ redirect đúng; back/forward/F5 ở mọi trang |
| Thiết bị thật (bắt buộc trước khi phát hành) | iPad Safari, iPad Edge/Chrome (vẫn là WebKit), iPhone Safari, Android Chrome, Windows Chrome/Edge, macOS Safari | Mở lần đầu, F5, quay lại từ bfcache, khoá màn hình, chuyển app, tua, đổi chất lượng, token hết hạn giữa lúc xem, đóng tab giữa chừng (kiểm tra tiến độ trên server) |
| Chỉ số sau phát hành | `system_logs` | So tỉ lệ `stall_detected`/phiên xem giữa iPad và máy khác, **chuẩn hoá theo số phiên**; so `ttffMs` |

Mục tiêu coverage ≥ 80% cho `features/lesson/**` và `shared/api/**`.

## 13. Lộ trình đề xuất

> Cherry là AI agent, nên cột "Ước lượng" tính theo **lượt làm việc của agent**, không tính theo tuần công của người. Thời gian thực tế bị chặn bởi các **cổng do người thực hiện**: review PR, test trên iPad/iPhone thật, deploy backend, pilot. Chi tiết từng task nằm ở [04-ke-hoach-thuc-thi.md](04-ke-hoach-thuc-thi.md).

| Giai đoạn | Nội dung | Kết quả | Ước lượng |
|---|---|---|---|
| **0. Nền móng** | Workspace `apps/learner`, Vite/TS/Tailwind/Router/Query, sinh client OpenAPI, token/theme, AppShell responsive (bottom nav / rail / sidebar), i18n, auth + refresh + guard, CI build + deploy staging, sửa cache nginx | Đăng nhập được trên staging, khung chạy tốt trên mọi breakpoint | 1–2 phiên agent · cổng: người đăng nhập thử trên staging |
| **1. Học bài (rủi ro cao nhất, làm sớm)** | Trình phát video (mục 7), PDF viewer, heartbeat queue, quiz, ghi chú, rating, dialog hoàn thành, **toàn bộ C1–C10 ở mục 7.4**, telemetry. Song song: backend #2 | `/lesson/:id` đầy đủ tính năng; E2E tuân thủ xanh trên chromium và webkit; qua checklist thiết bị thật | 3–5 phiên · cổng: **test iPad/iPhone thật** + backend #2 lên staging |
| **2. Duyệt nội dung** | Dashboard, Khoá học + bộ lọc trên URL, Chương trình/Module, danh sách bài, Thư viện (bảng/thẻ, xem trước, tải) | Đủ tính năng mục 1.3–1.7 tài liệu 02 | 2–3 phiên · cổng: QA so với tài liệu 02 |
| **3. Còn lại** | Q&A (FAQ, gửi câu hỏi), Hồ sơ, Chứng chỉ (danh sách), Thông báo, Cài đặt, trang pháp lý, redirect URL cũ | Tương đương đầy đủ tính năng | 1–2 phiên · cổng: QA |
| **4. Cứng hoá và chuyển đổi** | E2E, a11y audit, ngân sách hiệu năng, test thiết bị thật, beta cho nhóm pilot | Sẵn sàng thay bản Flutter | 1 phiên + **1–2 tuần pilot** |

Thời gian lịch dự kiến khoảng **3–5 tuần**, phần lớn là chờ test thiết bị, deploy backend và 1–2 tuần pilot. Đây là ước lượng thô. Backend #2 ở mục 11 **phải xong trước giai đoạn 4**. #2 là điều kiện bắt buộc để phát hành.

### 13.1 Chiến lược chuyển đổi
1. Chạy bản mới trên **staging** (`qa-bayer`) thay cho Flutter. QA đi hết checklist ở tài liệu 02.
2. **Beta trên prod, không đụng tới cấu hình đang phục vụ học viên:**
   - Dùng **domain phụ riêng** (ví dụ `beta-bayer.doctotek.com`, qua Cloudflare, có thể bật Cloudflare Access để chỉ nhóm pilot vào được). Nginx chỉ **thêm một server block mới** trỏ vào `/opt/bayer/learner-next`, `/api` proxy tới cùng container API.
   - **Không sửa** server block `bayer.doctotek.com`. **Không dùng** cách cookie `map $cookie_app`, vì cách đó phải sửa đúng block đang chạy prod.
   - Việc thêm block do **người** làm trong giờ ít truy cập: `nginx -t`, rồi `nginx -s reload` (không restart container), rồi kiểm tra ngay `bayer.doctotek.com` vẫn trả 200. Rollback: xoá block mới và reload.
   - Pilot dùng tài khoản học viên thật, có đồng ý tham gia, trong đó có người dùng iPad. Họ **tạo dữ liệu tiến độ thật**; đây là chủ đích của pilot.
   - Trong suốt pilot, **giữ `PROGRESS_ANTICHEAT` tắt**, vì cờ này tác động lên cả học viên đang dùng bản Flutter.
3. So telemetry giữa hai bản trong 1–2 tuần: stall/phiên, heartbeat_failure, ttff.
4. **Cắt chuyển (chỉ người làm, theo runbook T4.5 đã duyệt, trong giờ ít truy cập):**
   - Đổi `root` của `bayer.doctotek.com` sang bản mới **qua symlink** (`/opt/bayer/web` trỏ tới `releases/<sha>`), sau đó `nginx -t` và reload.
   - Giữ nguyên thư mục bản Flutter ít nhất 2 tuần. Rollback là đổi symlink rồi reload, dưới 1 phút.
   - Bật `PROGRESS_ANTICHEAT=1` **sau khi** đã chắc chắn không còn cần rollback về Flutter. Nếu phải rollback thì tắt cờ trước.
   - Theo dõi ngay sau khi cắt: tỉ lệ lỗi 4xx/5xx của `/api/progress/*` và `/api/auth/*`, số `heartbeat_failure` và `video_error` trong `system_logs`.
5. Sau đó xoá thư mục `web/` (bản build Flutter) khỏi repo `bayer-api`.

## 14. Câu hỏi

Đã chốt hết (xem mục 0). Chỗ nào chưa rõ thì áp dụng quyết định #5: làm giống bản hiện tại; nếu không làm giống được thì chọn phương án rẻ nhất và ghi lại trong PR.

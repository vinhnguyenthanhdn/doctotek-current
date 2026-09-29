# 04. Kế hoạch thực thi cho Cherry

> **Người đọc:** Cherry, AI coding agent sẽ viết web học viên mới.
> **Đọc trước:** [01](01-phan-tich-hien-trang.md) (hiện trạng), [02](02-spec-tinh-nang-va-api.md) (**tiêu chí nghiệm thu**), [03](03-huong-code-moi.md) (kiến trúc, mục 0 là các quyết định đã chốt, mục 7.4 là tuân thủ C1–C10).

## Luật làm việc

1. **Làm giống bản hiện tại.** Tham chiếu là code Flutter trên nhánh `origin/fix/ios-native-hls` của `bayer-doctotek-app` (đây là bản đang chạy prod), **không phải `main`**. Nếu không làm giống được, chọn **phương án rẻ nhất** rồi ghi một dòng "Lệch so với bản cũ: … vì …" trong mô tả PR. Luật này **không áp dụng cho C1–C10**: các tiêu chí này phải đạt đầy đủ.
2. **Không thêm tính năng mới.** Các API ở mục 2.1 tài liệu 02 chưa được dùng.
3. **Bug cũ:** chỉ sửa khi rẻ và không đổi hành vi người dùng thấy. Ví dụ: sai method `read-all`, xoá note bằng `DELETE`, quiz so khớp theo `Set`. Mỗi bug sửa đều ghi vào PR.
4. **Nhánh:**
   - Tạo `feat/learner-web` từ `develop` của `bayer-api`. Mỗi task (hoặc cụm task nhỏ) là một PR vào nhánh đó.
   - `develop` sẽ tự deploy staging, nên **chỉ merge vào `develop` khi người review đồng ý**.
   - Không push lên `master`.
5. **Mỗi PR phải có:** `npm run -w apps/learner typecheck && lint && test` xanh, và E2E liên quan xanh. Nếu có thay đổi UI thì kèm ảnh chụp ở 3 kích thước: 390×844, 820×1180, 1440×900.
6. **Việc Cherry không tự làm được**, cần báo người: test trên iPad/iPhone thật, sửa nginx và deploy prod, cấp tài khoản test staging, và merge các PR backend (`apps/api`).
7. **Bí mật:** không ghi token hay mật khẩu vào code hay PR. Tài khoản test lấy từ `.env-doctotek/` do người cung cấp qua biến môi trường của E2E.

## An toàn prod (BẮT BUỘC, ưu tiên cao hơn mọi mục khác)

Prod (`bayer.doctotek.com`, `internal-bayer.doctotek.com`, server `112.213.87.95`) **đang có học viên thật**. Tiến độ học và chứng chỉ là dữ liệu tuân thủ của Bayer. Cho tới lúc chuyển đổi được duyệt (T4.5), công việc này **không được làm thay đổi bất cứ thứ gì học viên prod nhìn thấy hoặc dữ liệu prod ghi nhận**.

**Cherry tuyệt đối không được:**
1. Push, merge hoặc mở PR vào `master` của `bayer-api`. Push vào `master` sẽ **tự deploy prod**. Không sửa `.github/workflows/deploy.yml` (workflow prod).
2. SSH, rsync hay chạy bất kỳ lệnh nào trên server prod `112.213.87.95`. Không sửa nginx, docker-compose hay thư mục `/opt/bayer/*` trên prod.
3. Ghi vào DB prod, kể cả `UPDATE` hay `DELETE` "để test". Nếu thật sự cần đọc log prod (`system_logs`), chỉ được dùng `SELECT` và phải báo người trước.
4. Chạy E2E, Playwright, script load hay test thủ công bằng tài khoản thật trên `bayer.doctotek.com`. Mỗi lần mở bài sẽ ghi heartbeat, lượt xem và tiến độ thật vào DB prod, làm sai báo cáo và chứng chỉ. **E2E chỉ chạy với MSW (local) hoặc staging `qa-bayer`**, bằng tài khoản test staging.
5. Gọi các API ghi dữ liệu trên prod: login thì được xem xét, nhưng heartbeat, submit quiz, rating, note, gửi câu hỏi, đọc thông báo thì không.
6. Để staging hoặc local trỏ vào API prod. `VITE_API_BASE` của mọi môi trường dev/CI là `/api` (proxy local) hoặc staging.

**Mọi thay đổi backend (`apps/api`) phải:**
- Mặc định **không đổi hành vi**. Ví dụ, anti-cheat ở T1.14 nằm sau cờ `PROGRESS_ANTICHEAT`, mặc định **tắt**. Như vậy dù code có vô tình lên prod thì học viên đang dùng Flutter vẫn không bị ảnh hưởng.
- Không có migration DB. Nếu buộc phải có, dừng lại và hỏi người.
- Tương thích ngược với app Flutter đang chạy: không đổi hay xoá field, route, method mà app Flutter đang gọi (xem 02 §2).

**Khi PR có dấu hiệu chạm prod** (file deploy prod, nginx prod, cờ mặc định bật, migration, đổi response API): ghi dòng đầu mô tả PR là `⚠️ CHẠM PROD: …`, và **không tự merge**.

Chỉ **người** mới được thao tác trên prod, và chỉ ở các bước T4.5 / G3, theo runbook đã duyệt.

## Môi trường làm việc (đã kiểm tra 28/09/2026)

| Môi trường | Tách khỏi prod? | Dùng cho | Lưu ý |
|---|---|---|---|
| **Local**: `docker/docker-compose.dev.yml` (postgres 16, redis, **mailhog**) + `apps/api` ở `:3000` + `apps/learner` (Vite, proxy `/api`) | **Có, hoàn toàn.** Email bị mailhog giữ lại, không gửi ra ngoài | Phát triển hằng ngày, unit test, component test, E2E với MSW | DB local **rỗng, chưa có seed**. Không có MinIO và media-processing, nên không phát được HLS thật ở local. Repo local `sub-project/bayer-api` đang ở `main` cũ, phải `git checkout develop` |
| **MSW** (mock trong trình duyệt/test) | Có | Phần lớn UI; giả lập mọi trạng thái (lỗi, rỗng, quiz đạt/trượt, token hết hạn) | Mock dựng theo response thật ở tài liệu 02 §2 |
| **Staging**: server `112.213.88.66`, API `internal-qa-bayer.doctotek.com/api`, web `qa-bayer` (bị Cloudflare Access chặn) | **Phần lớn có:** server, DB (`bayer_training_stg`), user DB và bucket (`bayer-uploads-stg`) đều riêng | Test tích hợp với API thật, HLS thật, E2E thật | Xem 4 rủi ro bên dưới |
| **Prod** | – | **Không dùng** | Xem mục "An toàn prod" |

**4 rủi ro của staging:**
1. **Staging gửi email thật** qua SMTP của Doctotek tới các địa chỉ lưu trong DB staging. Trong đó có 77 địa chỉ `@bayer.com`, có thể là hộp thư thật của nhân viên Bayer. Việc này **không ảnh hưởng prod**; rủi ro là gửi mail test tới người thật.
   - Phần lớn email do admin hoặc cron tạo ra, không liên quan tới web học viên.
   - Từ web học viên chỉ có **một luồng gửi mail cho người khác: "Gửi câu hỏi"** (`QA_NEW_QUESTION` gửi tới admin). Test luồng này bằng MSW; muốn test thật trên staging thì hỏi người trước.
   - Các luồng còn lại (quên mật khẩu, chứng chỉ) gửi về email của chính tài khoản test, nên an toàn khi dùng **tài khoản test do người cấp** với email mà team kiểm soát được.
2. **Media-processing dùng chung với prod** (`hls.doctotek.com`, tách theo `MEDIA_SERVICE_ENVIRONMENT=staging` và subscription `bayer-default-stg`). **Không upload hàng loạt video** lên staging; chỉ dùng các bài HLS đã có sẵn.
3. **Push vào `develop` là tự deploy staging**, trong khi team khác cũng đang QA trên đó. Bản web mới chỉ deploy vào thư mục riêng (T0.9), **không thay** Flutter trên `qa-bayer`.
4. **Staging chung với người khác:** không sửa hay xoá dữ liệu mà mình không tạo ra.

**Không dùng file `bayer_training_backup.sql`** (pg_dump khoảng 265 KB có sẵn trong lịch sử git, chứa 52 email `@bayer.com`) làm seed. Seed local phải là dữ liệu giả (T0.11).

## Cổng (người làm)

| Cổng | Khi nào | Người kiểm tra |
|---|---|---|
| G0 | Sau M0: đăng nhập được trên staging, khung chạy tốt ở 3 kích thước | Review nhanh |
| G1 | Sau M1: **checklist thiết bị thật** (03 §12) cho màn học bài, và C1–C8 trên iPad Safari | Người có iPad |
| G2 | Sau M2 và M3: đi hết checklist tài liệu 02 trên staging | QA |
| G3 | Sau M4: pilot 1–2 tuần trên prod, so telemetry | Người vận hành |

---

## M0. Nền móng

| # | Task | Xong khi |
|---|---|---|
| T0.1 | Tạo workspace `apps/learner` (Vite + React 19 + TS strict + Tailwind v4), script `dev/build/test/typecheck/lint`. Dev proxy `/api` sang `http://localhost:3000` | `npm run -w apps/learner dev` chạy; `build` ra `dist/` có asset hash |
| T0.2 | Sinh API client từ Swagger (`/api/docs-json`) vào `src/shared/api/generated`, thêm script `gen:api`. Wrapper `client.ts`: gắn Bearer, `ApiError`, timeout 20 s, retry GET 2 lần, refresh token single-flight | Unit test: 3 request cùng nhận 401 thì chỉ có 1 lần gọi `/auth/refresh`; refresh fail thì đăng xuất |
| T0.3 | Auth store: access token trong bộ nhớ, refresh token trong localStorage, **không lưu mật khẩu**. Khi khởi động: refresh rồi gọi `/users/me`. Đồng bộ đăng xuất giữa các tab qua `BroadcastChannel` | Test store; F5 vẫn giữ đăng nhập |
| T0.4 | Router: `RequireAuth` + `returnTo`; bảng redirect URL cũ (03 §5) | Unit test cho **từng** URL cũ; `/lesson/x` khi chưa đăng nhập đi qua `/login` rồi quay về `/lesson/x` |
| T0.5 | Design token (màu từ `lib/scr/theme/color.dart` bản cũ), font Inter tự host (subset vietnamese), các UI primitive (Button, Dialog/Sheet thích ứng, Tabs, Badge, EmptyState, Skeleton, Toast) | Storybook **không cần**; có trang `/__ui` chỉ bật ở môi trường dev |
| T0.6 | AppShell: bottom nav (< md), rail (md), sidebar + header (≥ lg), chuông kèm badge `unread-count`, menu avatar | E2E: 3 kích thước, đổi mục menu thì URL đổi; back/forward chạy đúng |
| T0.7 | Chuỗi hiển thị `shared/i18n/vi.ts`; nội dung pháp lý `content/legal/*.md`, **chép nguyên văn** từ các file Flutter liệt kê ở 02 §4 | So khớp văn bản bằng script diff (bỏ qua khoảng trắng) |
| T0.8 | Các màn Auth: login, forgot, activate, account-recovery/reset-password/new-credentials (02 §1.1). Validate bằng zod dùng chung | E2E với MSW; E2E thật trên staging |
| T0.9 | CI: job build/test `apps/learner` trong `.github/workflows/deploy-staging.yml`, rsync `dist/` vào thư mục **mới** trên staging (ví dụ `/opt/bayer/learner-next`). **Chưa** thay bản Flutter | Workflow xanh; file có trên server |
| T0.10 | Soạn diff nginx (03 §10.2) cho `qa-bayer` trỏ vào bản mới, kèm cách rollback. **Chỉ soạn**, người sẽ áp dụng | Có file diff trong PR |

| T0.11 | Script seed dữ liệu **giả** cho local: 1 learner, 1 chương trình, 2 module, bài video MP4, bài PDF, quiz single/multi, FAQ, thông báo. Email dạng `*@example.test` | `npm run -w apps/api seed:dev` tạo xong; không có dữ liệu thật |

→ **G0**

## M1. Màn học bài (rủi ro cao nhất)

| # | Task | Xong khi |
|---|---|---|
| T1.1 | `chooseSource.ts` theo bảng 03 §7.1, và `isAppleMobileWeb()` (bắt được iPad giả UA Mac) | Unit test đủ mọi hàng trong bảng, cộng 6 UA mẫu (iPhone, iPad, iPad-desktop-UA, Mac Safari, Android Chrome, Windows Edge) |
| T1.2 | `VideoPlayer`: native `<video playsinline>` và hls.js được `import()` động; Bearer **chỉ gửi tới origin API**; state chất lượng gắn với từng instance; khung 16:9 | Trên iPhone emulation không có request tải chunk hls.js; unit test `xhrSetup` không gửi header tới URL storage |
| T1.3 | Controls: play/pause, tốc độ (1.0/1.3/1.5/1.8/2.0), chất lượng (chỉ khi dùng hls.js) kèm badge băng thông, mute, fullscreen (`webkitEnterFullscreen` trên iPhone), phím tắt desktop, double-tap trên mobile | E2E chromium và webkit |
| T1.4 | Stall detector (vị trí không tiến quá 1.8 s), phục hồi theo tầng (03 §7.2), nút "Tải lại video" không reload cả trang, làm mới token/URL giữa phiên mà giữ `currentTime` | Unit test với fake timers; E2E dùng `page.route` để giả lỗi segment |
| T1.5 | `SeekLimiter` + chặn `seeking` + chặn `ratechange` > 2.0 (C1, C3) | Unit test; E2E: tua bằng thanh, bằng phím, và `video.currentTime = x` trong console đều bị kéo lại |
| T1.6 | `usePlaybackGuard`: tab ẩn, blur 2 s, IntersectionObserver, dialog chuyển tab, `beforeunload`, `useBlocker` cho back và đổi menu (C4–C8) | E2E cho từng hàng C4–C8 trên chromium và webkit |
| T1.7 | `heartbeatQueue`: nhịp 30 s như bản cũ; khi seek qua ngưỡng, khi ended, khi unmount; `pagehide`/hidden gửi bằng `fetch keepalive`; retry có backoff; gửi `durationWatched` là **thời gian xem thật cộng dồn**. PDF: gửi `pdf-page` với `timeSpentSeconds` thật và `totalPages` lấy từ pdf.js | Unit test (fake timers, offline rồi online); E2E đóng tab giữa chừng rồi `GET /progress/material/:id` thấy `lastPosition` đã cập nhật |
| T1.8 | Resume dialog (video hh:mm:ss / PDF trang N / "Học lại" khi đã hoàn thành); seek sau `canplay`; autoplay chỉ khi `userActivation` | E2E |
| T1.9 | PdfViewer (pdfjs-dist tự host): lật trang ngang, "NN / NN", đổi 16:9 ↔ 4:3, fullscreen, pinch trên mobile, phím ←/→ trên desktop; PPTX dùng cùng viewer | E2E với file PDF mẫu |
| T1.10 | Khối thông tin (Lưu, Tải khi đã hoàn thành, rating, metadata, mô tả thu gọn), Tóm tắt HTML (DOMPurify), Ghi chú (thêm/sửa/xoá bằng `DELETE`) | Component test với MSW |
| T1.11 | Quiz: khoá theo ngưỡng, làm từng câu, single/multi (dùng `Set`), xáo câu/đáp án, gửi, chuỗi dialog đạt/trượt, "Đi tới bài tiếp theo", "Xem chứng chỉ" (02 §1.6) | Unit test chọn đáp án (id "1" và "12" không lẫn nhau); E2E luồng đạt và trượt |
| T1.12 | Layout màn học: 2 cột (≥ lg) với panel tab Tóm tắt/Quiz/Ghi chú; mobile có video dính trên cùng và tab bên dưới | Ảnh chụp 3 kích thước |
| T1.13 | Telemetry: giữ tên event cũ, thêm `lessonId`, `playbackMode`, `level`, `appVersion` (git SHA), `ttffMs`; flush bằng keepalive; giữ lệnh `exportTelemetryLogs()` | Unit test logger; thấy event trong `system_logs` staging |
| T1.14 | **Backend PR riêng, vào `develop`:** bật lại `meetsDuration`/`meetsRealTime` trong `apps/api/src/progress/progress.service.ts`, **nằm sau cờ `PROGRESS_ANTICHEAT` (mặc định tắt)**, cập nhật `progress.service.spec.ts` | `npx jest --ci src/progress` xanh; có test **cả hai trạng thái**: cờ tắt thì hành vi giống hệt hiện tại; cờ bật thì heartbeat `position = duration` ngay sau khi mở bài cho ra `viewed = false` |

> **Lưu ý thứ tự triển khai T1.14:** bản Flutter vẫn gửi `durationWatched = position`. Nếu bật anti-cheat trên prod **trước khi** chuyển sang web mới, học viên dùng bản cũ có thể "hoàn thành" dễ hơn hoặc khó hơn (tuỳ ngưỡng `MIN_VIDEO_WATCH_RATIO`). Vì vậy chỉ bật cùng lúc với lúc chuyển sang bản mới: dùng một cờ env, ví dụ `PROGRESS_ANTICHEAT=1`.

→ **G1**

## M2. Duyệt nội dung

| # | Task | Xong khi |
|---|---|---|
| T2.1 | Dashboard (02 §1.3): 4 thẻ thống kê, chương trình, 3 hàng bài học, 2 banner, disclaimer, pull-to-refresh trên mobile | Component test; ảnh chụp 3 kích thước |
| T2.2 | `LessonCard`, `ProgramCard` dùng container query; prefetch khi hover trên desktop | |
| T2.3 | `/programs`, `/programs/:id`, `/lessons/:list` | E2E; F5 giữ nguyên trang |
| T2.4 | `/courses`: bộ lọc TA/brand/tuỳ chọn/nhãn nằm trên URL, chip, "Xoá bộ lọc", infinite scroll (limit 20 trên desktop, 10 trên mobile). Gửi `brandIds` giống bản cũ. **Không** thêm `taIds` (theo luật #1; bản cũ không gửi) | E2E: lọc, F5, back |
| T2.5 | `/library`: tab loại, sắp xếp, nhóm, bảng (≥ lg) hoặc thẻ (< lg), phân trang (10/5), xem trước (PDF/PPTX/video, video dùng hls.js như bản cũ), tải bằng `<a href download>` | E2E |

## M3. Còn lại

| # | Task | Xong khi |
|---|---|---|
| T3.1 | Q&A: trang chính, FAQ (TA rồi Brand, accordion, tải đính kèm), gửi câu hỏi (2 checkbox chép nguyên văn) | E2E |
| T3.2 | Hồ sơ, danh sách chứng chỉ, "Ghi chú của tôi" (sửa) | Component test |
| T3.3 | Thông báo: danh sách, đọc một, đọc tất cả (`POST /notifications/read-all`), badge | E2E |
| T3.4 | Cài đặt: đổi mật khẩu (server kiểm tra mật khẩu cũ; ≥ 8 ký tự, có chữ và số), Liên hệ (sheet), Chính sách, "Giới thiệu" (giữ dòng, không hành động), Đăng xuất | E2E |
| T3.5 | `/privacy`, `/contact` công khai | So văn bản |
| T3.6 | Trang 404 giống bản cũ ("Không tìm thấy trang..." kèm nút về trang chủ) | |

→ **G2**

## M4. Cứng hoá và chuyển đổi

| # | Task | Xong khi |
|---|---|---|
| T4.1 | Đối chiếu từng dòng trong 02 §1 với bản mới, xuất bảng ✅/❌ vào PR | Không còn ❌ |
| T4.2 | Kiểm tra ngân sách bundle bằng size-limit trong CI (03 §10.1); Lighthouse mobile ≥ 90 Performance cho `/` và `/lesson/:id` trên staging | Có số liệu trong PR |
| T4.3 | Audit a11y bằng axe (Playwright) cho các trang chính, không có lỗi mức serious/critical | |
| T4.4 | `version.json` và banner "Tải lại để cập nhật"; tự reload một lần khi gặp `ChunkLoadError` | E2E |
| T4.5 | **Chỉ soạn, không thực thi:** runbook chuyển đổi prod (03 §13.1 bước 2 và 4): block nginx cho domain beta, symlink release, lệnh rollback, thời điểm bật `PROGRESS_ANTICHEAT`, câu query `system_logs` để theo dõi | Người duyệt runbook và **tự thực hiện** |
| T4.6 | Sau khi chuyển đổi 2 tuần: PR xoá `bayer-api/web/` (bản build Flutter) | Người xác nhận không cần rollback |

→ **G3**

## Tham chiếu nhanh

| Cần | Ở đâu |
|---|---|
| Hành vi màn học bài bản cũ | `bayer-doctotek-app` (nhánh `origin/fix/ios-native-hls`) `lib/scr/view/course/course_view_screen.dart`, `course_bloc.dart`, `widgets/my_widgets/custom_cheview_controls.dart` |
| Logic HLS bản cũ | `third_party/video_player_web_hls/lib/src/video_player.dart` |
| Tiến độ, hoàn thành phía server | `bayer-api/apps/api/src/progress/progress.service.ts` |
| Route HLS có token | `bayer-api/apps/api/src/content/hls-playback.controller.ts`, `src/hls-token/` |
| Test và mock mẫu | `bayer-api/apps/web` (Vitest, MSW, Playwright) |
| Log lỗi client trên prod | Bảng `system_logs` (`context='client-event'`); cách query ở `AGENTS.md` gốc workspace |

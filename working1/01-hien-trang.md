# 01. Hiện trạng web DoctoTek (Flutter web)

> Khảo sát ngày 28/09/2026. Chỉ đọc code, không sửa gì.
> Nguồn:
> - `sub-project/edocjo-mobile` (nhánh `main`, v0.0.88+296), viết tắt là **M**.
> - `sub-project/edocjo-api` (nhánh `main`), viết tắt là **A**.
> - Cấu hình nginx trên server dev `112.213.88.192` (chỉ đọc).
>
> Đường dẫn trong tài liệu tính từ gốc repo tương ứng.

## 1. Tóm tắt

| Hạng mục | Hiện trạng |
|---|---|
| Sản phẩm | Nền tảng học tập y khoa cho nhân viên y tế: khoá học (VOD, livestream, CME), kênh học thuật (speaker), AI (EBM, soạn bài giảng, tóm tắt), tra cứu thuốc, bài viết, điểm thưởng, thanh toán khoá học |
| Client | **Một codebase Flutter** cho Android, iOS, Windows và **web**. 383 file, khoảng 85k dòng Dart, 75 route, `kIsWeb` xuất hiện 265 lần trong 100 file |
| State | flutter_bloc (15 bloc global tạo ở root, phụ thuộc lẫn nhau theo thứ tự), get_it, go_router |
| Backend | NestJS (khoảng 120k dòng TS, khoảng 45 module), Postgres, Redis/BullMQ, Elasticsearch, mediamtx, S3, GetStream (Video + Chat), Blaze (STT/dịch), Firebase Admin |
| Admin | Vue 3 + Element Plus + Vuetify (`A/doctorteck`, khoảng 100k dòng). **Không thuộc phạm vi** |
| Bundle web | `main.dart.js` **11.6 MB** + `canvaskit.wasm` 7.3 MB (chưa nén). Thư mục `web/` là **66 MB** (trong đó `assets/games` 13 MB) |
| Deploy web prod | Build Flutter **bằng tay** trên máy dev, commit vào `A/web/`, rồi `COPY web` vào image backend. Khi push `main` của `edocjo-api`, CI dùng `docker cp` chép vào `/var/www/web` (không xoá file cũ, không rollback web) |
| Deploy web dev | `M/.github/workflows/build-web-dev.yml` (chạy tay), label runner `dev-192`, `rsync --delete` vào `/var/www/web-dev`, serve ở `dev.doctotek.com`. Thực tế **không có runner nào tên `dev-192`**, nên workflow này không chạy được; đã **tắt** ngày 28/09. Bản đang chạy trên dev là Flutter 0.0.80+286 (đã backup ở `/var/www/web-dev.flutter-20260928`) |
| Test | Flutter: 7 file (chỉ caption/webvtt/socket). Backend: 98 spec nhưng CI **không chạy test** |

**Kết luận:** web đang chạy được nhưng gặp đúng những vấn đề cấu trúc của Flutter web:
- Tải nặng: khoảng 19 MB JS+WASM trước khi thấy khung hình đầu tiên.
- Không có DOM thật.
- Điều hướng desktop dùng chuỗi state thay cho URL.
- Nhiều hack platform view.

Ngoài ra, có **nhiều tính năng đang hỏng lặng lẽ** vì client và backend lệch nhau (mục 4).

## 2. Kiến trúc client

### 2.1 Điều hướng: hai hệ song song
- **Web rộng > 800px:** `HomeWebPage` (`M/lib/scr/web/home_web_page.dart`), gồm top bar 5 tab, sidebar Profile (280/260/76 px) và vùng giữa. Vùng giữa chọn widget theo chuỗi `WebPageBloc.mainWebPage` (khoảng 33 key: `course_view`, `course_payment_view`, `drug_search`, `point_history`…), **không đổi route**. URL được sửa tay bằng `history.replaceState`.
- **Web ≤ 800px và mobile:** `HomeScreen` với 5 tab (`LazyIndexedStack`), trang con đi bằng `router.push`.
- **Hệ quả:** back/forward của trình duyệt không lùi trong shell; F5 thường mất trang đang xem; nhiều trang không có URL riêng. Route truyền `extra` (List dynamic) nên không deep link được.
- Router **không có auth guard**; việc kiểm tra nằm ở `AuthenBloc.checkAuthen`. Hàm `redirect` gần như không làm gì, và với route có tham số còn trả về chuỗi mẫu (`app_routes.dart:804`).
- Breakpoint chính là `kIsWeb && width > 800`, kèm hàng loạt ngưỡng rải rác: 1200 (23 lần), 1500/1800, 1000, 950, 900, 700, 600, 2100.

### 2.2 API và phiên đăng nhập
- **Base URL:** lấy từ `--dart-define` (`M/lib/core/api/domain.dart`).
  - Prod mặc định: `https://backend.doctotek.com/api`.
  - Dev: `https://dev.doctotek.com/api` (cùng origin).
  - `WS_BASE_URL` khai báo nhưng không dùng; socket lấy origin của API.
- **Dio:**
  - Timeout **15 phút**; mọi status < 500 (trừ 401) được coi là thành công.
  - Refresh token có hàng đợi, nhưng crash khi lỗi mạng không có response (`api_helper.dart:284`).
  - `CancelToken` static: bị cancel khi mất mạng và không được tạo lại.
  - Lỗi 502 bật dialog bảo trì; bấm OK thì chuyển trang sang google.com.
- **Token:** access 15 phút, refresh 7 ngày (A/.env.example). Lưu **plaintext trong localStorage** (`shared_preferences`).
- **Đăng nhập:** email/mật khẩu (có OTP email khi đăng ký), Google (GSI trên web), Apple (JS SDK từ CDN của Apple). Facebook có package nhưng không dùng.
- Key AES hard-code (`M/lib/core/sercure/encryption_helper.dart:8`); gần như không dùng.
- Đổi mật khẩu so khớp mật khẩu cũ với `SessionModel.password` phía client. Trường này luôn rỗng, nên cần kiểm tra lại hành vi thật.

### 2.3 Realtime và media
| Mảng | Cách làm hiện tại |
|---|---|
| Socket | socket.io, transport websocket. Namespace `/ws/payments` (trạng thái thanh toán) và `/ws/livestream-captions` (chế độ test live-translate). Token nằm trong `auth`. Socket payment là **static**, giữ token cũ khi đổi tài khoản. Backend không có Redis adapter, nên chỉ chạy được một instance |
| Livestream | **GetStream Video (WebRTC SFU)**, xem với camera và mic tắt. Chat trong live dùng **GetStream Chat**, token lấy từ `GET /courses/:id/stream-token`. mediamtx chỉ nhận RTMP để lấy audio làm phụ đề |
| Phụ đề live | Server chạy pipeline: GetStream → RTMP → mediamtx → ffmpeg → Blaze → WebVTT. Client **poll `live.vtt` mỗi 750ms** và chọn cue theo đồng hồ server kèm playout delay. Chỉ có trên web |
| VOD | `video_player` + `chewie`, **không có hls.js**. HLS private (`/api/courses/:c/sessions/:s/hls-manifest`, cần Bearer) **không phát được** vì client không gắn header. Thực tế chỉ HLS public hoặc MP4 đã ký chạy được |
| Phụ đề VOD | `vod.vtt` được gắn thành `<track>` qua Blob; phải tìm `<video>` xuyên shadow DOM của Flutter và poll mỗi 250ms |
| Tiến độ | `POST /course-progress/event` (play/pause/seek/heartbeat mỗi 30s/stop). Backend bỏ các khoảng tăng quá 35s. **Đóng tab là mất event stop** (không có `pagehide`/keepalive) |
| Chặn tua | Chỉ ở UI, theo cờ `course.enableDragDropDuringPlayback`. Khi tắt thì chặn **cả tua tới lẫn tua lui**. Server không kiểm tra |
| Chặn chuyển tab | **Không có trên web** |
| Tài liệu | Tài liệu thuốc xem qua **Google Docs Viewer** (iframe, URL đã ký). Bài giảng thì tải về bằng Blob (mất đuôi file) |
| Upload | `POST /upload/media` multipart. Web đọc **cả file vào RAM**, kể cả video. Backend có presigned/resumable upload nhưng client không dùng |
| Push | FCM chỉ có trên mobile (theo topic). **Web không có push** |

### 2.4 Hạ tầng serve web
- **Prod** (xác minh từ bên ngoài ngày 28/09): **`doctotek.com` và `www.doctotek.com`**, qua Cloudflare proxy, serve bản Flutter 0.0.88+296 từ `/var/www/web` trên `.123`.
  - Mọi file đều `Cache-Control: no-cache, no-store`, nên mỗi lượt mở tải lại khoảng 19 MB.
  - `/.well-known/apple-app-site-association` và `assetlinks.json` là file tĩnh.
  - `web.doctotek.com` trả 520 (vhost chết). `sharing.doctotek.com` không có DNS.
  - Link chia sẻ, AASA (`paths: ["*"]`) và app link đều dùng `doctotek.com`.
- **Dev** (đã xác minh trên `.192`):
  - `dev.doctotek.com` có `root /var/www/web-dev`; `/api/`, `/ws/`, `/socket.io/` proxy sang `127.0.0.1:3001` (container `edocjo-backend-dev`, DB `edocjo_dev`, MinIO dev, **mailpit**, mediamtx dev).
  - `backend-dev.doctotek.com` proxy thẳng sang 3001.
  - Cùng máy còn chạy **`media-processing`** (`hls.doctotek.com`), **dùng cho prod** của nhiều khách.
- **CORS:** nginx prod gắn cứng `Access-Control-Allow-Origin: https://web.doctotek.com`, trong khi Nest cho phép regex `*.doctotek.com`. Hai lớp này có thể trùng hoặc xung đột header.

## 3. Nợ kỹ thuật và code chết
- **File lớn:** `custom_dialog.dart` 1740 dòng, `my_summary_view.dart` 1628, `course_view_screen.dart` 1419, `edit_profile_screen.dart` 1203, `profile_web_page.dart` 1056, `app_routes.dart` 920, `course_bloc.dart` 868.
- **Code chết hoặc demo:**
  - `lib/scr/demo_product` (6.9k dòng, demo cho Bayer/Urgo/Ankhang, vẫn route được qua `/corporate/*` và **tự chuyển tới đó khi workplace là `doctotek-*-vn`**).
  - `lib_game` cùng 13 MB asset (`/synbiotic`, public).
  - `/testUi`, lab live-translate (`main_live_translate_lab.dart`, `lib/scr/dev`).
  - `SocialScreen`/`MyPostsScreen` (không chỗ nào mở tới), `CourseWebPage`, `ResetPasswordScreen` (OTP), `FacebookLoginButton`, `BotLog` (no-op), 6 package không import.
- **Điều hướng và dialog** gọi qua `navigatorKey.currentContext!` toàn cục, kể cả trong lớp API.
- **Chuỗi hiển thị:** 972 dòng arb cho vi/en, nhưng locale cứng `vi`; header `accept-language` gửi `vn`.

## 4. Lệch client–backend: tính năng đang hỏng lặng lẽ

Backend bật `forbidNonWhitelisted: true`, nên body có field lạ sẽ bị **400**.

| # | Tính năng | Lỗi | Hệ quả |
|---|---|---|---|
| 1 | Tracking `POST /tracking/events` | Controller chỉ có ở nhánh backend `feature/tracking-user-event` (chưa merge) | 404, mất dữ liệu session_view/share |
| 2 | Sao theo session `/sessions/:id/ratings*` | Không có trên backend `main` | Mỗi lần mở session đều gọi và lỗi |
| 3 | Consent chương trình `POST /courses/learning-programs/:id/sponsor-consent` | Sai prefix; đúng là `/learning-programs/:id/sponsor-consent` | **Không vào được chương trình có consent** |
| 4 | Chia sẻ theo kênh `POST /referrals/links` | Gửi `sessionId`, DTO yêu cầu `courseId` | 400, **mọi nút chia sẻ FB/Zalo/Email đều lỗi** |
| 5 | Đăng ký email / Apple login khi có UTM | DTO không có `signup*` | 400 khi người dùng đến từ link có UTM |
| 6 | Bình luận đầu tiên | Gửi `rating: 0`, backend yêu cầu Min 1 | 400 |
| 7 | Đặt lại mật khẩu | Client cho 6 ký tự, backend cần 8 | Lỗi khó hiểu |
| 8 | Lượng giá và khảo sát | **Tráo dữ liệu** (`question_course.dart:40-61`) | Thẻ "lượng giá" hiện câu khảo sát và ngược lại |
| 9 | HLS private | Không gắn Bearer | Không phát được |
| 10 | Xoá tài khoản | `DeleteAccount` không có handler | Crash |
| 11 | Sửa hồ sơ | Ngày sinh bị ghi đè; workplace bị gửi rỗng | Mất dữ liệu mỗi lần lưu |
| 12 | Xoá lý lịch khoa học | Gọi nhầm API xoá chứng chỉ hành nghề; route xoá CV backend đã comment | Xoá nhầm dữ liệu |
| 13 | Link chia sẻ | `https://doctotek.com/<type>/<id>&ref_userid=` (thiếu `?`); `/post/:id/:ref_userid` hỏng; `SHARE_BASE_URL` không dùng | Link post hỏng |
| 14 | Thanh toán | QR là **asset tĩnh** (không dùng `order.qrCode`); tạo order lỗi thì loader quay mãi; trạng thái `waiting` hiện nhầm màn thành công | UX sai |
| 15 | Enroll | Client coi HTTP 400 là thành công; pay/cancel gửi courseId thay vì enrollmentId | |
| 16 | Thông báo | Chỉ lấy trang đầu, bấm vào không điều hướng | |
| 17 | Badge xác minh trên web | pending hiện "Đã xác minh" | Sai thông tin |

## 5. Rủi ro bảo mật backend phát hiện kèm (ngoài phạm vi viết lại web, cần báo người phụ trách)
- **Swagger public trên prod** (`/api/docs`); điều kiện tắt ở production đã bị comment (`A/src/main.ts:134-145`).
- **`POST /ai/proxy` có lỗ IDOR:** client tự gửi `speaker_id`/`conversation_id` và `query` được chuyển nguyên lên upstream, nên đọc hoặc xoá được hội thoại AI của người khác.
- **`/links` là bảng dùng chung**, không có userId: ai cũng xem và xoá được link của người khác.
- `PATCH /speakers/:id` không kiểm tra quyền sở hữu. Hai endpoint import `specializations` và `work-units` đang `@Public()`.
- Socket gateway để CORS `origin: '*'`.
- Job `deploy` của CI dùng `runs-on: self-hosted` **không có label**, nên có thể rơi vào runner `dev-192`.

## 6. Những gì nên giữ
- Phụ đề live theo đồng hồ server kèm playout delay (thuật toán `live_webvtt_client.dart`).
- Phụ đề VOD với reflow cue (56 ký tự, `line:88%`).
- Heartbeat tiến độ theo event (play/pause/seek/heartbeat/stop) kèm `userSessionId`.
- Luồng thanh toán QR kèm socket trạng thái; dùng lại order PENDING.
- Deep link lưu đích đến, đăng nhập xong mở lại; UTM và refCode.
- Guard kiểm tra bundle dev không chứa `backend.doctotek.com` (`build-web-dev.yml:56-65`).

# 02. Spec tính năng và API web DoctoTek (checklist nghiệm thu)

> Mỗi dòng ở mục 1 là một tính năng **web hiện tại đang có** (M, nhánh `main`). Bản mới phải có đủ, trừ code chết (mục 1.11) và những gì chỉ có trên mobile (mục 1.10). Cách làm các mục đặc biệt nằm ở mục 1.12.
> Tính năng đang hỏng vì lệch backend: xử lý theo mục 3.
> API: Swagger ở `/api/docs` (backend dev: `https://backend-dev.doctotek.com/api/docs`). Base prod là `https://backend.doctotek.com/api`; dev cùng origin `https://dev.doctotek.com/api`.

## 1. Tính năng theo màn

### 1.1 Công khai (chưa đăng nhập)
| Màn | URL hiện tại | Nội dung |
|---|---|---|
| Landing | `/about` (trang mặc định trên web khi chưa đăng nhập) | Hero, CTA store, khoá học ngắn, AI EBM, đối tác, đội ngũ, footer thông tin công ty (`footer_company_info`, `footer_copyright`) |
| Đăng nhập | `/signIn` | Email, mật khẩu (≥ 6 ký tự, ẩn/hiện); "Tạo tài khoản"; "Quên mật khẩu?"; Google (GSI) và Apple; link về `/about`. Desktop > 950px chia 2 cột |
| Đăng ký | `/signUp` (+ bottom sheet OTP) | Họ tên (≤ 27 ký tự), email, mật khẩu (≥ 6, có chữ và số), xác nhận mật khẩu, radio đồng ý Chính sách. Sau đó nhập OTP 6 số, gửi lại sau 60s |
| Quên / đặt lại mật khẩu | `/forgotPassword`, `/resetPassword?token=` | Link đặt lại gửi qua email (`FRONTEND_URL/resetPassword?token=`) |
| Pháp lý | `/privacy`, `/contact` | Văn bản dài (mục 4) |
| Game | `/synbiotic` | Giữ giống bản cũ (xem 1.12) |

### 1.2 Khung ứng dụng
- **5 tab:** Trang chủ · Khoá học · Kênh học thuật (Speaker) · AI (tester thì thấy hub ChatScreen) · Hồ sơ.
- **Desktop:** top bar kèm sidebar Profile thu gọn được. **Mobile:** bottom bar.
- Chuông thông báo kèm badge chưa đọc.
- Popup trang chủ (`/home-popups`) và popup AI (`/ai-popup`).
- Dialog "Mục tiêu học tập" tự bật khi user chưa có mục tiêu; có checkbox ẩn 3 ngày.
- **Hồ sơ bắt buộc lần đầu:** chưa có `profileInfo` thì tự chuyển sang EditProfile. Phải đồng ý `PrivacyPolicyModal` (chỉ bật được nút khi đã cuộn tới cuối) rồi mới lưu được.
- **Banner mở app** (`OpenInAppBanner`) cho in-app browser trên Android/iOS.

### 1.3 Trang chủ
- **Carousel banner:** web dùng `Ads`, mobile dùng `Ads Mobile`. `actionType` là course/speaker/post. Click ghi `POST /banners/:id/clicks`.
- **Chọn mục tiêu học tập (CTA)**, ô tìm khoá học.
- **Banner "Hỏi AI lâm sàng"**; tester thấy thêm banner "Tra cứu thuốc".
- **Các khối danh sách, mỗi khối có "Xem tất cả":**
  - Tiếp tục học (`/incompleteSessions`)
  - Hội thảo & CME sắp diễn ra (`/newCourses`)
  - Chương trình học mới nhất (`/learingPrograms`, có consent tài trợ)
  - Dành cho bạn (`/suggestCourses`)
- Danh sách "Xem nhiều" (`/mostPopularCourses`) có tải về nhưng không hiện ở trang chủ. **Làm giống bản cũ.**

### 1.4 Khoá học
**Tab Khoá học:**
- Banner.
- Tìm theo tên bệnh lý, sắp xếp Mới nhất / Xem nhiều nhất.
- Các section: Hội thảo & CME · Hiệp hội/Bệnh viện · Khoá học lâm sàng · Khoá học Doctotek (lọc speaker cố định `1d6b576e-…`) · Corporate Training Hub · Diễn tập (chỉ tester).

**Danh sách:**
- `/academicCourses`, `/organizationCourses` (lọc chuyên khoa và tổ chức), `/rehearsalCourses`, `/trainingHubs`, `/courseSearch`, `/coursePrograms`.
- Phân trang: 20 trên desktop, 10 trên mobile.

**Thẻ khoá học:**
- Thumbnail, badge TRỰC TIẾP / SẮP DIỄN RA / CME, tiêu đề, đơn vị tổ chức, ngày, giá hoặc Miễn phí.
- Nút Đăng ký / Xem khoá học / Xem lại / Xem video.
- Like/view (≥ 200), rating (≥ 3).

**Mở khoá học (`/course/:id`, desktop `course_view`):**
1. Nếu khoá cần consent và user chưa enroll: dialog consent HTML (chỉ tích được khi đã cuộn tới cuối), rồi enroll với `sponsorConsent: true`.
2. `FeatureGuard`: bắt buộc có hồ sơ (kiểm tra chứng chỉ đang tắt).
3. Khoá cần thanh toán (`needPayment()`) thì sang màn thanh toán. Có nhiều session thì hiện danh sách session. Session `live` thì sang màn livestream. Còn lại vào màn VOD.

**Màn VOD:**
- Player 16:9.
- Resume: "Bạn đã xem đến HH:MM:SS, tiếp tục xem?", có nút "Xem từ đầu".
- **Chặn tua theo `enableDragDropDuringPlayback`**: khi tắt thì chặn cả tới lẫn lui. Video xem thử thì tua tự do.
- CC xoay vòng: tắt / VI / EN+VI (web).
- Tiêu đề, ngày, like, lưu, sao, chia sẻ (Facebook/Zalo/Email/Khác), mô tả (Xem thêm), đơn vị tổ chức, tag.
- Diễn giả, có follow.
- **Nội dung khoá học:** session và bài giảng đính kèm, tải file.
- **Câu hỏi lượng giá:** khoá tới khi `completionRate ≥ evaluationThresholdPercent` (mặc định 40). Bấm lúc còn khoá thì toast "Cần xem hết X%".
- **Khảo sát:** không khoá.
- **Chứng nhận / CME:** luôn khoá, UI tĩnh.
- **Làm bài:** A/B/C…; `single_choice` chọn một, loại khác chọn nhiều. Gửi bulk xong hiện "Cảm ơn…". Không chấm điểm.
- **Đánh giá / nhận xét:** mỗi user một bản ghi; sửa, xoá (xoá = PATCH comment rỗng).
- **AI tóm tắt:** chỉ khi `enableAISummary && status=completed`. Desktop hiện panel bên phải. Lần đầu tự gửi HANDSHAKE, có bộ đếm "Chờ AI trả lời… N giây".
- **Session `scheduled`:** đếm ngược, "Hẹn Thông Báo" (nút rỗng), "Lưu Khóa Học".
- Khoá của tổ chức đặc biệt hoặc video-only: ẩn nội dung, quiz, khảo sát, chứng nhận.

**Màn livestream:**
- GetStream Video (xem, không camera/mic), số người xem, trạng thái backstage/ended.
- Nút bật âm thanh khi bị chặn autoplay.
- Phụ đề live 3 chế độ (chỉ web).
- Chat GetStream "Đặt câu hỏi"; tin của tester gắn `role=admin`.
- Lượng giá và khảo sát không khoá.

**Thanh toán khoá học (`/coursePayment`, `/paymentCourse`):**
- Video xem thử, chi tiết khoá, giá, "Đăng ký Ngay", "Lưu Khóa Học", "Khóa học này bao gồm:" (5 dòng).
- Tạo order bằng `POST /payments/orders` (luồng MANUAL, QR tĩnh), rồi hiện QR, tên TK / số TK / ngân hàng / nội dung (copy được), "Lưu mã QR".
- Socket `/ws/payments` báo trạng thái: approved → thành công ("Bắt đầu học ngay"); partial → lỗi "Thử lại"; rejected → "Đăng ký mới"; overpayment → thành công.

### 1.5 Kênh học thuật (Speaker)
- **Trang chủ Speaker:** banner, tìm kênh, 3 cột (chuyên gia `individual` / hiệp hội `brand` / tổ chức `organization`) kèm lọc chuyên khoa. Ô tìm rỗng thì gợi ý `top-speakers-by-courses`.
- **Trang kênh** (`/speakerProfile/:id`):
  - Ảnh, tên, chức danh, đơn vị, follower, Theo dõi.
  - 4 tab: Khoá học/Video · Bài viết · Tài liệu ("đang phát triển") · Kênh đã theo dõi ("đang phát triển").
  - Mã truy cập cho kênh video-only (Remote Config): hiện **chỉ kiểm tra đúng khi mở qua deep link**.
- **Kênh của tôi** (`/mySpeakerProfile`): nút "Đăng ký tạo khoá học" mở Google Form. Tạo/sửa kênh qua dialog: ảnh, chức danh, chuyên khoa, tiểu sử.
- **Bài viết của kênh** (`/speakerPost/:id`): HTML, danh sách "bài khác" sắp theo `shareCount`.
- **Kênh đang theo dõi** (`/speakerFollows`).

### 1.6 AI
- **Hub:** AI Soạn bài giảng (`Presentation_AI`), AI EBM (`Evidence_Based_Medicine_AI`). Tester có thêm bản "(Test)" với `trueExpert=1`.
- **Màn chat** (`/chatGenericAi`, `/chatPresentationAi`):
  - Drawer: chat mới, nâng cấp AI, thư viện, danh sách và tìm kiếm hội thoại, xoá.
  - Markdown kèm nút lựa chọn (`CHOICE`, `IMAGE_CHOICE`, `OVERVIEW`, `OPENER`), vote 5 mức, copy.
  - Đếm giây chờ; 403/429 hiện dialog nâng cấp và khảo sát; hướng dẫn lần đầu; disclaimer.
- **Hướng dẫn:** `/guidelineGenericAi`, `/guidelinePresentationAi`.
- **Gói AI:** `/planGenericAi`, `/planPresentationAi`. Không mua trực tuyến, chỉ có "Liên Hệ Ngay" và **nhập mã kích hoạt**. Lịch sử gói ở `/history*PlanAi`.
- **Thư viện link** (`/myLibary`): các link Google Slide do AI tạo; chia sẻ, xoá.

### 1.7 Tra cứu thuốc (chỉ tester thấy nút, nhưng URL `/drugSearch` ai cũng vào được: **giữ giống bản cũ**)
- **Tìm kiếm:** debounce 450ms, gợi ý biệt dược debounce 280ms. Facet biệt dược / nhà sản xuất; tab Tất cả và từng facet; danh sách 20/trang; "Xem thêm" facet.
- **Chi tiết** (`/drugDetail/:id`):
  - Thông tin (hàm lượng, dạng bào chế, đường dùng, NSX, ATC, phân nhóm), video, ảnh, tài liệu PDF, "Từ nhà tài trợ", Chia sẻ.
  - Tài liệu xem qua Google Docs Viewer; nếu lỗi thì refresh URL đã ký một lần.

### 1.8 Hồ sơ
- **Profile (sidebar desktop):**
  - Avatar, tên, "Chỉnh sửa thông tin", điểm.
  - Menu: Thông báo, Gói AI, Lịch sử điểm, Tra cứu thuốc (tester), Cài đặt.
  - Banner hồ sơ học thuật; kênh đang theo dõi (2); khoá đã lưu; khoá đã đăng ký.
- **ProfileWebPage (tab 4 trên desktop):**
  - Hero kèm badge duyệt.
  - 7 thẻ tổng quan (giờ học, chương trình, đang học, hoàn thành, CME, post-test, khảo sát), bấm mở panel `my_summary_view`.
  - "Khoá học của bạn" dạng tab, kênh theo dõi.
- **Xem hồ sơ** (`/myProfile`): nơi công tác, tỉnh, chuyên khoa, khoa/phòng, chức danh, năm tốt nghiệp, chứng chỉ hành nghề (trạng thái duyệt), lý lịch khoa học, Cá nhân/Tổ chức, SĐT, email, địa chỉ nhận CME, mục tiêu học tập.
- **Sửa hồ sơ** (`/editProfile`):
  - Avatar (base64, 1024px, chất lượng 60), họ tên*, SĐT*, giới tính, ngày sinh*, Cá nhân/Tổ chức, `medicalRole`, tỉnh* (autocomplete).
  - Nơi công tác* (theo tỉnh, có "Khác – nhập"), năm tốt nghiệp*, chuyên khoa* (chọn nhiều), khoa/phòng* (theo chuyên khoa đầu tiên), chứng chỉ hành nghề (1 file), lý lịch khoa học (≤ 5 file), địa chỉ CME.
  - Layout desktop 3 cột (≥ 1500) hoặc 2 cột (≥ 950).
  - Sau khi lưu: workplace `doctotek-bayer-vn`/urgo/ankhang thì chuyển sang `/corporate/*` (giữ, xem 1.12).
- **Chứng chỉ CME:**
  - Danh sách lọc theo nguồn/ngày.
  - Wizard 4 bước thêm chứng chỉ ngoài: file pdf/jpg/png ≤ 10MB, có checkbox cam kết.
  - Xoá.
- **Điểm:** `/pointHistory` (lịch sử), `/pointReward` (bảng cách nhận điểm theo nhóm).
- **Cài đặt:**
  - Đổi mục tiêu học, Đổi mật khẩu, Gói AI (ẩn trên web), Liên hệ, Chính sách, Giới thiệu, Đăng xuất, Xoá tài khoản.
  - Công tắc thông báo chỉ là state cục bộ.
- **Thông báo:** danh sách (trang đầu), đánh dấu đã đọc, "đánh dấu đọc tất cả" (có confirm), xoá từng cái.
- **Khoá học của tôi:** `/mySavedCourses`, `/myEnrollCourses`.

### 1.9 Bài viết (Social)
- **Đường vào thực tế:** chỉ `/postFuqllScreen` và link `/post/:id/:ref_userid` (đang hỏng). Feed `SocialScreen` và `/myPosts` không có chỗ mở tới.
- **Tạo/sửa bài:** tiêu đề, nội dung, nhiều ảnh, 1 video. Upload qua `/upload/media`.
- **Chi tiết bài:** bình luận phân trang 10, trả lời, like (cập nhật UI trước), chia sẻ.
- Tác giả bình luận luôn hiện "Ẩn danh". Sửa/xoá bài hoặc bình luận chỉ khi là chủ sở hữu.

### 1.10 Chỉ có trên mobile (không làm trên web)
Push FCM, local notification, Remote Config ép cập nhật app, pause/play khi app vào/ra background, mở file sau khi tải, chọn camera/thư viện, link preview trong bài viết, gói AI trong Cài đặt.

### 1.11 Code chết (không port)
`SocialScreen` feed, `MyPostsScreen`, `CourseWebPage`, `CoursesHomeScreen`, `ResetPasswordScreen` (OTP), `FacebookLoginButton`, `GuideUiBloc`, `/testUi`, `ApiDomain.share`/`chat`, `/protect/prompt`, `/me`, DevTools guard (đang tắt).

### 1.12 Demo corporate, game, lab (đã chốt: làm giống bản cũ)
| Mục | Bản cũ trên web | Cách làm ở web mới (rẻ nhất mà vẫn giống) |
|---|---|---|
| Demo corporate `/corporate/{bayer,urgo,ankhang}` + `/signIn` + `/home`, tự chuyển khi workplace là `doctotek-*-vn` | 3 bản copy gần như y hệt nhau (khoảng 6.9k dòng); state (ghi chú, Q&A, chứng chỉ demo) nằm trong RAM; đăng nhập thật | **Một** module `features/corporate` dùng chung cho cả 3 brand, chỉ khác logo/màu/text theo config. Giữ nguyên hành vi, kể cả state trong RAM |
| Game `/synbiotic` (public, 13 MB asset, `lib_game` khoảng 1.2k dòng Dart) | Nằm trong bundle Flutter chính | **Tách thành bản build Flutter web riêng chỉ chứa game** (entrypoint mới `lib/main_synbiotic.dart` trong `edocjo-mobile`, `flutter build web --base-href /synbiotic/`). Serve dạng thư mục tĩnh `/synbiotic/` trong web mới. Rẻ hơn viết lại game bằng JS |
| Lab live-translate | **Không có trong bản web đang chạy.** Chế độ test chỉ bật khi build với `LIVE_TRANSLATE_TEST=true`; lab là entrypoint riêng `main_live_translate_lab.dart` | Giữ đúng như vậy: **không đưa vào web mới**. Lab vẫn nằm trong repo Flutter như hiện tại |
| Chặn chuyển tab | Không có | Không làm |

## 2. API theo nhóm (đã đối chiếu backend `main`)

| Nhóm | Endpoint chính |
|---|---|
| Auth | `POST /auth/login`, `/auth/google-login` `{idToken, refCode?, signup*?}`, `/auth/apple-login` `{identityToken, fullName, refCode?}`, `/auth/initiate-register`, `/auth/verify-otp`, `/auth/forgot-password`, `/auth/reset-password` `{token,newPassword,confirmPassword}` (≥ 8), `/auth/refresh-token`, `/auth/change-password` |
| User / hồ sơ | `GET /users/profile`, `PATCH /users/:id/profile`, `POST /users/deactivate`, `POST/DELETE /users/professional-certificate`, `POST /users/scientific-profile`, `GET /specializations`, `/specializations/:id/categories`, `/specializations/:id/topics`, `/organizations`, `/work-units`, `/provinces` |
| CME | `GET/POST /certifications/cme`, `DELETE /certifications/cme/:id` |
| Trang chủ | `GET /banners`, `POST /banners/:id/clicks`, `GET /home-popups`, `GET /ai-popup`, `GET /learning-goals`, `GET /user-learning-goals/me`, `POST /user-learning-goals`, `GET /learning-programs?type=` |
| Khoá học | `GET /courses` (page, limit, type, status(es), isCME, isSaved, speakerIds[], organizationIds[], specializationIds[], topicIds[], search, sort), `/courses/enrolled`, `/courses/recommend`, `/courses/:id`, `POST/DELETE /courses/:id/like`, `/courses/:id/save`, `GET /courses/:id/sessions`, `GET /courses/:id/stream-token`, `GET /courses/:cid/sessions/:sid/hls-manifest` (Bearer) |
| Tiến độ | `GET /course-progress/courses/:cid/sessions/:sid/completion-rate`, `POST /course-progress/event` `{eventId, lessonId, userSessionId, eventType, position, timestamp}`, `GET /course-progress/incomplete-sessions`, `/study-hours/*`, `/in-progress`, `/completed-courses` |
| Lượng giá | `GET /sessions/:sid/questions?category=evaluation\|survey`, `POST /sessions/:sid/questions/responses/bulk`, `GET /assessments/quiz-results`, `/survey-results` |
| Enroll / consent | `POST /enrollments/:courseId/register {sponsorConsent}`, **`POST /learning-programs/:id/sponsor-consent`** |
| Đánh giá | `GET/POST/PATCH /courses/:id/ratings[/:rid]` (rating ≥ 1) |
| Phụ đề | `GET /livestream-captions/sessions/:sid/live.vtt\|vod.vtt?language=bilingual` (Bearer) |
| Thanh toán | `POST /payments/orders {idempotencyKey, productType:'course', productName, amount, productId}`, socket `/ws/payments` |
| Gói AI | `GET /subscriptions/ai/:aiName/plans`, `/subscriptions/active`, `/history`, `/ai/:aiName/history`, `POST /subscriptions/redeem {code}`, `GET /ai/usage/summary/:aiName` |
| AI | `POST /ai/proxy {method, query, payload, sourceType, trueExpert}` (20 lần/phút) |
| Speaker | `GET /speakers`, `/speakers/:id`, `/speakers/my-profile`, `/speakers/current/follows`, `POST /speakers`, `PATCH /speakers/:id`, `POST /speakers/:id/follow` (toggle), `GET /courses/analytics/top-speakers-by-courses` |
| Bài viết | `POST /upload/media`, `GET/POST /posts`, `GET/PATCH/DELETE /posts/:id`, `GET /posts/my-posts`, `GET/POST /comments/posts/:postId`, `PATCH/DELETE /comments/:id`, `GET /comments/:id/replies`, `POST /reactions`, `DELETE /reactions/:postId/:type` |
| Thuốc | `GET /drugs/search`, `/drugs/search/drugs`, `/drugs/:id`, `/brands` |
| Điểm | `GET /points/me`, `/points/history`, `/points/earn-catalog` |
| Thông báo | `GET /notifications`, `/notifications/unread-count`, `PATCH /notifications/:id/mark-read`, `/notifications/mark-all-read`, `DELETE /notifications/:id` |
| Chia sẻ | `POST /referrals/links {courseId}`, `POST /referrals/track/:refCode` (public), `GET /referrals/links/:refCode/course` |
| Thư viện link | `GET/POST /links`, `DELETE /links/:id` |
| Realtime | Socket.io namespace `/ws/payments`, `/ws/livestream-captions`; GetStream Video/Chat (API key theo môi trường) |

## 3. Chính sách với tính năng đang hỏng (theo luật "làm giống, không được thì rẻ nhất")

| # (01 §4) | Xử lý ở web mới | Lý do |
|---|---|---|
| 1 Tracking events | **Không gọi** cho tới khi backend merge `feature/tracking-user-event` | Endpoint không có trên `main`; gọi chỉ ra 404 |
| 2 Sao theo session | **Không hiện** | Như trên |
| 3 Consent chương trình | **Gọi đúng path** `/learning-programs/:id/sponsor-consent` | Endpoint đúng đã có sẵn; sửa rẻ |
| 4 Referral link | Gửi `courseId` | Rẻ; nút chia sẻ theo kênh chạy lại |
| 5 UTM khi đăng ký email/Apple | Gửi UTM chỉ khi DTO nhận; nếu không thì bỏ field | Tránh 400, không đổi backend |
| 6 Bình luận `rating: 0` | Không gửi `rating` khi chưa chọn sao | Tránh 400 |
| 7 Mật khẩu | Validate ≥ 8 ở màn đặt lại mật khẩu (giống backend) | |
| 8 Tráo lượng giá/khảo sát | **Sửa** | Bug rõ ràng, sửa rẻ |
| 9 HLS private | **Phát được bằng hls.js** kèm Bearer (xem 03 §7) | Bản cũ không phát được; hls.js là cách rẻ nhất |
| 10–12 Hồ sơ | Sửa (xoá tài khoản chạy được, không ghi đè ngày sinh/workplace, xoá CV không xoá nhầm; nếu backend không có route xoá CV thì ẩn nút) | Tránh mất dữ liệu người dùng |
| 13 Link chia sẻ | Sinh `https://doctotek.com/<type>/<id>?ref_userid=` | Rẻ |
| 14–15 Thanh toán, enroll | Hiện QR từ `order.qrCode` nếu có, nếu không thì dùng QR tĩnh như cũ; xử lý lỗi tạo order; không coi 400 là thành công | |
| 16–17 | Giống bản cũ, riêng badge xác minh thì sửa cho đúng trạng thái | |

## 4. URL và nội dung phải giữ
- **URL đang được chia sẻ hoặc gửi qua email** (AASA `paths: ["*"]`, app link `doctotek.com`): `/course/:id`, `/speakerProfile/:id`, `/speakerPost/:id`, `/drugDetail/:id`, `/post/:id/:ref_userid`, `/register?refId=&utm_*`, `/resetPassword?token=`, `/about`, `/privacy`, `/contact`, `/signIn`, `/signUp`.
- **Văn bản chép nguyên:**
  - `M/lib/scr/view/profile/privacy_policy_screen.dart:97-638`, sửa các lỗi chính tả và lỗi ghép chuỗi đã liệt kê trong báo cáo khảo sát: dòng 216-217, 561, 570-574, 468, 186.
  - Consent HTML do backend trả.
  - Các chuỗi l10n `footer_*`, disclaimer AI, các notice CME.

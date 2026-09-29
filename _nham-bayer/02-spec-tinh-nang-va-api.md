# 02. Spec tính năng và API (checklist port sang bản mới)

> Tài liệu này là **tiêu chí nghiệm thu** của bản mới. Đã chốt ngày 28/09/2026: **mọi tính năng của web hiện tại phải có trên web mới**, bao gồm chặn tua và chặn chuyển tab (tuân thủ bắt buộc). Chỉ được bỏ các mục ở 1.10: đó là code chết, không có trên UI.
> Nguồn: code Flutter trên nhánh `main` và `fix/ios-native-hls`, đối chiếu với controller NestJS trên `bayer-api` `origin/master`.
> Swagger đầy đủ: `https://internal-bayer.doctotek.com/api/docs`.

## 1. Tính năng theo màn hình

### 1.1 Xác thực (công khai)
| Màn | URL hiện tại | Chức năng |
|---|---|---|
| Đăng nhập | `/signIn` | Email và mật khẩu (tối thiểu 8 ký tự), nút ẩn/hiện mật khẩu. **Checkbox "Đồng ý với Điều khoản"** bắt buộc, link sang `/privacy`. Link "Quên mật khẩu?" |
| Quên mật khẩu | `/forgotPassword` | Nhập email, gọi `forgot-password`, hiện dialog "đã gửi email" |
| Đặt mật khẩu mới | `/reset-password`, `/account-recovery`, `/new-credentials` (`?token=`) | Mật khẩu mới và nhập lại (≥ 8 ký tự, có chữ và số), checkbox điều khoản. Thành công thì về trang đăng nhập |
| Kích hoạt tài khoản | `/activate?token=` | Giống màn trên nhưng gọi `activate`, xong **đăng nhập luôn** và vào home |

Khi chưa đăng nhập mà mở `/lesson/:id`: lưu đích đến lại, đăng nhập xong thì mở đúng bài đó.

### 1.2 Khung ứng dụng
- **5 mục chính:** Tổng quan · Khoá học · Thư viện tài liệu · Hỏi đáp · Hồ sơ.
- **Header:** logo, chuông thông báo kèm badge số chưa đọc, Cài đặt, avatar.
- **Sidebar hồ sơ** (desktop): avatar, tên, chức danh, Xem hồ sơ, Thông báo, Cài đặt, Đăng xuất. Thu gọn được.
- **Đang học mà rời bài** (đổi tab, back): hiện xác nhận "Bạn chưa hoàn thành bài học...".

### 1.3 Tổng quan (Dashboard)
- **4 thẻ thống kê:**
  - Tiến độ: `completionPercent`% và `completedLessons/mandatoryLessons`.
  - Số bài: Bắt buộc `mandatoryLessons/totalLessons` và Không bắt buộc `optionalLessons/totalLessons`. Bấm vào từng loại mở danh sách tương ứng.
  - Quiz đạt: `quizzesPassed/quizzesAssigned`.
  - Chứng chỉ: `certificatesEarned/lessonsWithCertificateEnabled`.
- **"Chương trình học của bạn":** hiện 1–2 thẻ, kèm "Xem tất cả". Mỗi thẻ có thumbnail, tên, nhãn trễ hạn, số khoá/video/tài liệu, module đầu tiên, % hoàn thành, nút "Bắt đầu học" (mở danh sách module).
- **3 hàng bài học, mỗi hàng có "Xem tất cả":** Sắp đến hạn · Mới đăng tải · Đã lưu.
- **2 banner:** FAQs sang Hỏi đáp; Thư viện tài liệu.
- **Disclaimer pháp lý Bayer ở cuối trang:** chép nguyên văn từ `home_dashboard_screen.dart`.
- Kéo để làm mới.

**Thẻ bài học** (dùng lại ở mọi nơi):
- Thumbnail kèm tag brand; tiêu đề; `TA1-TA2 / LEARNERLEVEL`.
- Badge loại (Video/PDF/PPTX); "Còn N phút"; Bắt buộc/Không bắt buộc; Hoàn thành/Chưa hoàn thành.
- "Hạn hoàn thành: dd/MM/yyyy" hoặc "Không thời hạn".
- Nút "Bắt đầu học".

### 1.4 Danh sách
| Màn | URL hiện tại | Dữ liệu |
|---|---|---|
| Tất cả chương trình | `/allProgram` | `dashboard/summary.myPrograms`, không phân trang |
| Module của chương trình | `/allModule` | `learner/courses?learningPathIds=[id]`, mỗi module kèm hàng bài của nó |
| Danh sách bài | `/allLesson` (tiêu đề truyền qua `extra`, F5 là mất) | Sắp đến hạn / Mới / Đã lưu lấy từ dữ liệu dashboard. Bắt buộc / Không bắt buộc lấy từ `learner/courses?isMandatory=` |

### 1.5 Khoá học (tab 1)
- **Bộ lọc:**
  - Tìm theo tên.
  - Dropdown Chương trình, rồi dropdown Khoá (module) phụ thuộc chương trình đã chọn.
  - Các nhóm lọc theo TA, mỗi nhóm có option là brand.
  - Nhóm "Tuỳ chọn": Bắt buộc / Không bắt buộc / Hoàn thành / Chưa hoàn thành / Sắp đến hạn / Đã quá hạn.
  - Nhóm "Nhãn tài liệu" theo `contentTypes`.
  - Chip hiển thị bộ lọc đang chọn, nút "Xoá bộ lọc", nút "Tất cả".
- **Kết quả:** danh sách module, mỗi module một hàng bài. Cuộn tới cuối thì tải thêm (`limit` 20 trên web, 10 trên mobile).
- **Bản mới cần sửa:** gửi đúng `taIds`; áp dụng lọc "Sắp đến hạn/Quá hạn"; đưa trạng thái lọc lên URL query.

### 1.6 Màn học bài: `/lesson/:id` (**giữ URL này, đang có trong email**)
**Luồng mở:**
1. `GET /progress/material/:id`. Nếu lỗi kèm `message` thì hiện dialog: đây là cách backend chặn bài.
2. `GET /quizzes/by-material/:id`.
3. `GET /progress/material/:id/notes`.
4. Link sâu thì gọi thêm `GET /content/materials/:id`.

**Nội dung:**
- **Video:**
  - Tỉ lệ 16:9.
  - Tốc độ 1.0/1.3/1.5/1.8/2.0.
  - Chất lượng Tự động hoặc chọn mức {h}p, kèm badge băng thông (chỉ khi dùng hls.js).
  - Tắt tiếng, toàn màn hình.
  - Lỗi: "Video chưa sẵn sàng" kèm nút "Tải lại".
- **PDF:** lật trang ngang, nút trước/sau, "NN / NN", đổi tỉ lệ 16:9 ↔ 4:3, chế độ toàn màn hình có điều hướng trang.
- **Panel "Tóm tắt nội dung"** (HTML `summary`): nằm bên phải trên desktop, ẩn/hiện được; nằm phía dưới trên mobile.

**Khối thông tin:**
- Tên bài, "(chương trình - khoá)".
- **Lưu bài** (`PATCH quick-view/:id`).
- **Tải tài liệu:** chỉ hiện khi đã hoàn thành.
- Ngày phát hành; trạng thái; "Approval number: {code} - EXP dd/MM/yyyy"; hạn hoàn thành.
- Đánh giá sao 1–5 (gửi ngay khi chọn), `avgRating`, lượt xem, lượt tải.
- Mô tả có "Xem thêm/Ẩn bớt".

**Bài kiểm tra:**
- Khoá cho tới khi xem đủ `completionRulePercent`%, hiện vòng tiến độ.
- Làm từng câu một: "Câu N", đánh dấu "(chọn nhiều)" nếu không phải `single_choice`, đáp án A/B/C.
- Nút Quay lại / Tiếp theo (phải chọn đáp án mới qua được) / Gửi.
- Client tự xáo câu hỏi nếu `shuffleQuestions`, xáo đáp án nếu `shuffleAnswers`.

**Ghi chú:** thêm (tối đa 1000 ký tự), sửa, xoá. Mỗi ghi chú hiện thời điểm tạo.

**Luật tiến độ và hoàn thành:**
- **Đủ ngưỡng:**
  - Video: `position/duration ≥ completionRulePercent/100`.
  - PDF: `trang/tổng trang ≥ completionRulePercent/100`.
  - Khi đủ thì mở khoá quiz và cuộn tới khối quiz.
- **Bài có quiz:**
  - Khi đủ ngưỡng: toast "Bài kiểm tra đã mở".
  - Nếu `quizRequired`: dialog "Làm ngay/Để sau". Bài chỉ hoàn thành khi qua quiz.
- **Bài không có quiz:** khi heartbeat trả `isCompleted`, hiện lần lượt dialog đánh giá rồi dialog "Bạn đã hoàn thành" (Rời bài / Ở lại).
- **Kết quả quiz:**
  - Đạt: dialog đánh giá, rồi "Chúc mừng", kèm "Đi tới bài tiếp theo" (`nextMaterialId`) và "Xem chứng chỉ" (`certificateId`).
  - Trượt: tỉ lệ đúng %, số câu đúng/sai, nút Đóng.
  - Kết quả **do server quyết định** (`passed`).
- **Chặn tua:** bài chưa hoàn thành thì không cho tua vượt vị trí xa nhất đã xem (max của `lastPosition` server và vị trí trong phiên). Tua lùi thì được.
- **Tiếp tục xem:**
  - Bài đã hoàn thành: "Học lại / Thoát".
  - Video: "Bạn đã xem đến HH:MM:SS, tiếp tục?".
  - PDF: "Bạn đã xem đến trang N".
- **Chống rời bài** (chỉ khi chưa đủ ngưỡng):
  - Chuyển tab: pause, quay lại thì hiện dialog cảnh báo.
  - Mất focus dưới 2s: bỏ qua.
  - `beforeunload`: hiện prompt của trình duyệt.
  - Back trình duyệt hoặc trong app: hỏi xác nhận.

### 1.7 Thư viện tài liệu (tab 2)
- **Bộ lọc:** giống Khoá học nhưng không có nhóm "Tuỳ chọn"; tìm theo tên tài liệu.
- **Tab loại:** Tất cả / Video / PDF / PPTX (`fileTypes`).
- **Sắp xếp:** Mới nhất / Cũ nhất / A–Z.
- **Hiển thị:**
  - Theo nhóm `contentType` (label, total); nếu không có nhóm thì danh sách phẳng.
  - Desktop dạng bảng: Tên / Loại / Thương hiệu / Cập nhật / Tải xuống.
- **Mỗi dòng:** thumbnail, tiêu đề, badge loại, brand (nếu không có thì chương trình, rồi module), lượt xem, lượt tải, `code - EXP`, nút xem trước, nút tải (theo `allowDownload`).
- **Phân trang:** có số trang, `limit` 10 trên desktop, 5 trên mobile.
- **Xem trước** (`documents/:id/download?type=preview`): mở toàn màn hình.
  - PDF và PPTX: hiện bản PDF đã convert.
  - Video: player có tua tự do.
- **Tải xuống** (`type=download`).

### 1.8 Hỏi đáp (tab 3)
- **Trang chính:** 2 thẻ "Câu hỏi thường gặp (FAQs)" và "Gửi câu hỏi của bạn".
- **FAQ:**
  - Tìm kiếm; chọn TA rồi mới chọn Brand thuộc TA đó.
  - Accordion theo nhóm (kèm số câu hỏi), cộng mục "Câu hỏi chưa phân nhóm".
  - Mỗi câu: câu trả lời dạng HTML, file đính kèm (tải được), `code - EXP`.
- **Gửi câu hỏi:**
  - Chọn TA, Brand; nhập nội dung câu hỏi.
  - **2 checkbox cam kết bắt buộc**, chép nguyên văn từ `send_question_screen.dart`. Nội dung gồm các email drugsafety, quality và medinfo của Bayer VN.
  - Sửa lỗi nhỏ (rẻ, không phải tính năng mới): Brand lọc theo TA đã chọn; không cho gửi nếu nội dung rỗng.

### 1.9 Hồ sơ, chứng chỉ, thông báo, cài đặt
- **Hồ sơ:**
  - Avatar, họ tên, chức danh, Line manager, Phòng ban, Alias/CWID, Email, Ngày gia nhập. Chỉ xem, không sửa.
  - Một số chứng chỉ kèm "Xem tất cả".
  - Bảng "Ghi chú của tôi": nội dung, ngày tạo, sửa.
- **Chứng chỉ:**
  - Lưới thẻ: TA, "CHỨNG CHỈ {TA}", chương trình/module/bài, ngày hoàn thành.
  - Giống bản hiện tại: chỉ danh sách. Xem/tải PDF để dành cho sau.
- **Thông báo:**
  - Danh sách: chấm trạng thái, nội dung, thời gian.
  - Bấm vào thì đánh dấu đã đọc; "Đánh dấu đã đọc tất cả".
  - Hiện chưa phân trang và chưa polling.
- **Cài đặt:**
  - Đổi mật khẩu: mật khẩu cũ, mới, xác nhận.
  - Liên hệ: `contact@doctotek.com`, số điện thoại 0986512367 (desktop copy vào clipboard), thông tin công ty.
  - Chính sách bảo mật.
  - Đăng xuất (có xác nhận).
- **Trang tĩnh công khai:**
  - `/privacy`: văn bản dài, chép nguyên văn từ `privacy_policy_screen.dart`.
  - `/contact`.

### 1.10 Không port (code chết, người dùng không thấy)
guide_ui, AI chat/plan/GetStream, notification local, `CourseModel`/`/course/learner/courses`, `PATCH /users/profile/update` (không có UI), `SessionResumeStore`.

> Mục menu "Giới thiệu" trong Cài đặt: **giữ nguyên như hiện tại** (hiện dòng chữ kèm mũi tên, bấm vào không làm gì).

---

## 2. API học viên đang dùng

Base: `/api`, cùng origin với web (nginx proxy), nên **không cần CORS**. Header gửi kèm: `Authorization: Bearer <accessToken>`.

| Nhóm | Method + path | Body / query | Dùng cho |
|---|---|---|---|
| Auth | `POST /auth/login` | `{email,password}` → `accessToken, refreshToken, user` | Đăng nhập |
| | `POST /auth/refresh` | `{refreshToken}` | Khi nhận 401 |
| | `POST /auth/forgot-password` | `{email}` | |
| | `POST /auth/reset-password` | `{token,password,confirmPassword}` | reset / account-recovery / new-credentials |
| | `POST /auth/activate` | `{token,password,confirmPassword}` → token | Kích hoạt |
| | `POST /auth/change-password` | `{currentPassword,newPassword,confirmNewPassword}` (server tự `bcrypt.compare`) | |
| User | `GET /users/me` | → ProfileModel | Khởi động app |
| Dashboard | `GET /dashboard/summary` | → `stats`, `myPrograms.items`, `newLessons.items`, `upcomingDeadlines.items` | |
| Catalog | `GET /learning-paths/learner/filters` | → `therapeuticAreas[{id,name,brands[{id,name,indications}]}]`, `contentTypes[{value,label}]` | Bộ lọc |
| | `GET /learning-paths/learner/list` | → chương trình | Dropdown chương trình |
| | `GET /learning-paths/learner/courses` | `page,limit,search,learningPathIds[],moduleIds[],brandIds[],contentTypes[],isMandatory,completionStatus` → `items[module{lessons[]}]` | Khoá học, module, bài bắt buộc |
| | `GET /learning-paths/learner/materials` | như trên | Fallback của Thư viện |
| | `GET /learning-paths/quick-view` · `PATCH /learning-paths/quick-view/:materialId` | | Bài đã lưu / bật tắt lưu |
| Bài học | `GET /content/materials/:id` | → LessonModel | Link sâu, bài tiếp theo |
| | `GET /progress/material/:id` | → `fileUrl, hlsUrl, hlsPlaybackUrl, fileType, lastPosition, completedPages, completionRulePercent, quizRequired, quizId, isCompleted, isQuickView, downloadable, avgRating, userRating, viewCount, downloadCount, code, summary...` | **Nguồn phát chuẩn** (không dùng `/content/materials` để lấy nguồn phát) |
| | `POST /progress/material/heartbeat` | `{materialId, positionSeconds, durationWatched}` → `progressPercent, isCompleted, quizRequired` | Video |
| | `POST /progress/material/pdf-page` | `{materialId,pageNumber,timeSpentSeconds,totalPages}` | PDF |
| | `POST /progress/material/:id/rating` | `{rating}` | |
| | `GET /progress/material/:id/notes` · `POST /progress/material/note` · `PATCH /progress/material/note/:noteId` · **`DELETE /progress/material/note/:noteId`** | | Ghi chú (app cũ gọi sai method khi xoá) |
| | `GET /progress/notes` | | Ghi chú của tôi |
| | `GET /content/materials/:id/download` | → `downloadUrl` | Tải tài liệu bài học |
| HLS | `GET /content/lessons/:materialId/hls/*` | Bearer **hoặc** `?t=` | Phát bài học |
| | `GET /documents/:id/hls/*` | **Chỉ Bearer** | Xem trước video trong Thư viện |
| Quiz | `GET /quizzes/by-material/:materialId` · `POST /quizzes/:id/submit` `{answers:{qId:[optId]}}` → `score, passed, certificateId, nextMaterialId` | | |
| Thư viện | `GET /documents/my/grouped` | `page,limit,sort,search,…,fileTypes[]` → `groups[]` hoặc `items,total` | |
| | `GET /documents/:id/download?type=preview\|download` | → `downloadUrl` | |
| Chứng chỉ | `GET /certificates/my` | → mảng | |
| Thông báo | `GET /notifications` · `GET /notifications/unread-count` · `PATCH /notifications/:id/read` · **`POST /notifications/read-all`** | | App cũ gọi `PATCH read-all`, sai method |
| Q&A | `GET /qa-hub/getQAFilters` `?search,taId,brandId` · `POST /qa-hub/questions` · `GET /qa-hub/attachments/:id/download` | | |
| Telemetry | `POST /log/client-events` | `{events:[...]}` | |

### 2.1 API backend đã có nhưng app chưa dùng (**ngoài phạm vi bản đầu**, để dành cho sau)
| Endpoint | Có thể dùng cho |
|---|---|
| `GET /certificates/:id`, `GET /certificates/:id/pdf` | Xem, tải chứng chỉ |
| `GET /qa-hub/my-questions` | "Câu hỏi tôi đã gửi" và trạng thái trả lời |
| `GET /notifications/bell-summary` | Popover chuông trên desktop |
| `GET /progress/course/:learningPathId`, `GET /progress/modules/:learningPathId`, `GET /progress/material/:id/status` | Tiến độ theo chương trình/module |
| `GET /learning-paths/:id/learner-detail`, `GET /learning-paths/my` | Trang chi tiết chương trình |
| `GET /profile/level`, `GET /profile/leaderboard/monthly` | Gamification (tuỳ chọn) |
| `GET /documents/my/:id` | Trang chi tiết tài liệu, deep link |

> Trước khi dùng các endpoint trên, cần xác nhận quyền và response thật trên Swagger hoặc staging.

## 3. URL phải giữ tương thích

| URL cũ | Lý do | Xử lý ở bản mới |
|---|---|---|
| `/lesson/:id` | Email nhắc học (`lesson/${materialId}`) | **Giữ nguyên** |
| `/activate?token=` | Email kích hoạt | **Giữ nguyên** |
| `/account-recovery?token=` | Email khôi phục | **Giữ nguyên** |
| `/reset-password?token=`, `/new-credentials?token=` | Link cũ | Giữ, hoặc redirect về một trang chung |
| `/home` | Email | Redirect sang `/` |
| `/signIn`, `/forgotPassword` | Bookmark | Redirect sang `/login`, `/forgot-password` |
| `/homeCourse`, `/homeLibary`, `/homeQAHub`, `/homeProfile`, `/courseMain`, `/allProgram`, `/allModule`, `/allLesson`, `/qaHome`, `/qaHub`, `/sendQuestion`, `/myProfile`, `/allCertificate`, `/setting`, `/changePassword` | Bookmark | Redirect 1-1 (xem bảng route ở tài liệu 03) |
| `/privacy`, `/contact`, `/notifications`, `/profile` | | Giữ nguyên |

## 4. Nội dung cần chép nguyên văn (pháp lý)
- Disclaimer Bayer ở cuối Dashboard: `lib/scr/view/home/home_dashboard_screen.dart`.
- Hai cam kết khi gửi câu hỏi: `lib/scr/view/qa_hub/send_question_screen.dart`.
- Chính sách bảo mật: `lib/scr/view/profile/privacy_policy_screen.dart`.
- Thông tin công ty: `lib/scr/widgets/bottom_sheet_option/contact_option.dart`, và trang Contact.
- Text checkbox "Đồng ý với Điều khoản" ở màn đăng nhập và đặt mật khẩu.

> Nên đưa các khối văn bản này vào file nội dung riêng (`content/legal/*.md`), để pháp chế sửa được mà không phải động vào code.

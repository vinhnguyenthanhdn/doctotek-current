# 01. Phân tích hiện trạng: Web học viên (Flutter web)

> Ngày khảo sát: 28/09/2026. Chỉ đọc code, không sửa gì.
> Nguồn:
> - `sub-project/bayer-doctotek-app`: nhánh `main` (0.0.47+113) và `origin/fix/ios-native-hls` (0.0.48+114).
> - `sub-project/bayer-api`: nhánh `origin/master` (prod).
> Đường dẫn file trong tài liệu tính từ gốc repo `bayer-doctotek-app`, trừ khi ghi rõ khác.

## 1. Tóm tắt

| Hạng mục | Hiện trạng |
|---|---|
| Sản phẩm | App học viên "Pharma Learning Hub" của Bayer, chạy ở `bayer.doctotek.com` |
| Công nghệ | Flutter 3.47, Dart ^3.11, render bằng **CanvasKit/WASM** (không có DOM thật) |
| Quy mô | 166 file Dart, khoảng 27.5k dòng trong `lib/`, cộng fork `third_party/video_player_web_hls` khoảng 1.1k dòng |
| State | flutter_bloc (5 bloc global sống suốt app), get_it, go_router |
| Kích thước tải lần đầu | `main.dart.js` 5.9 MB + `canvaskit.wasm` 5.4–7.3 MB, **chưa nén**. Cả thư mục `web/` là 45.7 MB |
| Cache | nginx prod đặt `no-store` cho **mọi** file `.js`/`.wasm`, nên **mỗi lần mở app đều tải lại 11–13 MB** |
| Test | 7 file, khoảng 48 case, chỉ phủ phần video và telemetry. Không có CI |
| Build/deploy | Build tay trên máy Windows cá nhân, copy tay sang `bayer-api/web/`, rồi commit bundle vào git |
| Bản prod | 0.0.48+114, build từ nhánh `fix/ios-native-hls`, **chưa merge vào `main`** |

**Kết luận:** mọi chức năng đều chạy được, nhưng nền tảng có 4 nhóm vấn đề mang tính cấu trúc. Chúng không sửa triệt để được nếu vẫn giữ Flutter web:
1. **Hiệu năng tải trên mobile:** 11–13 MB tải lại mỗi lần mở (do cấu hình nginx), cộng thời gian khởi động WASM.
2. **Video trên iOS/iPad:** Flutter phải nhúng `<video>` qua platform view và hack CSS. hls.js chạy qua ManagedMediaSource trên iPad làm khoảng 70% phiên xem bị stall. Bản vá native HLS hiện nằm ở nhánh riêng.
3. **Điều hướng web:** có hai hệ điều hướng song song. Desktop đổi trang bằng một chuỗi trong bloc, rồi sửa URL bằng `history.replaceState`. Vì vậy nút back/forward và F5 chạy không đúng, và nhiều trang không có URL riêng.
4. **Trải nghiệm web gốc:** không bôi đen hay copy chữ tự nhiên được, Ctrl+F không tìm được, accessibility kém, cuộn chuột phải hack, không có hover/focus/phím tắt theo chuẩn.

---

## 2. Kiến trúc hiện tại

### 2.1 Cấu trúc thư mục
```
lib/
  main.dart                 khởi tạo, đăng ký platform video HLS, MultiBlocProvider
  di/injection.dart         get_it: SessionModel (mutable, dùng chung), Dio, ApiHelper, repo, usecase, telemetry
  routes/app_routes.dart    go_router: route phẳng, redirect thực tế không làm gì
  core/
    api/                    domain.dart (base URL hard-code), endpoints.dart, api_helper.dart (Dio + refresh)
    data_local/             SharedPreferences (tức localStorage trên web)
    sercure/                AES với key hard-code (không dùng), key dịch vụ hard-code
    telemetry/              logger đẩy lô event lên /api/log/client-events
    providers/              danh sách bloc global
  scr/                      (tên gốc viết sai: "scr" thay vì "src")
    data/ domain/           lớp data/domain rất mỏng: usecase chỉ gọi thẳng repo
    view/                   authen, home, course, profile, qa_hub, guide_ui (chết)
    web/                    home_web_page.dart (khung desktop), web_page_bloc, web_support/js_web.dart
    widgets/                controls video, dialog, stall detector, lifecycle guard...
  l10n/                     vi/en, 219 key
  utils/
third_party/video_player_web_hls/   fork plugin video, hls.js lấy từ CDN
web/index.html                      nạp hls.js@1.6.16 (jsdelivr) và pdf.js 4.9.155 (cdnjs), cache-bust bằng ?v= sửa tay
```

### 2.2 Dependencies chính
- **Lõi:** flutter_bloc, get_it, go_router, dio, shared_preferences, equatable.
- **Video:** video_player, chewie, fork `video_player_web_hls`, video_player_web.
- **Tài liệu:** syncfusion_flutter_pdfviewer (trên web dựa vào pdf.js lấy từ CDN), flutter_html, photo_view.
- **UI:** google_fonts (tải font Inter lúc chạy), iconify_flutter, toastification.
- **Không dùng (0 import):** `provider`, `shimmer`, `carousel_slider`, `package_info_plus`, `device_info_plus`, `open_file`, `app_links`, `vector_math`, `collection`. `awesome_notifications` chỉ nằm trong một file chết.

### 2.3 Điều hướng: hai hệ song song
- **Màn ≤ 1100px hoặc iPad:** `HomeScreen`, gồm 5 tab bằng `LazyIndexedStack` và bottom bar. Trang con đi bằng `go_router push`.
- **Web > 1100px:** `HomeWebPage`, gồm top bar và sidebar. Nội dung chính được chọn theo chuỗi `WebPageBloc.mainWebPage` (`course_view`, `my_profile`, `all_lesson_saved`...), **không đi qua router**. URL chỉ được "vẽ lại" bằng `history.replaceState`.
- **Hệ quả:**
  - Back/forward của trình duyệt không khớp với trang đang hiện.
  - F5 thường mất trang đang xem.
  - Kéo cửa sổ qua ngưỡng 1100px sẽ đổi cả cây widget. Đã từng phải hotfix lỗi "resize giữa bài học nhảy về home".
  - Logic tab bị viết hai lần, ở `BottomMenubar` và ở `HomeWebPage`.
- Router **không có guard**. Việc kiểm tra đăng nhập nằm trong `AuthenBloc.checkAuthen`. Link sâu lúc chưa đăng nhập được giữ bằng biến static `courseIdWaitingLoad`.

### 2.4 State và luồng dữ liệu
- 5 bloc global: Authen, Home, Profile, Course, WebPage. Không bloc nào gắn với phạm vi màn hình.
- **Có 90 chỗ bloc, data hoặc core gọi thẳng UI** (`CustomDialog`, `ToastNotify`, `AppRoute.router`) thông qua `navigatorKey` global.
- Nhiều biến global mutable: `isLearningView`, `programSelected`, `selectedOptions`, `sortDocument`, `SessionModel`.
- Logic nghiệp vụ nằm trong widget. `course_view_screen.dart` dài 1565 dòng và gộp đủ thứ: player, heartbeat, retry, lifecycle, dựng URL HLS, gắn token. `document_library_screen.dart` dài 1482 dòng.
- Hậu quả thấy được trong git: các commit "restore code mất khi merge" (+411 dòng). Trong 60 commit gần nhất có khoảng 35 commit chỉ ghi "update code".

### 2.5 Lớp API (`lib/core/api/api_helper.dart`)
- Base URL **hard-code** `https://bayer.doctotek.com/api`. Không có env; muốn chuyển sang QA phải sửa code.
- Không có interceptor gắn token: từng hàm tự dựng header `Authorization`. `accept-language` gửi giá trị `'vn'` thay vì `'vi'`.
- Timeout **15 phút**. `withCredentials=true`. Mọi status khác 401 và nhỏ hơn 500 đều bị coi là thành công.
- Mọi lỗi Dio đều bị đổi thành `null`, nên UI không phân biệt được lỗi mạng, 4xx hay 5xx.
- Gặp 502: hiện dialog "Máy chủ đang bảo trì". Bấm OK thì trình duyệt **chuyển sang google.com**.
- Refresh token có hàng đợi retry, chạy được. Nếu refresh thất bại thì đăng xuất.
- Có một `CancelToken` static dùng chung cho toàn app.

### 2.6 Auth và bảo mật
| Vấn đề | Vị trí |
|---|---|
| **Lưu mật khẩu dạng rõ** trong localStorage (`flutter.USER_SESSION`) | `session_model.dart:7,35`, `authen_impl.dart:24` |
| Kiểm tra mật khẩu cũ **ở client** bằng mật khẩu đã lưu. Việc này thừa, vì backend đã tự `bcrypt.compare` | `change_password_screen.dart:173` |
| Key AES hard-code, IV rỗng (không dùng nhưng vẫn nằm trong bundle) | `lib/core/sercure/` |
| Key/ID dịch vụ bên thứ ba hard-code trong bundle JS | `lib/core/sercure/key_and_token.dart` |
| Kích hoạt tài khoản không lưu session, F5 là mất phiên | `authen_remote.dart:75-82` |
| Đăng xuất không gọi API, không xoá log telemetry và dữ liệu resume | `authen_bloc.dart:83-95` |
| Ngưỡng độ dài mật khẩu không thống nhất: đăng nhập/reset yêu cầu 8, đổi mật khẩu yêu cầu 6 | |

### 2.7 Responsive
- Breakpoint chính cứng ở `kIsWeb && width > 1100` (`helper.dart:19`). Ngoài ra còn rải rác các mốc 1000, 1200, 800, 1500, 1800 và `<=360`.
- Kích thước UI rẽ nhánh theo `kIsWeb` chứ không theo chiều rộng màn hình. Các màn có nhiều nhánh nhất: `home_dashboard_screen` (40), `my_profile_screen` (27), `course_view_screen` (26).
- Tablet 768–1100px dùng nguyên layout mobile. App khoá màn hình dọc.
- **Desktop chỉ là màn mobile đặt vào khung top bar + sidebar**: không có hover, focus, phím tắt hay bôi chọn chữ.

### 2.8 Build, cache, deploy
- Mỗi lần release phải sửa version bằng tay ở **3 chỗ**: `pubspec.yaml`, `web/index.html` (`flutter_bootstrap.js?v=`), `helper.dart` (`versionDoctotek`).
- Chỉ file bootstrap được bust cache; `main.dart.js` và canvaskit không có hash trong tên. Vì vậy nginx phải để `no-store` toàn bộ, dẫn tới việc tải lại 11–13 MB mỗi lần mở.
- `flutter_service_worker.js` vẫn được deploy, nên người dùng phải Ctrl+Shift+R sau mỗi lần phát hành.
- Build không tự copy `manifest.json` và icon, phải copy tay. Icon PWA chỉ có một file 23 KB dùng cho mọi kích thước.
- Thư viện nạp lúc chạy từ bên thứ ba, không có SRI: hls.js (jsdelivr), pdf.js (cdnjs), Google Fonts. Riêng nhánh pptx còn dùng Office viewer.

---

## 3. Video, tài liệu, tiến độ: phần khó nhất

### 3.1 Trình phát video
- **Kiến trúc:** `HlsDispatchVideoPlayerPlatform` chuyển URL `.m3u8` sang fork hls.js, còn MP4 dùng `video_player_web` mặc định.
- **Hack CSS:** phải tiêm CSS cho `flt-platform-view` (nếu không video MSE co lại còn khoảng 150px), ép `object-fit: fill` và tỉ lệ 16:9, vì fork báo sai tỉ lệ khung trên iPad.
- **Xác thực:** `xhrSetup` gắn Bearer cho mọi request, **trừ** URL có `X-Amz-Signature`. Lý do: signed URL MinIO sẽ trả 400 nếu request kèm header.
- **iOS, trên nhánh prod:** `isAppleMobileWeb()` bắt được cả iPad giả UA Mac. Khi đúng, app dùng `hlsPlaybackUrl?t=` và `<video>` native, không gửi header. Nếu native lỗi code 4 thì quay về hls.js.
- **Chất lượng HLS:** trạng thái nằm ở biến global `hlsLevelHeights`/`hlsCurrentLevel`, nên chỉ đúng khi có một player tại một thời điểm.
- **Stall:** phát hiện dựa trên vị trí phát (không tiến quá 1.8s), không tin cờ `isBuffering`. Phục hồi hls.js lần lượt `startLoad` rồi `recoverMediaError`. Nút "Tải lại" có lúc reload **cả trang**.
- **Chặn tua chỉ ở client:** không cho tua vượt vị trí xa nhất đã xem nếu bài chưa hoàn thành. Nhưng vẫn cho tốc độ 2x, và server không kiểm tra lại.
- **Autoplay:** chỉ phát khi `navigator.userActivation` đang bật. App phát `silence.mp3` để mở khoá âm thanh.
- **Resume:** hiện dialog "Bạn đã xem đến hh:mm:ss". Silent resume đã bị bỏ vì seek lúc buffer rỗng làm iOS kẹt cứng.

### 3.2 Tiến độ học (heartbeat)
| Loại | Endpoint | Lỗi hiện tại |
|---|---|---|
| Video | `POST /progress/material/heartbeat` `{materialId, positionSeconds, durationWatched}`, gửi mỗi 30s | `durationWatched` **bằng luôn vị trí**, không phải thời gian xem cộng dồn. `userSessionId` được sinh ra nhưng không gửi |
| PDF | `POST /progress/material/pdf-page` `{materialId, pageNumber, timeSpentSeconds, totalPages}` | `timeSpentSeconds` **bằng số trang**. Chỉ gửi khi đổi trang |
| Rời trang | `beforeunload`/`pagehide` | Callback **rỗng**: không gửi heartbeat cuối (không `sendBeacon` hay `keepalive`), có thể mất tới 30s tiến độ |
| Lỗi mạng | | Không retry, không xếp hàng; chỉ ghi telemetry |

### 3.3 Tài liệu
- **PDF:** Syncfusion viewer, trên web cần pdf.js từ CDN. Lỗi tải chỉ gọi logger no-op, người dùng không thấy thông báo.
- **PPTX: hai cách mâu thuẫn nhau.**
  - `main` coi PPTX như PDF, nhưng backend chỉ trả PDF khi đã convert xong (`conversionStatus==='done'`).
  - Nhánh `hotfix/pptx-preview` (chưa merge) dùng iframe Office Online và một ô màu đè lên để che menu tải xuống. Cách này đưa URL tài liệu ra Microsoft, cần pháp chế xem lại.
- **Tải xuống trên web:** app kéo **toàn bộ file vào RAM** qua Dio rồi tạo Blob, kể cả MP4 lớn.
- **Xem trước video trong Thư viện:** route `/documents/:id/hls/*` **chỉ nhận Bearer**, chưa có `?t=`, nên iPad vẫn đi đường hls.js và vẫn có nguy cơ stall.

### 3.4 Quiz, chứng chỉ, thông báo
- **Quiz:**
  - Hiện từng câu một. Kiểm tra đáp án đã chọn bằng `toString().contains(optionId)`, nên **id "1" sẽ khớp "12"**.
  - Khi trượt, số câu đúng được suy ngược từ điểm %.
  - Nộp lại không giới hạn.
- **Chứng chỉ:** chỉ liệt kê. Không xem hay tải PDF, không hiện mã xác thực, dù backend đã có `GET /certificates/:id/pdf`.
- **Thông báo:**
  - Chỉ tải khi vào home, không polling.
  - "Đánh dấu đã đọc tất cả" gọi **`PATCH`** `/notifications/read-all`, trong khi backend khai báo **`POST`**, nên nhiều khả năng đang lỗi.
  - `unread-count` được parse như chuỗi.
- **Ghi chú:** nút xoá gọi `PATCH` với body rỗng thay vì `DELETE`, mặc dù backend đã có `DELETE /progress/material/note/:noteId`.
- **Push/FCM:** không có.

### 3.5 Telemetry (giữ lại, đang có ích)
- Event: `stall_detected`, `stall_recovered`, `video_error`, `heartbeat_failure`, `lifecycle_transition`, gửi lên `POST /api/log/client-events` và lưu vào bảng `system_logs`.
- Cơ chế: đẩy theo lô mỗi 30s hoặc khi đủ 20 event; flush khi tab ẩn; lưu 1000 dòng vào localStorage; gõ `exportTelemetryLogs()` trong console để tải về.
- **Thiếu:**
  - `lessonId` trong event stall và error, nên phải join qua `sessionId`.
  - Chế độ phát (native hay hls.js), level chất lượng, chi tiết lỗi hls.js.
  - `appVersion` đang hard-code `'video-fix-2026-08-09'`.

---

## 4. Danh sách bug và nợ kỹ thuật cần biết khi viết lại

| # | Mức | Vấn đề | Ghi chú cho bản mới |
|---|---|---|---|
| 1 | Cao | Lưu mật khẩu dạng rõ trong localStorage | Không bao giờ lưu mật khẩu |
| 2 | Cao | Tải lại 11–13 MB mỗi lần mở (nginx `no-store` + bundle lớn) | Asset có hash, cache 1 năm, `index.html` no-cache |
| 3 | Cao | iPad stall với hls.js/MSE. Bản vá chưa merge vào `main`, chưa có cho Thư viện | Native HLS với token URL trên mọi WebKit iOS |
| 4 | Cao | Mất tiến độ khi đóng tab (không gửi heartbeat cuối) | `fetch(keepalive)` ở `pagehide` |
| 5 | TB | `durationWatched`/`timeSpentSeconds` sai nghĩa | Sửa payload, cần phối hợp backend |
| 6 | TB | Hai hệ điều hướng; back, F5 và URL sai | Mỗi trang một URL thật |
| 7 | TB | Lỗi mạng bị đổi thành `null`; 502 chuyển sang google.com | Lỗi có kiểu rõ ràng, UI tự quyết hiển thị |
| 8 | TB | Quiz so khớp chuỗi con (`"1"` khớp `"12"`) | Dùng `Set<string>` |
| 9 | TB | `read-all` sai method; xoá note sai method | Sinh client có kiểu từ Swagger |
| 10 | TB | PPTX preview mâu thuẫn giữa các nhánh | Chỉ chọn một cách: PDF đã convert |
| 11 | Thấp | Nhãn "Đang trễ hạn" đảo logic | |
| 12 | Thấp | Bộ lọc không bao giờ gửi `taIds`; "Sắp đến hạn/Quá hạn" bị bỏ qua ở tab Khoá học | Thiết kế lại bộ lọc |
| 13 | Thấp | Dropdown Brand ở "Gửi câu hỏi" không lọc theo TA đã chọn | |
| 14 | Thấp | Khoảng 539 chuỗi tiếng Việt hard-code ngoài l10n; không đổi được ngôn ngữ | i18n từ đầu |
| 15 | Thấp | Code chết: model AI chat, GetStream, guide_ui, notification local, 9 package không dùng | Không port |
| 16 | Thấp | `HomeBloc.connectivityListen` là `late` nhưng không bao giờ gán, `close()` sẽ ném lỗi | |

---

## 5. Những gì bản hiện tại làm đúng, nên giữ
- **Phát hiện stall theo vị trí phát**, không tin `waiting`/`isBuffering`.
- **Bỏ header cho signed URL** của storage.
- **Native HLS với `?t=` trên iOS**, phát hiện được cả iPad giả UA Mac.
- **Hoãn seek tới `canplay`**; resume đi qua một thao tác của người dùng.
- **Tạm dừng khi video khuất viewport hoặc tab ẩn**, debounce 2s cho mất focus ngắn.
- **Refresh token có hàng đợi** các request 401.
- **Telemetry video đẩy theo lô và export được** trên iPad.
- **Link sâu `/lesson/:id`** khi chưa đăng nhập: đăng nhập xong mở lại đúng bài.

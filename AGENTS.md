## Agent Browser
profiles:
  vinh: 49222

# Ghi chú làm việc — DoctoTek / Bayer (Pharma Learning Hub)

Mật khẩu, key, secret: **không ghi ở đây**. Xem `.env-doctotek/`:
- `account.txt`: tài khoản các dịch vụ, IP server, cách SSH.
- `bayer-prod.env`: env prod Bayer + tài khoản đăng nhập prod (learner/admin) ở cuối file.
- `doctotek-prod.env`, `doctotek-dev.env`: env DoctoTek.
- SSH key: `~/.ssh/access_server_ed25519`, user `root`. Dùng chung cho **cả 4 server** bên dưới.

## Bản đồ server (đã xác minh 2026-09-24)

| IP | Hostname | Vai trò | Chạy gì |
|---|---|---|---|
| **112.213.87.95** | cloudvps8795.superdata.vn | **PROD Bayer** | docker `bayer-nginx/web/api/postgres/redis`, tại `/opt/bayer`. DB `bayer_training`, user `bayer_user` |
| 112.213.88.66 | – | **Staging Bayer** (qa-bayer) | docker `bayer-*`, DB `bayer_training_stg`, user `bayer_stg` (KHÁC prod) |
| 112.213.88.123 | cloudvps88123.superdata.vn | Prod DoctoTek | edocjo prod + tendoanhnghiep. **Không có Bayer** |
| 112.213.88.192 | – | convertHLS + dev DoctoTek | `media-processing-*` (hls.doctotek.com), edocjo dev. **Không có Bayer** |

- Domain Bayer đều đi qua **Cloudflare** nên DNS không lộ IP gốc:
  - prod: `bayer.doctotek.com` (học viên), `internal-bayer.doctotek.com` (admin + API)
  - staging: `qa-bayer.doctotek.com`, `internal-qa-bayer.doctotek.com`
  - `qa-bayer.doctotek.com` bị **Cloudflare Access** chặn (302 sang trang đăng nhập) → user thường không test được staging. API `internal-qa-bayer…/api` vẫn gọi được. Kiểm tra bản web staging thì đọc thẳng `/opt/bayer/flutter-web/version.json` trên .66.
  - Script kiểm tra end-to-end fix HLS (18 check: native `?t=`, Bearer cũ, các ca 401) nên viết lại theo mẫu: login → `progress/material` → master/child/segment → token sai bài / không auth / token làm Bearer.
- Cách đã dùng để tìm IP prod khi chưa biết: mở log job "Deploy to Production VPS" trên GitHub Actions (`eDocJo/bayer-api`), bước **Set up job** in ra `Machine name: cloudvps8795`. Quy tắc tên superdata: `cloudvpsAABBB` tương ứng `112.213.AA.BBB`.
- Deploy prod/staging dùng **self-hosted runner** (nhãn `bayer-vps` / `bayer-staging-vps`), runner tự kết nối ra GitHub nên workflow không ghi IP.
- Postgres không mở cổng ra host. Truy vấn bằng: `ssh ... root@<IP> "docker exec -i bayer-postgres-1 psql -U <user> -d <db>" <<'SQL' ... SQL` (heredoc để khỏi lỗi escape nháy).
- Nhận env thật của container: `docker exec <container> env`. Đừng tin file `.env` local: staging dùng credential khác prod.

## Repo (clone ở `sub-project/`)

| Repo | Là gì | Ghi chú |
|---|---|---|
| `bayer-api` | Monorepo Bayer: `apps/api` (NestJS), `apps/web` (React admin), **`web/` = bản build Flutter web học viên** (commit sẵn) | Nhánh: `develop` → staging, `master` → prod (tự deploy khi push). `main` đã cũ, không dùng |
| `bayer-doctotek-app` | **Mã nguồn app học viên Flutter** (`bayer.doctotek.com`) | Build xong copy vào `bayer-api/web/`. Plugin video fork ở `third_party/video_player_web_hls` |
| `edocjo-api`, `edocjo-mobile` | DoctoTek (không phải Bayer) | |

### Build app Flutter
- Cần đúng Flutter **3.47.0** (`.fvmrc`, Dart ^3.11). `flutter` trên PATH (3.38.5) **không build được**.
- SDK đã cài ở `~/fvm/versions/3.47.0/bin/flutter` (clone tag 3.47.0).
- Build: `flutter build web --release --no-wasm-dry-run`, sau đó copy thủ công `web/manifest.json` và `web/icons/app_icon.png` vào `build/web/` (build không tự copy).
- Copy vào `bayer-api/web/` theo đúng danh sách file trong `scripts/build-web-fast.sh`.
- Mỗi lần phát hành: tăng `version` trong `pubspec.yaml` **và** tham số `flutter_bootstrap.js?v=` trong `web/index.html` (dạng `<ver><build><ddmmyyyy>`), nếu không iPad sẽ giữ JS cũ trong cache.
- Trước khi build, so `web/version.json` giữa `develop`/`master` với commit app, để biết bản build đang chạy lấy từ commit nào.

### Test API (`bayer-api/apps/api`)
- Lần đầu: `npm install` ở root, rồi build `packages/shared` (`npm run build`), nếu không jest báo thiếu `@bayer/shared`.
- `npx tsc --noEmit -p tsconfig.json`; `npx jest --ci <path>`.
- Chạy full suite đôi khi có 2 test Excel (`users`, `user-groups`) timeout 5s do máy tải nặng. Chạy riêng thì pass, không phải lỗi.

## Luồng video HLS (đọc trước khi debug video)

1. Admin upload MP4, backend gửi job sang **media-processing** (.192, `hls.doctotek.com`). Service convert ra 480p/720p, ghi vào bucket `bayer-uploads` (staging: `bayer-uploads-stg`, objstore43185), rồi callback về `internal-bayer…/api/media-jobs/callbacks/jobs`.
   - Kiểm tra job: DB `media_processing` trên .192 (`docker exec -i media-processing-postgres-1 psql -U doctotek_media -d media_processing`), bảng `media_jobs`, `job_attempts`, `callback_deliveries`.
   - `ffprobe`/`ffmpeg` có sẵn trong container `media-processing-worker-1`.
2. App học viên lấy nguồn phát từ **`GET /api/progress/material/:id`**, dùng các field `hlsUrl` / `fileUrl` / `hlsPlaybackUrl`. **Không** dùng `/content/materials/:id`.
3. `hlsUrl` là proxy `/api/content/lessons/:id/hls/master.m3u8`. Backend viết lại playlist: playlist con đi qua proxy, segment là **signed URL MinIO 24h**.
   - Gửi signed URL kèm header `Authorization` thì storage trả **400** (fork đã bỏ header cho URL có `X-Amz-Signature`).
   - Mở `master.m3u8` trực tiếp trên storage (không qua proxy) thì playlist con bị **403**.
4. Bài học gắn Document: `LessonContentResolver` chỉ dùng Document khi đúng 1 document hợp lệ (active, chưa hết hạn, đúng loại, có HLS).

### Fix iPad đứng video (2026-09-24)
- Triệu chứng: bài 6.1/6.2 Firialta_HF đứng ở giây 0, **chỉ trên iPad** (Edge/Safari iOS).
- Log prod cho thấy khoảng 70% phiên xem trên iPad bị stall, máy khác khoảng 11%. Xảy ra ở mọi bài HLS, 6.x cao nhất.
- Nguyên nhân: native `<video>` trên iOS không gắn được header nên app ép dùng hls.js qua ManagedMediaSource, không ổn định.
- Fix:
  - Backend thêm HLS token `?t=` (module `apps/api/src/hls-token/`, secret dẫn xuất riêng, gắn materialId, hạn 12h). Route HLS tách sang `HlsPlaybackController` với guard nhận Bearer **hoặc** `?t=`. API trả thêm `hlsPlaybackUrl`.
  - App trên iOS dùng `hlsPlaybackUrl` + trình phát HLS gốc (fork: headers rỗng + `canPlayHlsNatively` thì chạy native).
- Commit: `bayer-api` develop `697f231`, master `0448688`. App: nhánh `fix/ios-native-hls`, bản 0.0.48+114.

## Log lỗi phía client (quan trọng nhất khi khách báo lỗi)
- App gửi telemetry về `POST /api/log/client-events`, lưu vào bảng **`system_logs`** (`context='client-event'`). Event gồm `stall_detected`, `stall_recovered`, `video_error`, `heartbeat_failure`, `lifecycle_transition` (có `lessonId`, `sessionId`, `userAgent`).
- API `/api/system-logs` chỉ cho **superadmin**, `admin@bayer.com` bị 403. Nên **query thẳng DB prod** trên 112.213.87.95.
- `stall_detected` không có `lessonId`: join với `lifecycle_transition` qua `metadata->>'sessionId'`, hoặc đoán bài qua `durationSeconds`.
- Luôn **chuẩn hoá theo số phiên xem** trước khi kết luận "bài X lỗi". Stall dồn vào bài đang được học nhiều nhất.
- `audit_logs` chỉ ghi thao tác admin (upload/sửa), không có lượt xem. `updated_at` của material tăng cả khi có lượt xem (viewCount), **không phải** dấu hiệu có người sửa.

## Kinh nghiệm dùng browser tự động (astraler, CDP port 49222)
- App học viên là **Flutter web (CanvasKit)**: không có DOM input, `querySelector`/`fill`/`snapshot` đều rỗng. Phải thao tác bằng `mouse move/down/up` theo toạ độ + `keyboard type`, và kiểm tra bằng screenshot.
- **Tab bị ẩn/che thì Chrome dừng phát video**, bài nào cũng treo ở vòng xoay loading. Không phải lỗi app. Kiểm tra `document.visibilityState`. Đưa cửa sổ lên foreground (PowerShell `SetForegroundWindow` qua `powershell.exe -File`) rồi reload.
- Route học viên: `/lesson/<materialId>`, mở thẳng được, không cần click qua danh sách.
- Gọi `npx -y agent-browser` mỗi lần rất chậm. Dùng binary trực tiếp: `~/AppData/Local/npm-cache/_npx/<hash>/node_modules/.bin/agent-browser`.
- Test Chrome desktop **không tái hiện lỗi iPad**. Lỗi riêng iOS phải dựa vào log `system_logs` hoặc thiết bị thật.
- Lấy token từ trang đã đăng nhập: `localStorage.accessToken` (admin React), `flutter.USER_SESSION` (app học viên).

## Push repo GitHub

**Cách push doctotek-current repo lên GitHub:**

```bash
# 1. Dùng SSH key đã setup (id_ed25519_github)
export GIT_SSH_COMMAND="ssh -i ~/.ssh/id_ed25519_github"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_github

# 2. Thiết lập remote SSH
cd E:\Project2026\doctotek-current
git remote add origin git@github.com:vinhnguyenthanhdn/doctotek-current.git
git branch -M main
git push -u origin main
```

**Notes:**
- SSH key đã xác thực: `ssh -T git@github.com` trả `Hi vinhnguyenthanhdn!`
- Nếu repo chưa tồn tại, dùng `gh repo create doctotek-current --public --source=. --remote=origin --push` (sau khi `gh auth status` ok)
- `.gitignore` đã config: ignore `sub-project/`, `.env*`, `*.key`, `*.secret`, `node_modules/`
- Windows Credential Manager lưu token ở `LegacyGeneric:target=git:https://github.com` nhưng không thể extract via cmdline, dùng SSH thay thế

## Bẫy đã gặp (tránh lặp lại)
- Đừng kết luận nguyên nhân chỉ từ endpoint "trông giống". Phải xác định **đúng endpoint client thật sự gọi** (xem network, hoặc đọc mã nguồn/`main.dart.js`).
- `/api/reports/learning-paths/:id` và `/api/reports/users/:id` đang trả **500** trên prod (chưa điều tra).
- Git Bash đổi `origin/master:path` thành đường dẫn Windows. Dùng `MSYS_NO_PATHCONV=1 git show "origin/master:..."`.
- Grep cả `/home` trên server có thể treo. Luôn dùng `timeout` và giới hạn thư mục.

# 05. Style direction và kế hoạch tối ưu giao diện, UI/UX

> **Người đọc:** Vĩnh (duyệt), Cherry và agent làm UI trong `eDocJo/doctotek-web`.
> **Nguồn:**
> - Ảnh chụp doctotek.com (landing, đo màu từ pixel) ngày 28/09/2026.
> - `design/STYLE.md` và `design/screenshots/` trong repo `doctotek-web`, là ảnh prod chụp sau khi đăng nhập.
> - Trang tham khảo 8 nền tảng: [tham-khao-ui/index.html](tham-khao-ui/index.html).
>
> **Trạng thái:** **đã chốt ngày 28/09/2026** (mục 8). Task nằm trên AgentDesk, project General, tiền tố tiêu đề `UI-`.

**Quan hệ với `design/STYLE.md`:** `STYLE.md` là luật (token, thành phần, checklist). File này là **định hướng và kế hoạch**. Mục nào được duyệt thì chép vào `STYLE.md` mục 7 rồi mới làm. Khi hai file nói khác nhau, **`STYLE.md` thắng**.

## 1. Định hướng

**Một câu:** DoctoTek là nơi nhân viên y tế *học nhanh và tra cứu có chứng cứ*. Giao diện phải **sạch, sáng, đáng tin**. Màu cam là **tín hiệu hành động**, không dùng để trang trí.

| Tính cách | Thể hiện bằng | Tránh |
|---|---|---|
| Chuyên môn | Chữ rõ, số liệu có nguồn, ảnh thật của bài giảng, bác sĩ, hội nghị | Minh hoạ hoạt hình, emoji, câu chữ quảng cáo phóng đại |
| Gần gũi | Tiếng Việt tự nhiên, bo góc mềm, nhiều khoảng thở ở landing | Giao diện lạnh kiểu phần mềm bệnh viện |
| Nhanh | Mỗi khối có **một** việc chính, tìm kiếm luôn ở gần, skeleton thay chữ "Đang tải" | Nhiều nút ngang cấp, popup chặn nội dung |

## 2. Giữ nguyên (không đổi)

### 2.1 Màu

Đo trên doctotek.com và `design/screenshots/`. Tên token lấy theo `STYLE.md` mục 2.

| Vai trò | Mã | Token | Dùng ở |
|---|---|---|---|
| Cam chủ đạo | `#D93300` | `--brand` | Nút chính, tab đang chọn, link, giá, thanh tiêu đề khối, icon menu |
| Cam hover | `#B22A00` | `--brand-hover` | Hover/nhấn của nút cam |
| Viền cam nhạt | `#EC997F` | (chưa có token, nên thêm `--brand-line`) | Viền nút phụ ở landing |
| Nền cam nhạt | `#FFF4EF` / `#FFF9F8` | `--brand-soft` / `--brand-softer` | Nền icon tròn, tab đang chọn, hero "Khám phá nội dung" |
| Nền kem footer | `#FFF9F6` | (landing) | Footer landing |
| Nền xám nhạt | `#F5F5F5` (landing) / `#F7F8F9` (trong app) | `--bg-app` | Hero landing, vùng nội dung sau đăng nhập |
| Trắng | `#FFFFFF` | `--surface` | Header, thẻ, sidebar |
| Chữ | `#212121` / phụ `#757575` | `--text` / `--text-muted` | |
| Đen nút store | `#2C2C2C` | | Nút Google Play, App Store |
| **Tím, chỉ cho AI** | `#6D4EF2` | `--ai` | Nút "Hỏi AI", badge Beta, nền icon AI. **Landing không dùng tím** |

**Không thêm màu thương hiệu mới.** Xanh, teal, navy hay hồng thấy ở các trang tham khảo đều quy đổi như sau:

| Ở trang tham khảo | Quy về DoctoTek |
|---|---|
| Nền hero màu đậm (Osmosis xanh, AMBOSS teal) | Nền `#F5F5F5` + hoạ tiết mạch, hoặc `--brand-softer` |
| Nút chính màu khác | Nút đặc `--brand` |
| Dải CTA/tải app màu đậm (MIMS đỏ, Osmosis xanh) | Nền `--brand-soft` chữ `--text`, hoặc nền `#2C2C2C` chữ trắng |
| Thẻ số liệu, bento màu | Thẻ trắng viền `--border`, số màu `--brand` |
| Màu phân biệt tính năng | Chỉ cam (học, tra cứu) và tím (AI) |

### 2.2 Những thứ khác giữ nguyên
- **Font Inter** 400/500/600/700. Không thêm font hiển thị thứ hai.
- **Logo** `logo_doctotek.webp`.
- **Hoạ tiết mạch điện** đường mảnh ở hero landing và hero "Khám phá nội dung". Đây là dấu hiệu nhận diện riêng.
- **Khung app:**
  - Desktop: top bar 5 tab và sidebar Profile bên trái (280px, thu gọn còn 76px).
  - Mobile: header chỉ có logo và bottom nav 5 tab.
- **Câu chữ:** lấy từ `app_vi.arb`, không tự viết lại. Chỗ nào cần câu mới thì thuộc mức 3 (mục 5).
- **Thứ tự khối** ở trang chủ trong app và ở landing. Muốn đổi thì thuộc mức 2.

## 3. Nguyên tắc style

1. **Một hành động chính mỗi khối.** Mỗi khối chỉ có một nút cam đặc. Hành động phụ dùng nút viền hoặc link. Trên landing hiện có 3 nút gần ngang cấp trong hero, đây là chỗ sửa đầu tiên (xem P1).
2. **Trung tính trước, màu sau.** Trang chủ yếu là trắng và xám nhạt. Cam chiếm dưới khoảng 10% diện tích mỗi màn. Khi cam xuất hiện, người dùng hiểu đó là chỗ cần bấm.
3. **Tách lớp bằng viền mảnh, không bằng bóng.** Thẻ trắng, viền `--border`, bo 12px. Bóng chỉ dùng khi hover (xem `STYLE.md` 7.7) và cho lớp nổi (dialog, dropdown).
4. **Ảnh thật hơn minh hoạ.** Thumbnail bài giảng, ảnh chụp giao diện app, ảnh chuyên gia. Minh hoạ chỉ dùng cho icon tính năng đã có trong `design/assets/icons`.
5. **Hai mật độ.**
   - Landing (người lạ): thoáng. Mỗi khối một ý, padding dọc khoảng 72–96px ở desktop.
   - Trong app (người dùng quen): gọn như Medscape và MIMS. Khoảng cách giữa các khối 24–32px, thấy được nhiều thẻ trên một màn.
6. **Thang chữ cố định.** Dùng bảng `STYLE.md` mục 2. Tiêu đề trên landing thêm 2 cỡ: H1 40px/700 (mobile 28px) và H2 32px/600 (mobile 24px). Không đặt cỡ lẻ ngoài thang.
7. **Lưới 8px.** Khoảng cách chọn trong 4, 8, 12, 16, 24, 32, 48, 72, 96. Các phần tử cùng hàng dùng `gap`, không dùng margin lẻ.
8. **Chuyển động tiết chế.** Chỉ dùng cho hover thẻ, mở/đóng sheet, đổi tab ảnh (P5). Thời lượng 150–250ms. Tắt hết khi `prefers-reduced-motion`.
9. **Trạng thái luôn có hình.** Đủ 5 trạng thái: đang tải (skeleton), rỗng (chữ và đường đi tiếp), lỗi (lý do và nút thử lại), thành công, bị khoá (chưa đăng nhập hoặc chưa mua).

## 4. Mẫu bố cục học từ các trang tham khảo

Ảnh của từng trang nằm trong [tham-khao-ui/index.html](tham-khao-ui/index.html). Chỉ học **cấu trúc**; màu áp theo mục 2.1.

| Mã | Mẫu | Học từ | Áp vào màn | Mức |
|---|---|---|---|---|
| P1 | **Hero một CTA chính:** 1 nút cam đặc; 2 lối vào phụ thành 2 thẻ nhỏ có icon (Khóa học y khoa, AI y học chứng cứ) | Osmosis, Lecturio | Landing `/about` | 2 |
| P2 | **Thẻ khóa học chuẩn:** thumbnail 16:9, nhãn thời lượng, nhãn chuyên khoa hoặc CME (nếu API có), tiêu đề 2 dòng, đơn vị tổ chức, giá, nút dính đáy | MIMS VN | Trang chủ app, Khóa học, Kênh học thuật | 1 |
| P3 | **Tiêu đề khối có bộ lọc tại chỗ:** tiêu đề với thanh cam, chip lọc hoặc sắp xếp ngay cạnh, "Xem tất cả" bên phải | MIMS VN, Medscape | Các rail ở trang chủ app và Khóa học | 1 nếu chip đã có (Mới nhất / Xem nhiều nhất), còn lại 2 |
| P4 | **Một ô tìm kiếm cho hai việc:** tra cứu thuốc và hỏi AI, có công tắc hoặc tab chuyển chế độ ngay trong ô | Medscape (AI Mode) | Trang chủ app, nơi hiện có 2 thẻ Tra cứu thuốc và Hỏi AI | 2 |
| P5 | **Ảnh sản phẩm có tab:** một khung ảnh giao diện lớn, 3 tab tính năng AI (EBM, soạn bài giảng, tóm tắt) | AMBOSS | Khối "AI y học chứng cứ" ở landing | 2 (ảnh chụp app có sẵn) |
| P6 | **Khối tính năng xen kẽ:** chữ bên trái, ảnh chụp app bên phải, lần lượt đổi bên | Osmosis | Khối "Khóa học y khoa ngắn" ở landing | 2 |
| P7 | **Hộp tìm kiếm đè mép hero, kèm chip chuyên khoa** | Harvard HMS | Hero "Khám phá nội dung" ở tab Khóa học | 1 cho kích thước, độ nổi, focus; 2 nếu thêm chip |
| P8 | **Lối tắt theo đối tượng:** Bác sĩ, Dược sĩ, Điều dưỡng, Giảng viên | Osmosis, Lecturio | Landing, ngay dưới hero | 3 |
| P9 | **Dải số liệu, lời chứng thực, logo đối tác** | Harvard, AMBOSS, Osmosis | Khối Cộng đồng và Hợp tác ở landing | 3 |
| P10 | **FAQ accordion** về CME, chứng nhận, thanh toán | MIMS VN | Landing, trước footer | 3 |
| P11 | **Dải tải app** có mockup thiết bị, ngay trên footer | MIMS VN | Landing | 2 (ảnh mockup có sẵn) |
| P12 | **Thẻ case study** thay cho 5 icon đối tác | MedShr | Khối Hợp tác, `/corporate` | 3 |

## 5. Ba mức thay đổi

| Mức | Nghĩa | Ai quyết | Cách ghi trong PR |
|---|---|---|---|
| **1. Tinh chỉnh** | Giữ bố cục, thứ tự khối và câu chữ. Chỉ sửa độ đều, khoảng cách, trạng thái, tương phản, focus, hiệu năng | Cherry tự làm, nằm trong `STYLE.md` mục 7 | `UI/UX: …` |
| **2. Đổi bố cục** | Sắp lại hoặc đổi cách trình bày, **chỉ dùng nội dung và API đã có** | Vĩnh duyệt một lần theo danh sách mục 6.3 | `Lệch so với bản cũ: … vì …` |
| **3. Nội dung mới** | Cần câu chữ, số liệu, logo, lời chứng thực hoặc API mới | Chủ sản phẩm cung cấp nội dung | Chỉ làm khi đã có nội dung, không dùng dữ liệu giả |

## 6. Kế hoạch

### 6.1 Đợt 0: chuẩn bị (một PR, trước các đợt sau)

| ID | Việc | Nghiệm thu |
|---|---|---|
| U0.1 | Rà `src/style.css` theo token `STYLE.md` mục 2. Thêm `--brand-line: #EC997F`, `--bg-landing: #F5F5F5`, `--footer-landing: #FFF9F6`, `--store: #2C2C2C`. Thêm thang chữ landing (H1, H2) và thang khoảng cách 8px | Không còn mã màu hard-code ngoài `:root`/`@theme` (kiểm bằng grep `#[0-9a-fA-F]{3,6}` trong `src/`, trừ file token) |
| U0.2 | Chụp **baseline** các màn đã làm ở 390×844, 820×1180, 1440×900, lưu `design/baseline/` | Có ảnh để so trước và sau từng đợt |
| U0.3 | Đo Lighthouse (mobile) và axe cho landing `/about`, trang chủ app, Khóa học | Ghi số vào PR làm mốc |

### 6.2 Đợt 1: tinh chỉnh (mức 1, làm kèm M1–M3, không cần duyệt)

Làm ngay trong PR của màn tương ứng, không mở PR riêng.

| ID | Việc | Màn | Mẫu / nguồn | Nghiệm thu |
|---|---|---|---|---|
| U1.1 | Thẻ khóa học chuẩn: đều chiều cao, nút dính đáy, tiêu đề in hoa bằng CSS, tối đa 2 dòng, có `title` | Mọi rail/lưới khóa học | P2, `STYLE.md` 7.3, 7.5 | Trong cùng hàng, đáy nút thẳng hàng ở cả 3 kích thước chụp |
| U1.2 | Skeleton đúng hình thẻ thay cho "Đang tải dữ liệu…" | Trang chủ app, Khóa học, Kênh học thuật | `STYLE.md` 7.1 | CLS < 0.1 khi dữ liệu về |
| U1.3 | Trạng thái rỗng giữ câu cũ, thêm link sang route đã có | "Hội thảo & CME sắp diễn ra"… | `STYLE.md` 7.2 | Không có trang mới |
| U1.4 | Rail: mũi tên desktop tự ẩn ở hai đầu; mobile dùng snap, hé 12% thẻ kế tiếp | Mọi rail | `STYLE.md` 7.4 | Tab bàn phím tới được mũi tên; bấm mũi tên cuộn đúng 1 trang thẻ |
| U1.5 | Tiêu đề khối thống nhất: thanh cam 4×28px, tiêu đề 20/500, "Xem tất cả" cùng baseline | Mọi màn trong app | P3 | So ảnh: mọi tiêu đề khối cùng lề và cỡ |
| U1.6 | Ô tìm kiếm: pill cao 48px, focus ring cam, nút tròn cam có `aria-label`; bấm Enter là tìm | Trang chủ app, hero Khóa học | P7 (phần mức 1) | axe không lỗi; Enter hoạt động |
| U1.7 | Sửa tương phản theo `STYLE.md` 7.6: tab chưa chọn `#757575`; chữ phụ trên nền xám `#6B6B6B`; chữ cam trên nền cam nhạt ≥ 600 | Toàn app | `STYLE.md` 7.6 | axe không có lỗi `color-contrast` |
| U1.8 | Top bar desktop: mỗi tab tối đa khoảng 160px, canh giữa | Khung app ≥ 1024 | `STYLE.md` 7.8 | Ở 1440px, 5 tab không kéo giãn hết chiều rộng |
| U1.9 | Hover (chỉ khi `hover:hover`): thẻ nâng 2px, nút đổi `--brand-hover`; tắt khi reduced-motion | Toàn app | `STYLE.md` 7.7 | Không có chuyển động khi bật reduced-motion |
| U1.10 | Ảnh: `loading="lazy"`, `decoding="async"`, có `width`/`height`; ảnh LCP dùng `fetchpriority="high"`; banner có nút tạm dừng | Toàn app, landing | `STYLE.md` 7.10, 7.12 | LCP mobile ≤ 2.5s trên dev (Lighthouse, mạng 4G giả lập) |
| U1.11 | Landing, giữ bố cục: căn lại khoảng cách theo lưới 8px và thang chữ; khoảng cách dọc giữa các khối đều nhau | `/about` | Mục 3.5–3.7 | So với baseline: chỉ khác khoảng cách và cỡ chữ theo thang |

### 6.3 Đợt 2: đổi bố cục (mức 2, cần Vĩnh duyệt danh sách này)

**Đã duyệt ngày 28/09/2026:** U2.1, U2.2, U2.3, U2.5, U2.6 (có điều kiện). U2.4 không làm. Nên làm sau khi thẻ khóa học (U1.1) đã ổn định.

| ID | Việc | Màn | Mẫu | Dùng nội dung/API có sẵn | Rủi ro |
|---|---|---|---|---|---|
| U2.1 | Hero landing một CTA chính: giữ "Khám phá ngay" là nút đặc; "Xem khóa học nổi bật" và "Trải nghiệm AI Y học chứng cứ" thành 2 thẻ có icon, **giữ nguyên chữ và link** | `/about` | P1 | Chữ, link, icon `ai_ebm`, `online_video` | Thấp |
| U2.2 | Khối "Khóa học y khoa ngắn": thay mockup minh hoạ bằng ảnh chụp app thật, bố cục chữ/ảnh xen kẽ | `/about` | P6 | Ảnh chụp từ `design/screenshots/` hoặc chụp lại trên dev | Thấp. Không dùng dữ liệu của tài khoản thật trong ảnh |
| U2.3 | Khối "AI y học chứng cứ": ảnh giao diện kèm 3 tab (EBM, soạn bài giảng, tóm tắt), mỗi tab giữ câu mô tả cũ và nút "Thử tra cứu" | `/about` | P5 | Ảnh chụp tab AI trên dev | Trung bình: cần 3 ảnh chụp đẹp |
| U2.4 | Gộp 2 thẻ "Tra cứu thuốc" và "Hỏi AI lâm sàng" thành một ô, có tab chuyển chế độ. Tab "AI" dùng màu tím | Trang chủ app | P4 | Cùng 2 API như bản cũ | **Không làm.** Chốt phương án rẻ: giữ 2 thẻ, chỉ làm đều chiều cao và vị trí nút (mức 1, gộp vào task UI-1d) |
| U2.5 | Dải tải app có mockup, đặt ngay trên footer landing | `/about` | P11 | Nút store, ảnh app | Thấp |
| U2.6 | Chip chuyên khoa dưới ô tìm kiếm ở hero Khóa học | Tab Khóa học | P7 | **Chỉ làm nếu API khóa học đã lọc được theo chuyên khoa** (kiểm Swagger dev). Nếu không thì đóng task, ghi lý do | Không sửa backend |

### 6.4 Đợt 3: nội dung mới, lấy từ dữ liệu thật

Không dùng số liệu hoặc lời chứng thực giả, kể cả để demo. Nội dung dưới đây đọc từ **DB prod ngày 28/09/2026**, chỉ đọc (`default_transaction_read_only=on`).

| Dữ liệu | Giá trị trên prod | Quyết định |
|---|---|---|
| Khóa học | 122 khóa: 97 `completed`, 23 `rehearsal`, 2 `archived` | Hiện **"90+ khóa học"** |
| Chuyên gia (`speakers`) | 28, trong đó 13 có gắn khóa học | Hiện **"25+ chuyên gia"** |
| Hội, tổ chức (`organizations`) | 31: Doctotek và 30 hội/tổ chức chuyên môn. Chỉ 4 có logo | Hiện **"30 hội và tổ chức chuyên môn"** |
| Tra cứu thuốc | 1.503 thuốc, 846 hoạt chất | Hiện **"1.500+ thuốc, 840+ hoạt chất"** |
| Người dùng | 585 tài khoản | **Không hiện.** Số còn nhỏ, dễ phản tác dụng |
| Đánh giá (`course_ratings`) | 8 lượt, trung bình 4,25; bình luận dài nhất 40 ký tự; `course_feedbacks` rỗng | **Bỏ lời chứng thực (U3.3).** Không đủ nội dung, lại không có sự đồng ý của người viết |
| Logo có khóa học | Phân hội Xơ vữa động mạch Việt Nam (51 khóa), Hội tim mạch TP.HCM (31 khóa), United International Pharma (6), Doctotek | Chỉ dùng **2 hội**. Không đưa logo công ty dược lên landing. Ảnh (webp 400px, tải từ storage prod) lưu ở `design/assets/images/partners/`: `phan-hoi-xo-vua-dong-mach-vn.webp`, `hoi-tim-mach-tphcm.webp` |
| FAQ | Không có bảng FAQ trong DB. `app_vi.arb` có 4 câu hỏi và trả lời AI EBM (`ebm_faq_q1..a4`) và `faq_title` | Dùng **4 câu EBM** cho khối AI trên landing. Không tự viết FAQ về CME hoặc thanh toán |

- Số liệu **làm tròn xuống**, để trong một file hằng số, ghi ngày lấy và câu truy vấn. Không gọi API: `/organizations` và các API thống kê đều cần đăng nhập (trả 401). Khi cần cập nhật số thì người làm chạy lại truy vấn rồi sửa file.
- **Không làm:** U3.1 lối tắt đối tượng (chưa có trang đích riêng cho từng đối tượng), U3.3 lời chứng thực, U3.5 FAQ về CME, U3.6 carousel khóa học công khai (cần API mới trên `edocjo-api`, sẽ chạm prod).
- **Làm:** U3.2 dải số liệu và U3.4 logo 2 hội, gộp chung một task (UI-2d). FAQ EBM gộp vào task khối AI (UI-2c).

### 6.5 Đợt 4: đo lại và chốt (trong M4)

| ID | Việc | Nghiệm thu |
|---|---|---|
| U4.1 | Chụp lại toàn bộ màn ở 3 kích thước, đặt cạnh baseline U0.2 | Mọi khác biệt đều có mã U1.x/U2.x tương ứng |
| U4.2 | Lighthouse mobile landing và trang chủ app | Performance ≥ 90, Accessibility ≥ 95, CLS < 0.1, LCP ≤ 2.5s |
| U4.3 | axe toàn bộ route chính | Không có lỗi serious/critical |
| U4.4 | Test tay trên iPhone Safari và iPad | Bottom nav và safe-area đúng; input không zoom; video `playsinline` |

## 7. Checklist cho mỗi PR có đổi giao diện
- [ ] Màu chỉ lấy từ token. Không thêm màu thương hiệu. Tím chỉ xuất hiện ở AI.
- [ ] Mỗi khối có tối đa một nút cam đặc.
- [ ] Khoảng cách theo lưới 8px, cỡ chữ theo thang.
- [ ] Đủ trạng thái đang tải, rỗng, lỗi.
- [ ] Ảnh chụp 390×844, 820×1180, 1440×900, đặt cạnh ảnh prod hoặc baseline.
- [ ] Ghi mã việc (`U1.x`, `U2.x`) trong mô tả PR. Việc mức 2 phải ghi thêm `Lệch so với bản cũ: … vì …`.
- [ ] axe không có lỗi serious. Điều hướng bằng bàn phím đi đúng thứ tự.

## 8. Quyết định ngày 28/09/2026
1. **Đợt 2:** làm U2.1, U2.2, U2.3, U2.5. U2.6 chỉ làm nếu API đã hỗ trợ. **U2.4 không làm**, thay bằng phương án rẻ (giữ 2 thẻ, làm đều).
2. **Đợt 3:** chỉ làm dải số liệu, logo 2 hội và FAQ EBM, dùng dữ liệu thật ở mục 6.4. Các phần còn lại không làm.
3. **Đồng bộ:** file này và `tham-khao-ui/` chép sang `doctotek-web/docs/plan/`. `design/STYLE.md` mục 7 thêm dòng trỏ tới file này.
4. **Task:** 14 task `UI-*` trên AgentDesk, gán Cherry. Task UI-0a, UI-0b chạy trước; các task khác chờ approve theo thứ tự ở bảng dưới.

| Task | AgentDesk | Gồm | Chờ |
|---|---|---|---|
| UI-0a | GENERAL-17 | U0.1 token, thang chữ, thang khoảng cách | — |
| UI-0b | GENERAL-18 | U0.2 ảnh baseline, U0.3 Lighthouse và axe | — |
| UI-1a | GENERAL-19 | U1.1 thẻ khóa học chuẩn, U1.2 skeleton | UI-0a |
| UI-1b | GENERAL-20 | U1.3 trạng thái rỗng, U1.4 rail, U1.5 tiêu đề khối | UI-1a |
| UI-1c | GENERAL-21 | U1.6 ô tìm kiếm, U1.7 tương phản, U1.9 hover và reduced-motion | UI-0a |
| UI-1d | GENERAL-22 | U1.8 top bar desktop, 2 thẻ Tra cứu thuốc và Hỏi AI đều nhau | UI-0a |
| UI-1e | GENERAL-23 | U1.10 hiệu năng ảnh, nút tạm dừng banner | UI-1a |
| UI-2a | GENERAL-24 | U2.1 hero một CTA chính, U1.11 lưới 8px ở landing | UI-0a |
| UI-2b | GENERAL-25 | U2.2 khối Khóa học xen kẽ ảnh app thật | UI-2a |
| UI-2c | GENERAL-26 | U2.3 khối AI có tab, FAQ EBM | UI-2a |
| UI-2d | GENERAL-27 | U3.2 dải số liệu, U3.4 logo 2 hội | UI-2a |
| UI-2e | GENERAL-28 | U2.5 dải tải app trên footer | UI-2a |
| UI-2f | GENERAL-29 | U2.6 chip chuyên khoa (có điều kiện) | UI-1c |
| UI-4 | GENERAL-30 | U4.1–U4.4 đo lại và chốt | Tất cả task trên |

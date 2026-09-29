# Yêu cầu: Pop-up đồng ý chia sẻ dữ liệu khi đăng ký hội thảo

> Ghi nhận ngày 2026-09-24, từ trao đổi Zalo. Phục vụ event **Nhất Anh ngày 28/11/2026**.

## 1. Bối cảnh

- DoctoTek sắp tổ chức hội thảo cho **Công ty Nhất Anh**, phối hợp với **Hội Tim mạch học Việt Nam** (Hội / người tổ chức).
- Trong event, **dữ liệu của bác sĩ trên DoctoTek sẽ được chia sẻ cho 2 bên**: Hội và công ty Nhất Anh.
- Khi đăng ký tài khoản, bác sĩ đã đồng ý điều khoản chung. Điều khoản đó **không đủ** cho việc chia sẻ dữ liệu theo từng hội thảo.
- Vì vậy, **trước khi bác sĩ bấm submit đăng ký hội thảo**, app phải thông báo rõ: **thông tin nào của bác sĩ sẽ được chia sẻ, cho ai**. Bác sĩ phải đồng ý thì mới đăng ký được.
- Mẫu tham khảo: pop-up "Đồng ý" của app Z-waka (xem mục 4).

## 2. Luồng người dùng

App đang có card nội dung (ảnh, tiêu đề, mô tả, nút đỏ ở dưới). Ví dụ hiện tại: card "Các bài báo cáo về quản lý tăng huyết áp toàn diện" (SVCC 2026) với nút **"Bắt đầu học"**.

Khi card đó là **hội thảo**, nút đỏ đổi nhãn theo trạng thái sự kiện:

| Trạng thái hội thảo | Nhãn nút đỏ | Bấm vào |
|---|---|---|
| Nội dung học thường (không phải hội thảo) | Bắt đầu học | Vào học như hiện tại, **không** hiện pop-up |
| Hội thảo sắp diễn ra | **Đăng ký** | Hiện pop-up đồng ý chia sẻ thông tin |
| Hội thảo đang diễn ra | **Tham dự trực tuyến** | Hiện pop-up đồng ý chia sẻ thông tin |

Luồng:

1. Bác sĩ bấm **Đăng ký** hoặc **Tham dự trực tuyến**.
2. App hiện **pop-up "Đồng ý"** (mục 3).
3. Bác sĩ đọc và tick đồng ý cho **từng bên nhận**.
4. Khi đã tick đủ các ô bắt buộc, nút xác nhận mới bấm được. Bấm xác nhận thì đăng ký thành công (hoặc vào phòng xem trực tuyến).
5. Bấm **X** để đóng pop-up thì **không** đăng ký, không chia sẻ dữ liệu.

## 3. Nội dung pop-up

**Tiêu đề:** Đồng ý

**Câu dẫn:** "Bằng việc tham gia chương trình này, bạn đồng ý với các điều khoản sau:"

Mỗi **bên nhận dữ liệu** có **một khối riêng**. Với event 28/11 có 2 khối:

### Khối 1: Đồng ý cho Hội Tim mạch học Việt Nam (Người tổ chức)

- "DoctoTek sẽ chia sẻ các thông tin sau với Hội Tim mạch học Việt Nam: *[danh sách field]*"
- "Hội Tim mạch học Việt Nam có thể liên hệ với bạn liên quan đến chương trình này qua email, cuộc gọi điện thoại, SMS, tin nhắn hoặc thông báo trong ứng dụng và các ứng dụng nhắn tin bên thứ ba."
- ☐ Tôi đồng ý với các nội dung trên (bắt buộc)

### Khối 2: Đồng ý cho Công ty Nhất Anh

- Cấu trúc giống khối 1, thay tên bên nhận và danh sách field (nếu khác).
- ☐ Tôi đồng ý với các nội dung trên (bắt buộc)

Nội dung dài thì cuộn được trong pop-up. Nút xác nhận đặt ở cuối.

## 4. Mẫu tham khảo (Z-waka)

Danh sách field Z-waka chia sẻ cho Hội Tim mạch học Việt Nam:

User ID, Title, Name, Degree, Profile Photo, Profession, Speciality, Experience, State/Region/Province, Country, Engagement Data, Gender, Email Address, Date of Birth, Phone Number, Workplace, Medical License.

Khối thứ hai trong mẫu là "Đồng ý cho NovoNordisk (Vietnam)", cùng cấu trúc.

## 5. Lưu ý pháp lý (Nghị định 13/2023/NĐ-CP về bảo vệ dữ liệu cá nhân)

1. **Tách riêng từng bên nhận**, không gộp chung "Hội và Nhất Anh" vào một câu.
2. **Chỉ liệt kê field thực sự gửi.** Không chép nguyên danh sách của Z-waka nếu DoctoTek gửi ít hơn.
3. **Nêu mục đích** chia sẻ và **kênh liên hệ** mà bên nhận được dùng.
4. **Checkbox mặc định KHÔNG được tick sẵn.** Mẫu Z-waka đang tick sẵn, không làm theo điểm này, vì đồng ý phải là hành động chủ động của bác sĩ.
5. **Lưu bằng chứng đồng ý:** user, hội thảo, bên nhận, thời điểm đồng ý, và phiên bản/nội dung điều khoản lúc đó.

## 6. Đề xuất cho dev

- **Cấu hình theo từng hội thảo** (admin nhập): danh sách bên nhận; với mỗi bên có tên, vai trò (người tổ chức / nhà tài trợ), danh sách field chia sẻ, mục đích, kênh liên hệ, cờ bắt buộc.
- **Bảng lưu đồng ý** (gợi ý): `user_id`, `event_id`, `recipient`, `consent_text_version` (hoặc snapshot nội dung), `agreed_at`, `ip` / `user_agent`.
- **Nếu đã đồng ý cho hội thảo đó rồi** (vd đã bấm Đăng ký, sau đó bấm Tham dự trực tuyến): không bắt đồng ý lại, trừ khi điều khoản đổi phiên bản.
- **Xuất dữ liệu cho Hội / Nhất Anh:** chỉ xuất bác sĩ đã đồng ý với đúng bên đó, và chỉ các field đã liệt kê.

## 7. Câu hỏi còn mở

- [ ] Chính xác những field nào sẽ gửi cho Hội? Cho Nhất Anh? Hai danh sách có giống nhau không?
- [ ] Tên pháp nhân đầy đủ của Nhất Anh để hiển thị.
- [ ] Cả 2 khối đều bắt buộc, hay có khối tuỳ chọn (không tick vẫn tham dự được)?
- [ ] Nhãn nút xác nhận trong pop-up ("Xác nhận đăng ký" / "Tham dự")?
- [ ] Bác sĩ có được rút lại đồng ý sau khi đăng ký không? Nếu có thì rút ở đâu?
- [ ] Làm cho app mobile, web, hay cả hai? Hạn hoàn thành để kịp 28/11?

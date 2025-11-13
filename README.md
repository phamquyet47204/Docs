# Giải thích chi tiết các biểu đồ UML

## 1. Use Case Diagram (`use-case-diagram.puml`)

### Tại sao vẽ như vậy:
- **Actors phân cấp**: User là base actor, Admin/Teacher/Student kế thừa từ User vì tất cả đều có chức năng đăng nhập/đăng xuất cơ bản
- **Include relationships**: 
  - Đăng nhập `<<include>>` Xác thực middleware và Quản lý phiên vì đây là các bước bắt buộc không thể tách rời
  - Xác thực middleware `<<include>>` Kiểm tra quyền vì mọi request đều phải qua 2 bước này
- **Không dùng extend**: Vì không có use case nào là optional/conditional từ góc nhìn người dùng
- **Phân quyền theo vai trò**: Mỗi actor có quyền khác nhau (Admin full access, Teacher manage courses, Student view only)

### Điểm quan trọng:
- Middleware là use case ẩn (hệ thống tự động thực hiện)
- Đặt lại mật khẩu độc lập, không cần đăng nhập

---

## 2. Sequence Diagram - Login (`sequence-diagram-login.puml`)

### Tại sao vẽ như vậy:
- **Thứ tự tương tác**: User → UI → Controller → Service → DB phản ánh kiến trúc layered (presentation → business → data)
- **Alt fragment**: Dùng để xử lý 2 luồng (success/failure) - tránh vẽ 2 diagram riêng
- **Session Manager riêng biệt**: Tách Session Manager ra vì đây là component quan trọng, có thể dùng Redis/cache
- **Password compare trong Service**: Logic nghiệp vụ nằm ở Service layer, không để Controller xử lý
- **Return token + user + role**: Client cần cả 3 thông tin để lưu state và hiển thị UI phù hợp

### Điểm quan trọng:
- Không trả về password trong response
- Lỗi đăng nhập luôn trả về "Invalid credentials" (không nói rõ username hay password sai - security)
- Session được lưu DB để có thể revoke từ xa

---

## 3. Sequence Diagram - Logout (`sequence-diagram-logout.puml`)

### Tại sao vẽ như vậy:
- **Middleware verify trước**: Đảm bảo chỉ user đã đăng nhập mới logout được (tránh spam)
- **Alt fragment cho session invalid**: Vẫn cho phép logout kể cả session hết hạn (UX tốt hơn)
- **clearLocalToken() ở client**: Client phải tự xóa token local ngay cả khi server lỗi
- **Delete session từ DB**: Hard delete để giải phóng storage và đảm bảo token không dùng lại được

### Điểm quan trọng:
- Logout luôn thành công từ góc nhìn user (kể cả lỗi server)
- Ghi log để audit trail
- Client redirect về login page

---

## 4. Sequence Diagram - Reset Password (`sequence-diagram-reset-password.puml`)

### Tại sao vẽ như vậy:
- **Chia 2 partition**: Rõ ràng 2 giai đoạn (request reset và thực hiện reset)
- **Alt "User không tồn tại" vẫn return success**: Tránh email enumeration attack (hacker không biết email nào có trong hệ thống)
- **Email Service riêng**: Có thể dùng AWS SES, SendGrid - tách biệt để dễ thay đổi
- **Token có expiry**: Bảo mật - link reset chỉ dùng được 1 lần trong thời gian ngắn
- **Xóa tất cả session sau reset**: Buộc user đăng nhập lại trên mọi thiết bị (security best practice)

### Điểm quan trọng:
- Token là UUID random, không đoán được
- Gửi email xác nhận sau khi đổi password thành công
- Link reset chỉ dùng 1 lần (one-time token)

---

## 5. Sequence Diagram - Middleware (`sequence-diagram-middleware.puml`)

### Tại sao vẽ như vậy:
- **Nested alt fragments**: Xử lý nhiều điều kiện tuần tự (token → session → permission)
- **Tách Authentication và Authorization**: 2 concerns khác nhau, có thể reuse riêng lẻ
- **Role Manager riêng**: Quản lý permissions phức tạp, có thể cache
- **attachUser(request)**: Middleware thêm user info vào request để Controller dùng
- **checkExpiry() và update expiry**: Sliding session - session tự động gia hạn khi user active

### Điểm quan trọng:
- Middleware chạy trước mọi protected endpoint
- 401 (Unauthenticated) vs 403 (Unauthorized) - khác nhau rõ ràng
- Ghi log mọi unauthorized access attempt

---

## 6. State Diagram - Session (`state-diagram-session.puml`)

### Tại sao vẽ như vậy:
- **NoSession là initial state**: User mới vào hệ thống chưa có session
- **Authenticating là transient state**: Trạng thái tạm thời trong quá trình xác thực
- **Active có self-loop**: Session refresh expiry mỗi khi có request hợp lệ
- **2 cách kết thúc Active**: Tự nhiên (timeout) hoặc chủ động (logout/force logout)
- **Expired và Terminated đều về NoSession**: Cleanup hoàn toàn trước khi kết thúc

### Điểm quan trọng:
- Session timeout sau 30 phút không hoạt động
- Admin có thể force logout user khác
- Mọi state đều có thể về NoSession (fail-safe)

---

## 7. State Diagram - Authentication (`state-diagram-authentication.puml`)

### Tại sao vẽ như vậy:
- **Unauthenticated là initial state**: Trạng thái mặc định
- **LoginAttempt là transient state**: Đang trong quá trình kiểm tra credentials
- **Locked state**: Bảo vệ khỏi brute-force attack (5 lần sai → khóa)
- **PasswordResetRequired**: Xử lý trường hợp password hết hạn (compliance requirement)
- **Multiple paths về Unauthenticated**: Linh hoạt - có thể unlock bằng nhiều cách

### Điểm quan trọng:
- Account tự unlock sau thời gian chờ (VD: 15 phút)
- Admin có thể unlock thủ công
- Password expired buộc phải reset

---

## 8. Activity Diagram - Login (`activity-diagram-login.puml`)

### Tại sao vẽ như vậy:
- **Rate limit check đầu tiên**: Ngăn DDoS/brute-force sớm nhất có thể
- **Nested if cho failed login**: Tăng counter → check limit → lock nếu cần (logic tuần tự)
- **Reset counter khi success**: Quan trọng - không để counter tích lũy mãi mãi
- **Multiple stop points**: Mỗi lỗi có exit point riêng (fail-fast)
- **Ghi log cuối cùng**: Audit trail cho mọi login thành công

### Điểm quan trọng:
- Không nói rõ lỗi gì (username/password) - security
- Rate limit theo IP hoặc username
- Token được tạo mới mỗi lần login

---

## 9. Activity Diagram - Logout (`activity-diagram-logout.puml`)

### Tại sao vẽ như vậy:
- **Đơn giản hơn login**: Logout ít logic phức tạp hơn
- **Verify token trước**: Đảm bảo request hợp lệ
- **Client xóa token dù sao**: Ngay cả khi server lỗi, client vẫn "logout" (UX)
- **Nested if cho session**: Có thể session đã bị xóa trước đó (race condition)

### Điểm quan trọng:
- Logout luôn thành công từ phía client
- Ghi log để biết ai logout khi nào
- Clear toàn bộ state ở client

---

## 10. Activity Diagram - Reset Password (`activity-diagram-reset-password.puml`)

### Tại sao vẽ như vậy:
- **2 partitions rõ ràng**: 2 flows hoàn toàn khác nhau (request vs reset)
- **Vẫn return success khi user không tồn tại**: Security - tránh email enumeration
- **Multiple validation checks**: Token exists → not expired → password valid (defense in depth)
- **Xóa tất cả session**: Buộc re-login trên mọi thiết bị (security best practice)
- **Gửi email xác nhận**: User biết password đã đổi (phát hiện unauthorized change)

### Điểm quan trọng:
- Token expiry 1 giờ (balance security vs UX)
- Token chỉ dùng 1 lần (one-time use)
- Validate password strength trước khi lưu

---

## 11. Activity Diagram - Middleware (`activity-diagram-middleware.puml`)

### Tại sao vẽ như vậy:
- **2 partitions**: Tách rõ Authentication (ai bạn là?) vs Authorization (bạn được làm gì?)
- **Sequential checks**: Token → Session → Expiry → Permissions (fail-fast, check nhanh trước)
- **Update session expiry**: Sliding window - session tự gia hạn khi user active
- **Attach user vào request**: Controller cần biết user hiện tại là ai
- **Ghi log cả success và failure**: Audit trail đầy đủ

### Điểm quan trọng:
- Middleware chạy cho mọi protected endpoint
- 401 vs 403 - phân biệt rõ ràng
- Session expiry được refresh tự động

---

## 12. Activity Diagram - Role-Based Access (`activity-diagram-role-based-access.puml`)

### Tại sao vẽ như vậy:
- **Switch-case style với if-elseif**: Xử lý 3 roles khác nhau rõ ràng
- **Nested if trong mỗi role**: Mỗi role có logic phân quyền riêng
- **ADMIN đơn giản nhất**: Full access, không cần check thêm
- **TEACHER có ownership check**: Teacher chỉ edit được course của mình
- **STUDENT chỉ view own data**: Bảo mật - không xem được data của student khác
- **Default deny**: Mọi trường hợp không match đều bị từ chối

### Điểm quan trọng:
- Principle of least privilege - chỉ cho quyền tối thiểu cần thiết
- Ownership check cho Teacher (data isolation)
- Student không có quyền write gì cả

---

## Nguyên tắc thiết kế chung:

### Security First:
- Fail-safe: Mặc định deny, phải explicitly allow
- Defense in depth: Nhiều lớp kiểm tra
- Audit logging: Ghi log mọi hành động quan trọng
- Rate limiting: Chống brute-force
- Token expiry: Giới hạn thời gian sử dụng

### UX Balance:
- Logout luôn thành công (client-side)
- Error messages không leak information
- Session auto-refresh khi active
- Password reset user-friendly

### Architecture:
- Layered architecture: UI → Controller → Service → DB
- Separation of concerns: Auth vs Authorization
- Single responsibility: Mỗi component một nhiệm vụ
- Stateless middleware: Dễ scale horizontal

### Best Practices:
- Never store plain password
- Use UUID for tokens
- One-time use tokens
- Sliding session window
- Clear separation of roles

# UML Diagrams - Authentication & Authorization System

Các biểu đồ UML cho hệ thống xác thực và phân quyền của University Management System.

## Cách sử dụng

Các file `.puml` sử dụng PlantUML syntax. Bạn có thể render chúng bằng:

### 1. Online
- Truy cập: https://www.plantuml.com/plantuml/uml/
- Copy nội dung file `.puml` và paste vào

### 2. VS Code Extension
- Cài đặt extension: "PlantUML" by jebbs
- Mở file `.puml`
- Nhấn `Alt+D` để preview

### 3. Command Line
```bash
npm install -g node-plantuml
puml generate diagrams/*.puml -o output/
```

## Danh sách Diagrams

### Use Case Diagram
- **File**: `use-case-diagram.puml`
- **Mô tả**: Tổng quan các use case cho hệ thống xác thực, bao gồm đăng nhập, đăng xuất, đặt lại mật khẩu, và phân quyền theo vai trò

### Sequence Diagrams
1. **Login Flow** (`sequence-diagram-login.puml`)
   - Luồng đăng nhập từ UI đến database
   - Xử lý xác thực và tạo session

2. **Logout Flow** (`sequence-diagram-logout.puml`)
   - Luồng đăng xuất và hủy session
   - Xử lý token invalidation

3. **Password Reset** (`sequence-diagram-reset-password.puml`)
   - Yêu cầu đặt lại mật khẩu qua email
   - Xác thực token và cập nhật mật khẩu mới

4. **Middleware Authentication** (`sequence-diagram-middleware.puml`)
   - Luồng xác thực qua middleware
   - Kiểm tra quyền truy cập dựa trên role

### State Diagrams
1. **Session States** (`state-diagram-session.puml`)
   - Các trạng thái của session: Active, Expired, Terminated
   - Chuyển đổi giữa các trạng thái

2. **Authentication States** (`state-diagram-authentication.puml`)
   - Trạng thái xác thực: Unauthenticated, Authenticated, Locked
   - Xử lý password reset và account locking

### Activity Diagrams
1. **Login Process** (`activity-diagram-login.puml`)
   - Quy trình đăng nhập chi tiết
   - Xử lý rate limiting và account locking

2. **Logout Process** (`activity-diagram-logout.puml`)
   - Quy trình đăng xuất
   - Cleanup session và token

3. **Password Reset** (`activity-diagram-reset-password.puml`)
   - Quy trình yêu cầu và thực hiện reset password
   - Validation và security measures

4. **Middleware Flow** (`activity-diagram-middleware.puml`)
   - Luồng xử lý authentication middleware
   - Authorization checking

5. **Role-Based Access** (`activity-diagram-role-based-access.puml`)
   - Phân quyền theo vai trò: Admin, Teacher, Student
   - Kiểm tra permissions cho từng resource

## Chức năng được mô tả

✅ Đăng nhập/Đăng xuất
✅ Quản lý phiên (Session Management)
✅ Đặt lại mật khẩu (Password Reset)
✅ Xác thực qua Middleware
✅ Phân quyền dựa trên vai trò (Role-Based Access Control)

## Vai trò trong hệ thống

- **Admin**: Full access, quản lý users và roles
- **Teacher**: Quản lý courses, grades, attendance
- **Student**: Xem thông tin cá nhân, courses, grades

## Security Features

- Password hashing
- Session token management
- Rate limiting
- Account locking sau nhiều lần đăng nhập sai
- Token expiry và refresh
- Email verification cho password reset
- Audit logging

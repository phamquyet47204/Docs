# Báo cáo Triển khai Hệ thống Xác thực

## Tổng quan

Hệ thống xác thực được triển khai cho University Management System với kiến trúc phân tầng, hỗ trợ phân quyền dựa trên vai trò (RBAC) và quản lý phiên làm việc an toàn.

**Công nghệ:**
- Backend: Django 4.x + Django Session Framework
- Frontend: React + Vite
- Database: SQLite
- Authentication: Session-based

---

## 1. Triển khai Chế độ Đăng nhập/Đăng xuất

### 1.1. Kiến trúc Tổng quan

```
┌─────────────────────────────────────────────────────────────┐
│                      AUTHENTICATION FLOW                     │
└─────────────────────────────────────────────────────────────┘

Frontend (React)              Backend (Django)
┌──────────────┐             ┌──────────────────┐
│              │             │                  │
│  LoginPage   │────────────▶│  login_view()    │
│              │  POST       │                  │
└──────────────┘  /api/auth  └──────────────────┘
                  /login/              │
                                       ▼
                              ┌──────────────────┐
                              │ Authentication   │
                              │    Service       │
                              └──────────────────┘
                                       │
                                       ▼
                              ┌──────────────────┐
                              │   Create         │
                              │   Session        │
                              └──────────────────┘
                                       │
                                       ▼
                              ┌──────────────────┐
                              │  Return          │
                              │  session_key     │
                              └──────────────────┘
```

### 1.2. Luồng Đăng nhập

**Bước 1: User nhập thông tin**
- Username
- Password

**Bước 2: Frontend gửi request**
- Endpoint: `POST /api/auth/login/`
- Body: `{username, password}`

**Bước 3: Backend xác thực**
- Kiểm tra username/password
- Kiểm tra status = 'active'
- Cập nhật last_login

**Bước 4: Tạo session**
- Tạo session key
- Set expiry: 1 giờ
- Lưu vào database

**Bước 5: Trả về response**
```json
{
  "message": "Login successful",
  "session_key": "abc123...",
  "user": {
    "username": "student01",
    "email": "student@sgu.edu.vn",
    "full_name": "Nguyễn Văn A",
    "user_type": "student"
  }
}
```

**Bước 6: Frontend lưu thông tin**
- Lưu session_key vào localStorage
- Lưu user info vào AuthStorage
- Redirect đến trang chủ

### 1.3. Luồng Đăng xuất

```
User Click Logout
       │
       ▼
Frontend gửi POST /api/auth/logout/
       │
       ▼
Backend xóa session khỏi DB
       │
       ▼
Frontend xóa localStorage
       │
       ▼
Redirect về Login Page
```

### 1.4. Login theo Vai trò

Hệ thống hỗ trợ 3 endpoint login riêng:

| Endpoint | Vai trò | Mô tả |
|----------|---------|-------|
| `/api/auth/admin/login/` | Admin | Chỉ admin đăng nhập được |
| `/api/auth/teacher/login/` | Teacher | Chỉ giáo viên đăng nhập được |
| `/api/auth/student/login/` | Student | Chỉ sinh viên đăng nhập được |

**Lợi ích:**
- Tách biệt rõ ràng từng vai trò
- Dễ quản lý và audit
- Có thể áp dụng rate limiting riêng

---

## 2. Tạo Quyền dựa trên Vai trò (RBAC)

### 2.1. Mô hình Phân quyền

```
┌─────────────────────────────────────────────────────────────┐
│                    ROLE-BASED ACCESS CONTROL                 │
└─────────────────────────────────────────────────────────────┘

┌──────────────┐
│    ADMIN     │  ◄── Full Access
└──────────────┘
       │
       ├─ manage_users
       ├─ manage_departments
       ├─ manage_subjects
       ├─ generate_reports
       ├─ approve_documents
       └─ manage_tuition

┌──────────────┐
│   TEACHER    │  ◄── Limited Access
└──────────────┘
       │
       ├─ view_assigned_classes
       ├─ input_grades
       ├─ export_grades
       ├─ take_attendance
       └─ view_students

┌──────────────┐
│   STUDENT    │  ◄── Read Only
└──────────────┘
       │
       ├─ view_grades
       ├─ register_course
       ├─ view_schedule
       ├─ request_document
       └─ view_profile
```

### 2.2. Permission Matrix

| Chức năng | Admin | Teacher | Student |
|-----------|-------|---------|---------|
| Quản lý users | ✅ | ❌ | ❌ |
| Quản lý khoa | ✅ | ❌ | ❌ |
| Nhập điểm | ✅ | ✅ | ❌ |
| Xem điểm | ✅ | ✅ | ✅ (own) |
| Đăng ký môn học | ✅ | ❌ | ✅ |
| Duyệt tài liệu | ✅ | ❌ | ❌ |
| Xem lịch học | ✅ | ✅ | ✅ (own) |

### 2.3. Cách hoạt động

**1. Decorator-based Permission:**
```
@require_permission('input_grades')
def input_grades(request):
    # Chỉ user có permission 'input_grades' mới truy cập được
```

**2. Role-based Permission:**
```
@require_role('admin', 'teacher')
def manage_classes(request):
    # Chỉ admin và teacher mới truy cập được
```

**3. Middleware-based Permission:**
- Kiểm tra role theo path
- `/api/admin/*` → Chỉ admin
- `/api/teacher/*` → Chỉ teacher
- `/api/student/*` → Chỉ student

---

## 3. Thiết lập Quản lý Phiên

### 3.1. Middleware Chain

```
┌─────────────────────────────────────────────────────────────┐
│                      MIDDLEWARE FLOW                         │
└─────────────────────────────────────────────────────────────┘

Request
   │
   ▼
┌──────────────────────────┐
│ AuthenticationMiddleware │  ◄── Kiểm tra X-Session-Key
└──────────────────────────┘
   │
   ├─ Có session_key? ──NO──▶ 401 Unauthorized
   │
   ├─ Session hợp lệ? ──NO──▶ 401 Invalid Session
   │
   └─ YES
       │
       ▼
┌──────────────────────────┐
│ SessionValidation        │  ◄── Kiểm tra expiry
│ Middleware               │
└──────────────────────────┘
   │
   ├─ Session hết hạn? ──YES──▶ 401 Session Expired
   │
   └─ NO
       │
       ▼
┌──────────────────────────┐
│ RolePermission           │  ◄── Kiểm tra role theo path
│ Middleware               │
└──────────────────────────┘
   │
   ├─ Đúng role? ──NO──▶ 403 Forbidden
   │
   └─ YES
       │
       ▼
   View/Controller
```

### 3.2. Session Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                      SESSION LIFECYCLE                       │
└─────────────────────────────────────────────────────────────┘

Login Success
     │
     ▼
Create Session
     │
     ├─ session_key: random UUID
     ├─ expire_date: now + 1 hour
     └─ user_id: current user
     │
     ▼
Save to Database
     │
     ▼
┌─────────────────┐
│  Active Session │ ◄──┐
└─────────────────┘    │
     │                 │
     ├─ Request ───────┘  (Refresh expiry)
     │
     ├─ Timeout (1h) ──▶ Expired
     │
     └─ Logout ────────▶ Terminated
```

### 3.3. Exempt Paths

Các endpoint không cần xác thực:
- `/api/auth/login/`
- `/api/auth/logout/`
- `/api/auth/request-reset/`
- `/api/auth/reset-password/`
- `/admin/`
- `/static/`

---

## 4. Chức năng Đặt lại Mật khẩu

### 4.1. Luồng Reset Password

```
┌─────────────────────────────────────────────────────────────┐
│                  PASSWORD RESET FLOW                         │
└─────────────────────────────────────────────────────────────┘

GIAI ĐOẠN 1: YÊU CẦU RESET
───────────────────────────

User nhập email
     │
     ▼
POST /api/auth/request-reset/
     │
     ▼
Tìm user theo email
     │
     ├─ Tồn tại? ──YES──▶ Tạo token (UUID)
     │                    │
     │                    ├─ Expiry: 24h
     │                    ├─ is_used: false
     │                    └─ Lưu vào DB
     │                         │
     │                         ▼
     │                    Gửi email với link
     │
     └─ NO ──▶ Vẫn trả về success
               (Tránh email enumeration)


GIAI ĐOẠN 2: THỰC HIỆN RESET
─────────────────────────────

User click link trong email
     │
     ▼
Nhập password mới
     │
     ▼
POST /api/auth/reset-password/
     │
     ▼
Verify token
     │
     ├─ Token tồn tại? ──NO──▶ 400 Invalid Token
     │
     ├─ Token hết hạn? ──YES──▶ 400 Expired Token
     │
     └─ Valid
         │
         ▼
    Hash password mới
         │
         ▼
    Update password trong DB
         │
         ▼
    Đánh dấu token is_used = true
         │
         ▼
    Xóa TẤT CẢ session của user
         │
         ▼
    Return success
```

### 4.2. Token Security

**Đặc điểm:**
- Token: UUID random 32 bytes
- Expiry: 24 giờ
- One-time use: is_used flag
- Không đoán được

**Bảo mật:**
- Email enumeration prevention
- Token expiry
- Xóa tất cả session sau reset
- Gửi email xác nhận

---

## 5. Kiểm tra Luồng Xác thực

### 5.1. Authentication Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│              COMPLETE AUTHENTICATION FLOW                    │
└─────────────────────────────────────────────────────────────┘

┌──────────┐
│  Client  │
└────┬─────┘
     │
     │ 1. POST /api/auth/login/
     │    {username, password}
     ▼
┌──────────────────┐
│  Auth Controller │
└────┬─────────────┘
     │
     │ 2. authenticate_user()
     ▼
┌──────────────────┐
│  Auth Service    │
└────┬─────────────┘
     │
     │ 3. Check credentials
     ▼
┌──────────────────┐
│    Database      │
└────┬─────────────┘
     │
     │ 4. User found & active
     ▼
┌──────────────────┐
│ Session Manager  │
└────┬─────────────┘
     │
     │ 5. Create session
     │    - Generate session_key
     │    - Set expiry: 1h
     │    - Save to DB
     ▼
┌──────────────────┐
│  Return Response │
│  {session_key,   │
│   user, role}    │
└────┬─────────────┘
     │
     │ 6. Save to localStorage
     ▼
┌──────────────────┐
│  Client Storage  │
│  - session_key   │
│  - user info     │
└──────────────────┘
```

### 5.2. Protected Request Flow

```
┌─────────────────────────────────────────────────────────────┐
│              PROTECTED REQUEST FLOW                          │
└─────────────────────────────────────────────────────────────┘

Client Request
     │
     │ Header: X-Session-Key: abc123...
     ▼
┌──────────────────────────┐
│ AuthenticationMiddleware │
└──────────────────────────┘
     │
     ├─ Extract session_key
     ├─ Find session in DB
     ├─ Check expiry
     └─ Attach user to request
     │
     ▼
┌──────────────────────────┐
│ SessionValidation        │
└──────────────────────────┘
     │
     ├─ Validate session
     └─ Check expiry
     │
     ▼
┌──────────────────────────┐
│ RolePermission           │
└──────────────────────────┘
     │
     ├─ Check role vs path
     └─ Check permissions
     │
     ▼
┌──────────────────────────┐
│ View/Controller          │
└──────────────────────────┘
     │
     ├─ Process request
     └─ Return response
```

### 5.3. Error Handling

| Status Code | Ý nghĩa | Xử lý |
|-------------|---------|-------|
| 401 Unauthorized | Chưa đăng nhập / Session invalid | Redirect login |
| 403 Forbidden | Đã đăng nhập nhưng không đủ quyền | Show error |
| 429 Too Many Requests | Rate limit exceeded | Wait & retry |

---

## 6. Frontend SGU Implementation

### 6.1. Cấu trúc Frontend

```
SGU/
├── src/
│   ├── components/
│   │   ├── ui/              # UI components
│   │   ├── Layout.jsx       # Main layout
│   │   └── Sidebar.jsx      # Navigation
│   │
│   ├── pages/
│   │   ├── LoginPage.jsx    # Login form
│   │   ├── HomePage.jsx     # Dashboard
│   │   └── ProfilePage.jsx  # User profile
│   │
│   ├── services/
│   │   ├── authService.js   # Authentication logic
│   │   └── apiService.js    # API calls
│   │
│   ├── types/
│   │   └── user.js          # AuthStorage
│   │
│   └── config/
│       └── api.js           # API endpoints
```

### 6.2. Authentication Flow trong React

```
┌─────────────────────────────────────────────────────────────┐
│                  REACT AUTHENTICATION                        │
└─────────────────────────────────────────────────────────────┘

App.jsx
   │
   ├─ useEffect: Check authentication
   │     │
   │     ├─ authService.isAuthenticated()?
   │     │     │
   │     │     ├─ YES ──▶ authService.checkSession()
   │     │     │              │
   │     │     │              ├─ Valid ──▶ Set user state
   │     │     │              └─ Invalid ──▶ Clear & redirect
   │     │     │
   │     │     └─ NO ──▶ Show LoginPage
   │     │
   │     └─ setIsLoading(false)
   │
   ├─ if (!user) ──▶ <LoginPage />
   │
   └─ else ──▶ <Layout>
                   <Routes>
                     <Route path="/" element={<HomePage />} />
                     <Route path="/profile" element={<ProfilePage />} />
                   </Routes>
                 </Layout>
```

### 6.3. AuthStorage

**Chức năng:**
- `setCurrentUser(user)`: Lưu user vào localStorage
- `getCurrentUser()`: Lấy user từ localStorage
- `logout()`: Xóa user khỏi localStorage
- `isLoggedIn()`: Kiểm tra đã login chưa

**Storage Keys:**
- `sgu_current_user`: User info (JSON)
- `sgu_session_key`: Session key (string)

---

## 7. Security Features

### 7.1. Implemented Features

✅ **Authentication:**
- Session-based authentication
- Session timeout: 1 giờ
- Secure session storage

✅ **Authorization:**
- Role-based access control (RBAC)
- Permission-based authorization
- Path-based role checking

✅ **Password Security:**
- Password hashing (Django default)
- Password reset with token
- Token expiry: 24 giờ
- One-time use token

✅ **Session Management:**
- Middleware validation
- Auto-expiry
- Force logout capability

✅ **Attack Prevention:**
- Email enumeration prevention
- Brute-force protection (account locking)
- CSRF protection
- Rate limiting ready

### 7.2. Security Best Practices

**1. Fail-safe Design:**
- Mặc định deny access
- Phải explicitly allow

**2. Defense in Depth:**
- Multiple layers of validation
- Middleware chain

**3. Audit Logging:**
- Log mọi login/logout
- Log unauthorized access
- Track session activity

**4. Principle of Least Privilege:**
- Mỗi role chỉ có quyền tối thiểu
- Student: Read only own data
- Teacher: Manage own classes
- Admin: Full access

---

## 8. Kết luận

### 8.1. Tính năng đã triển khai

✅ **Đăng nhập/Đăng xuất**
- Session-based authentication
- Login theo vai trò
- Logout an toàn

✅ **RBAC (Role-Based Access Control)**
- 3 roles: Admin, Teacher, Student
- Permission matrix rõ ràng
- Decorator và Middleware support

✅ **Quản lý Phiên**
- Middleware chain validation
- Session timeout: 1 giờ
- Auto-expiry và refresh

✅ **Đặt lại Mật khẩu**
- Token-based reset
- Email verification
- Secure và user-friendly

✅ **Luồng Xác thực**
- Middleware authentication
- Permission checking
- Error handling

✅ **Frontend Integration**
- React components
- AuthService
- Protected routes

### 8.2. Ưu điểm

- **Bảo mật cao**: Multiple layers of security
- **Dễ mở rộng**: Modular architecture
- **Maintainable**: Clean code structure
- **User-friendly**: Good UX design

### 8.3. Cải tiến trong tương lai

- [ ] Rate limiting implementation
- [ ] Two-factor authentication (2FA)
- [ ] Email service integration
- [ ] Redis session storage
- [ ] JWT token option
- [ ] Refresh token mechanism
- [ ] OAuth2 integration

# BLUEPRINT API – CRS MICROSERVICES

Blueprint API mô tả tổng thể các API của hệ thống CRS Microservices, bao gồm
auth-service, course-service, registration-service, API Gateway và các API
giao tiếp nội bộ giữa các service.

## 1. Kiến trúc và cổng của hệ thống

| Thành phần | Cổng | Vai trò |
| :--- | :--- | :--- |
| React Frontend | 5173 | Giao diện người dùng |
| API Gateway | 8085 | Điểm truy cập duy nhất từ Frontend |
| auth-service | 8094 | Đăng nhập, xác thực và cấp JWT |
| course-service | 8900 | Quản lý môn học |
| registration-service | 8093 | Quản lý đăng ký học phần |
| MySQL | 3306 | Lưu trữ dữ liệu |

Frontend không gọi trực tiếp các service backend mà gửi request thông qua
API Gateway tại:

http://localhost:8085

---

## 2. auth-service

Cổng:

8094

Tiền tố khi qua Gateway:

/api/auth

| Method | Endpoint nội bộ | Endpoint qua Gateway | Mô tả | Yêu cầu |
| :--- | :--- | :--- | :--- | :--- |
| POST | /auth/login | /api/auth/login | Đăng nhập và trả về JWT | Public |
| POST | /auth/register | /api/auth/register | Đăng ký tài khoản nếu được triển khai | Public |

Ví dụ Frontend đăng nhập:

POST /api/auth/login

Request được chuyển theo luồng:

Frontend
→ API Gateway :8085
→ auth-service :8094
→ kiểm tra tài khoản
→ tạo JWT
→ trả JWT về Frontend

---

## 3. course-service

Cổng:

8900

Tiền tố khi qua Gateway:

/api/courses

| Method | Endpoint nội bộ | Endpoint qua Gateway | Mô tả | Yêu cầu |
| :--- | :--- | :--- | :--- | :--- |
| GET | /courses | /api/courses | Danh sách môn học, hỗ trợ search và phân trang | Public |
| GET | /courses/{id} | /api/courses/{id} | Xem chi tiết một môn học | Public |
| POST | /courses | /api/courses | Thêm môn học | ADMIN |
| PUT | /courses/{id} | /api/courses/{id} | Sửa môn học | ADMIN |
| DELETE | /courses/{id} | /api/courses/{id} | Xóa môn học | ADMIN |

Thông tin Course hiện sử dụng gồm:

- id
- tenMonHoc
- soTinChi
- soChoToiDa
- soChoConLai

Ví dụ dữ liệu Course hiện đã test thành công:

{
"id": 1,
"tenMonHoc": "Lap trinh Java",
"soTinChi": 3,
"soChoToiDa": 1,
"soChoConLai": 1
}

Frontend đã gọi thành công:

GET /api/courses

theo luồng:

React :5173
→ API Gateway :8085
→ course-service :8900
→ course_db
→ trả danh sách Course về React

---

## 4. API nội bộ của course-service

Các API này phục vụ giao tiếp giữa các microservice.

KHÔNG expose cho Frontend qua API Gateway.

| Method | Endpoint | Mô tả |
| :--- | :--- | :--- |
| PATCH | /internal/courses/{id}/reserve-seat | Kiểm tra số chỗ còn lại và trừ 1 chỗ |
| PATCH | /internal/courses/{id}/release-seat | Hoàn trả 1 chỗ khi hủy đăng ký |

### reserve-seat

registration-service gọi:

PATCH /internal/courses/{id}/reserve-seat

Mục đích:

1. Kiểm tra Course tồn tại.
2. Kiểm tra soChoConLai > 0.
3. Nếu còn chỗ thì giảm soChoConLai đi 1.
4. Nếu hết chỗ thì từ chối đăng ký.

### release-seat

registration-service gọi:

PATCH /internal/courses/{id}/release-seat

Mục đích:

1. Kiểm tra Course tồn tại.
2. Tăng soChoConLai lên 1.
3. Không cho vượt quá soChoToiDa.

---

## 5. registration-service

Cổng:

8093

Tiền tố khi qua Gateway:

/api/registrations

| Method | Endpoint nội bộ | Endpoint qua Gateway | Mô tả | Yêu cầu |
| :--- | :--- | :--- | :--- | :--- |
| POST | /registrations | /api/registrations | Đăng ký học phần | STUDENT |
| GET | /registrations/my | /api/registrations/my | Xem danh sách đăng ký của sinh viên hiện tại | STUDENT |
| DELETE | /registrations/{id} | /api/registrations/{id} | Hủy đăng ký học phần | STUDENT / ADMIN |

Khi sinh viên đăng ký học phần:

POST /api/registrations

Request đi theo luồng:

Frontend
→ Gateway :8085
→ registration-service :8093
→ course-service :8900
→ reserve-seat
→ lưu Registration
→ trả kết quả về Frontend

registration-service không tự ý thay đổi số chỗ của Course trong database
course-service.

---

## 6. API Gateway

Cổng:

8085

API Gateway là điểm truy cập duy nhất của React Frontend tới hệ thống backend.

Các route hiện tại:

| Service | Endpoint nội bộ | Endpoint qua Gateway |
| :--- | :--- | :--- |
| auth-service | /auth/login | /api/auth/login |
| auth-service | /auth/register | /api/auth/register |
| course-service | /courses | /api/courses |
| course-service | /courses/{id} | /api/courses/{id} |
| registration-service | /registrations | /api/registrations |
| registration-service | /registrations/my | /api/registrations/my |
| registration-service | /registrations/{id} | /api/registrations/{id} |
| course-service | /internal/courses/{id}/reserve-seat | KHÔNG expose |
| course-service | /internal/courses/{id}/release-seat | KHÔNG expose |

Gateway hiện chạy tại:

http://localhost:8085

Frontend cấu hình:

VITE_API_BASE_URL=http://localhost:8085

---

## 7. Authentication & Authorization

Hệ thống sử dụng JWT để xác thực người dùng.

Hai role chính:

- ADMIN
- STUDENT

| API | Quyền / Cơ chế |
| :--- | :--- |
| POST /api/auth/login | Public |
| POST /api/auth/register | Public |
| GET /api/courses | Public |
| GET /api/courses/{id} | Public |
| POST /api/courses | JWT + ADMIN |
| PUT /api/courses/{id} | JWT + ADMIN |
| DELETE /api/courses/{id} | JWT + ADMIN |
| POST /api/registrations | JWT + STUDENT |
| GET /api/registrations/my | JWT + STUDENT |
| DELETE /api/registrations/{id} | JWT + STUDENT hoặc ADMIN |
| /internal/courses/** | Chỉ dùng cho giao tiếp nội bộ giữa service |

JWT được gửi trong Header:

Authorization: Bearer <token>

---

## 8. Giao tiếp liên-service

| Service gọi | Service được gọi | Method | Endpoint | Mục đích |
| :--- | :--- | :--- | :--- | :--- |
| registration-service | course-service | PATCH | /internal/courses/{id}/reserve-seat | Giữ chỗ trước khi lưu Registration |
| registration-service | course-service | PATCH | /internal/courses/{id}/release-seat | Hoàn trả chỗ khi hủy đăng ký |

Ví dụ luồng đăng ký:

Student
→ React
→ Gateway
→ registration-service
→ course-service /reserve-seat
→ MySQL
→ registration-service lưu Registration
→ Gateway
→ React

---

## 9. Partner API

Hệ thống có API dành cho Partner sử dụng API Key.

Endpoint qua Gateway:

GET /api/public/courses

Luồng:

Partner
→ API Gateway :8085
→ kiểm tra API Key
→ course-service :8900
→ trả danh sách Course

API Key được cấu hình phía Gateway và không sử dụng JWT của
ADMIN/STUDENT.

---

## 10. CORS

Frontend chạy tại:

http://localhost:5173

Gateway chạy tại:

http://localhost:8085

CORS được cấu hình tập trung tại API Gateway.

Frontend không cần cấu hình CORS riêng tại từng microservice.

Luồng đã kiểm thử thành công:

React :5173
→ Axios
→ API Gateway :8085
→ course-service :8900
→ MySQL
→ React hiển thị dữ liệu Course

---

## 11. Luồng chính của hệ thống

| Luồng | Các bước |
| :--- | :--- |
| Đăng nhập | Frontend → Gateway → auth-service → JWT |
| Xem Course | Frontend → Gateway → course-service |
| Thêm Course | Frontend → Gateway → JWT ADMIN → course-service |
| Sửa Course | Frontend → Gateway → JWT ADMIN → course-service |
| Xóa Course | Frontend → Gateway → JWT ADMIN → course-service |
| Đăng ký học phần | Frontend → Gateway → registration-service → course-service reserve-seat → lưu Registration |
| Hủy đăng ký | Frontend → Gateway → registration-service → course-service release-seat → cập nhật Registration |
| Partner xem Course | Partner → Gateway → API Key → course-service |

---

## 12. Tổng quan kiến trúc

                         +----------------+
                         | React Frontend |
                         |     :5173      |
                         +-------+--------+
                                 |
                                 | HTTP / Axios
                                 v
                         +----------------+
                         |  API Gateway   |
                         |     :8085      |
                         +-------+--------+
                                 |
                +----------------+----------------+
                |                |                |
                v                v                v
        +---------------+ +---------------+ +----------------------+
        | auth-service  | |course-service | |registration-service  |
        |     :8094     | |     :8900     | |        :8093         |
        +---------------+ +-------+-------+ +----------+-----------+
                                  ^                     |
                                  |                     |
                                  +---------------------+
                                    Inter-service API

---

## 13. Tiến độ triển khai

- Buổi 1: Thiết kế tổng thể hệ thống Microservices và Blueprint API.
- Buổi 2: Hoàn thiện CRUD Course.
- Buổi 3: Bổ sung search, phân trang, registration-service và giao tiếp liên-service.
- Buổi 4: Bổ sung API Gateway, JWT, phân quyền ADMIN/STUDENT và API Key Partner.
- Buổi 5: Xây dựng React TypeScript Frontend, Axios và CORS; Frontend chỉ gọi backend thông qua API Gateway.

## 14. Trạng thái hiện tại

Các thành phần hiện tại:

- api-gateway: port 8085.
- auth-service: port 8094.
- course-service: port 8900.
- registration-service: port 8093.
- React Frontend: port 5173.
- MySQL: port 3306.

Đã kiểm thử thành công luồng:

React Frontend
→ API Gateway
→ course-service
→ MySQL
→ trả dữ liệu Course về Frontend.

Dữ liệu kiểm thử hiện tại:

Lap trinh Java
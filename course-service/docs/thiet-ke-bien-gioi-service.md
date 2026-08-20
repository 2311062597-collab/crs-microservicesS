# THIẾT KẾ BIÊN GIỚI SERVICE – CRS MICROSERVICES

## 1. Danh sách Service

Hệ thống CRS Microservices hiện tại gồm API Gateway, 3 backend service độc lập và React Frontend.

| Service | Cổng | Database | Trách nhiệm chính |
| :--- | :--- | :--- | :--- |
| api-gateway | 8085 | Không có DB | Điểm vào duy nhất, định tuyến request, xác thực sơ bộ, API Key và CORS |
| auth-service | 8094 | auth_db | Quản lý User, đăng nhập, sinh JWT và xác định quyền ADMIN/STUDENT |
| course-service | 8900 | course_db | Quản lý Course, tìm kiếm, phân trang và quản lý số chỗ |
| registration-service | 8093 | registration_db | Quản lý Registration và gọi course-service khi đăng ký/hủy học phần |
| crs-frontend | 5173 | Không có DB | Giao diện React TypeScript, gọi backend thông qua API Gateway |

---

## 2. Nguyên tắc sở hữu dữ liệu (Data Ownership)

- Mỗi backend service có DATABASE RIÊNG.
- Không service nào được truy cập trực tiếp database của service khác.
- Muốn lấy hoặc thay đổi dữ liệu thuộc service khác phải gọi REST API.
- `registration-service` KHÔNG có bảng Course, chỉ lưu `courseId`.
- `courseId` trong Registration chỉ là giá trị tham chiếu, không phải khóa ngoại sang `course_db`.
- `course-service` sở hữu dữ liệu Course và số chỗ.
- `registration-service` sở hữu dữ liệu Registration.
- `auth-service` sở hữu User và dữ liệu phục vụ xác thực.
- `api-gateway` không sở hữu database nghiệp vụ.
- Frontend không truy cập database.

### Data Ownership cụ thể

| Dữ liệu | Service sở hữu | Service khác truy cập bằng |
| :--- | :--- | :--- |
| User | auth-service | REST API / JWT |
| Thông tin xác thực | auth-service | JWT |
| Role ADMIN/STUDENT | auth-service | JWT |
| Course | course-service | REST API |
| tenMonHoc | course-service | REST API |
| soTinChi | course-service | REST API |
| soChoToiDa | course-service | REST API |
| soChoConLai | course-service | REST API |
| Registration | registration-service | REST API |
| studentId trong Registration | registration-service | Giá trị tham chiếu |
| courseId trong Registration | registration-service | Giá trị tham chiếu, không phải FK DB |

---

## 3. Bảng định tuyến API Gateway

API Gateway thực tế của hệ thống chạy tại:

```text
http://localhost:8085
```

| Route | Forward tới | Ghi chú |
| :--- | :--- | :--- |
| /api/auth/** | http://localhost:8094 | Login public, dùng cho xác thực |
| /api/courses | http://localhost:8900 | GET public, thao tác thay đổi cần quyền ADMIN |
| /api/courses/** | http://localhost:8900 | Chi tiết và CRUD Course |
| /api/registrations | http://localhost:8093 | Đăng ký học phần |
| /api/registrations/** | http://localhost:8093 | Các chức năng Registration |
| /api/public/courses | http://localhost:8900 | API dành cho Partner, sử dụng API Key |

Gateway thực hiện `RewritePath` trước khi chuyển request đến service tương ứng.

Ví dụ:

```text
Frontend:
GET http://localhost:8085/api/courses

        ↓ Gateway RewritePath

course-service:
GET http://localhost:8900/courses
```

---

## 4. Biên giới API công khai và API nội bộ

### 4.1. API công khai qua Gateway

| Service | Endpoint nội bộ | Endpoint qua Gateway | Được gọi bởi |
| :--- | :--- | :--- | :--- |
| auth-service | POST /auth/login | POST /api/auth/login | Frontend / Client |
| auth-service | POST /auth/register | POST /api/auth/register | Frontend / Client |
| course-service | GET /courses | GET /api/courses | Frontend / Client |
| course-service | GET /courses/{id} | GET /api/courses/{id} | Frontend / Client |
| course-service | POST /courses | POST /api/courses | ADMIN |
| course-service | PUT /courses/{id} | PUT /api/courses/{id} | ADMIN |
| course-service | DELETE /courses/{id} | DELETE /api/courses/{id} | ADMIN |
| registration-service | POST /registrations | POST /api/registrations | STUDENT |
| registration-service | GET /registrations/my | GET /api/registrations/my | STUDENT |
| registration-service | DELETE /registrations/{id} | DELETE /api/registrations/{id} | STUDENT / ADMIN |

### 4.2. API nội bộ

Các API dưới đây KHÔNG expose qua API Gateway.

| Service | Endpoint | Được gọi bởi | Qua Gateway |
| :--- | :--- | :--- | :--- |
| course-service | PATCH /internal/courses/{id}/reserve-seat | registration-service | KHÔNG |
| course-service | PATCH /internal/courses/{id}/release-seat | registration-service | KHÔNG |

Luồng gọi trực tiếp:

```text
registration-service :8093
          |
          | REST API nội bộ
          v
course-service :8900
```

Frontend không được gọi:

```text
/internal/courses/**
```

---

## 5. Biên giới giữa registration-service và course-service

`registration-service` KHÔNG truy cập trực tiếp `course_db`.

Hai service giao tiếp thông qua REST API nội bộ.

### Khi sinh viên đăng ký học phần

| Bước | Service | Hoạt động |
| :--- | :--- | :--- |
| 1 | React Frontend | Gửi POST /api/registrations |
| 2 | api-gateway :8085 | Xử lý request và định tuyến |
| 3 | registration-service :8093 | Kiểm tra thông tin đăng ký |
| 4 | registration-service :8093 | Kiểm tra đăng ký trùng |
| 5 | registration-service :8093 | Gọi PATCH /internal/courses/{id}/reserve-seat |
| 6 | course-service :8900 | Kiểm tra Course tồn tại |
| 7 | course-service :8900 | Kiểm tra `soChoConLai` |
| 8 | course-service :8900 | Trừ 1 khỏi `soChoConLai` |
| 9 | registration-service :8093 | Lưu Registration |
| 10 | api-gateway :8085 | Trả response |
| 11 | React Frontend | Nhận kết quả đăng ký |

Luồng:

```text
React :5173
     |
     | POST /api/registrations
     v
Gateway :8085
     |
     v
registration-service :8093
     |
     | PATCH /internal/courses/{id}/reserve-seat
     v
course-service :8900
     |
     v
course_db
```

### Khi hủy đăng ký

| Bước | Service | Hoạt động |
| :--- | :--- | :--- |
| 1 | React Frontend | Gửi DELETE /api/registrations/{id} |
| 2 | api-gateway :8085 | Định tuyến sang registration-service |
| 3 | registration-service :8093 | Tìm Registration |
| 4 | registration-service :8093 | Lấy courseId |
| 5 | registration-service :8093 | Gọi PATCH /internal/courses/{id}/release-seat |
| 6 | course-service :8900 | Hoàn trả 1 chỗ |
| 7 | registration-service :8093 | Cập nhật Registration thành DA_HUY |
| 8 | api-gateway :8085 | Trả response |
| 9 | React Frontend | Nhận kết quả |

---

## 6. Biên giới của course-service

`course-service` chạy tại:

```text
http://localhost:8900
```

Database:

```text
course_db
```

| Thành phần | Thuộc course-service | Không thuộc course-service |
| :--- | :---: | :---: |
| Course | Có | |
| id | Có | |
| tenMonHoc | Có | |
| soTinChi | Có | |
| soChoToiDa | Có | |
| soChoConLai | Có | |
| Registration | | Có |
| User | | Có |
| auth_db | | Có |
| registration_db | | Có |

### Quy tắc thay đổi số chỗ

| Thao tác | Nơi thực hiện |
| :--- | :--- |
| Tạo Course | course-service |
| Sửa Course | course-service |
| Xóa Course | course-service |
| Khởi tạo `soChoConLai` | course-service |
| Trừ `soChoConLai` | course-service → reserve-seat |
| Hoàn `soChoConLai` | course-service → release-seat |
| registration-service tự sửa `soChoConLai` | KHÔNG được phép |

Nguyên tắc:

```text
course-service là service DUY NHẤT
được quyền thay đổi dữ liệu Course.
```

---

## 7. Biên giới của registration-service

`registration-service` chạy tại:

```text
http://localhost:8093
```

Database:

```text
registration_db
```

| Thành phần | Thuộc registration-service |
| :--- | :---: |
| Registration | Có |
| id | Có |
| studentId | Có |
| courseId | Có |
| trangThai | Có |
| ngayDangKy | Có |
| course_db | Không |
| Course Entity | Không |
| soChoConLai | Không |

### Quy tắc

- Chỉ lưu `courseId`.
- Không tạo bảng Course.
- Không truy cập trực tiếp `course_db`.
- Không tự sửa `soChoConLai`.
- Muốn giữ chỗ phải gọi `reserve-seat`.
- Muốn hoàn chỗ phải gọi `release-seat`.
- Registration chỉ được lưu sau khi course-service xác nhận giữ chỗ thành công.

---

## 8. Biên giới của auth-service

`auth-service` chạy tại:

```text
http://localhost:8094
```

Database:

```text
auth_db
```

| Chức năng | auth-service |
| :--- | :---: |
| Quản lý User | Có |
| Đăng nhập | Có |
| Kiểm tra username/password | Có |
| Sinh JWT | Có |
| Xác định role ADMIN/STUDENT | Có |
| Quản lý Course | Không |
| Quản lý Registration | Không |
| Truy cập course_db | Không |
| Truy cập registration_db | Không |

Luồng đăng nhập:

```text
React :5173
     |
     | POST /api/auth/login
     v
Gateway :8085
     |
     v
auth-service :8094
     |
     v
auth_db
     |
     | JWT
     v
React
```

---

## 9. Biên giới của api-gateway

`api-gateway` chạy tại:

```text
http://localhost:8085
```

Gateway không có database riêng.

| Gateway được làm | Gateway không làm |
| :--- | :--- |
| Định tuyến Request | Quản lý Course |
| RewritePath | Quản lý Registration |
| Kiểm tra Authorization Header | Truy cập database nghiệp vụ |
| Xử lý API Key | Thay đổi soChoConLai |
| Cấu hình CORS | Tạo Registration |
| Chuyển request tới service | Chứa business logic của Course |
| Expose API công khai | Expose `/internal/**` |

### Security Boundary

| Route | Cơ chế |
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
| DELETE /api/registrations/{id} | JWT + STUDENT/ADMIN |
| /api/public/courses | X-API-KEY |
| /internal/courses/** | Không expose qua Gateway |

### CORS

Frontend hiện chạy tại:

```text
http://localhost:5173
```

Gateway cho phép origin:

```text
http://localhost:5173
```

CORS được xử lý tập trung tại Gateway.

---

## 10. Biên giới Frontend

Frontend hiện tại:

```text
crs-frontend
React + TypeScript + Vite
Port: 5173
```

Frontend KHÔNG phải backend service và không được truy cập trực tiếp các service phía sau.

### Frontend được phép gọi

| Frontend được phép | Frontend không được phép |
| :--- | :--- |
| http://localhost:8085/api/auth/** | http://localhost:8094/** |
| http://localhost:8085/api/courses/** | http://localhost:8900/** |
| http://localhost:8085/api/registrations/** | http://localhost:8093/** |
| http://localhost:8085/api/public/courses | /internal/** |

Nguyên tắc:

```text
Frontend
   ↓
API Gateway
   ↓
Microservices
```

Không:

```text
Frontend
   ↓
course-service trực tiếp
```

### Axios Boundary

Frontend sử dụng:

```text
src/api/axiosClient.ts
```

Biến môi trường:

```env
VITE_API_BASE_URL=http://localhost:8085
```

| Thành phần | Quy tắc |
| :--- | :--- |
| axiosClient.ts | Axios instance dùng chung |
| VITE_API_BASE_URL | Trỏ tới API Gateway :8085 |
| courseApi.ts | Gọi Course thông qua axiosClient |
| authApi.ts | Gọi Auth thông qua axiosClient khi triển khai |
| registrationApi.ts | Gọi Registration thông qua axiosClient khi triển khai |
| Hardcode localhost:8094 | Không |
| Hardcode localhost:8900 | Không |
| Hardcode localhost:8093 | Không |
| Gọi `/internal/**` | Không |

### Luồng đã kiểm thử thành công

Hệ thống hiện đã kiểm thử:

```text
React :5173
     ↓
Axios
     ↓
API Gateway :8085
     ↓
course-service :8900
     ↓
course_db
     ↓
API Gateway
     ↓
React
```

React đã nhận được Course:

```json
{
  "id": 1,
  "tenMonHoc": "Lap trinh Java",
  "soTinChi": 3,
  "soChoToiDa": 1,
  "soChoConLai": 1
}
```

---

## 11. Sơ đồ biên giới Service

```text
                           crs-frontend
                              :5173
                                |
                                | HTTP /api/**
                                | Axios
                                v
                     +-----------------------+
                     |      api-gateway      |
                     |         :8085         |
                     | Routing / JWT / CORS  |
                     |       API Key         |
                     +-----------+-----------+
                                 |
                +----------------+----------------+
                |                |                |
                v                v                v
       +----------------+ +---------------+ +----------------------+
       |  auth-service  | |course-service | |registration-service  |
       |     :8094      | |     :8900     | |        :8093         |
       +-------+--------+ +-------+-------+ +----------+-----------+
               |                  |                    |
               v                  v                    v
          +---------+        +-----------+       +-----------------+
          | auth_db |        | course_db |       | registration_db |
          +---------+        +-----------+       +-----------------+
                                  ^
                                  |
                                  | REST API nội bộ
                                  | reserve-seat
                                  | release-seat
                                  |
                         registration-service
                              :8093
```

---

## 12. Tổng kết biên giới Service

Hệ thống tuân theo các nguyên tắc chính:

1. Mỗi microservice chịu trách nhiệm cho một miền nghiệp vụ riêng.
2. Mỗi service sở hữu database riêng.
3. Service không truy cập trực tiếp database của service khác.
4. `course-service` là nơi duy nhất quản lý Course và số chỗ.
5. `registration-service` chỉ lưu thông tin Registration và `courseId`.
6. `registration-service` gọi REST API nội bộ của `course-service` khi cần giữ hoặc hoàn chỗ.
7. `auth-service` chịu trách nhiệm xác thực và JWT.
8. `api-gateway` là điểm truy cập duy nhất của Frontend.
9. Frontend không gọi trực tiếp các backend service.
10. API `/internal/**` không được expose ra Gateway.
11. CORS được xử lý tập trung tại Gateway.
12. Các service hiện sử dụng các cổng thực tế:

| Thành phần | Port |
| :--- | :---: |
| crs-frontend | 5173 |
| api-gateway | 8085 |
| auth-service | 8094 |
| course-service | 8900 |
| registration-service | 8093 |
| MySQL | 3306 |
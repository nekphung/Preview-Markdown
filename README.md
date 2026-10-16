# ERD HCMUS-HUB

> **Trạng thái:** bản đã chốt về mặt thiết kế cho task **S0-04** · Người phụ trách: Đoàn Thế Phùng · Người review: Trịnh Đỗ Minh Huy
>
> §9 và §10 đã được nhóm quyết xong (2026-10-16). Khi làm S1-02, S2-02, S3-02, S4-03 nhóm sẽ viết thành file migration trong `db/migrations/` và được phép chỉnh lại, **sửa luôn file này trong cùng Pull Request**.

## 0. Nguồn và phạm vi

| Mục | Nội dung |
|---|---|
| Tài liệu này bám theo | [`CODING_CONVENTION.md`](../CODING_CONVENTION.md) §11 (đặt tên, `created_at`/`updated_at`, soft delete, RLS, vector) và [`RULES.md`](../RULES.md) §4.4 |
| Phạm vi nghiệp vụ | [`Khảo sát thực trạng&Phạm vi bài toán.md`](<Khảo sát thực trạng&Phạm vi bài toán.md>) — 6 nhóm phân hệ cốt lõi |
| SRS | **Chưa có trong repo.** Mọi chỗ tài liệu này nhắc "SRS" là để tham chiếu tới tài liệu sẽ viết ở S0-02; khi SRS có, phải đối chiếu lại §8 và §10 |
| Mã task liên quan | S1-02 (profiles/faculties), S2-02 (quy chế, danh bạ, thông báo), S3-02 (tài liệu, kiểm duyệt, điểm), S4-03 (Knowledge Base, `pgvector`) |
| Khi nào được sửa | Cùng Pull Request với migration tương ứng. Không sửa migration đã gộp vào `staging`/`main` (`CODING_CONVENTION.md` §11.2) |

## 1. Quy ước chung

- Tên bảng, cột: `snake_case`; tên bảng số nhiều (mục 11.1 của `CODING_CONVENTION.md`).
- Khóa chính `id` kiểu `uuid`, mặc định `gen_random_uuid()`. Riêng `profiles.id` trùng `auth.users.id` của Supabase Auth.
- Khóa ngoại tên `<tên bảng số ít>_id`; ngoại lệ có nghĩa riêng: `uploader_id`, `reviewed_by`, `uploaded_by`, `created_by`.
- Mọi bảng có `created_at` và `updated_at` (`timestamptz`, `NOT NULL`, mặc định `now()`), **trừ các bảng chỉ ghi thêm, không sửa**: `point_transactions` (sổ điểm), `kb_chunks` (đoạn văn bản; đánh chỉ mục lại = xóa rồi thêm), `moderation_logs` (nhật ký kiểm duyệt) và `chat_messages` (lịch sử trò chuyện) — bốn bảng này chỉ có `created_at`.
- Bảng có thể ẩn/xóa có thêm `deleted_at` (`timestamptz`, trống = còn dùng). Truy vấn mặc định lọc `deleted_at IS NULL`.
- Chuỗi dùng kiểu `text`. Ràng buộc khai báo ngay trong database: `NOT NULL`, `UNIQUE`, `CHECK`, khóa ngoại.
- Email và mật khẩu **không** nằm trong `profiles`; chúng ở `auth.users` do Supabase Auth quản lý.

### 1.1 Ghi chú kỹ thuật cho người viết migration

- **Cột `updated_at`**: viết một trigger duy nhất rồi gắn vào mọi bảng có cột này, đừng mỗi bảng một kiểu.
- **`gen_random_uuid()`**: có sẵn từ PostgreSQL 13, **không** cần extension `pgcrypto`. Nhóm kiểm tra phiên bản PostgreSQL của dự án Supabase trước khi viết migration (task khởi tạo database) để chắc chắn ≥ 13.
- **`pgvector`**: bảng `kb_chunks` cần bật extension `vector` trước khi tạo cột `embedding`. Số chiều `N` chốt ở S4-03; đổi số chiều = migration mới + đánh chỉ mục lại toàn bộ vector.
- **Trình tự migration**: tạo bảng cha trước → tạo bảng con → thêm `CHECK` phụ thuộc (`documents.reject_reason`) → tạo index → bật RLS và viết policy.

## 2. Sơ đồ

Sơ đồ viết bằng Mermaid nên GitHub hiển thị trực tiếp; sửa sơ đồ bằng Pull Request như sửa code.

Quy ước đọc: `||` = bắt buộc có (NOT NULL) · `o|` = có thể trống (NULL) · `o{` = nhiều.

```mermaid
erDiagram
    faculties o|--o{ profiles : "có sinh viên"
    faculties o|--o{ contacts : "có danh bạ"
    faculties o|--o{ regulations : "có quy chế"
    faculties o|--o{ notification_sources : "có nguồn"
    faculties ||--o{ courses : "có môn học"
    faculties o|--o{ templates : "có template"
    profiles ||--o{ documents : "tải lên"
    profiles o|--o{ documents : "kiểm duyệt"
    profiles ||--o{ point_transactions : "nhận điểm"
    profiles ||--o{ templates : "đăng"
    profiles ||--o{ kb_documents : "tải lên"
    profiles o|--o{ notifications : "nhập thủ công"
    courses ||--o{ documents : "có tài liệu"
    documents o|--o{ point_transactions : "sinh điểm"
    notification_sources ||--o{ notifications : "cung cấp"
    notification_sources ||--o{ crawl_runs : "có lần thu thập"
    regulations o|--o{ kb_documents : "nguồn của"
    kb_documents ||--o{ kb_chunks : "được cắt thành"
    tags ||--o{ notifications : "được gắn tag"
    semesters ||--o{ documents : "thuộc học kỳ"
    documents ||--o{ moderation_logs : "được kiểm duyệt"
    profiles ||--o{ moderation_logs : "kiểm duyệt"
    documents ||--o{ document_reports : "bị phản ánh"
    profiles ||--o{ document_reports : "người gửi / xử lý"
    profiles ||--o{ chat_sessions : "trò chuyện"
    chat_sessions ||--o{ chat_messages : "gồm các tin nhắn"

    faculties {
        uuid id PK
        text code UK "mã Khoa"
        text name
        text website_url
        timestamptz created_at
        timestamptz updated_at
    }
    profiles {
        uuid id PK, FK "trùng auth.users.id"
        uuid faculty_id FK
        text full_name
        text student_id UK "MSSV"
        smallint cohort "khóa"
        text avatar_url
        user_role role "mặc định STUDENT"
        boolean has_student_email "huy hiệu email sinh viên"
        boolean is_locked
        int total_points
        int level
        timestamptz created_at
        timestamptz updated_at
    }
    contacts {
        uuid id PK
        uuid faculty_id FK "trống = cấp Trường"
        text unit_name
        text email
        text hotline
        text fanpage_url
        text source_url
        date last_verified_at
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    regulations {
        uuid id PK
        uuid faculty_id FK "trống = toàn Trường"
        text title
        regulation_category category
        smallint cohort_from "khóa bắt đầu áp dụng"
        smallint cohort_to "trống = từ cohort_from trở đi"
        text summary
        text file_path
        text source_url
        date effective_date
        date expired_at "trống = còn hiệu lực"
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    email_templates {
        uuid id PK
        text title
        text category
        text body
        text receiver_hint "nơi tiếp nhận"
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    notification_sources {
        uuid id PK
        uuid faculty_id FK "trống = Trường hoặc Phòng ban"
        text name
        text url UK
        source_kind kind
        boolean is_active
        timestamptz last_crawled_at
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    crawl_runs {
        uuid id PK
        uuid source_id FK
        crawl_run_status status "mặc định RUNNING"
        timestamptz started_at
        timestamptz finished_at
        int fetched_count
        int new_count
        text error_message
        timestamptz created_at
        timestamptz updated_at
    }
    tags {
        uuid id PK
        text name NN, UK
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    notifications {
        uuid id PK
        uuid source_id FK
        uuid created_by FK "chỉ khi nhập thủ công"
        uuid tag_id FK
        text title
        text summary
        timestamptz published_at
        timestamptz deadline_at
        text source_url UK "dùng để loại trùng"
        boolean is_hidden
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    semesters {
        uuid id PK
        text code NN, UK "ví dụ 2025-2026-1"
        text name
        smallint academic_year "năm đầu của năm học"
        smallint term_no "1, 2 hoặc 3"
        date start_date
        date end_date
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    courses {
        uuid id PK
        uuid faculty_id FK
        text code UK "mã môn"
        text name
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    documents {
        uuid id PK
        uuid uploader_id FK
        uuid course_id FK
        uuid semester_id FK
        uuid reviewed_by FK "trống khi chưa duyệt"
        text title
        text description
        document_type doc_type
        text file_path
        text file_name
        text mime_type
        bigint file_size_bytes
        document_status status "mặc định PENDING"
        text reject_reason
        timestamptz reviewed_at
        int download_count
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    moderation_logs {
        uuid id PK
        uuid document_id FK
        uuid moderator_id FK
        moderation_action action
        document_status from_status
        document_status to_status
        text reason
        timestamptz created_at
    }
    document_reports {
        uuid id PK
        uuid document_id FK
        uuid reporter_id FK
        report_reason reason
        text description
        report_status status "mặc định PENDING"
        uuid handled_by FK "trống khi chưa xử lý"
        timestamptz handled_at
        text handle_note
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    templates {
        uuid id PK
        uuid faculty_id FK "trống = dùng chung"
        uuid uploaded_by FK
        text title
        text description
        template_file_type file_type
        text file_path
        int download_count
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    point_transactions {
        uuid id PK
        uuid user_id FK
        uuid document_id FK "trống nếu điều chỉnh tay"
        int points
        point_reason reason
        timestamptz created_at
    }
    kb_documents {
        uuid id PK
        uuid regulation_id FK "trống nếu không có trong Cẩm nang"
        uuid uploaded_by FK
        text title
        text file_path
        text source_url
        date effective_date
        kb_status status "mặc định INDEXING"
        text error_message
        boolean is_active
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    kb_chunks {
        uuid id PK
        uuid kb_document_id FK
        int chunk_index
        text content
        text page_ref "để trích dẫn nguồn"
        vector embedding "pgvector, số chiều chốt ở S4-03"
        timestamptz created_at
    }
    chat_sessions {
        uuid id PK
        uuid user_id FK
        text title "tự sinh từ câu hỏi đầu tiên"
        int message_count
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    chat_messages {
        uuid id PK
        uuid session_id FK
        chat_message_role role
        text content
        jsonb citations "trích dẫn nguồn của câu trả lời AI"
        timestamptz created_at
    }
```

Ngoài các bảng trên còn hai đối tượng trong database:

- **View `public_leaderboard`** (`SECURITY DEFINER`): cột `full_name`, `avatar_url`, `level`, `total_points`. Cố ý **không** có `student_id`, `faculty_id`, `email` — xem §7.1 và §9.10.
- **Hàm helper phân quyền**: `is_moderator()`, `is_admin()` (`SECURITY DEFINER`) — xem §7.1.

`auth.users` (bảng của Supabase Auth) không vẽ vào sơ đồ nhưng có quan hệ 1–1 với `profiles`, xem bảng ở mục 3.

## 3. Danh sách quan hệ

Hành vi khi xóa (`ON DELETE`): hệ thống dùng **xóa mềm** nên hầu hết là `RESTRICT` (không cho xóa thật nếu còn dữ liệu con). Không dùng `SET NULL` cho `faculty_id` vì làm dữ liệu của một Khoa vô tình thành dữ liệu cấp Trường.

| Cha (1) | Con (n) | Khóa ngoại | Bắt buộc? | `ON DELETE` | Ý nghĩa |
|---|---|---|---|---|---|
| `auth.users` | `profiles` | `profiles.id` | Có (1–1) | `CASCADE` | Mỗi tài khoản đăng nhập có một hồ sơ |
| `faculties` | `profiles` | `faculty_id` | Không | `RESTRICT` | Khoa của sinh viên |
| `faculties` | `contacts` | `faculty_id` | Không | `RESTRICT` | Trống = đơn vị cấp Trường |
| `faculties` | `regulations` | `faculty_id` | Không | `RESTRICT` | Trống = quy chế toàn Trường |
| `faculties` | `notification_sources` | `faculty_id` | Không | `RESTRICT` | Trống = nguồn của Trường hoặc Phòng ban |
| `faculties` | `courses` | `faculty_id` | Có | `RESTRICT` | Môn học thuộc Khoa |
| `faculties` | `templates` | `faculty_id` | Không | `RESTRICT` | Trống = template dùng chung |
| `profiles` | `documents` | `uploader_id` | Có | `RESTRICT` | Người tải lên |
| `profiles` | `documents` | `reviewed_by` | Không | `RESTRICT` | Người duyệt/từ chối gần nhất |
| `profiles` | `point_transactions` | `user_id` | Có | `RESTRICT` | Người nhận điểm |
| `profiles` | `templates` | `uploaded_by` | Có | `RESTRICT` | Người đăng template |
| `profiles` | `kb_documents` | `uploaded_by` | Có | `RESTRICT` | Admin tải văn bản |
| `profiles` | `notifications` | `created_by` | Không | `RESTRICT` | Người nhập thủ công |
| `courses` | `documents` | `course_id` | Có | `RESTRICT` | Tài liệu thuộc môn |
| `documents` | `point_transactions` | `document_id` | Không | `RESTRICT` | Tài liệu sinh ra điểm |
| `notification_sources` | `notifications` | `source_id` | Có | `RESTRICT` | Nguồn của thông báo |
| `notification_sources` | `crawl_runs` | `source_id` | Có | `RESTRICT` | Một lần thu thập từ nguồn đó |
| `tags` | `notifications` | `tag_id` | Có | `RESTRICT` | Tag của thông báo |
| `semesters` | `documents` | `semester_id` | Có | `RESTRICT` | Học kỳ của tài liệu |
| `documents` | `moderation_logs` | `document_id` | Có | `RESTRICT` | Nhật ký kiểm duyệt tài liệu |
| `profiles` | `moderation_logs` | `moderator_id` | Có | `RESTRICT` | Người kiểm duyệt |
| `documents` | `document_reports` | `document_id` | Có | `RESTRICT` | Tài liệu bị phản ánh |
| `profiles` | `document_reports` | `reporter_id` | Có | `RESTRICT` | Người gửi báo cáo |
| `profiles` | `document_reports` | `handled_by` | Không | `RESTRICT` | Người xử lý báo cáo |
| `profiles` | `chat_sessions` | `user_id` | Có | `RESTRICT` | Lịch sử trò chuyện của người dùng |
| `chat_sessions` | `chat_messages` | `session_id` | Có | `CASCADE` | Xoá phiên thì xoá luôn tin nhắn |
| `regulations` | `kb_documents` | `regulation_id` | Không | `RESTRICT` | Văn bản AI lấy từ quy chế nào |
| `kb_documents` | `kb_chunks` | `kb_document_id` | Có | `CASCADE` | Xóa văn bản thì xóa luôn các đoạn |

`email_templates` là bảng độc lập, không có khóa ngoại.

## 4. Từ điển dữ liệu

Cột "Ràng buộc": `PK` khóa chính · `FK` khóa ngoại · `NN` `NOT NULL` · `UQ` duy nhất. `trống được` = cho phép `NULL`.

### 4.1 `faculties` — Khoa

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `code` | `text` | NN, UQ | Mã Khoa |
| `name` | `text` | NN | Tên Khoa |
| `website_url` | `text` | trống được | Trang chính thức của Khoa |
| `created_at`, `updated_at` | `timestamptz` | NN, NN | |

### 4.2 `profiles` — Hồ sơ người dùng

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK, FK → `auth.users.id` | |
| `faculty_id` | `uuid` | FK → `faculties.id` | Người dùng tự do có thể để trống |
| `full_name` | `text` | NN | |
| `student_id` | `text` | UQ | MSSV; trống với người dùng tự do. Nhiều dòng trống vẫn được (NULL không trùng nhau trong PostgreSQL) |
| `cohort` | `smallint` | trống được | Khóa học, ví dụ `2024` |
| `avatar_url` | `text` | trống được | |
| `role` | `user_role` | NN, mặc định `STUDENT` | **Người dùng không được tự sửa**; chỉ Admin đổi |
| `has_student_email` | `boolean` | NN, mặc định `false` | Đặt `true` khi email thuộc `@student.hcmus.edu.vn`. **Ghi bởi backend** khi tạo hồ sơ, người dùng không tự sửa |
| `is_locked` | `boolean` | NN, mặc định `false` | Khóa tài khoản vi phạm; chỉ Admin đổi |
| `total_points` | `int` | NN, mặc định `0`, `CHECK (total_points >= 0)` | Số lưu sẵn để xếp hạng nhanh. **Chỉ server ghi**, trong cùng transaction với `point_transactions` |
| `level` | `int` | NN, mặc định `1`, `CHECK (level >= 1)` | Suy ra từ điểm; cùng transaction với `total_points` |
| `created_at`, `updated_at` | `timestamptz` | NN, NN | |

### 4.3 `contacts` — Danh bạ học vụ

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `faculty_id` | `uuid` | FK | Trống = cấp Trường (Phòng Đào tạo, Phòng CTSV...) |
| `unit_name` | `text` | NN | |
| `email`, `hotline`, `fanpage_url` | `text` | trống được | `CHECK`: ít nhất một trong ba cột có giá trị |
| `source_url` | `text` | NN | Nguồn chính thức của thông tin |
| `last_verified_at` | `date` | trống được | Lần cuối nhóm đối chiếu với nguồn |
| `created_at`, `updated_at`, `deleted_at` | `timestamptz` | NN, NN, trống được | |

### 4.4 `regulations` — Cẩm nang quy chế

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `faculty_id` | `uuid` | FK | Trống = toàn Trường |
| `title` | `text` | NN | |
| `category` | `regulation_category` | NN | |
| `cohort_from` | `smallint` | trống được, `CHECK (cohort_from >= 1900)` | Khóa bắt đầu áp dụng, ví dụ `2024` |
| `cohort_to` | `smallint` | trống được, `CHECK (cohort_to IS NULL OR cohort_to >= cohort_from)` | Khóa cuối áp dụng. Trống = áp dụng từ `cohort_from` trở đi. Cả hai trống = mọi khóa |
| `summary` | `text` | trống được | Tóm lược |
| `file_path` | `text` | trống được | Bản PDF (cũng dùng lại cho Knowledge Base ở Sprint 4) |
| `source_url` | `text` | NN | Link văn bản chính thức |
| `effective_date` | `date` | trống được | Ngày có hiệu lực |
| `expired_at` | `date` | `CHECK (expired_at >= effective_date)` | Trống = còn hiệu lực; tránh dùng quy định cũ |
| `created_at`, `updated_at`, `deleted_at` | `timestamptz` | NN, NN, trống được | |

### 4.5 `email_templates` — Mẫu email học vụ

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `title` | `text` | NN | Ví dụ "Xin hoãn thi" |
| `category` | `text` | NN | Khiếu nại điểm, hoãn thi, nộp chứng chỉ... |
| `body` | `text` | NN | Nội dung mẫu, dùng cho nút sao chép |
| `receiver_hint` | `text` | trống được | Hướng dẫn nơi tiếp nhận |
| `created_at`, `updated_at`, `deleted_at` | `timestamptz` | NN, NN, trống được | |

### 4.6 `notification_sources` — Nguồn thu thập thông báo

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `faculty_id` | `uuid` | FK | Trống = Trường hoặc Phòng ban |
| `name` | `text` | NN | Tên đơn vị đăng, hiển thị ở "Đơn vị đăng" |
| `url` | `text` | UQ một phần | Trang nguồn; trống nếu `kind = MANUAL`. Xem §6 về `deleted_at` |
| `kind` | `source_kind` | NN | `AUTO` thu thập tự động, `MANUAL` nhập tay |
| `is_active` | `boolean` | NN, mặc định `true` | Tắt nguồn mà không xóa |
| `last_crawled_at` | `timestamptz` | trống được | Lần thu thập gần nhất; **chỉ bộ thu thập (backend) ghi** |
| `created_at`, `updated_at`, `deleted_at` | `timestamptz` | NN, NN, trống được | |

### 4.7 `notifications` — Thông báo

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `source_id` | `uuid` | NN, FK | Đơn vị đăng |
| `created_by` | `uuid` | FK → `profiles.id` | Chỉ có khi nhập thủ công |
| `tag_id` | `uuid` | NN, FK → `tags.id` | Mỗi thông báo đúng một Tag |
| `title` | `text` | NN | |
| `summary` | `text` | trống được | **Chỉ lưu tóm lược**, không sao chép toàn văn |
| `published_at` | `timestamptz` | NN | Ngày đăng |
| `deadline_at` | `timestamptz` | trống được | Trống = không có hạn |
| `source_url` | `text` | UQ một phần | Liên kết "Xem bản gốc"; dùng loại trùng. Có thể trống với thông báo nhập tay. Xem §6 về `deleted_at` |
| `is_hidden` | `boolean` | NN, mặc định `false` | Moderator ẩn thông báo lỗi |
| `created_at`, `updated_at`, `deleted_at` | `timestamptz` | NN, NN, trống được | |

Nhãn "Sắp hết hạn"/"Đã hết hạn" **không lưu**, tính từ `deadline_at` lúc truy vấn theo giờ Việt Nam.

### 4.8 `courses` — Môn học

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `faculty_id` | `uuid` | NN, FK | |
| `code` | `text` | NN, UQ một phần | Mã môn. Xem §6 về `deleted_at` |
| `name` | `text` | NN | |
| `created_at`, `updated_at`, `deleted_at` | `timestamptz` | NN, NN, trống được | |

### 4.9 `documents` — Tài liệu học tập

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `uploader_id` | `uuid` | NN, FK → `profiles.id` | |
| `course_id` | `uuid` | NN, FK | |
| `semester_id` | `uuid` | NN, FK → `semesters.id` | Học kỳ để lọc; dùng bảng `semesters` thay vì chuỗi tự do |
| `reviewed_by` | `uuid` | FK → `profiles.id` | Trống khi chưa duyệt; **chỉ Moderator/Admin ghi** |
| `title` | `text` | NN | |
| `description` | `text` | trống được | |
| `doc_type` | `document_type` | NN | Loại tài liệu |
| `file_path` | `text` | NN | Đường dẫn trong Storage, chưa gắn với Cloudinary hay Supabase (S3-03) |
| `file_name`, `mime_type` | `text` | trống được | Tên gốc, loại tệp |
| `file_size_bytes` | `bigint` | trống được, `CHECK (file_size_bytes > 0)` | |
| `status` | `document_status` | NN, mặc định `PENDING` | Chỉ `APPROVED` mới công khai |
| `reject_reason` | `text` | trống được | `CHECK`: bắt buộc có giá trị khi `status = 'REJECTED'` |
| `reviewed_at` | `timestamptz` | trống được | **Chỉ Moderator/Admin ghi**, đặt cùng lúc đổi `status` |
| `download_count` | `int` | NN, mặc định `0`, `CHECK (>= 0)` | **Backend tăng khi có lượt tải**, người dùng không tự sửa |
| `created_at`, `updated_at`, `deleted_at` | `timestamptz` | NN, NN, trống được | |

### 4.10 `templates` — Kho template báo cáo/slide

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `faculty_id` | `uuid` | FK | Trống = dùng chung toàn Trường |
| `uploaded_by` | `uuid` | NN, FK → `profiles.id` | |
| `title` | `text` | NN | |
| `description` | `text` | trống được | |
| `file_type` | `template_file_type` | NN | |
| `file_path` | `text` | NN | |
| `download_count` | `int` | NN, mặc định `0` | **Backend tăng khi có lượt tải** |
| `created_at`, `updated_at`, `deleted_at` | `timestamptz` | NN, NN, trống được | |

### 4.11 `point_transactions` — Sổ điểm (chỉ ghi thêm)

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `user_id` | `uuid` | NN, FK → `profiles.id` | |
| `document_id` | `uuid` | FK | Trống nếu điều chỉnh tay |
| `points` | `int` | NN, `CHECK (points <> 0)` | Số dương là cộng, số âm là thu hồi |
| `reason` | `point_reason` | NN | |
| `created_at` | `timestamptz` | NN | Không có `updated_at` vì không sửa dòng đã ghi |

Ràng buộc **một phần** `UNIQUE (document_id, reason) WHERE document_id IS NOT NULL` đảm bảo **một tài liệu chỉ được cộng điểm một lần** cho cùng một lý do, kể cả khi Moderator bấm duyệt lại; đồng thời vẫn cho phép nhiều dòng điều chỉnh tay (`document_id` trống). Xem §6.

### 4.12 `kb_documents` — Văn bản trong Knowledge Base

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `regulation_id` | `uuid` | FK | Trống nếu văn bản không có trong Cẩm nang |
| `uploaded_by` | `uuid` | NN, FK → `profiles.id` | Chỉ Admin |
| `title` | `text` | NN | Dùng khi trích dẫn nguồn |
| `file_path` | `text` | NN | |
| `source_url` | `text` | trống được | Link văn bản chính thức |
| `effective_date` | `date` | trống được | Để AI không trả lời theo quy định đã hết hiệu lực |
| `status` | `kb_status` | NN, mặc định `INDEXING` | **Backend ghi** khi đánh chỉ mục xong hoặc lỗi |
| `error_message` | `text` | trống được | Lý do khi `FAILED` |
| `is_active` | `boolean` | NN, mặc định `true` | Tắt để AI ngừng dùng văn bản |
| `created_at`, `updated_at`, `deleted_at` | `timestamptz` | NN, NN, trống được | |

### 4.13 `kb_chunks` — Đoạn văn bản và vector

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `kb_document_id` | `uuid` | NN, FK | `ON DELETE CASCADE` |
| `chunk_index` | `int` | NN | Thứ tự đoạn; `UNIQUE (kb_document_id, chunk_index)` |
| `content` | `text` | NN | Nội dung đoạn |
| `page_ref` | `text` | trống được | Trang/điều khoản để trích dẫn |
| `embedding` | `vector(N)` | NN | Số chiều `N` theo mô hình embedding, chốt ở S4-03 |
| `created_at` | `timestamptz` | NN | Không có `updated_at`: đánh chỉ mục lại = xóa rồi thêm |

### 4.14 `tags` — Tag thông báo

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `name` | `text` | NN, UQ một phần | Tên Tag hiển thị, ví dụ "Tuyển sinh". Xem §6 về `deleted_at` |
| `created_at`, `updated_at`, `deleted_at` | `timestamptz` | NN, NN, trống được | |

Admin quản lý danh sách Tag trong migration và có thể thêm/sửa qua giao diện quản trị. Mỗi thông báo đúng một Tag (`notifications.tag_id`).

### 4.15 `semesters` — Học kỳ

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `code` | `text` | NN, UQ một phần | Mã ngắn để gọi trong API, ví dụ `2025-2026-1`. Xem §6 về `deleted_at` |
| `name` | `text` | NN | Tên hiển thị, ví dụ "Học kỳ 1, năm học 2025-2026" |
| `academic_year` | `smallint` | NN, `CHECK (BETWEEN 2000 AND 2100)` | **Năm đầu** của năm học: `2025` nghĩa là `2025-2026` |
| `term_no` | `smallint` | NN, `CHECK (BETWEEN 1 AND 3)` | `1`, `2` hoặc `3` (học kỳ hè). Kiểu số để sắp xếp đúng thứ tự |
| `start_date`, `end_date` | `date` | trống được | Mốc thời gian, dùng để thống kê |
| `created_at`, `updated_at`, `deleted_at` | `timestamptz` | NN, NN, trống được | |

Học kỳ là khái niệm **toàn Trường** nên bảng này không có `faculty_id`. Nhóm nhập sẵn dữ liệu mốc bằng migration, không cần giao diện nhập.

### 4.16 `crawl_runs` — Nhật ký lần thu thập thông báo

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `source_id` | `uuid` | NN, FK → `notification_sources.id` | Nguồn vừa được thu thập |
| `status` | `crawl_run_status` | NN, mặc định `RUNNING` | **Chỉ bộ thu thập (backend) ghi** |
| `started_at` | `timestamptz` | NN, mặc định `now()` | |
| `finished_at` | `timestamptz` | trống được | Trống = đang chạy |
| `fetched_count` | `int` | NN, mặc định `0`, `CHECK (>= 0)` | Số bản ghi lấy được từ trang nguồn |
| `new_count` | `int` | NN, mặc định `0`, `CHECK (>= 0)` | Số thông báo mới sau khi loại trùng |
| `error_message` | `text` | trống được | Lý do khi `status = 'FAILED'` |
| `created_at`, `updated_at` | `timestamptz` | NN, NN | Có `updated_at` vì trạng thái chuyển `RUNNING → SUCCESS/FAILED` |

Không có `deleted_at`: nhật ký chỉ để tra cứu, không xoá mềm. `RULES.md` §4.4 yêu cầu thu thập "tần suất vừa phải" — bảng này là nơi kiểm tra điều đó.

### 4.17 `moderation_logs` — Nhật ký kiểm duyệt (chỉ ghi thêm)

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `document_id` | `uuid` | NN, FK → `documents.id` | Tài liệu bị kiểm duyệt |
| `moderator_id` | `uuid` | NN, FK → `profiles.id` | Người thực hiện |
| `action` | `moderation_action` | NN | `APPROVE` hoặc `REJECT` |
| `from_status` | `document_status` | NN | Trạng thái trước khi kiểm duyệt |
| `to_status` | `document_status` | NN | Trạng thái sau khi kiểm duyệt |
| `reason` | `text` | trống được | `CHECK`: bắt buộc có giá trị khi `action = 'REJECT'` |
| `created_at` | `timestamptz` | NN | Không có `updated_at` vì không sửa dòng đã ghi |

Mọi lần duyệt hoặc từ chối đều ghi một dòng, kể cả khi duyệt lại một tài liệu đã bị từ chối — đây là bằng chứng khi sinh viên khiếu nại.

### 4.18 `document_reports` — Báo cáo tài liệu vi phạm

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `document_id` | `uuid` | NN, FK → `documents.id` | Tài liệu bị phản ánh |
| `reporter_id` | `uuid` | NN, FK → `profiles.id` | Người gửi báo cáo |
| `reason` | `report_reason` | NN | Lý do, ví dụ `COPYRIGHT` |
| `description` | `text` | trống được | Mô tả thêm |
| `status` | `report_status` | NN, mặc định `PENDING` | **Chỉ Moderator/Admin ghi** |
| `handled_by` | `uuid` | FK → `profiles.id` | Trống khi chưa xử lý; **chỉ Moderator/Admin ghi** |
| `handled_at` | `timestamptz` | trống được | Đặt cùng lúc đổi `status` |
| `handle_note` | `text` | trống được | Kết quả xử lý, lý do bỏ qua |
| `created_at`, `updated_at`, `deleted_at` | `timestamptz` | NN, NN, trống được | |

Đáp ứng `CODING_CONVENTION.md` §13 (báo cáo/gỡ khi bị phản ánh). Khi `status = 'RESOLVED'` và người xử lý chọn gỡ tài liệu thì đặt `documents.deleted_at` (xoá mềm), nên `ON DELETE` của `document_id` vẫn là `RESTRICT`.

### 4.19 `chat_sessions` — Phiên hỏi chatbot

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `user_id` | `uuid` | NN, FK → `profiles.id` | Chủ sở hữu, duy nhất người được đọc |
| `title` | `text` | trống được | Backend tự sinh từ câu hỏi đầu tiên (cắt bớt ký tự) |
| `message_count` | `int` | NN, mặc định `0`, `CHECK (>= 0)` | **Backend ghi**, dùng để hiện "N tin nhắn" |
| `created_at`, `updated_at`, `deleted_at` | `timestamptz` | NN, NN, trống được | |

Xoá lịch sử: xoá mềm theo `deleted_at` để ẩn phiên, hoặc xoá thật phiên thì `ON DELETE CASCADE` xoá luôn `chat_messages`.

### 4.20 `chat_messages` — Tin nhắn trong phiên (chỉ ghi thêm)

| Cột | Kiểu | Ràng buộc | Ghi chú |
|---|---|---|---|
| `id` | `uuid` | PK | |
| `session_id` | `uuid` | NN, FK → `chat_sessions.id` | `ON DELETE CASCADE` |
| `role` | `chat_message_role` | NN | `USER` hoặc `ASSISTANT` |
| `content` | `text` | NN | Nội dung tin nhắn |
| `citations` | `jsonb` | trống được | Trích dẫn nguồn của câu trả lời: `{ "kb_document": "...", "page_ref": "..." }`. Chỉ `role = 'ASSISTANT'` mới có |
| `created_at` | `timestamptz` | NN | Không có `updated_at` vì không sửa tin nhắn đã gửi |

Không có `deleted_at`: xoá tin nhắn luôn theo cả phiên. Đây là **dữ liệu cá nhân** — RLS chỉ cho chủ phiên đọc, xem §7.

## 5. Kiểu enum

| Kiểu | Giá trị |
|---|---|
| `user_role` | `STUDENT`, `MODERATOR`, `ADMIN` |
| `regulation_category` | `REGISTRATION` (đăng ký học phần), `ACADEMIC_WARNING` (cảnh báo học vụ), `ENGLISH_STANDARD` (chuẩn Tiếng Anh), `OTHER` |
| `source_kind` | `AUTO`, `MANUAL` |
| `document_type` | `EXAM` (đề thi), `SLIDE`, `TEXTBOOK` (giáo trình), `OTHER` |
| `document_status` | `PENDING`, `APPROVED`, `REJECTED` |
| `template_file_type` | `WORD`, `POWERPOINT`, `LATEX`, `OTHER` |
| `point_reason` | `DOCUMENT_APPROVED`, `ADJUSTMENT` |
| `kb_status` | `INDEXING`, `READY`, `FAILED` |
| `crawl_run_status` | `RUNNING`, `SUCCESS`, `FAILED` |
| `moderation_action` | `APPROVE`, `REJECT` |
| `report_reason` | `COPYRIGHT` (bản quyền), `WRONG_CONTENT` (sai nội dung), `OUTDATED` (lỗi thời), `BROKEN_FILE` (tệp hỏng), `OTHER` |
| `report_status` | `PENDING`, `RESOLVED` (đã xử lý, có thể gỡ tài liệu), `DISMISSED` (báo cáo không có cơ sở) |
| `chat_message_role` | `USER`, `ASSISTANT` |

Luồng trạng thái tài liệu: `PENDING → APPROVED` hoặc `PENDING → REJECTED` (kèm lý do). Chuyển sai luồng thì backend báo lỗi.

## 6. Ràng buộc và chỉ mục đề xuất

### 6.1 Quy tắc chung

- **Bảng có `deleted_at` thì `UNIQUE` phải là chỉ mục duy nhất một phần** (`... WHERE deleted_at IS NULL`). Lý do: nếu `UNIQUE` thường thì sau khi xóa mềm một mã môn hay một URL thông báo, không thêm lại được bản ghi mới trùng giá trị cũ — sẽ kẹt cả vòng đời.
  - Áp dụng cho: `courses.code`, `notification_sources.url`, `notifications.source_url`, `tags.name`, `semesters.code`.
- **PostgreSQL không tự tạo chỉ mục cho cột khóa ngoại.** Mọi cột khóa ngoại dùng để lọc/join đều phải khai báo chỉ mục tay.
- Cột `uuid` cần vòng phân trang (keyset) thì cần một chỉ mục theo cột sắp xếp (ví dụ `(published_at DESC)`), không dựa vào `OFFSET`.

### 6.2 Danh sách đề xuất

| Mục tiêu | Đề xuất |
|---|---|
| Không cộng điểm trùng | `UNIQUE (document_id, reason) WHERE document_id IS NOT NULL` trên `point_transactions` |
| Loại thông báo trùng | Chỉ mục duy nhất một phần trên `notifications (source_url) WHERE deleted_at IS NULL` |
| Tạo lại mã môn / URL nguồn sau khi xóa mềm | Chỉ mục duy nhất một phần trên `courses (code) WHERE deleted_at IS NULL` và `notification_sources (url) WHERE deleted_at IS NULL` |
| Tạo lại tên Tag / mã học kỳ sau khi xóa mềm | Chỉ mục duy nhất một phần trên `tags (name)` và `semesters (code)`, đều kèm `WHERE deleted_at IS NULL` |
| Lọc thông báo theo Tag (yêu cầu ở §8) | Chỉ mục đơn trên `notifications.tag_id` |
| Lọc nhanh theo Khoa (yêu cầu ở §8) | Chỉ mục đơn trên: `contacts.faculty_id`, `regulations.faculty_id`, `courses.faculty_id`, `templates.faculty_id`, `profiles.faculty_id` |
| Lọc và sắp xếp thông báo | Chỉ mục `(published_at DESC)` và `(deadline_at)` trên `notifications`; thêm `is_hidden` vào đầu chỉ mục `published_at` nếu truy vấn luôn lọc thông báo bị ẩn |
| Hàng đợi kiểm duyệt | Chỉ mục `(status, created_at)` trên `documents` |
| Sổ điểm của một người | Chỉ mục `(user_id, created_at DESC)` trên `point_transactions`; chỉ mục `(course_id)` và `(uploader_id)` trên `documents` |
| Bảng xếp hạng | Chỉ mục `(total_points DESC)` trên `profiles` |
| Lọc quy chế theo khóa (yêu cầu ở §8) | Chỉ mục đơn trên `regulations.cohort_from`, `regulations.cohort_to`; chung với `faculty_id` thì dùng chỉ mục nhiều cột `(faculty_id, cohort_from)` |
| Lọc tài liệu theo học kỳ (yêu cầu ở §8) | Chỉ mục `(course_id, semester_id)` trên `documents` |
| Theo dõi bộ thu thập (S2-03) | Chỉ mục `(source_id, started_at DESC)` trên `crawl_runs` |
| Xem lịch sử kiểm duyệt một tài liệu | Chỉ mục `(document_id, created_at DESC)` trên `moderation_logs` |
| Hàng đợi báo cáo vi phạm | Chỉ mục `(status, created_at)` trên `document_reports`; chỉ mục `(reporter_id, created_at DESC)` để người dùng xem lại các báo cáo của mình |
| Danh sách phiên chat của tôi | Chỉ mục `(user_id, updated_at DESC)` trên `chat_sessions`; chỉ mục `(session_id, created_at)` trên `chat_messages` để đọc tin nhắn theo thứ tự |
| Tìm tài liệu < 1 giây | Chỉ mục tìm kiếm toàn văn (GIN) trên `documents.title`, `courses.code`/`courses.name`; chỉ mục `(status, course_id)`. Cần cân nhắc tìm không dấu tiếng Việt — xem ghi chú bên dưới |
| Tìm vector | Chỉ mục vector (ví dụ HNSW) trên `kb_chunks.embedding`, chốt ở S4-03 |

**Ghi chú tìm tiếng Việt không dấu:** hàm `unaccent()` mặc định **không** mang nhãn `IMMUTABLE`, nên không dùng trực tiếp trong biểu thức chỉ mục được — phải bọc bằng một hàm `IMMUTABLE` của riêng dự án. Cách ít rủi ro hơn cho phiên bản đầu là extension `pg_trgm` (`GIN` với `gin_trgm_ops`) cho tìm gần đúng, rồi chốt ở S3-02.

## 7. Quyền truy cập dự kiến (RLS)

Bật Row Level Security cho mọi bảng. Phần thực thi chi tiết viết trong migration; bảng này để người review soát ý định.

| Bảng | Ai đọc được | Ai ghi được |
|---|---|---|
| `faculties`, `contacts`, `regulations`, `email_templates`, `templates`, `courses` | Mọi người dùng (kể cả chưa đăng nhập, nếu nhóm đồng ý công khai) | Admin |
| `notifications`, `notification_sources` | Mọi người (không gồm thông báo `is_hidden`) | Admin, Moderator; bộ thu thập chạy bằng khóa backend |
| `tags` | Mọi người dùng | Admin, Moderator |
| `semesters` | Mọi người dùng | Admin |
| `crawl_runs` | Admin, Moderator | Chỉ backend (bộ thu thập); người dùng thường không đọc được |
| `moderation_logs` | Moderator, Admin; người tải tài liệu xem được lịch sử duyệt của chính tài liệu mình | Chỉ Moderator/Admin (backend ghi khi đổi `documents.status`) |
| `document_reports` | Người gửi xem báo cáo của mình; Moderator/Admin xem tất cả | Người đăng nhập gửi báo cáo; **chỉ** Moderator/Admin đổi `status`, `handled_by`, `handled_at`, `handle_note` |
| `profiles` | Chủ hồ sơ; bảng xếp hạng đọc qua **view công khai** (xem ghi chú bên dưới) | Chủ hồ sơ sửa thông tin cá nhân; **không** được sửa `role`, `is_locked`, `total_points`, `level`, `has_student_email`. Admin đổi `role`, `is_locked` |
| `documents` | Mọi người xem tài liệu `APPROVED`; chủ tài liệu xem tài liệu của mình; Moderator/Admin xem tất cả | Sinh viên đã đăng nhập thêm tài liệu `PENDING`; chỉ Moderator/Admin đổi `status` |
| `point_transactions` | Chủ sổ điểm | Chỉ backend (khóa `service`) |
| `kb_documents`, `kb_chunks` | Backend (để AI truy hồi); Admin xem danh sách | Admin tải lên; backend ghi chunk |
| `chat_sessions`, `chat_messages` | **Chỉ chủ phiên** (`chat_sessions.user_id = auth.uid()`) | Người dùng tạo phiên và gửi tin nhắn của mình; **không** ai sửa được tin nhắn đã gửi |

### 7.1 Hai điểm dễ làm sai khi viết policy

1. **RLS chỉ ở mức hàng, không ở mức cột.** Không thể làm policy `profiles` mà vẫn giấu `email`/`student_id` chỉ với RLS. Muốn công khai "tên, điểm, level" cho bảng xếp hạng mà giấu phần còn lại thì phải có **một view** (ví dụ `public_leaderboard`) đánh dấu `SECURITY DEFINER`, chỉ `SELECT` đúng ba cột đó, rồi mới cho phép đọc view. View cũng phải lọc `deleted_at IS NULL` để không lộ dữ liệu đã xóa mềm.
   - Đã chốt (§9.10): view trả `full_name`, `avatar_url`, `level`, `total_points`; **không** có `student_id`, `faculty_id`. Xem §10 dòng 9.
2. **Đừng đọc `profiles.role` trực tiếp trong policy của bảng khác.** Policy `documents` lấy `role` từ `profiles` sẽ đệ quy RLS trên chính `profiles`. Cách chuẩn: viết hàm helper `SECURITY DEFINER` (ví dụ `is_moderator()`, `is_admin()`) đọc vai trò rồi gọi hàm đó trong policy, và đặt `search_path` cố định cho hàm.

Ngoài ra: `anon` chỉ được chạm vào bảng thực sự cần đọc ở giao diện; mọi thao tác ghi đi qua FastAPI bằng khóa `service` nằm trong `.env` (`CODING_CONVENTION.md` §11.3).

### 7.2 Vài điểm riêng của bảng mới

- **Lịch sử chatbot là dữ liệu cá nhân.** Policy của `chat_messages` không được đọc `chat_sessions` một cách thoải mái vì sẽ đệ quy RLS; cách chuẩn là policy kiểu `EXISTS` gọi hàm helper `SECURITY DEFINER` (xem §7.1) để kiểm tra phiên có thuộc về người gọi không.
- **`document_reports` không cho người gửi sửa sau khi gửi.** Nếu cho sửa, người dùng có thể đổi `status` để lách. Chỉ `document_id` và `reason` được chốt ngay lúc gửi.
- **Xoá lịch sử chat là quyền của người dùng**, không phải quyền xoá của Admin: đặt `chat_sessions.deleted_at` (xoá mềm).
- **`crawl_runs` chỉ ghi bởi backend.** Không có policy nào cho người dùng thường ghi vào bảng này, kể cả Admin.

## 8. Kiểm tra bao phủ yêu cầu

| Yêu cầu | Đáp ứng bằng |
|---|---|
| Gắn huy hiệu khi dùng email sinh viên HCMUS | `profiles.has_student_email` |
| Lọc danh bạ, quy chế theo Khoa | `contacts.faculty_id`, `regulations.faculty_id` (+ chỉ mục ở §6) |
| Quy chế theo khóa/ngành, tránh dùng quy định cũ | `regulations.cohort_from`, `cohort_to`, `effective_date`, `expired_at` — lọc theo khóa đã làm được (xem §9.4) |
| Mẫu email kèm hướng dẫn nơi tiếp nhận | `email_templates.body`, `receiver_hint` |
| Thông báo có Tag, Ngày đăng, Tiêu đề, Đơn vị đăng, Deadline | `notifications.tag_id` → `tags.name`, `published_at`, `title`, `source_id`, `deadline_at` |
| Nhãn Sắp hết hạn / Đã hết hạn | Tính từ `notifications.deadline_at` |
| Loại trùng, nút "Xem bản gốc" | `notifications.source_url` duy nhất một phần (xem §6) |
| Thêm/sửa/tắt nguồn, ẩn thông báo lỗi, nhập tay | `notification_sources.is_active/kind`, `notifications.is_hidden/created_by` |
| Đổi vai trò, khóa tài khoản | `profiles.role`, `profiles.is_locked` |
| Lọc tài liệu theo Khoa, Học kỳ, Loại | `documents.course_id` → `courses.faculty_id`, `documents.semester_id` → `semesters`, `doc_type` — định dạng học kỳ đã chuẩn hoá (xem §10 dòng 7) |
| Duyệt hoặc từ chối kèm lý do | `documents.status/reject_reason/reviewed_by/reviewed_at` + lịch sử đầy đủ ở `moderation_logs` |
| Tích điểm khi được duyệt, không cộng trùng | `point_transactions` + chỉ mục duy nhất một phần `(document_id, reason)` |
| Bảng xếp hạng, level | `profiles.total_points`, `profiles.level` |
| Kho template Word, PowerPoint, LaTeX | `templates.file_type` |
| AI trả lời kèm trích dẫn nguồn | `kb_chunks.page_ref` → `kb_documents.title/source_url` |
| Admin tải PDF quy chế mới vào Knowledge Base | `kb_documents` → `kb_chunks` |
| Báo cáo/gỡ tài liệu bị phản ánh (bản quyền) | `document_reports` — đã có bảng, xem §10 dòng 6 (`CODING_CONVENTION.md` §13) |
| Xem lại hỏi chatbot trước đây | `chat_sessions`, `chat_messages` — chỉ chủ phiên đọc, xem §10 dòng 1 |

## 9. Quyết định thiết kế đã chốt

> Trạng thái: **đã chốt 2026-10-16** sau phiên review của nhóm. Mục 3 và 4 đã thay đổi so với bản nháp đầu tiên, căn cứ theo các câu hỏi mở ở §10.

1. **Một kiểu `id` cho mọi bảng: `uuid`.** `profiles.id` buộc là `uuid`; dùng `uuid` cho tất cả giúp frontend và API chỉ có một kiểu id (chuỗi).
2. **Vai trò là một cột enum của `profiles`** (`profiles.role`, 3 giá trị cố định, mỗi người một vai trò), **không** tạo bảng `roles`. Khớp với `RULES.md` §4.5. > **Cần làm:** file kế hoạch đang ghi S2-02 làm bảng "vai trò" — phải sửa lại cho khớp.
3. **Thêm bảng `tags`** (`id`, `name`); `notifications.tag_id` thay cho cột chữ `tag`. Mỗi thông báo đúng một Tag, chưa làm quan hệ nhiều–nhiều. Xem §4.14.
4. **`regulations` ghi khóa bằng hai cột số `cohort_from` / `cohort_to`** (`smallint`, cùng kiểu với `profiles.cohort`), bỏ cột chữ `applicable_cohort`. Không tạo bảng `programs` (ngành). Nhờ vậy lọc quy chế theo khóa làm được thật.
5. **`regulations` và `kb_documents` tách riêng**, nối bằng `kb_documents.regulation_id` (có thể trống). Cẩm nang là nội dung để đọc; Knowledge Base có vòng đời khác (đánh chỉ mục, bật/tắt).
6. **Điểm lấy từ `point_transactions`; `profiles.total_points` là số lưu sẵn** để xếp hạng nhanh. Server cập nhật cả hai trong cùng một transaction.
7. **Lưu `file_path`**, không gắn cứng URL, vì chưa chốt Cloudinary hay Supabase Storage (S3-03).
8. **`notifications.source_url` cho phép trống** để nhập tay được thông báo không có liên kết; khi đó khóa duy nhất không áp dụng để loại trùng.
9. **`UNIQUE` trên bảng có `deleted_at` là chỉ mục duy nhất một phần** (`WHERE deleted_at IS NULL`), để sau khi xóa mềm vẫn tạo lại được bản ghi trùng giá trị cũ. Áp dụng cho `courses.code`, `notification_sources.url`, `notifications.source_url`, `tags.name`, `semesters.code`.
10. **Bảng xếp hạng đọc qua view `public_leaderboard`** (`SECURITY DEFINER`), không dựa vào RLS của `profiles` — vì RLS không giấu được từng cột (xem §7.1). View trả `full_name`, `avatar_url`, `level`, `total_points`; **không** có `student_id` hay `faculty_id`.

## 10. Câu hỏi mở — đã giải quyết

> Trạng thái: **đã quyết 2026-10-16**. Cột "Kết quả" ghi lại cách chọn và hệ quả lên ERD; mục nào sinh bảng mới thì phải thêm task tương ứng vào file kế hoạch.

| # | Hạng mục | Kết quả | Hệ quả lên ERD |
|---|---|---|---|
| 1 | Lịch sử trò chuyện với chatbot | **Có**, RLS siết chặt: chỉ chủ phiên đọc và xoá được | Thêm `chat_sessions`, `chat_messages` (§4.19–4.20). Dữ liệu cá nhân — xem §7.2 |
| 2 | Phần thưởng đổi điểm (gói màu giao diện, khung Avatar) | **Chưa làm** ở phiên bản đầu | Không đụng ERD. Nếu làm sau: cần bảng `rewards` và thêm `REDEEM` vào enum `point_reason` |
| 3 | Bảng cấu hình `levels` (mốc điểm từng cấp) | **Không**, để trong code backend | Không đụng ERD. Đổi mốc điểm phải deploy lại backend |
| 4 | Nhật ký thu thập (`crawl_runs`) | **Có** | Thêm `crawl_runs` (§4.16). Cần cho `RULES.md` §4.4 (tần suất thu thập vừa phải) và gỡ lỗi S2-03 |
| 5 | Lịch sử kiểm duyệt (`moderation_logs`) | **Có**, ghi mọi lần duyệt/từ chối | Thêm `moderation_logs` (§4.17). `documents` vẫn giữ lần duyệt gần nhất để lọc nhanh |
| 6 | Báo cáo vi phạm tài liệu (`document_reports`) | **Có**, đầy đủ kèm luồng xử lý | Thêm `document_reports` (§4.18). Đáp ứng `CODING_CONVENTION.md` §13 |
| 7 | Chuẩn hoá học kỳ | **Bảng `semesters`**, `documents.semester_id` | Thêm `semesters` (§4.15), bỏ cột chữ `documents.semester`. Nhóm nhập sẵn dữ liệu mốc bằng migration |
| 8 | Lọc quy chế theo khóa | **Đã giải quyết bởi §9.4** (`cohort_from`/`cohort_to`) | Không cần bảng `programs` |
| 9 | Che tên ở bảng xếp hạng | **Hiện tên thật, giấu MSSV và Khoa** | View `public_leaderboard` chỉ có `full_name`, `avatar_url`, `level`, `total_points` (§9.10) |

## 11. Việc còn phải làm sau khi chốt

- **Bổ sung task cho 7 bảng mới** vào file kế hoạch: `tags`/`semesters` thuộc S2-02, `crawl_runs` thuộc S2-03, `moderation_logs`/`document_reports` thuộc S3-02, `chat_sessions`/`chat_messages` thuộc S4. Nếu không sẽ có bảng trong ERD mà không ai viết migration.
- **Sửa file kế hoạch** chỗ ghi S2-02 làm bảng "vai trò" cho khớp §9.2.
- **Đối chiếu lại §8 và §10 khi có SRS** (S0-02) — hiện mọi chỗ nhắc "SRS" là tham chiếu tới tài liệu sẽ viết sau.


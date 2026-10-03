# Lộ trình thực hiện: BookShare — Chia sẻ sách báo cũ (Android)

> Phạm vi: đồ án môn Phát triển ứng dụng Mobile, nhóm 4 người (1 leader + 3 thành viên).
> Thời gian: **4 tuần** (03/10/2026 → 31/10/2026). Nếu hạn nộp khác, co giãn các tuần ở mục 8 nhưng giữ nguyên thứ tự.
> Công nghệ: Kotlin + Android Studio + MVVM/Repository + Firebase (Auth, Firestore, Storage, FCM) + Room (SQLite) + Google Sign-In.

---

## 1. Mục tiêu và tiêu chí chấm

| Tiêu chí đề bài | Cách đáp ứng trong dự án |
|---|---|
| 40% code, phân chia rõ ràng khoa học | Kiến trúc MVVM + Repository; mỗi thành viên sở hữu trọn module; code review qua Pull Request |
| 40% báo cáo PDF theo mẫu, hạn chế copy/paste và chụp màn hình không giải thích | Viết báo cáo dần theo tuần, mỗi chức năng có mô tả luồng xử lý + giải thích code, không dán ảnh trống |
| 20% trình bày 5 phút + hỏi đáp | Kịch bản demo cố định (mục 13 ở file phân công), video demo dự phòng |
| Đề tài thuộc nhóm TMĐT / tin tức / mạng xã hội | BookShare: trao đổi/tặng/bán sách cũ (thương mại điện tử nhẹ) kết hợp tương tác cộng đồng (bình luận, yêu thích — chất mạng xã hội) |
| Admin đơn giản, thao tác như người dùng thường làm trên mobile | Admin tích hợp ngay trong app, duyệt/ẩn/xóa bài và khóa tài khoản — thao tác giống kiểm duyệt bài viết trên Facebook |
| Bắt buộc dùng Firebase | Authentication, Firestore, Storage, Cloud Messaging |
| Responsive khi xoay màn hình | ViewModel giữ state khi xoay; `layout-land/` riêng cho các màn chính |
| Khuyến khích sync offline với SQLite | Room là lớp bọc SQLite; kiến trúc single source of truth (Firestore → Room → UI) |
| Khai thác API hỗ trợ | Google Sign-In (xác thực), Firebase Cloud Messaging (đẩy thông báo) |
| Làm việc nhóm có minh chứng | GitHub Project đã tạo sẵn; quy trình branch → PR → review → merge; ảnh chụp đưa vào báo cáo |

Nguyên tắc: không tự dựng backend riêng ngoài Firebase; mọi dữ liệu đi qua Firestore/Storage, không hardcode dữ liệu mẫu trong code.

---

## 2. Phân tích nghiệp vụ (BA)

### 2.1 Tác nhân
- **Khách** (chưa đăng nhập): xem danh sách sách, xem chi tiết, tìm kiếm.
- **Thành viên**: khách + đăng bài, bình luận, yêu thích, gửi yêu cầu nhận sách, sửa hồ sơ.
- **Quản trị viên**: duyệt/ẩn/xóa bài đăng vi phạm, khóa/mở khóa tài khoản.

### 2.2 Yêu cầu chức năng

| Mã | Chức năng | Ưu tiên |
|---|---|---|
| FR01 | Đăng ký, đăng nhập, đăng xuất bằng Email | Cao |
| FR02 | Đăng nhập bằng Google (Google Sign-In) | Cao |
| FR03 | Hồ sơ cá nhân (xem/sửa, đổi avatar) | Trung bình |
| FR04 | Trang chủ: danh sách sách theo danh mục, dạng lưới | Cao |
| FR05 | Tìm kiếm + bộ lọc (danh mục, tình trạng, sắp xếp) | Cao |
| FR06 | Đăng, sửa, xóa bài đăng sách (ảnh, thông tin, tình trạng) | Cao |
| FR07 | Chi tiết sách (ảnh, mô tả, thông tin người đăng) | Cao |
| FR08 | Bình luận trên bài đăng | Cao |
| FR09 | Yêu thích (lưu/bỏ lưu sách) | Cao |
| FR10 | Yêu cầu nhận sách (gửi/đồng ý/từ chối) | Cao |
| FR11 | Thông báo đẩy (FCM) khi có yêu cầu/bình luận mới | Trung bình |
| FR12 | Đồng bộ offline (Room): xem lại được trang đã mở khi mất mạng | Cao — bắt buộc theo đề |
| FR13 | Responsive khi xoay màn hình | Cao — bắt buộc theo đề |
| FR14 | Admin: duyệt/ẩn/xóa bài đăng | Cao |
| FR15 | Admin: khóa/mở khóa người dùng | Thấp |

### 2.3 Quy tắc nghiệp vụ (cần chốt ở tuần 1)
- Trạng thái bài đăng: `available` (đang hiển thị), `reserved` (đã có người nhận), `given` (đã cho/bán), `hidden` (admin ẩn). Chốt: bài đăng mới có cần duyệt trước hay hiển thị ngay.
- Chỉ tác giả bài đăng và admin được sửa/xóa.
- Mỗi người chỉ yêu thích một bài một lần (khóa chính kép `userId + bookId`).
- Tài khoản bị khóa không đăng nhập được.
- Yêu cầu nhận sách: chỉ chủ bài đăng được đồng ý/từ chối; một bài có thể nhận nhiều yêu cầu nhưng chỉ chấp nhận một.
- Bình luận thuộc đúng một bài; chỉ tác giả bình luận và admin được xóa.

### 2.4 Phi chức năng
- Responsive, hiển thị tốt khi xoay ngang/dọc.
- Tiếng Việt có dấu đầy đủ.
- Firestore Security Rules chặn sửa/xóa dữ liệu của người khác; validate input ở cả client và rules.
- Hoạt động được ở chế độ offline với dữ liệu đã từng tải (Room).

### 2.5 Sản phẩm BA cần nộp trong tuần 1
- Bảng yêu cầu (mục 2.2), sơ đồ use case.
- Wireframe các màn chính (đã có mô tả 22 màn hình, đưa qua Google Stitch → Figma).
- Sơ đồ cấu trúc dữ liệu Firestore + sơ đồ bảng Room.

---

## 3. Công nghệ và cách áp dụng

| Công nghệ | Áp dụng cụ thể |
|---|---|
| Kotlin + MVVM | Mỗi màn hình: `Activity/Fragment` (View) + `ViewModel` + `Repository`; không xử lý logic trong View |
| Repository pattern | Lớp `Repository` làm nguồn dữ liệu duy nhất, ẩn việc dữ liệu đến từ Firestore hay Room |
| Firebase Authentication | Email/Password + Google Sign-In (cấu hình SHA-1) |
| Firestore | Lưu `users`, `books`, `comments`, `requests`; đọc bằng Flow/LiveData realtime |
| Firebase Storage | Lưu ảnh bìa sách, ảnh avatar |
| Firebase Cloud Messaging | Đẩy thông báo khi có yêu cầu nhận sách / bình luận mới |
| Room (SQLite) | Cache dữ liệu đã tải để xem offline; entity + DAO cho Book, User, Favorite |
| Material Design 3 | Giao diện chuẩn Android, hỗ trợ dark mode (tùy chọn) |
| Glide/Coil | Tải và cache ảnh, hỗ trợ hiển thị khi offline nếu ảnh đã tải trước đó |

Nguyên tắc phân tầng: Firestore Security Rules chỉ kiểm tra quyền truy cập dữ liệu; validate nghiệp vụ (độ dài, định dạng) nằm ở ViewModel trước khi gửi lên.

---

## 4. Cấu trúc thư mục

```
bookshare/
├── app/
│   ├── src/main/java/com/yourteam/bookshare/
│   │   ├── data/
│   │   │   ├── model/              # Book, User, Comment, Request
│   │   │   ├── remote/              # FirestoreDataSource, StorageDataSource
│   │   │   ├── local/                # Room: entity, dao, database
│   │   │   └── repository/           # BookRepository, AuthRepository, RequestRepository...
│   │   ├── ui/
│   │   │   ├── auth/                 # Login, Register, ForgotPassword
│   │   │   ├── home/                 # Home, Search
│   │   │   ├── detail/               # BookDetail, Comments
│   │   │   ├── post/                 # CreatePost, EditPost
│   │   │   ├── profile/              # Profile, EditProfile, Settings
│   │   │   ├── requests/             # Requests (sent/received), Favorites
│   │   │   ├── notifications/
│   │   │   └── admin/                # AdminDashboard, UserManagement
│   │   ├── util/                     # Constants, NetworkUtil, Extensions
│   │   └── BookShareApp.kt
│   ├── src/main/res/
│   │   ├── layout/   layout-land/    # bản responsive khi xoay ngang
│   │   ├── values/   drawable/ ...
│   └── google-services.json          # gitignore bản thật, commit bản .example
├── docs/
│   ├── srs.md, erd/, wireframes/
│   └── report-evidence/              # ảnh minh chứng theo tuần
├── .gitignore
└── README.md                          # cách build, cấu hình Firebase, chạy dự án
```

---

## 5. Cơ sở dữ liệu

### 5.1 Firestore (nguồn dữ liệu chính)
- `users/{uid}`: name, email, avatarUrl, role (`user`/`admin`), status (`active`/`locked`), createdAt.
- `books/{bookId}`: title, author, category, condition, description, imageUrls, ownerId, status, createdAt.
- `books/{bookId}/comments/{id}`: userId, content, createdAt.
- `requests/{id}`: bookId, requesterId, ownerId, status (`pending`/`accepted`/`rejected`).
- `users/{uid}/favorites/{bookId}`: savedAt.

### 5.2 Room (cache offline)
- `BookEntity`, `UserEntity`, `FavoriteEntity` — bản sao rút gọn của dữ liệu Firestore đã từng tải.
- `PendingActionEntity` (tùy chọn, nếu còn thời gian) — lưu thao tác offline để đồng bộ khi có mạng lại.

### 5.3 Quy tắc làm việc với dữ liệu
- Mọi thay đổi cấu trúc Firestore (thêm field, đổi tên collection) phải báo trong nhóm chat và cập nhật lại `docs/erd/`.
- Firestore Security Rules do **Leader duyệt** trước khi merge.
- Room schema version tăng dần, không sửa tay entity đã merge mà không tạo migration hoặc `fallbackToDestructiveMigration` (chỉ dùng tạm trong lúc phát triển).
- Firestore luôn kèm `createdAt`/`updatedAt` để sắp xếp và đồng bộ.

---

## 6. Dịch vụ và API sử dụng

| Dịch vụ | Dùng ở |
|---|---|
| Firebase Authentication (Email + Google) | Màn Đăng nhập/Đăng ký |
| Firestore (CRUD realtime) | Trang chủ, Chi tiết, Đăng bài, Bình luận, Yêu cầu |
| Firebase Storage | Upload ảnh bìa sách, avatar |
| Firebase Cloud Messaging | Thông báo yêu cầu mới / bình luận mới |
| Room (local) | Cache offline cho Trang chủ, Chi tiết, Yêu thích |
| Firestore Security Rules | Phân quyền đọc/ghi theo `uid` và `role` |

Mọi thao tác ghi dữ liệu quan trọng (đăng bài, gửi yêu cầu, admin xóa bài) nên có xử lý lỗi rõ ràng (hiện Toast/Snackbar), không để app crash khi mất mạng.

---

## 7. Phân công nhóm 4 người

| Người | Vai trò | Chức năng | Sở hữu |
|---|---|---|---|
| **Leader** | BA, kiến trúc, dữ liệu, deploy | SRS, cấu trúc Firestore + Room, khung MVVM/Repository, Security Rules, **FR14**, khung Admin, review PR, build release, tổng hợp báo cáo | `data/`, `admin/`, Firestore Rules, Room database class |
| **Khang** | Người dùng và hồ sơ | **FR01, FR02, FR03, FR15** | `ui/auth`, `ui/profile`, `AuthRepository`, `UserRepository` |
| **Phúc** | Sách và tìm kiếm (lõi forum) | **FR04, FR05, FR06, FR07** | `ui/home`, `ui/detail`, `ui/post`, `BookRepository` |
| **Vương** | Tương tác, offline, thông báo | **FR08, FR09, FR10, FR11, FR12, FR13** (phối hợp Leader cho FR12/FR13) | `ui/requests`, `ui/notifications`, `data/local` (Room), FCM |

Quy ước:
- Mỗi người chỉ sửa file thuộc module mình sở hữu. File dùng chung (`data/model`, Room database class, layout chung) do Leader duyệt; người khác cần gì thì tạo issue.
- Kiểm thử chéo: Khang test Phúc, Phúc test Vương, Vương test Khang, Leader test admin và tích hợp toàn bộ.
- Mỗi người tự viết phần "chức năng do tôi phụ trách" trong báo cáo, kèm ảnh và commit của chính mình.

---

## 8. Lộ trình 4 tuần

### Tuần 1 (03/10 – 09/10): BA và nền tảng
**Mục tiêu:** chốt yêu cầu, dựng hạ tầng nhóm, đăng nhập hoạt động.

Leader: viết SRS/use case, vẽ ERD Firestore + Room, dựng cấu trúc thư mục, `.gitignore`, `README`, thiết lập GitHub (`main`/`develop`, branch protection, Issue theo mã FR trong GitHub Project), khung MVVM/Repository, kết nối Firebase.

Khang: cài đặt môi trường, FR01 (đăng ký/đăng nhập Email), FR02 (Google Sign-In).

Phúc: dựng view tĩnh Trang chủ/Chi tiết/Đăng bài theo Figma.

Vương: dựng view tĩnh Yêu cầu/Thông báo, nghiên cứu Room + FCM.

**Bàn giao:** SRS, ERD, wireframe, backlog đầy đủ trên GitHub Project, đăng nhập Email + Google chạy được.

### Tuần 2 (10/10 – 16/10): Chức năng lõi
Leader: hoàn thiện Repository chung, Firestore Security Rules bản đầu, hỗ trợ tích hợp.
Khang: FR03 hồ sơ cá nhân.
Phúc: FR04 Trang chủ, FR05 Tìm kiếm, FR06 Đăng/sửa/xóa bài, FR07 Chi tiết sách.
Vương: FR08 Bình luận, FR09 Yêu thích.

**Bàn giao:** luồng chính hoạt động online — đăng ký → đăng bài → người khác tìm thấy, yêu thích, bình luận.

### Tuần 3 (17/10 – 23/10): Offline, Responsive, Admin
Leader: FR14 Admin duyệt/ẩn/xóa bài, hoàn thiện Security Rules chính thức.
Khang: FR15 Admin quản lý người dùng.
Phúc: Responsive (`layout-land/`) cho Trang chủ, Chi tiết, Đăng bài.
Vương: FR10 Yêu cầu nhận sách, FR12 Room/offline, FR11 FCM, hỗ trợ Responsive cho các màn còn lại.

**Bàn giao:** đáp ứng đủ yêu cầu kỹ thuật bắt buộc (Firebase + offline SQLite + responsive + Google API), có admin.

### Tuần 4 (24/10 – 31/10): Kiểm thử, Báo cáo, Trình bày
Cả nhóm: kiểm thử chéo, test đa thiết bị (xoay ngang + chế độ máy bay), sửa bug, hoàn thiện báo cáo PDF.
Leader: tổng hợp báo cáo, chuẩn bị bản release/APK, chụp minh chứng GitHub (Network graph, PR đã merge).
Cả nhóm: luyện kịch bản demo 5 phút, quay video dự phòng.

**Bàn giao (31/10):** code trên GitHub, báo cáo PDF, app chạy ổn định, sẵn sàng demo.

### Nhịp làm việc mỗi tuần
- Đầu tuần: họp phân task.
- Giữa tuần: họp 15 phút gỡ vướng.
- Cuối tuần: demo ngắn và ghi biên bản (ai làm gì, ai vướng gì, việc tuần sau).

---

## 9. Thứ tự ưu tiên khi bị trễ

1. **Giữ chắc:** FR01, FR02, FR04, FR05, FR06, FR07, FR12, FR13, FR14, kiến trúc MVVM, Firebase, minh chứng nhóm trên GitHub.
2. **Giữ vì điểm cộng:** FR08 (bình luận), FR09 (yêu thích), FR11 (FCM).
3. **Cắt đầu tiên nếu trễ:** FR15 (admin khóa user), FR10 có thể rút gọn (bỏ trạng thái từ chối, chỉ giữ gửi/đồng ý), đồng bộ offline khi ghi dữ liệu (giữ offline đọc, bỏ offline ghi).

---

## 10. Quy trình GitHub

- **Nhánh:** `main` (bảo vệ, chỉ nhận từ `develop`), `develop`, `feature/FR06-post-crud`, `fix/...`.
- **Branch protection:** bắt buộc Pull Request, ít nhất 1 review, không push trực tiếp lên `main`.
- **Issue:** tiêu đề dạng `[FR06] Đăng/sửa/xóa bài`, gán người, nhãn (`ui`, `data`, `firebase`, `bug`, `docs`), milestone theo tuần.
- **Project board:** Backlog → Ready → In Progress → In Review → Done; PR liên kết issue bằng `Closes #số`.
- **Commit:** `feat: thêm đăng nhập Google`, `fix: lỗi xoay màn hình trang chủ`, `docs: cập nhật README`.
- Nhánh `feature/` sống tối đa vài ngày rồi merge vào `develop`.
- Đầu mỗi ngày làm việc: `git pull origin develop`.

**Definition of Done cho một task:** chạy đúng trên máy người làm; có review; không crash; không lộ API key/`google-services.json` thật trong lịch sử Git; đã test trên cả chế độ dọc/ngang nếu là màn hình chính; đã có ảnh minh chứng nếu là chức năng chính.

---

## 11. Build và chuẩn bị release

**Yêu cầu:** Android Studio phiên bản mới, project Firebase đã bật Auth/Firestore/Storage/FCM, `google-services.json` đúng package name.

Các bước:
1. Kiểm tra `applicationId` khớp với Firebase project.
2. Build **debug APK** để test trên nhiều thiết bị/emulator trong tuần 4.
3. (Tùy chọn, nếu còn thời gian) build **release APK** có ký (keystore) để nộp kèm báo cáo — không bắt buộc nếu đề chỉ yêu cầu mã nguồn + demo.
4. Chạy checklist trước demo:
   - [ ] Đăng ký, đăng nhập Email, đăng nhập Google, đăng xuất
   - [ ] Đăng bài kèm ảnh, sửa, xóa
   - [ ] Bình luận, yêu thích
   - [ ] Tìm kiếm, bộ lọc
   - [ ] Yêu cầu nhận sách (gửi — đồng ý/từ chối)
   - [ ] Offline: tắt mạng, mở lại trang đã xem trước đó
   - [ ] Xoay ngang trên Trang chủ và Chi tiết sách
   - [ ] Admin: duyệt/ẩn/xóa bài, khóa/mở khóa user
5. Sao lưu một bản APK ổn định trước ngày demo, phòng khi bản mới nhất có lỗi phát sinh.

---

## 12. Minh chứng cần chụp cho báo cáo (làm dần theo tuần)

| Nhóm minh chứng | Nội dung ảnh chụp |
|---|---|
| GitHub | Sơ đồ nhánh (Insights → Network), danh sách PR đã review và merge, commit từng người |
| Quản lý tiến độ | GitHub Project board có task ở nhiều cột, milestone theo tuần |
| Firebase Console | Danh sách user trong Authentication, dữ liệu trong Firestore, file trong Storage (che thông tin nhạy cảm nếu cần) |
| Nhóm làm việc | Kênh trao đổi, ảnh họp nhóm, biên bản tuần |
| Sản phẩm | Ảnh từng chức năng chính, cả chế độ dọc lẫn ngang |
| Kỹ thuật | Cấu trúc package, ví dụ ViewModel/Repository, Security Rules, màn hình offline |

Lưu ảnh vào `docs/report-evidence/tuần-N/` ngay khi làm để cuối kỳ không phải chụp bù.

---

## 13. Mục lục báo cáo gợi ý

1. Giới thiệu đề tài và phạm vi
2. Phân tích yêu cầu (FR, use case)
3. Thiết kế (kiến trúc MVVM, cấu trúc Firestore/Room, wireframe)
4. Công nghệ và cách áp dụng (Firebase, Room, Google Sign-In, responsive)
5. Quản lý nhóm (Git, GitHub Project) kèm ảnh minh chứng
6. Kết quả (ảnh từng chức năng, cả hai hướng màn hình)
7. Phân công và đóng góp từng thành viên
8. Kết luận và hướng phát triển

---

## 14. Rủi ro và cách xử lý

| Rủi ro | Cách xử lý |
|---|---|
| Google Sign-In lỗi do SHA-1/cấu hình Firebase | Làm sớm trong tuần 1, test trên thiết bị thật, không chỉ emulator |
| Offline (Room) làm trễ vì đụng kiến trúc | Bắt đầu từ tuần 2 (model/entity song song), không dồn cuối tuần 3 |
| Thành viên lệch tiến độ | Xem board hằng tuần, chia lại task sớm |
| Làm sát hạn | Viết báo cáo và chụp minh chứng dần theo tuần, không dồn tuần 4 |
| Firestore đọc/ghi quá nhiều lần (tốn quota) | Dùng Flow/LiveData theo dõi thay vì gọi lại thủ công, cache qua Room |
| Lộ `google-services.json`/API key thật trong Git | Thêm vào `.gitignore`, rà lịch sử commit trước khi nộp |

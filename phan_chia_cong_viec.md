# Phân chia công việc — Website đặt vé xem phim

## 1. Ngôn ngữ và công nghệ đề xuất

### Lựa chọn phù hợp nhất với nhóm
- **Backend:** Java 17 hoặc 21, Spring Boot theo mô hình MVC.
- **Giao diện:** HTML, CSS, một ít JavaScript; dùng Thymeleaf để render trang từ Java. Có thể dùng Bootstrap để làm giao diện nhanh.
- **Cơ sở dữ liệu:** MySQL.
- **Kết nối DB:** Spring Data JPA/Hibernate; viết truy vấn SQL khi cần báo cáo hoặc khi giảng viên yêu cầu.
- **Build:** Maven; quản lý mã nguồn bằng Git và thống nhất nhánh `main`/nhánh tính năng.
- **Mật khẩu:** Spring Security và BCrypt (chỉ lưu chuỗi hash, tuyệt đối không lưu mật khẩu gốc).

## 2. Kiến trúc và quy ước dùng chung

### Luồng xử lý
`Browser → Controller → Service → Repository → MySQL`

- **Controller:** nhận request, kiểm tra đầu vào cơ bản, trả trang hoặc điều hướng; không viết truy vấn SQL và nghiệp vụ dài ở đây.
- **Service:** chứa quy tắc nghiệp vụ, ví dụ kiểm tra quyền, trạng thái phim, chỗ ngồi còn trống.
- **Repository:** đọc/ghi dữ liệu.
- **Entity/Model:** ánh xạ dữ liệu và quan hệ.
- **Template:** các trang Thymeleaf; CSS/ảnh để trong `static`.

Không để mỗi thành viên tạo một project hoặc một cấu hình kết nối DB riêng. Cùng dùng một project, một file cấu hình mẫu và cùng quy ước tên.

### Các bảng chính cần thống nhất sớm
- `users`: tài khoản, email duy nhất, `password_hash`, trạng thái, vai trò.
- `movies`, `categories`, `movie_category`: phim và thể loại.
- `theaters`, `rooms`, `seats`, `showtimes`: rạp, phòng, ghế và lịch chiếu.
- `bookings`, `booking_seats`, `payments`: đơn đặt, ghế đã đặt và giao dịch giả lập.

Các tên trên là gợi ý; cả nhóm cần thống nhất khóa chính/khóa ngoại, kiểu dữ liệu, tên trạng thái và lưu thành migration hoặc file `schema.sql` trước khi chia nhau làm sâu. Không để mỗi người tự sửa schema theo cách không báo cho nhóm.

### Quy tắc bàn giao giữa các phần
- Người 1 bàn giao API/service xác thực và thông tin người dùng hiện tại cho các trang cần đăng nhập.
- Người 2 bàn giao hàm lấy danh sách phim đang hiển thị và chi tiết phim cho người 3 tạo lịch chiếu.
- Người 3 bàn giao lịch chiếu cho người 4; người 4 bàn giao booking đã tạo cho người 5 xử lý thanh toán.
- Dùng chung ID entity và DB, không truyền nguyên đối tượng qua các module nếu chỉ cần ID.

## 3. Phân công theo thành viên

### Người 1 — Tài khoản, đăng nhập và phân quyền (User & Authentication)

**Mục tiêu:** người dùng đăng ký/đăng nhập được; người chưa đăng nhập, khách hàng và admin chỉ truy cập đúng chức năng của mình.

#### Phạm vi công việc
1. Đăng ký tài khoản Customer; đăng nhập, đăng xuất và xem/sửa hồ sơ cá nhân.
2. Phân quyền tối thiểu `CUSTOMER` và `ADMIN`. Chỉ mở rộng `STAFF` khi luồng sử dụng đã rõ (ví dụ được xem/soát vé nhưng không quản lý phim).
3. Bảo vệ các trang quản trị ở phía server, không chỉ ẩn nút Admin trên giao diện.
4. Tạo một tài khoản Admin mẫu bằng seed data hoặc hướng dẫn cấu hình; không cho người dùng tự đăng ký với quyền Admin.
5. Quên mật khẩu: nếu không có email thật, làm luồng demo có token/mã dùng một lần và ghi rõ là giả lập. Không gửi mật khẩu cũ hoặc đặt mật khẩu mặc định dễ đoán.

#### Dữ liệu gợi ý
`User`: `id`, `fullName`, `email` (unique), `passwordHash`, `phone` (nếu cần), `role`, `enabled`, `createdAt`.

Email nên được chuẩn hóa (trim, lowercase) và kiểm tra trùng. Không lưu mật khẩu gốc. Dùng BCrypt qua Spring Security; không tự viết thuật toán mã hóa mật khẩu.

#### Gợi ý cấu trúc Java
- `User` entity/model và `Role` enum.
- `UserRepository`: tìm theo email, kiểm tra email đã tồn tại.
- `AuthService`/`UserService`: đăng ký, cập nhật hồ sơ, đổi hoặc đặt lại mật khẩu.
- Controller cho `/register`, `/login`, `/logout`, `/profile`; cấu hình bảo vệ đường dẫn `/admin/**`.
- DTO/form riêng cho đăng ký và cập nhật hồ sơ; không bind trực tiếp trường `role` hoặc `passwordHash` từ form người dùng.

#### Chọn cách thể hiện OOP
Về thiết kế ứng dụng, phương án đơn giản và dễ quản lý nhất là **một `User` entity có `Role` enum** (`CUSTOMER`, `ADMIN`, có thể thêm `STAFF`). Phân quyền được cấu hình rõ ở Spring Security.

Nếu giảng viên **bắt buộc** phải có `abstract class User`, `Admin` và `Customer` kế thừa, hãy xác nhận trước rồi mới làm. Khi đó có thể dùng JPA inheritance kiểu `SINGLE_TABLE`, nhưng sẽ phức tạp hơn khi tạo tài khoản, truy vấn và phân quyền. Tránh vừa có subclass vừa có một trường role hoạt động độc lập mà không quy định cái nào là nguồn dữ liệu chính.

#### Trình tự thực hiện
1. Cùng nhóm chốt schema `users`, vai trò và cách đăng nhập (session/Spring Security).
2. Tạo entity, repository và seed Admin; chạy thử kết nối MySQL.
3. Làm đăng ký và validation: email đúng định dạng, email không trùng, mật khẩu đạt quy tắc tối thiểu.
4. Làm đăng nhập/đăng xuất; kiểm tra mật khẩu qua BCrypt.
5. Bảo vệ trang quản trị ở server; thử truy cập URL Admin trực tiếp bằng tài khoản Customer và trạng thái chưa đăng nhập.
6. Làm hồ sơ cá nhân và luồng quên mật khẩu demo nếu còn thời gian.
7. Viết test và bàn giao hàm/API cho các thành viên khác.

#### Tiêu chí hoàn thành
- Đăng ký hợp lệ tạo tài khoản Customer; email trùng bị từ chối.
- Database chỉ có password hash; đăng nhập đúng thành công, sai thất bại.
- Customer không truy cập được chức năng Admin kể cả khi nhập URL trực tiếp.
- Admin mẫu đăng nhập được; cập nhật hồ sơ không thể tự đổi role.
- Có hướng dẫn cấu hình tài khoản DB và dữ liệu Admin mẫu trong README, không commit mật khẩu thật.

#### Người 1 không phụ trách
Không tự sửa module phim/lịch chiếu/đặt vé. Khi module khác cần thông tin người dùng, cung cấp phương thức dùng chung để lấy user hiện tại thay vì mỗi người tự truy vấn bảng `users`.

### Người 2 — Phim và thể loại (Movie Catalog)

**Mục tiêu:** Admin quản lý được phim; khách xem được phim đang chiếu/sắp chiếu, tìm kiếm, lọc và xem chi tiết.

#### Dữ liệu gợi ý
- `Movie`: `id`, `title`, `director`, `durationMinutes`, `releaseDate`, `ageRating`, `synopsis`, `posterUrl`, `trailerUrl`, `status` hoặc `visible`, `createdAt`.
- `Category`: `id`, `name` (unique).
- `movie_category`: khóa ngoại `movie_id`, `category_id`, khóa chính ghép để tránh gắn trùng.

Một phim có thể có nhiều thể loại và một thể loại có thể thuộc nhiều phim, vì vậy dùng `@ManyToMany` hoặc entity trung gian `MovieCategory`; không dùng composition cho quan hệ này. Nếu cần lưu thêm thông tin trên lần gắn thể loại (ví dụ thứ tự/ghi chú), dùng entity trung gian.

#### Trạng thái và xóa phim
- Tách **trạng thái theo ngày chiếu** khỏi **ẩn/hiện trên website**. Trạng thái `NOW_SHOWING`/`COMING_SOON` có thể suy ra từ `releaseDate` và ngày hiện tại; trường `visible`/`active` dùng để Admin ẩn nội dung.
- Không xóa cứng phim đã được lịch chiếu/đơn đặt tham chiếu. Dùng ẩn/soft delete hoặc từ chối xóa khi đã có dữ liệu liên quan.
- Để đơn giản đồ án, poster/trailer có thể là URL; chỉ làm upload file nếu yêu cầu và thời gian cho phép.

#### Chức năng
- Admin: danh sách phim, thêm, sửa, kiểm tra dữ liệu, ẩn/hiện phim; quản lý thể loại nếu nằm trong phạm vi.
- Khách: danh sách phim đang chiếu/sắp chiếu, trang chi tiết, tìm theo tên và lọc theo thể loại.
- Sắp xếp theo ngày ra mắt hoặc tên; phân trang chỉ làm nếu số dữ liệu lớn hoặc đề bài yêu cầu.

#### Gợi ý cấu trúc Java
- `Movie` và `Category` entity/model.
- `MovieRepository`: tìm phim hiển thị, tìm theo tên, lọc theo category, lấy chi tiết.
- `MovieService`: kiểm tra thời lượng/ngày/tuổi, xử lý trạng thái hiển thị và quy tắc xóa.
- Controller cho trang danh sách, chi tiết và các form Admin; chỉ Admin được tạo/sửa/ẩn.
- Các template như `movies/list`, `movies/detail`, `admin/movies/form`.

#### Trình tự thực hiện
1. Chốt với nhóm các trường phim, category và trạng thái; tạo migration/seed vài phim mẫu.
2. Làm entity/repository và xác nhận quan hệ phim–thể loại chạy được trên MySQL.
3. Làm chức năng xem danh sách và chi tiết trước để người 3 có dữ liệu phim tạo lịch chiếu.
4. Làm tìm kiếm/lọc/sắp xếp; xử lý trường hợp không có kết quả và phim đã bị ẩn.
5. Làm form Admin thêm/sửa/ẩn; kiểm tra đầu vào cả server-side.
6. Viết test cho tìm kiếm, lọc, trạng thái và bàn giao phương thức lấy phim cho người 3.

#### Tiêu chí hoàn thành
- Khách chỉ thấy phim được phép hiển thị; lọc và tìm kiếm cho kết quả chính xác.
- Admin thêm/sửa/ẩn được phim; dữ liệu sai như thời lượng âm hoặc thiếu tên bị từ chối.
- Có thể gắn một phim với nhiều thể loại và một thể loại với nhiều phim.
- Phim đang được lịch chiếu/đặt vé sử dụng không bị xóa gây hỏng khóa ngoại.
- Có dữ liệu mẫu và hướng dẫn trang/luồng kiểm thử.

#### Người 2 không phụ trách
Không tự quản lý phòng chiếu, lịch chiếu, ghế hoặc booking. Người 3 chỉ cần nhận `movieId` và các hàm truy vấn phim đã thống nhất.

### Người 3 — Rạp, phòng và lịch chiếu (Theater & Showtime)

- Quản lý cụm rạp, phòng chiếu, loại phòng và sức chứa.
- Tạo/sửa/ẩn lịch chiếu gắn với phim và phòng.
- Kiểm tra lịch chiếu trùng trong cùng phòng; cần tính cả thời lượng phim và thời gian dọn phòng nếu nhóm đưa vào phạm vi.
- Loại phòng 2D/3D/IMAX có thể thể hiện bằng interface/strategy tính phụ phí; không cần tạo class con nếu không có hành vi khác nhau.
- Bàn giao `showtimeId`, thời gian, phòng và giá cơ bản cho người 4.

### Người 4 — Đặt vé và chọn ghế (Booking Core)

- Hiển thị sơ đồ ghế theo phòng/lịch chiếu; phân biệt ghế thường/VIP/đôi.
- Tính giá ở server; giao diện không được tự quyết định số tiền cuối.
- Khi xác nhận booking phải kiểm tra lại ghế còn trống trong transaction. Có unique constraint theo `(showtime_id, seat_id)` trong bảng ghế của booking để hạn chế hai người đặt cùng ghế.
- Nếu làm giữ ghế tạm thời, cần thời hạn hết hạn; nếu không đủ thời gian, chỉ giữ ghế ở bước xác nhận booking.
- Chặn quá 5 vé mỗi booking nếu đây là quy tắc của đề bài; hiển thị lỗi dễ hiểu.

### Người 5 — Thanh toán giả lập, vé và thống kê (Payment & Dashboard)

- Xác nhận thanh toán giả lập, tạo hóa đơn/vé và lịch sử đặt vé.
- Chỉ tính doanh thu từ giao dịch đã thanh toán thành công; thống nhất trạng thái booking/payment với người 4 trước.
- Có thể dùng `PaymentStrategy` để minh họa OOP; không nhập/lưu thông tin thẻ thật. Momo/ZaloPay trong đồ án chỉ nên là phương thức giả lập nếu không tích hợp cổng thật.
- Làm thống kê theo tháng/phim/rạp dựa trên dữ liệu booking/payment nhất quán.

## 4. Kiểm thử, tích hợp và cách làm việc nhóm

- Mỗi thành viên tạo nhánh theo tính năng, commit nhỏ và mở pull request để ít nhất một người khác xem trước khi ghép vào `main`.
- Thống nhất format thời gian, tiền tệ, tên bảng/cột, enum trạng thái và quy tắc đặt tên Java ngay từ đầu.
- Không commit mật khẩu DB, khóa bí mật hoặc dữ liệu cá nhân thật. Dùng `application-example.properties`/biến môi trường làm cấu hình mẫu.
- Có test cho nghiệp vụ chính: đăng nhập sai/đúng và phân quyền; tìm/lọc phim; trùng lịch chiếu; đặt ghế trùng; thanh toán thất bại/thành công.
- Chạy thử từ database rỗng bằng schema/migration và seed data. Mỗi người không chỉ kiểm tra trên máy của mình.
- README cần có: yêu cầu JDK, cách tạo DB, cấu hình kết nối, lệnh chạy, tài khoản demo, chức năng đã làm và giới hạn còn lại.

## 5. Thứ tự triển khai gợi ý

1. **Cả nhóm:** xác nhận rubric/đề bài, chốt stack, vẽ luồng người dùng, schema và giao diện tối thiểu.
2. **Khung project:** dựng ứng dụng Spring Boot kết nối MySQL, thêm layout chung và dữ liệu mẫu; kiểm tra máy tất cả thành viên chạy được.
3. **Làm song song:** người 1 làm xác thực; người 2 làm catalog; người 3 chuẩn bị rạp/phòng; người 4 chốt mô hình ghế/booking với người 3; người 5 thống nhất trạng thái booking/payment với người 4.
4. **Ghép theo luồng:** xem phim → chọn lịch → chọn ghế → tạo booking → thanh toán giả lập → xem vé/lịch sử.
5. **Hoàn thiện:** test quyền và lỗi, seed dữ liệu demo, sửa giao diện, viết README và tập demo/bảo vệ.

Ưu tiên làm một luồng đặt vé tối thiểu chạy xuyên suốt trước khi thêm biểu đồ, email thật, upload ảnh hoặc tích hợp cổng thanh toán.

## 6. Tham khảo giao diện

- Beta Cinemas: https://betacinemas.vn/
- Chỉ tham khảo bố cục và cách trình bày; không sao chép nguyên logo, nội dung, hình ảnh hoặc nhận diện thương hiệu.

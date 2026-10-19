# Test Plan - Clothing Shop

## 1. Mục tiêu

Xác minh các luồng mua hàng và thuê sản phẩm hoạt động đúng, dữ liệu và tồn kho nhất quán, đồng thời ngăn người dùng truy cập dữ liệu hoặc chức năng ngoài quyền hạn.

## 2. Phạm vi

**Trong phạm vi**

- Đăng ký, đăng nhập, phiên người dùng và phân quyền USER/ADMIN.
- Duyệt danh mục/sản phẩm, chọn biến thể kích cỡ-màu sắc.
- Giỏ hàng: thêm, sửa số lượng, xóa, giới hạn theo tồn kho và quyền sở hữu.
- Checkout: tạo đơn, tổng tiền, thanh toán COD, trừ tồn kho và làm trống giỏ.
- Đơn hàng: xem danh sách/chi tiết, quyền truy cập, hủy đơn và hoàn tồn kho.
- Thuê: kiểm tra ngày, số ngày, tổng tiền, tồn kho, hủy và chuyển trạng thái.
- Chức năng quản trị sản phẩm, danh mục, đơn hàng, thuê, người dùng và báo cáo.
- Hiển thị responsive cơ bản và các lỗi nhập liệu phổ biến.

**Ngoài phạm vi**

- Tích hợp cổng thanh toán/bên thứ ba không có trong ứng dụng hiện tại.
- Kiểm thử tải quy mô lớn và kiểm thử production.
- Nghiệp vụ chưa được xác định, ví dụ chính sách phí giao hàng hoặc phạt trả thuê trễ.

## 3. Chiến lược kiểm thử

| Tầng            | Phạm vi                                                                      | Công cụ/điều kiện đề xuất                                              |
| --------------- | ---------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Unit            | Validate input, tính tổng tiền/số ngày, quy tắc chuyển trạng thái            | Vitest; hiện chưa có test runner hoặc script test trong `package.json` |
| Integration     | API route/controller, service, Prisma transaction, quyền truy cập và dữ liệu | Vitest với MariaDB test riêng; không dùng database production          |
| E2E trình duyệt | Luồng người dùng từ UI đến API và database                                   | Playwright trên Chromium; ứng dụng chạy với dữ liệu test               |
| Smoke           | Ứng dụng khởi động, trang chính/API quan trọng phản hồi, đăng nhập test      | Chạy sau build/deploy lên môi trường test                              |

Với luồng thanh toán giả lập hiện tại, xác nhận trạng thái COD và bản ghi Payment; không gọi dịch vụ thanh toán thật. Kiểm thử concurrency checkout cần kiểm tra rằng tồn kho không âm và chỉ số lượng còn hàng được đặt thành công.

## 4. Môi trường và dữ liệu

- Node.js hiện có: `v22.13.0`.
- Ứng dụng cần Next.js 16, Prisma 7 và MariaDB; cấu hình kết nối lấy từ `DATABASE_URL`.
- Docker daemon hiện không khả dụng. Integration/E2E cần MariaDB test độc lập đang chạy cục bộ hoặc trong CI; nếu không có DB thì chỉ chạy được các test unit không phụ thuộc database.
- Dùng database/schema riêng cho test, migration và seed fixture xác định. Không chạy test phá hủy trên dữ liệu thật.
- Tạo tối thiểu: một USER, một ADMIN, hai USER độc lập; sản phẩm active có biến thể tồn kho dương, biến thể tồn kho thấp/hết hàng, và sản phẩm inactive.
- Dữ liệu test dùng email định danh theo lượt chạy; dọn dữ liệu tạo trong test sau mỗi suite hoặc reset database test trước suite. Test phải độc lập và chạy lặp lại được.
- Playwright chạy headless trong CI. Kiểm tra layout ở desktop và mobile; Chromium là trình duyệt bắt buộc, Firefox/WebKit là smoke mở rộng nếu pipeline hỗ trợ.

## 5. Ma trận test case

| ID       | Ưu tiên | Loại     | Kịch bản và kết quả mong đợi                                                                                                                                                             |
| -------- | ------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AUTH-01  | P0      | API/E2E  | Đăng ký với dữ liệu hợp lệ tạo USER mới; password không được trả về response.                                                                                                            |
| AUTH-02  | P1      | API      | Email sai định dạng, thiếu trường, password ngoài giới hạn hoặc email trùng bị từ chối đúng lỗi; không tạo bản ghi ngoài ý muốn.                                                         |
| AUTH-03  | P0      | API/E2E  | Đăng nhập đúng tạo phiên hợp lệ; sai mật khẩu trả lỗi xác thực và không cấp phiên.                                                                                                       |
| AUTH-04  | P0      | API/E2E  | USER truy cập route admin bị từ chối; ADMIN truy cập được. Không tin role do client gửi lên.                                                                                             |
| PROD-01  | P1      | E2E      | Duyệt danh sách/chi tiết sản phẩm, lọc theo danh mục và chọn biến thể; sản phẩm inactive không thể thêm vào giỏ/thuê.                                                                    |
| CART-01  | P0      | API/E2E  | Thêm biến thể hợp lệ vào giỏ; thêm lại cùng biến thể cập nhật số lượng, tổng số lượng và tổng tiền hiển thị chính xác.                                                                   |
| CART-02  | P0      | API      | Không thể thêm/cập nhật số lượng bằng 0, âm, không hợp lệ hoặc vượt tồn kho; lỗi không làm thay đổi giỏ.                                                                                 |
| CART-03  | P1      | API      | USER không thể đọc, sửa hoặc xóa item thuộc giỏ của USER khác; item không tồn tại trả lỗi phù hợp.                                                                                       |
| ORDER-01 | P0      | API/E2E  | Checkout giỏ hợp lệ tạo order, order items và payment COD; tổng tiền dùng giá bán; tồn kho giảm đúng; giỏ được làm trống.                                                                |
| ORDER-02 | P0      | API      | Checkout với giỏ trống, sản phẩm inactive hoặc tồn kho không đủ thất bại; transaction rollback, không tạo đơn/payment và không trừ tồn kho một phần.                                     |
| ORDER-03 | P0      | API/E2E  | USER chỉ xem được đơn của mình; ADMIN xem được đơn theo quyền quản trị.                                                                                                                  |
| ORDER-04 | P0      | API/E2E  | Hủy đơn PENDING chưa thanh toán chuyển sang CANCELLED và hoàn kho đúng một lần; đơn đã xử lý/đã thanh toán không được hủy.                                                               |
| RENT-01  | P0      | Unit/API | Ngày thuê đúng định dạng và khoảng hợp lệ được chấp nhận; ngày sai định dạng, ngày không tồn tại hoặc endDate trước startDate bị từ chối.                                                |
| RENT-02  | P0      | API/E2E  | Tạo thuê hợp lệ tính số ngày bao gồm cả ngày bắt đầu/kết thúc, nhân giá thuê với số lượng và số ngày; tạo payment COD và trừ kho đúng.                                                   |
| RENT-03  | P0      | API      | Danh sách item rỗng, variant không tồn tại/inactive, số lượng không hợp lệ hoặc vượt tồn kho bị từ chối; không có dữ liệu/stock cập nhật dở dang.                                        |
| RENT-04  | P0      | API/E2E  | Hủy thuê PENDING chưa thanh toán hoàn kho một lần; thuê đã xác nhận/đang thuê hoặc đã thanh toán không được hủy theo quy tắc hiện tại.                                                   |
| RENT-05  | P0      | API/E2E  | Chỉ cho phép `PENDING → CONFIRMED → RENTING → RETURNED`; chuyển trạng thái sai hoặc lặp lại bị từ chối. Khi RETURNED, payment COD pending chuyển PAID và tồn kho được hoàn đúng một lần. |
| ADMIN-01 | P1      | API/E2E  | ADMIN tạo/sửa/ẩn sản phẩm, biến thể và danh mục; dữ liệu bắt buộc/unique được kiểm tra; USER không thực hiện được thao tác quản trị.                                                     |
| ADMIN-02 | P1      | API/E2E  | ADMIN cập nhật trạng thái đơn theo luồng được hỗ trợ; tổng hợp báo cáo khớp dữ liệu order/rental đã seed.                                                                                |
| UI-01    | P1      | E2E      | Đăng ký/đăng nhập, giỏ, checkout và thuê có trạng thái loading/lỗi/thành công rõ ràng; reload không làm sai dữ liệu đã lưu.                                                              |
| UI-02    | P2      | E2E      | Các trang chính dùng được ở desktop/mobile, không tràn nội dung; thao tác form chính dùng được bằng bàn phím và có label phù hợp.                                                        |

## 6. Quy trình thực thi

1. Cài dependency bằng lockfile và kiểm tra `npm run lint`, `npm run build`.
2. Tạo database test riêng, chạy migration/seed, xác nhận ứng dụng kết nối đúng database test.
3. Chạy unit tests; sau đó integration tests cho auth, giỏ, checkout, đơn hàng và thuê.
4. Chạy E2E P0 theo thứ tự: đăng nhập → mua hàng → hủy đơn; đăng nhập → tạo thuê → xác nhận/nhận/trả; kiểm tra quyền USER/ADMIN.
5. Chạy các case P1/P2, kiểm tra responsive và tổng hợp kết quả theo ID.
6. Lặp lại các case transaction/hủy để xác nhận không trừ hoặc hoàn kho hai lần; chạy lại suite để phát hiện phụ thuộc thứ tự.

## 7. Điều kiện vào/ra

**Điều kiện vào**

- Build và lint thành công.
- MariaDB test riêng sẵn sàng, migration và fixture hoàn tất.
- Có tài khoản USER/ADMIN test và dữ liệu sản phẩm/biến thể đủ cho case.
- Playwright Chromium được cài nếu chạy E2E.

**Điều kiện hoàn tất**

- Tất cả test P0 pass; không còn lỗi nghiêm trọng/cao chưa có chấp thuận rủi ro.
- P1 pass hoặc có ghi nhận rõ lỗi còn lại, người chấp nhận và ảnh hưởng.
- Không có lỗi dữ liệu: tồn kho âm, order/rental thiếu item/payment, hoặc quyền truy cập chéo.
- Báo cáo ghi tổng pass/fail/blocked, môi trường, commit được kiểm thử và liên kết lỗi.

## 8. Trạng thái và giả định hiện tại

- Chưa phát hiện test suite trong `src`; `package.json` hiện chỉ có `dev`, `build`, `start`, `lint`. Cần bổ sung test infrastructure trước khi thực thi các tầng unit/integration/E2E.
- Docker hiện không chạy; không thể giả định có database container. Cần chuẩn bị MariaDB test độc lập hoặc bật Docker trong môi trường chạy test.
- Các expected result về thuê phản ánh quy tắc đang có trong service; cần xác nhận với nghiệp vụ rằng thu tiền COD khi chuyển `RETURNED` và hoàn kho ở thời điểm đó là mong muốn.
- Không có yêu cầu SLA hiệu năng hoặc trình duyệt hỗ trợ cụ thể; chưa đặt ngưỡng performance để tránh tạo tiêu chí tùy ý.


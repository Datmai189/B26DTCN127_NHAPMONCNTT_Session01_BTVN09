# B26DTCN127_NHAPMONCNTT_Session01_BTVN09

## Phân tích tính năng chụp ảnh bằng chứng giao hàng (POD) — FastShip

## 1. Phân tích IPO

Em mô tả luồng chụp và lưu ảnh POD theo mô hình IPO như sau:

| Thành phần | Nội dung |
|---|---|
| **Input** | Mã đơn hàng; trạng thái giao thành công; 2 ảnh do tài xế chụp; thông tin ảnh (định dạng, dung lượng, thời gian chụp); tài khoản/thiết bị tài xế. |
| **Process** | App yêu cầu chụp đủ 2 ảnh → kiểm tra ảnh hợp lệ và dung lượng → nén/giảm kích thước nếu cần → gửi ảnh lên server qua kết nối bảo mật → server kiểm tra lại, lưu file và liên kết file với mã đơn hàng. |
| **Output** | Đơn hàng có 2 ảnh POD xem được trên hệ thống; đường dẫn/tệp ảnh, thời gian tải lên và trạng thái thành công hoặc lỗi. |

Luồng xử lý chi tiết:

```text
Giao hàng thành công
        ↓
Tài xế chụp đủ 2 ảnh
        ↓
App kiểm tra từng ảnh có ≤ 5 MB?
   ├─ Có  → tải ảnh lên server
   └─ Không → nén/giảm độ phân giải → kiểm tra lại
                           ├─ Đạt → tải lên server
                           └─ Không đạt → báo tài xế chụp lại
        ↓
Server kiểm tra lại → lưu ảnh → gắn với đơn hàng → trả kết quả
```

## 2. Xử lý ảnh vượt quá 5 MB

Không nên đợi server báo đầy mới xử lý. App cần kiểm tra dung lượng **trước khi tải lên** để giảm thời gian chờ và băng thông của tài xế.

1. Khi ảnh lớn hơn 5 MB, app tự giảm độ phân giải và nén ảnh JPEG/HEIC. Mục tiêu nên là khoảng **4,5 MB hoặc thấp hơn**, thay vì nén vừa đúng 5 MB, để chừa khoảng an toàn.
2. Sau khi nén, app kiểm tra lại dung lượng và vẫn giữ ảnh đủ rõ để nhìn được bằng chứng giao hàng.
3. Nếu ảnh vẫn vượt 5 MB, app báo rõ lý do và yêu cầu tài xế chụp lại ở chế độ độ phân giải thấp hơn. Không được tự bỏ qua ảnh, vì mỗi đơn bắt buộc có đủ 2 ảnh POD.
4. Server vẫn phải kiểm tra lần hai: từ chối tệp lớn hơn 5 MB, chỉ chấp nhận định dạng ảnh cho phép, quét/kiểm tra tệp và gắn ảnh đúng mã đơn. Việc này ngăn trường hợp app cũ hoặc người dùng gửi tệp không hợp lệ trực tiếp lên server.

Theo em, nén ở app giúp trải nghiệm tốt hơn; kiểm tra lại ở server bảo vệ hệ thống. Hai lớp này cần đi cùng nhau.

## 3. Tính dung lượng lưu trữ trong 30 ngày

### Dữ liệu đầu vào

| Thông số | Giá trị |
|---|---:|
| Số đơn hàng mỗi ngày | 50.000 đơn |
| Số ảnh mỗi đơn | 2 ảnh |
| Dung lượng trung bình mỗi ảnh | 3 MB |
| Thời gian tính | 30 ngày |

### Tính theo ngày

```text
Số ảnh/ngày = 50.000 đơn × 2 ảnh = 100.000 ảnh
Dung lượng/ngày = 100.000 ảnh × 3 MB = 300.000 MB
```

### Tính trong một tháng

```text
Dung lượng/30 ngày = 300.000 MB × 30 = 9.000.000 MB
```

Quy đổi theo quy ước lưu trữ phổ biến của nhà cung cấp cloud:

```text
1 TB = 1.000.000 MB
9.000.000 MB = 9 TB
```

**Kết luận:** FastShip cần lưu khoảng **9 TB ảnh POD trong 1 tháng (30 ngày)**, chưa bao gồm bản sao lưu, ảnh thumbnail, metadata và dung lượng dự phòng.

Nếu dùng cách quy đổi nhị phân, `9.000.000 MB ÷ 1.048.576 ≈ 8,58 TiB`. Tuy nhiên, khi lập kế hoạch chi phí lưu trữ cloud, cách ghi **9 TB** là phù hợp hơn vì nhà cung cấp thường dùng hệ thập phân.

## 4. Nhận xét triển khai

- Con số 9 TB là theo **trung bình 3 MB/ảnh**. Trong tình huống xấu nhất, nếu mọi ảnh đều sát giới hạn 5 MB, dung lượng tháng có thể là: `50.000 × 2 × 5 MB × 30 = 15.000.000 MB = 15 TB`.
- Vì vậy, nếu chưa có cơ chế nén ổn định, em sẽ dự trù tối thiểu **15 TB/tháng** cho ảnh gốc; nếu tính cả backup và vận hành, cần cộng thêm dung lượng dự phòng.
- Nên lưu ảnh ở kho lưu trữ đối tượng (object storage), còn cơ sở dữ liệu chỉ lưu mã đơn, đường dẫn ảnh, thời gian và trạng thái. Cách này giúp hệ thống dễ mở rộng hơn so với nhét ảnh trực tiếp vào database.

## 5. Kết luận

Tính năng POD nhận 2 ảnh cho mỗi đơn hàng, app chịu trách nhiệm kiểm tra/nén ảnh trước khi tải lên và server kiểm tra lại trước khi lưu. Với dữ liệu trung bình, dung lượng một tháng là **9 TB**. Giới hạn 5 MB/ảnh phải được xử lý ngay trên app và bắt buộc xác thực lại ở server để tránh đầy hệ thống và bảo đảm mỗi đơn vẫn có đủ bằng chứng giao hàng.

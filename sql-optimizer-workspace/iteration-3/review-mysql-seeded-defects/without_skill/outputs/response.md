Có, query này có mấy lỗi khá nặng. Với 30 triệu dòng, gần như chắc chắn nó đang **quét toàn bảng** mỗi lần tổng đài viên tìm khách. Mình xếp theo mức độ ảnh hưởng dưới đây.

> Mình chưa chạy trên DB của bạn. Phần đánh giá dựa trên DDL và cách optimizer của MySQL 8.0 xử lý. Cuối bài có các bước để bạn tự kiểm tra bằng `EXPLAIN`.

---

## 1. `o.phone = 0912345678`: so sánh cột chuỗi với số (nghiêm trọng nhất)

`phone` là `VARCHAR(20)`, còn `0912345678` không có dấu nháy nên MySQL hiểu đây là **số** 912345678. Khi so sánh chuỗi với số, MySQL đổi **từng giá trị `phone`** sang số thực (double) rồi mới so sánh. Hệ quả:

- **`idx_phone` không dùng được.** Tài liệu MySQL ghi rõ là khi so sánh cột chuỗi với một số, MySQL không dùng được index trên cột đó. Đây là index chọn lọc nhất của query này, mất nó thì gần như chắc chắn phải full scan.
- **Kết quả có thể sai.** Số 0 ở đầu bị mất. Các chuỗi như `'912345678'`, `'0912345678'`, `' 912345678'`, `'912345678abc'` đều được coi là bằng 912345678. Mỗi dòng không phải số cũng sinh ra một warning (`Truncated incorrect DOUBLE value`).

**Sửa:** truyền tham số dạng chuỗi: `o.phone = '0912345678'`. Nếu code ứng dụng đang bind tham số kiểu int/long thì phải sửa ở đó, chỉ thêm nháy trong câu SQL là chưa đủ.

## 2. `DATE(o.created_at) = '2026-11-01'`: bọc hàm quanh cột có index

Khi bọc `created_at` trong `DATE()`, MySQL không range-scan được trên `idx_created` nữa vì phải tính hàm cho từng dòng.

**Sửa:** viết lại thành khoảng nửa mở:

```sql
o.created_at >= '2026-11-01' AND o.created_at < '2026-11-02'
```

(MySQL 8.0.13 trở lên có hỗ trợ functional index `((DATE(created_at)))`, nhưng viết theo khoảng như trên vẫn tốt hơn: không cần thêm index, và kết hợp được với index ghép ở mục 6.)

## 3. `ORDER BY RAND()`

Với `ORDER BY RAND()`, MySQL phải sinh số ngẫu nhiên cho **mọi** dòng khớp điều kiện, đưa vào bảng tạm hoặc filesort rồi mới lấy 10 dòng. Không index nào giúp được bước này.

- Nếu số dòng khớp ít (đã lọc tốt theo phone và ngày) thì chi phí này nhỏ. Nhưng hiện tại, vì lỗi 1 và 2, MySQL sắp xếp trên tập dòng lấy được từ full scan.
- Về nghiệp vụ, màn hình CSKH hiển thị ngẫu nhiên thì khá lạ: mỗi lần tải lại ra 10 đơn khác nhau, khách hỏi đơn mới nhất thì tổng đài viên không chắc là thấy được. Gần như chắc chắn bạn cần `ORDER BY created_at DESC`. Thứ tự này còn đọc thẳng được từ index ở mục 6 và dừng sau 10 dòng, không cần filesort.

Nếu thật sự cần lấy ngẫu nhiên thì vẫn giữ được, vì sau khi sửa 1 và 2 tập dòng khớp chỉ còn vài chục dòng. Nhưng hãy xác nhận lại yêu cầu này.

## 4. `customer_email LIKE '%@gmail.com' OR coupon_code = 'BF2026'`

- `LIKE` có `%` ở đầu thì không dùng được index B-tree trên `customer_email`. Cột này cũng chưa có index.
- `OR` giữa hai cột khác nhau khiến MySQL khó dùng index. Nếu MySQL cố dùng `idx_coupon` theo kiểu index_merge (union) thì lại cần nhánh email cũng có index, mà nhánh này không có.

Khi đã lọc theo `phone` và ngày bằng index thì điều kiện này chỉ còn là bộ lọc trên vài dòng, **không đáng lo**. Không cần thêm index cho nó.

Nếu sau này cần lọc theo domain email một mình, có thể thêm cột generated `email_domain` (`SUBSTRING_INDEX(customer_email, '@', -1)`) rồi đánh index trên cột đó.

## 5. `SELECT *` kéo theo `note TEXT` và `payload JSON`

- Phải đọc cả các cột lớn. `TEXT` và `JSON` thường nằm ở trang off-page, nên tốn thêm I/O cho mỗi dòng.
- Kết hợp với `ORDER BY RAND()`, các cột lớn này còn phải đi qua bảng tạm hoặc sort buffer, làm tốn bộ nhớ và dễ tràn ra đĩa.
- Màn hình danh sách thường không cần `note` và `payload`.

**Sửa:** chỉ chọn các cột cần hiển thị. Khi tổng đài viên mở chi tiết một đơn thì mới tải `note` và `payload` theo `id`.

## 6. Index: nên có index ghép `(phone, created_at)`

Hiện có `idx_phone` và `idx_created` riêng lẻ. Index tốt nhất cho query này:

```sql
ALTER TABLE orders
  ADD INDEX idx_phone_created (phone, created_at),
  ALGORITHM=INPLACE, LOCK=NONE;
```

- MySQL seek đúng số điện thoại, range-scan theo ngày, đọc sẵn theo thứ tự `created_at` (đọc ngược để ra DESC) và dừng sau 10 dòng.
- Sau khi tạo xong và kiểm tra ổn, `idx_phone` thành thừa vì nó là prefix của index mới. Có thể bỏ nó để giảm chi phí ghi, nhưng trước đó hãy kiểm tra xem còn query nào khác đang chỉ định dùng nó (FORCE/USE INDEX) không:
  ```sql
  ALTER TABLE orders DROP INDEX idx_phone, ALGORITHM=INPLACE, LOCK=NONE;
  ```
  Một cách an toàn hơn là `ALTER TABLE orders ALTER INDEX idx_phone INVISIBLE;` trước, theo dõi một thời gian rồi mới drop.
- Lưu ý khi vận hành trên bảng 30 triệu dòng:
  - Online DDL vẫn tốn thời gian, cần dung lượng đĩa cho file tạm và `innodb_online_alter_log_max_size`.
  - Nếu có replica thì sẽ có độ trễ replication.
  - Nên chạy giờ thấp điểm, hoặc dùng `gh-ost`/`pt-online-schema-change`.

## Query đề xuất

```sql
SELECT o.id, o.phone, o.customer_email, o.coupon_code, o.total, o.created_at
FROM orders o
WHERE o.phone = '0912345678'
  AND o.created_at >= '2026-11-01'
  AND o.created_at <  '2026-11-02'
  AND (o.customer_email LIKE '%@gmail.com' OR o.coupon_code = 'BF2026')
ORDER BY o.created_at DESC
LIMIT 10;
```

Trong code, nên truyền tham số qua prepared statement, phone dạng string, ngày dạng `[start, start + 1 day)`.

## Cách kiểm tra

1. Chạy `EXPLAIN FORMAT=TREE` (hoặc `EXPLAIN ANALYZE`, lưu ý lệnh này **chạy thật** query) cho query cũ. Nhiều khả năng bạn sẽ thấy `Table scan on o` (hoặc `type: ALL`), số dòng ước tính khoảng 30 triệu, kèm `Using filesort` / `Using temporary`. Sau khi chạy, `SHOW WARNINGS` sẽ hiện các warning về chuyển kiểu.
2. Chạy lại với query mới, sau khi đã tạo `idx_phone_created`. Kỳ vọng thấy `Index range scan on o using idx_phone_created` (hoặc `type: range`, `key: idx_phone_created`), không còn filesort, và `rows` chỉ vài dòng đến vài chục dòng.
3. Nếu chưa tạo index mới thì chỉ sửa 1 và 2 cũng đã giúp MySQL dùng được `idx_phone`. Riêng thay đổi này đã nhanh hơn rất nhiều so với full scan.

## Những điểm mình đã giả định, bạn xác nhận giúp

- **Lọc cố định một ngày có đúng nghiệp vụ không?** Màn hình "tìm khách" thường cần các đơn gần nhất của số điện thoại đó, không giới hạn một ngày. Nếu bỏ điều kiện ngày thì index `(phone, created_at)` cùng `ORDER BY created_at DESC LIMIT 10` vẫn rất nhanh.
- **Định dạng `phone` trong DB có thống nhất không?** Ví dụ `0912345678` so với `+84912345678` hay `84912345678`. Nếu không thống nhất thì so sánh bằng chuỗi sẽ bỏ sót đơn. Khi đó nên chuẩn hóa dữ liệu, hoặc thêm một cột phone đã chuẩn hóa, thay vì dựa vào việc ép kiểu ngầm sang số.
- **`ORDER BY RAND()` có phải yêu cầu thật không,** hay chỉ là để tạm?

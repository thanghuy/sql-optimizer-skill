Mình chưa có `SHOW CREATE TABLE`, `EXPLAIN` hay số liệu phân bố dữ liệu, nên phần dưới là **đề xuất dựa trên giả định hợp lý**. Bạn chạy thử trên staging/replica rồi kiểm tra bằng `EXPLAIN ANALYZE` trước khi đưa lên prod. Cuối bài có danh sách những thứ cần gửi thêm để mình chốt chính xác hơn.

## Vì sao query này chậm (khả năng cao)

1. **Thiếu index tổng hợp phù hợp**: MySQL có thể đang dùng index chỉ có `shop_id` (hoặc quét cả bảng), đọc toàn bộ đơn của shop rồi mới lọc `status`, `created_at`, sau đó **filesort**.
2. **`OFFSET 2000`**: MySQL phải đọc *và bỏ đi* 2000 dòng đầu, tức là xử lý 2050 dòng (kèm 2050 lần lookup về clustered index để lấy `order_no`, `total`) chỉ để trả về 50 dòng.
3. **`status IN (...)` + `ORDER BY created_at`**: nếu index là `(shop_id, status, created_at)` thì với 2 giá trị status, MySQL **không** dùng thứ tự của index để sort được (nó không merge 2 nhánh) → vẫn phải đọc hết các dòng khớp rồi filesort.

## Đề xuất index

### Phương án chính (khuyến nghị thử trước)

```sql
ALTER TABLE orders
  ADD INDEX idx_orders_shop_created_status (shop_id, created_at, status),
  ALGORITHM=INPLACE, LOCK=NONE;
```

Lý do đặt `created_at` trước `status`:
- `shop_id` (bằng) + `created_at` (range) giúp MySQL **đi ngược index theo đúng thứ tự `created_at DESC`**, không cần filesort, và **dừng sớm** khi đã đủ 2050 dòng khớp.
- `status` nằm trong index nên được lọc ngay trên index (Index Condition Pushdown), chỉ dòng nào khớp mới phải lookup về bảng.
- InnoDB tự gắn `id` (PK) vào cuối secondary index nên không cần thêm `id`.

Phương án này tốt khi `paid`/`shipped` chiếm phần lớn đơn của shop (thường đúng với bảng orders).

### Phương án thay thế: khi `paid`/`shipped` chỉ là thiểu số

Nếu phần lớn đơn của shop là `pending`/`cancelled`... thì đi theo `created_at` sẽ phải bỏ qua nhiều dòng. Khi đó dùng:

```sql
ALTER TABLE orders
  ADD INDEX idx_orders_shop_status_created (shop_id, status, created_at),
  ALGORITHM=INPLACE, LOCK=NONE;
```

Index này lọc chính xác hơn nhưng sẽ filesort tập kết quả. Muốn biết nên chọn cái nào, chạy query này:

```sql
SELECT status, COUNT(*) FROM orders
WHERE shop_id = 1207 AND created_at >= '2026-10-01'
GROUP BY status;
```

Nếu tỷ lệ chưa rõ, có thể tạo cả hai trên staging rồi so `EXPLAIN ANALYZE` (dùng `FORCE INDEX` để so từng cái), sau đó **chỉ giữ một** để đỡ tốn chi phí ghi.

## Sửa query (hiệu quả hơn hẳn so với chỉ thêm index)

Index chỉ giải quyết một phần. Thủ phạm còn lại là `OFFSET 2000`, và nó sẽ chậm dần khi người dùng lật sang các trang sau.

### Cách 1: Deferred join (không phải đổi API)

Lấy 50 `id` bằng index trước, sau đó mới join để lấy các cột còn lại. Nhờ vậy 2000 dòng bị bỏ qua không phải lookup về bảng nữa:

```sql
SELECT o.id, o.order_no, o.total, o.status, o.created_at
FROM orders o
JOIN (
    SELECT id
    FROM orders
    WHERE shop_id = 1207
      AND status IN ('paid','shipped')
      AND created_at >= '2026-10-01'
    ORDER BY created_at DESC, id DESC
    LIMIT 50 OFFSET 2000
) t ON t.id = o.id
ORDER BY o.created_at DESC, o.id DESC;
```

Subquery bên trong chỉ cần `(shop_id, created_at, status, id)`, và index ở phương án chính đã **covering** cho nó ("Using index").

### Cách 2: Keyset pagination (tốt nhất về lâu dài)

Bỏ OFFSET và dùng con trỏ là dòng cuối của trang trước:

```sql
SELECT id, order_no, total, status, created_at
FROM orders
WHERE shop_id = 1207
  AND status IN ('paid','shipped')
  AND created_at >= '2026-10-01'
  AND (created_at, id) < (:last_created_at, :last_id)   -- bỏ điều kiện này ở trang đầu
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

Với cách này trang nào cũng chạy nhanh như trang 1. Đổi lại, phía frontend/API phải chuyển sang kiểu "trang sau / trước" (cursor) thay vì nhảy thẳng tới trang N.
Lưu ý: ở một số phiên bản MySQL, row constructor `(a,b) < (x,y)` không tận dụng index tốt. Nếu `EXPLAIN` cho thấy không dùng range thì viết lại thành:
`AND (created_at < :last_created_at OR (created_at = :last_created_at AND id < :last_id))`.

Thêm `id` vào `ORDER BY` để thứ tự luôn ổn định khi có nhiều đơn trùng `created_at`. Không có nó, phân trang có thể bị lặp hoặc sót dòng.

## Lưu ý khi chạy trên prod

- `ALGORITHM=INPLACE, LOCK=NONE` (MySQL 5.6+/8.0) không khóa ghi, nhưng trên bảng lớn vẫn tốn IO/CPU và **gây lag replica** (replica áp DDL tuần tự). Nếu bảng lớn (hàng chục triệu dòng trở lên) thì nên dùng `gh-ost` hoặc `pt-online-schema-change`, và chạy giờ thấp điểm.
- Kiểm tra index đang có (`SHOW INDEX FROM orders`). Nếu đã có `(shop_id)` riêng lẻ thì sau khi index mới ổn định có thể xóa nó, vì index mới đã bao trọn prefix `shop_id`.
- Kiểm tra `created_at` có đúng kiểu `DATETIME`/`TIMESTAMP` không. Nếu là `VARCHAR` thì so sánh và sort sẽ sai hoặc chậm.
- Sau khi thêm index, chạy `EXPLAIN ANALYZE` lại để xác nhận: `key` là index mới, không còn `Using filesort` (với phương án chính), `rows` chỉ khoảng vài nghìn.

## Tóm tắt nhanh để báo sếp

1. **Làm ngay**: thêm index `(shop_id, created_at, status)`, dự kiến giảm từ vài giây xuống còn vài chục ms hoặc thấp hơn (cần đo lại để xác nhận).
2. **Làm thêm (ít rủi ro)**: chuyển query sang deferred join.
3. **Về lâu dài**: chuyển sang keyset pagination để trang sâu không chậm dần.

## Để mình chốt chính xác, bạn gửi giúp

- Kết quả `SHOW CREATE TABLE orders;` (các index hiện có, kiểu cột)
- Kết quả `EXPLAIN ANALYZE <query>` (hoặc `EXPLAIN FORMAT=JSON` nếu MySQL < 8.0.18)
- Phiên bản MySQL (`SELECT VERSION();`)
- Số dòng của bảng, và kết quả query `GROUP BY status` ở trên cho shop 1207
- Shop này có phải shop lớn bất thường không? Các shop khác query cũng chậm hay chỉ shop này?

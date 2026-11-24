Chưa có `SHOW CREATE TABLE`, `EXPLAIN` hay số dòng nên mình chưa khẳng định được 100%. Nhưng với query dạng này thì gần như chắc chắn có 2 vấn đề: **thiếu index phù hợp** và **OFFSET 2000**. Dưới đây là cách làm luôn, cuối bài có mấy thông tin cần bạn gửi để chốt.

## 1. Index nên tạo (làm ngay)

```sql
ALTER TABLE orders
  ADD INDEX idx_shop_created_status (shop_id, created_at, status),
  ALGORITHM=INPLACE, LOCK=NONE;
```

**Vì sao thứ tự là `(shop_id, created_at, status)` mà không phải `(shop_id, status, created_at)`?**

- `status IN ('paid','shipped')` có 2 giá trị. Nếu `status` đứng trước `created_at` thì MySQL phải quét 2 khoảng riêng, kết quả không còn sắp xếp theo `created_at` nữa, nên nó **phải gom hết các dòng khớp rồi filesort**, không dừng sớm ở dòng thứ 2050 được.
- Với `(shop_id, created_at, status)`: MySQL đi đúng vào `shop_id = 1207`, đọc ngược `created_at` từ mới về cũ (khớp `ORDER BY ... DESC`, **không filesort**), lọc `status` ngay trên index (Index Condition Pushdown), và **dừng khi đủ 2050 dòng**.

**Muốn nhanh hơn nữa thì dùng covering index** (query chỉ cần đọc index, không phải quay lại bảng). InnoDB tự kèm `id` (PK) trong secondary index rồi:

```sql
ALTER TABLE orders
  ADD INDEX idx_shop_created_cover (shop_id, created_at, status, order_no, total),
  ALGORITHM=INPLACE, LOCK=NONE;
```

Index này to hơn và làm ghi chậm hơn một chút. Mình nghĩ nên thử bản 3 cột trước, chưa đủ nhanh thì mới chuyển sang bản covering.

**Có một ngoại lệ:** nếu với shop này, đơn `paid`/`shipped` chỉ chiếm rất ít (ví dụ dưới 5% số đơn từ 01/10) thì `(shop_id, status, created_at)` lại tốt hơn, vì nó chỉ đọc đúng những dòng cần rồi filesort một tập nhỏ. Kiểm tra nhanh bằng:

```sql
SELECT status, COUNT(*) FROM orders
WHERE shop_id = 1207 AND created_at >= '2026-10-01'
GROUP BY status;
```

## 2. OFFSET 2000 là một phần nguyên nhân

`LIMIT 50 OFFSET 2000` nghĩa là MySQL phải đọc **2050 dòng** (kèm tra cứu lại bảng cho từng dòng) rồi bỏ đi 2000 dòng. Trang càng sâu thì càng chậm. Có 2 cách sửa.

**Cách A: deferred join** (không phải đổi API). Lấy `id` trên index trước, rồi mới join để lấy cột:

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

Subquery chỉ chạm index `(shop_id, created_at, status)` (có sẵn `id`), nên chỉ phải tra bảng đúng 50 lần thay vì 2050 lần.

**Cách B: keyset / seek pagination** (tốt nhất, nhưng phải sửa API). Client gửi lên `created_at` và `id` của dòng cuối trang trước:

```sql
SELECT id, order_no, total, status, created_at
FROM orders
WHERE shop_id = 1207
  AND status IN ('paid','shipped')
  AND created_at >= '2026-10-01'
  AND (created_at < :last_created_at
       OR (created_at = :last_created_at AND id < :last_id))
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

Trang nào cũng chỉ đọc khoảng 50 dòng. Đổi lại thì không nhảy thẳng đến "trang 41" được, chỉ đi next/prev.

Lưu ý: nên thêm `id` vào `ORDER BY`. Nếu nhiều đơn trùng `created_at` thì thứ tự hiện tại không ổn định, phân trang có thể lặp hoặc sót dòng.

## 3. Lưu ý khi chạy trên prod

- `ALGORITHM=INPLACE, LOCK=NONE` (MySQL 5.6+ / 8.0) không khóa ghi, nhưng bảng lớn vẫn tốn IO và gây **replication lag**. Nên chạy giờ thấp điểm. Bảng rất lớn (hàng chục/trăm triệu dòng) thì cân nhắc `gh-ost` hoặc `pt-online-schema-change`.
- Kiểm tra trước và sau bằng:
  ```sql
  EXPLAIN SELECT ...;            -- mong đợi: key = idx_shop_created_status, không có "Using filesort"
  EXPLAIN ANALYZE SELECT ...;    -- MySQL 8.0.18+, xem số dòng thực tế đã đọc
  ```
- Nếu đã có index `(shop_id)` riêng thì sau khi tạo index mới, index cũ thành thừa (index mới bắt đầu bằng `shop_id`). Có thể drop sau khi theo dõi ổn định.
- Nếu `created_at` không phải `DATETIME`/`TIMESTAMP` (ví dụ lưu dạng VARCHAR hay INT) thì báo mình, vì điều kiện so sánh sẽ khác.

## 4. Gửi mình mấy cái này để chốt phương án

1. `SHOW CREATE TABLE orders;` (để xem index hiện có)
2. `EXPLAIN` của query hiện tại
3. Phiên bản MySQL (`SELECT VERSION();`)
4. Bảng có khoảng bao nhiêu dòng, shop 1207 có bao nhiêu đơn từ 01/10
5. Kết quả query `GROUP BY status` ở mục 1
6. API này có bắt buộc nhảy trang theo số không, hay chỉ cần next/prev (infinite scroll)?

**Tóm lại nếu cần làm gấp:** tạo index `(shop_id, created_at, status)`, thêm `id DESC` vào `ORDER BY`, và đổi query sang dạng deferred join. Như vậy thường đủ đưa query từ vài giây xuống dưới chục mili-giây. Sau đó mới tính chuyển sang keyset pagination.

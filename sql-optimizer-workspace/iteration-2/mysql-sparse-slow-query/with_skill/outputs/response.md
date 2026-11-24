Gấp thì mình đưa luôn index nên tạo ngay bên dưới. Nhưng bạn chưa gửi version, schema hay số liệu nên phần này dựa trên giả định. Mình giả định **MySQL 8.0.x (InnoDB)**, chỗ nào phụ thuộc version thì có ghi chú. Bạn gửi thêm các thông tin ở phần cuối để mình xác nhận lại, vì index đúng cho 10 nghìn dòng có thể sai cho 100 triệu dòng.

---

## Diagnosis (chẩn đoán sơ bộ)

Câu query có 3 điểm tốn kém:

1. **`status IN ('paid','shipped')` là IN nhiều giá trị.** Nếu index hiện tại là `(shop_id, status, created_at)` (kiểu index hay gặp nhất), MySQL seek được 2 dải: (1207,'paid',…) và (1207,'shipped',…). Mỗi dải tự sắp theo `created_at`, nhưng gộp lại thì **không còn đúng thứ tự `created_at DESC`**. MySQL phải đọc **toàn bộ** dòng khớp từ 01/10 tới nay rồi mới sort (`Using filesort`), sau đó mới lấy 50 dòng.
   Nếu index chỉ có `shop_id` hoặc chỉ có `created_at` thì còn tệ hơn: phải lookup về bảng rất nhiều dòng rồi lọc.
2. **`OFFSET 2000`.** MySQL phải tạo ra 2050 dòng theo đúng thứ tự, bỏ 2000 dòng đầu và chỉ trả 50. Với index không covering, đó là 2050 lần lookup về clustered index (PK). Trang càng sâu càng chậm, chậm tuyến tính theo OFFSET.
3. **Không covering.** Query cần `order_no` và `total`, hai cột này hầu như chắc chắn không có trong index. Vì vậy mỗi dòng đọc qua (kể cả 2000 dòng bị bỏ) đều phải nhảy về bảng.

Nghi phạm số 1 cho mức 4–5s là **filesort trên toàn bộ đơn của shop 1207 từ 01/10, cộng với lookup bảng**. Cần `EXPLAIN` để xác nhận (xem mục Verify).

Tài liệu MySQL 8.0 nói index chỉ tránh được sort khi các cột ORDER BY khớp với key part theo thứ tự và "gaps filled by constants" (các key part phía trước phải là hằng số), xem [ORDER BY Optimization 8.0](https://dev.mysql.com/doc/refman/8.0/en/order-by-optimization.html). `status IN (2 giá trị)` không phải hằng số, nên `status` không được đứng trước `created_at`.

---

## Recommendation

### 1. Index chính (làm ngay, không cần sửa code)

```sql
ALTER TABLE orders
  ADD INDEX idx_orders_shop_created_id_status (shop_id, created_at, id, status),
  ALGORITHM=INPLACE, LOCK=NONE;
```

**ESR mapping:**
- **E** = `shop_id` (`= 1207`): seek thẳng vào phần dữ liệu của shop.
- **S** = `created_at`, sau đó `id`: dòng đọc ra đã đúng thứ tự `created_at DESC` (MySQL đọc index ngược, *Backward index scan*). Vì thế **không còn filesort**, và LIMIT dừng sớm sau khi đủ 2050 dòng khớp. `id` đặt ngay sau `created_at` để làm tie-breaker cho keyset pagination ở bước 2. Thêm `id` vào key không làm index to thêm: InnoDB vốn đã gắn PK vào cuối mọi secondary index, chỉ là giờ nó nằm đúng vị trí.
- **R** = `created_at >= '2026-10-01'` chặn điểm dừng của dải đọc (cùng cột với S). `status IN (...)` đặt **cuối**, chỉ làm filter trong index (Index Condition Pushdown). Dòng có status khác bị loại ngay trong index, không phải lookup về bảng.

**Vì sao không chọn `(shop_id, status, created_at)` (thứ tự E-R-S)?** Index đó seek chặt hơn nhưng luôn phải sort hết các dòng khớp. Nó chỉ thắng khi số đơn paid/shipped của shop 1207 từ 01/10 rất nhỏ (vài nghìn dòng). Khi đó sort vài nghìn dòng rẻ hơn đọc ngược và bỏ qua các status khác. Mình cần 3 con số ở mục câu hỏi (4b) để chốt. Với OFFSET 2000 thì ít nhất cũng phải có trên 2050 dòng khớp, nên ESR gần như chắc chắn có lợi hơn.

**Hiệu quả kỳ vọng:** từ "đọc + sort toàn bộ đơn của shop từ 01/10" xuống "đọc khoảng 2050 entry index đã đúng thứ tự (cộng các dòng status khác bị lọc trong index) + 2050 lookup". Thường giảm từ vài giây xuống còn chục đến vài chục ms, tuỳ data.

**Chi phí:**
- Thêm 1 secondary index: mỗi INSERT, và mỗi UPDATE đổi `status`/`created_at`, tốn thêm 1 lần ghi index. Với bảng orders, `status` đổi khá thường xuyên (paid → shipped → done…), nên cần số write/s để đánh giá.
- Dung lượng ước lượng khoảng (4–8 + 5 + 8 + ~10 byte/dòng) × số dòng, tức khoảng 30–40 byte/dòng trước overhead. Bạn gửi `SHOW INDEX` để mình xem index nào trở thành thừa (ví dụ `(shop_id)` hoặc `(shop_id, created_at)` là prefix của index mới, có thể bỏ để bù chi phí ghi).

**Lưu ý khi build trên prod:**
- `ALGORITHM=INPLACE, LOCK=NONE` không khoá ghi, nhưng vẫn tốn IO/CPU và có thể gây **replication lag** trên replica. Nên chạy ngoài giờ cao điểm.
- Cần chỗ trống trên đĩa xấp xỉ kích thước index mới (sort file tạm + `innodb_online_alter_log_max_size` cho DML phát sinh trong lúc build).
- Bảng rất lớn (hàng trăm triệu dòng) hoặc dùng Aurora/RDS có replica nhạy lag: cân nhắc `gh-ost` hoặc `pt-online-schema-change`.
- Nên thử trên staging có dữ liệu giống prod trước.

**Rollback:**
```sql
-- thử "tắt" index mà không xoá (MySQL 8.0+):
ALTER TABLE orders ALTER INDEX idx_orders_shop_created_id_status INVISIBLE;
-- bỏ hẳn:
ALTER TABLE orders DROP INDEX idx_orders_shop_created_id_status;
```

*(Tuỳ chọn, MySQL 8.0+: khai báo `created_at DESC, id DESC` để đọc xuôi thay vì đọc ngược. Lợi ích nhỏ, không bắt buộc. Trên 5.7 từ khoá DESC trong index bị bỏ qua, đọc ngược vẫn chạy được.)*

### 2. Bỏ OFFSET, chuyển sang keyset pagination (sửa code, lợi lớn nhất về lâu dài)

OFFSET 2000 nghĩa là luôn đọc rồi vứt 2000 dòng, và trang 100 sẽ chậm gấp đôi trang 50. Keyset pagination nhớ dòng cuối của trang trước (`created_at`, `id`) và đọc tiếp từ đó:

```sql
-- trang đầu
SELECT id, order_no, total, status, created_at
FROM orders
WHERE shop_id = 1207
  AND status IN ('paid','shipped')
  AND created_at >= '2026-10-01'
ORDER BY created_at DESC, id DESC
LIMIT 50;

-- trang sau: :last_created_at, :last_id = dòng cuối của trang trước
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

- `id` là tie-breaker bắt buộc. Nhiều đơn có thể trùng `created_at`, thiếu nó sẽ bị trùng hoặc mất dòng giữa các trang. Index ở bước 1 đã có `(shop_id, created_at, id, …)`, nên khớp đúng `ORDER BY created_at DESC, id DESC` và không filesort. Nếu dùng `(shop_id, created_at, status)` thì **không** khớp, vì `status` chen giữa `created_at` và `id`.
- Mình viết dạng `OR` mở rộng thay vì `(created_at, id) < (?, ?)` vì dạng mở rộng dễ đoán plan hơn trên MySQL. Kiểm tra lại bằng EXPLAIN.
- Mỗi trang chỉ đọc khoảng 50 dòng, dù là trang thứ mấy.
- Hạn chế: không "nhảy tới trang 41" được, chỉ có next/prev. Nếu UI bắt buộc nhảy trang thì xem bước 3.

### 3. Nếu vẫn phải giữ OFFSET: deferred join (chỉ lookup 50 dòng)

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
) page USING (id)
ORDER BY o.created_at DESC, o.id DESC;
```

Subquery chỉ cần các cột có trong index `(shop_id, created_at, id, status)`, nên covering (`Using index`). 2000 dòng bị bỏ không còn phải lookup về bảng, chỉ 50 dòng cuối mới lookup lấy `order_no` và `total`.

### 4. (Chỉ khi query cực hot) Covering index đầy đủ

`(shop_id, created_at, id, status, total, order_no)` loại bỏ hết lookup. Nhưng index to hơn đáng kể (nhất là nếu `order_no` là VARCHAR dài) và tăng chi phí ghi. Sau bước 1 và 2 thì mỗi request chỉ còn 50 lookup, nên thường **không đáng**. Chỉ làm khi số QPS cho thấy cần.

### Kiểm tra nhanh trước khi tạo index
- `shop_id` có phải kiểu số không? Nếu là VARCHAR thì phải viết `shop_id = '1207'`, nếu không MySQL ép kiểu cột và bỏ qua index.
- `created_at` có phải DATETIME/TIMESTAMP không (không phải VARCHAR)?

---

## Verify

Sau khi tạo index (ưu tiên trên staging), chạy và gửi mình kết quả:

```sql
-- plan ước lượng (8.0.16+)
EXPLAIN FORMAT=TREE
SELECT id, order_no, total, status, created_at
FROM orders
WHERE shop_id = 1207 AND status IN ('paid','shipped') AND created_at >= '2026-10-01'
ORDER BY created_at DESC
LIMIT 50 OFFSET 2000;

-- plan thực tế (8.0.18+). Lệnh này CHẠY THẬT câu query. Là SELECT nên không đổi dữ liệu, nhưng tốn tài nguyên như query thật.
EXPLAIN ANALYZE
SELECT ... (cùng query như trên);

-- số dòng thực sự đọc, so sánh trước/sau
FLUSH STATUS;
SELECT ... (cùng query);
SHOW SESSION STATUS LIKE 'Handler_read%';
```

Trên 5.7 không có `FORMAT=TREE` và `EXPLAIN ANALYZE`, dùng `EXPLAIN FORMAT=JSON` và bộ đếm `Handler_read%`.

**Plan tốt sẽ có:**
- `key = idx_orders_shop_created_id_status`, `type = range`
- **Không** có `Using filesort` trong Extra (tree format có `Index range scan ... (reverse)` / Backward index scan)
- `Using index condition` (status được lọc trong index)
- rows examined khoảng 2050 (OFFSET + LIMIT), không phải hàng chục/trăm nghìn; `Handler_read_prev` khoảng vài nghìn
- Với keyset (bước 2): rows examined khoảng 50

Nếu vẫn thấy filesort, hoặc optimizer không chọn index mới, gửi mình output của `EXPLAIN FORMAT=JSON` và optimizer trace để xem nó từ chối index vì sao. Có thể do thống kê cũ, khi đó chạy `ANALYZE TABLE orders;`.

Bảng so sánh để điền:

| Metric | Trước | Sau |
|---|---|---|
| Thời gian | 4–5s | |
| Rows examined | | |
| Using filesort | ? | |
| Index dùng | ? | |

---

## Assumptions / câu hỏi mở

Mình đang giả định: MySQL 8.0, InnoDB, PK là `id`, chưa có index phù hợp, và shop 1207 có lượng lớn đơn từ 01/10. Bạn gửi giúp các thông tin sau (số ước lượng là được, **không cần dữ liệu khách hàng thật**):

**1. Version**
- `SELECT VERSION();` Có phải RDS / Aurora / Cloud SQL hay tự host?

**2. Cấu trúc bảng và index hiện có**
```sql
SHOW CREATE TABLE orders\G
SHOW INDEX FROM orders;
```

**3. Plan hiện tại (trước khi tạo index)**: `EXPLAIN FORMAT=JSON` (hoặc `EXPLAIN ANALYZE` nếu 8.0.18+) của đúng câu query trên.

**4. Khối lượng dữ liệu**
```sql
-- a. kích thước bảng
SELECT table_rows, ROUND(data_length/1024/1024) AS data_mb, ROUND(index_length/1024/1024) AS index_mb
FROM information_schema.tables WHERE table_schema = DATABASE() AND table_name = 'orders';

-- b. độ chọn lọc (chạy trên replica)
SELECT COUNT(*) FROM orders WHERE shop_id = 1207;
SELECT COUNT(*) FROM orders WHERE shop_id = 1207 AND created_at >= '2026-10-01';
SELECT COUNT(*) FROM orders WHERE shop_id = 1207 AND created_at >= '2026-10-01' AND status IN ('paid','shipped');

-- c. phân bố status (có lệch không, ví dụ 80% là 'done'?)
SELECT status, COUNT(*) FROM orders GROUP BY status ORDER BY 2 DESC;
```
- Shop 1207 là shop lớn nhất hay shop bình thường? Shop lớn nhất có bao nhiêu đơn?
- Mỗi ngày thêm bao nhiêu đơn?

**5. Traffic**
- Query này chạy bao nhiêu lần/giây (bình thường / cao điểm)? Mục tiêu latency là bao nhiêu (ví dụ < 100ms)?
- Bảng `orders` có bao nhiêu insert/update mỗi giây? `status` có được update thường xuyên không?
```sql
SELECT digest_text, count_star, ROUND(avg_timer_wait/1e9,2) AS avg_ms, sum_rows_examined, sum_rows_sent
FROM performance_schema.events_statements_summary_by_digest
WHERE digest_text LIKE '%FROM `orders`%' ORDER BY sum_timer_wait DESC LIMIT 10;

SELECT count_insert, count_update, count_delete
FROM performance_schema.table_io_waits_summary_by_table
WHERE object_schema = DATABASE() AND object_name = 'orders';
```
- Query chạy trên primary hay replica? Là màn hình danh sách đơn của shop (OLTP) hay export/báo cáo?
- UI có thật sự cần nhảy tới trang 41 (OFFSET 2000) không, hay chỉ cần "xem thêm" (next/prev)? Câu này quyết định có làm keyset được hay không.

Có các số này mình sẽ chốt giữa ESR và E-R-S, xem có cần covering không, và chỉ ra index cũ nào có thể bỏ.

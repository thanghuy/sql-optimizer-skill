Mình đã review query theo MySQL 8.0.36 (docs tham chiếu: dev.mysql.com/doc/refman/8.0/). Kết luận ngắn: **query này đang full scan 30 triệu dòng mỗi lần tổng đài viên tìm khách**. Nguyên nhân chính là `o.phone = 0912345678` (số so với VARCHAR). Ngoài ra còn một lỗi **sai kết quả** đi kèm. Index thì đã có sẵn và đủ dùng, chỉ cần sửa query.

## Diagnosis

Query có 3 index ứng viên là `idx_phone`, `idx_created` và `idx_coupon`, nhưng cả 3 đều bị vô hiệu hoá:

- `phone = 0912345678`: so cột VARCHAR với một số nguyên, nên MySQL phải đổi **từng dòng** sang số thực (double) để so sánh. Theo tài liệu MySQL 8.0 ([Type Conversion](https://dev.mysql.com/doc/refman/8.0/en/type-conversion.html)): *"MySQL cannot use an index on the column to look up the value quickly."* Vì vậy `idx_phone` không dùng được.
- `DATE(o.created_at) = ...`: bọc cột trong hàm nên `idx_created` chỉ còn quét toàn bộ index, không seek được.
- `(customer_email LIKE '%@gmail.com' OR coupon_code = 'BF2026')`: có wildcard ở đầu, lại OR với cột khác, nên không thể index merge. `idx_coupon` cũng không dùng được.

Hệ quả là optimizer gần như chắc chắn chọn `type = ALL` trên khoảng 30M dòng. Mỗi dòng bị đọc kèm `note TEXT` và `payload JSON` (vì `SELECT *`), sau đó `ORDER BY RAND()` sinh số ngẫu nhiên cho mọi dòng khớp rồi filesort. Màn hình CSKH chạy query này mỗi lượt tìm khách, nên nhiều tổng đài viên cùng tìm một lúc là đủ làm CPU/IO của DB tăng vọt.

(Đây là phân tích từ SQL và schema bạn gửi. Mình chưa thấy EXPLAIN thật, phần Verify bên dưới có lệnh để xác nhận.)

## Issues found

| # | Where (quoted fragment) | Problem | Severity | Why it is slow / wrong | Fix |
|---|---|---|---|---|---|
| 1 | `o.phone = 0912345678` | Implicit cast VARCHAR sang số. **Sai cả kết quả** | **High (hiệu năng + đúng/sai)** | Cột bị cast từng dòng sang double nên `idx_phone` không dùng được, phải full scan. Số `0912345678` cũng mất số 0 đầu và trở thành `912345678`, nên sẽ khớp nhầm các chuỗi như `'912345678'`, `'00912345678'`, `'0912345678abc'`, `' 912345678'`. MySQL còn sinh warning truncation cho các giá trị không phải số. | Truyền đúng kiểu chuỗi: `o.phone = '0912345678'`. Nếu query sinh từ ORM hoặc code, hãy bind tham số dạng string, đừng parse sang int (số điện thoại không phải là số). |
| 2 | `DATE(o.created_at) = '2026-11-01'` | Hàm bọc cột (non-SARGable) | High | Index trên `created_at` không seek được, phải tính `DATE()` cho mọi dòng | Đổi sang khoảng nửa mở: `o.created_at >= '2026-11-01' AND o.created_at < '2026-11-02'` |
| 3 | `ORDER BY RAND()` | Sắp xếp ngẫu nhiên | Medium (High khi chưa sửa #1) | Sinh `RAND()` cho mọi dòng khớp rồi filesort toàn bộ, chỉ để lấy 10 dòng. Kết quả cũng không ổn định: mỗi lần tổng đài viên bấm tìm lại thấy 10 đơn khác nhau, dễ bỏ sót đơn. | Với màn hình CSKH, gần như chắc chắn nên dùng `ORDER BY o.created_at DESC, o.id DESC` (đơn mới nhất trước, có tie-breaker ổn định). Nếu thật sự cần ngẫu nhiên thì xem phần câu hỏi bên dưới. |
| 4 | `SELECT *` | Đọc cả `note TEXT`, `payload JSON` | Medium | Cột TEXT/JSON lớn nằm off-page, nên mỗi dòng phải đọc thêm trang. Khi full scan thì chi phí này nhân lên 30M lần. Cũng tốn băng thông cho dữ liệu mà màn hình danh sách không hiển thị. | Chỉ lấy các cột cần cho danh sách: `id, phone, customer_email, coupon_code, total, created_at`. Lấy `note` hoặc `payload` khi mở chi tiết một đơn (`WHERE id = ?`). |
| 5 | `customer_email LIKE '%@gmail.com' OR coupon_code = 'BF2026'` | Wildcard đầu + OR khác cột | Low (sau khi sửa #1) | Tự nó không index được. Nhưng sau khi `phone` seek được thì đây chỉ là filter trên vài dòng của một khách, nên chi phí không đáng kể. | Giữ nguyên làm filter. **Không** cần tạo index cho `email` hay `coupon` vì query này. (Nếu sau này có query lọc theo domain email trên toàn bảng thì mới tính đến cột `email_domain` generated + index.) |

Lưu ý: #1 là lỗi quan trọng nhất. Kể cả khi không quan tâm tốc độ, query hiện tại vẫn **có thể trả về đơn của người khác** (số điện thoại lưu thiếu số 0, có khoảng trắng hoặc ký tự lạ đầu chuỗi) cho tổng đài viên.

## Rewritten query

```sql
SELECT o.id, o.phone, o.customer_email, o.coupon_code, o.total, o.created_at
FROM orders o
WHERE o.phone = '0912345678'
  AND o.created_at >= '2026-11-01'
  AND o.created_at <  '2026-11-02'
  AND (o.customer_email LIKE '%@gmail.com' OR o.coupon_code = 'BF2026')
ORDER BY o.created_at DESC, o.id DESC
LIMIT 10;
```

Ở ứng dụng, giá trị phone và các mốc ngày nên đi qua bind parameter (prepared statement) dạng **string/datetime**, không nối chuỗi vào SQL. Nối chuỗi vừa có nguy cơ SQL injection, vừa dễ tái phát lỗi #1.

## Recommendation

**1. Sửa query (bắt buộc, không tốn gì).** Chỉ cần sửa #1 là `idx_phone` đã dùng được ngay: `type = ref` trên `phone`, đọc vài chục đơn của một số điện thoại thay vì 30M dòng. Đây là thay đổi mang lại gần như toàn bộ lợi ích.

**2. Index composite: tuỳ chọn, chỉ khi số liệu cho thấy cần.**

```sql
ALTER TABLE orders
  ADD INDEX idx_phone_created (phone, created_at),
  ALGORITHM=INPLACE, LOCK=NONE;
```

- ESR mapping: E = `phone`; S = `created_at DESC, id DESC` (InnoDB tự gắn PK `id` vào cuối mọi secondary index, nên thứ tự `(phone, created_at, id)` khớp ORDER BY khi đọc ngược); R = `created_at` (cùng cột với sort, nên range và sort không xung đột). `customer_email` và `coupon_code` chỉ làm filter trên vài dòng, không đưa vào index.
- Expected effect: seek đúng `phone` + ngày, đọc sẵn theo thứ tự mới nhất, không filesort, và LIMIT dừng sớm. So với `idx_phone` đơn thì chỉ khác biệt rõ khi **một số điện thoại có rất nhiều đơn** (khách sỉ, số tổng đài, số placeholder kiểu `0000000000`), hoặc khi màn hình tìm theo phone **không** kèm ngày mà cần "10 đơn mới nhất".
- Cost: khoảng 30M entry × (≈ 21–81 byte phone utf8mb4 + 5 byte DATETIME + 8 byte PK + overhead), ước chừng 1–1.5 GB. Mỗi INSERT/DELETE tốn thêm một lần ghi index. Build online trên 30M dòng mất vài phút tới hàng chục phút, cần thêm dung lượng tạm tương đương kích thước index. Nên chạy ngoài giờ cao điểm và theo dõi replication lag (hoặc dùng gh-ost / pt-online-schema-change).
- Replaces: `idx_phone` thành thừa, vì nó là leftmost prefix của index mới. Đừng drop ngay, hãy ẩn trước:
  ```sql
  ALTER TABLE orders ALTER INDEX idx_phone INVISIBLE;   -- MySQL 8.0+
  -- theo dõi vài ngày; nếu không query nào chậm đi:
  ALTER TABLE orders DROP INDEX idx_phone, ALGORITHM=INPLACE, LOCK=NONE;
  ```
- Rollback: `ALTER TABLE orders ALTER INDEX idx_phone VISIBLE;` và `ALTER TABLE orders DROP INDEX idx_phone_created;`

Nếu bảng ghi nhiều và mỗi số điện thoại chỉ có ít đơn, mình khuyên **chỉ làm bước 1** và giữ nguyên `idx_phone`.

**3. Không cần thêm gì cho `idx_created` / `idx_coupon` vì query này.** Nhưng hãy kiểm tra xem chúng có query nào khác dùng không (lệnh `sys.schema_unused_indexes` bên dưới). Index không ai dùng trên bảng 30M dòng chỉ làm chậm ghi.

## Verify

Chạy trên staging hoặc replica có dữ liệu giống production (MySQL 8.0.36 hỗ trợ cả hai lệnh):

```sql
EXPLAIN FORMAT=TREE
SELECT ... ;          -- query đã viết lại, ước lượng, không chạy thật

EXPLAIN ANALYZE
SELECT ... ;          -- chạy thật (SELECT nên an toàn), có actual rows/time

-- so sánh trước/sau:
FLUSH STATUS;
SELECT ... ;
SHOW SESSION STATUS LIKE 'Handler_read%';
```

Plan tốt cần có:
- Query cũ: `Table scan on o` / `type = ALL`, `rows` khoảng 30M, `Using filesort`, `Handler_read_rnd_next` khoảng 30M. Đây là baseline để so sánh.
- Query mới (chỉ `idx_phone`): `Index lookup on o using idx_phone (phone='0912345678')`, actual rows bằng số đơn của số đó (vài chục), có thể có sort nhỏ.
- Query mới + `idx_phone_created`: `Index range scan on o using idx_phone_created (reverse)`, **không** có `Sort` / `Using filesort`, actual rows ≤ vài chục, thời gian ở mức ms.
- Nếu vẫn thấy `key = NULL`: kiểm tra xem tham số có thật sự được gửi dưới dạng chuỗi không (bật general log hoặc xem `performance_schema.events_statements_history` để thấy SQL thực tế mà ORM gửi).

## Assumptions / open questions

Vì đây là lần trả lời một lượt, mình đã giả định như sau. Bạn trả lời giúp để mình chốt có cần index composite không:

1. **Thứ tự hiển thị:** `ORDER BY RAND()` có phải là cố ý không? Mình giả định CSKH cần "đơn mới nhất trước". Nếu thật sự cần lấy ngẫu nhiên 10 đơn thì sau khi sửa #1 tập dữ liệu đã rất nhỏ, nên `RAND()` trên vài chục dòng chấp nhận được. Chỉ là kết quả sẽ không lặp lại được giữa các lần bấm.
2. **Bộ lọc thực tế của màn hình:** tổng đài viên luôn tìm theo phone + một ngày cụ thể, hay thường chỉ theo phone? Còn điều kiện email gmail / coupon là cố định hay do người dùng chọn? Nếu là bộ lọc tuỳ chọn thì nên sinh SQL động theo từng tổ hợp, không dùng kiểu catch-all.
3. **Phân bố dữ liệu:** một số điện thoại có tối đa bao nhiêu đơn? Có số "rác" nào có rất nhiều đơn không?
   ```sql
   SELECT phone, COUNT(*) c FROM orders GROUP BY phone ORDER BY c DESC LIMIT 20;  -- chạy trên replica
   ```
4. **Định dạng phone đã lưu:** có chuẩn hoá không (luôn `09...`, hay lẫn `+84...`, có khoảng trắng)? Nếu lẫn định dạng thì nên chuẩn hoá khi ghi. Đừng "sửa" bằng cách bọc hàm quanh cột trong WHERE, vì như vậy lại mất index.
5. **Traffic:** số lượt tìm mỗi phút lúc cao điểm, latency hiện tại và mục tiêu, cùng số insert/update mỗi giây vào `orders`:
   ```sql
   SELECT digest_text, count_star, ROUND(avg_timer_wait/1e9,2) avg_ms, sum_rows_examined, sum_rows_sent, sum_no_index_used
   FROM performance_schema.events_statements_summary_by_digest
   WHERE digest_text LIKE '%FROM `orders`%' ORDER BY sum_timer_wait DESC LIMIT 10;

   SELECT count_insert, count_update, count_delete
   FROM performance_schema.table_io_waits_summary_by_table
   WHERE object_schema = DATABASE() AND object_name = 'orders';
   ```
6. **Index hiện có:** gửi giúp kết quả `SHOW INDEX FROM orders;`, `SELECT * FROM sys.schema_unused_indexes WHERE object_name='orders';` và cho biết có chạy trên RDS/Aurora không, để mình kiểm tra `idx_created` / `idx_coupon` có còn ai dùng không.

Gửi mình kết quả `EXPLAIN ANALYZE` của query mới, mình sẽ xem tiếp và chốt có cần `idx_phone_created` hay không.

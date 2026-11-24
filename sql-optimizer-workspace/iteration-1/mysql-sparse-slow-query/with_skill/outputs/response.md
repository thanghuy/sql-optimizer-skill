Mình đưa luôn một index có thể tạo ngay, kèm cách kiểm chứng. Hiện bạn chưa gửi schema, index đang có, số dòng hay plan, nên phần dưới dựa trên **giả định** (ghi rõ ở cuối). Gửi thêm vài thông tin ở mục "Cần bạn gửi" thì mình chốt lại cho chắc, tránh tạo nhầm index trên prod.

## Chẩn đoán (tạm thời, chưa có plan)

Query có 3 vấn đề:

1. **`status IN ('paid','shipped')` làm mất thứ tự sort.** Nếu index là `(shop_id, status, created_at)` (kiểu index hay được tạo nhất), MySQL seek được 2 khoảng (paid, shipped). Nhưng mỗi khoảng có thứ tự `created_at` riêng, MySQL không merge 2 khoảng theo thứ tự được. Vì vậy nó phải đọc **toàn bộ** đơn paid/shipped của shop từ 01/10, rồi **filesort**, rồi mới cắt LIMIT. Nếu chỉ có index `(shop_id)` hoặc không có index phù hợp thì còn tệ hơn: đọc hết đơn của shop, lookup từng dòng về bảng, rồi sort.
2. **`OFFSET 2000`**: MySQL phải đọc và bỏ đi 2000 dòng trước khi trả 50 dòng. Trang càng sâu càng chậm.
3. **Không covering**: mỗi dòng đọc từ index lại phải quay về clustered index (PK) để lấy `order_no`, `total`. Hàng nghìn lần random I/O như vậy dễ lên vài giây.

## Đề xuất

### 1. Index chính (theo ESR, có điều chỉnh vì `IN`)

```sql
ALTER TABLE orders
  ADD INDEX idx_orders_shop_created_cover (shop_id, created_at, id, status, total, order_no),
  ALGORITHM=INPLACE, LOCK=NONE;
```

Vai trò từng cột theo ESR:
- **E** = `shop_id`: điều kiện bằng.
- **S** = `created_at`: trùng với `ORDER BY created_at DESC`. Cột này đồng thời là **R** (`>= '2026-10-01'`). Vì sort và range nằm trên cùng một cột nên đặt ngay sau E là tối ưu: MySQL seek tới `shop_id = 1207`, quét ngược từ mới nhất về 01/10, và **dừng sớm** khi đủ 2050 dòng khớp. Không có filesort.
- `id`: thêm tường minh để thứ tự trong index là `(created_at, id)`. Nhờ vậy keyset pagination ở bước 2 với tie-breaker `id` cũng không phải sort. Trong InnoDB, PK vốn được nối vào cuối mọi secondary index; khai báo tường minh ở đây thì InnoDB không lưu thêm lần nữa, chỉ đổi vị trí của nó.
- `status`: **cố ý không đặt trước `created_at`**. Đây là ngoại lệ của ESR: `IN` nhiều giá trị đứng trước cột sort sẽ gây filesort. Ở vị trí này, `status` được lọc ngay trong index, không cần đọc bảng.
- `total`, `order_no`: để index **covering**. Kỳ vọng EXPLAIN hiện `Using index`, không lookup về bảng.

Hiệu quả kỳ vọng: số entry phải đọc ≈ 2050 × (số đơn của shop từ 01/10 ÷ số đơn paid+shipped trong đó). Nếu paid+shipped chiếm khoảng 50–90% đơn (thường gặp), MySQL chỉ đọc khoảng 2–4 nghìn entry trong index. Query thường sẽ xuống mức vài ms đến vài chục ms.

Khi nào index này **không** tốt nhất: khi paid+shipped rất hiếm, ví dụ dưới 2–5% đơn của shop từ 01/10. Lúc đó quét theo thời gian phải bỏ qua quá nhiều dòng. Phương án ngược lại, `(shop_id, status, created_at, total, order_no)` (seek đúng 2 status rồi filesort một tập nhỏ), sẽ rẻ hơn. Vì vậy mình cần số liệu phân bố `status` ở dưới.

Chi phí:
- Thêm 1 secondary index: mỗi INSERT, và mỗi UPDATE đổi `status`/`total`/`order_no`, tốn thêm một lần ghi index. Bảng orders thường đổi `status` vài lần trong vòng đời đơn, nên mức này thường chấp nhận được, nhưng cần biết lượng ghi/giây.
- Dung lượng: khoảng (8 + 5 + 8 + ~10 + 8 + độ dài `order_no`) byte × số dòng. Ví dụ 50M dòng, khoảng 60–70 B/entry, tức cỡ 3–4 GB. Lúc build cần thêm temp space cỡ đó.
- Build online với `LOCK=NONE` vẫn tốn I/O và có thể gây replication lag trên bảng lớn. Nên chạy ngoài giờ cao điểm. Bảng rất lớn thì dùng `gh-ost` / `pt-online-schema-change`.

Nếu bảng ghi rất nhiều và bạn muốn index gọn hơn: `(shop_id, created_at, status)` vẫn bỏ được filesort, chỉ mất phần covering, nên vẫn phải lookup khoảng 2050 dòng về PK.

Rollback: thử trước bằng cách tắt index, rồi xoá nếu không cần.
```sql
ALTER TABLE orders ALTER INDEX idx_orders_shop_created_cover INVISIBLE;  -- MySQL 8.0+, tắt mà không xoá
ALTER TABLE orders DROP INDEX idx_orders_shop_created_cover, ALGORITHM=INPLACE, LOCK=NONE;
```

Sau khi index mới chạy ổn vài ngày, kiểm tra index cũ dạng `(shop_id)` hoặc `(shop_id, created_at)`. Đó là leftmost prefix của index mới nên thường thừa (trừ khi phục vụ foreign key). Xem `sys.schema_unused_indexes` / `sys.schema_redundant_indexes`, cho INVISIBLE trước, rồi mới DROP.

### 2. Sửa query: bỏ `OFFSET`, dùng keyset pagination

`OFFSET 2000` là chi phí cố định không index nào xoá được, và trang càng sâu càng chậm. Bạn nên cho app nhớ dòng cuối của trang trước (`created_at`, `id`) rồi lấy trang tiếp theo như sau:

```sql
-- Trang đầu
SELECT id, order_no, total, status, created_at
FROM orders
WHERE shop_id = 1207
  AND status IN ('paid','shipped')
  AND created_at >= '2026-10-01'
ORDER BY created_at DESC, id DESC
LIMIT 50;

-- Trang sau: :last_created_at, :last_id = dòng cuối của trang trước
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

- Thêm `id DESC` để thứ tự ổn định. Nhiều đơn cùng `created_at` sẽ không bị lặp hoặc mất giữa các trang (query hiện tại có rủi ro này).
- Viết dạng `OR` thay cho `(created_at, id) < (?, ?)`, vì MySQL không phải lúc nào cũng dùng range scan cho row constructor.
- Cột `id` đặt ngay sau `created_at` trong index ở trên chính là để phục vụ `ORDER BY created_at DESC, id DESC` này mà không cần sort.
- Đổi kiểu phân trang ảnh hưởng UI (không nhảy thẳng tới "trang 41" được), nên cần bàn với team app. Nếu buộc phải giữ OFFSET thì index ở mục 1 vẫn giúp nhiều, vì 2050 entry covering đọc rất nhanh.

Lưu ý nhỏ: nếu `created_at` là `DATETIME`/`TIMESTAMP` thì so với chuỗi `'2026-10-01'` vẫn dùng index bình thường. Nếu nó là `VARCHAR` thì cần báo mình. Kiểu `status` (ENUM hay VARCHAR) không ảnh hưởng, miễn không bọc hàm quanh cột.

## Kiểm chứng

Chạy trên staging có dữ liệu giống prod, hoặc trên replica. `EXPLAIN ANALYZE` **thực thi** query; với SELECT thì an toàn về dữ liệu, nhưng vẫn tốn tài nguyên như chạy thật.

```sql
SELECT VERSION();

-- Trước và sau khi tạo index
EXPLAIN FORMAT=TREE
SELECT id, order_no, total, status, created_at FROM orders
WHERE shop_id = 1207 AND status IN ('paid','shipped') AND created_at >= '2026-10-01'
ORDER BY created_at DESC LIMIT 50 OFFSET 2000;

EXPLAIN ANALYZE                -- MySQL 8.0.18+
SELECT id, order_no, total, status, created_at FROM orders
WHERE shop_id = 1207 AND status IN ('paid','shipped') AND created_at >= '2026-10-01'
ORDER BY created_at DESC LIMIT 50 OFFSET 2000;

-- Đếm số dòng đọc thực tế
FLUSH STATUS;
SELECT ... ;                   -- chạy lại query
SHOW SESSION STATUS LIKE 'Handler_read%';
```

Plan tốt sẽ có:
- `key = idx_orders_shop_created_cover`, `type = range`.
- `Extra` có `Using index` (covering), có thể kèm `Backward index scan`, và **không** có `Using filesort`.
- Trong `EXPLAIN ANALYZE`: số dòng thực tế đọc từ index chỉ vài nghìn, gần 2050, không phải hàng trăm nghìn. Không có node `Sort`.
- `Handler_read_prev` khoảng vài nghìn, `Handler_read_rnd_next` khoảng 0.

Nếu vẫn thấy `Using filesort` hoặc số dòng đọc lớn hơn nhiều so với 2050, gửi lại output cho mình phân tích tiếp. Thường đó là do paid+shipped hiếm (khi đó đổi sang phương án `(shop_id, status, created_at, ...)`) hoặc do thống kê cũ (`ANALYZE TABLE orders;`).

Mẹo nhỏ: nếu query gần như luôn `ORDER BY created_at DESC`, có thể khai báo `created_at DESC` trong index (MySQL 8.0+) để tránh backward scan. Lợi ích nhỏ, không bắt buộc.

## Giả định và câu hỏi

Mình đang giả định:
- MySQL 8.0+ trên InnoDB, PK là `id` (BIGINT).
- Bảng lớn (hàng chục triệu dòng), shop 1207 có nhiều đơn.
- paid+shipped chiếm phần lớn đơn.
- Chưa có index nào phù hợp.

**Cần bạn gửi** (chạy được trên replica; không cần dữ liệu khách hàng, chỉ cần schema và số đếm):

1. Version: `SELECT VERSION();`. Có phải RDS hoặc Aurora không?
2. Cấu trúc và index hiện có:
   ```sql
   SHOW CREATE TABLE orders\G
   SHOW INDEX FROM orders;
   ```
3. Khối lượng dữ liệu và phân bố:
   ```sql
   SELECT table_rows, ROUND(data_length/1024/1024) data_mb, ROUND(index_length/1024/1024) index_mb
   FROM information_schema.tables WHERE table_schema = DATABASE() AND table_name = 'orders';

   SELECT COUNT(*) FROM orders WHERE shop_id = 1207;
   SELECT status, COUNT(*) FROM orders
   WHERE shop_id = 1207 AND created_at >= '2026-10-01' GROUP BY status;
   ```
   Thêm số đơn của shop lớn nhất (trường hợp xấu nhất) và số đơn mới mỗi ngày.
4. Traffic:
   - Query này chạy bao nhiêu lần/giây (hoặc /phút), lúc bình thường và lúc cao điểm?
   - Latency mục tiêu là bao nhiêu?
   - Bảng `orders` có bao nhiêu insert/update mỗi giây?
   - Có thể lấy số liệu từ:
   ```sql
   SELECT digest_text, count_star, ROUND(avg_timer_wait/1e9,2) avg_ms, sum_rows_examined, sum_rows_sent
   FROM performance_schema.events_statements_summary_by_digest
   WHERE digest_text LIKE '%FROM `orders`%' ORDER BY sum_timer_wait DESC LIMIT 10;
   ```
5. Plan hiện tại: output `EXPLAIN FORMAT=TREE` (và `EXPLAIN ANALYZE` nếu được) của query trên.
6. Query do ORM sinh ra không? `OFFSET` có thể lên sâu tới đâu? Ngoài query này, `orders` còn query nóng nào khác lọc theo `shop_id` không? Mình cần biết để gộp index, không tạo thêm index thừa.

Có số ở mục 3 là mình chốt được nên dùng `(shop_id, created_at, …)` hay `(shop_id, status, created_at, …)`.

*Tham khảo: MySQL 8.0 Reference Manual, ORDER BY Optimization (https://dev.mysql.com/doc/refman/8.0/en/order-by-optimization.html): index chỉ tránh được sort khi các cột đứng trước cột ORDER BY là hằng (điều kiện bằng); index DESC hoặc backward scan dùng được cho `ORDER BY ... DESC`.*

# sql-optimizer — AI skill tối ưu SQL cho 10 hệ DB

Skill theo chuẩn mở **Agent Skills** (`SKILL.md` + thư mục `references/`), cắm được vào Claude Code, Claude.ai / Claude Desktop, và các agent khác đọc được Markdown (Cursor, Codex, Gemini CLI, Copilot…).

Hỗ trợ: PostgreSQL, MySQL, MariaDB, SQL Server, Oracle, SQLite, IBM Db2, Snowflake, BigQuery, ClickHouse.

## Quy trình skill thực hiện

| Bước | Việc làm | Vì sao |
|---|---|---|
| 0 | Hỏi engine + **version chính xác** | Doc và cú pháp sinh ra phải đúng version (vd `EXPLAIN (GENERIC_PLAN)` chỉ có từ PG 16) |
| 1 | Đọc doc chính thức của đúng version | Tránh khuyên sai do kiến thức cũ |
| 2 | Hỏi cấu trúc bảng + index hiện có | Thường nên mở rộng index cũ thay vì thêm mới |
| 3 | Hỏi lượng data, cardinality, skew | Selectivity quyết định index có được dùng không |
| 4 | Hỏi traffic: QPS, read/write, SLO | Cân lợi đọc vs chi phí ghi của mỗi index |
| 5 | **Soát lỗi query** → bảng *Issues found* (vị trí, lỗi, mức độ, vì sao chậm, cách sửa) + query viết lại | Nhiều query chậm không phải do thiếu index mà do làm index vô dụng |
| 6 | Thiết kế index theo **ESR** (Equality → Sort → Range) | Seek trên E, đọc sẵn thứ tự S, lọc R — bỏ được bước sort |
| 7 | Xin query plan thật, phân tích, lặp lại | Chứng minh bằng số liệu trước/sau |

Với Snowflake / BigQuery / ClickHouse (không có B-tree), ESR được chuyển thành thứ tự partition / clustering / sort key.

## Cấu trúc

```
sql-optimizer/                       # thư mục skill (cái cần cài)
├── SKILL.md                         # quy trình chính
├── assets/intake-template.md        # bảng câu hỏi gửi user một lần
└── references/
    ├── query-antipatterns.md        # danh mục lỗi query + lỗi "vô hình" (sniffing, bind type, collation, stats, FK) + những thứ trông sai nhưng ổn
    ├── esr-rule.md                  # ESR, ngoại lệ, covering, tie-breaker, checklist trước khi DROP index
    ├── query-plan-analysis.md       # đọc plan, red flags, so sánh trước/sau
    └── databases/<db>.md ×10        # link doc theo version, lệnh lấy schema/index/stats/traffic/plan

evals/                               # 16 test case (không nằm trong gói skill)
sql-optimizer-workspace/             # kết quả 4 vòng test + trang review HTML
```

## Cài đặt

**Claude Code** (dùng cho mọi project):
```bash
cp -r sql-optimizer ~/.claude/skills/
```
Hoặc chỉ cho một project: copy vào `<project>/.claude/skills/`.

**Claude.ai / Claude Desktop**: Settings → Capabilities → Skills → Upload, chọn `sql-optimizer.skill`.

**Agent khác (Cursor, Codex, Gemini CLI, Copilot…)**: copy thư mục `sql-optimizer` vào repo và thêm vào `AGENTS.md` / rules file:
```
For any SQL performance, slow query, query review, index or EXPLAIN task, follow sql-optimizer/SKILL.md
and read the referenced files it points to.
```

## Kết quả test (4 vòng, có skill vs không skill)

| Vòng | Nội dung | Có skill | Không skill |
|---|---|---|---|
| 1 | Thiết kế index, 4 DB | 32/33 | 32/33 |
| 2 | Tiêu chí siết + version (6 case) | **52/52** | 46/52 |
| 3 | Lỗi query cài sẵn + query sạch | 42/42 | 42/42 |
| 4 | Lỗi khó: sniffing, bind type, collation, FK, stats | **40/40** | 36/40 |

Model gốc đã có kiến thức DB tốt; giá trị đo được của skill là **kỷ luật quy trình**: hỏi đủ traffic/data, ghi version tối thiểu cho tính năng, index khớp ORDER BY + tie-breaker, bảng lỗi có mức độ, tự kiểm tra con số. Đổi lại mỗi lượt chậm hơn ~45–55s và tốn thêm ~20–30k token.

## Dùng thử

> Query này chạy 3s trên MySQL 8.0, bảng orders 50 triệu dòng:
> `SELECT * FROM orders WHERE customer_id = 42 AND created_at > '2026-01-01' ORDER BY total DESC LIMIT 20`

Agent sẽ soát lỗi query ngay, hỏi cấu trúc bảng/index/data/traffic, đề xuất index theo ESR (hoặc E-R-S tuỳ selectivity), rồi xin `EXPLAIN ANALYZE` để kiểm chứng.

## License

[MIT](LICENSE) — tự do dùng, sửa, phân phối lại, kể cả cho mục đích thương mại, miễn giữ lại thông báo bản quyền.

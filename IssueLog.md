# Nhật Ký Vấn Đề

Ghi lại các vấn đề kiến trúc / thiết kế đang mở, chưa có quyết định cuối.

---

## [OPEN] DO-LIVE-001 — Chuyển Delivery Order sang live query Odoo (bỏ local DB sync)

**Ngày mở:** 2026-04-10  
**Người mở:** Ngo Huu Duc  
**Trạng thái:** OPEN — chờ quyết định về cơ chế khoá cutoff

### Bối cảnh

Hiện tại:
- **Purchase Order**: live query từ Odoo qua XML-RPC, không lưu local DB.
- **Delivery Order**: sync về PostgreSQL mỗi 30 phút qua `do_sync` job.

Đề xuất: chuyển DO sang live query Odoo giống PO — bỏ bảng `delivery_orders` và `delivery_order_lines`, đọc/ghi trực tiếp `stock.picking` + `stock.move.line`.

### Blocker 1 — Cơ chế khoá cutoff

Trạng thái `locked` là khái niệm riêng của portal, Odoo không có tương đương.

#### Hướng A — Cutoff gọi `button_validate()` trên Odoo
- Picking chuyển `assigned → done` lúc 23:00.
- Khoá tự nhiên: picking `done` không ai sửa được.
- **Rủi ro**: validate trước khi hàng thực sự được nhận, cửa hàng mất quyền tự validate Receipt.

#### Hướng B — Cutoff chỉ block portal UI (khuyến nghị)
- Tạo bảng nhỏ `portal_cutoff_locks(picking_id, cut_at)`.
- PATCH endpoint kiểm tra: nếu `picking_id` trong bảng → trả 423 Locked.
- Odoo picking giữ `assigned`; cửa hàng tự validate khi nhận hàng thực tế.
- **Ưu điểm**: tách biệt "vendor không sửa được nữa" vs "đã nhận hàng" — đúng nghiệp vụ hơn.

### Blocker 2 — Save phải ghi qty_done lên Odoo ngay lập tức

Hiện tại nút **Save** chỉ ghi vào local DB; cutoff job mới đẩy `qty_done` lên Odoo.  
Nếu bỏ local DB, mỗi lần vendor bấm Save phải ghi `qty_done` từng dòng `stock.move.line` lên Odoo qua XML-RPC ngay lập tức.  
Với phiếu nhiều dòng (50+), có thể ảnh hưởng tốc độ phản hồi.

### Blocker 3 — Nhập nhằng `received_qty`

`stock.move.line.qty_done` phục vụ hai mục đích:
- Trước khi picking `done`: là số lượng vendor nhập (delivery qty).
- Sau khi picking `done`: là số lượng cửa hàng xác nhận thực nhận (received qty) — cửa hàng có thể điều chỉnh.

Hiện tại hai giá trị được lưu thành hai cột riêng trong local DB.  
Nếu chuyển sang live, sau khi receipt validation cửa hàng ghi đè `qty_done`, ta mất số liệu vendor đã nhập ban đầu.

**Phương án:** Lưu delivery qty của vendor vào bảng `portal_cutoff_locks` cùng lúc khoá, hoặc chấp nhận chỉ hiển thị qty cuối cùng từ Odoo.

### Blocker 4 — Timestamp khoá (`locked_at`)

Banner "Đã khoá lúc 23:00 ngày {date}" cần biết thời điểm khoá.  
Odoo không có trường này trên `stock.picking`.  
Dù chọn Hướng A hay B, đều cần bảng `portal_cutoff_locks(picking_id, cut_at)` để lưu timestamp.

### Blocker 5 — Phân trang & tìm kiếm danh sách DO

Hiện tại dùng SQL `LIMIT/OFFSET` + `WHERE` trên portal DB.  
Live từ Odoo: `search_read` trên `stock.picking` hỗ trợ limit/offset nhưng lọc theo tên PO (`origin`), ngày, vendor cùng lúc kém linh hoạt hơn.  
Cần thiết kế domain Odoo cẩn thận.

### Blocker 6 — Admin unlock

Hiện tại: cập nhật `delivery_orders.status` trong portal DB.  
Live: tương đương là xoá dòng tương ứng khỏi `portal_cutoff_locks`.  
Đơn giản nếu chọn Hướng B.

### Việc cần làm (khi có quyết định)

- [ ] Chọn Hướng A hoặc B cho cutoff (Blocker 1)
- [ ] Tạo bảng `portal_cutoff_locks(picking_id, cut_at)` (Blocker 1, 4)
- [ ] Quyết định xử lý `received_qty` (Blocker 3)
- [ ] Xoá bảng `delivery_orders`, `delivery_order_lines` (migration drop)
- [ ] Xoá `do_sync` job (hoặc chuyển thành cutoff-only job)
- [ ] Viết `OdooDeliveryClient` — live GET picking / move lines + batch supplier info + UoM
- [ ] Cập nhật `delivery_orders.py` API — đọc từ Odoo, Save ghi trực tiếp lên Odoo (Blocker 2)
- [ ] Cập nhật `do_cutoff.py` — chỉ khoá, không cần push qty_done
- [ ] Cập nhật PDF generation — đọc từ Odoo
- [ ] Cập nhật Admin unlock flow — xoá khỏi `portal_cutoff_locks` (Blocker 6)

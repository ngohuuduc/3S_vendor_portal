# Issue Log

Ghi lại các vấn đề kiến trúc / thiết kế đang mở, chưa có quyết định cuối.

---

## [OPEN] DO-LIVE-001 — Chuyển Delivery Order sang live query Odoo (bỏ local DB sync)

**Ngày mở:** 2026-04-10  
**Người mở:** Ngo Huu Duc  
**Trạng thái:** OPEN — chờ quyết định về cutoff job

### Bối cảnh

Hiện tại:
- **Purchase Order**: live query từ Odoo qua XML-RPC, không lưu local DB.
- **Delivery Order**: sync về PostgreSQL mỗi 30 phút qua `do_sync` job.

Đề xuất: chuyển DO sang live query Odoo giống PO — bỏ bảng `delivery_orders` và `delivery_order_lines`, đọc/ghi trực tiếp `stock.picking` + `stock.move.line`.

### Vấn đề còn mở

**Câu hỏi:** Sau 23:00 cutoff, picking trên Odoo nên ở trạng thái nào?

#### Hướng A — Cutoff gọi `button_validate()` trên Odoo

- Picking chuyển `assigned → done` lúc 23:00.
- Lock tự nhiên: picking `done` không ai sửa được.
- **Rủi ro**: validate trước khi hàng thực sự được nhận, cửa hàng mất quyền tự validate Receipt.

#### Hướng B — Cutoff chỉ block portal UI (khuyến nghị)

- Tạo bảng nhỏ `portal_cutoff_locks(picking_id, cut_at)`.
- PATCH endpoint kiểm tra: nếu `picking_id` trong bảng → trả 423 Locked.
- Odoo picking giữ `assigned`; cửa hàng tự validate khi nhận hàng thực tế.
- **Ưu điểm**: tách biệt "vendor không sửa được nữa" vs "đã nhận hàng" — đúng nghiệp vụ hơn.

### Việc cần làm (khi có quyết định)

- [ ] Chọn Hướng A hoặc B cho cutoff
- [ ] Xoá bảng `delivery_orders`, `delivery_order_lines` (migration drop)
- [ ] Xoá `do_sync` job (hoặc chuyển thành cutoff-only job)
- [ ] Viết `OdooDeliveryClient` — live GET picking/move lines + batch supplier info + uom
- [ ] Cập nhật `delivery_orders.py` API — đọc từ Odoo thay vì portal DB
- [ ] Cập nhật `do_cutoff.py` — không cần push qty_done (đã ghi trên Save), chỉ lock
- [ ] Quyết định xử lý Blocker 2 (per-line note)
- [ ] Cập nhật PDF generation — đọc từ Odoo
- [ ] Cập nhật Admin unlock flow

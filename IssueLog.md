# Nhật Ký Vấn Đề

Ghi lại các vấn đề kiến trúc / thiết kế đang mở, chưa có quyết định cuối.

---

## [OPEN] LOG-NOTE-ATTRIBUTION-001 — Người thực hiện trong Log Note khi vendor edit qua Portal

**Ngày mở:** 2026-05-24
**Người mở:** trang.nat
**Trạng thái:** OPEN — chờ Đức quyết định có làm hay không (do phức tạp thực thi)

### Vấn đề

Spec yêu cầu Log Note hiển thị tên `res.users.name` của người thực hiện thay đổi — cho cả thay đổi do vendor làm trên Portal lẫn thay đổi do cửa hàng/admin làm trực tiếp trên Odoo.

**Vấn đề thực thi:** Portal ghi xuống Odoo qua **service account** (1 tài khoản XML-RPC duy nhất, dùng chung cho mọi vendor). Vì vậy:
- `mail.tracking.value.create_uid` luôn là **service account**, không phải vendor cụ thể
- Hiển thị tên service account → vô nghĩa với vendor (NCC sẽ thấy mọi thay đổi đều do "vendor-portal-svc" thực hiện)
- Để hiển thị đúng tên vendor, Portal phải **chủ động inject metadata** vào `mail.message.body` (ví dụ "Thực hiện bởi: NCC ABC qua Portal") lúc ghi xuống, sau đó parse khi đọc — implementation phức tạp hơn

### Lựa chọn

- **A.** Bỏ qua attribution chi tiết: chỉ hiển thị Log Note cho thay đổi từ cửa hàng/admin (Odoo native users), không track changes do vendor làm — đơn giản nhưng vendor mất khả năng xem lại thao tác của chính mình
- **B.** Inject vendor name vào message body khi Portal ghi — phức tạp hơn nhưng hiển thị đầy đủ
- **C.** Hiển thị "service account" cho mọi thay đổi từ Portal — đơn giản nhất nhưng UX kém

### Hành động kế tiếp

- [ ] Đức review và quyết định A / B / C
- [ ] Cập nhật spec Log Note theo quyết định

---

## [OPEN] PRINT-DO-001 — Nút In Phiếu Giao Nhận chưa in được

**Ngày mở:** 2026-05-24
**Người mở:** trang.nat
**Trạng thái:** OPEN — bug implementation, không phải vấn đề spec
**Cần escalate:** Đức review để đánh giá ưu tiên / SLA

### Mô tả

Nút "In" trên giao diện Phiếu Giao Nhận (DO Form) đang bị lỗi — bấm vào không tạo được PDF / không tải xuống. Spec hiện vẫn yêu cầu PDF tạo on-the-fly mỗi lần in (Vietnamese, không lưu trên server) — chức năng đúng spec nhưng không hoạt động.

### Phạm vi cần kiểm tra

- Endpoint backend tạo PDF (WeasyPrint hoặc tương đương)
- Quyền truy cập (vendor-scoped)
- Cả 4 loại phiếu: Phiếu Giao Hàng (DO), Phiếu Nhận Hàng (RO) — kiểm tra xem cả 2 cùng lỗi hay chỉ DO

### Hành động kế tiếp

- [ ] Dev verify endpoint generate PDF
- [ ] Test thủ công trên cả 4 loại phiếu (PO/PGH/ĐTH/PNH) sau khi tách menu

---

## [OPEN] RO-AUTOCONFIRM-001 — Odoo tự xác nhận trả hàng sau hạn chót

**Ngày mở:** 2026-04-18
**Người mở:** trang.nat
**Trạng thái:** OPEN — dependency phía Odoo team, chờ xác nhận

### Bối cảnh

Decision mới (2026-04-18): hạn chót của RO = `pickup_date` + 7 ngày dương lịch. Sau hạn chót, **Odoo tự xác nhận trả hàng** (validate `stock.picking` outgoing) và portal chỉ đọc state `done` từ Odoo để reflect lên UI. Portal KHÔNG đẩy validate lên Odoo.

### Dependency cần Odoo team xác nhận

1. **Scheduled action / cron** tự validate picking trả hàng khi `scheduled_date + 7 ngày <= now` — Odoo 16 CE standard không có sẵn chức năng này
2. **Thời điểm chính xác chạy scheduled action** trong ngày (đầu ngày, cuối ngày, hay theo cron định kỳ?) — ảnh hưởng đến banner hiển thị ngày hạn chót
3. **Side effect hàng tồn kho:** khi auto-validate, Odoo sẽ tạo stock move outgoing, trừ tồn kho cửa hàng. Cần xác nhận đây là behavior mong muốn (vs. chỉ đổi state không tạo move)
4. **Cơ chế rollback:** nếu vendor thực tế đến lấy sau hạn chót, có reverse được không?

### Các quyết định đã gắn với issue này

- [README.md — dòng quyết định "Trả hàng"](../README.md)
- [PROCESS_FLOW.md — RO state machine + comparison table](../PROCESS_FLOW.md)

### Hành động kế tiếp

- [ ] trang.nat raise với Odoo team về các điểm trên
- [ ] Ghi lại câu trả lời vào issue này rồi đóng

---

## [CLOSED] DO-LIVE-001 — Chuyển Delivery Order sang live query Odoo (bỏ local DB sync)

**Ngày mở:** 2026-04-10  
**Ngày đóng:** 2026-04-11  
**Người mở:** Ngo Huu Duc  
**Trạng thái:** CLOSED — đã implement đầy đủ

### Bối cảnh

Hiện tại:
- **Purchase Order**: live query từ Odoo qua XML-RPC, không lưu local DB.
- **Delivery Order**: sync về PostgreSQL mỗi 30 phút qua `do_sync` job.

Đề xuất: chuyển DO sang live query Odoo giống PO — bỏ bảng `delivery_orders` và `delivery_order_lines`, đọc/ghi trực tiếp `stock.picking` + `stock.move.line`.

### ~~Blocker 1~~ — Cơ chế khoá cutoff — **DECIDED: Hướng C**

**Quyết định (2026-04-11):** Khoá hoàn toàn về mặt giao diện, **không tác động gì lên Odoo, không cần bảng portal_cutoff_locks**.

- Lock state được **derive thuần túy từ `scheduled_date`** của picking: nếu `scheduled_date.date() <= today (UTC+7)` → locked.
- PATCH endpoint kiểm tra điều kiện này tại thời điểm request — không cần job, không cần bảng phụ.
- Odoo picking giữ nguyên state `assigned` — cửa hàng tự validate khi nhận hàng thực tế.
- `do_cutoff.py` job **bị xoá hoàn toàn** — không còn việc gì để làm.
- Admin muốn unlock → cập nhật `scheduled_date` trực tiếp trên Odoo.

`push_do_to_odoo()` (hiện push `scheduled_date`, `note`, `qty_done` tại cutoff) **bị bỏ hoàn toàn** — tất cả các giá trị này được ghi real-time qua PATCH endpoint.

### ~~Blocker 2~~ — Save phải ghi qty_done lên Odoo ngay lập tức — **RESOLVED**

**Kết luận (2026-04-11):** Thực tế rất ít PO có trên 10 dòng. Phải gọi N lần `write()` riêng biệt (mỗi dòng một `qty_done` khác nhau), nhưng với N < 10 thì chấp nhận được. Dùng `asyncio.gather()` để song song hoá. Khi implement nên thêm `logger.info("elapsed=%.3fs", ...)` để có số liệu thực tế.

### ~~Blocker 3~~ — Nhập nhằng `received_qty` — **RESOLVED**

**Quyết định (2026-04-11), được chị xác nhận lại 2026-05-24:** Vendor không cần thấy số lượng gốc của mình sau khi cửa hàng validate. **Chỉ dùng 1 field duy nhất** `stock.move.line.qty_done` (hoặc `stock.move.quantity_done` ở cấp move = computed sum) — vendor ghi vào field này trong state Mới, cửa hàng ghi đè (nếu có sửa khi confirm). Portal đổi **label** theo picking state:

- `assigned` (Mới / Đã Gửi): hiển thị label **"SL Giao"** (vendor đang nhập hoặc đã khoá)
- `done` (Đã Hoàn Thành): hiển thị label **"SL Thực Nhận"** (cửa hàng đã chốt — có thể bị ghi đè so với giá trị vendor nhập nếu hàng hư hỏng)

Không cần snapshot, không cần bảng phụ — Portal chỉ đọc 1 field từ Odoo và đổi label theo state.

**Lưu ý:** Bản update 2026-05-24 trước đó (em ghi 2 field `product_uom_qty` + `quantity_done`) là **SAI** — đã revert về cơ chế 1 field gốc.

### ~~Blocker 4~~ — Timestamp khoá (`locked_at`) — **RESOLVED**

**Kết luận (2026-04-11):** Timestamp là deterministic — không cần lưu. Banner tính:

```python
cut_at = datetime.combine(picking_scheduled_date - timedelta(days=1),
                          time(23, 0), tzinfo=ZoneInfo("Asia/Ho_Chi_Minh"))
```

### ~~Blocker 5~~ — Phân trang & tìm kiếm danh sách DO — **RESOLVED**

**Kết luận (2026-04-11):** Domain Odoo đã giới hạn `partner_id` + 3 tháng → result set nhỏ, `search_read` với `limit/offset` là đủ. Pattern `list_purchase_orders()` trong `VendorScopedOdooClient` làm mẫu. `picking_type_code` phải bao gồm cả `outgoing` (phiếu trả hàng):

```python
domain = [
    ("partner_id", "=", self.partner_id),
    ("picking_type_code", "in", ["incoming", "outgoing"]),
    ("state", "in", ["assigned", "done"]),
    ("scheduled_date", ">=", three_months_ago),
]
```

### ~~Blocker 6~~ — Admin unlock — **RESOLVED (out of scope)**

**Quyết định (2026-04-11):** Không có tính năng admin unlock trong portal. Nếu cần mở khoá, admin cập nhật `scheduled_date` trực tiếp trên Odoo → lock condition tự được giải phóng.

---

### Ghi chú — Quyền vendor trên DO

**Luồng nghiệp vụ:**
- Sau khi PO được xác nhận, Odoo tự động tạo một picking ở trạng thái `assigned` (= `ready` trên portal).
- Vendor được phép: chỉnh `qty_done` (delivery_qty) và thêm note trên từng dòng.
- Vendor **không được phép** xác nhận/validate DO — việc này do cửa hàng thực hiện trực tiếp trên Odoo sau khi nhận hàng thực tế.

**Guard khi live query** — thay thế `do.status not in ("waiting", "ready")`:

```python
_TZ = ZoneInfo("Asia/Ho_Chi_Minh")
today = datetime.now(_TZ).date()

scheduled = picking["scheduled_date"]  # date object (UTC+7)
is_locked = scheduled <= today
is_editable = picking["state"] in ("confirmed", "waiting", "assigned")

if is_locked or not is_editable:
    raise HTTPException(423, ...)
```

### Ghi chú — Mapping state khi live query

Khi live query, không còn cột `status` local. Status được derive tại thời điểm đọc:

```python
_TZ = ZoneInfo("Asia/Ho_Chi_Minh")

def _derive_status(picking: dict) -> str:
    today = datetime.now(_TZ).date()
    if picking["scheduled_date"] <= today:
        return "locked"
    return {
        "draft":     "waiting",
        "confirmed": "waiting",
        "waiting":   "waiting",
        "assigned":  "ready",
        "done":      "done",
        "cancel":    "cancelled",
    }.get(picking["state"], "waiting")
```

Hàm này dùng nhất quán ở cả list endpoint và detail endpoint.

---

### Việc đã làm (2026-04-11)

- [x] ~~Blocker 1~~ Hướng C: khoá UI-only từ `scheduled_date`, không cần job/bảng
- [x] ~~Blocker 2~~ Không phải blocker thực sự (< 10 dòng/PO, asyncio.gather)
- [x] ~~Blocker 3~~ received_qty: chỉ dùng qty_done từ Odoo, đổi label theo state
- [x] ~~Blocker 4~~ Timestamp deterministic từ `scheduled_date`
- [x] ~~Blocker 5~~ Result set nhỏ, dùng pattern list_purchase_orders
- [x] ~~Blocker 6~~ Out of scope — admin dùng Odoo trực tiếp
- [x] Xoá `do_sync.py` + `do_cutoff.py` jobs; cập nhật `scheduler.py`
- [x] Xoá bảng `delivery_orders`, `delivery_order_lines` — migration `d0e1f2a3b4c5`
- [x] Xoá model `delivery_order.py`; cập nhật `models/__init__.py`, `vendor_user.py`
- [x] Viết `list_delivery_orders()`, `get_delivery_order()`, `write_picking_data()` + helpers (`_derive_do_status`, `_is_do_locked`, `_locked_at_str`) trong `VendorScopedOdooClient`
- [x] Viết lại `delivery_orders.py` API — list/detail/patch đọc/ghi live Odoo
- [x] Cập nhật `webhooks.py` — `receipt_validated` không còn đọc local DO
- [x] Cập nhật `event_consumer.py` — `picking_assigned` handler là no-op (live query mode)
- [x] Cập nhật `admin.py` — xoá DO unlock endpoint, DO count queries; `list_vendor_delivery_orders` dùng Odoo live
- [x] Cập nhật `purchase_orders.py` — xoá DO batch query từ local DB; `do` field trả về `None`
- [x] Cập nhật `main.py` — xoá import `do_sync`
- [x] Cập nhật tests — xoá fixtures và tests gắn với local DO model

### Còn lại / Out of scope

- PDF generation cho DO — vẫn dùng Odoo data nhưng chưa implement lại (nếu có)

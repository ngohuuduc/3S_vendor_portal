# Vendor Portal — Lộ Trình Triển Khai

> Các phase kỹ thuật, DB schema, API endpoints, và các lưu ý cho developer.
> Xem business logic và quy trình tại [README.md](README.md) và [PROCESS_FLOW.md](PROCESS_FLOW.md).

---

## Quyết Định Stack Kỹ Thuật

| Vấn đề | Quyết định |
|---|---|
| Frontend stack | React + Vite |
| Backend stack | FastAPI (Python) |
| Cơ sở dữ liệu | PostgreSQL (thuộc portal) |
| Cache / rate limiting | Redis |
| Triển khai | Docker Compose trên VM riêng biệt, Nginx reverse proxy |
| SSL / TLS | Xử lý ở tầng infrastructure (load balancer / reverse proxy) |
| HTTP client (frontend) | Native `fetch` API — không dùng axios hay thư viện HTTP bên ngoài |
| Capture chữ ký | `signature_pad.js` → PNG gửi lên backend |
| Audit logging (DB) | Các hành động được ghi: `login`, `po_confirm`, `po_reject`, `po_auto_cancel`, `do_update`, `do_sign`, `do_unlock`, `rn_confirm_sign`, `receipt_validated` |
| Vai trò Admin | Bảng `admin_users` riêng biệt, mật khẩu được seed từ biến môi trường `ADMIN_INITIAL_PASSWORD` khi khởi động lần đầu |

---

## Odoo ↔ Portal Sync — Các Quyết Định Đã Xác Nhận

> Tập hợp các điểm kỹ thuật đã được xác nhận với IT team.

| # | Chủ đề | Mô tả hiện tại | Cần xác nhận |
|---|---|---|---|
| 1 | **Order Deadline / auto-cancel** | PO auto-cancel sau 7 ngày past Expected Arrival nếu vendor không action | ✅ Trường `date_planned` trên `purchase.order` |
| 2 | **Vendor profile sync** | Sync job chạy mỗi 6h, đọc `res.partner` từ Odoo | ✅ Batch job mỗi 6h — đã documented |
| 3 | **PO data sync** | Portal đọc PO trực tiếp từ Odoo qua XML-RPC on-demand | ✅ Có cache, refresh mỗi 10 phút |
| 4 | **PO confirmation → Odoo** | Portal gọi `button_confirm` trên `purchase.order` | ✅ Không có validation thêm phía Odoo |
| 5 | **PO rejection → Odoo** | Portal gọi `button_cancel` trên `purchase.order` | ✅ `button_cancel` hoạt động ở state `sent` trên Odoo 16 CE |
| 6 | **DO data** | Odoo tự tạo DO (`stock.picking`) khi PO confirmed | ✅ Odoo tạo — portal đọc qua XML-RPC, không tự tạo |
| 7 | **23:00 Cutoff → Push Odoo** | Lúc 23:00 đêm trước ngày giao, DO bị Locked và push lên Odoo | ✅ `scheduled_date` trên `stock.picking` + `quantity_done` trên `stock.move.line`. Ký không push — chỉ 23:00 mới push |
| 8 | **Receipt validation (Odoo → Portal)** | Cửa hàng xác nhận Receipt → Portal cập nhật DO status = Done | ✅ Nightly batch sync (23:00) — không cần Odoo module. Nếu DO vẫn Draft → tự động chuyển Done |
| 9 | **Vendor deactivation** | Vendor deactivated trên Odoo → sync set `is_active = FALSE` | ✅ Chờ sync cycle 6h — không critical |

**Cơ chế sync (confirmed):**
- **Portal → Odoo (PO):** XML-RPC write real-time — xác nhận/từ chối PO
- **Portal → Odoo (DO):** 23:00 cutoff job — push `scheduled_date` + `quantity_done` lên Odoo Receipt. Không push khi vendor ký
- **Odoo → Portal (hồ sơ):** Scheduled batch job mỗi 6h
- **Odoo → Portal (dữ liệu PO):** XML-RPC on-demand, có cache, refresh mỗi 10 phút
- **Odoo → Portal (receipt validated):** Nightly batch sync lúc 23:00 — không cần Odoo module
- **DO PDF:** Tạo on-the-fly mỗi lần vendor in — không lưu trên server

---

## Tóm Tắt Luồng Dữ Liệu

### Luồng xác thực
1. Nhà cung cấp nhập **Vendor ID** (số nguyên — `res.partner.id` từ Odoo) và mật khẩu trên trang đăng nhập
2. Backend tra cứu `vendor_users` theo `odoo_partner_id`
3. Mật khẩu được xác minh bằng bcrypt — cùng một thông báo lỗi cho mọi trường hợp thất bại
4. Thành công, cấp phát JWT access token (30 phút) và refresh token (7 ngày)
5. Tất cả request tiếp theo mang access token trong header `Authorization`
6. Access token hết hạn được tự động làm mới bằng refresh token
7. Khi refresh thất bại, nhà cung cấp được chuyển hướng về trang đăng nhập

### Luồng đồng bộ hồ sơ
1. Scheduled job chạy mỗi 6 giờ, đọc `res.partner` nơi `is_vendor = True` và có `email`
2. **Partner mới:** tạo dòng `vendor_users` (không có mật khẩu, chưa kích hoạt) liên kết qua `odoo_partner_id`. Tạo invite token 24h, gửi email chào mừng qua AWS SES chứa **Vendor ID** và link đặt mật khẩu
3. **Partner đã tồn tại:** chỉ đồng bộ các trường hồ sơ (`full_name`, `company_name`, `phone`, `tax_id`) — `hashed_password` **không bao giờ bị ghi đè** bởi sync
4. **Partner bị vô hiệu hoá trong Odoo:** đặt `is_active = FALSE` — nhà cung cấp không thể đăng nhập nữa
5. Partner không có email trong Odoo bị bỏ qua và ghi log để xử lý thủ công (cần email để gửi email chào mừng)

### Luồng Delivery Order (DO)
1. Khi nhà cung cấp xác nhận PO, Odoo tự động tạo 1 DO (`stock.picking`). Portal đọc qua XML-RPC, tạo bản ghi `delivery_orders` trong DB nội bộ, pre-fill qty từ PO
2. Nhà cung cấp chỉnh sửa DO: đặt ngày giao hàng và số lượng từng dòng sản phẩm (qty phải <= qty đặt hàng từ PO)
3. Nhà cung cấp có thể **lưu nhiều lần, ký nhiều lần, in nhiều phiên bản** — tất cả khi DO ở trạng thái Draft
4. Ký = vẽ chữ ký điện tử + ghi chú tuỳ chọn → backend tạo PDF tức thời (on-the-fly, WeasyPrint) để tải/in. **Ký không khoá DO, không push Odoo, không lưu PDF trên server**
5. Mỗi lần in, PDF được tạo mới với dữ liệu mới nhất — vendor luôn có phiên bản cập nhật
6. **Lúc 23:00 đêm trước ngày giao hàng:** scheduled job tự động chuyển DO status → **Locked**
7. Job push lên Odoo qua XML-RPC: `scheduled_date` → `stock.picking` + `quantity_done` → `stock.move.line` (pre-fill để cửa hàng không phải nhập liệu)
8. DO bị khoá vĩnh viễn — vendor không thể sửa hay ký lại. Chỉ portal admin mới có thể mở khoá (Locked → Draft, vendor được thông báo qua email)

### Luồng xác nhận biên nhận (phía cửa hàng, phản ánh trên portal)
1. Nhà cung cấp giao hàng đến cửa hàng với DO in ra (2 bản giấy, cả hai bên ký, mỗi bên giữ 1 bản)
2. Cửa hàng xem xét Receipt trong Odoo — qty đã được pre-fill từ DO push. Cửa hàng có thể điều chỉnh nếu hàng hư hỏng
3. Cửa hàng xác nhận Receipt trong Odoo — `qty_done` được finalize, stock move được tạo
4. Nightly batch sync lúc 23:00 — portal phát hiện Receipt đã confirm, cập nhật DO status → **Done**, lưu `received_qty` từ Odoo
5. Nếu DO vẫn ở Draft (vendor chưa dùng portal) — portal tự động chuyển → Done
6. Portal gửi email đến nhà cung cấp xác nhận biên nhận, cảnh báo nếu có số lượng chênh lệch giữa DO và biên nhận
7. Nhà cung cấp có thể thấy cả số lượng DO của mình và số lượng thực tế cửa hàng xác nhận trên trang chi tiết DO
8. Nhà cung cấp có thể xuất dữ liệu dưới dạng PDF hoặc CSV để lập hoá đơn và đối soát

### Cơ chế sync (Odoo → Portal) — confirmed: Nightly batch
- **Nightly batch sync lúc 23:00** — portal đọc tất cả Receipt đã được confirm trong ngày qua XML-RPC
- Không cần Odoo module, chỉ cần quyền XML-RPC read trên `stock.picking`
- Cùng job 23:00 cũng chạy: DO cutoff (Draft → Locked + push Odoo) và PO auto-cancel (7 ngày)
- Portal → Odoo communication (xác nhận/từ chối PO, đẩy DO) là real-time via XML-RPC

---

## Phase 0 — Thiết Lập Dự Án & Hạ Tầng
**Ước tính: 0.5–1 ngày**

### Mục tiêu
- Thiết lập cấu trúc monorepo với các thư mục `/frontend`, `/backend`, `/infra`
- Cấu hình Docker Compose với năm service: frontend, backend, PostgreSQL, Redis, Nginx
- Thiết lập chiến lược biến môi trường và quản lý secrets
- Thiết lập kết nối mạng giữa portal VM và Odoo VM
- Cấu hình AWS SES: xác minh sender domain, tạo IAM user với quyền gửi SES, lưu credentials dưới dạng secrets

### Cấu trúc repo
```
vendor-portal/
├── frontend/               # React + Vite
│   ├── src/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api/            # fetch wrapper + TanStack Query hooks
│   │   └── i18n/           # File dịch Tiếng Việt + Tiếng Anh
│   └── Dockerfile
├── backend/                # FastAPI
│   ├── app/
│   │   ├── api/            # route handlers
│   │   ├── services/       # odoo_client, pdf_service, email_service
│   │   ├── models/         # SQLAlchemy models
│   │   ├── schemas/        # Pydantic schemas
│   │   ├── jobs/           # sync_vendors, scheduler
│   │   └── core/           # config, security, dependencies
│   ├── alembic/            # DB migrations
│   └── Dockerfile
├── infra/
│   ├── docker-compose.yml
│   ├── docker-compose.prod.yml
│   ├── nginx/
│   │   └── nginx.conf
│   └── secrets/            # gitignored
├── .env.example
└── README.md
```

### Các quyết định chính
- **Monorepo** giữ frontend và backend versioning cùng nhau, đơn giản hoá triển khai
- **Docker Compose** với config dev và prod riêng biệt — prod dùng Docker secrets cho các giá trị nhạy cảm
- **Thiết lập AWS SES:** xác minh sending domain trong SES, tạo IAM user chuyên dụng chỉ có quyền `ses:SendEmail`, lưu `AWS_ACCESS_KEY_ID` và `AWS_SECRET_ACCESS_KEY` dưới dạng Docker secrets. Dùng `boto3` trong email service của FastAPI.
- **Kết nối Odoo:** IP của portal VM phải được whitelist trên firewall của Odoo VM ở cổng 8069. Một Odoo service account chuyên dụng với API key là credential duy nhất mà portal sử dụng.

### Biến môi trường cần thiết
- `ODOO_URL`, `ODOO_DB`, `ODOO_USERNAME`, `ODOO_API_KEY`
- `JWT_SECRET_KEY`, `JWT_REFRESH_SECRET`
- `DB_PASSWORD`, `REDIS_URL`
- `FRONTEND_URL` (cho CORS và link email)
- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SES_REGION`, `SES_SENDER_EMAIL`
- ~~`INTERNAL_NOTIFICATION_EMAIL`~~ — đã bỏ: thông báo cửa hàng gửi đến email của PO creator từ Odoo, không phải hộp thư chung
- `ADMIN_INITIAL_PASSWORD` (dùng để seed tài khoản admin đầu tiên khi khởi động lần đầu)

---

## Phase 1 — Tầng Tích Hợp Odoo
**Ước tính: 1–2 ngày**

### Mục tiêu
- Xây dựng Odoo XML-RPC client dưới dạng singleton service
- Triển khai vendor-scoped query wrapper (phân lập theo hàng)
- Triển khai và lên lịch job đồng bộ partner
- Xác minh tất cả các query Odoo cần thiết hoạt động đúng với instance thực

### Các quyết định thiết kế chính
- **Singleton OdooClient:** xác thực một lần khi khởi động, tái sử dụng `uid` cho tất cả các lời gọi. Tự động kết nối lại khi bị ngắt kết nối
- **VendorScopedOdooClient:** một wrapper inject `[['partner_id', '=', partner_id]]` vào mọi domain filter ở tầng service — không có route handler nào có thể vô tình bỏ qua vendor scoping
- **Lên lịch sync job:** APScheduler chạy trong tiến trình FastAPI, mỗi 6 giờ cho hồ sơ vendor. Có thể kích hoạt thủ công qua internal admin endpoint
- **Odoo → Portal sync (TBD):** Cơ chế để portal biết khi cửa hàng xác nhận Receipt đang chờ xác nhận từ team. Các lựa chọn: webhook (cần Odoo module), polling, hoặc nightly batch. Dù cơ chế nào, portal sẽ đọc `qty_done` đã xác nhận qua XML-RPC và cập nhật trạng thái DO thành Done
- **PDF session:** một `requests.Session()` riêng biệt được xác thực qua `/web/session/authenticate` được duy trì để tải PDF — `uid` XML-RPC không thể truy cập các endpoint `/report/pdf/`

### Các model Odoo trong phạm vi
| Model | Mục đích | Các trường chính |
|---|---|---|
| `res.partner` | Đồng bộ hồ sơ vendor | `id`, `name`, `email` (bắt buộc — sync bỏ qua partner không có email), `company_name`, `phone`, `mobile`, `vat` (Mã số thuế), `is_vendor` (bộ lọc: chỉ sync nơi `is_vendor = True` và có `email`) |
| `purchase.order` | Danh sách PO, chi tiết, xác nhận, từ chối | `name`, `partner_id`, `state`, `date_order`, `date_planned` (Expected Arrival), `amount_total`, `order_line`, `picking_ids`, `user_id` (PO creator để gửi email) — gọi `button_confirm` hoặc `button_cancel` khi vendor hành động |
| `purchase.order.line` | Các dòng sản phẩm PO | `product_id`, `name`, `product_qty`, `qty_received`, `price_unit`, `product_uom` (UoM của dòng này) |
| `product.product` | Thông tin sản phẩm cho DO PDF | `id`, `name`, `barcode` |
| `stock.picking` | Header Receipt | `name`, `state`, `scheduled_date`, `origin`, `move_line_ids` |
| `stock.move.line` | Các dòng Receipt | `product_id`, `product_uom_qty`, `qty_done`, `lot_id`, `state` |
| `stock.warehouse` | Mã cửa hàng cho DO PDF | `code` (tên ngắn, dùng làm Mã cửa hàng trên DO in) |

### Lưu ý tên trường Odoo 16 CE
- `stock.move.line.qty_done` → đổi tên thành `quantity` trong Odoo 17+
- `stock.picking.move_lines` → đổi tên thành `move_ids` trong Odoo 17+
- Luôn chỉ định danh sách trường tường minh trong `search_read` — đọc tất cả trường trên Odoo 16 có thể gây lỗi serialization trên các computed field như `tax_totals`

### Quy tắc lọc dữ liệu (áp dụng ở backend, không phải frontend)
- PO: chỉ trả về `state IN ('sent', 'purchase', 'cancel')` — `draft` và `done` bị loại
- Receipt: chỉ trả về `state IN ('assigned', 'done')`
- Tất cả các query đều được scoped theo `partner_id` qua VendorScopedOdooClient

---

## Phase 2 — Hệ Thống Xác Thực
**Ước tính: 1 ngày**

### Mục tiêu
- Định nghĩa các bảng DB: `vendor_users`, `admin_users`, `password_reset_tokens`, `delivery_orders`, `delivery_order_lines`, `return_notes`, `return_note_lines`, và `audit_log`
- Triển khai JWT access + refresh token cấp phát và xác minh, với role claim (`vendor` hoặc `admin`)
- Triển khai luồng mời vendor (đặt mật khẩu lần đầu qua token từ email AWS SES)
- Triển khai luồng quên mật khẩu cho vendor (cùng cơ chế token)
- Triển khai seed tài khoản admin từ biến môi trường `ADMIN_INITIAL_PASSWORD` khi khởi động lần đầu
- Triển khai FastAPI auth dependency dùng cho tất cả route được bảo vệ, với role-based access control

### Các bảng cơ sở dữ liệu

**`admin_users`**
| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | serial PK | ID portal nội bộ |
| `username` | varchar, unique | định danh đăng nhập cho admin (không phải Odoo ID) |
| `hashed_password` | varchar | được seed từ `ADMIN_INITIAL_PASSWORD` khi khởi động lần đầu |
| `full_name` | varchar | đặt thủ công |
| `email` | varchar | để nhận thông báo và đặt lại mật khẩu |
| `is_active` | boolean | TRUE mặc định |
| `created_at` | timestamp | |
| `last_login` | timestamp | |

**Seed tài khoản Admin:** Khi ứng dụng khởi động lần đầu, nếu không có dòng nào trong `admin_users`, backend chèn một tài khoản admin mặc định sử dụng `ADMIN_INITIAL_PASSWORD` từ môi trường. Admin phải đổi mật khẩu này sau lần đăng nhập đầu tiên.

**`vendor_users`**
| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | serial PK | ID portal nội bộ |
| `odoo_partner_id` | integer, unique | **định danh đăng nhập** — `res.partner.id` từ Odoo, cố định, không bao giờ thay đổi |
| `email` | varchar | chỉ để gửi email thông báo — được sync từ `res.partner.email` của Odoo khi tạo tài khoản, không dùng để đăng nhập |
| `hashed_password` | varchar | NULL cho đến khi chấp nhận lời mời |
| `full_name` | varchar | được sync từ Odoo — tên người liên hệ |
| `company_name` | varchar | được sync từ Odoo |
| `phone` | varchar | được sync từ Odoo — số điện thoại di động/cố định |
| `tax_id` | varchar | được sync từ `res.partner.vat` của Odoo — dùng cho DO PDF |
| `preferred_language` | varchar | `'vi'` hoặc `'en'`, do vendor đặt, mặc định `'vi'` |
| `is_active` | boolean | FALSE cho đến khi đặt mật khẩu lần đầu |
| `created_at` | timestamp | |
| `last_login` | timestamp | cập nhật mỗi lần đăng nhập thành công |

**`password_reset_tokens`**
| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | serial PK | |
| `user_id` | FK → vendor_users | |
| `token` | varchar, unique | `secrets.token_urlsafe(32)` |
| `expires_at` | timestamp | 24 giờ kể từ khi tạo |
| `used` | boolean | đánh dấu TRUE sau khi dùng |

**`delivery_orders`**
| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | serial PK | ID DO nội bộ của portal |
| `po_odoo_id` | integer, unique | Odoo `purchase.order.id` mà DO này thuộc về |
| `picking_id` | integer | Odoo `stock.picking.id` (Receipt liên kết) |
| `vendor_id` | FK → vendor_users | vendor sở hữu DO này |
| `delivery_date` | date | ngày giao hàng theo kế hoạch do vendor đặt |
| `status` | varchar | `draft`, `locked`, `done`, hoặc `cancelled` |
| `latest_signature` | text | base64 PNG chữ ký mới nhất (ghi đè mỗi lần ký — dùng để render PDF on-the-fly) |
| `vendor_comment` | text | ghi chú tự do tuỳ chọn — cập nhật mỗi lần vendor ký |
| `locked_at` | timestamp | NULL cho đến khi 23:00 cutoff khoá DO |
| `pushed_at` | timestamp | NULL cho đến khi push lên Odoo (cùng lúc locked_at) |
| `unlocked_at` | timestamp | NULL trừ khi admin đã mở khoá |
| `unlocked_by` | FK → admin_users | admin đã mở khoá, NULL nếu chưa mở khoá |
| `created_at` | timestamp | thời điểm DO được tạo tự động (khi PO xác nhận) |
| `updated_at` | timestamp | lần chỉnh sửa cuối của vendor |

> **Lưu ý:** Không có `pdf_path` — PDF tạo on-the-fly mỗi lần vendor in (WeasyPrint). Không có `signed_at` — ký chỉ là action, vendor có thể ký lại bất kỳ lúc nào khi Draft.

**`delivery_order_lines`**
| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | serial PK | |
| `do_id` | FK → delivery_orders | DO cha |
| `line_number` | integer | số thứ tự dòng |
| `product_id` | integer | Odoo `product.product.id` |
| `product_barcode` | varchar | barcode sản phẩm được cache từ Odoo |
| `product_name` | varchar | tên sản phẩm được cache |
| `uom` | varchar | đơn vị tính, kế thừa từ dòng PO (ví dụ: Thùng 12 Chai, Kg). Không thể thay đổi bởi vendor |
| `ordered_qty` | decimal | số lượng từ dòng PO (tham chiếu chỉ đọc) |
| `delivery_qty` | decimal | số lượng giao do vendor nhập (phải <= ordered_qty) |
| `received_qty` | decimal | NULL cho đến khi cửa hàng xác nhận biên nhận; qty_done cuối cùng từ Odoo |

**`return_notes`** (cùng cấu trúc với delivery_orders, dành cho trả hàng — **KHÔNG dùng 23:00 cutoff**; hạn chót = `pickup_date` + 7 ngày, sau đó **Odoo tự xác nhận trả hàng** qua scheduled action phía Odoo, portal chỉ đọc và reflect)
| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | serial PK | ID RN nội bộ của portal |
| `rpo_odoo_id` | integer, unique | Odoo `purchase.order.id` (RPO — PO với qty âm) |
| `picking_id` | integer | Odoo `stock.picking.id` (outgoing shipment trả hàng) |
| `vendor_id` | FK → vendor_users | vendor sở hữu RN này |
| `pickup_date` | date | ngày vendor đặt để lấy hàng trả về |
| `status` | varchar | `draft` hoặc `done` (không có `locked` và không có `cancelled` — vendor không thể từ chối RPO). Portal derive `done` khi Odoo `stock.picking.state = done`, bất kể xác nhận do cửa hàng thao tác tay hay do Odoo tự động validate sau hạn chót (`pickup_date` + 7 ngày) |
| `latest_signature` | text | base64 PNG chữ ký mới nhất (dùng để render PDF on-the-fly) |
| `locked_at` | timestamp | NULL cho đến khi 23:00 cutoff |
| `created_at` | timestamp | |
| `updated_at` | timestamp | |

**`return_note_lines`**
| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | serial PK | |
| `rn_id` | FK → return_notes | RN cha |
| `line_number` | integer | số thứ tự dòng |
| `product_id` | integer | Odoo `product.product.id` |
| `product_barcode` | varchar | barcode sản phẩm được cache |
| `product_name` | varchar | tên sản phẩm được cache |
| `uom` | varchar | đơn vị tính từ dòng RPO |
| `return_qty` | decimal | số lượng đang trả (chỉ đọc — do cửa hàng đặt, vendor không thể thay đổi; pre-fill mặc định = số lượng đặt, đúng decimal UoM: kg/lít/mét → 2 chữ số thập phân, các UoM khác (gram, thùng, chai, cái, hộp, units) → số nguyên) |

**`audit_log`**
| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | serial PK | |
| `actor_type` | varchar | `vendor` hoặc `admin` |
| `actor_id` | integer | `vendor_users.id` hoặc `admin_users.id` |
| `action` | varchar | một trong: `login`, `po_confirm`, `po_reject`, `po_auto_cancel`, `do_update`, `do_sign`, `do_unlock`, `rn_confirm_sign`, `receipt_validated` |
| `target_model` | varchar | ví dụ: `receipt`, `vendor` |
| `target_id` | integer | ví dụ: `picking_id`, `partner_id` |
| `detail` | jsonb | metadata theo từng hành động (ví dụ: dòng nào được cập nhật, qty cũ/mới) |
| `created_at` | timestamp | thời điểm hành động xảy ra |
| `ip_address` | varchar | IP client cho các sự kiện đăng nhập |

### Các endpoint xác thực
| Method | Endpoint | Mô tả |
|---|---|---|
| POST | `/api/auth/login` | Vendor ID (số nguyên) + mật khẩu → access + refresh token (role: `vendor`) |
| POST | `/api/admin/auth/login` | Username + mật khẩu → access + refresh token (role: `admin`) |
| POST | `/api/auth/refresh` | Refresh token → access token mới (giữ nguyên role) |
| POST | `/api/auth/set-password` | Đặt mật khẩu lần đầu khi được mời hoặc đặt lại qua token (chỉ cho vendor) |
| POST | `/api/auth/forgot-password` | Gửi email đặt lại qua AWS SES theo Vendor ID |
| POST | `/api/auth/logout` | Đưa refresh token vào blacklist trong Redis |

### JWT role claim
JWT access token mang trường `role` (`vendor` hoặc `admin`). Tất cả endpoint chỉ dành cho admin kiểm tra `role == 'admin'` qua một FastAPI dependency chuyên dụng. Endpoint vendor kiểm tra `role == 'vendor'`. Token vendor không thể truy cập route admin, và token admin không thể dùng để giả mạo vendor.

### Các lưu ý bảo mật
- **Đăng nhập an toàn về timing:** luôn chạy `bcrypt.verify` kể cả khi Vendor ID không tồn tại
- **Thông báo lỗi nhất quán:** cùng một thông báo cho Vendor ID không tồn tại và mật khẩu sai — không tiết lộ ID có tồn tại hay không
- **Token blacklist:** khi đăng xuất, refresh token được thêm vào Redis set với TTL khớp thời gian còn lại của token
- **Hết hạn invite token:** 24 giờ. Nếu hết hạn, vendor phải yêu cầu token mới qua quên mật khẩu

---

## Phase 3 — Các API Endpoint Cốt Lõi
**Ước tính: 2–3 ngày**

### Mục tiêu
- Triển khai tất cả endpoint dữ liệu với vendor-scoped Odoo queries
- Thực thi logic khoá DO phía server
- Triển khai pipeline tạo và ký PDF
- Triển khai webhook receiver cho xác nhận biên nhận Odoo
- Triển khai tất cả trigger thông báo email qua AWS SES

### Danh sách đầy đủ endpoint
| Method | Endpoint | Mô tả |
|---|---|---|
| GET | `/api/vendors/me` | Hồ sơ vendor hiện tại |
| PATCH | `/api/vendors/me/language` | Cập nhật ngôn ngữ ưa thích (`vi` hoặc `en`) |
| GET | `/api/purchase-orders` | Danh sách PO có phân trang với trạng thái portal, có thể lọc theo số PO và khoảng ngày |
| GET | `/api/purchase-orders/{id}` | Chi tiết PO với các dòng đặt hàng + DO liên kết + so sánh biên nhận |
| POST | `/api/purchase-orders/{id}/confirm` | Xác nhận Sent RFQ — gọi `button_confirm` trên Odoo; trả về 409 nếu PO không ở trạng thái `sent` |
| POST | `/api/purchase-orders/{id}/reject` | Từ chối Sent RFQ — gọi `button_cancel` trên Odoo + email đến PO creator; trả về 409 nếu PO không ở trạng thái `sent` |
| GET | `/api/delivery-orders/{do_id}` | Chi tiết DO với các dòng sản phẩm, ngày giao hàng, qty giao, và qty nhận (khi Done) |
| PATCH | `/api/delivery-orders/{do_id}/lines` | Cập nhật số lượng + ngày giao hàng trên DO (bị chặn nếu đã ký/khoá); qty phải <= qty đặt hàng |
| POST | `/api/delivery-orders/{do_id}/sign` | Gửi PNG chữ ký + ghi chú tuỳ chọn. Lưu signature vào DB, trả về PDF on-the-fly. **Không khoá DO, không push Odoo** |
| GET | `/api/delivery-orders/{do_id}/pdf` | Tạo và tải PDF on-the-fly (phiên bản mới nhất). Có thể gọi nhiều lần |
| GET | `/api/return-orders` | Danh sách RPO có phân trang cho vendor hiện tại |
| GET | `/api/return-orders/{id}` | Chi tiết RPO với các dòng trả hàng và Return Note liên kết |
| GET | `/api/return-notes/{rn_id}` | Chi tiết Return Note với các dòng sản phẩm và ngày lấy hàng |
| PATCH | `/api/return-notes/{rn_id}/pickup-date` | Đặt hoặc cập nhật ngày lấy hàng trên RN (bị chặn nếu đã ký) |
| POST | `/api/return-notes/{rn_id}/confirm-and-sign` | Xác nhận RPO + gửi PNG chữ ký trong một bước. Khoá RN, tạo PDF |
| GET | `/api/return-notes/{rn_id}/pdf` | Tải xuống RN PDF đã ký (cùng định dạng với DO PDF) |
| GET | `/api/export` | Xuất dữ liệu dưới dạng PDF hoặc CSV. Query params: `format` (pdf/csv), `type` (individual/summary), `ids` (bulk), `date_from`, `date_to`. Bao gồm PO, DO, RPO, RN |
| POST | `/api/webhooks/odoo/receipt-validated` | Webhook receiver: Odoo thông báo portal khi Receipt được validated — kích hoạt DO status → Done + email vendor |

### Endpoint chỉ dành cho Admin
| Method | Endpoint | Mô tả |
|---|---|---|
| GET | `/api/admin/vendors` | Liệt kê tất cả tài khoản vendor với trạng thái, lần đăng nhập cuối, số DO đã ký, số DO đang chờ |
| GET | `/api/admin/vendors/{partner_id}` | Xem chi tiết một vendor cụ thể — PO và DO của họ (chỉ đọc) |
| PATCH | `/api/admin/vendors/{partner_id}/deactivate` | Vô hiệu hoá tài khoản vendor |
| PATCH | `/api/admin/vendors/{partner_id}/reactivate` | Kích hoạt lại tài khoản vendor |
| POST | `/api/admin/sync` | Kích hoạt thủ công job đồng bộ partner Odoo |
| GET | `/api/admin/sync/status` | Trả về thời gian sync cuối, số vendor đã sync, danh sách vendor bị bỏ qua (không có email) |
| GET | `/api/admin/delivery-orders/{do_id}/pdf` | Tải xuống DO PDF đã ký của bất kỳ vendor nào |
| POST | `/api/admin/delivery-orders/{do_id}/unlock` | Gỡ bỏ khoá DO — gửi email đến vendor, vendor có thể cập nhật và ký lại |
| GET | `/api/admin/audit-log` | Xem audit log có phân trang, có thể lọc theo actor, loại hành động, và khoảng ngày |

Tất cả endpoint admin yêu cầu `role == 'admin'` trong JWT. Chúng có prefix `/api/admin/` để phân biệt rõ ràng và áp dụng rate limiting policy riêng.

### Lọc danh sách PO
Endpoint `GET /api/purchase-orders` chấp nhận các query parameter tuỳ chọn sau:
- `q` — tìm kiếm một phần số PO (ví dụ: `PO004`)
- `date_from` — lọc PO có `date_order >= date_from`
- `date_to` — lọc PO có `date_order <= date_to`
- `page` và `page_size` — phân trang (page size mặc định: 20)

Lọc được áp dụng phía server trong Odoo domain filter, không phải ở frontend.

### Cách tiếp cận khoá DO — 23:00 Cutoff
- **Ký (Draft):** `POST /api/delivery-orders/{do_id}/sign` lưu `latest_signature` + `vendor_comment` trên `delivery_orders`, trả về PDF on-the-fly. **Status vẫn là Draft** — vendor có thể tiếp tục sửa/ký/in
- **23:00 Cutoff:** Scheduled job chuyển status → `locked`, đặt `locked_at` + `pushed_at`, push `scheduled_date` + `quantity_done` lên Odoo Receipt
- **Locked:** `PATCH /lines` trả về HTTP 423. Vendor chỉ có thể in PDF phiên bản cuối
- **Mở khoá:** portal admin gọi `POST /api/admin/delivery-orders/{do_id}/unlock`, đặt status → `draft`, đặt `unlocked_at` + `unlocked_by`, gửi email vendor. Vendor có thể sửa/ký/in lại

### Cách tiếp cận validation (không có `button_validate` phía portal)
Vì cửa hàng quyết định về backorder trong Odoo, portal **không** gọi `button_validate`. Vai trò của vendor là:
1. Chỉnh sửa DO: ngày giao hàng + số lượng (qty <= qty đặt hàng, lưu/ký/in nhiều lần)
2. 23:00 cutoff khoá DO và push qty lên Odoo Receipt

Odoo picking vẫn ở trạng thái `assigned` sau khi push — qty đã được pre-fill nhưng cửa hàng vẫn có thể điều chỉnh trước khi confirm. Chỉ khi cửa hàng xác nhận Receipt thì `qty_done` mới được finalize và stock move được tạo.

### Receipt validated (Odoo → Portal) — Nightly batch sync
Khi cửa hàng xác nhận Receipt trong Odoo, nightly batch sync lúc 23:00 phát hiện:
1. Portal nhận thông báo với `picking_id`
2. Portal đọc các giá trị `qty_done` đã xác nhận từ Odoo qua XML-RPC
3. Trạng thái DO trên portal chuyển thành **Done** — số lượng thực tế nhận được được lưu trên các dòng DO
4. Portal gửi email đến vendor xác nhận biên nhận. Nếu có số lượng chênh lệch giữa DO và biên nhận, email bao gồm cảnh báo
5. Vendor có thể đăng nhập portal để xem chi tiết và xuất PDF/CSV

### Cách tiếp cận audit logging
Mỗi hành động chính được ghi vào bảng `audit_log` đồng bộ trong cùng request. Các loại hành động được ghi:

| Hành động | Trigger | Chi tiết được lưu |
|---|---|---|
| `login` | Vendor hoặc admin đăng nhập thành công | loại actor, actor ID, IP address, timestamp |
| `po_confirm` | `POST /purchase-orders/:id/confirm` | PO ID, tên PO, trạng thái trước (`sent`), trạng thái sau (`purchase`) |
| `po_reject` | `POST /purchase-orders/:id/reject` | PO ID, tên PO, trạng thái trước (`sent`), trạng thái sau (`cancel`) |
| `po_auto_cancel` | Scheduled job (7 ngày sau Expected Arrival) | PO ID, tên PO, ngày Expected Arrival, số ngày quá hạn |
| `do_update` | `PATCH /delivery-orders/:id/lines` | DO ID, danh sách dòng với qty cũ và mới |
| `do_sign` | `POST /delivery-orders/:id/sign` | DO ID, ghi chú vendor (nếu có), đường dẫn PDF, đường dẫn chữ ký |
| `do_unlock` | `POST /admin/delivery-orders/:id/unlock` | DO ID, admin đã mở khoá |
| `receipt_validated` | Thông báo Odoo → Portal | picking ID, DO status → Done, mọi chênh lệch số lượng |

Audit log có thể xem bởi admin qua `GET /api/admin/audit-log`, có thể lọc theo actor, loại hành động, và khoảng ngày, phân trang 50 dòng mỗi trang. Không thể chỉnh sửa hoặc xoá qua portal.

### Thông báo email (AWS SES)

| Trigger | Người nhận | Nội dung |
|---|---|---|
| Vendor mới được sync | Nhà cung cấp | Email chào mừng với Vendor ID (số nguyên) + link đặt mật khẩu |
| Yêu cầu quên mật khẩu | Nhà cung cấp | Link đặt lại (hết hạn sau 24h) |
| Vendor từ chối RFQ | PO creator (nhân viên cửa hàng) | PO bị từ chối + hủy trong Odoo |
| PO tự động hủy (7 ngày) | Nhà cung cấp + PO creator | PO tự động hủy — không phản hồi trong 7 ngày sau Expected Arrival |
| Cửa hàng xác nhận biên nhận | Nhà cung cấp | Xác nhận biên nhận. Cảnh báo nếu có số lượng chênh lệch giữa DO và biên nhận |
| DO được admin mở khoá | Nhà cung cấp | DO đã được mở khoá để chỉnh sửa lại |
| RPO được cửa hàng tạo | Nhà cung cấp | Gửi bởi Odoo nội bộ (nút Send by Email) — không qua AWS SES |

**Không gửi email:** Vendor xác nhận PO (dữ liệu được đẩy lên Odoo theo thời gian thực), Vendor ký DO/RN (dữ liệu được đẩy lên Odoo tự động).

Tất cả email do portal gửi dùng AWS SES theo `preferred_language` của vendor cho email gửi đến vendor, và tiếng Việt cho email gửi đến cửa hàng. Email cửa hàng gửi đến email của PO creator (từ `purchase.order.user_id` của Odoo), không phải hộp thư chung. Thông báo RPO được Odoo gửi, không phải portal.

### Cách tiếp cận pipeline DO / RN PDF
1. PDF được tạo **on-the-fly** mỗi lần vendor gọi endpoint — luôn phản ánh dữ liệu mới nhất
2. Render HTML → PDF bằng WeasyPrint, **bằng tiếng Việt**
3. **Không lưu PDF trên server** — tạo mới mỗi lần request
4. Trả về PDF dưới dạng binary response với `Content-Disposition: attachment`
5. Nếu vendor đã ký (có `latest_signature`), chữ ký được nhúng vào PDF. Nếu chưa ký, PDF vẫn tạo được nhưng không có chữ ký

**Lưu ý:** RN PDF dùng **cùng layout và nội dung** với DO PDF, với tiêu đề đổi thành "Biên Bản Trả Hàng" và ngày giao hàng được thay thế bằng ngày lấy hàng.

### Đặc tả nội dung DO PDF
DO PDF in ra bao gồm các phần sau, tất cả bằng tiếng Việt:

**Header:**
- Số PO hiển thị dưới dạng **barcode Code128** (có thể quét bằng máy quét cầm tay của cửa hàng)
- Vendor ID / Mã NCC (`res.partner.id`)
- Mã số thuế Vendor (`res.partner.vat`)
- Số điện thoại di động Vendor (`res.partner.phone` hoặc `mobile`)
- Email liên hệ Vendor (`res.partner.email`)
- Mã cửa hàng (`stock.warehouse.code` — tên ngắn của warehouse)
- Ngày xác nhận đơn hàng
- Ngày giao hàng (ngày duy nhất do vendor đặt trên DO)

**Cột bảng sản phẩm:**

| # | Cột (Tiếng Việt) | Cột (Tiếng Anh) | Mô tả |
|---|---|---|---|
| 1 | Số thứ tự | Line number | Số thứ tự dòng |
| 2 | Mã vạch | Barcode | Barcode sản phẩm |
| 3 | Tên sản phẩm | Product name | Mô tả sản phẩm |
| 4 | Đơn vị tính | UoM | Đơn vị tính (kế thừa từ PO, ví dụ: Thùng 12 Chai, Kg) |
| 5 | Số lượng giao | Delivery qty | Số lượng vendor dự kiến giao |
| 6 | Số lượng thực nhận | Received qty | Để trắng — cửa hàng điền trên giấy |
| 8 | SL chênh lệch | Discrepancy qty | Để trắng — để cửa hàng ghi chênh lệch |
| 9 | Ghi chú | Notes | Để trắng — cho ghi chú cửa hàng |

**Footer:**
- Chữ ký điện tử của vendor (PNG nhúng vào)
- Ghi chú vendor (nếu có khi ký)
- Timestamp của chữ ký
- Ô chữ ký trống cho chữ ký tay của cửa hàng

---

## Phase 4 — React Frontend
**Ước tính: 3–4 ngày**

### Mục tiêu
- Xây dựng tất cả trang portal với UI sạch, thân thiện với mobile
- Triển khai hỗ trợ song ngữ (Tiếng Việt + Tiếng Anh, người dùng tự chuyển đổi)
- Triển khai API client dựa trên `fetch` với JWT refresh tự động
- Tích hợp `signature_pad.js` để capture chữ ký
- Triển khai form chỉnh sửa DO với validation số lượng (qty <= qty đặt) và xử lý trạng thái khoá

### Cách tiếp cận Internationalisation (i18n)
- Dùng `react-i18next` với hai file locale: `vi.json` và `en.json`
- Ngôn ngữ được lưu trong `vendor_users.preferred_language` (lưu trữ phía server)
- Nút chuyển ngôn ngữ (VI / EN) hiển thị trên thanh điều hướng trên cùng ở tất cả các trang
- Khi chuyển ngôn ngữ, frontend gọi `PATCH /api/vendors/me/language` và cập nhật locale của i18next ngay lập tức — không cần tải lại trang
- Ngôn ngữ mặc định: Tiếng Việt (`vi`)
- Tất cả chuỗi UI, nhãn, thông báo lỗi, và văn bản nút đều được dịch
- Nội dung email do backend gửi tuân theo `preferred_language` đã lưu của vendor

### Trang & routes
| Route | Trang | Vai trò | Mô tả |
|---|---|---|---|
| `/login` | Đăng nhập | Cả hai | Vendor ID (số nguyên) + mật khẩu + nút chuyển ngôn ngữ |
| `/admin/login` | Đăng nhập Admin | Admin | Username + mật khẩu (trang đăng nhập riêng biệt) |
| `/set-password` | Đặt mật khẩu | Nhà cung cấp | Đặt mật khẩu lần đầu khi được mời và đặt lại khi quên |
| `/forgot-password` | Quên mật khẩu | Nhà cung cấp | Nhập Vendor ID để nhận link đặt lại |
| `/dashboard` | Dashboard | Nhà cung cấp | Danh sách PO với tìm kiếm/lọc, badge trạng thái, chọn checkbox + nút "Xuất" (PDF riêng lẻ/tổng hợp hoặc CSV) |
| `/purchase-orders/:id` | Chi tiết PO | Nhà cung cấp | Các dòng đặt hàng + DO liên kết + so sánh biên nhận (qty vendor vs qty cửa hàng) |
| `/delivery-orders/:id` | Chi tiết DO | Nhà cung cấp | Các dòng sản phẩm với qty giao có thể chỉnh sửa + ngày giao hàng duy nhất (khoá nếu đã ký) |
| `/delivery-orders/:id/sign` | Ký DO | Nhà cung cấp | Signature pad + ô ghi chú + bước xác nhận cuối |
| `/delivery-orders/:id/pdf` | DO PDF | Nhà cung cấp | Tải xuống/in DO PDF đã ký (có thể in nhiều lần) |
| `/returns` | Danh sách Trả Hàng | Nhà cung cấp | Danh sách RPO với tìm kiếm/lọc, badge trạng thái |
| `/return-orders/:id` | Chi tiết RPO | Nhà cung cấp | Các dòng trả hàng + Return Note liên kết |
| `/return-notes/:id` | Chi tiết RN | Nhà cung cấp | Các dòng sản phẩm trả hàng + bộ chọn ngày lấy hàng (qty chỉ đọc) + Xác nhận & Ký |
| `/return-notes/:id/pdf` | RN PDF | Nhà cung cấp | Tải xuống/in RN PDF đã ký |
| `/profile` | Hồ sơ | Nhà cung cấp | Hồ sơ vendor chỉ đọc + tuỳ chọn ngôn ngữ |
| `/admin/dashboard` | Dashboard Admin | Admin | Thống kê tổng hợp: số lượng vendor, DO/RN đang chờ, trạng thái sync |
| `/admin/vendors` | Danh sách Vendor | Admin | Tất cả tài khoản vendor với trạng thái, lần đăng nhập cuối, số DO/RN |
| `/admin/vendors/:id` | Chi tiết Vendor | Admin | PO, DO, RPO, RN của vendor cụ thể (xem chi tiết chỉ đọc) |
| `/admin/sync` | Trạng thái Sync | Admin | Thời gian sync cuối, vendor bị bỏ qua, nút kích hoạt thủ công |

### Nội dung dashboard Admin
Dashboard admin hiển thị tổng quan:
- **Tóm tắt vendor:** tổng số vendor, số đang hoạt động, số không hoạt động
- **Bảng vendor:** danh sách có thể sắp xếp hiển thị tên vendor, Vendor ID, trạng thái tài khoản (hoạt động/không hoạt động), ngày đăng nhập cuối, số DO đã ký, số DO đang chờ (Draft, chưa ký)
- **Panel trạng thái sync:** timestamp sync cuối, số vendor đã sync, số vendor bị bỏ qua (không có email), nút "Chạy Sync Ngay"
- Click vào bất kỳ dòng vendor nào điều hướng đến `/admin/vendors/:id` — xem chỉ đọc PO và DO của vendor đó, với nút tải xuống cho DO PDF đã ký và nút mở khoá cho DO bị khoá

### Điều hướng Admin
Admin thấy cùng thanh điều hướng trên cùng với vendor, với các mục thêm: **Nhà cung cấp**, **Trạng thái Sync**, **Audit Log**. Nút chuyển ngôn ngữ có mặt — phần admin hỗ trợ cả Tiếng Việt và Tiếng Anh. Các route admin được bảo vệ — JWT vendor không thể truy cập route `/admin/*`.

### Cách tiếp cận HTTP client (native fetch)
- Một wrapper `apiFetch(path, options)` duy nhất xử lý tất cả API call
- Gắn Bearer token từ `localStorage` vào mỗi request
- Khi nhận 401: gọi `/api/auth/refresh`, lưu access token mới, thử lại một lần
- Khi refresh thất bại: xóa storage, chuyển hướng về `/login`
- Không gọi `fetch` trực tiếp trong component — tất cả đều qua wrapper này

### Cách tiếp cận state management
- **TanStack Query** cho tất cả server state — danh sách PO, chi tiết DO, hồ sơ
- **React local state** (`useState`) cho form input, trạng thái chữ ký, toggle UI
- Không cần global state manager ở quy mô này

### Cách tiếp cận responsive design
- Portal hoàn toàn responsive — hoạt động tốt trên cả desktop và mobile
- Tailwind CSS utility classes cho layout (hoặc CSS framework tương đương)
- Canvas signature pad tự điều chỉnh kích thước theo chiều rộng màn hình trên mobile
- Bảng trên mobile thu gọn thành layout dạng card để dễ đọc

### Tóm tắt dashboard Vendor
Phía trên danh sách PO, dashboard vendor hiển thị các thẻ tóm tắt:
- **Chờ xác nhận** — PO đang chờ vendor xác nhận
- **Đã xác nhận** — PO đã xác nhận, DO đang ở các trạng thái khác nhau
- **Đã huỷ** — PO đã bị huỷ

### Giao diện tìm kiếm & lọc danh sách PO
- Thanh tìm kiếm theo số PO (tìm kiếm một phần, debounced 300ms trước khi gọi API)
- Bộ chọn khoảng ngày cho `date_from` và `date_to`
- Badge trạng thái trên mỗi dòng PO (Chờ xác nhận / Đã xác nhận / Đã huỷ) với mã màu
- Trạng thái DO hiển thị cạnh trạng thái PO (Draft / Signed / Done / Cancelled)
- Điều khiển phân trang (trước / sau), page size cố định là 20

### Chi tiết DO — các trường hiển thị cho mỗi dòng sản phẩm
| Trường | Tiếng Việt | Nguồn |
|---|---|---|
| Số thứ tự | Số thứ tự | Số thứ tự dòng |
| Barcode sản phẩm | Mã vạch | từ Odoo `product.product` → `barcode` |
| Tên sản phẩm | Tên sản phẩm | `delivery_order_lines` → `product_name` |
| UoM (chỉ đọc) | Đơn vị tính | `delivery_order_lines` → `uom` (kế thừa từ dòng PO, không thể thay đổi) |
| Số lượng đặt hàng (chỉ đọc) | SL đặt hàng | `delivery_order_lines` → `ordered_qty` (từ dòng PO) |
| Số lượng giao (có thể chỉnh sửa) | Số lượng giao | `delivery_order_lines` → `delivery_qty` (phải <= ordered_qty) |
| Số lượng thực nhận (chỉ đọc) | Số lượng thực nhận | `delivery_order_lines` → `received_qty` (NULL cho đến khi cửa hàng xác nhận; hiển thị qty_done cuối cùng của cửa hàng) |
| Đơn giá | Đơn giá | từ Odoo `purchase.order.line` → `price_unit` |
| Thành tiền | Thành tiền | Tính toán: đơn giá x số lượng giao (chỉ frontend, không lưu) |

### Hành vi chi tiết DO
- Bộ chọn ngày giao hàng ở đầu form DO
- Mỗi dòng sản phẩm hiển thị tất cả các trường trên; số lượng giao là trường duy nhất có thể chỉnh sửa (cùng với ngày giao hàng)
- Validation: số lượng giao phải <= số lượng đặt — cả frontend và backend đều thực thi
- Nút "Lưu" gửi `PATCH /delivery-orders/:id/lines` — có thể nhấn nhiều lần trước khi ký
- Nếu API trả về `locked: true`, tất cả input bị vô hiệu hoá và hiển thị thông báo khoá
- Nút "Ký DO" ở cuối chỉ được bật khi ngày giao hàng đã đặt và tất cả giá trị số lượng giao > 0
- Sau khi ký, trang hiển thị tóm tắt chỉ đọc với nút "In DO"
- Sau khi cửa hàng xác nhận biên nhận, cột `received_qty` được điền — vendor thấy cả số lượng giao và số lượng thực nhận của cửa hàng cạnh nhau

### Chi tiết PO — Xem DO và biên nhận
Trang chi tiết PO hiển thị DO liên kết với trạng thái của nó. Khi DO ở trạng thái Done (cửa hàng xác nhận biên nhận):
- **Số lượng DO của vendor** — những gì vendor dự định giao
- **Số lượng cửa hàng thực nhận** — những gì cửa hàng thực sự xác nhận trong Odoo
- So sánh từng dòng, làm nổi bật sự khác biệt giữa số lượng giao và số lượng nhận

### Trang lịch sử PDF
- Liệt kê tất cả DO vendor đã ký, theo thứ tự thời gian đảo ngược
- Mỗi dòng hiển thị: số PO, mã DO, ngày giao hàng, ngày ký, ghi chú vendor (nếu có), nút tải xuống
- Tải xuống gọi `GET /api/delivery-orders/{do_id}/pdf` — PDF được giữ 24 tháng (khớp với thời hạn lưu PO)

### Capture chữ ký và ghi chú
- `signature_pad.js` render trên canvas HTML5 (tự điều chỉnh cho mobile)
- Ô ghi chú tự do tuỳ chọn phía trên signature pad — vendor có thể mô tả điều kiện giao hàng hoặc ghi chú
- Nút "Xóa" chỉ reset canvas
- "Xác nhận & Gửi" gửi cả PNG chữ ký (base64) và văn bản ghi chú lên `POST /api/delivery-orders/:id/sign`
- Pad và ô ghi chú bị vô hiệu hoá sau khi gửi thành công
- Hiển thị trạng thái loading khi PDF đang được tạo phía server

### Các lưu ý UX quan trọng
- Nhãn trường đăng nhập: **"Mã nhà cung cấp / Vendor ID"** với văn bản hướng dẫn giải thích đã được cung cấp trong email chào mừng (phải là input kiểu number)
- Các dòng DO hiển thị số lượng đặt hàng, số lượng giao, và số lượng thực nhận (khi có) cạnh nhau để dễ so sánh
- PO ở trạng thái Waiting hiển thị nút "Xác nhận PO" và "Từ chối" nổi bật
- Thông báo khoá trên DO đã ký: "Phiếu giao hàng đã được ký / Delivery Order has been signed"
- Placeholder ô ghi chú: "Ghi chú giao hàng / Delivery notes (optional)"
- Tất cả form submission vô hiệu hoá nút khi đang gửi để tránh gửi trùng

---

## Phase 5 — Tăng Cường Bảo Mật
**Ước tính: 0.5 ngày**

### Chiến lược rate limiting (SlowAPI + Redis)
| Nhóm endpoint | Giới hạn |
|---|---|
| `/api/auth/login` | 5 request / phút / IP |
| `/api/auth/forgot-password` | 3 request / phút / IP |
| `/api/auth/set-password` | 5 request / phút / IP |
| Tất cả route `/api/` khác | 60 request / phút / người dùng |

### CORS
- Chỉ cho phép đúng `FRONTEND_URL` — không bao giờ dùng wildcard `*` với credentials
- `allow_credentials=True`

### HTTPS / TLS
TLS termination được xử lý ở tầng infrastructure (load balancer hoặc upstream reverse proxy) — Nginx của portal không quản lý SSL certificate. Nginx portal lắng nghe trên HTTP nội bộ và tin tưởng upstream proxy thực thi HTTPS. Đảm bảo upstream proxy truyền header `X-Forwarded-Proto: https` và `X-Real-IP` đến backend.

### Nginx security headers
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `Strict-Transport-Security: max-age=31536000` (đặt ở upstream nếu TLS ở upstream)
- `Referrer-Policy: no-referrer`

### Bảo mật AWS SES
- IAM user chuyên dụng chỉ có quyền `ses:SendEmail` — không có quyền AWS nào khác
- Sender domain được xác minh trong SES
- Credentials được lưu dưới dạng Docker secrets, không bao giờ trong file `.env` trên production

---

## Phase 6 — Triển Khai Production
**Ước tính: 0.5–1 ngày**

### Chính sách khởi động lại service
Tất cả container: `restart: unless-stopped`. Health check trên backend container (`GET /health`), 3 lần retry trước khi đánh dấu unhealthy.

### Routing Nginx reverse proxy
- Tất cả `/api/*` → container FastAPI backend
- Tất cả `/admin/*` → container React SPA (route admin được xử lý phía client bởi React Router)
- Tất cả đường dẫn khác → container React SPA, catch-all phục vụ `index.html`

### Backup PostgreSQL
- `pg_dump` hàng ngày qua cron trên portal VM
- Nén bằng gzip, lưu trong `/backups/`, giữ 30 ngày
- Tuỳ chọn đồng bộ lên Azure Blob Storage
- File PDF lưu trên filesystem server (giữ 24 tháng) — đưa vào chiến lược backup VM
- Scheduled cleanup job xóa PO và DO, PDF cũ hơn 24 tháng

### Checklist go-live
- [ ] API key tài khoản dịch vụ Odoo được tạo và kiểm tra kết nối từ portal VM
- [ ] Firewall: portal VM → Odoo VM cổng 8069 mở
- [ ] Sender domain AWS SES được xác minh, IAM credentials đã cấu hình
- [ ] Xác minh trường email PO creator có thể truy cập qua Odoo XML-RPC (`purchase.order` → `user_id` → `email`)
- [ ] `ADMIN_INITIAL_PASSWORD` đã đặt và tài khoản admin đầu tiên đã được seed
- [ ] Tất cả secrets production đã đặt trong Docker secrets files
- [ ] Upstream load balancer / proxy đã cấu hình chuyển tiếp HTTPS
- [ ] Job đồng bộ partner chạy thủ công một lần — tài khoản vendor ban đầu được tạo
- [ ] Cơ chế sync Odoo → Portal đã cấu hình và kiểm tra (webhook, polling, hoặc batch — TBD)
- [ ] Ít nhất một vendor thử nghiệm: nhận lời mời → đặt mật khẩu → đăng nhập → thấy dashboard → xác nhận PO → chỉnh sửa + ký DO → in DO PDF → giao đến cửa hàng → cửa hàng xác nhận biên nhận trong Odoo → trạng thái DO Done → nhận email vendor
- [ ] Kiểm tra từ chối RFQ: vendor từ chối → Odoo hủy → nhận email cửa hàng
- [ ] Email thông báo nhận được bởi vendor và cửa hàng
- [ ] DO bị khoá xác nhận vendor không thể chỉnh sửa
- [ ] Admin có thể xem vendor, tải DO PDF, mở khoá DO (vendor nhận email)
- [ ] Audit log hiển thị tất cả hành động từ lần chạy thử nghiệm
- [ ] Giao diện Tiếng Việt và Tiếng Anh đều được xác minh trên desktop và mobile

---

## Tóm Tắt Lộ Trình

| Phase | Mô tả | Ước tính |
|---|---|---|
| 0 | Thiết lập repo, Docker Compose, Odoo API key, AWS SES | 0.5–1 ngày |
| 1 | Tầng Odoo XML-RPC + job đồng bộ partner | 1–2 ngày |
| 2 | JWT auth + hệ thống role + luồng invite/reset + DB schema | 1–1.5 ngày |
| 3 | API endpoints cốt lõi + admin endpoints + PDF + emails | 3–4 ngày |
| 4 | React frontend (vendor portal + phần admin, song ngữ) | 4–5 ngày |
| 5 | Tăng cường bảo mật | 0.5 ngày |
| 6 | Triển khai production + checklist go-live | 0.5–1 ngày |
| **Tổng** | | **~11–15 ngày** |

---

## Các Lưu Ý Quan Trọng

1. **Đăng nhập vendor dùng Vendor ID (số nguyên), không phải email** — trường đăng nhập frontend phải là input `number`; email chào mừng phải nêu rõ Vendor ID; `odoo_partner_id` là lookup key trong `vendor_users`
2. **Đăng nhập admin riêng biệt** — admin dùng username (không phải Vendor ID) trên trang đăng nhập riêng `/admin/login`; JWT admin mang `role: admin`
3. **Đồng bộ hồ sơ không bao giờ đụng đến `hashed_password`** — chỉ `full_name`, `company_name`, `phone`, `tax_id` bị ghi đè bởi sync job; `email` chỉ được sync khi tạo tài khoản lần đầu, không bao giờ nữa
3. **Portal không gọi `button_validate`** — vendor chỉnh sửa DO và ký; Odoo validation do cửa hàng thực hiện. Odoo picking giữ nguyên trạng thái `assigned` sau khi vendor ký
4. **Khoá DO chỉ phía portal** — trạng thái khoá nằm trong bảng `delivery_orders`; Odoo không biết về nó. Admin mở khoá qua phần admin portal
5. **Admin có thể mở khoá DO — vendor không thể** — endpoint mở khoá chỉ dành cho admin; vendor không có tùy chọn tự mở khoá
6. **PDF cần HTTP session riêng** — kết nối XML-RPC không thể tải các endpoint `/report/pdf/`; cần một `requests.Session()` riêng biệt được xác thực qua `/web/session/authenticate`
7. **PDF được lưu 24 tháng** — scheduled cleanup job thực thi điều này; đảm bảo filesystem server có đủ dung lượng và được đưa vào chiến lược backup VM
8. **Ghi chú vendor là tuỳ chọn nhưng phải được lưu** — dù trống, trường `vendor_comment` trong `delivery_orders` phải được lưu tường minh (chuỗi rỗng, không phải NULL) để phân biệt "không có ghi chú" với lỗi dữ liệu
9. **Audit log chỉ ghi thêm** — không có endpoint xóa hoặc cập nhật cho `audit_log`; lỗi ghi phải được log nhưng không được làm hành động cha thất bại (non-blocking)
10. **Tên trường Odoo 16** — `qty_done` trên `stock.move.line` (không phải `quantity`), `move_lines` trên `stock.picking` (không phải `move_ids`)

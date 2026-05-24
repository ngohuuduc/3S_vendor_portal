# 3SACH Vendor Portal — Luồng Quy Trình

---

## Tổng Quan Nghiệp Vụ (Nhìn Đơn Giản)

```mermaid
flowchart LR
    classDef store    fill:#DBEAFE,stroke:#3B82F6,color:#1E3A5F,font-weight:bold
    classDef vendor   fill:#D1FAE5,stroke:#10B981,color:#064E3B,font-weight:bold
    classDef portal   fill:#EDE9FE,stroke:#7C3AED,color:#2E1065,font-weight:bold
    classDef internal fill:#FEF3C7,stroke:#F59E0B,color:#451A03,font-weight:bold

    S1([🏪 Cửa hàng\ntạo RFQ\ntrên Odoo]):::store
    S2([📧 Gửi RFQ\ncho nhà cung cấp]):::store

    V1([✅ Nhà cung cấp chọn ngày giao\nvà xác nhận PO trên Portal]):::vendor
    P_DO([🔄 Odoo tự tạo DO\nngày giao = date_planned đã xác nhận]):::portal
    V2([📦 Điền số lượng\nvà ghi chú tổng phiếu\ntrên Portal]):::vendor

    P_CUT{{23:00 đêm trước\nngày giao\nDO chuyển Đã Gửi\n(UI lock only, qty đã ghi\nreal-time từ trước)}}:::portal

    V3([🖨️ In DO PDF\nbản cuối — 2 bản]):::vendor
    V4([🚚 Cầm 2 tờ DO\nra cửa hàng]):::vendor

    I1([🏪 Cửa hàng ký\nphysically 2 tờ DO\nmỗi bên giữ 1 tờ]):::store
    I2([✔️ Cửa hàng\nxác nhận Receipt\ntrên Odoo]):::store

    S1 --> S2 --> V1
    V1 --> P_DO --> V2
    V2 --> P_CUT
    P_CUT --> V3
    V3 --> V4
    V4 --> I1
    I1 --> I2
    I2 --> DONE([🎉 Hoàn tất]):::store
```

> **Màu sắc:** 🔵 Xanh dương = Cửa hàng | 🟢 Xanh lá = Nhà cung cấp | 🟣 Tím = Hệ thống Portal | 🟡 Vàng = Nội bộ 3Sach

---

## Toàn Bộ Quy Trình Mua Hàng: RFQ -> Xác Nhận PO -> DO -> Giao Hàng -> Biên Nhận

```mermaid
flowchart TD
    subgraph ONBOARD["Onboarding Nhà Cung Cấp"]
        A1(["Nhà cung cấp mới được thêm vào Odoo\nis_vendor = True + có email"]) --> A2[Job đồng bộ chạy mỗi 6h]
        A2 --> A3{Có email?}
        A3 -- Không --> A4[Bỏ qua & ghi log để xử lý thủ công]
        A3 -- Có --> A5["Tạo tài khoản portal\nchưa kích hoạt, chưa có mật khẩu"]
        A5 --> A6["Gửi Email Chào Mừng qua AWS SES\nVendor ID + link đặt mật khẩu 24h"]
        A6 --> A7["Nhà cung cấp đặt mật khẩu\nTài khoản được kích hoạt"]
    end

    subgraph RFQ_PO["RFQ đến Xác Nhận / Từ Chối PO"]
        B1(["Cửa hàng tạo RFQ trong Odoo\nNhập ngày mong muốn giao hàng (date_planned)"]) --> B2["Sent RFQ\nEmail gửi cho nhà cung cấp"]
        B2 --> B2A["NCC xem RFQ trên Portal\nChọn ngày dự kiến giao hàng\n(không quá 7 ngày sau date_planned)"]
        B2A --> B3{"Hành động\ncủa nhà cung cấp?"}
        B3 -- Xác nhận --> B4["NCC nhấp 'Xác Nhận Đơn Đặt Hàng'\nPortal cập nhật date_planned\n+ button_confirm cùng lúc"]
        B4 --> B6["PO được xác nhận trong Odoo\nstate: sent → purchase\nKhông gửi email"]
        B3 -- Từ chối --> B7["NCC nhấp 'Từ Chối Đơn Đặt Hàng'\nPopup: điền lý do từ chối"]
        B7 --> B7A["NCC điền lý do và nhấp 'Xác nhận'\nPortal log lý do từ chối"]
        B7A --> B8["Portal gọi button_cancel\ntrên purchase.order qua XML-RPC"]
        B8 --> B9["RFQ bị hủy trong Odoo\nstate: sent → cancel"]
        B9 --> B10["Email đến Buyer\n(user_id, fallback create_uid)\n+ Kho Nhận Hàng\n(picking_type_id.warehouse_id)\nRFQ bị từ chối + lý do"]
        B3 -- Không hành động / 7 ngày dương lịch sau date_planned --> B3X["Portal tự động hủy\nbutton_cancel qua XML-RPC"]
        B3X --> B3Y["Email đến NCC + Buyer\n+ Kho Nhận Hàng\nPO tự động hủy — không phản hồi"]
    end

    subgraph DO["Delivery Order — Vendor Portal"]
        C1["Odoo tự tạo 1 DO (stock.picking)\nkhi PO confirmed\nNgày giao = date_planned đã xác nhận bởi NCC"] --> C2["NCC sửa DO trên portal:\n- Số lượng giao (≤ qty đặt, đúng decimal UoM)\n- Ghi chú tổng phiếu (header)\n(Ngày giao không chỉnh được — lấy từ PO)"]
        C2 --> C3["Nhà cung cấp in DO PDF\n(in trực tiếp, không cần ký)\nCó thể in nhiều lần"]
        C3 --> C4["Hạn chót: 23:00 đêm trước ngày giao\n(= scheduled_date − 1 ngày)\nDO tự động → Đã Gửi (Sent)\nUI lock only — không push\n(qty đã ghi real-time qua PATCH)"]
        C4 --> C5["Nhà cung cấp in DO PDF phiên bản cuối\nIn 2 bản"]
    end

    subgraph DELIVERY["Giao Hàng Thực Tế — Cửa Hàng"]
        D1["Nhà cung cấp mang 2 bản DO in\nđến cửa hàng cùng với hàng hoá"] --> D2["Cửa hàng nhận hàng\nKiểm tra số lượng\n(qty đã pre-fill từ DO)"]
        D2 --> D3["Cả hai bên ký tay\n2 bản giấy DO\nMỗi bên giữ 1 bản"]
        D3 --> D4["Cửa hàng xác nhận trên Odoo\n(có thể điều chỉnh qty nếu hàng hư)"]
    end

    subgraph STORE_CONFIRM["Xác Nhận Biên Nhận — Odoo"]
        E1["Cửa hàng xác nhận Receipt\nqty_done finalized\nstock move được tạo"] --> E2["Portal live query Odoo\nDO → Đã Hoàn Thành (Done)\n(real-time, không nightly sync)"]
        E2 --> E3["Email đến nhà cung cấp\nXác nhận biên nhận (đơn thuần)\nKhông kèm cảnh báo chênh lệch"]
    end

    subgraph UNLOCK["Ngoại Lệ: Mở Khoá DO (qua Odoo, KHÔNG qua Portal)"]
        G1(["Nhà cung cấp nhập sai số lượng"]) --> G2["Admin cập nhật scheduled_date\nTRÊN ODOO (đẩy về tương lai)"]
        G2 --> G3["Lock condition tự giải phóng\n(scheduled_date <= today = false)"]
        G3 --> G4["Vendor cập nhật DO và in lại\ntrên Portal (state Mới trở lại)"]
    end

    subgraph RETURNS["Quy Trình Trả Hàng"]
        R1(["Cửa hàng tạo RPO trong Odoo\nEmail gửi cho nhà cung cấp"]) --> R2["RPO xuất hiện trên portal\nqty giao pre-fill = qty đặt\n(không cần bấm Xác Nhận RPO)"]
        R2 --> R3["Nhà cung cấp chỉnh ngày nhận hàng\nKhông thể từ chối hoặc thay đổi qty"]
        R3 --> R4["Nhà cung cấp in RO PDF\nBanner: 'NCC vui lòng đến lấy hàng\ntrước hạn chót (ngày trả + 7 ngày dương lịch)'"]
        R4 --> R5["Nhà cung cấp đến cửa hàng\nđể lấy hàng trả về\nCả hai ký 2 bản giấy"]
        R5 --> R6{"Cửa hàng\nxác nhận Odoo?"}
        R6 -- Có --> R7[Trạng thái RO: Done]
        R6 -- "Quá hạn chót\n(ngày trả + 7 ngày dương lịch)" --> R8["Odoo tự xác nhận\n(scheduled action Odoo\n— TBD Odoo team)\n→ portal sync state done"]
    end

    %% Kết nối các giai đoạn chính
    A7 --> B2
    B6 --> C1
    C5 --> D1
    D4 --> E1
    C4 -.->|"Trạng thái Đã Gửi (vendor cần unlock)"| G1
    G4 -.->|"Quay lại chỉnh sửa"| C2
```

---

## Sơ Đồ Swimlane (4 Bên Tham Gia)

```mermaid
sequenceDiagram
    actor Vendor as Nhà cung cấp
    participant Portal as Vendor Portal
    participant Odoo as Odoo (XML-RPC)
    participant Store as Cửa hàng (chỉ Odoo)

    Note over Vendor,Store: ── Giai đoạn 1: RFQ — Xác Nhận Ngày & Xác Nhận PO ──
    Odoo->>Vendor: Email Sent RFQ (kèm date_planned — ngày mong muốn giao)
    Vendor->>Portal: Chọn ngày dự kiến giao hàng (không quá 7 ngày sau date_planned)
    Vendor->>Portal: Nhấp "Xác Nhận Đơn Đặt Hàng" (hoặc "Từ Chối" + lý do)
    Portal->>Odoo: Cập nhật date_planned + button_confirm cùng lúc (hoặc button_cancel)
    Odoo-->>Portal: DO được tạo tự động (nếu xác nhận)

    Note over Vendor,Store: ── Giai đoạn 2: Delivery Order — Sửa qty & In ──
    Portal->>Vendor: DO sẵn sàng — ngày giao cố định từ PO
    Vendor->>Portal: Chỉnh qty (đúng decimal UoM)
    Vendor->>Portal: In DO PDF (in trực tiếp, có thể in nhiều lần)
    Note over Portal: 23:00 đêm trước scheduled_date → DO chuyển Đã Gửi (UI lock only, qty đã ghi real-time qua PATCH)
    Portal->>Vendor: In DO PDF phiên bản cuối (2 bản)

    Note over Vendor,Store: ── Giai đoạn 3: Giao Hàng Thực Tế ──
    Vendor->>Store: Mang 2 bản DO + hàng hoá đến cửa hàng
    Store->>Vendor: Cả hai ký tay 2 bản giấy DO — mỗi bên giữ 1 bản

    Note over Vendor,Store: ── Giai đoạn 4: Xác Nhận Biên Nhận trên Odoo ──
    Store->>Odoo: Xem xét Receipt, điều chỉnh số lượng nếu hàng hư
    Store->>Odoo: Xác nhận Receipt — qty_done finalized
    Odoo->>Portal: Portal live query (real-time) — DO → Đã Hoàn Thành
    Portal->>Vendor: Email: xác nhận biên nhận (đơn thuần, không cảnh báo chênh lệch — Log Note hiển thị lịch sử thay đổi qty_done nếu cần)

    Note over Vendor,Store: ── Giai đoạn 5: Trả Hàng ──
    Store->>Odoo: Tạo RPO
    Odoo->>Vendor: Email: thông báo đơn trả hàng
    Vendor->>Portal: Xem RPO, chỉnh ngày nhận hàng (không cần xác nhận RPO)
    Vendor->>Portal: In RO PDF
    Vendor->>Store: Lấy hàng trả về với RO in ra (2 bản)
    Store->>Odoo: Xác nhận biên nhận trả hàng
    Odoo->>Portal: Trạng thái RO -> Done

    Note over Vendor,Store: ── Giai đoạn 6: Xuất Dữ Liệu ──
    Vendor->>Portal: Xuất PDF (riêng lẻ hoặc tổng hợp) hoặc CSV

    Note over Vendor,Store: ── Ngoại Lệ: Mở Khoá DO (qua Odoo, không qua Portal) ──
    Store->>Odoo: Admin cập nhật scheduled_date (đẩy về tương lai)
    Note over Portal: Lock condition tự giải phóng (scheduled_date > today)
    Vendor->>Portal: Chỉnh sửa lại số lượng DO và in lại
```

---

## Luồng Xác Thực

### Xác Thực Nhà Cung Cấp
1. Nhà cung cấp nhập **Vendor ID** (số nguyên — `res.partner.id` từ Odoo) và mật khẩu trên trang đăng nhập
2. Backend tra cứu `vendor_users` theo `odoo_partner_id`; xác minh bcrypt luôn chạy (kể cả khi ID không tồn tại — an toàn về mặt timing)
3. Cùng một thông báo lỗi chung cho cả sai ID lẫn sai mật khẩu — không tiết lộ ID có tồn tại hay không
4. Thành công: cấp phát **JWT access token** (30 phút) và **refresh token** (7 ngày), cả hai đều mang `role: vendor`
5. Tất cả các request tiếp theo mang access token trong header `Authorization`
6. Khi nhận 401: frontend gọi thầm `/api/auth/refresh` → thử lại một lần với token mới
7. Khi refresh thất bại: xóa storage → chuyển hướng về `/login`

### Xác Thực Admin
- Admin đăng nhập tại `/admin/login` với **username** (không phải Vendor ID) và mật khẩu
- JWT mang `role: admin` — token admin không thể truy cập route vendor và ngược lại
- Tài khoản admin đầu tiên được seed từ biến môi trường `ADMIN_INITIAL_PASSWORD` khi khởi động

### Token Blacklist
- Khi đăng xuất, refresh token được thêm vào Redis với TTL khớp với thời gian còn lại của token

---

## Thông Báo Email

| Sự kiện | Người nhận | Ngôn ngữ | Nội dung |
|---|---|---|---|
| Tài khoản vendor mới được tạo (job đồng bộ) | Nhà cung cấp | Tiếng Việt (mặc định) | Vendor ID (số nguyên) + link đặt mật khẩu (hết hạn sau 24h) |
| Yêu cầu đặt lại mật khẩu | Nhà cung cấp | Ngôn ngữ ưa thích của nhà cung cấp | Link đặt lại (hết hạn sau 24h) |
| RFQ bị từ chối bởi nhà cung cấp | **Buyer** (`purchase.order.user_id`, fallback `create_uid`) **+ Kho Nhận Hàng** (`purchase.order.picking_type_id.warehouse_id`) | Tiếng Việt | Mã RFQ, tên nhà cung cấp, lý do |
| PO tự động hủy (7 ngày dương lịch sau Expected Arrival) | Nhà cung cấp + **Buyer + Kho Nhận Hàng** | Ngôn ngữ ưa thích / Tiếng Việt | PO tự động hủy — không phản hồi trong 7 ngày |
| DO được in bởi nhà cung cấp | — | — | Không gửi email khi in |
| Biên nhận được cửa hàng xác nhận | Nhà cung cấp | Ngôn ngữ ưa thích của nhà cung cấp | Xác nhận biên nhận đơn thuần (số PO, mã biên nhận). Không kèm cảnh báo chênh lệch — vendor xem Log Note nếu cần |
| ~~Biên nhận được cửa hàng xác nhận (qty chênh lệch)~~ | — | — | **Bỏ email cảnh báo chênh lệch** — chỉ gửi 1 email xác nhận biên nhận đơn thuần. Vendor xem Log Note (mail.tracking.value) nếu cần biết lịch sử thay đổi `qty_done` |
| ~~DO được admin mở khoá~~ | — | — | **Không gửi email** — không có chức năng admin unlock trên Portal (DO-LIVE-001 Blocker 6). Admin reset `scheduled_date` trên Odoo trực tiếp |

> Tất cả email gửi đến nhà cung cấp tuân theo `vendor_users.preferred_language` (`vi` hoặc `en`). Email nội bộ luôn bằng tiếng Việt. Gửi email qua **AWS SES** với IAM user chuyên dụng (chỉ có quyền `ses:SendEmail`).

---

## Quyền Hạn Admin

Portal admin dùng chung bố cục giao diện với nhà cung cấp và có thêm các mục menu: **Nhà cung cấp**, **Trạng thái Sync**, **Audit Log**.

| Quyền hạn | Endpoint |
|---|---|
| Xem toàn bộ tài khoản nhà cung cấp (trạng thái, lần đăng nhập cuối, số biên nhận) | `GET /api/admin/vendors` |
| Xem chi tiết PO và biên nhận của một nhà cung cấp cụ thể | `GET /api/admin/vendors/{partner_id}` |
| Kích hoạt / vô hiệu hoá tài khoản nhà cung cấp | `PATCH /api/admin/vendors/{partner_id}/deactivate|reactivate` |
| Tải xuống PDF DO của bất kỳ nhà cung cấp nào | `GET /api/admin/delivery-orders/{do_id}/pdf` |
| ~~Mở khoá DO đã Locked~~ — **không có endpoint trên Portal** (admin mở khoá bằng cách cập nhật `scheduled_date` trên Odoo trực tiếp; xem DO-LIVE-001 Blocker 6) | — |
| Kích hoạt thủ công job đồng bộ partner Odoo | `POST /api/admin/sync` |
| Xem trạng thái sync (lần chạy cuối, số vendor đã sync, số bị bỏ qua) | `GET /api/admin/sync/status` |
| Xem audit log có phân trang | `GET /api/admin/audit-log` |

**Audit log** ghi lại các loại hành động: `login`, `po_confirm`, `po_reject`, `po_auto_cancel`, `do_update`, `do_print`, `rn_print`, `receipt_validated`. Chỉ được ghi thêm — không thể chỉnh sửa hoặc xoá qua portal. (Đã bỏ `do_unlock` vì không có chức năng admin unlock trên Portal.)

**Không thể chỉnh sửa hồ sơ trong portal.** Mọi thay đổi hồ sơ nhà cung cấp (tên, email, điện thoại, công ty) phải thực hiện trong Odoo và sẽ được đồng bộ trong chu kỳ 6 giờ tiếp theo. Admin không thể chỉnh sửa hồ sơ nhà cung cấp trực tiếp.

---

## Ánh Xạ Trạng Thái PO: Portal vs Odoo

Portal và Odoo duy trì **nhãn trạng thái khác nhau**. Hành vi gốc của Odoo không bao giờ bị thay đổi.

| Trạng thái PO trên Portal | Trạng thái Odoo | Điều kiện kích hoạt | Nhà cung cấp có thể làm |
|---|---|---|---|
| **Waiting** | `sent` | Cửa hàng gửi RFQ | Xác nhận hoặc Từ chối |
| **Confirmed** | `purchase` | Nhà cung cấp xác nhận PO trên portal | Xem DO, xuất dữ liệu |
| **Canceled** | `cancel` | Nhà cung cấp từ chối, cửa hàng hủy, hoặc tự động hủy (7 ngày sau Expected Arrival) | Chỉ đọc |

---

## State Machine của DO

> **Label song ngữ hiển thị trên giao diện (VN / EN):** 4 trạng thái — Draft → **Mới (New)**, Locked → **Đã Gửi (Sent)**, Done → **Đã Hoàn Thành (Done)**, Cancelled → **Đã Huỷ (Canceled)**.

```mermaid
stateDiagram-v2
    [*] --> Draft : PO được xác nhận,\nOdoo tự động tạo DO\n(VN: "Mới")

    note right of Draft
        Vendor sửa qty_done\nvà ghi chú tổng phiếu\n(qty mặc định = qty đặt)
    end note

    Draft --> Locked : Hạn chót: 23:00 đêm trước\nscheduled_date trên DO\n(scheduled_date − 1 ngày)\nUI lock only (không push)\n(VN: "Đã Gửi")

    Locked --> Draft : Admin cập nhật scheduled_date\nTRÊN ODOO (đẩy về tương lai)\n→ lock condition tự giải phóng

    Locked --> Done : Cửa hàng confirm Receipt\n(real-time, live query)

    Draft --> Done : Live query phát hiện\nReceipt đã confirm\n(vendor chưa dùng portal)

    Done --> [*]

    Draft --> Canceled : PO bị hủy\n(trước khi xác nhận biên nhận)

    Canceled --> [*]

    note right of Draft
        NCC sửa qty, in PDF — tự do
        Ngày giao cố định (= scheduled_date đã xác nhận)
        Qty phải <= qty đặt, đúng decimal UoM
        (kg → 0.1, units → số nguyên)
        Hạn chót hiển thị: giờ + ngày DO bị khoá
        In = tạo PDF on-the-fly (không khoá)
    end note

    note right of Locked
        Hạn chót: 23:00 đêm trước scheduled_date
        UI lock only (derive từ scheduled_date.date() <= today)
        Không có job push, không có nightly sync
        NCC: chỉ đọc, in PDF phiên bản cuối
    end note

    note right of Done
        Nhà cung cấp: chỉ đọc, cột SL Giao đổi label thành "SL Thực Nhận"
        (1 field qty_done, cửa hàng có thể đã ghi đè giá trị vendor nhập)
        Có thể xuất PDF/CSV
        Email xác nhận biên nhận đơn thuần (không cảnh báo chênh lệch)
    end note

    note right of Canceled
        Nhà cung cấp: chỉ đọc
        Chỉ có thể xảy ra nếu Receipt chưa được xác nhận
    end note
```

---

## State Machine của RO (Return Order)

> **Label song ngữ (VN / EN):** 2 trạng thái duy nhất — `assigned` → **Chờ Thu Hồi (Waiting)**, `done` → **Đã Hoàn Thành (Done)**. **Không có Đã Huỷ** vì Odoo return picking không có state cancel độc lập.

```mermaid
stateDiagram-v2
    [*] --> ChoThuHoi : RPO được cửa hàng tạo,\nRO tạo tự động\nqty giao mặc định = qty đặt\n(VN: "Chờ Thu Hồi")

    ChoThuHoi --> DaHoanThanh : Cửa hàng xác nhận return receipt\ntrên Odoo (trước hạn chót)\n(VN: "Đã Hoàn Thành")

    ChoThuHoi --> DaHoanThanh : Quá hạn chót (ngày trả + 7 ngày dương lịch)\n→ Odoo tự xác nhận (scheduled action Odoo — TBD)\n→ portal đọc state done

    DaHoanThanh --> [*]

    note right of ChoThuHoi
        Nhà cung cấp: chỉnh ngày nhận hàng, in PDF — tự do
        Không thể thay đổi số lượng, không thể từ chối
        Không cần bấm Xác Nhận RPO
        Banner: "NCC vui lòng đến lấy hàng
        trước hạn chót (dd/mm/yyyy)"
        Hạn chót = ngày trả dự kiến + 7 ngày
        (KHÔNG dùng cơ chế 23:00 cutoff như DO)
    end note

    note right of DaHoanThanh
        Trả hàng hoàn tất — có thể do:
        (a) cửa hàng xác nhận trên Odoo, hoặc
        (b) Odoo tự xác nhận sau hạn chót
            (scheduled action phía Odoo;
             portal chỉ đọc state)
        Có thể xuất PDF/CSV
    end note
```

---

## Trả Hàng: RPO & Return Order (Biên Bản Trả Hàng)

| Khái niệm | Quy trình mua hàng | Quy trình trả hàng |
|---|---|---|
| Đơn hàng | PO (Purchase Order) | RPO (Return Purchase Order) |
| Chứng từ giao nhận | DO (Delivery Order) | RO (Return Order / Biên Bản Trả Hàng) |
| Nhà cung cấp có thể chỉnh sửa | Số lượng giao (chỉ giảm, không tăng vượt qty đặt; đúng decimal UoM: kg/lít/mét → 2 chữ số thập phân, còn lại (gram, thùng, chai, cái, hộp, units) → số nguyên). Ngày giao cố định từ PO | Chỉ ngày nhận hàng (không thay đổi qty hay UoM) |
| Số lượng giao mặc định | = Số lượng đặt (pre-fill khi PO confirmed) | = Số lượng đặt (pre-fill khi RPO được tạo) |
| Nhà cung cấp cần xác nhận | Có (bấm "Xác Nhận" PO) | Không (không cần bấm xác nhận RPO) |
| Hạn chót / cơ chế khoá | 23:00 đêm trước `scheduled_date` → DO Locked, vendor chỉ đọc | Ngày trả hàng dự kiến + **7 ngày dương lịch** → **Odoo tự xác nhận trả hàng** (scheduled action Odoo, **TBD phía Odoo team**); portal chỉ đọc state `done` và reflect |
| Banner/note hạn chót | "Phiếu giao nhận bị khoá lúc 23:00 ngày dd/mm/yyyy" | **"Nhà cung cấp vui lòng đến lấy hàng trước hạn chót (dd/mm/yyyy)"** |
| Chữ ký điện tử | Không — chỉ in PDF | Không — chỉ in PDF |
| PDF có thể in | Có (DO PDF, in nhiều lần) | Có (RO PDF, cùng định dạng) |
| Trao đổi vật lý | Nhà cung cấp giao hàng đến cửa hàng | Nhà cung cấp lấy hàng từ cửa hàng |

---

## Lưu Trữ Dữ Liệu

- Nhà cung cấp có thể xem dữ liệu PO trong **24 tháng** kể từ ngày tạo PO
- Áp dụng cho **tất cả trạng thái PO**: Waiting, Confirmed, Canceled
- Áp dụng cho **tất cả trạng thái DO**: Mới (New), Đã Gửi (Sent), Đã Hoàn Thành (Done), Đã Huỷ (Canceled)
- Áp dụng cho trả hàng (RPO/RO) như nhau
- PO cũ hơn 24 tháng bị **xoá vĩnh viễn** khỏi cơ sở dữ liệu portal
- Một scheduled cleanup job chạy định kỳ để thực thi quy tắc này

---

## Xuất Dữ Liệu

- Nhà cung cấp có thể xuất dữ liệu dưới dạng **PDF** hoặc **CSV**
- PDF: chứng từ riêng lẻ hoặc báo cáo tổng hợp nhiều bản ghi
- CSV: dữ liệu hàng loạt để nhà cung cấp chỉnh sửa và nhập vào hệ thống của họ
- Giao diện hỗ trợ chọn một hoặc nhiều bản ghi để xuất
- Có bộ lọc theo khoảng ngày
- Bao gồm cả PO/DO thông thường lẫn trả hàng (RPO/RO)

---

## Câu Hỏi Chờ Xác Nhận ==> đã xác nhận 

| # | Chủ đề | Chi tiết | Trả lời |
|---|---|---|---|
| 1 | **~~Hai job 23:00 chạy cùng lúc~~** (Q&A lịch sử) | ~~Job cutoff (lock DO + push Odoo) và job nightly sync...~~ | **Đã giải quyết bởi [DO-LIVE-001](IssueLog.md):** Không có job cutoff, không có nightly sync. DO update theo thời gian thực qua live query Odoo. 23:00 chỉ là UI lock trên Portal (derive từ `scheduled_date.date() <= today`). | 
| 2 | **RO có cần Unlock không?** | DO không có endpoint unlock trên Portal — admin reset `scheduled_date` trên Odoo. RO hiện không có cơ chế tương đương. Nếu ngày nhận hàng sai, có cách sửa không? | Unlock xảy ra trên Odoo không xảy trên portal. RO chỉ cho vendor chỉnh `pickup_date`. | 

---

**Bảng Thuật Ngữ**

| Thuật ngữ | Ý nghĩa |
|---|---|
| NCC | Nhà cung cấp (Vendor) |
| DO | Delivery Order — chứng từ giao hàng của nhà cung cấp, được chỉnh sửa và in trên portal |
| RO | Return Order / Biên Bản Trả Hàng — tương đương DO cho quy trình trả hàng, được nhà cung cấp in |
| Receipt | Phiếu nhập kho — bản ghi hàng nhập trong Odoo, được xác nhận bởi cửa hàng |
| RPO | Return Purchase Order — tương đương PO cho quy trình trả hàng, do cửa hàng tạo |
| SL | Số lượng (Quantity) |
| RFQ | Request for Quotation |
| PO | Purchase Order |

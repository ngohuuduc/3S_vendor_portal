# Vendor Portal - Tài Liệu Đặc Tả
> Tài liệu Specs & Approach — không bao gồm code triển khai

| Tài liệu | Mục đích |
|---|---|
| **README.md** (file này) | Business logic, quy trình nghiệp vụ, và các quyết định thiết kế cấp cao |
| [PROCESS_FLOW.md](PROCESS_FLOW.md) | Sơ đồ luồng chi tiết: RFQ → PO → DO → Giao hàng → Xác nhận, kèm swimlane và state machine |
| [roadmap.md](roadmap.md) | Kế hoạch triển khai theo phase: DB schema, API endpoints, cấu hình hạ tầng, checklist go-live |

---

## Tổng Quan Dự Án

Một cổng thông tin web song ngữ (Tiếng Việt + Tiếng Anh) độc lập phục vụ hai loại người dùng: **nhà cung cấp (vendor)** và **quản trị viên portal (portal admin)**. Vendor đăng nhập bằng **Vendor ID** (`res.partner.id` từ Odoo), xác nhận hoặc từ chối các RFQ đã gửi, chỉnh sửa Delivery Order (số lượng, ngày giao hàng, ghi chú) và in DO PDF, đồng thời theo dõi trạng thái giao hàng cho đến khi cửa hàng xác nhận biên nhận. Portal cũng hỗ trợ trả hàng thông qua Return Purchase Order (RPO) và Biên Bản Trả Hàng (Goods Return Note). Vendor có thể xuất dữ liệu dưới dạng PDF hoặc CSV phục vụ lập hóa đơn và đối soát. Portal admin dùng chung giao diện với vendor nhưng có thêm quyền xem toàn bộ vendor, toàn bộ PO và DO trong hệ thống, kích hoạt đồng bộ Odoo thủ công, mở khóa DO đã Locked, và tải xuống PDF của bất kỳ vendor nào. Portal chạy trên một VM riêng biệt với Odoo và tích hợp qua Odoo XML-RPC API bằng một tài khoản dịch vụ chuyên dụng. **Vendor chỉ truy cập portal; cửa hàng chỉ truy cập Odoo.** Toàn bộ email được gửi qua AWS SES.

---

## Các Quyết Định Đã Xác Nhận

| Vấn đề | Quyết định |
|---|---|
| Phiên bản Odoo | 16 Community Edition |
| Giao thức Odoo API | XML-RPC (Python `xmlrpc.client`) |
| Định danh đăng nhập Vendor | `res.partner.id` (số nguyên, do Odoo cấp — không bao giờ thay đổi) |
| Mật khẩu Vendor | Thuộc portal, lưu trong portal PostgreSQL nội bộ |
| Nguồn dữ liệu hồ sơ | Đồng bộ một chiều từ Odoo `res.partner` cho các trường hồ sơ (tên, công ty, điện thoại, mã số thuế). Email được sao chép từ Odoo chỉ khi tạo tài khoản lần đầu (dùng để gửi email chào mừng), sau đó không bao giờ bị ghi đè bởi sync |
| Khởi tạo tài khoản | Tự động từ các partner Odoo có `is_vendor = True` **và** có `email` |
| Ngôn ngữ portal | Tiếng Việt + Tiếng Anh (song ngữ, người dùng tự chuyển đổi) |
| Trạng thái PO trên portal | Waiting (`sent`), Confirmed (`purchase`), Canceled (`cancel`) — Draft (`draft`) không hiển thị. Tự động huỷ sau 7 ngày kể từ Expected Arrival nếu vendor chưa xác nhận hoặc từ chối |
| Trạng thái DO trên portal | Draft (vendor sửa qty_done + ghi chú tổng phiếu), Locked (23:00 cutoff → push quantity_done), Done (cửa hàng xác nhận), Cancelled — **DO-LIVE-001: không lưu local DB, derive trực tiếp từ Odoo** |
| Trạng thái PO Odoo (không thay đổi) | RFQ / RFQ Sent / Purchase Order / Canceled — portal không thay đổi hành vi gốc của Odoo |
| DO trên mỗi PO | Odoo tự động tạo đúng 1 DO (stock.picking) khi PO được xác nhận — portal đọc và hiển thị. Vendor không thể tạo thêm DO |
| Xác nhận ngày giao PO | Trường "ngày dự kiến giao hàng" trên Portal **mặc định pre-fill bằng `date_planned` cửa hàng đã chọn** (không hiển thị placeholder dd/mm/yyyy trống). NCC có thể giữ nguyên hoặc chọn lại ngày khác trước khi xác nhận. Portal cập nhật `date_planned` trên `purchase.order`. Ngày vendor chọn không được quá 7 ngày sau `date_planned` của cửa hàng |
| Quy trình xác nhận PO | 1 bước: Vendor chọn ngày giao → nhấp **"Xác Nhận Đơn Đặt Hàng"** → portal cập nhật `date_planned` trên PO + gọi `button_confirm` cùng lúc → Odoo tạo DO với `scheduled_date` = `date_planned` |
| Từ chối PO | Khi NCC nhấp "Từ Chối", popup yêu cầu điền lý do. Sau khi xác nhận → push cancel về Odoo + log lý do |
| Ngày giao hàng DO | **Cố định** — lấy từ `scheduled_date` đã được cập nhật khi NCC xác nhận ngày giao trên PO. NCC không chỉnh ngày trên DO |
| Chỉnh sửa DO | Khi PO được xác nhận, **số lượng giao trên DO mặc định = số lượng đặt** (không phải 0). Vendor có thể chỉnh giảm nhưng **không được tăng vượt số lượng đặt** (hệ thống chặn cứng, không chỉ cảnh báo). Decimal theo UoM: **kg / lít / mét → 2 chữ số thập phân**; **các UoM còn lại (bao gồm gram, thùng, chai, cái, hộp, units) → 0 chữ số thập phân (số nguyên)**. Vendor cũng nhập ghi chú tổng phiếu giao hàng (header-level). Không chỉnh ngày, không có ghi chú riêng từng dòng SP. **Nếu sau đó cửa hàng sửa qty trên PO Odoo (ví dụ 10 → 12): portal KHÔNG đọc lại để refresh default — giữ nguyên giá trị vendor đã lưu** |
| Điều kiện khoá DO | 23:00 cutoff đêm trước ngày giao hàng — DO tự động Locked |
| Ký DO | **Không cần ký điện tử** — vendor chỉ cần in PDF trực tiếp |
| In DO | PDF tạo on-the-fly mỗi lần in (phiên bản mới nhất). Không lưu PDF trên server |
| Hạn chót DO | Hiển thị giờ và ngày cụ thể mà DO bị khoá (ví dụ: 23:00 ngày 15/04/2026). Sau thời điểm này vendor không thể chỉnh sửa trên portal |
| Ngôn ngữ DO PDF | Chỉ tiếng Việt — toàn bộ DO PDF in ra đều dùng nhãn tiếng Việt |
| Nội dung DO PDF | PDF tiếng Việt — barcode PO, thông tin vendor, mã cửa hàng, ngày xác nhận PO, ngày giao hàng, bảng sản phẩm (barcode, tên, UoM, số lượng giao, cột trống để cửa hàng điền) |
| Xử lý UoM | Một UoM duy nhất cho mỗi dòng sản phẩm, kế thừa từ PO. Vendor chỉ điều chỉnh **giảm** số lượng theo **decimal precision của UoM**: **kg / lít / mét → 2 chữ số thập phân** (ví dụ 1.25 kg, 0.75 lít); **các UoM còn lại → 0 chữ số thập phân, chỉ số nguyên** (bao gồm gram, thùng, chai, cái, hộp, units…). Không thay đổi UoM. Nếu UoM sai, PO phải được tạo lại. **Lưu ý kỹ thuật:** Odoo đang config `decimal_precision` của Product UoM = 0.001 (3 chữ số), nhưng portal chủ động giới hạn input xuống 2 chữ số cho kg/lít/mét và 0 chữ số cho các UoM khác — đây là quy tắc UX phía portal |
| Xác nhận biên nhận | Cửa hàng xác nhận Receipt trong Odoo — qty_done finalized. **DO-LIVE-001:** DO status derive trực tiếp từ `stock.picking.state` (done → Done). Webhook `receipt-validated` ghi audit + gửi email vendor |
| Trả hàng | RPO = purchase.order với qty âm → tạo stock.picking loại Return (outgoing). Vendor **không cần xác nhận RPO**, chỉ chỉnh ngày nhận hàng và in RN PDF. **Số lượng giao mặc định = số lượng đặt** (không phải 0). **Hạn chót RN = ngày trả hàng dự kiến + 7 ngày dương lịch** (calendar days, thống nhất cùng cách tính với auto-cancel PO; không dùng cơ chế 23:00 cutoff như DO). Sau hạn chót, **Odoo tự xác nhận trả hàng** (scheduled action phía Odoo, không phải portal) → portal chỉ đọc picking `state = done` và reflect. Portal **không** proactively đẩy gì lên Odoo. **Thời điểm chạy scheduled action và logic cụ thể là dependency phía Odoo team — đang chờ xác nhận, TBD.** Note hiển thị: *"Nhà cung cấp vui lòng đến lấy hàng trước hạn chót (dd/mm/yyyy)"* (chỉ ngày, không có giờ) — **không** dùng text "Phiếu giao nhận bị khoá lúc …" như DO |
| Tự động huỷ PO | Nếu vendor không xác nhận hoặc từ chối trong vòng **7 ngày dương lịch** (calendar days, không trừ T7/CN/lễ) kể từ Expected Arrival date, PO tự động bị huỷ |
| Vendor xác nhận PO | 1 bước: Vendor chọn ngày giao → nhấp "Xác Nhận Đơn Đặt Hàng" → portal cập nhật `date_planned` + gọi `button_confirm` cùng lúc → Odoo tạo DO với `scheduled_date` = `date_planned` (không gửi email) |
| Vendor từ chối RFQ | NCC nhấp "Từ Chối" → popup điền lý do → xác nhận → portal push cancel về Odoo + log lý do + **email gửi Buyer (`purchase.order.user_id` — người mua phụ trách PO)** kèm lý do |
| 23:00 Cutoff DO | 23:00 đêm trước `scheduled_date` → DO chuyển Locked + job push `quantity_done` vào `stock.move.line` trên Odoo để pre-fill cho store. `scheduled_date` không cần push — Odoo đã tự set khi tạo DO từ `date_planned`. Admin mở khoá bằng cách reset `scheduled_date` trên Odoo |
| Lưu trữ dữ liệu | 24 tháng — PO cũ hơn 24 tháng bị xóa vĩnh viễn khỏi portal DB. Áp dụng cho mọi trạng thái |
| Xuất dữ liệu | Vendor có thể xuất dưới dạng PDF (riêng lẻ hoặc tổng hợp) hoặc CSV. Xuất đơn lẻ hoặc hàng loạt. Có bộ lọc theo khoảng ngày |
| Xử lý backorder | Vendor chỉ nộp qty — cửa hàng xác nhận và quyết định trong Odoo |
| Vai trò Admin | Tài khoản admin riêng biệt, mật khẩu do portal quản lý |
| Quyền Admin | Xem toàn bộ vendor, kích hoạt sync, xem toàn bộ PO/DO, mở khóa DO đã Locked, tải xuống bất kỳ PDF nào |
| Giao diện Admin | Cùng bố cục với vendor portal, có thêm các mục menu dành cho admin |
| Tìm kiếm danh sách PO | Lọc theo số PO và khoảng ngày |
| Tóm tắt dashboard Vendor | Số lượng PO theo trạng thái (Waiting, Confirmed, Canceled) hiển thị phía trên danh sách PO |
| Ghi chú của Vendor trên DO | Ghi chú tổng ở cấp phiếu giao hàng (header) — một ô text cho toàn bộ DO. Không có ghi chú riêng từng dòng sản phẩm |
| Lưu trữ PDF | PDF không lưu trên server — tạo on-the-fly mỗi lần vendor in |
| Thiết kế responsive | Hoạt động tốt trên cả desktop và mobile |
| Tài khoản Vendor | 1 Odoo partner = 1 tài khoản portal. Đăng nhập bằng Vendor ID (`res.partner.id`). Không hỗ trợ nhiều người dùng trên cùng một vendor |
| Thay đổi hồ sơ | Mọi thay đổi hồ sơ phải thực hiện qua Odoo — admin không thể chỉnh sửa trong portal |
| Ngôn ngữ Admin | Song ngữ (Tiếng Việt + Tiếng Anh) — giống vendor portal |
| Audit logging | Các hành động người dùng quan trọng được ghi lại: đăng nhập, xác nhận/từ chối PO, cập nhật/in/mở khóa DO, xác nhận biên nhận |
| Thông báo email | Mời dùng, đặt lại mật khẩu, **hủy PO (gửi Buyer)**, xác nhận biên nhận (gửi vendor kèm cảnh báo nếu có sai lệch), DO được mở khóa (gửi vendor) |
| Người nhận email cửa hàng | Email gửi đến **Buyer — người mua phụ trách PO trên Odoo (`purchase.order.user_id`)**. **Fallback:** nếu `user_id` trống thì gửi về email của `create_uid` (người đã tạo PO). Không bao giờ gửi đến hộp thư chung của cả nhóm |
| Dịch vụ email | AWS SES |

---

## Tổng Quan Kiến Trúc

```mermaid
flowchart TD
    classDef client   fill:#FEF3C7,stroke:#F59E0B,color:#451A03,font-weight:bold
    classDef infra    fill:#DBEAFE,stroke:#3B82F6,color:#1E3A5F,font-weight:bold
    classDef app      fill:#D1FAE5,stroke:#10B981,color:#064E3B,font-weight:bold
    classDef store    fill:#EDE9FE,stroke:#7C3AED,color:#2E1065,font-weight:bold
    classDef external fill:#FEE2E2,stroke:#EF4444,color:#7F1D1D,font-weight:bold

    Browser(["Trình duyệt Vendor / Admin"]):::client

    subgraph VM_PORTAL["Portal VM — Docker Compose"]
        Nginx["Nginx\nReverse Proxy + TLS termination"]:::infra
        React["React + Vite\nContainer Frontend\nphục vụ /"]:::app
        FastAPI["FastAPI\nContainer Backend\nphục vụ /api/"]:::app
        PG["PostgreSQL\nTài khoản vendor, token\nBản ghi DO, audit log"]:::app
        Redis["Redis\nRate limiting\nToken blacklist"]:::app
    end

    subgraph VM_ODOO["Odoo VM"]
        Odoo["Odoo 16 CE\nMua hàng, Kho\nDữ liệu Partner"]:::store
    end

    SES["AWS SES\nEmail đi"]:::external

    Browser -->|HTTPS| Nginx
    Nginx -->|"/"| React
    Nginx -->|"/api/"| FastAPI
    FastAPI --- PG
    FastAPI --- Redis
    FastAPI -->|"XML-RPC — đọc/ghi\nbutton_confirm, button_cancel\nqty_done, đồng bộ partner"| Odoo
    FastAPI -->|"HTTP session\nTải xuống PDF"| Odoo
    Odoo -.->|"Batch sync hằng đêm 23:00\nbiên nhận đã xác nhận"| FastAPI
    FastAPI -->|"Gửi email\nboto3"| SES
```

**Nguyên tắc chính:** Frontend React không bao giờ liên lạc trực tiếp với Odoo. Toàn bộ giao tiếp với Odoo được proxy qua backend FastAPI sử dụng một tài khoản dịch vụ duy nhất. Thông tin xác thực của vendor không bao giờ rời khỏi cơ sở dữ liệu nội bộ của portal.

---

> Chi tiết implementation flows (Auth, Sync, DO, Receipt) xem tại [Data Flow Summary](roadmap.md#data-flow-summary) trong roadmap.md.

---

## Nghiệp Vụ và Luồng Xử Lý

Phần này mô tả hành vi của portal theo ngôn ngữ nghiệp vụ, dành cho giao tiếp với các bên liên quan. Bao gồm bốn kịch bản chính: onboarding vendor mới, sử dụng portal hằng ngày, quy trình xác nhận giao hàng, và xử lý ngoại lệ.

---

### 1. Onboarding Vendor

**Điều kiện kích hoạt:** Một vendor mới được đăng ký trong Odoo với `is_vendor = True` **và** có địa chỉ email hợp lệ trên bản ghi partner.

**Những gì xảy ra tự động:**
1. Job đồng bộ portal chạy mỗi 6 giờ và phát hiện vendor mới trong Odoo
2. Một tài khoản portal được tạo, liên kết với Odoo ID của vendor.
3. Quản trị viên cần Kích Hoạt Tài khoản của nhà cung cấp từ portal. 
4. Nhà Cung cấp nhận được **Email Chào Mừng** (mặc định bằng tiếng Việt) bao gồm:
   - **Vendor ID** (`res.partner.id`) — số nguyên dùng để đăng nhập
   - Một **link đặt mật khẩu** có hiệu lực trong 24 giờ
5. Nhà cung cấp nhấp vào link, đặt mật khẩu của mình, và tài khoản trở nên hoạt động
6. Từ thời điểm này, vendor có thể đăng nhập bất kỳ lúc nào bằng ID và mật khẩu

**Nếu vendor bỏ lỡ cửa sổ 24h:** họ dùng tùy chọn "Quên mật khẩu" trên trang đăng nhập, nhập Vendor ID và nhận được link đặt lại mới.

**Nếu vendor không có email trong Odoo:** job đồng bộ bỏ qua họ và ghi log trường hợp này. Đội nhà phải thêm email vào partner Odoo — tài khoản sẽ được tạo vào chu kỳ đồng bộ tiếp theo.

**Cập nhật hồ sơ:** Nếu tên, số điện thoại, mã số thuế hoặc tên công ty của vendor thay đổi trong Odoo, portal phản ánh những thay đổi đó tự động vào lần đồng bộ tiếp theo. **Mật khẩu** của vendor chỉ được quản lý trên portal và không bao giờ bị ghi đè bởi sync. Vendor ID không bao giờ thay đổi — đây là `res.partner.id` cố định do Odoo cấp.


---

### 2. Xem Purchase Order

**Ai xem được gì:** Mỗi vendor chỉ thấy Purchase Order của chính mình. Về mặt kỹ thuật, vendor không thể xem dữ liệu của vendor khác.

**Những PO nào được hiển thị (trạng thái portal):**
- **Waiting/Chờ xác nhận** — RFQ đã được gửi cho vendor và đang chờ xác nhận. Vendor có thể xác nhận hoặc từ chối.
- **Confirmed/Đã xác nhận** — PO đã được duyệt, DO đã được tạo, Vendor có thể in PDF 
- **Canceled** — PO đã bị hủy (vendor từ chối, hoặc cửa hàng hủy). Chỉ đọc.

RFQ ở trạng thái Draft không được hiển thị. Vendor có thể xem dữ liệu PO trong **3 tháng** gần nhất.

**Xác nhận đơn đặt hàng:**
- Trang chi tiết PO ở trạng thái Waiting hiển thị ngày mong muốn giao hàng từ cửa hàng (`date_planned`)
- Trường "ngày dự kiến giao" **mặc định pre-fill bằng `date_planned` của cửa hàng** — NCC có thể giữ nguyên hoặc chọn lại (không quá 7 ngày sau `date_planned` của cửa hàng)
- NCC nhấp **"Xác Nhận Đơn Đặt Hàng"** → portal cập nhật `date_planned` trên `purchase.order` + gọi `button_confirm` cùng lúc → Odoo tạo DO tự động (`stock.picking`) với `scheduled_date` = `date_planned`
- Sau khi xác nhận, PO chỉ đọc, hiển thị link đến DO và cần phải liên hệ nhân viên mua hàng để được cập nhật PO sau khi xác nhận.

**Từ chối đơn đặt hàng:**
- Nút "Từ Chối" hiển thị cùng trang chi tiết PO ở trạng thái Waiting
- NCC nhấp "Từ Chối" → **popup yêu cầu điền lý do từ chối**
- NCC điền lý do và nhấp "Xác nhận" → portal push cancel về Odoo + **log lý do**
- Portal gửi email thông báo đến **Buyer** (`purchase.order.user_id`) kèm lý do từ chối
- Từ chối là cuối cùng — cửa hàng phải tạo RFQ mới nếu muốn đặt hàng lại. 

**Tự động hủy:**
- Nếu vendor không xác nhận hoặc từ chối PO ở trạng thái Waiting trong vòng **7 ngày dương lịch kể từ Expected Arrival date** (`date_planned` trên `purchase.order`), portal tự động hủy trong Odoo. 
- Một scheduled job kiểm tra hằng ngày các PO Waiting quá hạn
- Email thông báo gửi đến **cả vendor lẫn Buyer** (`purchase.order.user_id`) khi PO bị tự động hủy

**Tìm kiếm và lọc:**
- Vendor có thể tìm kiếm theo số PO (ví dụ: gõ "PO004" để lọc danh sách tức thì)
- Vendor có thể lọc theo khoảng ngày (ví dụ: "PO từ tháng Một đến tháng Ba")
- Kết quả được phân trang — 20 PO mỗi trang

**Xem chi tiết PO:** nhấp vào PO hiển thị toàn bộ danh sách sản phẩm đặt hàng với số lượng và ngày giao hàng dự kiến, cùng với DO liên kết và trạng thái của nó (Draft / Locked / Done / Canceled).

---

### 3. Delivery Order & Quy Trình Giao Hàng

Đây là quy trình nghiệp vụ cốt lõi của portal — từ RFQ đến giao hàng thực tế và xác nhận biên nhận.

```mermaid
flowchart TB
    classDef vendor fill:#D1FAE5,stroke:#10B981,color:#064E3B,font-weight:bold
    classDef portal fill:#EDE9FE,stroke:#7C3AED,color:#2E1065,font-weight:bold
    classDef store  fill:#DBEAFE,stroke:#3B82F6,color:#1E3A5F,font-weight:bold

    subgraph R1["① RFQ — Xác Nhận Ngày & PO"]
        direction LR
        S1(["Store tạo RFQ\n+ ngày mong muốn\nGửi email Vendor"]):::store
        V0{"NCC\nphản hồi?"}
        V_C(["Chọn ngày giao\n+ Xác nhận PO"]):::vendor
        V_R(["Từ chối PO\n+ điền lý do"]):::vendor
        V_A(["Không phản hồi\nsau 7 ngày"]):::portal
        P_X["PO bị huỷ\nEmail + lý do"]:::portal
        S1 --> V0
        V0 -- Xác nhận --> V_C
        V0 -- Từ chối --> V_R
        V0 -- Không phản hồi --> V_A
        V_R --> P_X
        V_A --> P_X
    end

    subgraph R2["② Delivery Order — Chỉnh qty & In"]
        direction LR
        P_DO["Odoo tự tạo DO\nNgày giao cố định từ PO"]:::portal
        V1(["NCC chỉnh qty\n(đúng decimal UoM)"]):::vendor
        P1["In DO PDF\n(nhiều lần)"]:::portal
        P_DO --> V1 --> P1
    end

    subgraph R3["③ Giao hàng & Xác nhận"]
        direction LR
        V3(["Vendor in 2 bản\nMang đến cửa hàng"]):::vendor
        D1(["Nhận hàng\nKý physical 2 bản"]):::store
        D2(["Cửa hàng xác nhận\ntrên Odoo"]):::store
        P2["DO: Done\nEmail xác nhận Vendor"]:::portal
        V3 --> D1 --> D2 --> P2
    end

    V_C --> P_DO
    P1  --> V3
```

> 🟢 Xanh lá = Vendor | 🔵 Xanh dương = Store | 🟣 Tím = Portal/System

#### 3.1 RFQ & PO Confirmation

- Store tạo RFQ trên Odoo với ngày mong muốn giao hàng (`date_planned`) — email tự động gửi cho Vendor
- Trên portal, trường ngày giao dự kiến **mặc định hiển thị `date_planned` của cửa hàng** — Vendor có thể giữ nguyên hoặc chọn lại ngày khác (không quá 7 ngày sau `date_planned` của cửa hàng) → nhấp **"Xác Nhận Đơn Đặt Hàng"** → portal cập nhật `date_planned` + gọi `button_confirm` cùng lúc → DO tự động tạo
- Nếu Vendor từ chối: popup lý do → PO bị huỷ trong Odoo + log lý do, email thông báo gửi **Buyer** (`purchase.order.user_id`) kèm lý do
- Nếu không phản hồi sau **7 ngày dương lịch** kể từ `date_planned`: hệ thống tự động huỷ PO, email gửi cả Vendor lẫn **Buyer**

#### 3.2 Delivery Order — Chỉnh qty & In

- Khi PO confirmed, **Odoo tự động tạo 1 DO** (stock.picking) — portal đọc qua XML-RPC
- `scheduled_date` trên DO = `date_planned` đã xác nhận — Odoo tự gán khi tạo DO, **ngày giao cố định, không chỉnh được trên DO**
- Portal đọc DO từ Odoo qua XML-RPC, **số lượng giao mặc định = số lượng đặt** (pre-fill đầy đủ, không phải 0), trạng thái **Draft**
- **Lưu ý về đồng bộ qty:** sau khi vendor đã lưu qty trên portal, nếu cửa hàng chỉnh `product_qty` trên PO Odoo (ví dụ tăng 10 → 12), portal **không** đọc lại để ghi đè giá trị đã lưu của vendor — vendor vẫn giữ nguyên qty họ đã nhập cho đến khi chính họ chỉnh sửa lại
- Vendor chỉnh sửa: **số lượng giao** từng dòng — chỉ được **chỉnh giảm**, **không được tăng vượt qty đặt** (hệ thống chặn cứng). Decimal theo UoM: **kg / lít / mét → 2 chữ số thập phân**, **các UoM khác (bao gồm gram, thùng, chai, cái, hộp, units) → 0 chữ số thập phân (số nguyên)**. Và **ghi chú tổng phiếu** (một ô text header, không có ghi chú riêng từng dòng SP)
- DO hiển thị **hạn chót** dưới dạng giờ + ngày cụ thể (ví dụ: 23:00 ngày 15/04/2026) — là thời điểm DO bị khoá, vendor không thể chỉnh sửa sau đó
- **Không cần ký điện tử** — vendor in DO PDF trực tiếp từ giao diện
- Vendor có thể **lưu nhiều lần, in nhiều phiên bản** — tất cả khi DO vẫn ở Draft
- PDF không lưu trên server — mỗi lần in luôn tạo phiên bản mới nhất

#### 3.3 23:00 Cutoff — Khoá DO & Push Odoo

- Lúc **23:00 đêm trước ngày giao dự kiến**: DO bị khóa trên giao diện web, các thông tin đã được cập nhật theo thời gian thực trước 23PM . Vendor không thể chỉnh sửa sau thời điểm này.
- Ngày giao cố định từ PO — NCC đã chọn khi xác nhận PO và không thể đổi trên DO.
- DO bị khoá — vendor không thể chỉnh sửa nhưng vẫn có thể in PDF.

#### 3.4 Giao hàng thực tế

- Vendor in 2 bản PDF DO (phiên bản cuối cùng sau khi Locked)
- (ngoài hệ thống) Vendor mang hàng + 2 bản DO đến cửa hàng
- (ngoài hệ thống) Cả hai bên ký tay 2 bản paper — mỗi bên giữ 1 bản làm chứng từ

#### 3.5 Receipt Confirmation — Xác nhận nhận hàng

- Cửa hàng xem Receipt trong Odoo — qty đã được pre-fill từ DO (có thể điều chỉnh nếu hàng hư hỏng)
- Cửa hàng xác nhận Receipt → qty_done finalized, stock move được tạo trên Odoo
- ~~Nightly sync 23:00 → portal cập nhật DO status → **Done**, lưu qty thực nhận~~ Bước trên. 
- Email tự động gửi Vendor xác nhận — nếu qty nhận khác với DO sẽ có cảnh báo chi tiết. 
- ~~**Trường hợp vendor chưa dùng portal** (DO vẫn Draft): portal tự động chuyển DO → Done khi phát hiện Receipt đã confirm~~ Ngoài phạm vi hệ thống.   

> Sơ đồ toàn bộ flow (bao gồm Returns, Unlock DO) xem tại [PROCESS_FLOW.md](PROCESS_FLOW.md).

---

### 4. Vòng Đời Trạng Thái PO và DO

**Trạng thái PO (portal quản lý):**

| Trạng thái PO trên Portal | Trạng thái PO Odoo | Điều kiện kích hoạt | Vendor có thể làm |
|---|---|---|---|
| **Waiting** | `sent` | Cửa hàng gửi RFQ | Xác nhận hoặc Từ chối |
| **Confirmed** | `purchase` | Vendor xác nhận PO | Xem DO, xuất dữ liệu |
| **Canceled** | `cancel` | Vendor từ chối hoặc cửa hàng hủy | Chỉ đọc |

**Trạng thái DO (portal quản lý):**

| Trạng thái DO | Điều kiện kích hoạt | Vendor có thể làm |
|---|---|---|
| **Draft** | PO confirmed, DO được tạo tự động (pre-fill qty từ PO) | Sửa qty + ghi chú tổng phiếu, lưu/in nhiều lần |
| **Locked** | 23:00 cutoff đêm trước ngày giao hàng → push `quantity_done` vào `stock.move.line` trên Odoo | Chỉ đọc, in PDF phiên bản cuối |
| **Done** | Nightly sync phát hiện cửa hàng đã confirm Receipt trong Odoo | Chỉ đọc, xem qty thực nhận bên cạnh qty giao, xuất PDF/CSV |
| **Canceled** | PO bị hủy (trước khi Receipt được xác nhận) | Chỉ đọc |

**Quy tắc hủy:**
- Một PO chỉ có thể bị hủy nếu Receipt liên kết **chưa** được cửa hàng xác nhận (không có `qty_done`)
- Nếu Receipt đã được xác nhận trong Odoo, việc hủy bị chặn — portal hiển thị cảnh báo và vendor phải liên hệ bộ phận mua hàng để giải quyết

---

### 5. Khoá và Mở Khoá DO

**23:00 Cutoff — tại sao khoá DO:**
Lúc 23:00 đêm trước ngày giao hàng (`scheduled_date`), DO bị khoá tự động. Mục đích:
1. Xác định số lượng cuối cùng vendor sẽ giao
2. ~~Push `quantity_done` vào `stock.move.line` trên Odoo để pre-fill cho store — cửa hàng không phải nhập liệu thủ công~~ Được cập nhật theo thời gian thực 
3. Cửa hàng vẫn có thể điều chỉnh qty trên Purchase Order của odoo trước khi confirm (ví dụ: hàng hư hỏng). 

**Ý nghĩa thực tế của "Locked":**
- Tất cả các trường số lượng trên DO đều chỉ đọc trong portal
- Vendor vẫn có thể in PDF phiên bản cuối
- Vendor không thể chỉnh sửa sau hạn chót

**Trước khi Locked (Draft):** vendor tự do sửa/in nhiều lần.

**Quy trình mở khoá:**
Nếu số lượng nhập sai và vendor cần nộp lại, portal admin có thể mở khóa DO Locked trực tiếp từ phần admin (`/admin/vendors/:id`). Việc mở khóa sẽ:
1. Chuyển DO status từ Locked → Draft
2. Gửi email thông báo đến vendor
3. Vendor có thể cập nhật số lượng và ghi chú tổng phiếu, rồi in lại
4. Hành động mở khóa được ghi vào audit log

---

### 6. Tóm Tắt Thông Báo Email

| Sự kiện | Người nhận | Ngôn ngữ | Nội dung |
|---|---|---|---|
| Tài khoản vendor mới được tạo | Vendor | Tiếng Việt (mặc định) | Vendor ID (số nguyên) + link đặt mật khẩu |
| Yêu cầu đặt lại mật khẩu | Vendor | Ngôn ngữ ưa thích của vendor | Link đặt lại (hết hạn sau 24h) |
| Vendor từ chối RFQ | **Buyer** (`purchase.order.user_id`) | Tiếng Việt | PO bị từ chối + hủy trong Odoo |
| PO tự động hủy (7 ngày) | Vendor + **Buyer** | Ngôn ngữ ưa thích của vendor / Tiếng Việt | PO tự động hủy do không phản hồi trong 7 ngày kể từ Expected Arrival |
| Cửa hàng xác nhận biên nhận | Vendor | Ngôn ngữ ưa thích của vendor | Thông báo xác nhận biên nhận. Cảnh báo nếu có sai lệch số lượng giữa DO và biên nhận |
| RPO được cửa hàng tạo | Vendor | Gửi bởi Odoo (Send by Email) | Thông báo đơn trả hàng — vendor nên đăng nhập portal để xác nhận |
| DO được admin mở khóa | Vendor | Ngôn ngữ ưa thích của vendor | Thông báo DO đã được mở khóa để chỉnh sửa lại |

**Lưu ý:** Không gửi email khi vendor xác nhận PO (xác nhận được đẩy lên Odoo theo thời gian thực) hoặc khi vendor in DO/RN PDF. Dữ liệu DO/RN được push lên Odoo tự động qua 23:00 cutoff. **Người nhận email từ cửa hàng luôn là Buyer — người mua phụ trách PO trong Odoo (`purchase.order.user_id`); nếu `user_id` trống, fallback về email của `create_uid`. Không bao giờ gửi đến hộp thư chung.** Email RPO được Odoo gửi nội bộ (không phải do portal gửi).

---

### 7. Những Gì Portal KHÔNG Làm

Điều quan trọng không kém là các bên liên quan cần hiểu rõ phạm vi của portal:

- **Không xác nhận biến động hàng tồn kho** — mọi xác nhận trong Odoo do cửa hàng thực hiện trong Odoo
- **Không tạo Purchase Order** — vendor có thể xác nhận hoặc từ chối Sent RFQ nhưng không thể tạo PO hoặc chỉnh sửa dòng PO
- **Không xử lý hóa đơn hay thanh toán** — nằm ngoài phạm vi của portal này (vendor có thể xuất dữ liệu cho mục đích lập hóa đơn của riêng mình)
- **Không quản lý backorder** — cửa hàng quyết định về backorder trong Odoo sau khi xem xét số lượng
- **Không thay đổi hành vi gốc của Odoo** — các trạng thái và quy trình của Odoo không bị thay đổi
- **Không để lộ thông tin xác thực Odoo cho vendor** — vendor không có quyền truy cập Odoo, dù trực tiếp hay gián tiếp
- **Không cho phép vendor xem dữ liệu của vendor khác** — được thực thi ở mọi tầng của hệ thống

---

### 8. Chi Tiết Giao Diện (UI Specifications)

#### 8.1 PO List View (Danh sách Đơn Đặt Hàng)

| Cột | Nguồn dữ liệu | Ghi chú |
|---|---|---|
| Số đơn hàng | `purchase.order.name` | |
| Ngày đặt hàng | `purchase.order.date_order` | |
| Ngày giao hàng (dự kiến) | `purchase.order.date_planned` | |
| **Warehouse (địa điểm giao)** | `stock.picking.picking_type_id` → `stock.warehouse.name` | **Mới** — tên địa điểm giao hàng |
| Trạng thái | Portal PO status (Waiting / Confirmed / Canceled) | Badge màu |
| Tổng tiền | `purchase.order.amount_total` | |

**Tính năng trên PO List:**
- **Tick chọn** một hoặc nhiều phiếu PO → xuất ra file **PDF** hoặc **CSV**
- **Sort** theo tên (số PO)
- **Filter** theo: ngày giao hàng, trạng thái phiếu PO
- Phân trang: 20 PO/trang

#### 8.2 PO Form View (Chi tiết Đơn Đặt Hàng)

**Thông tin chung:**
- Bổ sung **Warehouse** (địa điểm giao hàng) hiển thị trên PO

**Bảng danh sách sản phẩm — các cột:**

| Cột | Nguồn dữ liệu | Ghi chú |
|---|---|---|
| Tên sản phẩm | `purchase.order.line.name` | Dùng field `name` trực tiếp (không phải `product_id.name`) |
| Barcode | `product.product.barcode` | **Mới** |
| Mã NCC | `product.supplierinfo.product_code` | **Mới** — mã sản phẩm của nhà cung cấp |
| Tên SP NCC | `product.supplierinfo.product_name` | **Mới** — tên sản phẩm theo nhà cung cấp |
| Số lượng | `purchase.order.line.product_qty` | |
| Đơn giá | `purchase.order.line.price_unit` | |
| Discount | `purchase.order.line.discount` | **Mới** — phần trăm chiết khấu |
| Thuế (VAT) | `purchase.order.line.taxes_id` | Hiển thị: VAT 5%, 8%, 10% giống giao diện Odoo |
| Thành tiền (-V) | `purchase.order.line.price_subtotal` | **Đổi tên** — "Thành tiền" → "Thành tiền (-V)" (chưa VAT) |
| Tổng tiền (+V) | `purchase.order.line.price_total` | **Đổi tên** — "Tổng tiền" → "Tổng tiền (+V)" (đã VAT) |

> Cột **Thành tiền (-V)** và **Tổng tiền (+V)** nằm cuối cùng trong bảng.

**Nút hành động:**
1. Hiển thị ngày mong muốn giao hàng từ cửa hàng (`date_planned`). Trường ngày dự kiến giao **mặc định pre-fill bằng `date_planned` của cửa hàng**; NCC có thể giữ nguyên hoặc chọn lại (không quá 7 ngày sau `date_planned`). Nhấp **"Xác Nhận Đơn Đặt Hàng"** → portal cập nhật `date_planned` + gọi `button_confirm` cùng lúc
2. Nhấp **"Từ Chối"** → popup điền lý do → xác nhận → cancel + log lý do
3. Sau khi xác nhận PO → hiển thị link đến phiếu giao hàng (DO)

#### 8.3 DO Form View (Phiếu Giao Hàng)

**Bảng danh sách sản phẩm — các cột:**

| Cột | Nguồn dữ liệu | Ghi chú |
|---|---|---|
| Tên sản phẩm | `purchase.order.line.name` | Giống PO — dùng field `name` |
| Barcode | `product.product.barcode` | Giống PO |
| Mã NCC | `product.supplierinfo.product_code` | Giống PO |
| Tên SP NCC | `product.supplierinfo.product_name` | Giống PO |
| Số lượng đặt | `purchase.order.line.product_qty` | Qty từ PO (read-only) |
| Số lượng giao | `stock.move.line.quantity_done` | **Mặc định = số lượng đặt** khi PO confirmed. Editable (chỉ chỉnh giảm, không tăng vượt qty đặt). Decimal theo UoM: **kg / lít / mét → 2 chữ số thập phân**, các UoM khác (gram, thùng, chai, cái, hộp, units…) → số nguyên |

**Ghi chú phiếu:** Portal DB — `delivery_orders.note` — một ô text ở **header DO**, vendor điền khi ở trạng thái Draft, áp dụng cho toàn bộ phiếu. Không có ghi chú riêng từng dòng SP.

**Thay đổi quan trọng:**
- **Bỏ phần Ký phiếu giao nhận** — không cần chữ ký điện tử trên Portal
- **Chỉ cần In file PDF** — vendor nhấn nút In/Tải PDF trực tiếp
- **Ghi chú ở cấp phiếu (header)** — một ô text duy nhất cho toàn bộ DO, không có ghi chú riêng từng dòng SP

**Ngày giao hàng trên DO:**
- Ngày giao **cố định** = `scheduled_date` được Odoo gán tự động khi NCC xác nhận PO
- NCC **không chỉnh được** ngày giao trên DO — chỉ chỉnh qty
- **Hạn chót** hiển thị dưới dạng giờ + ngày cụ thể (23:00 ngày DD/MM/YYYY) — thời điểm DO bị khoá

#### 8.4 RPO / RN View (Đơn Trả Hàng)

- Giao diện và tính năng **tương tự PO** (cùng các cột sản phẩm: tên SP, barcode, mã NCC, tên SP NCC)
- Vendor **không cần bấm "Xác Nhận"** đơn RPO — chỉ chỉnh **ngày nhận hàng**
- **Số lượng giao mặc định = số lượng đặt** (pre-fill đầy đủ, không phải 0)
- **Hạn chót RN = ngày trả hàng dự kiến + 7 ngày dương lịch** (khác với DO dùng 23:00 cutoff đêm trước ngày giao)
- Sau hạn chót → **Odoo tự xác nhận trả hàng** (scheduled action phía Odoo validate `stock.picking`) → portal đọc lại state `done` từ Odoo và cập nhật RN sang trạng thái Done. Portal **không** chủ động đẩy validate lên Odoo
- ⚠️ **Dependency phía Odoo:** scheduled action tự validate picking trả hàng sau hạn chót hiện chưa có trên Odoo 16 CE standard. Cần **Odoo team xác nhận module/cron sẽ implement** — logic cụ thể và thời điểm chạy đang chờ xác nhận (TBD)
- Banner/note trên RN hiển thị: **"Nhà cung cấp vui lòng đến lấy hàng trước hạn chót (dd/mm/yyyy)"** — không dùng text "Phiếu giao nhận bị khoá lúc …" như DO
- In RN PDF giống DO (không cần ký)

---

> Chi tiết kỹ thuật về cơ chế sync Odoo ↔ Portal xem tại [Odoo ↔ Portal Sync](roadmap.md#odoo--portal-sync--open-questions--pending-it-confirmation) trong roadmap.md.

---

> Các phase triển khai, DB schema, API endpoints, và lưu ý cho developer xem tại [roadmap.md](roadmap.md).

---

### Các Câu Hỏi Chưa Giải Quyết ==> Đã giải quyết 

| # | Chủ đề | Chi tiết | Trạng thái |
|---|---|---|---|
| 1 | **Quên mật khẩu — cập nhật email** | Vendor đăng nhập bằng Vendor ID. Khi quên mật khẩu, link reset gửi qua email (lấy từ Odoo lúc tạo tài khoản). Nếu email thực tế thay đổi — ai cập nhật email trên portal? Vendor tự sửa? Admin sửa? Hay sync lại từ Odoo? | Portal gửi 1 link reset mật khẩu, đối với Email thì phải cập nhật trên odoo với admin, portal không được phép. Vendor không có quyền sửa thông tin của mình trên Portal. Điều này đã xác định trong tài liệu này ở confirmed decision |
| 2 | **`is_vendor` hay `supplier_rank > 0`?** | Docs hiện ghi `is_vendor = True`, nhưng Odoo 16 CE standard dùng `supplier_rank > 0` để đánh dấu vendor. Trường `is_vendor` không phải standard field. Cần xác nhận trường nào đang dùng trên instance thực tế | Xác Nhận L **`is_vendor`** là field chính thức trên Odoo hiện tại do một module customisation. không sử dụng **`supplier_rank`**|
| 3 | **Ngày giao hàng trên DO vs PO** | Ngày giao trên DO có chỉnh được không? | ✅ **Đã xác nhận: Ngày giao trên DO cố định.** NCC chọn ngày giao khi xác nhận PO (không quá 7 ngày sau `date_planned` của cửa hàng) — portal cập nhật `date_planned` + `button_confirm` cùng lúc. Odoo tạo DO với `scheduled_date` = `date_planned`. NCC không chỉnh ngày trên DO, chỉ chỉnh qty |

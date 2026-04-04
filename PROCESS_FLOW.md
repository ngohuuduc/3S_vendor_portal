# 3SACH Vendor Portal — Process Flow

---

## Business Overview (Simple View)

```mermaid
flowchart LR
    classDef store    fill:#DBEAFE,stroke:#3B82F6,color:#1E3A5F,font-weight:bold
    classDef vendor   fill:#D1FAE5,stroke:#10B981,color:#064E3B,font-weight:bold
    classDef portal   fill:#EDE9FE,stroke:#7C3AED,color:#2E1065,font-weight:bold
    classDef internal fill:#FEF3C7,stroke:#F59E0B,color:#451A03,font-weight:bold
    classDef decision fill:#FFF,stroke:#6B7280,color:#111827

    S1([🏪 Cửa hàng\ntạo RFQ\ntrên Odoo]):::store
    S2([📧 Gửi RFQ\ncho nhà cung cấp]):::store

    V1([✅ Nhà cung cấp\nxác nhận PO\ntrên Portal]):::vendor
    V2([📦 Điền số lượng\nvà ngày giao\ntrên Portal]):::vendor
    V3([🚚 Giao hàng\nthực tế\ncho kho]):::vendor
    V4([✍️ Ký xác nhận\ntrên Portal\nvà tải PDF]):::vendor

    P1{{Portal tự động\nkhoá DO\nlúc 23:00}}:::portal
    P2{{Portal tạo\nPDF có chữ ký\nvà gửi email}}:::portal

    I1([🔍 Nội bộ\nkiểm tra\nsố lượng]):::internal
    I2([✔️ Xác nhận\nnhập kho\ntrên Odoo]):::internal

    S1 --> S2 --> V1
    V1 --> V2
    V2 --> P1
    P1 --> V3
    V3 --> V4
    V4 --> P2
    P2 --> I1
    I1 --> I2
    I2 --> DONE([🎉 Hoàn tất]):::store
```

> **Màu sắc:** 🔵 Cửa hàng &nbsp;|&nbsp; 🟢 Nhà cung cấp &nbsp;|&nbsp; 🟣 Hệ thống Portal &nbsp;|&nbsp; 🟡 Nội bộ 3Sach

---

## Full Purchase Workflow: RFQ → PO Confirm → Delivery → Receipt

```mermaid
flowchart TD
    subgraph ONBOARD["🟦 Vendor Onboarding"]
        A1([New vendor added in Odoo\nsupplier_rank > 0]) --> A2[Sync job runs every 6h]
        A2 --> A3{Has email?}
        A3 -- No --> A4[Skip & log for manual follow-up]
        A3 -- Yes --> A5[Create portal account\ninactive, no password]
        A5 --> A6[Send Welcome Email via AWS SES\nVendor ID + set-password link 24h]
        A6 --> A7[Vendor sets password\nAccount becomes active]
    end

    subgraph RFQ_PO["🟨 RFQ → PO Confirmation  ·  Cửa hàng / Vendor Portal"]
        B1([Cửa hàng tạo RFQ trên Odoo]) --> B2[Sent RFQ\nEmail gửi cho NCC]
        B2 --> B3{Vendor xác nhận PO\ntrước Order Deadline?}
        B3 -- No --> B3X([PO không được xác nhận\nRFQ hết hạn])
        B3 -- Yes --> B4[Vendor clicks 'Confirm PO'\non Vendor Portal]
        B4 --> B5[Portal calls button_confirm\non purchase.order via XML-RPC]
        B5 --> B6[PO Confirmed\nOdoo cập nhật trạng thái: sent → purchase]
    end

    subgraph DO["🟧 Delivery Order  ·  Vendor Portal"]
        C1[DO sinh ra tự động\nIn và sửa được] --> C2[NCC chỉnh DO trên portal\nNgày giao + số lượng\nSL giao ≤ SL đặt]
        C2 --> C3{Trước 23:00\nđêm trước ngày giao?}
        C3 -- Yes --> C4[DO vẫn có thể chỉnh sửa]
        C4 --> C3
        C3 -- No / 23:00 CUTOFF --> C5[Hệ thống xác nhận SL giao\nKhóa chỉnh sửa trên Vendor Portal]
        C5 --> C6[DO bị khóa\nKhông sửa được]
        C5 --> C7[Push vào Odoo\nCập nhật Receipt: ngày + SL dự kiến]
    end

    subgraph DELIVERY["🟩 Physical Delivery  ·  Kho nhận"]
        D1[Kho nhận DO\nXác nhận SL] --> D2[NCC giao hàng\nKèm 2 tờ DO]
        D2 --> D3[Ký xác nhận 2 tờ DO\nmỗi bên giữ 1 tờ]
    end

    subgraph PORTAL_CONFIRM["🟪 Receipt Confirmation  ·  Vendor Portal"]
        E1[Vendor mở Receipt\ntrạng thái: Ready / assigned] --> E2[Vendor nhập qty_done\nthực tế từng sản phẩm\ncó thể lưu nhiều lần]
        E2 --> E3[Vendor clicks 'Sign & Confirm']
        E3 --> E4[Vendor ký tên trên màn hình\nsignature_pad.js → PNG]
        E4 --> E5[Portal tạo signed PDF\nOdoo delivery slip + trang xác nhận\nWeasyPrint + pypdf]
        E5 --> E6[Receipt bị khóa trên portal\nKhông chỉnh sửa được]
        E6 --> E7A[Email xác nhận → Vendor\nkèm PDF đính kèm]
        E6 --> E7B[Email cảnh báo → Internal team\nVendor name, PO ref, link Odoo]
    end

    subgraph ODOO_VALIDATE["🟥 Odoo Validation  ·  Internal Team"]
        F1[Internal team review\nsố lượng trong Odoo] --> F2{Số lượng OK?}
        F2 -- Yes --> F3[Admin validates Receipt trong Odoo]
        F2 -- No / Short --> F4[Admin điều chỉnh + quyết định backorder]
        F4 --> F3
        F3 --> F5([Receipt Done\nHoàn tất nhập kho ✓])
    end

    subgraph UNLOCK["🔓 Exception: Unlock Receipt"]
        G1([Vendor nhập sai số lượng]) --> G2[Portal Admin mở khóa\ntại /admin/vendors/:id]
        G2 --> G3[Unlock ghi vào receipt_locks\nunlocked_at + unlocked_by]
        G3 --> G4[Vendor cập nhật qty_done\nvà ký lại]
        G4 --> E3
    end

    %% Connect major phases
    A7 --> B2
    B6 --> C1
    C6 --> D1
    C7 --> E1
    D3 --> E1
    E6 --> F1
    E6 -.->|"Signed / Locked state"| UNLOCK
```

---

## Swimlane View (4 Actors)

```mermaid
sequenceDiagram
    actor Vendor
    participant Portal as Vendor Portal
    participant Odoo as Odoo (XML-RPC)
    participant Kho as Kho nhận
    participant Team as Internal Team

    Note over Vendor,Team: ── Phase 1: RFQ & PO Confirmation ──
    Odoo->>Vendor: Sent RFQ email (NCC)
    Vendor->>Portal: Confirm PO (trước Order Deadline)
    Portal->>Odoo: button_confirm on purchase.order
    Odoo-->>Portal: PO state: sent → purchase
    Odoo-->>Portal: DO / Receipt created (assigned)

    Note over Vendor,Team: ── Phase 2: Delivery Order Edit ──
    Portal->>Vendor: DO available — editable
    Vendor->>Portal: Chỉnh DO: ngày giao + SL giao (≤ SL đặt)
    Note over Portal: 23:00 CUTOFF — đêm trước ngày giao
    Portal->>Portal: Lock DO, confirm SL giao
    Portal->>Odoo: Push Receipt: ngày + SL dự kiến

    Note over Vendor,Team: ── Phase 3: Physical Delivery ──
    Portal->>Kho: Gửi DO (2 bản)
    Vendor->>Kho: Giao hàng kèm 2 tờ DO
    Kho->>Vendor: Ký xác nhận — mỗi bên 1 tờ

    Note over Vendor,Team: ── Phase 4: Portal Receipt Confirmation ──
    Vendor->>Portal: Mở Receipt (Ready), nhập qty_done
    Vendor->>Portal: Sign & Confirm + vẽ chữ ký
    Portal->>Portal: Tạo signed PDF (Odoo slip + chữ ký)
    Portal->>Odoo: (Receipt vẫn assigned — chờ admin validate)
    Portal->>Vendor: Email xác nhận + PDF đính kèm
    Portal->>Team: Email cảnh báo: vendor, PO ref, link Odoo

    Note over Vendor,Team: ── Phase 5: Odoo Validation ──
    Team->>Odoo: Review qty, validate Receipt
    Odoo-->>Portal: Receipt state: done
    Note over Odoo: Hoàn tất nhập kho ✓
```

---

## Receipt State Machine

```mermaid
stateDiagram-v2
    [*] --> Ready : PO confirmed,\nDO locked after cutoff

    Ready --> Signed : Vendor nhập qty_done\nvà ký tên

    Signed --> Ready : Portal Admin unlock\n(logged in receipt_locks)

    Signed --> Done : Internal team\nvalidates in Odoo

    Done --> [*]

    note right of Ready
        Odoo state: assigned
        Vendor: có thể nhập qty_done
        Portal: editable
    end note

    note right of Signed
        Odoo state: assigned (pending)
        Vendor: read-only, có thể tải PDF
        Portal: locked
    end note

    note right of Done
        Odoo state: done
        Vendor: read-only, có thể tải PDF
        Portal: locked
    end note
```

---

**Glossary**

| Term | Meaning |
|---|---|
| NCC | Nhà cung cấp (Vendor) |
| DO | Delivery Order |
| SL | Số lượng (Quantity) |
| RFQ | Request for Quotation |
| PO | Purchase Order |
| Receipt | Phiếu nhập kho Odoo |
| Cutoff | 23:00 đêm trước ngày giao — DO bị khóa sau mốc này |

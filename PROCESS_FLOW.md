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

    S1([Store creates\nRFQ in Odoo]):::store
    S2([RFQ sent\nto vendor]):::store

    D1{Vendor confirms\nor rejects?}:::decision
    D1_NO([Vendor rejects RFQ\non Portal]):::vendor
    D1_CANCEL([Portal cancels RFQ\nin Odoo + email\nto PO creator]):::portal

    V1([Vendor confirms PO\non Portal]):::vendor
    V2([DO auto-created\nVendor edits qty + date]):::vendor
    V3([Vendor signs DO\ndigitally on Portal]):::vendor
    V4([Print DO PDF\nwith signature + barcode]):::vendor
    V5([Deliver goods\nto store with paper DO]):::vendor

    P1{{Portal pushes\nqty + date to Odoo\nReceipt}}:::portal

    I1([Store receives goods\nconfirms Receipt in Odoo]):::internal
    I2{{Portal notified\nDO status -> Done}}:::portal
    I3([Vendor sees final\nqty on Portal\nCan export PDF/CSV]):::vendor

    S1 --> S2 --> D1
    D1 -- Reject --> D1_NO --> D1_CANCEL
    D1 -- Confirm --> V1 --> V2 --> V3 --> V4 --> V5
    V3 --> P1
    V5 --> I1 --> I2 --> I3
```

> **Colours:** Blue = Store | Green = Vendor | Purple = Portal System | Yellow = Internal 3Sach

---

## Full Purchase Workflow: RFQ -> PO Confirm -> DO -> Delivery -> Receipt

```mermaid
flowchart TD
    subgraph ONBOARD["Vendor Onboarding"]
        A1([New vendor added in Odoo\nsupplier_rank > 0]) --> A2[Sync job runs every 6h]
        A2 --> A3{Has email?}
        A3 -- No --> A4[Skip & log for manual follow-up]
        A3 -- Yes --> A5[Create portal account\ninactive, no password]
        A5 --> A6[Send Welcome Email via AWS SES\nEmail login + set-password link 24h]
        A6 --> A7[Vendor sets password\nAccount becomes active]
    end

    subgraph RFQ_PO["RFQ -> PO Confirmation / Rejection"]
        B1([Store creates RFQ in Odoo]) --> B2[Sent RFQ\nEmail sent to vendor]
        B2 --> B3{Vendor action\non Portal?}
        B3 -- Confirm --> B4[Vendor clicks Confirm PO\non Vendor Portal]
        B4 --> B5[Portal calls button_confirm\non purchase.order via XML-RPC]
        B5 --> B6[PO Confirmed in Odoo\nstate: sent -> purchase\nNo email sent]
        B3 -- Reject --> B7[Vendor clicks Reject\non Vendor Portal]
        B7 --> B8[Portal calls button_cancel\non purchase.order via XML-RPC]
        B8 --> B9[RFQ Cancelled in Odoo\nstate: sent -> cancel]
        B9 --> B10[Email to PO creator\nRFQ rejected by vendor]
        B3 -- No action --> B3X([RFQ expires\nif Order Deadline passes])
    end

    subgraph DO["Delivery Order — Vendor Portal"]
        C1[1 DO auto-created per confirmed PO\nStatus: Draft] --> C2[Vendor edits DO on portal\nSingle delivery date + quantities\nQty <= ordered qty]
        C2 --> C3[Vendor clicks Sign DO\nDraws digital signature\nOptional comment]
        C3 --> C4[DO status: Signed\nLocked — no further edits]
        C4 --> C5[Portal pushes to Odoo\nDelivery date + set quantities\ninto Receipt]
        C4 --> C6[Vendor can print DO PDF\nVietnamese, includes barcode\nCan print multiple times]
    end

    subgraph DELIVERY["Physical Delivery — Store"]
        D1[Vendor brings printed DO\nto store with goods] --> D2[Store receives goods\nChecks quantities]
        D2 --> D3[Both parties sign\n2 paper copies of DO\nEach keeps 1 copy]
    end

    subgraph STORE_CONFIRM["Receipt Confirmation — Odoo"]
        E1[Store reviews Receipt in Odoo\nCan adjust set quantities] --> E2[Store confirms Receipt\nqty_done is finalized]
        E2 --> E3[Portal notified\nDO status -> Done]
        E3 --> E4[Email to vendor:\nReceipt confirmed\nAlerts if qty differs]
    end

    subgraph UNLOCK["Exception: Unlock DO"]
        G1([Vendor entered wrong quantities]) --> G2[Portal Admin unlocks DO\nDO status -> Draft]
        G2 --> G3[Email notification sent to vendor]
        G3 --> G4[Vendor updates DO\nand re-signs]
        G4 --> C3
    end

    subgraph RETURNS["Returns Flow"]
        R1([Store creates RPO in Odoo]) --> R2[RPO appears on portal\nwith Goods Return Note]
        R2 --> R3[Similar workflow to PO/DO]
    end

    %% Connect major phases
    A7 --> B2
    B6 --> C1
    C6 --> D1
    D3 --> E1
    C4 -.->|"Signed / Locked state"| UNLOCK
```

---

## Swimlane View (4 Actors)

```mermaid
sequenceDiagram
    actor Vendor
    participant Portal as Vendor Portal
    participant Odoo as Odoo (XML-RPC)
    participant Store as Store (Odoo only)

    Note over Vendor,Store: ── Phase 1: RFQ & PO Confirmation ──
    Odoo->>Vendor: Sent RFQ email
    alt Vendor confirms
        Vendor->>Portal: Click "Confirm PO"
        Portal->>Odoo: button_confirm on purchase.order
        Odoo-->>Portal: PO state: sent -> purchase
        Note over Portal: No email sent on confirm
    else Vendor rejects
        Vendor->>Portal: Click "Reject"
        Portal->>Odoo: button_cancel on purchase.order
        Odoo-->>Portal: PO state: sent -> cancel
        Portal->>Store: Email to PO creator: RFQ rejected
    end

    Note over Vendor,Store: ── Phase 2: Delivery Order (Portal) ──
    Portal->>Vendor: DO available (Draft) — editable
    Vendor->>Portal: Edit DO: single delivery date + quantities (qty <= ordered)
    Vendor->>Portal: Sign DO digitally + optional comment
    Portal->>Portal: Lock DO (status: Signed)
    Portal->>Odoo: Push delivery date + set quantities into Receipt
    Portal->>Vendor: DO PDF available for printing (Vietnamese, with PO barcode)
    Note over Portal: No email sent on sign

    Note over Vendor,Store: ── Phase 3: Physical Delivery ──
    Vendor->>Store: Deliver goods with printed DO (2 copies)
    Store->>Vendor: Both sign paper copies — each keeps 1

    Note over Vendor,Store: ── Phase 4: Store Receipt Confirmation (Odoo) ──
    Store->>Odoo: Review Receipt, adjust quantities if needed
    Store->>Odoo: Confirm Receipt (qty_done finalized)
    Odoo->>Portal: Receipt validated (sync mechanism TBD)
    Portal->>Portal: DO status -> Done, final received qty stored
    Portal->>Vendor: Email: receipt confirmed (alerts if any qty differs)

    Note over Vendor,Store: ── Phase 5: Export ──
    Vendor->>Portal: Export PDF or CSV for invoicing

    Note over Vendor,Store: ── Exception: Unlock DO ──
    Portal->>Vendor: Admin unlocks DO (status -> Draft) + email notification
    Vendor->>Portal: Re-edit DO quantities and re-sign
```

---

## PO Status Mapping: Portal vs Odoo

Portal and Odoo maintain **different status labels**. Odoo's base behaviour is never modified.

| Portal PO Status | Odoo State | Trigger | Vendor can do |
|---|---|---|---|
| **Waiting** | `sent` | Store sends RFQ | Confirm or Reject |
| **Confirmed** | `purchase` | Vendor confirms PO on portal | View DO, export data |
| **Cancelled** | `cancel` | Vendor rejects, or store cancels | Read-only |

---

## DO State Machine

```mermaid
stateDiagram-v2
    [*] --> Draft : PO confirmed,\nDO auto-created

    Draft --> Signed : Vendor edits qty + date,\nsigns digitally

    Signed --> Draft : Portal Admin unlocks\n(vendor notified by email)

    Signed --> Done : Store confirms receipt\nin Odoo

    Done --> [*]

    Draft --> Cancelled : PO cancelled\n(before receipt confirmation)

    Cancelled --> [*]

    note right of Draft
        Vendor: can edit delivery date + quantities
        Quantities must be <= ordered qty
        Single date for entire DO
        Portal: editable
    end note

    note right of Signed
        Vendor: read-only, can print DO PDF
        Portal: locked, pushed to Odoo
        Odoo Receipt: set quantities populated
        No email sent
    end note

    note right of Done
        Vendor: read-only, sees final received qty
        Can export PDF/CSV
        Email sent if qty differs
    end note

    note right of Cancelled
        Vendor: read-only
        Only possible if Receipt not yet confirmed
    end note
```

---

## Returns: RPO & Goods Return Note

The portal supports returns via **Return Purchase Order (RPO)** and **Goods Return Note (GRN)**, mirroring the PO/DO flow.

| Concept | Purchase Flow | Returns Flow |
|---|---|---|
| Order | PO (Purchase Order) | RPO (Return Purchase Order) |
| Delivery document | DO (Delivery Order) | GRN (Goods Return Note) |
| Statuses | Same lifecycle | Same lifecycle |

> Detailed RPO/GRN workflow to be specified after core PO/DO flow is finalized.

---

## Data Retention

- Vendors can view PO data for **24 months** from PO creation date
- Applies to **all statuses** equally: Waiting, Confirmed, Cancelled
- Applies to **all DO statuses**: Draft, Signed, Done, Cancelled
- POs older than 24 months are **permanently deleted** from the portal database
- A scheduled cleanup job runs periodically to enforce this rule

---

## Data Export

- Vendors can export data as **PDF** or **CSV**
- Scope: PO number, DO quantities, receipt quantities, delivery date, receipt confirmation date
- Date range filter available
- Includes both regular POs/DOs and returns (RPO/GRN)

---

**Glossary**

| Term | Meaning |
|---|---|
| NCC | Nhà cung cấp (Vendor) |
| DO | Delivery Order — vendor's planned delivery document, edited and signed on portal |
| GRN | Goods Return Note — return equivalent of DO |
| Receipt | Phiếu nhập kho — Odoo's incoming shipment record, confirmed by store |
| RPO | Return Purchase Order — return equivalent of PO |
| SL | Số lượng (Quantity) |
| RFQ | Request for Quotation |
| PO | Purchase Order |
| Set Quantities | Pre-filled quantities in Odoo Receipt from vendor's DO (not yet qty_done) |
| qty_done | Final received quantity, set only when store confirms Receipt in Odoo |

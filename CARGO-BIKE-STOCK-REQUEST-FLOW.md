# Cargo Bike Stock Request Flow

Dokumen ini menjelaskan alur lengkap permintaan stok dari WD_MOBILE (Cargo Bike) ke DC_ADMIN, sampai stok masuk ke inventory cargo bike.

---

## Daftar Isi

1. [Overview](#overview)
2. [Fase 1: Get Suggested Stock](#fase-1-get-suggested-stock)
3. [Fase 2: Create Stock Request](#fase-2-create-stock-request)
4. [Fase 3: DC Review Request](#fase-3-dc-review-request)
5. [Fase 4: DC Allocate Stock](#fase-4-dc-allocate-stock)
6. [Fase 5: Stock Masuk ke Cargo Bike](#fase-5-stock-masuk-ke-cargo-bike)
7. [Status Flow Diagram](#status-flow-diagram)
8. [Contoh Skenario Lengkap](#contoh-skenario-lengkap)
9. [Business Rules](#business-rules)

---

## Overview

```
WD_MOBILE                    DC_ADMIN                    SYSTEM
    |                            |                          |
    |── Get Suggested Request ───|                          |
    |                            |                          |
    |── Create Stock Request ────|                          |
    |                            |                          |
    |                            |── Review Request         |
    |                            |                          |
    |                            |── Allocate Stock ────────|
    |                            |                          |
    |◄── Stock Ready ◄───────────|                          |
    |                            |                          |
```

**Actor:**
- **WD_MOBILE**: User Cargo Bike, meminta stok
- **DC_ADMIN**: Distribution Center Admin, approve dan allocate stok

---

## Fase 1: Get Suggested Stock

### 1.1 Get Suggested Stock Request

**Endpoint:** `GET /api/v1/cb-operations/{operationId}/request-stock/suggested`

**Role:** WD_MOBILE

**Deskripsi:**
Sistem otomatis menghitung kebutuhan stok berdasarkan:
```
suggestedQty = maxAllocation - currentStock
```

**Response:**
```json
{
  "success": true,
  "data": {
    "dailyOpId": "operation_cuid",
    "cargoBikeId": "cb_001_id",
    "totalItems": 3,
    "items": [
      {
        "productId": "product_1",
        "product": {
          "id": "product_1",
          "sku": "PRD-001",
          "name": "Minyak Goreng 1L"
        },
        "maxCapacity": 50,
        "currentStock": 20,
        "requestedQty": 30,
        "notes": null
      },
      {
        "productId": "product_2",
        "product": {
          "id": "product_2",
          "sku": "PRD-002",
          "name": "Beras 5kg"
        },
        "maxCapacity": 30,
        "currentStock": 0,
        "requestedQty": 30,
        "notes": null
      },
      {
        "productId": "product_3",
        "product": {
          "id": "product_3",
          "sku": "PRD-003",
          "name": "Gula Pasir 1kg"
        },
        "maxCapacity": 40,
        "currentStock": 10,
        "requestedQty": 30,
        "notes": null
      }
    ]
  }
}
```

**Kalkulasi:**
| Produk | Max Allocation | Current Stock | Suggested Request |
|--------|----------------|---------------|-------------------|
| Minyak Goreng 1L | 50 | 20 | 30 |
| Beras 5kg | 30 | 0 | 30 |
| Gula Pasir 1kg | 40 | 10 | 30 |

**Catatan:**
- Jika `currentStock >= maxAllocation`, item tidak muncul di suggested
- User bisa request manual (tidak harus pakai suggested)

---

## Fase 2: Create Stock Request

### 2.1 Auto Request Stock

**Endpoint:** `POST /api/v1/cb-operations/{operationId}/request-stock/auto`

**Role:** WD_MOBILE

**Deskripsi:**
Buat stock request otomatis berdasarkan suggested items. Tidak perlu kirim body, sistem pakai suggested calculation.

**Response:**
```json
{
  "success": true,
  "message": "Stock request created automatically",
  "data": {
    "id": "stock_request_cuid",
    "dailyOpId": "operation_cuid",
    "cargoBikeId": "cb_001_id",
    "dcId": "dc_jakarta_id",
    "requestedBy": "user_cuid",
    "status": "PENDING",
    "requestedAt": "2026-02-09T08:10:00.000Z",
    "items": [
      {
        "id": "request_item_1",
        "productId": "product_1",
        "product": {
          "id": "product_1",
          "sku": "PRD-001",
          "name": "Minyak Goreng 1L"
        },
        "requestedQty": 30,
        "approvedQty": null,
        "actualQty": null
      },
      {
        "id": "request_item_2",
        "productId": "product_2",
        "product": {
          "id": "product_2",
          "sku": "PRD-002",
          "name": "Beras 5kg"
        },
        "requestedQty": 30,
        "approvedQty": null,
        "actualQty": null
      },
      {
        "id": "request_item_3",
        "productId": "product_3",
        "product": {
          "id": "product_3",
          "sku": "PRD-003",
          "name": "Gula Pasir 1kg"
        },
        "requestedQty": 30,
        "approvedQty": null,
        "actualQty": null
      }
    ]
  }
}
```

**Status Changes:**
- Daily Operation: `STARTED` → `STOCK_REQUESTED`
- Stock Request: `PENDING`

### 2.2 Manual Request Stock (Alternatif)

Jika user ingin request manual (tidak pakai suggested):

**Endpoint:** Manual via POST ke `/cb-operations/{id}/request-stock` dengan body custom.

User bisa modifikasi quantity atau hapus item tertentu dari request.

---

## Fase 3: DC Review Request

### 3.1 DC Lihat Pending Requests

**Endpoint:** `GET /api/v1/cb-operations?page=1&limit=10&status=STOCK_REQUESTED`

**Role:** DC_ADMIN

**Deskripsi:**
DC Admin melihat semua request yang statusnya `STOCK_REQUESTED`.

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "id": "operation_cuid",
      "cargoBikeId": "cb_001_id",
      "cargoBike": {
        "code": "CB-001",
        "name": "Cargo Bike Jakarta 1"
      },
      "status": "STOCK_REQUESTED",
      "stockRequestedAt": "2026-02-09T08:10:00.000Z",
      "stockRequest": {
        "id": "stock_request_cuid",
        "items": [
          {
            "productId": "product_1",
            "requestedQty": 30
          }
        ]
      }
    }
  ],
  "meta": {
    "page": 1,
    "limit": 10,
    "total": 5
  }
}
```

### 3.2 DC Lihat Detail Request

**Endpoint:** `GET /api/v1/cb-operations/{operationId}`

**Role:** DC_ADMIN

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "operation_cuid",
    "cargoBike": {
      "id": "cb_001_id",
      "code": "CB-001",
      "name": "Cargo Bike Jakarta 1"
    },
    "shift": {
      "id": "shift_pagi_id",
      "name": "Pagi",
      "startTime": "08:00",
      "endTime": "14:00"
    },
    "status": "STOCK_REQUESTED",
    "stockRequest": {
      "id": "stock_request_cuid",
      "status": "PENDING",
      "requestedBy": "user_cuid",
      "requestedAt": "2026-02-09T08:10:00.000Z",
      "items": [
        {
          "id": "request_item_1",
          "productId": "product_1",
          "product": {
            "id": "product_1",
            "sku": "PRD-001",
            "name": "Minyak Goreng 1L"
          },
          "requestedQty": 30
        }
      ]
    }
  }
}
```

---

## Fase 4: DC Allocate Stock

### 4.1 DC Allocate Stock

**Endpoint:** `POST /api/v1/cb-operations/{requestId}/allocate`

**Role:** DC_ADMIN

**Deskripsi:**
DC Admin allocate stok ke cargo bike. Bisa:
- Full allocate (approvedQty = requestedQty)
- Partial allocate (approvedQty < requestedQty)

**Request:**
```json
{
  "items": [
    {
      "productId": "product_1",
      "actualQty": 30
    },
    {
      "productId": "product_2",
      "actualQty": 25
    },
    {
      "productId": "product_3",
      "actualQty": 30
    }
  ]
}
```

**Response:**
```json
{
  "success": true,
  "message": "Stock allocated successfully",
  "data": {
    "id": "stock_request_cuid",
    "status": "APPROVED",
    "approvedBy": "dc_admin_cuid",
    "approvedAt": "2026-02-09T08:30:00.000Z",
    "allocatedBy": "dc_admin_cuid",
    "allocatedAt": "2026-02-09T08:30:00.000Z",
    "items": [
      {
        "id": "request_item_1",
        "productId": "product_1",
        "product": {
          "sku": "PRD-001",
          "name": "Minyak Goreng 1L"
        },
        "requestedQty": 30,
        "approvedQty": 30,
        "actualQty": 30
      },
      {
        "id": "request_item_2",
        "productId": "product_2",
        "product": {
          "sku": "PRD-002",
          "name": "Beras 5kg"
        },
        "requestedQty": 30,
        "approvedQty": 25,
        "actualQty": 25
      },
      {
        "id": "request_item_3",
        "productId": "product_3",
        "product": {
          "sku": "PRD-003",
          "name": "Gula Pasir 1kg"
        },
        "requestedQty": 30,
        "approvedQty": 30,
        "actualQty": 30
      }
    ]
  }
}
```

**Catatan:**
- `requestedQty`: Jumlah yang diminta WD_MOBILE
- `approvedQty`: Jumlah yang diapprove DC_ADMIN
- `actualQty`: Jumlah yang sebenarnya dikirim (bisa beda karena partial)

---

## Fase 5: Stock Masuk ke Cargo Bike

### 5.1 Proses Backend (Otomatis)

Saat DC_ADMIN call allocate endpoint, sistem otomatis:

```
1. Kurangi DC Inventory
   DCInventory.qtyGood -= actualQty

2. Tambah Cargo Bike Inventory  
   CargoBikeInventory.qtyGood += actualQty
   (Upsert: create if not exist, update if exist)

3. Update Daily Operation Status
   Status: STOCK_REQUESTED → STOCK_ALLOCATED

4. Update Stock Request Status
   Status: PENDING → APPROVED
```

### 5.2 Data Flow Diagram

```
BEFORE ALLOCATE:
┌─────────────────────────────────────────────────────────────┐
│ DC INVENTORY                                                │
│ ┌─────────────┬─────────────────┬─────────────────────────┐ │
│ │ Product     │ qtyGood         │ qtyReserved             │ │
│ ├─────────────┼─────────────────┼─────────────────────────┤ │
│ │ PRD-001     │ 100             │ 0                       │ │
│ │ PRD-002     │ 50              │ 0                       │ │
│ │ PRD-003     │ 80              │ 0                       │ │
│ └─────────────┴─────────────────┴─────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ CARGO BIKE INVENTORY                                        │
│ ┌─────────────┬─────────────────┐                           │
│ │ Product     │ qtyGood         │                           │
│ ├─────────────┼─────────────────┤                           │
│ │ PRD-001     │ 20              │                           │
│ │ PRD-002     │ 0               │                           │
│ │ PRD-003     │ 10              │                           │
│ └─────────────┴─────────────────┘                           │
└─────────────────────────────────────────────────────────────┘

AFTER ALLOCATE (30, 25, 30):
┌─────────────────────────────────────────────────────────────┐
│ DC INVENTORY                                                │
│ ┌─────────────┬─────────────────┬─────────────────────────┐ │
│ │ Product     │ qtyGood         │ qtyReserved             │ │
│ ├─────────────┼─────────────────┼─────────────────────────┤ │
│ │ PRD-001     │ 70 (-30)        │ 0                       │ │
│ │ PRD-002     │ 25 (-25)        │ 0                       │ │
│ │ PRD-003     │ 50 (-30)        │ 0                       │ │
│ └─────────────┴─────────────────┴─────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│ CARGO BIKE INVENTORY                                        │
│ ┌─────────────┬─────────────────┐                           │
│ │ Product     │ qtyGood         │                           │
│ ├─────────────┼─────────────────┤                           │
│ │ PRD-001     │ 50 (+30)        │                           │
│ │ PRD-002     │ 25 (+25)        │                           │
│ │ PRD-003     │ 40 (+30)        │                           │
│ └─────────────┴─────────────────┘                           │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 WD_MOBILE Cek Stok

Setelah allocate, WD_MOBILE bisa cek stok mereka:

**Endpoint:** `GET /api/v1/mobile/dashboard`

**Response:**
```json
{
  "success": true,
  "data": {
    "inventory": [
      {
        "productId": "product_1",
        "sku": "PRD-001",
        "name": "Minyak Goreng 1L",
        "qtyGood": 50,
        "maxCapacity": 50
      },
      {
        "productId": "product_2",
        "sku": "PRD-002",
        "name": "Beras 5kg",
        "qtyGood": 25,
        "maxCapacity": 30
      }
    ],
    "status": "STOCK_ALLOCATED",
    "readyToSell": true
  }
}
```

---

## Status Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    WD_MOBILE (User)                             │
│                                                                 │
│  ┌─────────────────┐                                            │
│  │  STARTED        │── Get Suggested Request ──┐                │
│  │  (Daily Op)     │                           │                │
│  └─────────────────┘                           ▼                │
│                                ┌─────────────────────────────┐  │
│                                │  Suggested Items            │  │
│                                │  - Product A: request 30    │  │
│                                │  - Product B: request 25    │  │
│                                └──────────────┬──────────────┘  │
│                                               │                 │
│                                               ▼                 │
│                                ┌─────────────────────────────┐  │
│                                │  Create Stock Request       │  │
│                                │  POST /request-stock/auto   │  │
│                                └──────────────┬──────────────┘  │
│                                               │                 │
└───────────────────────────────────────────────┼─────────────────┘
                                                │
                                                ▼ Request masuk ke DC
┌─────────────────────────────────────────────────────────────────┐
│                    DC_ADMIN                                      │
│                                                                 │
│  ┌─────────────────┐                                            │
│  │  Lihat Pending  │◄───────────────────────────────────────────┘
│  │  Requests       │
│  └────────┬────────┘
│           │
│           ▼ Review & Allocate
│  ┌─────────────────────────────┐
│  │  Allocate Stock             │
│  │  POST /{requestId}/allocate │
│  │                             │
│  │  Item A: 30 (full)          │
│  │  Item B: 25 (partial)       │
│  │  Item C: 30 (full)          │
│  └─────────────┬───────────────┘
│                │
└────────────────┼─────────────────────────────────────────────────┘
                 │
                 ▼ System Process (Transaction)
┌─────────────────────────────────────────────────────────────────┐
│                    SYSTEM PROCESS                                │
│                                                                 │
│  1. Kurangi DC Inventory                                        │
│     DC.qtyGood -= actualQty                                     │
│                                                                 │
│  2. Tambah Cargo Bike Inventory                                 │
│     CargoBike.qtyGood += actualQty                              │
│     (Upsert: create or update)                                  │
│                                                                 │
│  3. Update Status                                               │
│     DailyOp: STOCK_REQUESTED → STOCK_ALLOCATED                  │
│     Request: PENDING → APPROVED                                 │
│                                                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼ Notify WD_MOBILE
┌─────────────────────────────────────────────────────────────────┐
│                    WD_MOBILE (User)                             │
│                                                                 │
│  ┌─────────────────────────────┐                                │
│  │  STOCK_ALLOCATED            │◄───────────────────────────────┘
│  │                             │
│  │  Stok siap diambil:         │
│  │  - Minyak: 30 pcs           │
│  │  - Beras: 25 pcs            │
│  │  - Gula: 30 pcs             │
│  │                             │
│  │  Status: READY TO SELL      │
│  └─────────────────────────────┘
│           │
│           ▼
│  ┌─────────────────────────────┐
│  │  Mulai Jualan!              │
│  │  POST /transactions         │
│  └─────────────────────────────┘
│
└─────────────────────────────────────────────────────────────────┘
```

---

## Contoh Skenario Lengkap

### Skenario: Request Stok Pagi

**Kondisi Awal:**
- Cargo Bike: CB-001
- Shift: Pagi (08:00-14:00)
- Daily Operation: STARTED
- DC Inventory: Minyak 100pcs, Beras 50pcs, Gula 80pcs

**Timeline:**

| Waktu | Aktor | Aktivitas | Request | Response |
|-------|-------|-----------|---------|----------|
| 08:05 | WD_MOBILE | Get suggested stock | `GET /cb-operations/{id}/request-stock/suggested` | 3 items (30, 30, 30) |
| 08:10 | WD_MOBILE | Auto request stock | `POST /cb-operations/{id}/request-stock/auto` | Request created, status: PENDING |
| 08:15 | DC_ADMIN | Lihat pending requests | `GET /cb-operations?status=STOCK_REQUESTED` | List requests |
| 08:30 | DC_ADMIN | Allocate stock | `POST /cb-operations/{id}/allocate` | Items: 30, 25, 30 |
| 08:30 | SYSTEM | Transfer stock | Transaction | DC: -30, -25, -30<br>CB: +30, +25, +30 |
| 08:31 | WD_MOBILE | Cek dashboard | `GET /mobile/dashboard` | Stock allocated, ready to sell |

**Data Setelah Allocate:**

| Lokasi | Minyak | Beras | Gula |
|--------|--------|-------|------|
| DC Inventory | 70 | 25 | 50 |
| Cargo Bike Inventory | 50 | 25 | 40 |

---

## Business Rules

### 1. Allocation Rules
- **Full Allocation**: `actualQty = requestedQty`
- **Partial Allocation**: `actualQty < requestedQty` (jika stok DC terbatas)
- **Reject**: `actualQty = 0` (jika stok habis)

### 2. Stock Availability Check
- DC harus cek `DCInventory.qtyGood >= requestedQty`
- Jika tidak cukup, bisa partial allocate atau reject

### 3. Max Capacity Enforcement
- WD_MOBILE tidak bisa melebihi `maxAllocation` per product
- Suggested request otomatis calculate: `max - current`

### 4. One Request Per Day
- 1 Daily Operation hanya bisa punya 1 Stock Request
- Jika sudah ada, harus tunggu sampai allocated/done

### 5. Status Dependency
- Hanya bisa request stock jika Daily Operation status: `STARTED`
- Tidak bisa request jika status: `STOCK_REQUESTED`, `STOCK_ALLOCATED`, `ON_DUTY`, dll

---

## Sequence Diagram

```
WD_MOBILE                    DC_ADMIN                     DATABASE
    |                            |                            |
    |── Get Suggested Request ───|                            |
    |◄── Suggested Items ────────|                            |
    |                            |                            |
    |── Create Stock Request ────|                            |
    |                            |── Save Request ────────────|
    |                            |◄── Saved ──────────────────|
    |◄── Request Created ────────|                            |
    |                            |                            |
    |                            |── Get Pending Requests ────|
    |                            |◄── Requests ───────────────|
    |                            |                            |
    |                            |── Allocate Stock ──────────|
    |                            |                            |
    |                            |    BEGIN TRANSACTION       |
    |                            |                            |
    |                            |── Update DC Inventory ─────|
    |                            |◄── Updated ────────────────|
    |                            |                            |
    |                            |── Update CB Inventory ─────|
    |                            |◄── Updated ────────────────|
    |                            |                            |
    |                            |── Update Request Status ───|
    |                            |◄── Updated ────────────────|
    |                            |                            |
    |                            |    COMMIT TRANSACTION      |
    |                            |                            |
    |◄── Notification: Stock Ready ──────────────────────────|
    |                            |                            |
```

---

*Dokumen ini dibuat untuk Arcson POS Backend*
*Tanggal: 2026-02-09*

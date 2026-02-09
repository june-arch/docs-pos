# Cargo Bike Daily Operation Flow

Dokumen ini menjelaskan alur lengkap operasional harian Cargo Bike dari konfigurasi awal sampai end day.

---

## Daftar Isi

1. [Overview](#overview)
2. [Fase 1: Konfigurasi Awal](#fase-1-konfigurasi-awal)
3. [Fase 2: Shift Assignment](#fase-2-shift-assignment)
4. [Fase 3: User Login & Start Day](#fase-3-user-login--start-day)
5. [Fase 4: Stock Request & Allocation](#fase-4-stock-request--allocation)
6. [Fase 5: End Day Check](#fase-5-end-day-check)
7. [Fase 6: Payment (Jika Ada)](#fase-6-payment-jika-ada)
8. [Status Flow Diagram](#status-flow-diagram)
9. [Daftar Endpoint](#daftar-endpoint)
10. [Contoh Skenario Lengkap](#contoh-skenario-lengkap)

---

## Overview

Flow ini mencakup seluruh siklus operasional harian Cargo Bike:

 ```
   WDI_ADMIN: Assign shift (SCHEDULED)
       ↓
   WD_MOBILE: Start shift (SCHEDULED → ACTIVE)
       ↓
   WD_MOBILE: Start day (STARTED)
       ↓
   WD_MOBILE: Request stock (STOCK_REQUESTED)
       ↓
   DC_ADMIN: Allocate stock (STOCK_ALLOCATED)
       ↓
   WD_MOBILE: Jualan (ON_DUTY)
       ↓
   DC_ADMIN: End day check (END_DAY_CHECKING → WAITING_PAYMENT/COMPLETED)
       ↓
   WD_MOBILE: Pay (jika ada) (COMPLETED)
 ```


```
Konfigurasi → Assignment → Start Day → Stock Request → 
Jualan → End Day Check → Payment (jika perlu) → Complete
```

**Actor:**
- **WDI_ADMIN**: Head Office, manage master data
- **DC_ADMIN**: Distribution Center Admin, manage operasional
- **WD_MOBILE**: User Cargo Bike, jualan

---

## Fase 1: Konfigurasi Awal

### 1.1 Create Cargo Bike

**Endpoint:** `POST /api/v1/cargo-bikes`

**Request:**
```json
{
  "code": "CB-001",
  "name": "Cargo Bike Jakarta 1",
  "plateNumber": "B 1234 ABC",
  "dcId": "dc_jakarta_id",
  "status": "ACTIVE"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "cargo_bike_cuid",
    "code": "CB-001",
    "name": "Cargo Bike Jakarta 1",
    "plateNumber": "B 1234 ABC",
    "dcId": "dc_jakarta_id",
    "status": "ACTIVE",
    "createdAt": "2026-02-09T00:00:00.000Z"
  }
}
```

**Role:** WDI_ADMIN

---

### 1.2 Create Shift

**Endpoint:** `POST /api/v1/cargo-bikes/{cargoBikeId}/shifts`

**Request:**
```json
{
  "name": "Pagi",
  "shiftCode": "SHIFT-PAGI",
  "startTime": "08:00",
  "endTime": "14:00",
  "sequence": 1
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "shift_cuid",
    "name": "Pagi",
    "shiftCode": "SHIFT-PAGI",
    "startTime": "08:00",
    "endTime": "14:00",
    "sequence": 1,
    "isActive": true
  }
}
```

**Role:** DC_ADMIN atau WDI_ADMIN

---

### 1.3 Create Product Allocation

**Endpoint:** `POST /api/v1/cargo-bikes/{cargoBikeId}/allocations`

**Request:**
```json
{
  "productId": "product_id",
  "maxQuantity": 50,
  "notes": "Maksimal 50 pcs di cargo bike"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "allocation_cuid",
    "cargoBikeId": "cargo_bike_cuid",
    "productId": "product_id",
    "maxQuantity": 50,
    "isActive": true
  }
}
```

**Role:** WDI_ADMIN

**Catatan:**
- Allocation menentukan kapasitas maksimal stok per produk di cargo bike
- Sistem akan otomatis menghitung suggested request berdasarkan: maxQuantity - currentStock

---

## Fase 2: Shift Assignment

### 2.1 Assign User ke Shift

**Endpoint:** `POST /api/v1/shift-assignments`

**Request:**
```json
{
  "cargoBikeId": "cb_001_id",
  "shiftId": "shift_pagi_id",
  "userId": "wd_mobile_user_id",
  "date": "2026-02-09"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "shift_assignment_cuid",
    "cargoBikeId": "cb_001_id",
    "shiftId": "shift_pagi_id",
    "userId": "wd_mobile_user_id",
    "date": "2026-02-09",
    "status": "SCHEDULED",
    "assignedBy": "admin_user_id"
  }
}
```

**Role:** DC_ADMIN

**Validasi:**
- User harus punya role WD_MOBILE
- Tidak boleh ada assignment double untuk cargo bike + shift + date yang sama

---

## Fase 3: User Login & Start Day

### 3.1 User Login

**Endpoint:** `POST /api/v1/auth/login`

**Request:**
```json
{
  "email": "vendor@example.com",
  "password": "Password123"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "user_cuid",
      "email": "vendor@example.com",
      "name": "John Vendor",
      "role": "WD_MOBILE"
    },
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
  }
}
```

**Role:** Public

---

### 3.2 Get My Current Shift

**Endpoint:** `GET /api/v1/shift-assignments/my/current`

**Response:**
```json
{
  "success": true,
  "data": {
    "session": {
      "id": "session_cuid",
      "loginAt": "2026-02-09T07:55:00.000Z",
      "lastActivityAt": "2026-02-09T07:55:00.000Z",
      "expectedLogoutAt": "2026-02-09T14:10:00.000Z"
    },
    "assignment": {
      "id": "assignment_cuid",
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
      "status": "SCHEDULED"
    }
  }
}
```

**Role:** WD_MOBILE

---

### 3.3 Start Shift

**Endpoint:** `POST /api/v1/shift-assignments/{assignmentId}/start`

**Request:**
```json
{
  "deviceId": "device_123",
  "fcmToken": "fcm_token_xxx"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Shift started successfully",
  "data": {
    "id": "assignment_cuid",
    "status": "ACTIVE",
    "startedAt": "2026-02-09T08:00:00.000Z",
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
    }
  }
}
```

**Role:** WD_MOBILE

**Catatan:**
- Status shift berubah dari SCHEDULED → ACTIVE
- Auto create UserActiveSession dengan expectedLogoutAt = endTime + 10 menit
- Auto logout akan terjadi setelah 10 menit tolerance dari endTime

---

### 3.4 Start Day (Mulai Operasional Harian)

**Endpoint:** `POST /api/v1/cb-operations/cargo-bikes/{cargoBikeId}/start`

**Request:**
```json
{
  "shiftId": "shift_pagi_id"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Daily operation started",
  "data": {
    "id": "operation_cuid",
    "cargoBikeId": "cb_001_id",
    "shiftId": "shift_pagi_id",
    "date": "2026-02-09T00:00:00.000Z",
    "status": "STARTED",
    "startedAt": "2026-02-09T08:05:00.000Z",
    "startedBy": "user_cuid"
  }
}
```

**Role:** WD_MOBILE

**Catatan:**
- 1 cargo bike + 1 shift + 1 date = 1 daily operation
- Jika sudah ada, akan return existing operation
- Status: STARTED

---

## Fase 4: Stock Request & Allocation

### 4.1 Get Suggested Stock Request

**Endpoint:** `GET /api/v1/cb-operations/{operationId}/request-stock/suggested`

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
          "name": "Product A"
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
          "name": "Product B"
        },
        "maxCapacity": 30,
        "currentStock": 0,
        "requestedQty": 30,
        "notes": null
      }
    ]
  }
}
```

**Role:** WD_MOBILE

**Formula:**
```
requestedQty = maxQuantity - currentStock
```

**Contoh:**
- Max allocation: 50
- Current stock: 20
- Suggested request: 30

---

### 4.2 Auto Request Stock

**Endpoint:** `POST /api/v1/cb-operations/{operationId}/request-stock/auto`

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
    "items": [
      {
        "id": "request_item_1",
        "productId": "product_1",
        "requestedQty": 30
      },
      {
        "id": "request_item_2",
        "productId": "product_2",
        "requestedQty": 30
      }
    ]
  }
}
```

**Role:** WD_MOBILE

**Catatan:**
- Auto calculate dari suggested items
- Status operation berubah: STARTED → STOCK_REQUESTED

---

### 4.3 DC Admin Allocate Stock (Approve)

**Endpoint:** `POST /api/v1/cb-operations/{requestId}/allocate`

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
    "approvedAt": "2026-02-09T08:30:00.000Z"
  }
}
```

**Role:** DC_ADMIN

**Proses Backend:**
1. Kurangi DC Inventory (qtyGood)
2. Tambah CargoBike Inventory (qtyGood)
3. Update status operation: STOCK_REQUESTED → STOCK_ALLOCATED

---

### 4.4 User On Duty (Jualan)

User mulai berjualan menggunakan stok yang sudah diallocate.

**Create Transaction:**

**Endpoint:** `POST /api/v1/transactions`

**Request:**
```json
{
  "items": [
    {
      "productId": "product_1",
      "quantity": 2,
      "price": 7500,
      "discount": 0
    }
  ],
  "discount": 0,
  "tax": 0,
  "paymentMethod": "CASH",
  "amountPaid": 15000,
  "customerName": "Customer A",
  "cargoBikeId": "cb_001_id"
}
```

**Role:** WD_MOBILE

**Catatan:**
- Setiap transaksi akan mengurangi CargoBike Inventory
- Status operation: ON_DUTY

---

## Fase 5: End Day Check

### 5.1 Start End Day Check

**Endpoint:** `POST /api/v1/cb-operations/{operationId}/end-day`

**Request:**
```json
{
  "notes": "Memulai pengecekan akhir shift"
}
```

**Response:**
```json
{
  "success": true,
  "message": "End day check started",
  "data": {
    "id": "check_cuid",
    "dailyOpId": "operation_cuid",
    "cargoBikeId": "cb_001_id",
    "dcId": "dc_jakarta_id",
    "status": "CHECKING",
    "checkedBy": "dc_admin_cuid",
    "checkedAt": "2026-02-09T13:50:00.000Z",
    "items": [
      {
        "id": "check_item_1",
        "productId": "product_1",
        "morningQty": 30,
        "soldQty": 5,
        "expectedQty": 25,
        "actualQty": 0,
        "status": "PENDING"
      },
      {
        "id": "check_item_2",
        "productId": "product_2",
        "morningQty": 25,
        "soldQty": 3,
        "expectedQty": 22,
        "actualQty": 0,
        "status": "PENDING"
      }
    ]
  }
}
```

**Role:** DC_ADMIN

**Proses Auto:**
1. Ambil data inventory cargo bike saat ini
2. Hitung soldQty dari transaksi hari ini
3. Generate expectedQty: morningQty - soldQty
4. Buat checklist item untuk dicek

**Status:** ON_DUTY → END_DAY_CHECKING

---

### 5.2 Check Item

**Endpoint:** `PATCH /api/v1/cb-operations/check/{checkId}/items`

**Request:**
```json
{
  "productId": "product_1",
  "actualQty": 23,
  "damagedQty": 1,
  "notes": "1 item rusak karena jatuh"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "check_item_1",
    "productId": "product_1",
    "morningQty": 30,
    "soldQty": 5,
    "expectedQty": 25,
    "actualQty": 23,
    "damagedQty": 1,
    "missingQty": 1,
    "missingValue": 5000,
    "damagedValue": 5000,
    "totalLossValue": 10000,
    "status": "SHORTAGE",
    "notes": "1 item rusak karena jatuh",
    "checkedAt": "2026-02-09T13:55:00.000Z",
    "product": {
      "id": "product_1",
      "sku": "PRD-001",
      "name": "Product A"
    }
  }
}
```

**Role:** DC_ADMIN

**Formula:**
```
missingQty = expectedQty - actualQty - damagedQty
missingValue = missingQty * costPrice
damagedValue = damagedQty * costPrice
totalLossValue = missingValue + damagedValue
```

**Status Item:**
- OK: actual + damaged = expected
- SHORTAGE: ada kekurangan

---

### 5.3 Complete End Day Check

**Endpoint:** `POST /api/v1/cb-operations/check/{checkId}/complete`

**Request:**
```json
{
  "notes": "Pengecekan selesai"
}
```

**Response:**
```json
{
  "success": true,
  "message": "End day check completed",
  "data": {
    "id": "check_cuid",
    "status": "WAITING_PAYMENT",
    "totalItemsChecked": 2,
    "totalMissingQty": 1,
    "totalMissingValue": 5000,
    "totalDamagedQty": 1,
    "totalDamagedValue": 5000,
    "totalLossValue": 10000,
    "paymentRequired": true,
    "paymentAmount": 10000,
    "paymentStatus": "PENDING"
  }
}
```

**Role:** DC_ADMIN

**Proses Backend:**
1. Hitung total kekurangan dan kerusakan
2. Jika totalLossValue > 0: Status WAITING_PAYMENT
3. Jika totalLossValue = 0: Status COMPLETED
4. Return stok (actualQty) ke DC Inventory
5. Reset CargoBike Inventory (delete all)

**Status:**
- Ada kekurangan: END_DAY_CHECKING → WAITING_PAYMENT
- Tidak ada kekurangan: END_DAY_CHECKING → COMPLETED

---

## Fase 6: Payment (Jika Ada)

### 6.1 Pay Missing Stock

**Endpoint:** `POST /api/v1/cb-operations/check/{checkId}/pay`

**Request:**
```json
{
  "amount": 10000
}
```

**Response:**
```json
{
  "success": true,
  "message": "Payment successful",
  "data": {
    "id": "check_cuid",
    "status": "COMPLETED",
    "paidAmount": 10000,
    "paidAt": "2026-02-09T14:05:00.000Z",
    "paymentStatus": "PAID",
    "completedAt": "2026-02-09T14:05:00.000Z"
  }
}
```

**Role:** WD_MOBILE

**Proses Backend:**
1. Cek saldo DOKU Wallet user
2. Deduct saldo wallet
3. Update check status: COMPLETED
4. Update operation status: COMPLETED

**Validasi:**
- Saldo wallet harus cukup
- Status check harus WAITING_PAYMENT

---

## Status Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                      KONFIGURASI                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ Cargo Bike  │  │   Shift     │  │  Product Allocation     │  │
│  │  (WDI)      │  │ (DC/WDI)    │  │       (WDI)             │  │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘  │
└─────────┼────────────────┼─────────────────────┼────────────────┘
          │                │                     │
          ▼                ▼                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SHIFT ASSIGNMENT                             │
│              POST /shift-assignments (DC_ADMIN)                 │
│                          │                                      │
│                          ▼                                      │
│                   ┌─────────────┐                               │
│                   │  SCHEDULED  │                               │
│                   └──────┬──────┘                               │
└──────────────────────────┼──────────────────────────────────────┘
                           │
                           ▼ USER LOGIN & START
┌─────────────────────────────────────────────────────────────────┐
│                    START SHIFT (WD_MOBILE)                      │
│  POST /shift-assignments/{id}/start                             │
│                          │                                      │
│                          ▼                                      │
│                    ┌─────────────┐                              │
│                    │   ACTIVE    │                              │
│                    └──────┬──────┘                              │
└───────────────────────────┼─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     START DAY (WD_MOBILE)                       │
│  POST /cb-operations/cargo-bikes/{id}/start                     │
│                          │                                      │
│                          ▼                                      │
│                    ┌─────────────┐                              │
│                    │   STARTED   │                              │
│                    └──────┬──────┘                              │
└───────────────────────────┼─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   STOCK REQUEST (WD_MOBILE)                     │
│  GET/POST /cb-operations/{id}/request-stock/*                   │
│                          │                                      │
│                          ▼                                      │
│                ┌─────────────────────┐                          │
│                │   STOCK_REQUESTED   │                          │
│                └──────────┬──────────┘                          │
└───────────────────────────┼─────────────────────────────────────┘
                            │
                            ▼ DC ALLOCATE
┌─────────────────────────────────────────────────────────────────┐
│                  ALLOCATE STOCK (DC_ADMIN)                      │
│  POST /cb-operations/{requestId}/allocate                       │
│                          │                                      │
│                          ▼                                      │
│                ┌─────────────────────┐                          │
│                │   STOCK_ALLOCATED   │                          │
│                └──────────┬──────────┘                          │
└───────────────────────────┼─────────────────────────────────────┘
                            │
                            ▼ USER JUALAN
┌─────────────────────────────────────────────────────────────────┐
│                      ON DUTY (WD_MOBILE)                        │
│  POST /transactions (jualan)                                    │
│                          │                                      │
│                          ▼                                      │
│                  ┌───────────────┐                              │
│                  │    ON_DUTY    │                              │
│                  └───────┬───────┘                              │
└──────────────────────────┼──────────────────────────────────────┘
                           │
                           ▼ DC START CHECK
┌─────────────────────────────────────────────────────────────────┐
│                 END DAY CHECK (DC_ADMIN)                        │
│  POST /cb-operations/{id}/end-day                               │
│                          │                                      │
│                          ▼                                      │
│            ┌─────────────────────────┐                          │
│            │    END_DAY_CHECKING     │                          │
│            └────────────┬────────────┘                          │
└─────────────────────────┼───────────────────────────────────────┘
                          │
                          ▼ CHECK ITEMS
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               │               ▼
   ┌─────────────┐        │        ┌─────────────┐
   │   TDK ADA   │        │        │  ADA SHORT  │
   │  KEKURANGAN │        │        │   / DAMAGE  │
   └──────┬──────┘        │        └──────┬──────┘
          │               │               │
          ▼               │               ▼
   ┌─────────────┐        │        ┌─────────────┐
   │  COMPLETED  │        │        │WAITING_PAY  │
   │             │        │        │   MENT      │
   └─────────────┘        │        └──────┬──────┘
                          │               │
                          │               ▼ USER BAYAR
                          │        ┌─────────────┐
                          │        │  COMPLETED  │
                          │        │  (setelah   │
                          │        │  payment)   │
                          │        └─────────────┘
                          │
                          ▼
              ┌─────────────────────┐
              │     FLOW SELESAI    │
              └─────────────────────┘
```

---

## Daftar Endpoint

### Konfigurasi (WDI_ADMIN)
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| POST | `/cargo-bikes` | Create cargo bike |
| POST | `/cargo-bikes/{id}/shifts` | Create shift |
| POST | `/cargo-bikes/{id}/allocations` | Create product allocation |

### Shift Assignment (DC_ADMIN)
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| POST | `/shift-assignments` | Assign user ke shift |
| PATCH | `/shift-assignments/{id}` | Update assignment |
| DELETE | `/shift-assignments/{id}` | Delete assignment |

### User Operations (WD_MOBILE)
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| POST | `/auth/login` | Login |
| GET | `/shift-assignments/my/current` | Get shift aktif |
| GET | `/shift-assignments/my/schedule` | Get jadwal |
| POST | `/shift-assignments/{id}/start` | Start shift |
| POST | `/shift-assignments/{id}/end` | End shift |
| POST | `/cb-operations/cargo-bikes/{id}/start` | Start day |
| GET | `/cb-operations/{id}/request-stock/suggested` | Get suggested request |
| POST | `/cb-operations/{id}/request-stock/auto` | Auto request stock |
| POST | `/transactions` | Create transaction |
| POST | `/cb-operations/check/{id}/pay` | Pay missing stock |

### DC Operations (DC_ADMIN)
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| POST | `/cb-operations/{id}/allocate` | Allocate stock |
| POST | `/cb-operations/{id}/end-day` | Start end day check |
| PATCH | `/cb-operations/check/{id}/items` | Check item |
| POST | `/cb-operations/check/{id}/complete` | Complete check |

---

## Contoh Skenario Lengkap

### Skenario: Hari Kerja Cargo Bike

**Setup (Sebelumnya):**
- Cargo Bike: CB-001 (Pagi: 08:00-14:00)
- Product Allocation: Product A (max 50), Product B (max 30)
- User: vendor@example.com (WD_MOBILE)
- DC Admin: dcadmin@example.com (DC_ADMIN)

**Timeline:**

| Waktu | Aktor | Aktivitas | Endpoint | Status |
|-------|-------|-----------|----------|--------|
| 07:00 | DC_ADMIN | Assign shift | POST /shift-assignments | SCHEDULED |
| 08:00 | WD_MOBILE | Login | POST /auth/login | - |
| 08:00 | WD_MOBILE | Start shift | POST /shift-assignments/{id}/start | ACTIVE |
| 08:05 | WD_MOBILE | Start day | POST /cb-operations/cargo-bikes/{id}/start | STARTED |
| 08:10 | WD_MOBILE | Request stock | POST /cb-operations/{id}/request-stock/auto | STOCK_REQUESTED |
| 08:30 | DC_ADMIN | Allocate stock | POST /cb-operations/{id}/allocate | STOCK_ALLOCATED |
| 09:00-13:00 | WD_MOBILE | Jualan | POST /transactions | ON_DUTY |
| 13:50 | DC_ADMIN | Start end day check | POST /cb-operations/{id}/end-day | END_DAY_CHECKING |
| 13:55 | DC_ADMIN | Check items | PATCH /cb-operations/check/{id}/items | CHECKING |
| 14:00 | DC_ADMIN | Complete check | POST /cb-operations/check/{id}/complete | WAITING_PAYMENT/COMPLETED |
| 14:05 | WD_MOBILE | Bayar (jika ada) | POST /cb-operations/check/{id}/pay | COMPLETED |

---

## Catatan Penting

1. **Auto Logout**: User akan otomatis logout 10 menit setelah shift end time
2. **Shift Extension**: User bisa request extend shift via POST /shift-assignments/{id}/extend
3. **Stock Calculation**: Suggested request = maxAllocation - currentStock
4. **Payment**: Kekurangan stok harus dibayar via DOKU Wallet
5. **Inventory Flow**: DC → Cargo Bike (start day) → DC (end day)

---

*Dokumen ini dibuat untuk Arcson POS Backend*
*Tanggal: 2026-02-09*

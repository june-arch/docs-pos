# DC Stock Request to HO Flow (Order Management v2.0)

Dokumen ini menjelaskan alur lengkap permintaan stok dari Distribution Center (DC) ke Head Office (HO), diproses melalui Order Management v2.0, sampai barang sampai di DC dengan status complete.

---

## Daftar Isi

1. [Overview](#overview)
2. [Fase 1: DC Create Stock Request](#fase-1-dc-create-stock-request)
3. [Fase 2: HO Review dan Approve](#fase-2-ho-review-dan-approve)
4. [Fase 3: Supplier Process Order](#fase-3-supplier-process-order)
5. [Fase 4: Shipment dan Arrival](#fase-4-shipment-dan-arrival)
6. [Fase 5: DC Receive dan Check](#fase-5-dc-receive-dan-check)
7. [Status Flow Diagram](#status-flow-diagram)
8. [Perbandingan: Old vs New Flow](#perbandingan-old-vs-new-flow)
9. [Contoh Skenario Lengkap](#contoh-skenario-lengkap)

---

## Overview

```
DC_ADMIN                    HO_ADMIN (WDI)              SUPPLIER
    |                            |                          |
    |── Create Stock Request ────|                          |
    |                            |                          |
    |◄──── Request Received ◄────|                          |
    |                            |                          |
    |                            |── Review Items ──────────|
    |                            |    (By Supplier)         |
    |                            |                          |
    |                            |── Approve Supplier ──────|
    |                            |    (Create PO)           |
    |                            |                          |
    |◄── Order Created ◄─────────|                          |
    |                            |                          |
    |                            |                          |── Process
    |                            |                          |    Order
    |                            |                          |
    |                            |◄── Shipped ◄─────────────|
    |                            |                          |
    |◄── Barang Sampai ◄─────────|                          |
    |                            |                          |
    |── Mark Arrived ────────────|                          |
    |                            |                          |
    |── Start Checking ──────────|                          |
    |                            |                          |
    |── Check Items ─────────────|                          |
    |                            |                          |
    |── Complete ────────────────|                          |
    |                            |                          |
```

**Actor:**
- **DC_ADMIN**: Distribution Center Admin, request dan terima barang
- **WDI_ADMIN**: Head Office, approve dan manage supplier
- **SUPPLIER**: Vendor yang supply barang (external)

---

## Fase 1: DC Create Stock Request

### 1.1 DC Lihat Min Stock Products

**Endpoint:** `GET /api/v1/dc-stock-requests/min-stock-products`

**Role:** DC_ADMIN

**Deskripsi:**
Sistem otomatis menampilkan produk yang sudah mencapai minimum stock (untuk auto-suggestion).

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "productId": "product_1",
      "sku": "PRD-001",
      "name": "Minyak Goreng 1L",
      "currentStock": 15,
      "minStock": 20,
      "suggestedRequest": 85
    },
    {
      "productId": "product_2",
      "sku": "PRD-002",
      "name": "Beras 5kg",
      "currentStock": 8,
      "minStock": 10,
      "suggestedRequest": 42
    }
  ]
}
```

---

### 1.2 DC Create Stock Request

**Endpoint:** `POST /api/v1/dc-stock-requests`

**Role:** DC_ADMIN

**Request:**
```json
{
  "notes": "Request stok bulan Februari",
  "items": [
    {
      "productId": "product_1",
      "qtyRequested": 100,
      "notes": "Stok hampir habis"
    },
    {
      "productId": "product_2",
      "qtyRequested": 50
    },
    {
      "productId": "product_3",
      "qtyRequested": 75
    }
  ]
}
```

**Response:**
```json
{
  "success": true,
  "message": "Request stok berhasil dibuat",
  "data": {
    "id": "dc_stock_request_cuid",
    "requestNumber": "REQ-DC-20260209-0001",
    "distributionCenterId": "dc_jakarta_id",
    "requestedById": "dc_admin_cuid",
    "status": "PENDING",
    "totalItems": 225,
    "notes": "Request stok bulan Februari",
    "items": [
      {
        "id": "item_1",
        "productId": "product_1",
        "product": {
          "sku": "PRD-001",
          "name": "Minyak Goreng 1L"
        },
        "qtyRequested": 100,
        "status": "PENDING"
      },
      {
        "id": "item_2",
        "productId": "product_2",
        "product": {
          "sku": "PRD-002",
          "name": "Beras 5kg"
        },
        "qtyRequested": 50,
        "status": "PENDING"
      }
    ],
    "createdAt": "2026-02-09T08:00:00.000Z"
  }
}
```

**Status:**
- DC Stock Request: `PENDING`
- Items: `PENDING`

---

### 1.3 DC Cancel Request (Opsional)

**Endpoint:** `POST /api/v1/dc-stock-requests/{requestId}/cancel`

**Role:** DC_ADMIN

**Kondisi:** Hanya bisa cancel jika status masih `PENDING`

---

## Fase 2: HO Review dan Approve

### 2.1 HO Lihat Daftar Requests

**Endpoint:** `GET /api/v1/dc-stock-requests?status=PENDING`

**Role:** WDI_ADMIN

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "id": "dc_stock_request_cuid",
      "requestNumber": "REQ-DC-20260209-0001",
      "distributionCenter": {
        "id": "dc_jakarta_id",
        "code": "DC-001",
        "name": "DC Jakarta"
      },
      "requestedBy": {
        "id": "dc_admin_cuid",
        "name": "Admin DC"
      },
      "status": "PENDING",
      "totalItems": 225,
      "createdAt": "2026-02-09T08:00:00.000Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 10,
    "total": 5
  }
}
```

---

### 2.2 HO Get Items Grouped by Supplier

**Endpoint:** `GET /api/v1/dc-stock-requests/{requestId}/supplier-groups`

**Role:** WDI_ADMIN

**Deskripsi:**
Sistem otomatis mengelompokkan item berdasarkan supplier (dari product.supplierId).

**Response:**
```json
{
  "success": true,
  "data": {
    "request": {
      "id": "dc_stock_request_cuid",
      "requestNumber": "REQ-DC-20260209-0001",
      "status": "PENDING"
    },
    "groups": [
      {
        "supplier": {
          "id": "supplier_a_id",
          "name": "Supplier A",
          "code": "SUP-A"
        },
        "items": [
          {
            "id": "item_1",
            "productId": "product_1",
            "product": {
              "sku": "PRD-001",
              "name": "Minyak Goreng 1L"
            },
            "qtyRequested": 100
          },
          {
            "id": "item_3",
            "productId": "product_3",
            "product": {
              "sku": "PRD-003",
              "name": "Gula Pasir 1kg"
            },
            "qtyRequested": 75
          }
        ],
        "totalQty": 175
      },
      {
        "supplier": {
          "id": "supplier_b_id",
          "name": "Supplier B",
          "code": "SUP-B"
        },
        "items": [
          {
            "id": "item_2",
            "productId": "product_2",
            "product": {
              "sku": "PRD-002",
              "name": "Beras 5kg"
            },
            "qtyRequested": 50
          }
        ],
        "totalQty": 50
      }
    ]
  }
}
```

---

### 2.3 HO Approve Items (Order Management v2.0)

**Endpoint:** `POST /api/v1/dc-stock-requests/{requestId}/approve-items`

**Role:** WDI_ADMIN

**Request:**
```json
{
  "items": [
    {
      "productId": "product_1",
      "approved": true,
      "qtyApproved": 100,
      "unitPrice": 8500
    },
    {
      "productId": "product_2",
      "approved": true,
      "qtyApproved": 50,
      "unitPrice": 52000
    },
    {
      "productId": "product_3",
      "approved": false,
      "rejectReason": "Stok supplier tidak tersedia"
    }
  ]
}
```

**Response:**
```json
{
  "success": true,
  "message": "Item approval berhasil. Approved: 2, Rejected: 1",
  "data": {
    "request": {
      "id": "dc_stock_request_cuid",
      "status": "APPROVED",
      "approvedAt": "2026-02-09T09:00:00.000Z",
      "approvedById": "wdi_admin_cuid"
    },
    "approvedCount": 2,
    "rejectedCount": 1
  }
}
```

**Status Changes:**
- DC Stock Request: `PENDING` → `APPROVED`
- Approved items: status updated

---

### 2.4 HO Create Order Management (Untuk Setiap Supplier)

Setelah approve items, HO create order di Order Management v2.0 untuk setiap supplier.

**Endpoint:** `POST /api/v1/order-management/{requestId}/approve-supplier`

**Role:** WDI_ADMIN

**Request:**
```json
{
  "notes": "Order untuk Supplier A",
  "expectedDate": "2026-02-15",
  "items": [
    {
      "productId": "product_1",
      "qtyApproved": 100,
      "unitPrice": 8500,
      "supplierId": "supplier_a_id",
      "notes": "Urgent"
    }
  ]
}
```

**Response:**
```json
{
  "success": true,
  "message": "Order created for supplier",
  "data": {
    "orderId": "order_management_cuid",
    "orderNumber": "SO-20260209-0001-S1",
    "supplier": {
      "id": "supplier_a_id",
      "name": "Supplier A"
    },
    "status": "SUBMITTED",
    "items": [
      {
        "productId": "product_1",
        "qtyApproved": 100,
        "unitPrice": 8500,
        "subtotal": 850000
      }
    ],
    "totalAmount": 850000,
    "createdAt": "2026-02-09T09:30:00.000Z"
  }
}
```

**Status:**
- Order Management: `SUBMITTED`
- Supplier Order: `SUBMITTED`

---

### 2.5 HO Upload PO Documents

**Endpoint:** `POST /api/v1/dc-stock-requests/{requestId}/po-documents`

**Role:** WDI_ADMIN

**Request:**
- Content-Type: multipart/form-data
- Field: `documents` (files)

**Response:**
```json
{
  "success": true,
  "message": "2 PO document(s) uploaded successfully",
  "data": {
    "documents": [
      {
        "id": "doc_1",
        "fileName": "PO_Supplier_A.pdf",
        "filePath": "/uploads/documents/PO_Supplier_A.pdf"
      }
    ],
    "count": 2
  }
}
```

---

## Fase 3: Supplier Process Order

### 3.1 HO Update Order Status to Processing

**Endpoint:** `PATCH /api/v1/order-management/{orderId}/status`

**Role:** WDI_ADMIN

**Request:**
```json
{
  "status": "PROCESSING",
  "notes": "Order sedang diproses supplier"
}
```

**Status:** `SUBMITTED` → `PROCESSING`

---

### 3.2 HO Update Order Status to Shipped

**Endpoint:** `POST /api/v1/dc-stock-requests/{requestId}/supplier-order/shipped`

**Role:** WDI_ADMIN

**Request:**
```json
{
  "notes": "Barang sudah dikirim via logistik XYZ"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Order status updated to SHIPPED",
  "data": {
    "id": "dc_stock_request_cuid",
    "status": "SHIPPED",
    "shippedAt": "2026-02-12T10:00:00.000Z"
  }
}
```

**Status:** `PROCESSING` → `SHIPPED`

---

## Fase 4: Shipment dan Arrival

### 4.1 DC Get Order List

**Endpoint:** `GET /api/v1/order-management`

**Role:** DC_ADMIN (bisa lihat order untuk DC mereka)

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "id": "order_management_cuid",
      "orderNumber": "SO-20260209-0001-S1",
      "supplier": {
        "name": "Supplier A"
      },
      "status": "SHIPPED",
      "totalItems": 100,
      "totalAmount": 850000
    }
  ]
}
```

---

### 4.2 DC Mark Supplier Arrived

**Endpoint:** `POST /api/v1/order-management/{orderId}/suppliers/{supplierId}/arrived`

**Role:** DC_ADMIN

**Response:**
```json
{
  "success": true,
  "message": "Supplier marked as arrived",
  "data": {
    "orderId": "order_management_cuid",
    "supplierId": "supplier_a_id",
    "status": "ARRIVED",
    "arrivedAt": "2026-02-14T08:00:00.000Z"
  }
}
```

**Status:** `SHIPPED` → `ARRIVED`

---

## Fase 5: DC Receive dan Check

### 5.1 DC Start Checking

**Endpoint:** `POST /api/v1/order-management/{orderId}/suppliers/{supplierId}/checking`

**Role:** DC_ADMIN

**Response:**
```json
{
  "success": true,
  "message": "Start checking supplier items",
  "data": {
    "orderId": "order_management_cuid",
    "supplierId": "supplier_a_id",
    "status": "CHECKING"
  }
}
```

**Status:** `ARRIVED` → `CHECKING`

---

### 5.2 DC Check Item

**Endpoint:** `POST /api/v1/order-management/{orderId}/suppliers/{supplierId}/items/{itemId}/check`

**Role:** DC_ADMIN

**Request:**
```json
{
  "qtyNormal": 95,
  "qtyDamaged": 3,
  "qtyLost": 2,
  "notes": "3 rusak kemasan, 2 hilang"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Item checked successfully",
  "data": {
    "itemId": "item_1",
    "product": {
      "sku": "PRD-001",
      "name": "Minyak Goreng 1L"
    },
    "qtyApproved": 100,
    "qtyNormal": 95,
    "qtyDamaged": 3,
    "qtyLost": 2,
    "checkStatus": "CHECKED"
  }
}
```

---

### 5.3 DC Inventory Receive (Setelah Semua Items Checked)

Setelah semua items di-check, sistem otomatis:

```
1. Update DC Inventory:
   - qtyGood += qtyNormal
   - qtyDamaged += qtyDamaged

2. Update Order Management Status:
   - CHECKING → DONE (untuk supplier ini)

3. Update DC Stock Request Status:
   - Jika semua supplier done: SHIPPED → DONE
```

**Endpoint:** Otomatis atau manual via DC Inventory receive

**Manual Endpoint:** `POST /api/v1/dc-inventory/receive`

**Request:**
```json
{
  "distributionCenterId": "dc_jakarta_id",
  "supplierOrderId": "order_management_cuid",
  "notes": "Penerimaan dari Supplier A",
  "items": [
    {
      "productId": "product_1",
      "qtyReceived": 95,
      "qtyDamaged": 3,
      "costPrice": 8500
    }
  ]
}
```

---

## Status Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DC STOCK REQUEST                                │
│                                                                         │
│  ┌───────────┐    ┌───────────┐                     ┌───────────┐       │
│  │  PENDING  │───→│ APPROVED  │────────────────────→│   DONE    │       │
│  └───────────┘    └───────────┘                     └───────────┘       │
│       │                                                                 │
│       │                              ┌───────────┐                      │
│       └─────────────────────────────→│ CANCELLED │                      │
│                                      └───────────┘                      │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                       ORDER MANAGEMENT (Per Supplier)                   │
│                                                                         │
│                   ┌───────────┐    ┌───────────┐    ┌───────────┐       │
│                   │PROCESSING │───→│  SHIPPED  │───→│  ARRIVED  │       │
│                   └───────────┘    └───────────┘    └───────────┘       │
│                                                          │              │
│                                                          ▼              │
│                                                    ┌───────────┐        │
│                                                    │  CHECKING │        │
│                                                    └───────────┘        │
│                                                          │              │
│                                                          ▼              │
│                                                    ┌───────────┐        │
│                                                    │    DONE   │        │
│                                                    └───────────┘        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Perbandingan: Old vs New Flow

### Old Flow (Deprecated - v1.0)
```
DC Request → HO Approve → DC Mark Arrived → DC Start Checking → 
DC Check Item → DC Complete

Endpoints:
- POST /dc-stock-requests/:id/approve
- POST /dc-stock-requests/:id/arrived
- POST /dc-stock-requests/:id/start-checking
- POST /dc-stock-requests/:id/check-item
- POST /dc-stock-requests/:id/complete
```

### New Flow (Order Management v2.0)
```
DC Request → HO Approve Items → HO Create Order (per Supplier) → 
Supplier Process → Shipped → DC Mark Arrived → DC Start Checking → 
DC Check Item → Auto Complete

Endpoints:
- POST /dc-stock-requests/:id/approve-items
- POST /order-management/:id/approve-supplier
- POST /order-management/:id/suppliers/:id/arrived
- POST /order-management/:id/suppliers/:id/checking
- POST /order-management/:id/suppliers/:id/items/:id/check
```

**Perbedaan Utama:**
| Aspek | Old Flow | New Flow |
|-------|----------|----------|
| Supplier Management | Tidak ada | Multi-supplier support |
| PO Document | Setelah approve | Setelah approve supplier |
| Checking | Per request | Per supplier |
| Flexibility | Single supplier | Bisa beda supplier per item |

---

## Contoh Skenario Lengkap

### Skenario: Request Stok DC Jakarta

**Setup:**
- DC: Jakarta (DC-001)
- Supplier A: Minyak Goreng, Gula
- Supplier B: Beras
- HO: WDI_ADMIN

**Timeline:**

| Waktu | Aktor | Aktivitas | Endpoint | Status |
|-------|-------|-----------|----------|--------|
| 08:00 | DC_ADMIN | Create request | `POST /dc-stock-requests` | PENDING |
| 09:00 | WDI_ADMIN | Get supplier groups | `GET /dc-stock-requests/{id}/supplier-groups` | - |
| 09:30 | WDI_ADMIN | Approve items | `POST /dc-stock-requests/{id}/approve-items` | APPROVED |
| 10:00 | WDI_ADMIN | Create order Supplier A | `POST /order-management/{id}/approve-supplier` | SUBMITTED |
| 10:30 | WDI_ADMIN | Create order Supplier B | `POST /order-management/{id}/approve-supplier` | SUBMITTED |
| 11:00 | WDI_ADMIN | Upload PO | `POST /dc-stock-requests/{id}/po-documents` | - |
| Day+1 | WDI_ADMIN | Update status | `PATCH /order-management/{id}/status` | PROCESSING |
| Day+3 | WDI_ADMIN | Mark shipped | `POST /dc-stock-requests/{id}/supplier-order/shipped` | SHIPPED |
| Day+5 | DC_ADMIN | Mark arrived | `POST /order-management/{id}/suppliers/{id}/arrived` | ARRIVED |
| Day+5 | DC_ADMIN | Start checking | `POST /order-management/{id}/suppliers/{id}/checking` | CHECKING |
| Day+5 | DC_ADMIN | Check items | `POST /order-management/{id}/suppliers/{id}/items/{id}/check` | CHECKED |
| Day+5 | DC_ADMIN | Receive to inventory | `POST /dc-inventory/receive` | DONE |

---

## Endpoint Summary

### DC_ADMIN Endpoints:
| Method | Endpoint | Fungsi |
|--------|----------|--------|
| POST | `/dc-stock-requests` | Create request |
| GET | `/dc-stock-requests` | List requests |
| POST | `/dc-stock-requests/{id}/cancel` | Cancel request |
| GET | `/order-management` | List orders |
| POST | `/order-management/{id}/suppliers/{id}/arrived` | Mark arrived |
| POST | `/order-management/{id}/suppliers/{id}/checking` | Start checking |
| POST | `/order-management/{id}/suppliers/{id}/items/{id}/check` | Check item |
| POST | `/dc-inventory/receive` | Receive stock |

### WDI_ADMIN Endpoints:
| Method | Endpoint | Fungsi |
|--------|----------|--------|
| GET | `/dc-stock-requests` | List all requests |
| GET | `/dc-stock-requests/{id}/supplier-groups` | Get by supplier |
| POST | `/dc-stock-requests/{id}/approve-items` | Approve items |
| POST | `/order-management/{id}/approve-supplier` | Create order |
| POST | `/dc-stock-requests/{id}/po-documents` | Upload PO |
| PATCH | `/order-management/{id}/status` | Update status |
| POST | `/dc-stock-requests/{id}/supplier-order/shipped` | Mark shipped |

---

*Dokumen ini dibuat untuk Arcson POS Backend*
*Tanggal: 2026-02-09*
*Version: Order Management v2.0*

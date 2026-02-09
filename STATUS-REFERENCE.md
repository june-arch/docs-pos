# Status Reference - Arcson POS Backend

Dokumen ini berisi daftar lengkap semua status di sistem POS Backend.

---

## 1. Daily Operation Status (CargoBikeDailyOperation)

Status untuk operasional harian cargo bike.

| Status | Kode | Deskripsi | Actor Trigger |
|--------|------|-----------|---------------|
| **STARTED** | `STARTED` | Daily operation baru dibuat setelah start day | WD_MOBILE (user start day) |
| **STOCK_REQUESTED** | `STOCK_REQUESTED` | User sudah membuat stock request | WD_MOBILE (create request) |
| **STOCK_ALLOCATED** | `STOCK_ALLOCATED` | DC sudah allocate stock | DC_ADMIN (allocate) |
| **ON_DUTY** | `ON_DUTY` | User sedang berjualan | Auto (setelah allocate + transaction) |
| **END_DAY_CHECKING** | `END_DAY_CHECKING` | DC sedang melakukan end day check | DC_ADMIN (start end day) |
| **COMPLETED** | `COMPLETED` | Operasional hari ini selesai | Auto (setelah check complete + payment) |

### Flow Status Daily Operation:
```
STARTED 
    ↓ (User request stock)
STOCK_REQUESTED 
    ↓ (DC allocate)
STOCK_ALLOCATED 
    ↓ (User jualan)
ON_DUTY 
    ↓ (DC start check)
END_DAY_CHECKING 
    ↓ (Check complete)
    ├─→ Ada shortage: WAITING_PAYMENT → COMPLETED (after payment)
    └─→ Tidak ada shortage: COMPLETED
```

---

## 2. Stock Request Status (CargoBikeStockRequest)

Status untuk request stok dari cargo bike ke DC.

| Status | Kode | Deskripsi | Actor Trigger |
|--------|------|-----------|---------------|
| **PENDING** | `PENDING` | Request baru dibuat, menunggu DC | WD_MOBILE (create request) |
| **APPROVED** | `APPROVED` | DC sudah approve dan allocate | DC_ADMIN (allocate) |
| **REJECTED** | `REJECTED` | DC menolak request | DC_ADMIN (reject - jika ada) |
| **PARTIAL** | `PARTIAL` | Partial allocation (jika implement) | DC_ADMIN |

### Flow Status Stock Request:
```
PENDING 
    ↓ (DC allocate)
APPROVED
```

**Catatan:** Saat ini hanya ada PENDING → APPROVED. Reject/partial opsional.

---

## 3. Shift Assignment Status (ShiftAssignment)

Status untuk penjadwalan shift cargo bike.

| Status | Kode | Deskripsi | Actor Trigger |
|--------|------|-----------|---------------|
| **SCHEDULED** | `SCHEDULED` | Shift sudah diassign ke user | DC_ADMIN (create assignment) |
| **ACTIVE** | `ACTIVE` | User sudah start shift | WD_MOBILE (start shift) |
| **COMPLETED** | `COMPLETED` | User end shift normal | WD_MOBILE (end shift) |
| **AUTO_ENDED** | `AUTO_ENDED` | Shift berakhir otomatis oleh sistem | System (auto logout) |
| **CANCELLED** | `CANCELLED` | Shift dibatalkan sebelum mulai | DC_ADMIN (delete assignment) |

### Flow Status Shift Assignment:
```
SCHEDULED 
    ↓ (User start)
ACTIVE 
    ↓ (User end / Auto end)
    ├─→ COMPLETED (manual end)
    └─→ AUTO_ENDED (system timeout)
```

---

## 4. End Day Check Status (CargoBikeEndDayCheck)

Status untuk proses pengecekan akhir shift.

| Status | Kode | Deskripsi | Actor Trigger |
|--------|------|-----------|---------------|
| **CHECKING** | `CHECKING` | DC sedang melakukan pengecekan | DC_ADMIN (start end day) |
| **WAITING_PAYMENT** | `WAITING_PAYMENT` | Ada shortage, menunggu pembayaran | Auto (check complete with shortage) |
| **COMPLETED** | `COMPLETED` | Pengecekan selesai | WD_MOBILE (pay) / Auto (no shortage) |

### Flow Status End Day Check:
```
CHECKING 
    ↓ (Check items complete)
    ├─→ totalLossValue > 0: WAITING_PAYMENT 
    │                           ↓ (User pay)
    │                       COMPLETED
    │
    └─→ totalLossValue = 0: COMPLETED
```

---

## 5. End Day Item Status (CargoBikeEndDayItem)

Status untuk setiap item dalam end day check.

| Status | Kode | Deskripsi | Kondisi |
|--------|------|-----------|---------|
| **PENDING** | `PENDING` | Item belum dicek | Initial status |
| **OK** | `OK` | Item sesuai (actual + damaged = expected) | actualQty + damagedQty = expectedQty |
| **SHORTAGE** | `SHORTAGE` | Ada kekurangan | actualQty + damagedQty < expectedQty |

### Perhitungan:
```
expectedQty = morningQty - soldQty
missingQty = expectedQty - actualQty - damagedQty

Status:
- OK: missingQty = 0
- SHORTAGE: missingQty > 0
```

---

## 6. DC Stock Request Status (DCStockRequest)

Status untuk request stok dari DC ke Head Office (WDI_ADMIN).

| Status | Kode | Deskripsi | Actor Trigger |
|--------|------|-----------|---------------|
| **PENDING** | `PENDING` | Request dari DC, menunggu HO approve | DC_ADMIN (create request) |
| **APPROVED** | `APPROVED` | HO approve request | WDI_ADMIN (approve) |
| **REJECTED** | `REJECTED` | HO tolak request | WDI_ADMIN (reject) |
| **PROCESSING** | `PROCESSING` | Sedang diproses | WDI_ADMIN (update status) |
| **SHIPPED** | `SHIPPED` | Barang sudah dikirim supplier | WDI_ADMIN / System |
| **ARRIVED** | `ARRIVED` | Barang sampai di DC | DC_ADMIN (mark arrived) |
| **CHECKING** | `CHECKING` | DC sedang cek barang | DC_ADMIN (start checking) |
| **DONE** | `DONE` | Penerimaan selesai | DC_ADMIN (complete receive) |
| **CANCELLED** | `CANCELLED` | Request dibatalkan | DC_ADMIN (cancel) |

### Flow Status DC Stock Request:
```
PENDING 
    ↓ (HO approve)
APPROVED 
    ↓ (HO process)
PROCESSING 
    ↓ (Supplier kirim)
SHIPPED 
    ↓ (Barang sampai)
ARRIVED 
    ↓ (DC mulai cek)
CHECKING 
    ↓ (Cek selesai)
DONE

Alternative:
PENDING → REJECTED (HO tolak)
PENDING → CANCELLED (DC batal)
```

---

## 7. Supplier Order Status (SupplierOrder)

Status untuk order ke supplier (via Order Management v2.0).

| Status | Kode | Deskripsi |
|--------|------|-----------|
| **SUBMITTED** | `SUBMITTED` | Order dibuat |
| **PROCESSING** | `PROCESSING` | Sedang diproses supplier |
| **SHIPPED** | `SHIPPED` | Barang dikirim |
| **ARRIVED** | `ARRIVED` | Barang sampai di DC |
| **CHECKING** | `CHECKING` | Sedang dicek |
| **DONE** | `DONE` | Selesai |

---

## 8. Transaction Status (Transaction)

Status untuk transaksi penjualan.

| Status | Kode | Deskripsi |
|--------|------|-----------|
| **PENDING** | `PENDING` | Transaksi dalam proses |
| **COMPLETED** | `COMPLETED` | Transaksi selesai |
| **CANCELLED** | `CANCELLED` | Transaksi dibatalkan |

---

## 9. Transaction Type (Transaction)

| Type | Kode | Deskripsi |
|------|------|-----------|
| **SALE** | `SALE` | Penjualan normal |
| **RETURN** | `RETURN` | Retur barang |
| **VOID** | `VOID` | Transaksi dibatalkan |

---

## 10. Payment Method (Transaction)

| Method | Kode | Deskripsi |
|--------|------|-----------|
| **CASH** | `CASH` | Tunai |
| **QRIS** | `QRIS` | QRIS payment |
| **BANK_TRANSFER** | `BANK_TRANSFER` | Transfer bank |
| **OTHER** | `OTHER` | Lainnya |

---

## 11. DOKU Wallet Transaction Type (DokuWalletTransaction)

| Type | Kode | Deskripsi |
|------|------|-----------|
| **TOPUP** | `TOPUP` | Isi saldo |
| **PAYMENT** | `PAYMENT` | Pembayaran |
| **TRANSFER** | `TRANSFER` | Transfer (jika ada) |

---

## 12. User Role (User)

| Role | Kode | Deskripsi | Akses |
|------|------|-----------|-------|
| **WDI_ADMIN** | `WDI_ADMIN` | Head Office Admin | Full access |
| **DC_ADMIN** | `DC_ADMIN` | Distribution Center Admin | DC scope |
| **WD_MOBILE** | `WD_MOBILE` | Warung Desa Mobile (Cargo Bike) | Mobile only |

---

## 13. Cargo Bike Status (CargoBike)

| Status | Kode | Deskripsi |
|--------|------|-----------|
| **ACTIVE** | `ACTIVE` | Aktif |
| **INACTIVE** | `INACTIVE` | Non-aktif |
| **MAINTENANCE** | `MAINTENANCE` | Sedang perawatan |
| **SUSPENDED** | `SUSPENDED` | Ditangguhkan |

---

## Ringkasan Flow Status Utama

### Daily Operation Flow:
```
STARTED → STOCK_REQUESTED → STOCK_ALLOCATED → ON_DUTY → 
END_DAY_CHECKING → (WAITING_PAYMENT) → COMPLETED
```

### Shift Assignment Flow:
```
SCHEDULED → ACTIVE → COMPLETED / AUTO_ENDED
```

### DC Stock Request Flow:
```
PENDING → APPROVED → PROCESSING → SHIPPED → ARRIVED → CHECKING → DONE
```

### End Day Check Flow:
```
CHECKING → (WAITING_PAYMENT) → COMPLETED
```

---

*Dokumen ini dibuat untuk Arcson POS Backend*
*Tanggal: 2026-02-09*

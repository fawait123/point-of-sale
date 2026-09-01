# POS System — Product Requirements Document

## Versi
- Versi: 1.0
- Tanggal: 2025-08-09
- Status: Draft

## Ringkasan / Overview
Aplikasi Point of Sale (POS) untuk operasional usaha ritel/merchant. Mengintegrasikan empat alur bisnis inti — penjualan, stok (inventory), pembukuan (akuntansi), dan laporan — dalam satu sistem terpusat. Nilai bisnis: transaksi real-time tanpa kertas, stok akurat (hindari oversell), rekonsiliasi kas otomatis, dan laporan keuangan yang bisa di-query per periode. Menghilangkan kebutuhan spreadsheet manual dan mengurangi error hitung manual.

## Sasaran & Non-Sasaran
- Sasaran:Owner/operator kasir, admin stok, akuntan, pemilik usaha kecil-menengah (SME).
- Non-Sasaran:Perusahaan multi-gudang enterprise dengan ERP kompleks, e-commerce marketplace, pihak ketiga yang butuh integrasi API dua arah penuh.

## Persona & Use Case
- Persona utama: Kasir (input transaksi cepat, scan barcode, cetak struk). Admin Stok (stok masuk/keluar, alert low stock). Akuntan (jurnal, rekonsiliasi). Pemilik (laporan per periode).
- Use case inti: Kasir scan produk → sistem cek stok → checkout → kurangi stok → simpan ke database → backend catat ke journal akuntansi. Admin dapat notifikasi low stock. Pemilik buka laporan penjualan bulan ini.

## Persyaratan Fungsional (Requirements)

### FR-1: Penjualan (Sales)
- Deskripsi: Kreasi transaksi baru, pencarian produk (autocomplete), edit/transaksi, hapus, cetak struk (termals), export CSV/Excel. Status transaksi (Bayar/Piutang).
- Acceptance criteria:
  - Given user buka form penjualan, When ketik nama produk, Then muncul daftar produk matching dari tabel produk.
  - Given stok produk 5 dan cart ada 6, When user coba tambah, Then sistem tolak dan tampilkan pesan insufficient stock.
  - Given transaksi disimpan, When dibuka lagi, Then nomor invoice dan total muncul di list transaksi.

### FR-2: Inventory (Inventory)
- Deskripsi: CRUD produk, kategori, harga beli/jual, stok mutasi (stok masuk/keluar), stok opname, alert low stock.
- Acceptance criteria:
  - Given produk stok 2, When transaksi jual 1, Then stok jadi 1 dan tercatat di stock_movements.
  - Given threshold 3, When stok <= 3, Then muncul banner low stock di dashboard.

### FR-3: Akuntansi (Accounting)
- Deskripsi: Chart of accounts, jurnal otomatis per transaksi, entri manual, rekonsiliasi kas, laba rugi.
- Acceptance criteria:
  - Given transaksi tunai tersimpan, When jurnal di-generate, Then debit Kas dan kredit Penjualan tercatat otomatis dengan balance 0.

### FR-4: Laporan Penjualan (Reports)
- Deskripsi: Filter perhari/perminggu/perbulan/pertahun, total penjualan, produk terlaris, perbandingan periode (YoY/MoM). Export.
- Acceptance criteria:
  - Given pilih bulan, When klik laporan, Then total penjualan dan produk terlaris muncul sesuai rentang tanggal.

## Persyaratan Non-Fungsional
- Performa: Query sales dengan filter tanggal max 2 detik, list produk autocomplete < 300ms.
- Keamanan: JWT auth, RBAC (kasir/admin/akuntan), input sanitized, semua endpoint di-protect.
- Skalabilitas: Backend stateless, DB connection pool, horizontal scale via load balancer.
- Reliability: Backup DB harian, error logging.
- Kompatibilitas: VPS Linux (Debian/Ubuntu), Go 1.22+, Vue 3.

## Arsitektur & Struktur Teknis

### 1. Frontend
**Struktur Folder & File (feature-driven):**
```
src/
  assets/
  components/
    ui/            # shadcn-vue primitives (button, input, dialog, table)
    product/       # ProductCard, ProductModal, ProductSearch
    sale/          # SaleForm, SaleTable, SaleItemRow, StrukModal
    inventory/     # InventoryTable, StockMovementModal, LowStockBanner
    accounting/    # JournalView, AccountPicker, RecconciliationTable
    report/        # ReportFilters, ReportChart, ReportTable
  composables/
    useProducts.ts
    useSales.ts
    useInventory.ts
    useAccounting.ts
    useReports.ts
  views/
    login/
    sales/
    inventory/
    accounting/
    reports/
  lib/
    api/           # axios instance + endpoint per fitur
    store/         # pinia stores
  router/
  types/           # Product, Sale, Journal, Report, Account
  plugins/         # pinia, router, axios interceptor
```
**Saran Library/Package:**
- `vue` 3.5 — reactive frontend core.
- `vue-router` — navigasi antar modul.
- `pinia` — state management (products, sales, inventory, accounting).
- `axios` — HTTP client ke backend.
- `vuetify` atau `element-plus` — UI component library (table, form, dialog).
- `vue-sonner` — toast notifikasi.
- `tailwindcss` — styling utility.
- `day.js` — format/filter tanggal.
- `chart.js` + `vue-chartjs` — grafik laporan.

### 2. Backend
**Struktur Folder & File:**
```
cmd/api/main.go
internal/
  config/          # load env (DATABASE_URL, PORT)
  database/        # pgx connection, migrations
  models/          # struct: Product, Sale, SaleItem, StockMovement, Journal, Account
  repo/
    product_repo.go
    sale_repo.go
    inventory_repo.go
    accounting_repo.go
    report_repo.go
  service/
    product_service.go
    sale_service.go
    inventory_service.go
    accounting_service.go
    report_service.go
  handler/
    product_handler.go
    sale_handler.go
    inventory_handler.go
    accounting_handler.go
    report_handler.go
  router/          # wire routes
  middleware/      # auth, rbac
pkg/
go.mod
```
**Endpoint Contract:**

**Penjualan**
- `POST /api/v1/sales`
  - Request:
    ```json
    { "items": [ {"productId": 1, "qty": 2, "price": 10000}, {"productId": 2, "qty": 1, "price": 50000} ], "paymentMethod": "cash", "customerId": 5 }
    ```
  - Response:
    ```json
    { "id": "INV-2025-0001", "items": [...], "subtotal": 50000, "total": 50000, "status": "paid", "createdAt": "2025-08-09T10:00:00Z" }
    ```
- `GET /api/v1/sales?from=&to=&status=`
  - Response: `{ "sales": [ { "id": "INV-...", "date": "...", "items": [...], "total": 50000 } ] }`
- `GET /api/v1/sales/{id}` → detail satu transaksi.
- `PUT /api/v1/sales/{id}` → edit.
- `DELETE /api/v1/sales/{id}` → hapus.
- `GET /api/v1/sales/{id}/receipt` → struk (PDF/JSON).

**Inventory**
- `GET /api/v1/products`, `POST /api/v1/products`, `PUT /api/v1/products/{id}`, `DELETE /api/v1/products/{id}`.
- `POST /api/v1/stock-movements`
  - Request: `{ "productId": 1, "type": "in|out", "qty": 3, "note": "pembelian" }`
  - Response: `{ "id": 1, "productId": 1, "type": "in", "qty": 3, "balance": 10, "note": "pembelian" }`

**Akuntansi**
- `GET /api/v1/accounts` → daftar chart of accounts.
- `POST /api/v1/journals`
  - Request: `{ "entries": [ {"accountId": 1, "debit": 50000, "credit": 0}, {"accountId": 2, "debit": 0, "credit": 50000} ], "narration": "Penjualan tunai INV-0001", "saleId": "INV-0001" }`
  - Response: `{ "id": 1, "entries": [...], "balance": 0, "date": "2025-08-09" }`
- `POST /api/v1/journals/generate/{saleId}` → generate jurnal otomatis dari transaksi.
- `GET /api/v1/journals/reconciliation/{accountId}` → rekonsiliasi.

**Laporan**
- `GET /api/v1/reports/sales?period=daily|weekly|monthly|yearly&date=`
  - Response: `{ "period": "monthly", "date": "2025-08", "totalSales": 1200000, "transactions": [...], "topProducts": [ { "productId": 1, "name": "A", "qty": 120, "revenue": 1200000 } ] }`

### 3. Database
**Struktur Tabel (PostgreSQL):**
```sql
-- users: kasir/admin/akuntan
users(id uuid PK, username text, password_hash text, role text, created_at timestamptz)

-- chart of accounts
accounts(id uuid PK, code text, name text, type text, balance numeric)

-- products
products(id uuid PK, sku text, name text, category_id uuid, cost_price numeric, selling_price numeric, stock integer, threshold integer, created_at timestamptz)

-- categories (FK ke products)
categories(id uuid PK, name text)

-- sales (header)
sales(id uuid PK, invoice_no text, user_id uuid FK->users, customer_id uuid FK->customers, subtotal numeric, discount numeric, total numeric, payment_method text, status text, created_at timestamptz)

-- sale_items
sale_items(id uuid PK, sale_id uuid FK->sales, product_id uuid FK->products, qty integer, price numeric, subtotal numeric, created_at timestamptz)

-- customers
customers(id uuid PK, name text, phone text, address text, balance numeric)

-- stock_movements
stock_movements(id uuid PK, product_id uuid FK->products, movement_type text, qty integer, balance_after integer, note text, created_at timestamptz)

-- journals
journals(id uuid PK, date date, narration text, user_id uuid FK->users, sale_id uuid FK->sales, is_auto boolean, created_at timestamptz)

-- journal_entries
journal_entries(id uuid PK, journal_id uuid FK->journals, account_id uuid FK->accounts, debit numeric, credit numeric, created_at timestamptz)

-- index: sales(created_at), sale_items(sale_id), stock_movements(product_id), journals(date), journal_entries(journal_id)
```
**Relasi:** users → sales (1:N). products → sale_items (1:N) dan products → stock_movements (1:N). sales → journals (1:1) dan journals → journal_entries (1:N), constrained debit=credit per journal. accounts → journal_entries (N:1).

## Batasan & Risiko
- Batasan teknis: VPS manual tanpa auto-scaling; single instance jadi bottleneck saat concurrency tinggi. Mitigasi: connection pool, query index, cache Redis opsional.
- Batasan bisnis: Tidak support multi-gudang/multi-mata-mono; asumsi merchant single location.
- Risiko: Oversell (stok jadi negatif). Mitigasi: guard di service layer sebelum commit transaksi.
- Risiko: Inkonsistensi kas vs penjualan. Mitigasi: jurnal otomatis + rekonsiliasi.
- Risiko: Data loss. Mitigasi: backup DB harian + WAL.

## Task Breakdown (Outline)
- T1: Setup VPS, Go backend, PostgreSQL, Vue frontend skeleton.
- T2: Auth (login, JWT, RBAC).
- T3: Product CRUD + kategori + low stock alert.
- T4: Sales CRUD + autocomplete + stock guard + struk.
- T5: Stock movements + stock_movements table.
- T6: Chart of accounts + journal manual.
- T7: Jurnal otomatis dari transaksi (debit/credit).
- T8: Rekonsiliasi kas.
- T9: Report engine (daily/weekly/monthly/yearly) + export.
- T10: Deployment, backup, monitoring.
- T11: Testing end-to-end.

</PRD>
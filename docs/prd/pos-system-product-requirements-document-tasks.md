---
prd: "POS System — Product Requirements Document"
task_count: 13
---
## T1. Schema database PostgreSQL

### Deskripsi
Buat semua tabel sesuai spesifikasi: users, accounts (chart of accounts), products, categories, sales, sale_items, customers, stock_movements, journals, journal_entries, plus index pada sales(created_at), sale_items(sale_id), stock_movements(product_id), journals(date), journal_entries(journal_id). Terapkan relasi FK dan constraint debit=credit pada journal_entries.

### Acceptance Criteria
- [ ] Migration berjalan tanpa error; query join products->sale_items dan journals->journal_entries mengembalikan data benar; constraint balance 0 terpenuhi pada journal otomatis.

## T2. Scaffolding frontend Vue 3.5

### Deskripsi
Inisialisasi proyek Vue 3.5 + vue-router + pinia + axios + tailwindcss. Bangun struktur folder feature-driven (components: ui, product, sale, inventory, accounting, report; composables useProducts/useSales/useInventory/useAccounting/useReports; views: login/sales/inventory/accounting/reports; lib/api; store; types; plugins). Buat type definitions untuk Product, Sale, Journal, Report, Account.

### Acceptance Criteria
- [ ] npm install sukses; router dapat navigasi ke setiap view; pinia store ter-register di plugins; axios instance tersedia dengan interceptor; typecheck tanpa error.

## T3. Sistem autentikasi JWT dan RBAC

### Deskripsi
Buat endpoint login, issue JWT, dan middleware RBAC yang memprotect semua endpoint. Middleware memvalidasi role (kasir/admin/akuntan) dan mengunci akses sesuai role. Frontend simpan token, kirim Authorization header, tampilkan login saat token tidak valid.

### Acceptance Criteria
- [ ] Request ke endpoint tanpa token dapat 401; request dengan token valid dan role cocok dapat 200; request dengan role tidak sesuai dapat 403; JWT ter-signatur dan tidak bisa dipalsukan.

## T4. Manajemen produk dan kategori

### Deskripsi
Implementasi CRUD produk (sku, name, category_id, cost_price, selling_price, stock, threshold) dan kategori. Backend expose GET/POST/PUT/DELETE products dan categories. Frontend sediakan InventoryTable, ProductModal, ProductSearch, dan LowStockBanner.

### Acceptance Criteria
- [ ] POST produk dengan threshold 3 disimpan dan tercatat; GET products dengan filter kategori mengembalikan hasil benar; banner low stock muncul di dashboard ketika stok <= threshold.

## T5. Penjualan: kreasi dan autocomplete

### Deskripsi
Implementasi POST /api/v1/sales menerima items [{productId, qty, price}] + paymentMethod + customerId, hitung subtotal/total, generate invoice_no, simpan sale header + sale_items. Frontend sediakan SaleForm dengan autocomplete produk (autocomplete < 300ms) dan cart. GET /api/v1/sales?from=&to=&status= untuk list transaksi.

### Acceptance Criteria
- [ ] POST penjualan dengan 2 item menghasilkan invoice INV-2025-0001, subtotal dan total benar; autocomplete menampilkan daftar produk matching dalam < 300ms; transaksi tersimpan muncul di list dengan invoice_no dan total.

## T6. Guard stok dan struk

### Deskripsi
Tambahkan guard di service layer yang menolak penambahaan jika stok tidak cukup (misal stok 5, cart 6 → tolak dengan pesan insufficient stock) sebelum commit. Implementasi GET /api/v1/sales/{id}/receipt untuk cetak struk (termsal) dan export CSV/Excel.

### Acceptance Criteria
- [ ] Transaksi dengan qty melebihi stok menolak dan mengembalikan pesan insufficient stock; transaksi valid commit dan mengurangi stok di database; endpoint receipt menghasilkan struk yang bisa dicetak.

## T7. Stok mutasi

### Deskripsi
Implementasi POST /api/v1/stock-movements menerima {productId, type: in|out, qty, note}, hitung balance_after, catat ke tabel stock_movements. Frontend sediakan StockMovementModal.

### Acceptance Criteria
- [ ] POST stock movement type in qty 3 dengan balance 10 sebelumnya menghasilkan balance_after 13 dan tercatat di stock_movements; type out mengurangi balance; GET stock_movements per produk mengembalikan riwayat benar.

## T8. Chart of accounts dan jurnal manual

### Deskripsi
Implementasi GET /api/v1/accounts untuk daftar chart of accounts, POST /api/v1/journals untuk entri jurnal manual dengan entries [{accountId, debit, credit}] + narration, validasi balance 0. Frontend sediakan JournalView dan AccountPicker.

### Acceptance Criteria
- [ ] GET accounts mengembalikan daftar account; POST jurnal manual dengan debit 50000 credit 0 untuk satu account dan debit 0 credit 50000 untuk account lain menghasilkan balance 0; POST jurnal tidak balance menolak.

## T9. Jurnal otomatis dari transaksi

### Deskripsi
Implementasi POST /api/v1/journals/generate/{saleId} yang generate jurnal otomatis dari transaksi: debit Kas dan kredit Penjualan sesuai total transaksi, is_auto=true, balance 0. Kaitkan sale_id ke journals.

### Acceptance Criteria
- [ ] Generate jurnal dari INV-0001 total 50000 menghasilkan debit Kas 50000 dan kredit Penjualan 50000 dengan balance 0; is_auto=true; sale_id terisi; total debit = total kredit.

## T10. Rekonsiliasi kas

### Deskripsi
Implementasi GET /api/v1/journals/reconciliation/{accountId} yang menghitung selisih debit dan credit per account, menampilkan entri jurnal yang terkait, dan menandai ketidaksesuaian kas vs penjualan.

### Acceptance Criteria
- [ ] Rekonsiliasi account Kas menampilkan total debit dan kredit beserta selisihnya; account dengan balance 0 ditandai sesuai; account tidak balance ditandai inkonsisten.

## T11. Report engine dan export

### Deskripsi
Implementasi GET /api/v1/reports/sales?period=daily|weekly|monthly|yearly&date= yang menghitung total penjualan, produk terlaris, dan perbandingan periode (YoY/MoM) dalam rentang tanggal. Frontend sediakan ReportFilters, ReportChart (chart.js), ReportTable, ReportFilters, dan export CSV/Excel.

### Acceptance Criteria
- [ ] Query dengan filter monthly date 2025-08 mengembalikan totalSales, transaksi, dan topProducts sesuai rentang; query dengan filter tanggal max 2 detik; export CSV/Excel menghasilkan file yang bisa dibuka.

## T12. Deployment, backup, dan monitoring

### Deskripsi
Deploy backend Go 1.22+ di VPS Linux (Debian/Ubuntu), setup PostgreSQL dengan connection pool, proteksi endpoint, dan horizontal scale via load balancer. Konfigurasi backup DB harian + WAL dan error logging.

### Acceptance Criteria
- [ ] Backend berjalan di VPS dengan DATABASE_URL dan PORT dari env; connection pool aktif; backup DB harian tercatat; error logging menangkap kegagalan; endpoint accessible setelah auth.

## T13. Testing end-to-end

### Deskripsi
Buat suite test yang memverifikasi alur lengkap: login → kreasi penjualan → stok berkurang + jurnal otomatis → laporan. Test guard stok, balance jurnal, dan query report. Test frontend untuk autocomplete, checkout, dan render laporan.

### Acceptance Criteria
- [ ] Test end-to-end berjalan hijau dari login sampai laporan; guard stok gagal saat stok tidak cukup; jurnal otomatis balance 0; report mengembalikan angka benar untuk periode yang dipilih.

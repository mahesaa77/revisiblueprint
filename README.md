# Toko Pakaian — Platform Retail Pakaian

Monorepo modular monolith (NestJS + Next.js) untuk toko pakaian: katalog via scan QR, kasir (POS),
reservasi dengan DP, inventori, dan akuntansi double-entry otomatis. Scope mengikuti
[`Blueprint_Arsitektur_Website_Toko_Pakaian.docx`](./Blueprint_Arsitektur_Website_Toko_Pakaian.docx);
rencana fase ada di [`PLAN.md`](./PLAN.md).

## Status fitur

| Modul | Sudah berjalan | Belum / ditunda |
|---|---|---|
| **M1 Katalog** | Cari + filter kategori + paginasi, detail via SKU, stok real-time, payload QR per SKU, produk terkait | Gambar multi-angle, wishlist, pemindaian kamera (ZXing) |
| **M2 POS** | Transaksi, diskon, PPN (konfigurabel), split payment, kembalian, idempotency key, `batch-sync`, retur | IndexedDB/Dexie + Workbox, shift kasir, cetak struk thermal |
| **M3 Reservasi** | Konfigurasi DP admin (persen/nominal), hold stok 24–48 jam, bayar DP, lunasi, batal, auto-release kedaluwarsa | Kalender fitting, reminder WhatsApp, Midtrans, kebijakan refund |
| **M4 Inventori** | Stok tidak boleh minus, hold/lepas/konfirmasi, kartu stok, stok masuk | BOM, bahan baku, opname, transfer outlet |
| **M5 Akuntansi** | Jurnal double-entry otomatis dari event, jurnal immutable & seimbang, neraca saldo, laba rugi sederhana, buku besar | Neraca, arus kas, export PDF/Excel, HPP FIFO |
| **Auth/RBAC** | — | JWT + 5 role (belum ada; endpoint saat ini terbuka) |
| **Database** | — | Penyimpanan masih in-memory (reset saat restart); Prisma + PostgreSQL berikutnya |

> **Peringatan:** Chart of Accounts adalah **placeholder** (`isPlaceholder: true`) dan belum divalidasi
> akuntan. Laporan tidak boleh dipakai untuk keputusan pajak. PPN default 0% sampai aturan fiskal diputuskan.

## Arsitektur singkat

```
apps/api            NestJS, prefix /api/v1
  src/platform      error, event bus (port + adapter in-memory), config, filter HTTP, health
  src/modules/<m>   domain/ application/ infrastructure/ presentation/ per bounded context
apps/web            Next.js 15 (route group customer & kasir)
packages/contracts  skema Zod bersama (validasi API + typing client)
```

- Lapisan `application` adalah kelas TypeScript biasa tanpa decorator NestJS — mudah diuji dan
  dipindah ke service terpisah kelak. NestJS hanya dipakai di `presentation` dan `*.module.ts`.
- Antar-modul hanya berbicara lewat service yang diekspor atau **event** (`transaksi.dibuat`,
  `transaksi.retur`, `reservasi.dp-dibayar`, `reservasi.lunas`). Akuntansi berlangganan event-event itu,
  jadi tidak ada jalur input jurnal manual.
- Semua error memakai amplop `{ statusCode, code, message, details? }`.

## Menjalankan

Prasyarat: Node ≥ 22, pnpm ≥ 10.

```bash
pnpm install
cp apps/api/.env.example apps/api/.env      # opsional; ada default
pnpm --filter @toko/contracts build
pnpm dev                                    # API :3000, Web :3001
```

Buka <http://localhost:3001> (katalog), `/kasir` (POS), `/reservasi`. Contoh SKU: `KMJ-OXF-BIR-M`, `KAO-BSC-HTM-M`.

Dengan Docker: `docker compose up --build`.

## Perintah

```bash
pnpm typecheck      # contracts -> api -> web
pnpm test           # Jest: unit + integrasi + e2e API
pnpm build
pnpm format:check
```

## Contoh API

```bash
curl 'localhost:3000/api/v1/produk?kategori=kaos'
curl -X POST localhost:3000/api/v1/transaksi -H 'content-type: application/json' -d '{
  "idempotencyKey":"demo-0001",
  "items":[{"sku":"KAO-BSC-HTM-M","qty":2}],
  "pembayaran":[{"metode":"TUNAI","jumlah":200000}]}'
curl localhost:3000/api/v1/laporan/neraca-saldo
```

Daftar lengkap endpoint: [`docs/API.md`](./docs/API.md).

## Pengujian

Jest mencakup: aturan stok (tidak minus, all-or-nothing), perhitungan harga/pajak/split payment,
idempotensi transaksi dan `batch-sync`, retur, jurnal seimbang & immutable, alur reservasi + auto-release
(dengan jam yang bisa dimajukan), serta e2e HTTP (validasi Zod, amplop error, alur kasir dan reservasi).

## Konvensi

Dokumen dan commit berbahasa Indonesia: `<tipe>(<modul>): <deskripsi>`, mis. `feat(penjualan): tambah retur`.
Jangan commit file `.env`; gunakan `.env.example`.

## Batasan yang diketahui

- Data hilang saat API restart (in-memory); konkurensi dijamin hanya di satu proses.
- Retur memakai harga setelah diskon proporsional; pajak reservasi belum dihitung.
- Pembatalan setelah DP dibayar ditolak sampai kebijakan refund ditentukan.
- Offline POS memakai `localStorage` sebagai antrian sementara, bukan IndexedDB.

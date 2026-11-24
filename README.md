# Diecast Hub — Discover, Collect & Trade

Prototype interaktif marketplace diecast multi-vendor dengan **3D Product Viewer (Three.js)** dan **Admin Dashboard** ber-CRUD. Seluruh aplikasi berada dalam satu file: `index.html` (HTML5 + CSS3 + Vanilla JS + Three.js). Tidak ada build step sama sekali.

## Menjalankan website

Three.js dimuat sebagai ES Module dari CDN, jadi jalankan lewat local server (bukan `file://`) dan pastikan ada koneksi internet.

```bash
# dari folder proyek
python3 -m http.server 3000
# buka http://localhost:3000/   (otomatis membuka index.html)
```

Alternatif: `npx serve .`, ekstensi Live Server VS Code, atau server statis apa pun.

`diecast_hub.html` hanya halaman redirect kecil ke `index.html`, supaya tautan lama tetap jalan.

## Deploy ke Vercel + pasang domain

Proyek ini statis murni (satu `index.html`, tanpa build). `vercel.json` hanya berisi **security header** (CSP, HSTS, `nosniff`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`) dan `cleanUrls` — tidak ada konfigurasi build.

1. Pastikan `index.html`, `README.md`, dan `.gitignore` sudah ada di repositori GitHub (branch `main`).
2. Buka [vercel.com/new](https://vercel.com/new) → **Import** repositori `diecast_hub`.
3. Isi pengaturan:
   - **Framework Preset:** `Other`
   - **Build Command:** kosongkan (atau `None`)
   - **Output Directory:** kosongkan (root)
   - **Install Command:** kosongkan
4. Klik **Deploy**. Situs akan tampil di `https://<nama-proyek>.vercel.app/` — bukan di `/index.html`, karena Vercel otomatis menyajikan `index.html` untuk root.
5. Setiap `git push` ke `main` akan otomatis di-deploy ulang.

Memasang domain sendiri:

1. Buka proyek di Vercel → **Settings → Domains → Add**.
2. Masukkan domain (mis. `diecasthub.com`) dan pilih apakah `www` juga dipakai.
3. Ikuti instruksi DNS yang muncul di registrar domain Anda:
   - Domain apex (`diecasthub.com`): record **A** → `76.76.21.21`
   - Subdomain / `www`: record **CNAME** → `cname.vercel-dns.com`
4. Tunggu propagasi DNS (biasanya beberapa menit) sampai status domain **Valid Configuration**. HTTPS diterbitkan otomatis oleh Vercel.

Catatan: foto produk dan model memakai URL eksternal (CDN Sketchfab, toko online), jadi koneksi internet tetap dibutuhkan setelah deploy.

**Kebutuhan browser:** WebGL aktif (Chrome, Edge, Firefox, Safari modern). Touch control bekerja pada perangkat sentuh. Jika WebGL atau CDN tidak tersedia, viewer menampilkan pesan fallback dan foto/spesifikasi produk tetap bisa dibuka.

## Login administrator

- Buka tombol **Admin** di navigasi, atau tautan **Admin Dashboard** di footer.
- Username: `admin`
- Password: `admin123` (kredensial demo — **ganti setelah login pertama**)

Sesi login disimpan di `sessionStorage` dengan batas waktu (idle 30 menit, maksimal 8 jam). Logout mengembalikan ke halaman login dan menutup dashboard. Dashboard tidak dapat dibuka tanpa login.

Keamanan yang aktif di prototype ini:

- Password disimpan sebagai **hash PBKDF2-SHA256** (120.000 iterasi + salt acak), bukan plaintext. Data lama dimigrasikan otomatis.
- **Rate limit**: maksimal 5 percobaan login per 15 menit, lalu terkunci 15 menit.
- **MFA/TOTP opsional** (authenticator 6 digit + 8 kode pemulihan) — aktifkan di **Admin → Keamanan**.
- Di production (HTTPS non-localhost) dashboard **mewajibkan ganti password** sebelum bisa dipakai.
- Log keamanan 90 hari, snapshot cadangan otomatis, dan tombol hapus data lokal.

Detail lengkap, termasuk hasil uji dan batasannya: [SECURITY.md](SECURITY.md).

## Fitur yang sudah berfungsi

**Publik**
- Homepage, brand showcase, katalog, global search, filter brand/kategori/harga/skala/base/rarity, sorting.
- Product detail page: 3D viewer, tab 3D View / Product Photos / Specifications, info lengkap, rating & ulasan, rekomendasi produk serupa.
- Wishlist (drawer + tombol hati), keranjang (ubah qty, total), checkout demo, Rarity Checker yang membaca data admin.
- Seller demo dashboard: tambah/edit/hapus produk, ringkasan pesanan & penjualan demo.

**3D Viewer**
- WebGL renderer, OrbitControls (drag rotate, scroll/pinch zoom, damping halus), auto rotate, reset view, zoom +/−, fullscreen, responsive canvas.
- `GLTFLoader` untuk model GLB/GLTF asli; jika kosong/gagal dimuat → model prosedural Three.js (bodi, kabin kaca, roda + velg, lampu depan/belakang, bumper, spion, spoiler, knalpot, fender) sesuai jenis kendaraan dan warna produk.
- Fallback eksplisit saat model gagal dimuat atau WebGL tidak tersedia; resource Three.js dibersihkan saat viewer ditutup (geometri, material, tekstur, renderer, animation loop).
- Dukungan **embed Sketchfab** untuk model yang tidak tersedia sebagai file GLB (dipakai produk Lamborghini Revuelto SV). Iframe dimuat hanya pada halaman produk tersebut, dengan kredit pembuat dan tombol fullscreen/reset/auto-rotate yang tetap berfungsi.

**Admin Dashboard**
- Dashboard overview: 8 kartu statistik dari data nyata, grafik penjualan & pertumbuhan pengguna (canvas), produk terpopuler/terbaru, aktivitas, seller pending.
- CRUD: produk, brand, kategori, diecast database, model 3D (dengan preview), rarity, user, seller, toko, pesanan, pembayaran, komisi, ulasan, banner, laporan.
- Verifikasi seller/toko, moderasi ulasan, verifikasi pembayaran & komisi, pengaturan situs, profil + ubah password, ekspor JSON, reset data demo.
- Pencarian, filter, pagination, modal form, konfirmasi hapus, validasi, notifikasi sukses/error, loading & empty state, sidebar drawer di mobile.

## Menambahkan foto produk

Di admin → **Product Management** → Tambah/Edit Produk:
- Tempel URL foto pada **URL foto produk**, atau
- Gunakan **Unggah foto produk** (maks 800 KB; disimpan sebagai data URL di LocalStorage).

Galeri tambahan diisi satu URL per baris.

## Menambahkan model 3D GLB/GLTF

1. Siapkan file `.glb` / `.gltf` (+ tekstur) dan host agar bisa diakses via HTTPS, misalnya di object storage/CDN. Server file harus mengizinkan **CORS** karena model dimuat oleh browser.
2. Admin → **3D Model Management** → **Tambah Model 3D** (atau Edit model produk terkait).
3. Isi **Nama model**, hubungkan ke **produk**, tempel **URL file GLB/GLTF**, pilih status `GLB tersedia`, atur warna/tipe, lalu simpan.
4. Alternatif: isi **URL file GLB/GLTF** langsung pada form produk (Admin → Product Management). Field ini menulis ke model 3D yang terhubung.
5. Buka halaman detail produk — viewer otomatis memakai model tersebut. Jika gagal dimuat, fallback prosedural tampil beserta peringatan.

Catatan: model GLB di-normalisasi otomatis (skala ±4.2 unit, diletakkan di atas bidang lantai). Model prosedural diberi label **MODEL DEMONSTRASI** dan bukan replika presisi kendaraan asli.

## Model 3D dari Sketchfab (Lamborghini Revuelto SV)

Produk **Lamborghini Revuelto SV** (id 7) memakai model Sketchfab [2027 Lamborghini Revuelto SV](https://sketchfab.com/3d-models/2027-lamborghini-revuelto-sv-ab7a50fea3504658990b0c00012189c5) karya **VTX** lewat embed resmi, karena halaman model tersebut tidak menyediakan unduhan GLB.

- Viewer tetap menyediakan tombol reset, auto-rotate, dan fullscreen. Zoom dilakukan langsung di dalam model (drag/scroll), sehingga tombol zoom +/− dinonaktifkan pada mode embed.
- Auto-rotate memuat ulang embed dengan parameter `autospin`, jadi kamera kembali ke posisi awal saat tombol ditekan.
- Kredit pembuat dan tautan ke Sketchfab selalu ditampilkan di viewer (kewajiban atribusi).
- Butuh koneksi internet; jika embed diblokir jaringan, tab **Product Photos** dan **Specifications** tetap berfungsi.
- Model ini cukup berat (337 ribu triangle, 201 ribu vertex). Pemuatan pertama bisa memakan puluhan detik tergantung koneksi/GPU; setelah masuk cache browser, muat ulang embed lebih cepat.
- Kamera awal mengikuti tampilan yang disimpan pembuat model (close-up depan). Scroll/pinch di dalam model untuk melihat keseluruhan mobil; tombol ⟲ mengembalikan tampilan awal dari Sketchfab.

Menambahkan model Sketchfab lain: Admin → **3D Model Management** → isi **Sketchfab Model ID / URL** dengan ID 32 karakter atau tautan model. Status otomatis menjadi `Sketchfab embed`. Jika kelak Anda punya file `.glb` berlisensi, kosongkan field Sketchfab dan isi URL GLB agar viewer memakai WebGL penuh.

## Keamanan

Ringkasan kontrol, hasil uji, dan batasan ada di [SECURITY.md](SECURITY.md). Yang paling cepat dicek:

| Kontrol | Status |
|---|---|
| Password di-hash PBKDF2-SHA256 + salt | ✅ |
| Rate limit login (5×/15 menit) | ✅ |
| MFA TOTP + kode pemulihan | ✅ (aktifkan di Admin → Keamanan) |
| Sesi timeout + wajib ganti password di production | ✅ |
| Escaping output + validasi URL/angka (anti-XSS) | ✅ |
| CSP, HSTS, `nosniff`, `X-Frame-Options`, `Referrer-Policy` | ✅ (header `vercel.json` + meta) |
| Log keamanan & retensi 90 hari | ✅ |
| Snapshot cadangan (3 versi) + ekspor JSON | ✅ |
| Kebijakan privasi + hapus data lokal | ✅ |
| Verifikasi sesi di server, WAF, audit log terpusat | ❌ butuh backend |

## Penyimpanan data

- `localStorage` → `dh_data_v2` (skema versi 3; produk, brand, kategori, model 3D, rarity, user, seller, toko, pesanan, pembayaran, komisi, ulasan, banner, laporan, pengaturan, aktivitas). Data lama versi 2 dimigrasikan otomatis tanpa menghapus produk lain.
- `localStorage` → `dh_cart`, `dh_wishlist`.
- `sessionStorage` → `dh_admin_session`.
- Tidak ada IndexedDB pada versi ini; file 3D disimpan sebagai URL, bukan blob. Reset data: Admin → Settings → **Reset data demo**.

Data awal produk, brand, kategori, model 3D, rarity, transaksi, dan pengguna tersedia sebagai data demonstrasi.

## Batasan (memerlukan backend)

- Autentikasi admin berbasis kredensial statis di LocalStorage — bukan keamanan produksi.
- Checkout, pembayaran, pengiriman, dan status transaksi hanya simulasi; tidak ada payment gateway.
- Data hanya tersimpan di browser/perangkat pengguna, tidak tersinkron antar perangkat.
- Upload gambar dibatasi LocalStorage (maks 800 KB per gambar); untuk produksi gunakan object storage.
- Model GLB memerlukan hosting dengan CORS; tidak ada proxy upload model.
- Statistik dashboard dihitung dari data lokal, bukan dari database server.

# Keamanan Diecast Hub

Dokumen ini memetakan **Daftar Periksa Keamanan Minimum UMKM** ke implementasi nyata di repositori ini, plus batasannya.

> Aplikasi ini **statis** (satu `index.html`, tanpa backend). Artinya kontrol autentikasi melindungi data yang ada di perangkat pengguna, bukan database server. Untuk produksi komersial, autentikasi, pembayaran, dan data pelanggan wajib dipindah ke backend (butuh sesi server-side, hash di server, dan audit log terpusat).

## 1. Autentikasi & rate limiting — Tinggi

| Kontrol | Implementasi | Lokasi |
|---|---|---|
| Password tidak disimpan plaintext | PBKDF2-SHA256, 120.000 iterasi, salt acak 128-bit per password. Seed hanya menyimpan hash. Data lama ber-plaintext dimigrasikan otomatis saat halaman dimuat. | `setAdminPassword()`, `verifyAdminPassword()`, `ensurePasswordHash()` |
| Batas percobaan login | Maksimal **5 percobaan / 15 menit** → akun terkunci 15 menit (dihitung dari `localStorage`). | `registerLoginFailure()`, `loginLockRemaining()` |
| Perbandingan konstan | Perbandingan password/hash tanpa early-exit. | `timingSafeEqual()` |
| MFA admin | **TOTP RFC 6238** (HMAC-SHA1, 6 digit, 30 detik, toleransi ±1 langkah) + **8 kode pemulihan** sekali pakai (disimpan sebagai SHA-256). | `totpCode()`, `verifyTotp()`, `consumeRecoveryCode()` |
| Sesi berbatas waktu | Idle timeout (default 30 menit, dapat diubah) + batas absolut 8 jam + token sesi acak. | `createAdminSession()`, `isAdminAuthed()` |
| Wajib ganti password default | Di production (`https` non-localhost), dashboard terkunci oleh modal sampai password demo diganti. | `openForcedPasswordChange()` |
| Kebijakan password | Minimal 8 karakter, harus huruf + angka, menolak password umum. | `passwordIssues()` |

## 2. HTTPS & .gitignore — Tinggi

- Deploy Vercel → TLS otomatis. `vercel.json` menambahkan **HSTS** (`max-age=63072000; includeSubDomains`).
- Halaman memaksa redirect `http → https` di host non-lokal (`initSecurity()`); localhost dan preview `.style.dev` dikecualikan.
- `.gitignore` mengabaikan `.env`, `.env.*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `secrets.json`, `.vercel`, plus `artifacts/` dan `*.zip`.
- Tidak ada API key, token, atau kredensial pihak ketiga di dalam kode.

## 3. Validasi input & OWASP Top 10 — Tinggi

- **A01 Broken Access Control** — dashboard hanya render setelah `isAdminAuthed()`; setiap render halaman admin memeriksa ulang sesi.
- **A02 Cryptographic Failures** — PBKDF2-SHA256 untuk password, SHA-256 untuk kode pemulihan, CSPRNG (`crypto.getRandomValues`) untuk salt/secret/token.
- **A03 Injection / XSS** — semua nilai dirender lewat `esc()`; argumen handler memakai `jsArg()` (JSON + HTML escape) sehingga nama brand/produk tidak bisa keluar dari string JS; URL dibatasi `safeUrl()` (`http`, `https`, `data:image/*` saja — `javascript:` dan `data:text/html` ditolak); input dibatasi panjangnya lewat `sanText()` / `sanMultiline()`; angka dipaksa non-negatif.
- **A05 Security Misconfiguration** — CSP ketat (meta + header), `object-src 'none'`, `base-uri 'none'`, `frame-ancestors 'self'`, `nosniff`, `Referrer-Policy`, `Permissions-Policy`, `X-Frame-Options: SAMEORIGIN`.
- **A07 Identification & Auth Failures** — rate limit, MFA, sesi timeout, rotasi password.
- **A09 Logging & Monitoring Failures** — log keamanan 90 hari (lihat bagian 5).

Catatan CSP: `script-src` masih memerlukan `'unsafe-inline'` karena seluruh CSS/JS aplikasi berada di satu file. Ini melemahkan proteksi XSS dibanding CSP strict, sehingga escaping output tetap menjadi pertahanan utama.

## 4. Kontrol akses & isolasi dev/prod — Sedang

- Login demo hanya **ditampilkan** di lingkungan pengembangan (localhost / preview). Di production tampil peringatan untuk mengganti kredensial.
- Badge `PRODUCTION` / `DEVELOPMENT` pada halaman **Admin → Keamanan** menunjukkan mode aktif.
- Semua data aplikasi berada di origin pengguna masing-masing (`localStorage`), tidak ada data bersama antar pengguna.

## 5. Dependensi, log & backup — Sedang

- Dependensi runtime: hanya **Three.js** dan add-on resminya, dimuat dari `cdn.jsdelivr.net` dengan **versi dipin** (`three@0.160.0`) — bukan `latest`, sehingga tidak berubah diam-diam.
- Tidak ada `package.json`, `node_modules`, atau build step → permukaan supply chain minimal.

**Hasil audit dependensi (24 November 2026):**

| Sumber | Hasil untuk `three@0.160.0` |
|---|---|
| Snyk Vulnerability DB | Tidak ada direct vulnerability pada paket ini |
| deps.dev (OSV/GitHub Advisory) | Tidak ada advisory terdeteksi; paket tanpa dependensi |
| GitHub Advisory | Satu-satunya CVE terkait (CVE-2022-0177) hanya memengaruhi `< 0.137.0` dan statusnya **withdrawn** |

Versi terbaru saat audit adalah `three@0.187.0` (30 Oktober 2026). Karena tidak ada CVE yang berlaku untuk r160 dan upgrade 27 rilis berisiko memecahkan viewer prosedural, versi dipertahankan **dipin** — bukan dibiarkan `latest`. Jadwalkan review ulang setiap 90 hari atau saat ada advisory baru:

- <https://security.snyk.io/package/npm/three>
- <https://deps.dev/npm/three>
- <https://github.com/mrdoob/three.js/releases>
- **Log**: `DB.activity` (aktivitas admin) dan `dh_security_log` (login sukses/gagal, lockout, MFA, perubahan password, snapshot) dipangkas otomatis ke **90 hari**, maksimal 300 entri keamanan.
- **Backup**: snapshot otomatis tiap 30 menit (maks 3 versi) + tombol *Buat snapshot*, *Pulihkan*, *Unduh*, dan ekspor JSON penuh. Retensi log diatur `SEC.retentionDays`.

## 6. Kebijakan privasi — Rendah

Halaman **Kebijakan Privasi** (tautan di footer) menjelaskan data yang disimpan, pihak ketiga (jsdelivr, Sketchfab, host foto produk), retensi log 90 hari, hak ekspor/hapus data, dan kontak. Tombol **Hapus semua data lokal** ada di Admin → Keamanan.

## Cara menguji

1. **Rate limit** — login salah 5×; percobaan ke-5 mengunci 15 menit (pesan + entri `login_blocked` di log).
2. **MFA** — Admin → Keamanan → *Aktifkan MFA*; tambahkan secret di authenticator, masukkan kode 6 digit, lalu login ulang (password + kode).
3. **XSS** — Admin → Product Management → tambah produk dengan nama `<img src=x onerror=alert(1)>`; nama tampil sebagai teks, tidak ada dialog.
4. **CSP** — DevTools → Console: tidak ada pelanggaran CSP saat memuat katalog, viewer Three.js, dan embed Sketchfab.
5. **Header** — setelah deploy: `curl -I https://<domain>` harus menampilkan `content-security-policy`, `strict-transport-security`, `x-content-type-options`, `x-frame-options`.

## Batasan yang perlu backend

- Kredensial admin dapat dibaca dari kode sumber (hash PBKDF2, bukan plaintext) — penyerang bisa mencoba brute force offline. Untuk produksi, verifikasi harus di server dengan rate limit per akun/IP.
- Rate limit berbasis `localStorage` dapat dihapus pengguna; proteksi sesungguhnya butuh server.
- Siapa pun yang punya akses fisik ke browser dapat melihat data di `localStorage`.
- Belum ada WAF, monitoring server, atau rotasi kunci — semua itu di luar aplikasi statis.

## Melaporkan masalah

Kirim detail reproduksi ke `hello@diecasthub.id`. Jangan buka issue publik untuk temuan yang dapat dieksploitasi.

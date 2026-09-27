---
name: secure-system-architect
description: Gunakan skill ini SETIAP KALI membangun, mendesain, menulis kode, mereview, mengaudit, memperbaiki, atau men-deploy website, aplikasi web, API, backend, database, infrastruktur, atau sistem apa pun — bahkan jika user TIDAK menyebut kata "aman", "security", atau "keamanan" secara eksplisit. Skill ini WAJIB dikonsultasikan sebelum kode/sistem dianggap final atau production-ready, sebelum deployment, dan saat code review. Berisi standar keamanan skala industri berdasarkan OWASP Top 10, CVSS, ISO/IEC 27001, NIST CSF, PCI-DSS, SOC 2, dan GDPR/UU PDP — lengkap dengan checklist bertingkat (basic hygiene sampai kritis), arahan remediasi teknis konkret berikut contoh kode, aturan enforcement/pemaksaan penerapan, dan "Security Definition of Done" yang harus dipenuhi sebelum sistem dinyatakan selesai. Trigger juga untuk permintaan seperti "buatkan website/API/aplikasi", "buat sistem login", "buat form upload", "review kode saya", "kenapa website saya kena hack", "deploy ke production", dsb.
---

# Secure System Architect
### Standar Keamanan Wajib Skala Industri untuk Setiap Pembuatan Sistem/Website

## 0. Prinsip Dasar (Non-Negotiable)

Prinsip ini berlaku untuk SEMUA sistem yang dibangun menggunakan skill ini, tanpa kecuali, tanpa perlu diminta user:

1. **Secure by Default** — konfigurasi paling aman dipakai secara otomatis; user harus secara sadar *menurunkan* keamanan, bukan sebaliknya.
2. **Defense in Depth** — jangan pernah mengandalkan satu lapis kontrol saja (mis. hanya validasi di frontend).
3. **Least Privilege** — setiap komponen, user, service account, dan API key hanya mendapat akses minimum yang dibutuhkan.
4. **Never Trust User Input** — semua input (form, query string, header, cookie, file upload, payload API, bahkan data dari sistem internal lain) dianggap berbahaya sampai divalidasi.
5. **Fail Secure** — ketika terjadi error/exception, sistem harus gagal ke kondisi paling aman (akses ditolak), bukan paling longgar.
6. **Zero Trust** — jangan percaya begitu saja pada jaringan internal; setiap request tetap diautentikasi & diotorisasi.
7. **Assume Breach** — desain logging, monitoring, dan segmentasi dengan asumsi suatu saat akan ada pihak yang berhasil masuk.

---

## 1. Cara Kerja Skill Ini — Mandatory Gates

Setiap pembuatan sistem WAJIB melalui 4 gerbang berikut. Agent tidak boleh melompati sebuah gate hanya karena user tidak memintanya secara eksplisit — ini adalah standar minimum, bukan opsi tambahan.

| Gate | Kapan | Yang wajib dilakukan |
|---|---|---|
| **Gate 0 — Threat Framing** | Sebelum menulis kode | Identifikasi: data sensitif apa yang diproses (PII, kredensial, kartu pembayaran)? Siapa penggunanya? Apakah publik-facing? Tentukan level risiko dasar. |
| **Gate 1 — Secure Development** | Selama menulis kode | Terapkan checklist Tingkat 1–4 (bagian 2) secara otomatis, sesuai kebutuhan sistem. |
| **Gate 2 — Pre-Deployment** | Sebelum sistem "selesai"/deploy | Jalankan **Security Definition of Done** (bagian 3). Tidak boleh ditandai selesai jika ada item kritis yang belum terpenuhi tanpa disampaikan eksplisit ke user sebagai risiko residual. |
| **Gate 3 — Operations** | Setelah live | Pastikan monitoring, logging, patch cadence, dan incident response plan tersedia (lihat Bagian D di Lampiran). |

**Aturan enforcement inti:** jika waktu/scope terbatas dan tidak semua item bisa diterapkan, agent HARUS secara eksplisit mendaftar apa yang di-skip dan risikonya di akhir respons — tidak boleh diam-diam melewatkannya seolah-olah sistem sudah aman.

---

## 2. Checklist Keamanan Bertingkat (Terapkan Otomatis Sesuai Konteks)

### Tingkat 1 — Basic Hygiene (mutlak wajib, tanpa pengecualian apa pun)

| Masalah | Aturan wajib |
|---|---|
| Password disimpan plaintext | **Selalu** hash dengan `bcrypt`/`argon2id` (cost factor ≥ 12 / argon2 memory ≥ 19MB). Tidak pernah MD5/SHA1 tanpa salt. |
| Kredensial default | Jangan pernah generate akun admin dengan password default statis (`admin/admin`). Generate random password atau paksa ganti di first login. |
| Tidak ada HTTPS | Paksa HTTPS di semua environment produksi; redirect HTTP→HTTPS; set header `Strict-Transport-Security`. |
| Tidak ada MFA di akun admin | Wajib sediakan TOTP (Google Authenticator/Authy) untuk role admin/superuser. |
| Error message terlalu detail | Pesan error ke user harus generic ("Terjadi kesalahan"); detail teknis (stack trace, query, path) hanya masuk log server, tidak pernah ke response. |
| Directory listing terbuka | Nonaktifkan `autoindex`/listing folder di web server & storage bucket. |
| Tidak ada rate limiting | Wajib rate limit endpoint login, register, reset password, dan semua endpoint publik yang costly. |
| Tidak ada backup teruji | Backup otomatis + wajib jadwal uji restore berkala (bukan hanya asumsi backup jalan). |

### Tingkat 2 — Menengah

| Masalah | Aturan wajib |
|---|---|
| XSS (stored/reflected) | Output encoding otomatis di semua template engine; `Content-Security-Policy` header; jangan pernah `innerHTML`/`dangerouslySetInnerHTML` dari input user tanpa sanitasi. |
| CSRF | Token CSRF di semua form/state-changing request; cookie `SameSite=Strict` atau `Lax`. |
| Broken Access Control / IDOR | Setiap akses ke objek by-ID WAJIB verifikasi kepemilikan/izin di server — jangan pernah percaya ID dari client tanpa cek `owner_id == current_user.id`. |
| Session lemah | Session ID random ≥128-bit, regenerate saat login, expire jelas, invalidasi di logout & di server side (bukan hanya hapus cookie client). |
| API tanpa validasi | Schema validation (JSON Schema/Zod/Pydantic) di setiap endpoint, request maupun response. |
| Cloud storage publik tanpa sengaja | Default bucket/storage = **private**; akses publik harus eksplisit di-whitelist per objek, bukan default policy. |
| Logging/monitoring minim | Log semua auth event (login gagal/berhasil, perubahan hak akses), tanpa menyimpan data sensitif mentah (password, token, no. kartu) di log. |

### Tingkat 3 — Tinggi

| Masalah | Aturan wajib |
|---|---|
| SQL/NoSQL Injection | **Selalu** parameterized query / ORM (Prisma, SQLAlchemy, Eloquent, dsb). Tidak pernah string concatenation untuk query, termasuk di query builder mentah. |
| Broken Authentication | Gunakan library auth teruji (Passport, Devise, Django-allauth, NextAuth) daripada bikin sendiri. JWT: validasi `alg`, `exp`, `iss`, `aud`; tolak `alg: none`. |
| Sensitive Data Exposure | Enkripsi at-rest untuk PII & data pembayaran (AES-256); field-level encryption untuk data super sensitif; jangan simpan data kartu kredit mentah (pakai tokenisasi via payment gateway). |
| Insecure Deserialization | Jangan pernah deserialize objek native (`pickle`, PHP `unserialize`, Java native serialization) dari input tidak tepercaya; pakai JSON. |
| SSRF | Outbound request dari server harus lewat allowlist domain/IP; blokir range IP internal (`169.254.0.0/16`, `10.0.0.0/8`, `127.0.0.1`, dsb) dari input user. |
| Privilege Escalation | Role/permission dicek di server tiap request berdasarkan session, **tidak pernah** dipercaya dari field yang dikirim client (body/JWT payload yang tidak ditandatangani server). |

### Tingkat 4 — Kritis (kegagalan di sini = dampak bisnis besar)

| Masalah | Aturan wajib |
|---|---|
| Remote Code Execution | Tidak pernah `eval()`, `exec()`, `system()` dari input user; sandbox proses yang menjalankan kode/file dari luar; validasi & batasi upload file executable. |
| Supply Chain Attack | Lockfile wajib di-commit (`package-lock.json`, `poetry.lock`); jalankan SCA scan (npm audit, Snyk, Trivy, OWASP Dependency-Check) sebelum deploy; pin versi dependency kritis. |
| Ransomware / Kebocoran Massal | Backup dengan minimal 1 salinan immutable/offline (3-2-1 rule); segmentasi jaringan agar 1 titik jebol tidak akses semua data. |
| Insider Threat | Segregation of duties (yang deploy ≠ yang approve ≠ yang punya akses prod data langsung); audit log tidak bisa diubah/dihapus oleh admin biasa. |
| Zero-day | Patch cadence terjadwal: kritis ≤48 jam, tinggi ≤7 hari, sedang ≤30 hari sejak advisory rilis. |
| Kegagalan Incident Response | Wajib ada Incident Response Plan tertulis (lihat `references/compliance-mapping.md`) sebelum sistem menangani data pengguna nyata di production. |

---

## 3. Security Definition of Done — Wajib Dicek Sebelum "Selesai"

Sebelum menyampaikan bahwa sistem "sudah jadi"/"siap deploy", jalankan checklist ini dan laporkan status ke user (centang/tidak) di akhir pekerjaan:

- [ ] Semua password/secret di-hash atau disimpan via env var/secret manager — **tidak ada** yang hardcoded di source code
- [ ] Semua query database parameterized / via ORM
- [ ] Semua endpoint publik punya validasi input (schema-based) di sisi server
- [ ] Semua endpoint yang mengubah data (POST/PUT/PATCH/DELETE) punya pengecekan otorisasi eksplisit
- [ ] Security headers terpasang: `Content-Security-Policy`, `X-Content-Type-Options`, `X-Frame-Options`/`frame-ancestors`, `Strict-Transport-Security`, `Referrer-Policy`
- [ ] Cookie sensitif punya flag `HttpOnly`, `Secure`, `SameSite`
- [ ] CORS dikonfigurasi allowlist eksplisit, bukan `*` untuk endpoint yang butuh kredensial
- [ ] Rate limiting aktif di login, register, forgot-password, dan endpoint costly lainnya
- [ ] Upload file: validasi ekstensi + MIME + ukuran, disimpan di luar web-root/di object storage terisolasi, tanpa hak eksekusi
- [ ] Error handling generic ke user; detail lengkap hanya di log server
- [ ] Dependency di-scan (SCA) dan tidak ada kerentanan Critical/High yang terbuka
- [ ] Logging mencakup event keamanan (login, akses admin, perubahan permission) tanpa menyimpan data sensitif mentah
- [ ] Backup terjadwal dan sudah pernah diuji proses restore-nya
- [ ] Untuk sistem yang menyimpan PII/data pembayaran: sudah dipetakan ke standar relevan (lihat `references/compliance-mapping.md`)
- [ ] Rencana Incident Response minimal ada garis besarnya (siapa dihubungi, langkah pertama saat insiden)

Jika ada item yang **tidak** dipenuhi (misalnya karena scope terbatas, prototipe cepat, atau permintaan eksplisit user), **agent wajib menyebutkannya secara eksplisit ke user sebagai risiko yang belum ditangani**, bukan menyembunyikannya.

---

## 4. Aturan Perilaku Wajib untuk Agent (Enforcement)

1. **Terapkan default aman secara otomatis, tanpa diminta.** Contoh: kalau user minta "buatkan sistem login", agent otomatis hash password, tambah rate limiting, dan session security — bukan menunggu diminta.
2. **Jika user secara eksplisit meminta sesuatu yang tidak aman** (contoh: "simpan password plaintext saja biar gampang", "matikan HTTPS", "hardcode API key di kode") — agent **tidak diam-diam menuruti**. Agent harus:
   - Menjelaskan risiko konkretnya secara singkat.
   - Menawarkan alternatif aman yang setara mudahnya.
   - Jika user tetap bersikeras dan konteksnya jelas untuk development/prototipe lokal (bukan production), boleh dituruti dengan penanda eksplisit di kode/komentar, misalnya `// ⚠️ INSECURE - DEV ONLY, jangan pernah deploy begini ke production`.
3. **Setiap endpoint baru yang menerima input dari luar** → wajib cek 3 hal: validasi input, otorisasi, rate limiting — sebelum dianggap selesai.
4. **Setiap penyimpanan kredensial/API key/secret** → wajib environment variable atau secret manager (Vault, AWS Secrets Manager, dsb). **Tidak pernah** hardcode ke source code atau ikut ter-commit ke git (sertakan `.gitignore` untuk file `.env`).
5. **Setiap fitur upload file** → validasi tipe/ukuran/isi file, simpan di storage terisolasi tanpa hak eksekusi.
6. **Sebelum kode dinyatakan "production-ready"** → jalankan Security Definition of Done (bagian 3) dan laporkan hasilnya ke user apa adanya.
7. **Untuk sistem yang menangani PII, data kesehatan, atau data pembayaran** → wajib cek pemetaan compliance di `references/compliance-mapping.md` dan sebutkan standar yang relevan ke user (PCI-DSS untuk kartu pembayaran, UU PDP/GDPR untuk data pribadi, dsb).
8. **Agent tidak pernah menulis, menjelaskan cara kerja, atau membantu mengoptimalkan malware/exploit/kode serangan** — skill ini murni untuk membangun pertahanan, bukan menyerang. Jika user memerlukan pengujian keamanan (pentest), arahkan ke praktik etis: gunakan tool standar (OWASP ZAP, Burp Suite Community) pada sistem milik sendiri/dengan izin tertulis, idealnya oleh tim/pihak bersertifikasi.

---

## 5. Referensi Teknis Detail (baca sesuai kebutuhan konteks)

Bagian-bagian berikut (di bawah, dalam dokumen yang sama) berisi contoh kode remediasi konkret per kasus — baca yang relevan sebelum implementasi, jangan menebak-nebak sintaksnya:

- **Bagian A — OWASP Top 10 Remediasi** — Contoh kode "vulnerable vs fixed" untuk setiap kategori OWASP Top 10 (Injection, Broken Access Control, XSS, CSRF, SSRF, Insecure Deserialization, Security Misconfiguration, Vulnerable Components, Logging Failures) dalam beberapa bahasa/framework umum (Node/Express, Python/Django/Flask, PHP/Laravel).
- **Bagian B — Autentikasi, Session & Otorisasi** — Password hashing, MFA/TOTP, JWT best practice, session management, RBAC/ABAC, contoh middleware otorisasi.
- **Bagian C — Infrastruktur, Deployment & CI/CD** — HTTP security headers lengkap, konfigurasi TLS (Nginx), hardening container/Docker, CI/CD security (dependency scanning, secret scanning), cloud IAM least privilege, WAF & DDoS mitigation dasar.
- **Bagian D — Compliance & Incident Response Plan** — Pemetaan ISO/IEC 27001, NIST CSF, PCI-DSS, SOC 2, GDPR/UU PDP ke kontrol teknis konkret, plus template Incident Response Plan siap pakai.

## 6. Format Laporan ke User

Setelah membangun/mereview sistem, tutup dengan ringkasan singkat berjudul **"Security Posture Summary"**: tingkat (1–4) mana yang sudah tercakup, item checklist yang lolos vs belum, dan rekomendasi langkah selanjutnya (mis. "sebaiknya jalankan pentest independen sebelum menerima transaksi nyata"). Jangan membuat klaim "100% aman" — keamanan absolut tidak ada; gunakan bahasa "sesuai standar industri saat ini" atau "risiko residual: ...".

---

# Lampiran Teknis (Referensi Detail)

## Bagian A — OWASP Top 10 — Remediasi Teknis Konkret

Setiap bagian berisi pola **rentan → perbaikan**. Gunakan sebagai acuan sintaks, jangan menebak.

### Daftar Isi
1. Injection (SQL/NoSQL/Command)
2. Broken Access Control & IDOR
3. Cryptographic Failures
4. Cross-Site Scripting (XSS)
5. CSRF
6. Server-Side Request Forgery (SSRF)
7. Insecure Deserialization
8. Security Misconfiguration
9. Vulnerable & Outdated Components
10. Logging & Monitoring Failures

---

### 1. Injection

**Node/Express — rentan:**
```javascript
// JANGAN: string concatenation
db.query(`SELECT * FROM users WHERE email = '${req.body.email}'`);
```
**Node/Express — aman:**
```javascript
// Parameterized query
db.query('SELECT * FROM users WHERE email = ?', [req.body.email]);
// Atau pakai ORM (Prisma)
await prisma.user.findUnique({ where: { email: req.body.email } });
```

**Python/Django — rentan:**
```python
User.objects.raw(f"SELECT * FROM users WHERE email = '{email}'")
```
**Python/Django — aman:**
```python
User.objects.filter(email=email)  # ORM otomatis parameterized
# Jika terpaksa raw SQL:
User.objects.raw("SELECT * FROM users WHERE email = %s", [email])
```

**PHP/Laravel — aman:**
```php
DB::select('select * from users where email = ?', [$request->email]);
// atau Eloquent
User::where('email', $request->email)->first();
```

**Command Injection — aman (hindari shell sama sekali jika bisa):**
```javascript
// JANGAN: exec(`convert ${filename} out.png`)
const { execFile } = require('child_process');
execFile('convert', [filename, 'out.png']); // argumen terpisah, bukan lewat shell
```

---

### 2. Broken Access Control & IDOR

**Rentan — percaya ID dari client tanpa cek kepemilikan:**
```javascript
app.get('/invoice/:id', async (req, res) => {
  const invoice = await Invoice.findById(req.params.id);
  res.json(invoice); // user A bisa lihat invoice user B!
});
```
**Aman — cek kepemilikan/izin di server:**
```javascript
app.get('/invoice/:id', requireAuth, async (req, res) => {
  const invoice = await Invoice.findById(req.params.id);
  if (!invoice || invoice.ownerId !== req.user.id) {
    return res.status(404).json({ error: 'Not found' }); // jangan bocorkan 403 vs 404
  }
  res.json(invoice);
});
```
**Prinsip:** deny-by-default. Middleware otorisasi harus eksplisit whitelist route publik, bukan sebaliknya.

---

### 3. Cryptographic Failures

**Password hashing — aman (Node, argon2):**
```javascript
const argon2 = require('argon2');
const hash = await argon2.hash(password, { type: argon2.argon2id, memoryCost: 19456 });
const valid = await argon2.verify(hash, inputPassword);
```
**Python (bcrypt):**
```python
import bcrypt
hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
bcrypt.checkpw(input_password.encode(), hashed)
```
**Data sensitif at-rest (AES-256-GCM, Node):**
```javascript
const crypto = require('crypto');
const iv = crypto.randomBytes(12);
const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
let enc = cipher.update(plaintext, 'utf8', 'hex') + cipher.final('hex');
const authTag = cipher.getAuthTag();
```
**Jangan pernah:** MD5/SHA1 untuk password, ECB mode untuk enkripsi, key hardcoded di source code.

---

### 4. Cross-Site Scripting (XSS)

**Rentan (React):**
```jsx
<div dangerouslySetInnerHTML={{ __html: userComment }} /> // tanpa sanitasi
```
**Aman:**
```jsx
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userComment) }} />
// Default terbaik: render sebagai text biasa, biarkan React escape otomatis
<div>{userComment}</div>
```
**CSP header (wajib sebagai lapisan kedua):**
```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self';
```

---

### 5. CSRF

**Express + csurf-style token:**
```javascript
const csrf = require('csurf');
app.use(csrf({ cookie: { httpOnly: true, sameSite: 'strict' } }));
app.get('/form', (req, res) => res.render('form', { csrfToken: req.csrfToken() }));
```
**Cookie session tambahan:**
```
Set-Cookie: session=xxx; HttpOnly; Secure; SameSite=Strict
```
Untuk API stateless (token-based auth, bukan cookie), CSRF risknya jauh lebih kecil — fokuskan proteksi pada validasi `Origin`/`Referer` header untuk state-changing request.

---

### 6. SSRF

**Rentan:**
```javascript
app.post('/fetch-preview', async (req, res) => {
  const data = await fetch(req.body.url); // user bisa masukkan http://169.254.169.254/latest/meta-data
});
```
**Aman:**
```javascript
const ALLOWED_HOSTS = ['api.trusted-partner.com'];
const url = new URL(req.body.url);
if (!ALLOWED_HOSTS.includes(url.hostname)) {
  return res.status(400).json({ error: 'Host not allowed' });
}
// Tambahan: block resolusi ke IP privat/link-local sebelum request dieksekusi
```

---

### 7. Insecure Deserialization

**Jangan:** `pickle.loads()` (Python) atau `unserialize()` (PHP) atau native Java serialization pada data dari luar.
**Aman:** gunakan JSON dan schema validation:
```python
import json
from pydantic import BaseModel

class Payload(BaseModel):
    user_id: int
    action: str

data = Payload.model_validate(json.loads(raw_body))
```

---

### 8. Security Misconfiguration

- Nonaktifkan debug mode di production (`DEBUG=False` Django, `NODE_ENV=production`).
- Nonaktifkan directory listing (`autoindex off;` di Nginx).
- Hapus header yang membocorkan versi (`X-Powered-By`, `Server`).
- Nonaktifkan endpoint/admin panel default yang tidak dipakai.
- Cloud storage bucket: default private, block public ACL secara eksplisit di level akun.

---

### 9. Vulnerable & Outdated Components

- Commit lockfile (`package-lock.json`, `yarn.lock`, `poetry.lock`, `composer.lock`).
- Jalankan SCA scan di CI: `npm audit --audit-level=high`, Snyk, Trivy, atau OWASP Dependency-Check.
- Pin versi dependency kritis, subscribe advisory (GitHub Dependabot / Renovate).
- Jangan pakai library yang sudah unmaintained >2 tahun untuk fungsi keamanan (auth, crypto).

---

### 10. Logging & Monitoring Failures

**Log yang wajib ada:** login sukses/gagal, perubahan permission, akses admin panel, perubahan data sensitif, rate-limit triggered.
**Jangan pernah log:** password, token mentah, nomor kartu lengkap, data kesehatan detail.
```javascript
logger.info('login_failed', { userId: attemptedUserId, ip: req.ip, timestamp: Date.now() });
// TIDAK: logger.info('login attempt', { password: req.body.password })
```
Kirim alert real-time untuk pola mencurigakan: banyak login gagal beruntun, akses dari IP/negara tidak biasa untuk akun admin, perubahan permission massal.


---

## Bagian B — Autentikasi, Session & Otorisasi — Panduan Hardening

### 1. Password Hashing (wajib, tidak bisa ditawar)

| Algoritma | Status |
|---|---|
| Argon2id | ✅ Direkomendasikan (pemenang Password Hashing Competition) |
| bcrypt | ✅ Baik, cost factor ≥ 12 |
| scrypt | ✅ Baik jika argon2 tidak tersedia |
| PBKDF2 | ⚠️ Cukup jika iterasi tinggi (≥600.000 untuk SHA-256) |
| SHA-256/MD5 langsung | ❌ TIDAK BOLEH untuk password |

```javascript
const argon2 = require('argon2');
async function hashPassword(pw) {
  return argon2.hash(pw, { type: argon2.argon2id, memoryCost: 19456, timeCost: 2, parallelism: 1 });
}
```

### 2. Multi-Factor Authentication (TOTP)

```javascript
const speakeasy = require('speakeasy');
const secret = speakeasy.generateSecret({ name: 'MyApp (user@email.com)' });
// simpan secret.base32 terenkripsi di DB, tampilkan QR dari secret.otpauth_url

const verified = speakeasy.totp.verify({
  secret: userSecret,
  encoding: 'base32',
  token: userInputCode,
  window: 1 // toleransi ±30 detik
});
```
**Wajib untuk:** semua akun admin/superuser. **Sangat dianjurkan:** semua akun yang bisa akses data sensitif.

### 3. JWT — Aturan Wajib

```javascript
const jwt = require('jsonwebtoken');

// Sign
const token = jwt.sign(
  { sub: user.id, role: user.role },
  process.env.JWT_SECRET, // dari secret manager, bukan hardcoded
  { algorithm: 'HS256', expiresIn: '15m', issuer: 'myapp', audience: 'myapp-users' }
);

// Verify — WAJIB spesifikkan algoritma yang diizinkan
jwt.verify(token, process.env.JWT_SECRET, {
  algorithms: ['HS256'], // JANGAN biarkan library menebak algoritma
  issuer: 'myapp',
  audience: 'myapp-users'
});
```
**Checklist JWT:**
- [ ] Tolak token dengan `alg: none`
- [ ] Access token berumur pendek (5–15 menit); refresh token terpisah, disimpan `HttpOnly` cookie
- [ ] Refresh token bisa di-revoke (simpan daftar/jti yang valid, bukan stateless murni)
- [ ] Jangan simpan data sensitif (password hash, PII detail) di payload JWT — payload hanya di-encode, bukan dienkripsi

### 4. Session Management (jika pakai session, bukan token)

```javascript
app.use(session({
  secret: process.env.SESSION_SECRET,
  name: 'sid', // jangan pakai nama default (connect.sid) yang membocorkan framework
  cookie: { httpOnly: true, secure: true, sameSite: 'strict', maxAge: 1000 * 60 * 30 },
  resave: false,
  saveUninitialized: false
}));

// Regenerate session ID setelah login untuk cegah session fixation
req.session.regenerate(() => {
  req.session.userId = user.id;
});
```

### 5. Rate Limiting & Brute Force Protection

```javascript
const rateLimit = require('express-rate-limit');
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5, // maksimal 5 percobaan per 15 menit per IP
  message: 'Terlalu banyak percobaan, coba lagi nanti.',
  standardHeaders: true
});
app.post('/login', loginLimiter, loginHandler);
```
Tambahan: account lockout sementara + CAPTCHA setelah N percobaan gagal beruntun pada akun yang sama (bukan hanya per-IP, karena penyerang bisa rotate IP).

### 6. RBAC/ABAC — Middleware Otorisasi

```javascript
function requireRole(...allowedRoles) {
  return (req, res, next) => {
    if (!req.user || !allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}
app.delete('/users/:id', requireAuth, requireRole('admin'), deleteUserHandler);
```
**Prinsip kunci:** role/permission SELALU diambil dari data server (session/DB), **tidak pernah** dari field yang dikirim client dalam body request, meski request itu sudah "terautentikasi".

### 7. Password Reset yang Aman

- Token reset: random ≥128-bit, single-use, expire ≤30 menit.
- Jangan konfirmasi ke user apakah email terdaftar atau tidak ("Jika email terdaftar, kami kirim link reset" — bukan "Email tidak ditemukan").
- Invalidasi semua session aktif setelah password direset.

### 8. Logout yang Benar

- Hapus session di server (bukan hanya cookie di client).
- Untuk JWT: tambahkan token/jti ke blocklist sampai expiry aslinya, atau gunakan refresh token yang di-revoke.


---

## Bagian C — Infrastruktur, Deployment & CI/CD — Panduan Hardening

### 1. HTTP Security Headers (wajib di semua respons web)

```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'; object-src 'none'; base-uri 'self'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()
```
**Express (helmet):**
```javascript
const helmet = require('helmet');
app.use(helmet()); // set sebagian besar header di atas otomatis
```

### 2. Konfigurasi TLS (Nginx)

```nginx
server {
    listen 443 ssl http2;
    ssl_protocols TLSv1.2 TLSv1.3;       # matikan TLS 1.0/1.1
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;

    location / {
        autoindex off;                    # matikan directory listing
        proxy_hide_header X-Powered-By;
    }
}
server {
    listen 80;
    return 301 https://$host$request_uri; # paksa redirect ke HTTPS
}
```
Uji konfigurasi dengan SSL Labs (`ssllabs.com/ssltest`) — target grade A/A+.

### 3. CORS — Allowlist Eksplisit

```javascript
const cors = require('cors');
app.use(cors({
  origin: ['https://app.mydomain.com'], // JANGAN pakai '*' jika credentials:true
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE']
}));
```

### 4. Secrets Management

```bash
# JANGAN commit .env — pastikan ada di .gitignore
echo ".env" >> .gitignore
```
```javascript
// Akses via environment variable, bukan hardcoded
const dbPassword = process.env.DB_PASSWORD;
```
Untuk skala industri: gunakan secret manager terkelola (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault, Azure Key Vault) dengan rotasi otomatis, bukan sekadar file `.env` di server produksi.

**Deteksi kebocoran secret di CI:**
```yaml
# contoh step CI (gitleaks)
- name: Scan for secrets
  run: gitleaks detect --source . --verbose
```

### 5. Container/Docker Hardening

```dockerfile
FROM node:20-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --chown=appuser:appgroup . .
RUN npm ci --omit=dev
USER appuser              # jangan jalankan sebagai root
EXPOSE 3000
CMD ["node", "server.js"]
```
Checklist tambahan:
- [ ] Base image minimal (alpine/distroless), bukan `latest` tag mengambang
- [ ] Read-only root filesystem jika memungkinkan (`--read-only`)
- [ ] Scan image dengan Trivy/Grype sebelum push ke registry
- [ ] Jangan expose Docker socket ke container

### 6. Cloud IAM — Least Privilege

- Buat role/service-account terpisah per service, jangan pakai 1 akun "god mode" untuk semua.
- Policy IAM eksplisit per resource (`s3:GetObject` pada bucket spesifik), hindari `*` di action & resource.
- Aktifkan MFA wajib untuk akun root/owner cloud.
- Audit permission secara berkala (unused permissions → cabut).

### 7. Dependency & CI/CD Security

```yaml
# contoh GitHub Actions
- run: npm audit --audit-level=high
- run: npx snyk test
- name: SAST scan
  run: semgrep --config=auto .
```
- Wajib branch protection: tidak bisa push langsung ke `main`, wajib PR + review.
- Signed commits/tags untuk rilis produksi jika memungkinkan.
- Pipeline deploy tidak menyimpan credential produksi di plaintext di file YAML — gunakan CI secret store.

### 8. Rate Limiting & WAF di Level Infrastruktur

- Pasang WAF (Cloudflare, AWS WAF, ModSecurity) di depan aplikasi untuk filter pola serangan umum (SQLi, XSS signature) sebagai lapisan tambahan — bukan pengganti validasi di kode.
- Aktifkan proteksi DDoS dasar dari provider (rate-based rules, geo-blocking jika relevan).

### 9. Backup & Disaster Recovery

- Ikuti aturan 3-2-1: 3 salinan data, 2 media berbeda, 1 salinan offsite/offline (immutable, tidak bisa dienkripsi ransomware yang sama).
- Definisikan RTO (Recovery Time Objective) & RPO (Recovery Point Objective) secara eksplisit.
- **Uji restore backup secara berkala** — backup yang tidak pernah dites restore-nya dianggap tidak ada.

### 10. Monitoring & Alerting

- Sentralisasi log (ELK stack, Datadog, CloudWatch, Grafana Loki).
- Alert otomatis untuk: lonjakan error 5xx, banyak login gagal, akses admin di luar jam kerja normal, perubahan security group/firewall.
- Uptime & health check endpoint terpisah dari endpoint bisnis.


---

## Bagian D — Pemetaan Standar & Regulasi + Incident Response Plan

> Catatan: bagian regulasi (UU PDP/GDPR) berubah dari waktu ke waktu. Untuk kasus nyata bernilai hukum tinggi, verifikasi ke sumber resmi terkini atau konsultasikan dengan bagian legal/DPO — dokumen ini adalah panduan teknis awal, bukan nasihat hukum.

### 1. Tabel Pemetaan Standar → Kontrol Teknis

| Standar | Fokus | Kontrol teknis yang harus ada |
|---|---|---|
| **ISO/IEC 27001** | Manajemen keamanan informasi menyeluruh | Kebijakan tertulis, risk assessment berkala, access control policy, asset inventory, audit internal |
| **NIST CSF** | Identify–Protect–Detect–Respond–Recover | Inventaris aset & risiko, kontrol preventif (bagian checklist SKILL.md), logging/monitoring, IR plan, DR plan |
| **PCI-DSS** | Wajib jika memproses/menyimpan data kartu pembayaran | Jangan simpan data kartu mentah (pakai tokenisasi via payment gateway bersertifikat PCI), segmentasi jaringan, enkripsi transmisi, log akses ke data kartu, scan kerentanan triwulanan |
| **SOC 2** | Kepercayaan pelanggan enterprise/SaaS | Access control terdokumentasi, change management, monitoring berkelanjutan, audit trail |
| **OWASP ASVS** | Verifikasi keamanan aplikasi | Checklist Tingkat 1–4 di SKILL.md utama pada dasarnya adalah ringkasan praktis dari ASVS |
| **GDPR (Uni Eropa)** | Perlindungan data warga UE | Data minimization, consent eksplisit, right to erasure, breach notification ≤72 jam ke otoritas |
| **UU No. 27/2022 PDP (Indonesia)** | Perlindungan data pribadi WNI | Consent eksplisit (Pasal 20), langkah teknis & organisasi memadai (Pasal 35), notifikasi tertulis kebocoran ke subjek data & lembaga pengawas **paling lambat 3×24 jam** sejak diketahui (Pasal 46), akuntabilitas pengendali data (Pasal 47) |

### 2. Kapan Compliance Apa yang Relevan (aturan cepat)

- Sistem menyimpan/memproses **nomor kartu pembayaran** → PCI-DSS wajib dicek, idealnya gunakan payment gateway (Midtrans, Xendit, Stripe) agar data kartu tidak pernah menyentuh server sendiri.
- Sistem memproses **data pribadi pengguna Indonesia** (nama, NIK, email, nomor HP, dsb) → UU PDP berlaku, minimal: consent eksplisit, keamanan teknis memadai, siap proses notifikasi 3×24 jam jika ada insiden.
- Sistem punya pengguna/klien di **Uni Eropa** → GDPR berlaku, meski perusahaan berbasis di Indonesia.
- Sistem dijual ke **enterprise/klien B2B** → siapkan diri untuk audit SOC 2 sejak awal (lebih murah dibangun dari awal daripada di-retrofit).
- Sistem menangani **data kesehatan** → tambahkan enkripsi field-level & access log lebih ketat (prinsip serupa HIPAA meski beroperasi di Indonesia — praktik baik, bukan kewajiban hukum langsung kecuali diatur sektor spesifik).

### 3. Template Incident Response Plan (IRP) — Siap Pakai, Sesuaikan

#### Fase 1 — Preparation (sebelum insiden terjadi)
- Daftar kontak darurat: siapa dihubungi pertama (tech lead/CTO), siapa yang authorize komunikasi ke publik.
- Akses darurat terdokumentasi (siapa punya akses ke infra kritis saat orang biasa tidak available).
- Backup & runbook restore sudah diuji.

#### Fase 2 — Identification
- Bagaimana insiden terdeteksi (alert monitoring, laporan user, laporan pihak ketiga)?
- Klasifikasi tingkat keparahan (low/medium/high/critical) berdasarkan data yang terdampak & jumlah pengguna.

#### Fase 3 — Containment
- Langkah cepat: isolasi sistem/akun yang terdampak, revoke kredensial/token yang dicurigai bocor, matikan fitur yang jadi celah sementara jika perlu.
- Jangan hapus log/evidence — perlu untuk investigasi & kepatuhan.

#### Fase 4 — Eradication
- Cari root cause (bukan hanya gejala), patch celah, rotasi semua secret/kredensial yang berpotensi terekspos.

#### Fase 5 — Recovery
- Restore dari backup bersih, verifikasi integritas sebelum kembali online, monitoring ketat pasca-insiden.

#### Fase 6 — Notification (jangan lupa, sering terlewat)
- Jika data pribadi pengguna Indonesia terdampak: siapkan notifikasi tertulis ke subjek data & lembaga pengawas terkait sesuai tenggat yang berlaku saat ini (rujuk Pasal 46 UU PDP; konfirmasikan mekanisme pelaporan resmi terkini ke lembaga pengawas).
- Isi notifikasi minimal: jenis data yang terdampak, kapan & bagaimana insiden terjadi, langkah penanganan/pemulihan yang sudah/akan dilakukan.

#### Fase 7 — Lessons Learned
- Post-mortem tanpa menyalahkan individu (blameless), dokumentasikan, perbarui checklist keamanan & IRP berdasarkan temuan.

### 4. Prioritas Jika Sumber Daya Terbatas

Untuk startup/tim kecil yang tidak bisa langsung penuhi semua standar formal, urutan prioritas realistis:
1. Checklist Tingkat 1 & 2 di SKILL.md utama (basic hygiene + menengah) — dampak terbesar per usaha terkecil.
2. UU PDP dasar jika menangani data pengguna Indonesia (consent + keamanan teknis dasar).
3. PCI-DSS *scope reduction* — serahkan pemrosesan kartu ke payment gateway pihak ketiga sepenuhnya agar tidak perlu sertifikasi PCI penuh.
4. ISO 27001/SOC 2 formal — biasanya baru relevan saat mulai berhadapan dengan klien enterprise yang mensyaratkannya secara kontraktual.


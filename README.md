# README — Widget Komentar Sederhana (HTML + Firebase Firestore)

Komponen komentar ringan, 1 file HTML, tanpa bundler. Menyimpan dan menampilkan komentar per-halaman menggunakan **Firebase Firestore**. Cocok buat embed di situs statis (GitHub Pages, Google Sites via embed HTML, dll).

---

## Fitur

* ✅ **Tanpa build step** — cuma satu file `.html`.
* ✅ **Per-halaman** via query `?page=...` (fallback ke `unknown` bila tidak ada).
* ✅ **Tulis & baca** komentar dari Firestore.
* ✅ **Urut terbaru** (desc) — otomatis fallback kalau indeks komposit belum dibuat.
* ✅ **Keamanan dasar**: escaping HTML + linkify URL.
* 🎨 **Tema playful** warna kuning pastel; mudah diubah via CSS variables.

---

## Prasyarat

1. Project **Firebase** aktif.
2. **Cloud Firestore** di mode Native.
3. Koleksi **`comments`** (akan dibuat otomatis saat `addDoc` pertama berhasil).
4. Atur **Firestore Security Rules** (lihat bagian *Keamanan & Rules*).

---

## Cara Pakai (Cepat)

1. **Buat Project Firebase** → aktifkan **Firestore**.

2. **Ganti konfigurasi** di kode:

   ```js
   // ====== GANTI DENGAN PUNYAMU ======
   const firebaseConfig = { /* ... */ };
   // ===================================
   ```

3. **Atur Firestore Rules** sementara untuk pengujian (lihat contoh rules di bawah).

4. **Buka file** `komentar.html` di browser, atau host di GitHub Pages / domainmu.

5. **Tambahkan parameter page** di URL:

   * `komentar.html?page=/artikel/slug`
   * Tanpa parameter → tersimpan sebagai `pageId = "unknown"`.

> Catatan (Google Sites): gunakan embed HTML. Beberapa kebijakan sandbox bisa membatasi navigasi, tapi Firestore client berfungsi selama domain tidak diblokir dan rules mengizinkan.

---

## Struktur Data

Koleksi: `comments`

Dokumen contoh:

```json
{
  "pageId": "/artikel/slug-1",
  "name": "Rena",
  "text": "Suka banget sama artikelnya!",
  "ts": <serverTimestamp>,
  "ref": "https://referrer.example/page"
}
```

Field `ts` diisi `serverTimestamp()`.

---

## Perilaku Penting

* **Penentuan halaman**:
  `pageId = URLSearchParams.get("page") || "unknown"`
* **Pengambilan komentar**:

  1. Coba query: `where("pageId","==",pageId) + orderBy("ts","desc") + limit(50)`
  2. Jika error “requires an index”, fallback: query tanpa `orderBy`, lalu sort manual di klien (desc).
* **Keamanan tampilan**:
  `escapeHtml` mencegah XSS; `linkify` mendeteksi `http(s)://...` dan membungkus dengan `<a target="_blank" rel="noopener noreferrer">`.

---

## Keamanan & Rules (Penting!)

Untuk produksi, **jangan** biarkan write publik tanpa kontrol. Minimal:

### Opsi A — Development (mudah tapi riskan)

Mengizinkan read publik & create dengan validasi dasar:

```js
// Firestore Rules (sementara untuk dev)
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /comments/{id} {
      allow read: if true; // publik boleh baca

      allow create: if
        request.time < timestamp.date(2100,1,1) &&
        request.resource.data.pageId is string &&
        request.resource.data.name is string &&
        request.resource.data.text is string &&
        request.resource.data.text.size() > 1 &&
        request.resource.data.text.size() <= 2000 &&
        request.resource.data.name.size() <= 80 &&
        // hanya serverTimestamp untuk 'ts'
        request.resource.data.ts == request.time;

      // Disallow update/delete publik
      allow update, delete: if false;
    }
  }
}
```

> Keterangan: `request.resource.data.ts == request.time` mensyaratkan ts diisi server (kira-kira). Di klien, kita set `serverTimestamp()`, yang Firestore evaluasi di server.

### Opsi B — Produksi (disarankan)

* Gunakan **Authentication** (Anonymous/Email) lalu batasi write ke user terautentikasi.
* Tambahkan **reCAPTCHA** / rate limit (Cloud Functions).
* Moderasi: simpan ke `pending_comments`, admin memindahkan ke `comments` setelah review.

---

## Indeks Komposit (Supaya Query Utama Cepat)

Query: `where("pageId","==", pageId).orderBy("ts","desc")`

Buat **Composite Index**:

* **Collection**: `comments`
* **Fields**:

  * `pageId` — Asc
  * `ts` — Desc

Cara buat:

* Jalankan aplikasi; Firestore akan melempar error dengan link untuk membuat index. Klik, konfirmasi.
* Atau buat manual di **Firestore → Indexes → Composite → Add Index**.

Sambil index dibuat, kode sudah punya **fallback** (tanpa `orderBy`, sort manual di klien).

---

## Kustomisasi Tampilan

Ubah variabel CSS di bagian `:root`:

```css
:root {
  --card:#fff7cc;   /* kartu */
  --ink:#3a2f00;    /* teks */
  --muted:#7a6f3a;  /* teks sekunder */
  --line:#ffd633;   /* garis aksen */
}
```

Komponen utama:

* `.wrap` — kontainer
* `form` — kartu input
* `.list` — daftar komentar
* `.item` — kartu komentar
* `.who`, `.when`, `.text` — tipografi

---

## Deploy

* **GitHub Pages**: commit file, aktifkan Pages. Pastikan rules Firestore sudah sesuai domain publik.
* **Custom domain / hosting statis**: unggah file.
* **Google Sites**: mode embed HTML; pastikan query `?page=...` di URL yang di-embed (atau set nilai default lain di kode).

> **Restrict API key**: di Google Cloud Console → Credentials → batasi **HTTP referrers (web sites)** hanya ke domain kamu.

---

## Troubleshooting

* **Semua pageId jadi `unknown`**
  Pastikan URL menyertakan `?page=...`. Google Sites kadang “menyembunyikan” query; kamu bisa:

  * Mengisi `pageId` secara manual (hardcode) untuk tiap embed, **atau**
  * Membaca `document.referrer` (sudah disimpan ke field `ref`) sekadar jejak.

* **Error: “requires an index”**
  Itu normal saat pertama pakai query gabungan. Klik link yang disediakan Firestore, buat index. Sementara, fallback di klien sudah aktif.

* **Tidak bisa kirim komentar (PERMISSION\_DENIED)**
  Cek **Firestore Rules** — pastikan `create` diizinkan sesuai validasi yang kamu pakai.

* **Komentar tidak muncul urut terbaru**
  Indeks komposit belum siap; tunggu index selesai atau cek fallback sort di klien tetap berjalan.

* **Tidak ingin publik menulis**
  Aktifkan Auth, ubah rules agar `request.auth != null`.

---

## Catatan Kode

* Pemanggilan Firebase menggunakan **ES Modules dari CDN**:

  ```html
  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/12.1.0/firebase-app.js";
    import { getFirestore, ... } from "https://www.gstatic.com/firebasejs/12.1.0/firebase-firestore.js";
  </script>
  ```
* **Escaping & Linkify**:

  * `escapeHtml()` melindungi dari XSS.
  * `linkify()` membungkus URL menjadi tautan; dikombinasikan setelah escape.

---

## Roadmap (Opsional)

* 🔒 Auth (Anonymous / Email) + moderasi.
* 🧹 Spam control (rate-limit, reCAPTCHA).
* 🗑️ Fitur hapus/edit (hanya admin/penulis).
* 📄 Pagination / infinite scroll untuk >50 komentar.
* 🌐 i18n (format tanggal lokal/UTC).

---

## Lisensi

Bebas dipakai untuk proyek pribadi/komersial. Tambahkan kredit jika berkenan.

---

## Kredit

Dibangun di atas **Firebase Firestore** dan HTML murni. Desain kuning pastel yang hangat biar kolom komentar terasa ramah. Selanjutnya kamu bisa menyatu-padankan dengan tema situsmu, atau meng-abstract ke komponen Web Component/React bila butuh reuse.

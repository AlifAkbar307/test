# Memahami Kotak Hitam — Cara Kerja Tool Hub-mu

Ditulis untuk kamu yang paham HTML dasar + sedikit Python, tapi belum React.
Tiap konsep pakai kode dari project-mu sendiri sebagai contoh.

**Cara baca:** kamu TIDAK perlu hafal apa pun. Tujuannya supaya saat lihat kode,
kamu bisa bilang "oh, ini bagian yang itu" — bukan buta total. Baca satu bagian,
istirahat, lanjut. Tidak harus sekali duduk.

Ada tanda 🔍 **Coba sendiri** di beberapa tempat — kalau kamu penasaran seperti
waktu ngetes Python, itu latihan kecil yang bisa kamu jalankan.

---

## Bagian 1 — Async & Fetch (kenapa ada "loading", kenapa bisa gagal)

Ini yang kamu paling penasaran, dan muncul di Currency Converter.

### Masalah dasarnya: internet itu lambat, kode itu cepat

Waktu kamu minta kurs dari server (`cdn.jsdelivr.net`), datanya harus melakukan
perjalanan: dari browsermu → ke server (mungkin di negara lain) → balik lagi.
Itu makan waktu — mungkin 0,3 detik, mungkin 3 detik kalau internet lemot.

Masalahnya: JavaScript (bahasa app-mu) secara default **tidak mau menunggu**. Dia
akan lanjut ke baris berikutnya secepat mungkin. Bayangkan kamu nyuruh orang
"beliin kopi" lalu langsung lanjut ngomong tanpa nunggu kopinya dateng — kamu
ngomong ke tangan kosong. Itu yang terjadi kalau kode tidak diberi tahu untuk
menunggu.

### Solusinya: `async` dan `await`

Lihat kode fetchRates-mu (disederhanakan):

```javascript
async function fetchRates(base) {
  const res = await fetch(url);      // (1) minta data, TUNGGU sampai datang
  const data = await res.json();     // (2) ubah jadi objek, TUNGGU juga
  return data;                       // (3) baru kembalikan
}
```

- **`fetch(url)`** = "pergi ke alamat ini, ambil datanya". Ini si "beliin kopi".
- **`await`** = "TUNGGU di sini sampai selesai, jangan lanjut dulu". Ini yang
  bikin kode sabar nunggu kopinya dateng.
- **`async`** = penanda di atas fungsi yang bilang "fungsi ini boleh pakai await".
  Kamu tidak bisa pakai `await` tanpa `async` di atasnya — mereka sepasang.

Jadi baris (2) tidak akan jalan sampai baris (1) benar-benar selesai. Kopinya
dateng dulu, baru lanjut.

### Kenapa ada tampilan "Mengambil kurs..."

Karena menunggu itu makan waktu, kamu tidak mau layar kosong/beku selama nunggu.
Jadi kode bilang: "sambil nunggu, tampilkan tulisan Mengambil kurs...". Di kodemu:

```javascript
setLoading(true);          // sebelum fetch: nyalakan status "loading"
// ... await fetch ...     // nunggu
setLoading(false);         // setelah selesai: matikan
```

Saat `loading` true, UI menampilkan "Mengambil kurs…". Saat false, tampilkan hasil.
Itu kenapa kamu lihat teks itu sekilas sebelum angka muncul.

### Kenapa perlu try/catch (fail-loud)

Fetch bisa GAGAL — internet putus, server mati, alamat salah. Kalau tidak
diantisipasi, app-mu error dan blank. Jadi:

```javascript
try {
  const res = await fetch(url);   // coba ambil
  // ... pakai datanya ...
} catch {
  // kalau gagal, masuk sini — tampilkan pesan error, bukan blank
}
```

`try` = "coba lakukan ini". `catch` = "kalau di dalam try ada yang meledak,
tangkap di sini dan lakukan ini". Ini yang bikin converter-mu menampilkan
"Gagal mengambil kurs" (fail-loud) daripada layar rusak diam-diam.

### Kenapa ada dua URL (fallback)

Kodemu coba CDN utama dulu; kalau gagal, coba yang kedua:

```javascript
const urls = [urlUtama, urlCadangan];
for (const url of urls) {
  try {
    const res = await fetch(url);
    if (res.ok) return data;   // berhasil? langsung kembalikan, berhenti
  } catch {
    // gagal? loop lanjut ke url berikutnya
  }
}
throw new Error("Gagal");   // dua-duanya gagal? baru menyerah
```

Ini seperti punya nomor telepon cadangan: kalau yang pertama tak diangkat, coba
yang kedua sebelum menyerah. Bikin app lebih tahan banting.

🔍 **Coba sendiri (di browser):** buka Currency Converter, tekan F12 (buka
DevTools), tab "Network". Convert sesuatu. Kamu akan lihat baris permintaan ke
jsdelivr — itu si fetch. Klik, lihat "Response" — itu data JSON kurs mentahnya.
Kamu lagi ngeliat "kopi"-nya dateng.

---

## Bagian 2 — Regex Separator-Terakhir (trik parser angka)

Ini di Quote Parser, dan kamu yang minta desainnya. Ini niche tapi elegan.

### Masalahnya: dua format angka yang bertolak belakang

- `431,000.00` → koma = ribuan, titik = desimal (format Inggris)
- `55.268.000,00` → titik = ribuan, koma = desimal (format Indonesia)

Kalau parser salah tebak, `55.268` bisa dibaca 55 (bukan 55 ribu) — meleset 1000x.
Ini "jebakan 1000x" yang kita bahas.

### Solusi: yang muncul TERAKHIR pasti desimal

Trik-nya sederhana tapi pintar: di angka mana pun, pemisah **desimal selalu paling
belakang** (karena desimal ada di ujung). Jadi:
- `431,000.00` → yang terakhir muncul `.` → titik = desimal ✓
- `55.268.000,00` → yang terakhir muncul `,` → koma = desimal ✓

Tanpa perlu tahu format apa, cukup lihat mana yang paling kanan.

### Di mana regex masuk

Regex (Regular Expression) = cara mencari/mengganti pola teks. Contoh dari kodemu:

```javascript
s.replace(/[,\s]/g, '')
```

Bongkar potongan ini:
- `/.../ ` = tanda ini regex (seperti tanda kutip menandai teks biasa)
- `[,\s]` = "cari koma ATAU spasi". Kurung siku `[]` artinya "salah satu dari ini".
  `\s` = kode khusus untuk spasi.
- `g` = "global" = ganti SEMUA yang cocok, bukan cuma yang pertama
- `.replace(..., '')` = ganti yang ketemu dengan kosong (yaitu: hapus)

Jadi baris itu artinya: **"hapus semua koma dan spasi dari teks s"**. Berguna untuk
membersihkan `IDR 431,000` jadi angka mentah `431000`.

### Kenapa regex worth dikenal

Regex itu seperti "ctrl+F super" — bisa cari pola, bukan cuma teks persis. Misal
"cari semua yang berbentuk angka" atau "cari yang diawali IDR". Sekali paham,
kepakai di mana-mana (bahkan di VS Code search kamu bisa pakai regex).

🔍 **Coba sendiri (Python, karena kamu sudah familiar):**
```python
import re
print(re.sub(r'[,\s]', '', 'IDR 431,000.00'))   # hasil: IDR431000.00
```
`re.sub` di Python = `.replace` regex di JavaScript. Pola `[,\s]`-nya sama persis.
Coba ganti polanya, lihat apa yang kehapus.

---

## Bagian 3 — Data Terpisah dari Logika (konsep paling berharga)

Ini prinsip yang kita pegang dari awal, dan yang paling transferable ke project
apa pun. Kalau cuma satu hal yang kamu bawa, ambil ini.

### Analogi: resep vs dapur

Bayangkan restoran. Ada **resep** (daftar bahan + takaran) dan ada **dapur** (alat
masak + cara masak). Kalau resep ditulis di dinding dapur pakai cat permanen,
tiap ganti menu kamu harus ngecat ulang dinding. Ribet, dan gampang ngerusak dapur.

Lebih baik: resep di **kartu terpisah**, dapur cuma "baca kartu lalu masak". Ganti
menu = ganti kartu, dapur tidak disentuh.

Di project-mu:
- **Kartu resep** = `src/content/data.ts` (semua snippet, tabel, aturan, mata uang)
- **Dapur** = komponen UI (JiraHelper.tsx, dll) yang cuma "baca data.ts lalu tampilkan"

### Contoh nyata dari kodemu

Di data.ts:
```javascript
export const SNIPPET_GROUPS = [
  { id: "sp3bp", name: "SP3BP", snippets: [ ... ] },
  // ...22 snippet
];
```

Di komponen (dapur):
```javascript
{SNIPPET_GROUPS.map((group) => (
  // gambar tiap grup di layar
))}
```

Kata `.map` itu artinya "untuk SETIAP item di daftar, lakukan ini". Jadi komponen
tidak tahu-menahu isi snippet-nya apa — dia cuma bilang "apa pun yang ada di
SNIPPET_GROUPS, gambar satu per satu". 

**Inilah kenapa kamu bisa nambah snippet tanpa AI:** kamu edit kartu resep
(data.ts), tambah satu entri, dan dapur otomatis menggambarnya. Kamu tidak pernah
menyentuh "cara masak". Ini yang kita jaga mati-matian sepanjang project.

### Kenapa ini penting untukmu

Kalau data & logika tercampur, tiap perubahan kecil butuh paham seluruh kode (atau
minta AI). Kalau terpisah, perubahan konten = edit satu file data yang gampang
dibaca. Ini bedanya "bisa maintain sendiri" vs "selalu tergantung AI".

---

## Bagian 4 — State di React (kenapa field kosong saat ganti halaman)

Ini menjelaskan banyak perilaku app-mu yang mungkin terasa misterius.

### State = ingatan sementara komponen

"State" itu data yang bisa berubah saat app jalan — mis. teks yang lagi kamu ketik
di field nama. Di React ditulis begini:

```javascript
const [namaLengkap, setNamaLengkap] = useState("");
```

Bongkar:
- `useState("")` = "buat satu ingatan, nilai awalnya kosong `""`"
- `namaLengkap` = nilai ingatannya sekarang (buat dibaca)
- `setNamaLengkap` = fungsi untuk MENGUBAH ingatannya
- Selalu berpasangan: `[nilai, caraUbahNilai]`

Jadi saat kamu ketik di field, tiap huruf memanggil `setNamaLengkap(...)`, yang
memperbarui `namaLengkap`, yang bikin layar gambar ulang dengan teks baru. Itu
kenapa yang kamu ketik langsung muncul.

### Kenapa hilang saat ganti halaman

State itu **hidup selama komponen tampil**. Waktu kamu pindah dari Jira Helper ke
halaman lain, komponen Jira Helper "dimatikan" (istilahnya: unmount) — dan
ingatannya ikut hilang. Balik lagi = komponen dibuat baru, ingatan mulai kosong.

**Ini bukan bug — ini disengaja** (kita bahas ini). Untuk nama customer, kamu JUSTRU
mau ini: biar nama customer kemarin tidak nyangkut ke quote hari ini. Kalau mau
disimpan permanen, butuh "localStorage" — tapi kita sengaja tidak pakai, karena
data nyangkut lebih berbahaya daripada ketik ulang.

### Kaitannya dengan Currency Converter

Ingat converter menyimpan kurs di "cache" biar tidak fetch berulang? Itu juga state:
```javascript
const [cache, setCache] = useState({});
```
Kurs yang sudah diambil disimpan di ingatan ini selama halaman terbuka. Ganti "Ke"
mata uang tidak fetch ulang karena kursnya sudah ada di cache. Tutup/pindah halaman
= cache hilang, fetch lagi nanti. Konsisten dengan sifat state.

---

## Bagian 5 — Clipboard: Text vs HTML (kenapa `**` gagal, `<strong>` jalan)

Kamu kena masalah ini waktu bikin judul tabel Jira tebal.

### Clipboard bisa menyimpan dua "rasa" sekaligus

Waktu kamu copy sesuatu, clipboard bisa menyimpan versi **teks polos** DAN versi
**HTML** (teks berformat) bersamaan. Aplikasi tujuan memilih mana yang dibaca.

- Notepad cuma paham teks polos → tampil apa adanya
- Jira/Google Docs paham HTML → tampil berformat (tabel, tebal, dll)

### Kenapa tabelmu jadi tabel di Jira

Tombol salin tabelmu menaruh versi HTML `<table>` ke clipboard:
```javascript
await navigator.clipboard.write([
  new ClipboardItem({
    'text/html': new Blob([htmlTabel], { type: 'text/html' }),
    'text/plain': new Blob([teksPolos], { type: 'text/plain' })
  })
]);
```
Jira baca `text/html`, lihat `<table>`, gambar tabel. Kalau cuma teks polos
(seperti versi awal kita), Jira tidak punya HTML → tempel sebagai teks biasa.

### Kenapa `**tebal**` gagal tapi `<strong>` berhasil

- `**tebal**` = sintaks **Markdown** (bahasa format lain)
- `<strong>tebal</strong>` = sintaks **HTML**

Tabelmu dikirim sebagai HTML. Mencampur Markdown ke dalam HTML = Jira bingung,
pilih salah satu. Karena tabel sudah HTML, judul juga harus HTML (`<strong>`) biar
sekeluarga. Itu kenapa `<strong>` jalan bareng tabel, `**` tidak.

### Kenapa butuh HTTPS

`navigator.clipboard.write` (versi canggih dengan HTML) cuma jalan di koneksi aman
(HTTPS). Preview lokal kadang bukan HTTPS → gagal. Vercel HTTPS → jalan. Itu kenapa
aku selalu bilang "tes di Vercel, bukan preview".

---

## Bagian 6 — Alur Git & GitHub (yang kemarin tenggelam)

Ini yang kamu eksplisit minta. Aku jelaskan model mentalnya dulu, baru perintahnya.

### Tiga tempat berbeda (sering ketuker)

- **Git** = program di laptopmu yang mencatat versi/perubahan file. Offline, lokal.
- **GitHub** = penyimpanan online untuk kode. Seperti Google Drive khusus kode.
- **Vercel** = layanan yang ambil kode dari GitHub, "masak" jadi website, sajikan.

Alur besar: **kode di laptop → (Git) → GitHub → (Vercel) → website live.**

### Model mental: snapshot berlapis

Bayangkan Git seperti tombol "save game" yang bisa kamu tekan kapan saja. Tiap
"save" (disebut **commit**) menyimpan foto seluruh project saat itu, dengan catatan.
Kalau ada yang rusak, kamu bisa balik ke save sebelumnya.

### Perintah dasar & artinya

```bash
git add .
```
"Siapkan SEMUA perubahan untuk disimpan." Titik `.` = semua file. Ini seperti
memilih apa yang mau masuk ke foto. (File di `.gitignore` otomatis dilewati.)

```bash
git commit -m "fix currency converter"
```
"Ambil fotonya, kasih catatan 'fix currency converter'." Ini save game-nya. `-m` =
message (catatan). Catatan yang jelas menolongmu ingat "waktu itu aku ngapain".

```bash
git push
```
"Kirim foto-fotoku ke GitHub." Sebelum push, semua commit cuma ada di laptop.
Setelah push, naik ke GitHub, dan Vercel otomatis mulai bikin website versi baru.

### Kenapa ada `.gitignore`

Beberapa folder tidak perlu dikirim: `node_modules` (ratusan MB library yang bisa
di-download ulang), `dist` (hasil masak yang Vercel bikin sendiri). `.gitignore` =
daftar "jangan pernah masukkan ini ke foto". Bikin repo ringan & bersih.

### Alur harian kamu sekarang (paling penting)

Karena Vercel auto-deploy tiap push, alur maintenance-mu jadi:

1. Edit file di GitHub (atau lokal) — mis. tambah snippet di data.ts
2. `git add .` → `git commit -m "..."` → `git push` (atau commit lewat tombol GitHub)
3. Tunggu 1-2 menit → Vercel otomatis deploy → website update

Kalau edit lewat **GitHub web** langsung (tombol pensil di file), kamu skip perintah
git — GitHub yang commit untukmu saat kamu klik "Commit changes". Vercel tetap
auto-deploy. Ini cara paling gampang untuk perubahan kecil.

### Kalau build gagal

Vercel TIDAK menerbitkan build yang gagal — website lama tetap jalan. Jadi kesalahan
editmu paling banter "perubahan belum muncul", bukan "situs rusak di depan tim". Ini
yang bikin GitHub-edit jadi latihan maintaining yang aman: salah = ketahuan, tapi
tidak merusak yang live. Kamu lihat error di dashboard Vercel, perbaiki, push lagi.

🔍 **Coba sendiri:** buka repo di GitHub, edit satu kata di komentar data.ts lewat
tombol pensil, klik "Commit changes". Buka Vercel → lihat deployment baru muncul &
jalan. Kamu baru saja melakukan siklus penuh tanpa terminal.

---

## Kalau nanti mau nulis kode sendiri (bukan tujuan utama, tapi kalau penasaran)

Urutan belajar yang aku sarankan, dari yang paling berguna untukmu:

1. **Selesaikan HTML** (kamu sudah mulai) + **CSS** secukupnya. Ini fondasi visual.
2. **JavaScript dasar** — variabel, fungsi, array, `.map`, if/else. Ini "logika"-nya.
   Kamu sudah punya intuisi dari Python; JS mirip, beda sintaks.
3. **React** — baru setelah JS nyaman. Konsep inti: komponen (potongan UI yang bisa
   dipakai ulang) + state (yang sudah kamu pahami di Bagian 4).
4. **Git** — kamu sudah pakai; tinggal perdalam kalau perlu (branch, dll).

Jangan lompat ke React sebelum JS. Itu kesalahan umum yang bikin frustrasi — React
itu JS, kalau JS-nya kabur, React terasa sihir yang tidak masuk akal.

**Cara belajar yang cocok buat kamu** (dari caramu ngetes Python): ambil satu fungsi
kecil di project ini, tempel ke tempat yang bisa kamu jalankan, ubah-ubah, lihat apa
yang berubah. Belajar dengan "ngutak-ngatik yang sudah ada" lebih nempel daripada
baca teori. Kamu sudah punya seluruh project sebagai bahan latihan.

---

## Penutup

Kamu tidak perlu paham semua ini untuk terus jalan — tool-mu sudah live dan kamu
sudah bisa maintain konten sendiri. Tapi memahami "kotak hitam"-nya bikin kamu:
- Tidak panik saat ada error (tahu kira-kira di mana masalahnya)
- Bisa mengarahkan AI lebih tepat (tahu apa yang kamu minta)
- Perlahan lepas dari ketergantungan penuh ke AI

Itu garis sehat yang kita bahas: paham cukup untuk mendiagnosis & mengarahkan.
Bukan harus bisa nulis semuanya dari nol — tapi tidak buta juga.

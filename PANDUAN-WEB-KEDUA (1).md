# Panduan Memasang Web Promosi WhatsApp (Web Baru dari Awal)

Isi paket: website (Next.js), SQL database lengkap (`supabase/schema.sql`), dan backend WhatsApp (`baileys-server/`) untuk panel Pterodactyl.

Setiap web baru butuh **satu set sendiri**: repository GitHub, project Vercel, project Supabase, dan satu server di panel Pterodactyl. Jangan memakai ulang kunci rahasia dari web pertama.

---
## 0. Siapkan 4 kunci rahasia (catat di tempat aman)
Buat 3 teks acak (huruf + angka saja, minimal 32 karakter) memakai pembuat password acak apa pun:
- `KUNCI_API`   → dipakai sebagai BAILEYS_API_KEY
- `KUNCI_CRON`  → dipakai sebagai CRON_SECRET
- `KUNCI_STOP`  → dipakai sebagai OPTOUT_CALLBACK_SECRET (opsional)

## 1. Supabase (database)
1. Buka supabase.com → **New project** (beri nama, pilih region terdekat, buat password database).
2. Tunggu selesai, lalu buka **SQL Editor → New query**. Tempel **seluruh isi `supabase/schema.sql`** → **Run**. Harus muncul "Success".
3. Buka **Authentication → Providers → Email** → matikan **Confirm email** (supaya pengguna bisa langsung masuk setelah daftar). Simpan.
4. Buka **Project Settings → API**. Salin 3 nilai ini:
   - **Project URL** → `NEXT_PUBLIC_SUPABASE_URL`
   - **anon public key** → `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - **service_role key** → `SUPABASE_SERVICE_ROLE_KEY` (RAHASIA, jangan dibagikan)

## 2. GitHub (menaruh kode)
Browser HP tidak bisa mengunggah folder, jadi pakai pengekstrak otomatis:
1. Buat repository baru (boleh private). Centang "Add a README" agar tidak kosong.
2. **Add file → Create new file**. Nama file ketik persis: `.github/workflows/unzip-project.yml`. Isi dengan isi file `unzip-project.yml` dari paket ini. **Commit**.
3. Di halaman utama repository: **Add file → Upload files** → pilih **`project.zip`** (nama harus tetap `project.zip`) → **Commit**.
4. Buka tab **Actions**, tunggu proses "Ekstrak project" bercentang hijau. Folder `app`, `lib`, `package.json`, dan lainnya akan muncul sendiri.
   - Jika gagal izin (403): **Settings → Actions → General → Workflow permissions → Read and write permissions → Save**, lalu jalankan ulang.

## 3. Vercel (menayangkan web)
1. vercel.com → **Add New → Project** → pilih repository tadi → **Import** (Framework: Next.js terdeteksi otomatis).
2. Buka **Environment Variables**, isi (centang **Production**; tipe Secret untuk kunci):

| Nama | Nilai |
|---|---|
| NEXT_PUBLIC_SUPABASE_URL | dari Supabase |
| NEXT_PUBLIC_SUPABASE_ANON_KEY | dari Supabase |
| SUPABASE_SERVICE_ROLE_KEY | dari Supabase |
| BAILEYS_API_KEY | `KUNCI_API` |
| CRON_SECRET | `KUNCI_CRON` |
| BAILEYS_API_URL | isi nanti di langkah 5 (sementara `http://belum-diisi`) |
| OPTOUT_CALLBACK_SECRET | `KUNCI_STOP` (opsional) |
| NEXT_PUBLIC_APP_NAME | nama web kamu (opsional) |

3. Tekan **Deploy**. Catat alamat webnya, misalnya `https://namaweb.vercel.app`.

## 4. Panel Pterodactyl (backend WhatsApp)
Pakai **satu server baru** untuk web ini (jangan digabung dengan web pertama).
1. Buat/siapkan server di panel dengan egg **Node.js**, versi **20 atau 22** (tab Startup → Docker image).
2. Di tab **Files**, folder utama (`/home/container`), unggah: `server.js` dan `package.json` (dari folder `baileys-server/` atau paket `panel-baru.zip`).
3. **New File** bernama `.env` (dengan titik di depan), isi:
```
BAILEYS_API_KEY=KUNCI_API
CRON_URL=https://namaweb.vercel.app/api/cron/send
CRON_SECRET=KUNCI_CRON
OPTOUT_CALLBACK_URL=https://namaweb.vercel.app/api/webhooks/baileys
OPTOUT_CALLBACK_SECRET=KUNCI_STOP
```
Ganti `KUNCI_...` dan `namaweb.vercel.app` dengan nilai aslimu. Tekan **SAVE CONTENT**.
4. Tab **Console** → **Start**. Tunggu pemasangan pustaka (1 sampai 2 menit). Harus muncul `Pengirim otomatis aktif` dan `Baileys API listening on ...`.
5. Catat **alamat dan port** server (terlihat di Console, misalnya `namaserver.hostkita.help:20994`).
6. Tes di browser: `http://ALAMAT:PORT/health` → harus tampil `{"ok":true,"service":"el-shop-baileys"}`.
   - Ditolak padahal server menyala? Tambahkan `PORT=nomorport` di `.env`, Restart. Jika tetap ditolak, port belum dibuka hosting (hubungi penyedia).

## 5. Hubungkan web dengan panel
1. Di Vercel → Environment Variables, ubah **BAILEYS_API_URL** menjadi `http://ALAMAT:PORT` (tanpa `/` di akhir).
2. **Redeploy** (Deployments → deployment teratas → titik tiga → Redeploy). Tunggu **Ready**.

## 6. Jadikan dirimu admin
1. Buka webnya dan **Daftar** akun (ini akan jadi akun admin).
2. Di Supabase SQL Editor jalankan (ganti emailnya):
```
update public.profiles set role = 'admin'
where id = (select id from auth.users where email = 'EMAIL-KAMU');
```
3. Keluar lalu masuk lagi. Menu admin (Beranda, Perangkat, Database, Campaign, Pengguna, Penarikan, Riwayat) akan muncul.

## 7. Pengaturan awal (akun admin)
1. Menu **Pengguna**: isi **tarif per pesan**, **minimal penarikan**, dan **komisi referral**, lalu simpan.
2. Menu **Database**: unggah file `.txt` berisi nomor (satu per baris), centang konfirmasi, **Impor nomor**.
3. Menu **Campaign**: buat campaign (isi pesan, tombol link opsional) → tekan **Aktifkan**.
4. Menu **Perangkat**: tautkan WhatsApp lewat **QR** atau **Pairing Code**.
5. Beranda → pilih kecepatan (Slow disarankan) → **Mulai**. Saldo bertambah hanya untuk pesan yang sampai (centang 2).

## 8. Jika ada masalah
| Gejala | Penyebab / solusi |
|---|---|
| "Unauthorized" di Perangkat | BAILEYS_API_KEY di Vercel dan `.env` panel tidak sama persis. Samakan lalu Redeploy + Restart. |
| "tidak dapat dijangkau" | BAILEYS_API_URL salah atau port belum terbuka. Tes `/health`. |
| Pesan tidak terkirim | Cek Console panel; pastikan CRON_URL dan CRON_SECRET benar dan perangkat berstatus terhubung. |
| Saldo tidak bertambah | Pesan harus sampai (centang 2). Cek status di Riwayat. Pastikan `CRON_URL` terisi agar laporan sampai terkirim. |
| Tampilan tidak berubah setelah update | Cek Vercel → Deployments: harus **Ready**. Jika Error, buka Build Logs. |

## 9. Memperbarui web di kemudian hari
Unggah `project.zip` baru (atau zip berisi file yang berubah, dinamai `project.zip`) ke halaman utama repository. Pengekstrak otomatis menimpa file lama dan Vercel membangun ulang. Jika ada file SQL baru, jalankan di Supabase SQL Editor lebih dulu.

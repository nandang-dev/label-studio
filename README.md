# Label Studio — Labeling Foto Layak / Tidak Layak

Label Studio yang di-host di Coolify supaya **tim lain** bisa melabeli foto muatan dari browser. Hasilnya dipakai
untuk melatih classifier *fast reject* di Stack Inspection API (lihat `brief/PLAN-fast-reject.md`).
Repo ini **hanya berisi konfigurasi**; aplikasinya memakai image resmi
[`heartexlabs/label-studio`](https://github.com/HumanSignal/label-studio) (Apache-2.0) + PostgreSQL.

```
   Tim labeling (browser) ──https──▶ Coolify proxy ──▶ label-studio:8080 ──▶ postgres:5432
                                                          │
                                         /data/label-studio/data   (foto + media, di VPS)
```

## Isi repo

| File | Isi |
|---|---|
| `docker-compose.yaml` | Service `label-studio` (1.13.1, sama dengan carton-trainer) + `postgres` 16, volume, health check |
| `label-config.xml` | Tampilan labeling: 4 pilihan dengan hotkey 1–4 |
| `PANDUAN-LABELING.md` | Panduan singkat untuk tim labeling (boleh dibagikan) |
| `.env.example` | Variabel untuk menjalankan lokal |

## Kebutuhan resource

±1,5 GB RAM saat dipakai (Label Studio dibatasi 2 GB, PostgreSQL 512 MB). Foto ±2–3 MB per file →
1.000 foto ±3 GB disk. CPU hampir tidak terpakai, jadi tidak mengganggu Stack Inspection API.

## Deploy di Coolify

1. **Siapkan folder di VPS** (Coolify → Servers → localhost → Terminal, atau SSH):
   ```sh
   sudo mkdir -p /data/label-studio/data /data/label-studio/postgres
   sudo chown -R 1001:0 /data/label-studio/data
   ```
   Label Studio berjalan sebagai user `1001`; tanpa langkah ini muncul `permission denied`.
   Folder `postgres` diatur sendiri oleh container PostgreSQL.
2. **DNS**: buat record `A label → IP VPS` (contoh domain: `label.bagdja.com`).
3. **New Resource → Docker Compose** → pilih repo ini → Save.
4. **Environment Variables**:
   | Variabel | Contoh | Catatan |
   |---|---|---|
   | `LABEL_STUDIO_HOST` | `https://label.bagdja.com` | URL publik lengkap, tanpa `/` di akhir |
   | `LABEL_STUDIO_USERNAME` | `anotator@kantor.local` | Admin, **sama dengan Label Studio lokal** di carton-trainer |
   | `LABEL_STUDIO_PASSWORD` | (salin dari `carton-trainer/.env`) | Sama dengan `LABEL_STUDIO_PASSWORD` lokal supaya login-nya sama |

   Password database (`SERVICE_PASSWORD_POSTGRES`) **dibuat otomatis oleh Coolify**, tidak perlu diisi.
   Jangan diubah setelah deploy pertama: PostgreSQL menyimpan password saat database dibuat.
5. **Domain**: di service `label-studio`, isi `https://label.bagdja.com:8080`
   (`:8080` = port di dalam container, tidak muncul di browser).
6. **Deploy**. Kedua service harus *Running (healthy)*. Start pertama ±1–2 menit (migrasi database).
7. Buka `https://label.bagdja.com`, login dengan email & password langkah 4.

## Siapkan project labeling

1. **Create Project** → nama `Fast Reject v1`.
2. **Labeling Setup → Custom template → Code**: tempel isi `label-config.xml` → Save.
3. **Settings → Annotation**: *Distribute Labeling Tasks* = **Auto** supaya setiap orang mendapat foto berbeda.
4. Foto **jangan** di-upload manual satu per satu. Foto dipilih dan di-upload lewat skrip pre-sort
   (`carton-trainer`), sekaligus dengan prediksi awal, sehingga tim cukup setuju/mengoreksi.
   Skrip membutuhkan **Access Token** admin: avatar (kanan atas) → *Account & Settings* → *Access Token*.

## Hubungkan carton-trainer (Mac) ke instance ini

Access Token di Coolify **berbeda** dengan token Label Studio lokal walaupun user-nya sama. Untuk mengirim
foto/prediksi ke instance Coolify, di `carton-trainer`:

- `training.yaml` → `labelstudio.url: https://label.bagdja.com` dan `labelstudio.project_id: <id project>`
- `.env` → `LABEL_STUDIO_API_KEY=<Access Token dari instance Coolify>`

Kembalikan ke `http://localhost:8080` + token lokal jika ingin memakai Label Studio lokal lagi.

## Undang anggota tim

1. Avatar → **Organization** → **Add People** → salin link undangan.
2. Kirim link + `PANDUAN-LABELING.md` ke tim. Mereka daftar dengan email masing-masing lewat link itu.
3. Pendaftaran **tanpa link ditolak** (`LABEL_STUDIO_DISABLE_SIGNUP_WITHOUT_LINK=true`).

Label Studio Community tidak punya pembagian peran: semua anggota bisa melihat semua project di instance ini.
Karena itu instance ini **khusus untuk labeling foto muatan**.

## Keamanan

- Foto berisi plat nomor dan wajah orang: jangan bagikan link undangan di grup terbuka.
- Batasi akses ke IP kantor/VPN jika memungkinkan (firewall VPS / pengaturan proxy Coolify).
- Setelah labeling selesai: export hasil, lalu **Stop** service di Coolify (data tetap di `/data/label-studio`).

## Backup & export hasil

- **Export hasil**: Project → **Export** → JSON (cukup ini untuk training).
- **Backup lengkap**: folder `/data/label-studio/` (stop service dulu supaya database konsisten).

## Menjalankan lokal (opsional)

```sh
cp .env.example .env   # isi LABEL_STUDIO_PASSWORD dan SERVICE_PASSWORD_POSTGRES
docker compose up -d   # tidak mem-publish port; tambahkan `ports: ["8080:8080"]` untuk akses dari browser
```

## Status pengujian

Diuji lokal (Colima, 2026-10-19) dengan `heartexlabs/label-studio:1.13.1` + `postgres:16-alpine`: kedua container
*healthy*, admin dari env bisa login, data tersimpan di PostgreSQL (`htx_user`), pendaftaran tanpa link ditolak
(403), `label-config.xml` valid (project dibuat lewat API dengan 4 pilihan). **Belum diuji di Coolify**
(domain, HTTPS, `CSRF_TRUSTED_ORIGINS`).

# TamaScript — Bot Care Responder (Apps Script)

> [English](DOCUMENTATION.md) · **Bahasa Indonesia**

> **Dokumen hidup.** Perbarui file ini setiap kali perilakunya berubah supaya
> siapa pun bisa memahami script ini dengan cepat dan mengubahnya dengan aman.

---

## Daftar isi

1. [Apa ini](#1-apa-ini)
2. [Script-nya](#2-script-nya)
3. [Alur tingkat tinggi](#3-alur-tingkat-tinggi)
4. [Setup & dependensi](#4-setup--dependensi)
5. [Alur utama secara detail](#5-alur-utama-secara-detail)
6. [Cara kerja kategorisasi](#6-cara-kerja-kategorisasi)
7. [Perintah status](#7-perintah-status)
8. [Struktur sheet](#8-struktur-sheet)
9. [Deploy](#9-deploy)
10. [Checklist pemeliharaan](#10-checklist-pemeliharaan)

---

## 1. Apa ini

Google Apps Script yang menjalankan **bot Google Chat** bernama
`@careresponder-app`.

Agen customer-support melaporkan masalah di Google Chat; bot membaca pesannya,
menebak kategori masalah (pencocokan kata kunci fuzzy), lalu mencatatnya ke
Google Sheet — satu tab per bulan (`Jan` … `Des`). Agen juga bisa mengubah status
laporan dari thread yang sama.

**Runtime:** Apps Script V8 · **Timezone:** `Asia/Jakarta` · **Advanced service:**
Google Chat API (`Chat`) · **Endpoint:** Google Chat app.

---

## 2. Script-nya

Repo ini hanya berisi satu file Apps Script:

| File | Peran |
|---|---|
| `skriptdoublecase.js` | **Bot-nya** — parsing, kategorisasi, cek duplikat, menulis ke sheet, perintah status |
| `README.md` / `docs/` | Dokumentasi |

> Di project Apps Script, kode bot ini bisa juga bernama `careskript.js`. File ini
> adalah sumber kebenaran untuk logika bot.

---

## 3. Alur tingkat tinggi

```
 User mengirim laporan di Google Chat
        │
        ▼
   onMessage(event)                     (skriptdoublecase.js)
        │
        ├─ parse teks   → description, source, MSISDN, User ID, Unique ID
        ├─ fuzzy-match  → Issue Type (kategori)
        ├─ cek duplikat → jika sudah tercatat, tolak dengan lokasi data lama
        │
        ▼
   appendRow ke tab "<Bulan>" pada spreadsheet SHEET_ID
        │
        ▼
   balas konfirmasi di thread yang sama
```

---

## 4. Setup & dependensi

### 4.1 Script Property yang wajib

Script membaca ID spreadsheet target dari Script Properties:

```js
PropertiesService.getScriptProperties().getProperty("SHEET_ID")
```

Jika tidak ada, bot melempar error. Cara mengaturnya:

1. Buka editor Apps Script.
2. **Project Settings → Script Properties → Add a property**.
3. Key = `SHEET_ID`, Value = ID spreadsheet (string panjang di URL sheet, antara
   `/d/` dan `/edit`).

### 4.2 Advanced service

- **Google Chat API** (`Chat`, v1) harus aktif di project.

### 4.3 Scope OAuth

- `https://www.googleapis.com/auth/spreadsheets`
- `https://www.googleapis.com/auth/chat.messages`

> Bot harus ditambahkan ke space Google Chat tujuan dan dikonfigurasi menerima
> pesan sebagai Chat app (`onMessage`).

---

## 5. Alur utama secara detail

Entry point:

- `onMessage(event)` — ada pesan dikirim di space tempat bot berada.
- `onAddToSpace(event)` — bot ditambahkan ke space.
- `onRemoveFromSpace(event)` — bot dihapus dari space.

`onMessage` menentukan apa yang dilakukan terhadap teks:

1. **`input`** → baca ulang pesan yang di-reply lalu proses.
2. **Saran typo** → kalau teks terlihat seperti perintah salah ketik, bot
   menyarankan perintah yang dimaksud (dan tidak mencatatnya).
3. **Guard pesan pendek** → pesan 1–2 karakter yang menyebut bot ditolak (tidak
   dicatat).
4. **Perintah status** (`checking`, `waiting`, `progress`, `done`, `closed`,
   `reopen`) → `careUpdateStatus()`.
5. **Lainnya** → `careProcessReport()` (mencatat laporan baru).

### `careProcessReport(rawText, spaceUrl, threadName, messageObj, spaceName)`

1. Lewati jika pesan ini sudah pernah diproses (script cache).
2. `careParseMessage()` → `{ description, source, msisdn, userId, uniqueId }`.
3. `careFuzzyMatch()` → kategori (issue type).
4. `careFindDuplicateReport()` → tolak jika sudah tercatat (dari Unique ID, User
   ID, MSISDN, atau cocok description + source lama).
5. `getMonthSheet()` → tab bulan berjalan (otomatis dibuat jika belum ada).
6. `appendRow([...])` → tulis laporan ke tab.
7. Balas ringkasan konfirmasi di thread.

Konkurensi dijaga dengan `LockService.getScriptLock()` di sekitar pengecekan
duplikat + append, supaya dua pesan tidak saling balapan.

### `careUpdateStatus(spaceName, threadName, newStatus)`

Mencari semua baris yang **Thread URL*** sama dengan URL thread ini (di semua tab
bulanan) lalu mengubah kolom **Status***.

### Helper

- `careParseMessage` — mengambil description, source, MSISDN, User ID, Unique ID.
- `careSanitizeForSheet` — menambahkan awalan `= + - @` agar sheet tidak
  menganggap teks sebagai formula.
- `careBuildThreadUrl` / `careBuildDeepLink` — membangun link `chat.google.com`.
- `careAlreadyProcessed` / `careMarkProcessed` — idempotensi berbasis script cache.

---

## 6. Cara kerja kategorisasi

Otaknya adalah array `CATEGORIES`. Setiap entry:

```js
{ name: "Account Problem", group: "Akun", keywords: ["akun", "account", "banned"] }
```

`careFuzzyMatch(text)` memberi skor ke setiap kategori dan mengembalikan
kecocokan terbaik (ambang ≥ 15). Untuk memaksa pemetaan, tambahkan ke
`CATEGORY_OVERRIDES`.

---

## 7. Perintah status

| Perintah | Status yang di-set pada baris |
|---|---|
| `checking` | PIC Checking |
| `waiting` | Waiting User Reply |
| `progress` | In Progress Fixing |
| `done` | Solved |
| `closed` | Closed |
| `reopen` | Waiting PIC Reply |

---

## 8. Struktur sheet

Bot bekerja pada tab bulanan berdasarkan **posisi** — memakai enam kolom pertama:

| # | Kolom | Diisi bot |
|---|---|---|
| 1 | `Date (Automatic)` | timestamp laporan |
| 2 | `Status*` | status laporan |
| 3 | `Issue Type*` | kategori |
| 4 | `Source*` | source |
| 5 | `Thread URL*` | link ke thread Chat |
| 6 | `Description*` | deskripsi hasil parsing |

Tracker live ("C&R Tracker SUPER 2026") punya kolom tambahan setelah ini
(`Platform`, `Notes`, `Root Cause (Oncall)`, `PIC Oncall (Automatic)`,
`Escalation Status (Automatic)`, `Priority (Automatic)`,
`Date Solved/Closed (Automatic)`); bot tidak menyentuhnya.

> Kalau tab bulan belum ada, `getMonthSheet()` membuatnya dan menulis header
> sederhana:
> `Timestamp, Status, Issue Type, Source, Thread, Description, PIC, Note, Resolution, Closed At`.
> Catatan: header ini lebih sederhana dari layout tracker di atas.

---

## 9. Deploy

1. Buka project Apps Script, paste `skriptdoublecase.js` (atau `careskript.js`)
   lalu simpan.
2. Buat versi baru / update deployment supaya Chat app memakai kode terbaru.
3. Uji dulu di space Chat test.

> `clasp` juga bisa dipakai (`clasp push` → `clasp version` → `clasp deploy`).

---

## 10. Checklist pemeliharaan

Saat kamu melakukan perubahan, pastikan:

- [ ] Script Property `SHEET_ID` masih menunjuk spreadsheet yang benar.
- [ ] Kategori/keyword baru sudah didokumentasikan di §6.
- [ ] Tab bulan tetap memakai header kanonik (lihat §8).
- [ ] Deployment Chat app memakai versi terbaru.
- [ ] Dokumen ini sudah diperbarui.
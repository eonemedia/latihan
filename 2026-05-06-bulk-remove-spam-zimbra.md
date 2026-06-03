---
Tanggal: 2026-05-06
Topik: Script Bulk Remove Spam Email Zimbra via CLI
Kategori: Teknologi
Tag: [zimbra, bash, spam, email-admin, zmmailbox, shell-script]
Model: Claude Sonnet 4.6

---

# bulk-remove-spam.sh — Dokumentasi

## Deskripsi

Script bash untuk menghapus email spam secara massal dari inbox pengguna Zimbra. Pengirim spam diberikan sebagai argumen wajib, keyword subject bersifat **opsional**, dan daftar penerima (target mailbox) dibaca dari file CSV.

Jika subject tidak diberikan, script akan menghapus **semua email dari pengirim tersebut** tanpa memfilter subject.

---

## Pemakaian

```bash
# Dengan filter subject
./bulk-remove-spam.sh [-n] "email-pengirim-spam" "kata/kalimat subject" daftar-penerima.csv

# Tanpa filter subject (hapus semua email dari pengirim)
./bulk-remove-spam.sh [-n] "email-pengirim-spam" daftar-penerima.csv
```

### Argumen

| Posisi     | Argumen                | Wajib  | Keterangan                                                   |
| ---------- | ---------------------- | ------ | ------------------------------------------------------------ |
| (opsional) | `-n`                   | Tidak  | Dry-run — simulasi tanpa benar-benar menghapus email         |
| 1          | `email-pengirim-spam`  | **Ya** | Alamat email pengirim spam (nilai `FROM`)                    |
| 2          | `kata/kalimat subject` | Tidak  | Keyword yang dicari di subject (contain). Jika dilewati, semua subject diproses |
| 2 atau 3   | `daftar-penerima.csv`  | **Ya** | File CSV berisi daftar email penerima target                 |

> **Deteksi otomatis:** Script mendeteksi apakah argumen ke-2 adalah file CSV atau keyword subject. Jika argumen ke-2 adalah file yang ada di filesystem, maka dianggap sebagai CSV (tanpa subject). Jika bukan, dianggap sebagai keyword subject dan CSV dibaca dari argumen ke-3.

---

## Format File CSV

File CSV hanya memerlukan **satu kolom** berisi daftar email penerima. Baris pertama (header) dilewati secara otomatis.

```csv
email
user1@domain.com
user2@domain.com
user3@domain.com
```

> **Catatan:** Kolom tambahan setelah kolom pertama diabaikan. File CSV dari Windows (dengan carriage return `\r`) ditangani otomatis.

---

## Contoh Pemakaian

### 1. Dry-run dengan filter subject

```bash
./bulk-remove-spam.sh -n "spammer@evil.com" "Promo Hadiah Gratis" daftar-penerima.csv
```

### 2. Eksekusi dengan filter subject

```bash
./bulk-remove-spam.sh "spammer@evil.com" "Promo Hadiah Gratis" daftar-penerima.csv
```

### 3. Dry-run tanpa filter subject (semua email dari pengirim)

```bash
./bulk-remove-spam.sh -n "spammer@evil.com" daftar-penerima.csv
```

### 4. Eksekusi tanpa filter subject

```bash
./bulk-remove-spam.sh "spammer@evil.com" daftar-penerima.csv
```

---

## Logika Filter Query

Script membangun filter `zmmailbox` secara otomatis sesuai argumen yang diberikan:

| Kondisi        | Filter yang dipakai                                       |
| -------------- | --------------------------------------------------------- |
| Dengan subject | `subject:'Promo Hadiah Gratis' AND from:spammer@evil.com` |
| Tanpa subject  | `from:spammer@evil.com`                                   |

---

## Alur Kerja Script

```
Baca argumen (FROM, [SUBJECT_KEYWORD], CSV)
        │
        ▼
Deteksi argumen ke-2: file atau keyword?
  ├─ File → CSV=$2, SUBJECT_KEYWORD=""
  └─ Bukan file → SUBJECT_KEYWORD=$2, CSV=$3
        │
        ▼
Validasi FROM & CSV
        │
        ▼
Bangun FILTER:
  ├─ Ada subject → "subject:'...' AND from:..."
  └─ Tidak ada  → "from:..."
        │
        ▼
Loop setiap baris CSV (email penerima / TO)
        │
        ▼
Jalankan pencarian via zmmailbox
        │
        ├─ Tidak ada hasil → log NONE, lanjut ke penerima berikutnya
        │
        └─ Ada hasil (list Message ID)
                │
                ├─ [DRY-RUN] → log DRYRUN, tidak hapus
                │
                └─ [NORMAL]  → hapus tiap ID via `zmmailbox dm`
                                → log OK / ERROR
```

---

## Log File

Semua aktivitas dicatat di:

```
/var/log/bulk-remove-mail.log
```

### Format entri log

| Status                | Format                                                      |
| --------------------- | ----------------------------------------------------------- |
| Tidak ada email cocok | `YYYY-MM-DD HH:MM:SS <TO> \| NONE \| <FILTER>`              |
| Dry-run               | `YYYY-MM-DD HH:MM:SS DRYRUN <TO> \| ID <msgID> \| <FILTER>` |
| Berhasil dihapus      | `YYYY-MM-DD HH:MM:SS OK    <TO> \| ID <msgID> \| <FILTER>`  |
| Gagal dihapus         | `YYYY-MM-DD HH:MM:SS ERROR <TO> \| ID <msgID> \| <FILTER>`  |

### Contoh isi log

```
=== 2026-05-06 10:00:00 START bulk remove ===
FROM   : spammer@evil.com
SUBJECT: Promo Hadiah Gratis
CSV    : daftar-penerima.csv | Dry-run: false
FILTER : subject:'Promo Hadiah Gratis' AND from:spammer@evil.com
2026-05-06 10:00:01 user1@domain.com | NONE   | subject:'Promo Hadiah Gratis' AND from:spammer@evil.com
2026-05-06 10:00:03 OK    user2@domain.com | ID 1042 | subject:'Promo Hadiah Gratis' AND from:spammer@evil.com
2026-05-06 10:00:04 OK    user2@domain.com | ID 1087 | subject:'Promo Hadiah Gratis' AND from:spammer@evil.com
2026-05-06 10:00:05 ERROR user3@domain.com | ID 993  | subject:'Promo Hadiah Gratis' AND from:spammer@evil.com
=== 2026-05-06 10:00:06 END bulk remove ===
```

---

## Persyaratan Sistem

- **OS:** Linux (dengan Zimbra terinstal)
- **Shell:** Bash
- **Hak akses:** Harus dijalankan sebagai `root` (script melakukan `su - zimbra`)
- **Zimbra:** `zmmailbox` harus tersedia dan dapat diakses oleh user `zimbra`

---

## Catatan Penting

1. **Gunakan dry-run (`-n`) terlebih dahulu** sebelum eksekusi nyata untuk memverifikasi daftar email yang akan dihapus.
2. **Penghapusan bersifat permanen** — email yang dihapus via `zmmailbox dm` tidak masuk ke Trash.
3. **Tanpa subject = hapus semua** — jika subject tidak diberikan, seluruh email dari pengirim tersebut di mailbox target akan dihapus. Gunakan dengan hati-hati.
4. **Keyword subject bersifat contain** — script mencari email yang *mengandung* kata/kalimat tersebut di subject, bukan exact match.
5. **`set +H`** diaktifkan untuk menonaktifkan history expansion bash, mencegah error jika subject mengandung karakter `!`.

---

## Riwayat Perubahan

| Versi | Tanggal    | Perubahan                                                 |
| ----- | ---------- | --------------------------------------------------------- |
| v1.0  | 2026-05-06 | Rilis awal — FROM & SUBJECT dari argumen CLI, CSV 1 kolom |
| v1.1  | 2026-05-06 | Subject menjadi opsional — deteksi otomatis argumen ke-2  |

---

## Source Code

```bash
#!/bin/bash
# bulk-remove-spam.sh
# Bulk remove email Zimbra berdasarkan pengirim spam + (opsional) keyword subject + daftar penerima (CSV)
#
# Pemakaian:
#   ./bulk-remove-spam.sh [-n] "email-pengirim-spam" "kata/kalimat subject" daftar-penerima.csv
#   ./bulk-remove-spam.sh [-n] "email-pengirim-spam" daftar-penerima.csv

set +H   # disable history expansion

DRYRUN=false
if [ "$1" == "-n" ]; then
  DRYRUN=true
  shift
fi

FROM="$1"
LOGFILE="/var/log/bulk-remove-mail.log"

# --- Deteksi apakah argumen ke-2 adalah CSV atau subject ---
# Jika argumen ke-2 adalah file yang ada → tidak ada subject
if [ -f "$2" ]; then
  SUBJECT_KEYWORD=""
  CSV="$2"
else
  SUBJECT_KEYWORD="$2"
  CSV="$3"
fi

# --- Validasi argumen ---
if [ -z "$FROM" ] || [ -z "$CSV" ] || [ ! -f "$CSV" ]; then
  echo "Pemakaian:"
  echo "  $0 [-n] \"email-pengirim-spam\" \"kata/kalimat subject\" daftar-penerima.csv"
  echo "  $0 [-n] \"email-pengirim-spam\" daftar-penerima.csv"
  echo ""
  echo "  -n  : dry-run (tidak benar-benar hapus)"
  exit 1
fi

# --- Bangun filter query ---
if [ -n "$SUBJECT_KEYWORD" ]; then
  FILTER="subject:'$SUBJECT_KEYWORD' AND from:$FROM"
else
  FILTER="from:$FROM"
fi

echo "=== $(date '+%F %T') START bulk remove ===" | tee -a "$LOGFILE"
echo "FROM   : $FROM" | tee -a "$LOGFILE"
echo "SUBJECT: ${SUBJECT_KEYWORD:-'(semua subject)'}" | tee -a "$LOGFILE"
echo "CSV    : $CSV | Dry-run: $DRYRUN" | tee -a "$LOGFILE"
echo "FILTER : $FILTER" | tee -a "$LOGFILE"

# --- Baca CSV (skip header, ambil kolom pertama) ---
tail -n +2 "$CSV" | while IFS=',' read -r TO _REST; do
  # Trim whitespace & carriage return
  TO=$(echo "$TO" | sed 's/^ *//;s/ *$//;s/\r//')

  # Skip baris kosong
  [ -z "$TO" ] && continue

  echo ""
  echo ">> Target mailbox : $TO"
  echo "   From           : $FROM"
  echo "   Subject        : ${SUBJECT_KEYWORD:-'(semua subject)'}"

  MSGIDS=$(su - zimbra -c \
    "zmmailbox -z -m '$TO' s -t message \"$FILTER\"" \
    2>/dev/null | awk 'NR>4 && $2 ~ /^[0-9]+$/ {print $2}')

  if [ -z "$MSGIDS" ]; then
    echo "   Tidak ada email cocok"
    echo "$(date '+%F %T') $TO | NONE | $FILTER" >> "$LOGFILE"
    continue
  fi

  for ID in $MSGIDS; do
    if $DRYRUN; then
      echo "   [DRY-RUN] Akan hapus msgID $ID"
      echo "$(date '+%F %T') DRYRUN $TO | ID $ID | $FILTER" >> "$LOGFILE"
    else
      echo "   Hapus msgID $ID"
      su - zimbra -c "zmmailbox -z -m '$TO' dm $ID"
      if [ $? -eq 0 ]; then
        echo "$(date '+%F %T') OK    $TO | ID $ID | $FILTER" >> "$LOGFILE"
      else
        echo "$(date '+%F %T') ERROR $TO | ID $ID | $FILTER" >> "$LOGFILE"
      fi
    fi
  done
done

echo ""
echo "=== $(date '+%F %T') END bulk remove ===" | tee -a "$LOGFILE"
```
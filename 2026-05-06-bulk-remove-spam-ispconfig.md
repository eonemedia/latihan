---
Tanggal: 2026-05-06
Topik: Script Bulk Remove Spam Email ISPConfig via CLI
Kategori: Teknologi
Tag: [ispconfig, bash, spam, email-admin, dovecot, maildir, shell-script]
Model: Claude Sonnet 4.6

---

# bulk-remove-spam-ispconfig.sh — Dokumentasi

## Deskripsi

Script bash untuk menghapus email spam secara massal dari mailbox pengguna ISPConfig (Postfix + Dovecot + Maildir). Pengirim spam diberikan sebagai argumen wajib, keyword subject bersifat **opsional**, dan daftar penerima (target mailbox) dibaca dari file CSV.

Jika subject tidak diberikan, script akan menghapus **semua email dari pengirim tersebut** tanpa memfilter subject.

Berbeda dengan versi Zimbra yang menggunakan `zmmailbox`, script ini bekerja langsung pada file Maildir menggunakan `grep` dan `rm`, sehingga lebih reliable dan tidak bergantung pada indeks email yang mungkin belum ter-update.

---

## Pemakaian

```bash
# Dengan filter subject
./bulk-remove-spam-ispconfig.sh [-n] "email-pengirim-spam" "kata/kalimat subject" daftar-penerima.csv

# Tanpa filter subject (hapus semua email dari pengirim)
./bulk-remove-spam-ispconfig.sh [-n] "email-pengirim-spam" daftar-penerima.csv
```

### Argumen

| Posisi     | Argumen                | Wajib  | Keterangan                                                                      |
| ---------- | ---------------------- | ------ | ------------------------------------------------------------------------------- |
| (opsional) | `-n`                   | Tidak  | Dry-run — simulasi tanpa benar-benar menghapus email                            |
| 1          | `email-pengirim-spam`  | **Ya** | Alamat email pengirim spam (dicocokkan dengan header `From:`)                   |
| 2          | `kata/kalimat subject` | Tidak  | Keyword yang dicari di subject (contain). Jika dilewati, semua subject diproses |
| 2 atau 3   | `daftar-penerima.csv`  | **Ya** | File CSV berisi daftar email penerima target                                    |

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
./bulk-remove-spam-ispconfig.sh -n "spammer@evil.com" "Promo Hadiah Gratis" daftar-penerima.csv
```

### 2. Eksekusi dengan filter subject

```bash
./bulk-remove-spam-ispconfig.sh "spammer@evil.com" "Promo Hadiah Gratis" daftar-penerima.csv
```

### 3. Dry-run tanpa filter subject (semua email dari pengirim)

```bash
./bulk-remove-spam-ispconfig.sh -n "spammer@evil.com" daftar-penerima.csv
```

### 4. Eksekusi tanpa filter subject

```bash
./bulk-remove-spam-ispconfig.sh "spammer@evil.com" daftar-penerima.csv
```

---

## Penanganan Subject Ter-encode (MIME Encoded-Word)

Email spam sering menggunakan subject yang di-encode dalam format **MIME encoded-word** agar lolos filter:

```
Subject: =?UTF-8?B?8J+TqVN5c3RlbSBNYWludGVuYW5jZTog...?=
```

Script secara otomatis men-decode subject sebelum dicocokkan dengan keyword, sehingga kamu tetap bisa menggunakan teks biasa sebagai filter:

```bash
# Keyword dalam teks asli — script akan decode subject secara otomatis
./bulk-remove-spam-ispconfig.sh -n "spammer@evil.com" "System Maintenance" daftar-penerima.csv
./bulk-remove-spam-ispconfig.sh -n "spammer@evil.com" "Authentication" daftar-penerima.csv
```

| Format Encoding | Contoh                          | Keterangan              |
| --------------- | ------------------------------- | ----------------------- |
| Base64          | `=?UTF-8?B?...?=`               | Paling umum pada spam   |
| Quoted-Printable| `=?UTF-8?Q?Sistem_=F0=9F=93=A9?=` | Kadang dipakai juga   |

Subject yang ditampilkan di output terminal dan log juga sudah dalam bentuk teks asli (sudah di-decode).

---

## Logika Pencarian Email

Script tidak menggunakan tool pencarian Dovecot (`doveadm search`), melainkan **grep langsung ke file Maildir**. Pendekatan ini lebih reliable karena tidak bergantung pada indeks Dovecot yang mungkin belum ter-update.

| Kondisi        | Metode pencarian                                                          |
| -------------- | ------------------------------------------------------------------------- |
| Dengan subject | `grep From: <file>` cocok **DAN** decode subject lalu cocokkan keyword    |
| Tanpa subject  | `grep From: <file>` cocok saja                                            |

Pencarian mencakup **semua folder** di Maildir secara rekursif, termasuk `INBOX`, `.Junk`, `.Spam`, `.Sent`, dan subfolder lainnya.

---

## Mapping Email ke Folder Maildir

ISPConfig menyimpan mailbox di:

```
/var/vmail/<domain>/<username>/Maildir/
```

Username di filesystem bisa berbeda dari bagian lokal alamat email:

| Email di CSV               | Kemungkinan nama folder |
| -------------------------- | ----------------------- |
| `firstname.lastname@domain.com` | `firstname_lastname` |
| `firstname_lastname@domain.com` | `firstname.lastname` |
| `firstname.lastname@domain.com` | `firstname.lastname` |

Script secara otomatis mencoba **tiga kandidat** folder (nama asli, versi `_`, versi `.`) sehingga mapping berjalan tanpa konfigurasi manual.

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
Loop setiap baris CSV (email penerima / TO)
        │
        ▼
Cari path Maildir (coba 3 kandidat username)
  ├─ Tidak ditemukan → log SKIP, lanjut ke penerima berikutnya
  └─ Ditemukan
        │
        ▼
find semua file di */cur/* dan */new/* (rekursif)
        │
        ▼
Per file: grep header From: → cocok?
  └─ Ya → ada SUBJECT_KEYWORD?
            ├─ Tidak → file cocok ✓
            └─ Ya → ambil header Subject:
                      → decode MIME encoded-word (python3)
                      → grep keyword → cocok?
                          ├─ Ya → file cocok ✓
                          └─ Tidak → lewati file
        │
        ▼
Kumpulkan semua file cocok
  ├─ Kosong → log NONE, lanjut ke penerima berikutnya
  └─ Ada hasil
        │
        ├─ [DRY-RUN] → tampilkan info From/Subject/File, log DRYRUN
        │
        └─ [NORMAL]  → rm -f file
                        ├─ Berhasil → log OK
                        └─ Gagal    → log ERROR
        │
        ▼
doveadm force-resync (rebuild index Dovecot)
```

---

## Log File

Semua aktivitas dicatat di:

```
/var/log/bulk-remove-mail-ispconfig.log
```

### Format entri log

| Status                | Format                                                              |
| --------------------- | ------------------------------------------------------------------- |
| Maildir tidak ada     | `YYYY-MM-DD HH:MM:SS SKIP  <TO> \| Maildir tidak ditemukan`         |
| Tidak ada email cocok | `YYYY-MM-DD HH:MM:SS NONE  <TO> \| FROM=<FROM> \| SUBJECT=<filter>` |
| Dry-run               | `YYYY-MM-DD HH:MM:SS DRYRUN <TO> \| <folder> \| <From> \| <Subject>` |
| Berhasil dihapus      | `YYYY-MM-DD HH:MM:SS OK    <TO> \| <folder> \| <From> \| <Subject>` |
| Gagal dihapus         | `YYYY-MM-DD HH:MM:SS ERROR <TO> \| <folder> \| <From> \| <Subject>` |

### Contoh isi log

```
=== 2026-05-06 10:00:00 START bulk remove ISPConfig v4 ===
FROM   : spammer@evil.com
SUBJECT: System Maintenance
CSV    : daftar-penerima.csv | Dry-run: false
2026-05-06 10:00:01 NONE  user1@domain.com | FROM=spammer@evil.com | SUBJECT=System Maintenance
2026-05-06 10:00:03 OK    user2@domain.com | INBOX | "Spammer" <spammer@evil.com> | 📩System Maintenance:***-Access Authentication
2026-05-06 10:00:04 OK    user2@domain.com | .Junk | "Spammer" <spammer@evil.com> | 📩System Maintenance:***-Access Authentication
2026-05-06 10:00:05 ERROR user3@domain.com | INBOX | "Spammer" <spammer@evil.com> | 📩System Maintenance:***-Access Authentication
=== 2026-05-06 10:00:06 END bulk remove ISPConfig v4 ===
```

> **Catatan:** Subject di log sudah ditampilkan dalam teks asli (sudah di-decode dari Base64/Quoted-Printable).

---

## Persyaratan Sistem

- **OS:** Linux (dengan ISPConfig terinstall)
- **Shell:** Bash
- **Hak akses:** Harus dijalankan sebagai `root`
- **Python:** `python3` harus tersedia (untuk decode MIME encoded-word subject)
- **Dovecot:** `doveadm` harus tersedia (untuk rebuild index setelah penghapusan)
- **Maildir:** Lokasi default `/var/vmail/<domain>/<username>/Maildir/`

---

## Catatan Penting

1. **Gunakan dry-run (`-n`) terlebih dahulu** sebelum eksekusi nyata untuk memverifikasi daftar email yang akan dihapus.
2. **Penghapusan bersifat permanen** — email dihapus langsung dari filesystem (`rm -f`) dan tidak masuk ke Trash.
3. **Tanpa subject = hapus semua** — jika subject tidak diberikan, seluruh email dari pengirim tersebut di mailbox target akan dihapus. Gunakan dengan hati-hati.
4. **Keyword subject bersifat contain** — script mencari email yang *mengandung* kata/kalimat tersebut di subject, bukan exact match, dan bersifat case-insensitive.
5. **Subject ter-encode ditangani otomatis** — script men-decode MIME encoded-word (`=?UTF-8?B?...?=`) sebelum dicocokkan, sehingga keyword selalu ditulis dalam teks biasa.
6. **Semua folder diperiksa** — pencarian tidak terbatas pada INBOX, tetapi mencakup semua subfolder Maildir (`.Junk`, `.Spam`, `.Sent`, dll).
7. **Index Dovecot diperbarui otomatis** — setelah penghapusan, script menjalankan `doveadm force-resync` agar tampilan webmail (Roundcube, dll) sinkron dengan kondisi aktual di filesystem.
8. **`set +H`** diaktifkan untuk menonaktifkan history expansion bash, mencegah error jika subject mengandung karakter `!`.

---

## Riwayat Perubahan

| Versi | Tanggal    | Perubahan                                                                          |
| ----- | ---------- | ---------------------------------------------------------------------------------- |
| v1.0  | 2026-05-06 | Rilis awal — pencarian via `doveadm search`, hapus via `doveadm expunge`           |
| v2.0  | 2026-05-06 | Pencarian via grep langsung ke Maildir, hapus via `rm -f`, cari semua subfolder    |
| v3.0  | 2026-05-06 | Fix mapping username: support titik (`.`) dan underscore (`_`) sebagai separator  |
| v4.0  | 2026-05-06 | Decode MIME encoded-word subject sebelum dicocokkan; subject di log tampil decoded |

---

## Source Code

```bash
#!/bin/bash
# bulk-remove-spam-ispconfig.sh v4.1
# Bulk remove email spam di ISPConfig (Postfix + Dovecot + Maildir)
#
# Pemakaian:
#   ./bulk-remove-spam-ispconfig.sh [-n] "email-pengirim-spam" ["keyword subject"] daftar-penerima.csv
#
# Fix v4.1:
#   - Perbaikan pengambilan Subject multiline/folded (RFC 2822)
#     grep -m1 diganti dengan extract_raw_subject() yang membaca
#     semua continuation line sebelum di-decode — subject tidak terpotong lagi

set +H  # nonaktifkan history expansion

# ─── Warna ──────────────────────────────────────────────────────────────────
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
CYAN='\033[0;36m'
GRAY='\033[0;90m'
NC='\033[0m'

# ─── Konfigurasi ────────────────────────────────────────────────────────────
LOGFILE="/var/log/bulk-remove-mail-ispconfig.log"
VMAIL_BASE="/var/vmail"

# ─── Cek root ───────────────────────────────────────────────────────────────
if [ "$(id -u)" -ne 0 ]; then
  echo -e "${RED}[ERROR]${NC} Script harus dijalankan sebagai root."
  exit 1
fi

# ─── Dry-run flag ───────────────────────────────────────────────────────────
DRYRUN=false
if [ "$1" == "-n" ]; then
  DRYRUN=true
  shift
fi

FROM="$1"

# ─── Deteksi argumen ke-2: CSV atau keyword subject ─────────────────────────
if [ -f "$2" ]; then
  SUBJECT_KEYWORD=""
  CSV="$2"
else
  SUBJECT_KEYWORD="$2"
  CSV="$3"
fi

# ─── Validasi ───────────────────────────────────────────────────────────────
if [ -z "$FROM" ] || [ -z "$CSV" ] || [ ! -f "$CSV" ]; then
  echo ""
  echo "Pemakaian:"
  echo "  $0 [-n] \"email-pengirim\" [\"keyword subject\"] daftar-penerima.csv"
  echo ""
  echo "  -n  : dry-run (tidak benar-benar hapus)"
  echo ""
  echo "Contoh:"
  echo "  $0 -n \"spammer@evil.com\" \"Promo Gratis\" penerima.csv"
  echo "  $0 -n \"spammer@evil.com\" penerima.csv"
  echo "  $0 \"spammer@evil.com\" \"Promo Gratis\" penerima.csv"
  exit 1
fi

# ─── Logging ────────────────────────────────────────────────────────────────
log() {
  echo "$(date '+%F %T') $*" >> "$LOGFILE"
}

# ─── Ambil header Subject multiline dari file email ─────────────────────────
# Subject bisa terlipat (folded) ke beberapa baris (RFC 2822):
#   Subject: =?UTF-8?B?...bagian1...?=
#    =?UTF-8?B?...bagian2...?=        ← diawali spasi = continuation line
#    =?UTF-8?B?...bagian3...?=
# grep -m1 hanya ambil baris pertama sehingga decode terpotong.
# Fungsi ini membaca semua baris hingga header Subject lengkap.
extract_raw_subject() {
  local filepath="$1"
  local in_subject=0
  local raw=""
  while IFS= read -r line; do
    # Baris kosong = awal body email, hentikan pembacaan header
    if [ -z "$line" ] || [ "$line" = $'\r' ]; then
      break
    fi
    if echo "$line" | grep -qi "^Subject:"; then
      in_subject=1
      raw="$line"
    elif [ "$in_subject" -eq 1 ] && echo "$line" | grep -q "^[[:space:]]"; then
      # Continuation line — gabungkan dengan spasi
      raw="$raw $line"
    elif [ "$in_subject" -eq 1 ]; then
      # Header baru ditemukan, Subject sudah selesai
      break
    fi
  done < "$filepath"
  # Buang prefix "Subject: " dan carriage return
  echo "$raw" | sed 's/^Subject: //i' | tr -d '\r'
}

# ─── Decode MIME encoded-word pada Subject ──────────────────────────────────
# Format: =?charset?B?base64?= atau =?charset?Q?quoted-printable?=
decode_subject() {
  local raw="$1"
  python3 -c "
import sys, email.header, re

raw = sys.argv[1]
# Normalkan spasi berlebih antar encoded-word
raw = re.sub(r'\s+', ' ', raw).strip()

try:
    parts = email.header.decode_header(raw)
    result = []
    for part, charset in parts:
        if isinstance(part, bytes):
            result.append(part.decode(charset or 'utf-8', errors='replace'))
        else:
            result.append(str(part))
    print(''.join(result))
except Exception:
    print(raw)
" "$raw" 2>/dev/null || echo "$raw"
}

# ─── Cari path Maildir dari alamat email ────────────────────────────────────
# ISPConfig menyimpan di /var/vmail/<domain>/<username>/Maildir/
# Username di filesystem bisa pakai _ meski email pakai . atau sebaliknya
find_maildir() {
  local email="$1"
  local domain="${email#*@}"
  local localpart="${email%@*}"

  local variant_underscore="${localpart//./_}"
  local variant_dot="${localpart//_/.}"

  local domain_path="$VMAIL_BASE/$domain"

  if [ ! -d "$domain_path" ]; then
    return 1
  fi

  for candidate in "$localpart" "$variant_underscore" "$variant_dot"; do
    local maildir="$domain_path/$candidate/Maildir"
    if [ -d "$maildir" ]; then
      echo "$maildir"
      return 0
    fi
  done

  return 1
}

# ─── Cari semua file email yang cocok di seluruh Maildir ────────────────────
search_emails() {
  local maildir="$1"
  local from_pat="$2"
  local subj_pat="$3"

  local matched=()

  while IFS= read -r filepath; do
    [ -f "$filepath" ] || continue

    # Cek header From:
    if ! grep -qi "^From:.*${from_pat}" "$filepath" 2>/dev/null; then
      continue
    fi

    # Jika ada filter subject: extract multiline → decode → cocokkan
    if [ -n "$subj_pat" ]; then
      local raw_subj decoded_subj
      raw_subj=$(extract_raw_subject "$filepath")
      decoded_subj=$(decode_subject "$raw_subj")

      if ! echo "$decoded_subj" | grep -qi "$subj_pat" 2>/dev/null; then
        continue
      fi
    fi

    matched+=("$filepath")
  done < <(find "$maildir" \( -path "*/cur/*" -o -path "*/new/*" \) -type f 2>/dev/null)

  printf '%s\n' "${matched[@]}"
}

# ─── Proses satu mailbox ─────────────────────────────────────────────────────
process_mailbox() {
  local TO="$1"
  local maildir

  maildir=$(find_maildir "$TO")
  if [ -z "$maildir" ]; then
    echo -e "   ${RED}[SKIP]${NC} Maildir tidak ditemukan"
    echo -e "   ${GRAY}Cek manual: ls $VMAIL_BASE/<domain>/${NC}"
    log "SKIP  $TO | Maildir tidak ditemukan"
    return
  fi

  echo -e "   ${GRAY}Maildir: $maildir${NC}"

  local matched_files=()
  while IFS= read -r f; do
    [ -n "$f" ] && matched_files+=("$f")
  done < <(search_emails "$maildir" "$FROM" "$SUBJECT_KEYWORD")

  local total="${#matched_files[@]}"

  if [ "$total" -eq 0 ]; then
    echo -e "   ${YELLOW}Tidak ada email cocok${NC}"
    log "NONE  $TO | FROM=$FROM | SUBJECT=${SUBJECT_KEYWORD:-*}"
    return
  fi

  echo -e "   Ditemukan: ${CYAN}$total email${NC}"

  local ok=0
  local err=0

  for filepath in "${matched_files[@]}"; do
    local fname from_hdr subj_hdr folder_name raw_subj
    fname=$(basename "$filepath")
    from_hdr=$(grep -im1 "^From:" "$filepath" | sed 's/^From: //i' | tr -d '\r')

    # Ambil subject lengkap (multiline) lalu decode
    raw_subj=$(extract_raw_subject "$filepath")
    subj_hdr=$(decode_subject "$raw_subj")

    # Nama folder
    local rel_path="${filepath#${maildir}/}"
    folder_name=$(echo "$rel_path" | cut -d'/' -f1)
    if [ "$folder_name" == "cur" ] || [ "$folder_name" == "new" ]; then
      folder_name="INBOX"
    fi

    if $DRYRUN; then
      echo -e ""
      echo -e "   ${YELLOW}[DRY-RUN]${NC} Folder : $folder_name"
      echo -e "            From    : $from_hdr"
      echo -e "            Subject : $subj_hdr"
      echo -e "            File    : $fname"
      log "DRYRUN $TO | $folder_name | $from_hdr | $subj_hdr"
    else
      if rm -f "$filepath" 2>/dev/null; then
        echo -e "   ${GREEN}[OK]${NC} [$folder_name] $subj_hdr"
        log "OK    $TO | $folder_name | $from_hdr | $subj_hdr"
        (( ok++ ))
      else
        echo -e "   ${RED}[ERROR]${NC} Gagal hapus: $fname"
        log "ERROR $TO | $folder_name | $from_hdr | $subj_hdr"
        (( err++ ))
      fi
    fi
  done

  # Rebuild Dovecot index agar webmail sinkron
  if ! $DRYRUN && [ "$ok" -gt 0 ]; then
    echo -e "   ${GRAY}Rebuild index Dovecot...${NC}"
    doveadm force-resync -u "$TO" '*' &>/dev/null \
      || doveadm index -u "$TO" '*' &>/dev/null \
      || true
    echo -e "   ─────────────────────────────────────"
    echo -e "   ${GREEN}Berhasil dihapus : $ok${NC}"
    [ "$err" -gt 0 ] && echo -e "   ${RED}Gagal            : $err${NC}"
  fi
}

# ─── Header ─────────────────────────────────────────────────────────────────
{
  echo "=== $(date '+%F %T') START bulk remove ISPConfig v4.1 ==="
  echo "FROM   : $FROM"
  echo "SUBJECT: ${SUBJECT_KEYWORD:-'(semua subject)'}"
  echo "CSV    : $CSV | Dry-run: $DRYRUN"
} | tee -a "$LOGFILE"

echo ""
if $DRYRUN; then
  echo -e "${YELLOW}══ MODE: DRY-RUN — tidak ada yang dihapus ══${NC}"
else
  echo -e "${RED}══ MODE: EKSEKUSI — email dihapus permanen! ══${NC}"
fi
echo ""

# ─── Loop CSV ───────────────────────────────────────────────────────────────
tail -n +2 "$CSV" | while IFS=',' read -r TO _REST; do
  TO=$(echo "$TO" | sed 's/^ *//;s/ *$//;s/\r//')
  [ -z "$TO" ] && continue

  echo "──────────────────────────────────────────────"
  echo -e ">> ${CYAN}$TO${NC}"

  process_mailbox "$TO"
  echo ""
done

# ─── Footer ─────────────────────────────────────────────────────────────────
{
  echo "=== $(date '+%F %T') END bulk remove ISPConfig v4.1 ==="
} | tee -a "$LOGFILE"

echo -e "Log: ${CYAN}$LOGFILE${NC}"
```

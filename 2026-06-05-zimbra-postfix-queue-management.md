---
Tanggal: 2026-06-05
Topik: Zimbra Postfix Queue Management - Bounce & Queue Lifetime
Kategori: Teknologi
Tag: [zimbra, postfix, email, queue, bounce, over-quota, lmtp, mail-server]
Model: Claude Sonnet 4.6
---

# Zimbra: Manajemen Queue & Auto-Bounce Email

Panduan ini mencakup penanganan email yang tertahan di queue Postfix pada Zimbra 8.8.15, khususnya kasus **over-quota (452 4.2.2)** beserta konfigurasi auto-bounce agar pengirim mendapat notifikasi dalam waktu wajar.

---

## 1. Latar Belakang

Secara default, Postfix menyimpan email yang gagal terkirim di queue selama **5 hari** sebelum mengirim bounce ke pengirim. Untuk kasus over-quota, ini terlalu lama — pengirim baru tahu emailnya gagal setelah hampir seminggu.

### Contoh Log Over-Quota

```
Jun  4 21:05:27 mail postfix/lmtp[221571]: F23FB2863E55: to=<dwinovianto@domainku.co.id>,
  relay=mail.domainku.co.id[192.168.3.3]:7025, delay=105454,
  dsn=4.2.2, status=deferred (host mail.domainku.co.id said: 452 4.2.2 Over quota)
```

- **DSN 4.2.2** = soft error (mailbox penuh), Postfix akan retry otomatis
- **Status deferred** = email menunggu antrian retry berikutnya, **bukan hold**
- Email retry tiap ~70 menit sampai `maximal_queue_lifetime` tercapai

---

## 2. Status Queue Postfix

| Status | Keterangan |
|---|---|
| **active** | Sedang aktif dikirim saat ini |
| **deferred** | Gagal, menunggu retry otomatis berikutnya |
| **hold** | Ditahan manual/policy, **tidak retry otomatis** |
| **incoming** | Baru masuk, belum diproses |
| **corrupt** | Rusak, tidak bisa diproses |

> **Tanda `!`** di depan Queue ID pada output `mailq` menandakan status **hold**.  
> Tanpa `!` = deferred/normal, akan retry otomatis.

```
# Contoh output mailq - status HOLD (ada tanda !)
!A1B2C3D4E    1234   Thu Jun  5 08:00:00  pengirim@domain.com
                                           penerima@domain.com

# Contoh output mailq - status DEFERRED (tanpa tanda !)
A1B2C3D4E     1234   Thu Jun  5 08:00:00  pengirim@domain.com
                                           penerima@domain.com
```

---

## 3. Parameter Queue Lifetime

### `maximal_queue_lifetime`
Berapa lama Postfix menyimpan email **asli** di queue sebelum menyerah dan bounce ke pengirim.

```
Pengirim → [Postfix Queue] → retry... retry... retry...
                                  ↓ (setelah maximal_queue_lifetime)
                             BOUNCE ke pengirim
```

### `bounce_queue_lifetime`
Setelah bounce digenerate, pesan bounce itu sendiri juga masuk queue. Parameter ini mengatur berapa lama **pesan bounce** disimpan jika gagal terkirim ke pengirim asli.

```
[Bounce notification] → [Postfix Queue] → retry...
                                ↓ (setelah bounce_queue_lifetime)
                          Bounce dibuang (double-bounce)
```

### Perbandingan

| Parameter | Objek yang dikontrol | Default |
|---|---|---|
| `maximal_queue_lifetime` | Email asli di queue | 5d |
| `bounce_queue_lifetime` | Pesan bounce itu sendiri | 5d |

---

## 4. Cek Konfigurasi Saat Ini

```bash
su - zimbra

# Cek via zmprov (cara yang benar di Zimbra)
zmprov gcf zimbraMtaMaximalQueueLifetime
zmprov gcf zimbraMtaBounceQueueLifetime

# Verifikasi langsung ke Postfix
postconf maximal_queue_lifetime
postconf bounce_queue_lifetime
```

> **Catatan:** Jangan gunakan `zmlocalconfig -e postfix_maximal_queue_lifetime` untuk membaca nilai — gunakan tanpa flag `-e`. Namun untuk setting MTA di Zimbra, `zmprov` adalah cara yang tepat.

---

## 5. Konfigurasi Auto-Bounce

### Rekomendasi untuk Kasus Over-Quota

```bash
su - zimbra

# Set bounce setelah 1 hari (lebih wajar dari default 5 hari)
zmprov mcf zimbraMtaMaximalQueueLifetime 1d
zmprov mcf zimbraMtaBounceQueueLifetime 1d

# Apply ke Postfix
postfix reload

# Verifikasi
postconf maximal_queue_lifetime
postconf bounce_queue_lifetime
```

### Opsi Nilai yang Umum Digunakan

| Nilai | Keterangan |
|---|---|
| `5d` | Default Postfix — terlalu lama untuk over-quota |
| `1d` | Rekomendasi umum — pengirim tahu dalam 1 hari |
| `6h` | Agresif — cocok untuk lingkungan internal |
| `4h` | Sangat agresif — pengirim cepat tahu, tapi sedikit ruang retry |

> **Pertimbangan:** Nilai terlalu kecil berisiko bounce prematur jika gangguan bersifat sementara (misal server restart singkat). Nilai `1d` umumnya cukup seimbang.

---

## 6. Manajemen Queue Manual

### Lihat Queue

```bash
# Tampilkan semua queue
mailq

# Atau
postqueue -p

# Filter hanya yang HOLD (ada tanda !)
postqueue -p | grep "^!"

# Cek detail satu message ID
postcat -q F23FB2863E55
```

### Hapus Email dari Queue

```bash
# Hapus satu message ID spesifik
postsuper -d F23FB2863E55

# Hapus SEMUA yang deferred (hati-hati!)
postsuper -d ALL deferred

# Hapus SEMUA queue (sangat hati-hati!)
postsuper -d ALL
```

### Lepas dari Hold

```bash
# Lepas satu message dari hold ke queue aktif
postsuper -H F23FB2863E55

# Lepas semua yang hold
postsuper -H ALL
```

### Paksa Retry Sekarang

```bash
# Flush semua deferred untuk retry langsung
postqueue -f

# Atau
postfix flush
```

---

## 7. Solusi Root Cause: Mailbox Over-Quota

Konfigurasi di atas hanya mempercepat notifikasi bounce ke pengirim. Solusi sesungguhnya adalah menangani mailbox yang penuh:

```bash
su - zimbra

# Cek quota user
zmprov ga dwinovianto@domainku.co.id | grep -i quota

# Tambah quota (contoh: set ke 2GB = 2147483648 bytes)
zmprov ma dwinovianto@domainku.co.id zimbraMailQuota 2147483648

# Set unlimited quota (tidak disarankan untuk produksi)
zmprov ma dwinovianto@domainku.co.id zimbraMailQuota 0
```

Alternatif lain: minta user mengosongkan folder **Trash** dan **Junk** via Zimbra Webmail, atau admin purge via CLI.

---

## 8. Monitoring

```bash
# Monitor log real-time
tail -f /var/log/zimbra.log | grep -E "deferred|bounced|expired"

# Hitung jumlah email deferred saat ini
mailq | grep -c "^[A-F0-9]"

# Ringkasan queue
postqueue -p | tail -1
```

---

## Referensi

- Zimbra Wiki: MTA Configuration
- Postfix Docs: `postconf(5)` — `maximal_queue_lifetime`, `bounce_queue_lifetime`
- `man postsuper` — queue management tool

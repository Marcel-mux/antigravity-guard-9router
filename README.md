# Antigravity Guard untuk 9Router

Auto-heal guard buat akun OAuth **Antigravity** di **9Router**. Bikin akun Google kamu pulih otomatis kalau kena error jaringan, token basi, atau model lock yang nyangkut.

```
Tanpa guard:  jaringan blip → refresh token gagal → akun di-set mati → HILANG sampai dinyalain manual
Dengan guard: jaringan blip → guard cek tiap 5 menit → token di-refresh → akun pulih otomatis
```

---

## Sebelum Instalasi — Checklist Wajib

Baca ini dulu. Kalau ada satu aja yang belum, guard-nya gak bakal jalan.

| # | Prasyarat | Cara Cek | Kalau Belum Ada |
|---|---|---|---|
| 1 | **9Router udah jalan** | `systemctl status 9router` atau cek dashboard | Ini buat ngerawat 9Router, bukan bikin. Install 9Router dulu |
| 2 | **Akun Antigravity udah di-connect OAuth** | 9Router Dashboard → Providers → Antigravity, ada akunnya | Connect akun lewat dashboard dulu |
| 3 | **Ada AI agent (Hermes / Claude Code / dll)** | — | Dokumen ini ditulis sebagai *spec* buat dieksekusi AI agent. Tanpa itu, kamu harus nulis scriptnya manual |
| 4 | **Akses root/sudo ke VPS** | `whoami` → `root` | Guard perlu tulis file, chmod, cron, restart service |
| 5 | **Python 3** | `python3 --version` | Hampir semua Linux udah ada |
| 6 | **Tahu cara restart 9Router kamu** | systemd / docker / pm2? | Wajib — guard bakal restart 9Router tiap kali pulihin akun |

### Yang Perlu Diperhatikan Sebelum Mulai

**1. Guard ini GAK bikin akun kamu aman dari ban Google.**
Ini poin paling sering disalahpahami. Guard cuma benerin **kerusakan teknis** (jaringan, token basi, lock). Kalau kamu nembak dari IP datacenter VPS dengan volume gede, Google tetep bisa nge-razia. Guard gak ngejauh-ngejauhkan kamu dari itu.

**2. Restart 9Router = koneksi putus sebentar.**
Tiap kali guard berhasil pulihin akun, dia restart 9Router. Kalau 9Router kamu dipakai buat **produksi** (bot WhatsApp, service lain), request yang lagi jalan bisa terputus. Pertimbangkan dulu.

**3. Butuh 2 kredensial OAuth (`client_id` + `client_secret`).**
Ini identitas aplikasi Antigravity, bukan password pribadimu. Kalau gak tau nilainya, lihat bagian [Cara Dapetin Kredensial](#cara-dapetin-kredensial).

**4. Guard cuma nyentuh akun yang AKTIF.**
Akun yang kamu matiin sendiri (`isActive=0`) bakal dilewatin total. Ini disengaja — biar guard gak ngehidupin paksa akun yang sengaja kamu off.

**5. Rawan rusak kalau 9Router update.**
Script baca langsung dari struktur DB 9Router. Kalau versi baru ganti struktur tabel, script harus disesuaikan.

**6. Gak ada garansi.**
Ini tool komunitas buat setup spesifik, bukan produk resmi. Test dulu di environment yang aman.

---

## Apa yang Guard Ini Lakuin

| Item | Detail |
|---|---|
| **Nama script** | `antigravity_guard.py` |
| **Jadwal** | Cron, tiap **5 menit** |
| **Tugas** | "Dokter jaga" buat akun Antigravity di 9Router |
| **Output** | Cuma lapor kalau ada yang dipulihkan (kalau semua sehat → diam total) |

### Penyakit yang Bisa / Gak Bisa Disembuhin

| Penyakit | Guard Bisa? |
|---|---|
| 401 + `fetch failed` (jaringan VPS blip) | ✅ Ya |
| AccessToken expired (~1 jam umurnya) | ✅ Ya |
| `modelLock` nyangkut | ✅ Ya |
| Akun ke-set `isActive=0` akibat error | ✅ Ya |
| `invalid_grant` (refreshToken dicabut) | ❌ Wajib login ulang |
| 403 `VALIDATION_REQUIRED` (akun belum verif) | ❌ Wajib verifikasi manual |
| 429 rate limit (kuota habis) | ❌ Tunggu reset (~4-5 jam) |

---

## Instalasi

### Langkah 0 — Kirim Spec Ini ke AI Agent Kamu

Cara paling gampang: kirim link repo ini ke Hermes / AI agent kamu, terus bilang:

> *"Baca dokumen di link ini. Cek environment 9Router gue dulu, terus bikin script `antigravity_guard.py` sesuai spec, pasang cron tiap 5 menit."*

AI agent bakal otomatis ngurus: cek 9Router, cari kredensial, tulis script, tes manual, pasang cron.

### Langkah 1 — Cari Kredensial OAuth

Guard butuh 2 nilai: `CLIENT_ID` dan `CLIENT_SECRET`. Ini **identitas aplikasi Antigravity**, bukan password pribadi kamu.

Cara cari (urut dari paling gampang):

**a. Dari DB 9Router**
```bash
python3 -c "
import sqlite3
con = sqlite3.connect('file:/root/.9router/db/data.sqlite?mode=ro', uri=True)
cur = con.cursor()
cur.execute(\"SELECT name FROM sqlite_master WHERE type='table'\")
print([r[0] for r in cur.fetchall()])
"
```
Cek tabel config/settings, kemungkinan tersimpan di sana.

**b. Dari folder installasi 9Router**
```bash
grep -rn "client_id\|client_secret" /usr/lib/node_modules/9router/app/ 2>/dev/null | head
```

**c. Dari bundle node_modules**
```bash
grep -roh "GOCSPX-[A-Za-z0-9_-]*" /usr/lib/node_modules/9router/ 2>/dev/null | head
```

**d. Kalau semua gagal**
Minta yang setup 9Router kamu. Kalau tetap gak ketemu — **jangan karang nilai**. Guard bakal gagal refresh token kalau kredensialnya salah.

### Langkah 2 — Tulis Script

Simpan sebagai `antigravity_guard.py`. Kode lengkap ada di file [`antigravity-guard-9router.md`](antigravity-guard-9router.md) bagian 6.

**4 baris yang wajib kamu ganti:**

```python
DB_PATH = "/root/.9router/db/data.sqlite"           # lokasi data.sqlite 9Router kamu
STATE_FILE = "/root/.hermes/cache/antigravity_guard.state"
RESTART_CMD = ["systemctl", "restart", "9router.service"]  # docker: ["docker","restart","nama"] / pm2: ["pm2","restart","nama"]
CLIENT_ID = "GANTI_DENGAN_CLIENT_ID_ANTIGRAVITY"     # lihat Langkah 1
CLIENT_SECRET = "GANTI_DENGAN_CLIENT_SECRET_ANTIGRAVITY"
```

### Langkah 3 — Tes Manual (WAJIB, jangan skip)

```bash
chmod +x antigravity_guard.py
python3 antigravity_guard.py
```

| Output | Artinya |
|---|---|
| **Diam total** (gak ada output) | ✅ Semua akun sehat — ini hasil yang bener |
| `ModuleNotFoundError` | ❌ Ada dependency kurang |
| Traceback lain | ❌ Ada bug — jangan lanjut ke cron |
| `🛡️ Berhasil auto-heal...` | ✅ Guard kerja, ada akun yang dipulihkan |

**Jangan pasang cron sebelum tes manual lolos.**

### Langkah 4 — Pasang Cron

```bash
crontab -e
```

Tambahkan:
```
*/5 * * * * /usr/bin/python3 /path/to/antigravity_guard.py >> /var/log/antigravity_guard.log 2>&1
```

Verifikasi:
```bash
crontab -l | grep antigravity
tail -f /var/log/antigravity_guard.log
```

---

## Cara Verifikasi Guard-nya Jalan

### 1. Cek cron terpasang
```bash
crontab -l
```

### 2. Cek log (5 menit setelah pasang)
```bash
tail -20 /var/log/antigravity_guard.log
```
Log kosong = semua akun sehat. Itu normal.

### 3. Tes paksa: rusak satu akun, liat guard benerin

```bash
# Backup DB dulu
cp /root/.9router/db/data.sqlite /root/.9router/db/data.sqlite.bak

# Tanam error dummy di salah satu akun aktif
python3 -c "
import sqlite3, json
con = sqlite3.connect('/root/.9router/db/data.sqlite')
cur = con.cursor()
cur.execute(\"SELECT id, data FROM providerConnections WHERE provider='antigravity' AND isActive=1\")
row = cur.fetchone()
d = json.loads(row[1])
d['lastError'] = 'DUMMY TEST ERROR'
d['errorCode'] = 999
cur.execute('UPDATE providerConnections SET data=? WHERE id=?', (json.dumps(d), row[0]))
con.commit()
print('Error dummy ditanam di', row[0])
"

# Jalanin guard, tunggu hasilnya
python3 antigravity_guard.py
```

**Hasil yang diharapkan** (dalam ~2 menit):
```
🛡️ [Antigravity Guard] Berhasil auto-heal & aktifkan akun: akun@email.com. Me-restart 9router...
```

Kalau muncul itu — guard kamu jalan sempurna.

### 4. Cek akun beneran kepulihkan di DB
```bash
python3 -c "
import sqlite3, json
con = sqlite3.connect('file:/root/.9router/db/data.sqlite?mode=ro', uri=True)
cur = con.cursor()
cur.execute(\"SELECT email, isActive, data FROM providerConnections WHERE provider='antigravity'\")
for email, active, data in cur.fetchall():
    d = json.loads(data)
    print(f'{email}: active={active} | errorCode={d.get(\"errorCode\")} | testStatus={d.get(\"testStatus\")}')
"
```
`errorCode` harus `None` dan `testStatus` harus `active`.

---

## Troubleshooting

### Guard diam terus, tapi akun tetep error
- **Cek apakah akunnya `isActive=1`.** Guard sengaja ngelewatin akun yang dimatiin.
- **Cek jenis error-nya.** Kalau `invalid_grant` atau 403, guard emang gak bisa benerin — dia cuma bisa notif.

### Crontab gak jalan
- Cek cron aktif: `systemctl status cron`
- Cek path python: `which python3` → pastiin sama dengan yang di crontab
- Cron gak punya PATH lengkap — pakai **absolute path** buat python DAN script

### `ModuleNotFoundError`
Script cuma pake stdlib (`sqlite3`, `urllib`, `json`). Kalau error, berarti python kamu gak lengkap:
```bash
apt install python3-full
```

### Guard jalan tapi 9Router gagal restart
- Cek `RESTART_CMD` sesuai dengan setup kamu:
  - systemd: `["systemctl", "restart", "9router.service"]`
  - docker: `["docker", "restart", "nama-container"]`
  - pm2: `["pm2", "restart", "nama-proses"]`
- Cek nama service-nya benar: `systemctl list-units | grep -i 9router`

### Kredensial salah / gak ketemu
Gejala: log nunjukin `HTTP_400` pas refresh token.
- Nilai `CLIENT_ID`/`CLIENT_SECRET` harus PERSIS sama dengan yang dipakai 9Router
- Cek ulang Langkah 1 — jangan asal tempel nilai dari sumber gak jelas

### Akun terus balik ke `isActive=0` sendiri
- Berarti 9Router tetep gagal pakai akun itu → cek jenis error di log 9Router:
  ```bash
  journalctl -u 9router.service --since "1 hour ago" | grep -i antigravity
  ```

### Guard malah nyalain akun yang gue matiin
- Versi lama punya bug ini. Pastiin script kamu ada baris:
  ```python
  if is_active == 0:
      continue
  ```

---

## Batasan

Guard ini **TIDAK**:
- nyentuh akun yang dimatiin sendiri (`isActive=0`)
- bisa login ulang kalau refreshToken dicabut
- bisa lewatin verifikasi akun Google
- nambah kuota
- bikin akun aman dari razia Google
- nyentuh WA bot / sniper / provider lain

Guard ini **CUMA** nambal 3 hal:
1. Akibat gangguan jaringan
2. Token basi
3. Lock yang nyangkut

### Analogi

9Router = mesin mobil bagus, tapi gak punya dongkrak.
Guard = montir yang ngecek tiap 5 menit.

Ban bocor (jaringan blip) → guard nambal langsung.
Mesin rusak parah (token dicabut / perlu verifikasi) → guard cuma bisa nelpon:
*"Bos, ini gak bisa gue benerin, turun tangan dong."*

---

## File di Repo Ini

| File | Isi |
|---|---|
| `README.md` | Dokumen ini — instalasi, verifikasi, troubleshooting |
| `antigravity-guard-9router.md` | Spec lengkap + kode script Python |

---

## Disclaimer

Tool komunitas, bukan produk resmi 9Router atau Google. Dipakai dengan risiko sendiri.

Repo ini **gak mengandung kredensial apa pun** — kamu harus ambil sendiri dari 9Router kamu (lihat [Cara Dapetin Kredensial](#cara-dapetin-kredensial)).

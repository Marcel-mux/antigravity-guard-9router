# Antigravity Guard untuk 9Router — Spec + Script Siap Pakai

Dokumen ini buat **Hermes kamu**. Isinya: spec lengkap + kode script siap pakai.
Guard ini bikin akun OAuth Antigravity di 9Router otomatis pulih kalau kena error jaringan / token basi / lock nyangkut.

---

## INSTRUKSI BUAT HERMES

Kalau kamu (Hermes) lagi baca ini, tugasmu:

1. **Cek dulu** environment user: apakah `9router` jalan (systemd/docker/pm2), di mana `data.sqlite`-nya, dan apakah ada akun Antigravity yang sudah di-connect OAuth.
2. **Ambil 2 kredensial** yang dibutuhkan (`CLIENT_ID` & `CLIENT_SECRET`) — lihat bagian 5. **JANGAN tanya user password/API key apapun di chat.**
3. **Tulis script** dari kode di bagian 6 ke `antigravity_guard.py`, sesuaikan `DB_PATH` & `RESTART_CMD`.
4. **Tes manual dulu** (jangan langsung pasang cron).
5. **Pasang cron** tiap 5 menit setelah tes lolos.

---

## 1. Apa yang Guard Ini Lakuin

| Item | Detail |
|---|---|
| **Nama script** | `antigravity_guard.py` |
| **Jadwal** | Cron, tiap **5 menit** |
| **Tugas** | "Dokter jaga" buat akun Antigravity di 9Router |
| **Output** | Cuma lapor kalau ada yang dipulihkan (kalau normal → diam total) |

---

## 2. Alur Kerja

```
[SETIAP 5 MENIT]

1. Buka DB 9Router → ambil semua koneksi provider='antigravity'
   ↓
2. SKIP akun dengan isActive = 0  (dimatiin manual → JANGAN disentuh)
   ↓
3. Cek tiap akun aktif: ada masalah?
   → errorCode gak null?
   → lastError ada isinya?
   → testStatus bukan "active"?
   → ada sisa modelLock?
   ↓
4. Kalau ADA masalah:
   a. Tes accessToken sekarang (request ringan ke API Google)
      → 200 OK? bersihin error + lock. SELESAI.
      → gagal? lanjut b
   b. Tukar refreshToken ke Google
      → dapet token baru? tes lagi
         → 200 OK? isActive=1, bersihin error, restart 9Router
      → "invalid_grant"? notif: wajib login ulang
   c. 403 "VALIDATION_REQUIRED"? notif: akun butuh verifikasi
   ↓
5. Ada yang dipulihkan → lapor. Gak ada → diam.
```

---

## 3. Penyakit yang Bisa / Gak Bisa Disembuhin

| Penyakit | Guard Bisa? |
|---|---|
| 401 + `fetch failed` (jaringan VPS blip) | ✅ Ya |
| AccessToken expired (~1 jam umur) | ✅ Ya |
| `modelLock` nyangkut | ✅ Ya |
| `isActive=0` akibat error | ✅ Ya |
| `invalid_grant` (refreshToken dicabut) | ❌ Login ulang |
| 403 `VALIDATION_REQUIRED` | ❌ Verifikasi manual |
| 429 rate limit | ❌ Tunggu reset (~4-5 jam) |

---

## 4. Data di DB yang Dipakai

Tabel `providerConnections`:

| Kolom | Fungsi |
|---|---|
| `provider` | Filter `= 'antigravity'` |
| `email` | Identitas akun |
| `isActive` | 1 = aktif, 0 = dimatiin manual (SKIP) |
| `data` (JSON) | `refreshToken`, `accessToken`, `expiresAt`, `projectId`, `lastError`, `errorCode`, `testStatus`, `modelLock_*` |
| `updatedAt` | Timestamp update terakhir |

**Endpoint Google:**
- Token refresh: `https://oauth2.googleapis.com/token`
- Tes API: `https://daily-cloudcode-pa.googleapis.com/v1internal:generateContent`
- Header: `User-Agent: antigravity/ide/<versi> <os>/<arch>`

---

## 5. ⚠️ CARA DAPETIN KREDENSIAL (WAJIB DIBACA HERMES)

Script butuh 2 nilai: `CLIENT_ID` dan `CLIENT_SECRET`. Ini **identitas aplikasi Antigravity**, bukan password pribadi user.

Cara cari (urut dari paling gampang):

1. **Dari DB 9Router** — cek tabel config/settings, kemungkinan tersimpan di sana.
2. **Dari folder installasi 9Router** — cari file config/`.env`/`credentials`:
   ```
   grep -rn "client_id\|client_secret" /path/to/9router/app/ 2>/dev/null | head
   ```
3. **Dari node_modules 9Router** — biasanya di-bundle di chunk file:
   ```
   grep -roh "GOCSPX-[A-Za-z0-9_-]*" /path/to/9router/ | head
   ```
4. **Kalau semua gagal** — minta user buka 9Router Dashboard → Providers → Antigravity, lihat detail koneksi / log.

**Kalau tetap gak ketemu:** jangan karang nilai. Laporkan ke user: *"Kredensial OAuth Antigravity gak ketemu di sistem. Bisa dicek di 9Router Dashboard atau minta dari yang setup?"*

---

## 6. Script Lengkap

Simpan sebagai `antigravity_guard.py`. **Ganti `CLIENT_ID` dan `CLIENT_SECRET` dulu** (lihat bagian 5).

```python
#!/usr/bin/env python3
"""
Antigravity Multi-Account Guard for 9Router
Cek semua providerConnections Antigravity di data.sqlite 9Router.

Untuk setiap akun:
  - Kalau error (401, 404, unavailable, modelLock):
    - Refresh access token via Google OAuth kalau perlu.
    - Tes koneksi pakai model standar (gemini-3.8-flash-high).
    - Kalau 200 OK: aktifkan koneksi, bersihin modelLock & error.
    - Kalau 403 "Verify your account": alert user buat verifikasi Google.
    - Kalau invalid_grant / revoked: alert user buat login ulang.
  - Akun dengan isActive=0 (dimatiin manual user) -> DILEWATI total.
Restart 9router kalau ada akun yang berhasil dipulihkan.
"""

import os
import json
import sqlite3
import urllib.request
import urllib.parse
import urllib.error
import datetime
import subprocess
import time

# ============ SESUAIKAN BAGIAN INI ============
DB_PATH = "/root/.9router/db/data.sqlite"           # lokasi data.sqlite 9Router kamu
STATE_FILE = "/root/.hermes/cache/antigravity_guard.state"
RESTART_CMD = ["systemctl", "restart", "9router.service"]  # docker: ["docker","restart","nama"] / pm2: ["pm2","restart","nama"]
CLIENT_ID = "GANTI_DENGAN_CLIENT_ID_ANTIGRAVITY"
CLIENT_SECRET = "GANTI_DENGAN_CLIENT_SECRET_ANTIGRAVITY"
# ==============================================


def load_state():
    if os.path.exists(STATE_FILE):
        try:
            with open(STATE_FILE, "r") as f:
                return json.load(f)
        except Exception:
            pass
    return {"last_alert": {}, "status": "ok", "last_recovery": 0}


def save_state(state):
    os.makedirs(os.path.dirname(STATE_FILE), exist_ok=True)
    with open(STATE_FILE, "w") as f:
        json.dump(state, f)


def refresh_google_token(refresh_token):
    post_data = urllib.parse.urlencode({
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
        "refresh_token": refresh_token,
        "grant_type": "refresh_token",
    }).encode("utf-8")

    req = urllib.request.Request(
        "https://oauth2.googleapis.com/token",
        data=post_data,
        headers={"Content-Type": "application/x-www-form-urlencoded"}
    )
    try:
        with urllib.request.urlopen(req, timeout=15) as resp:
            body = json.loads(resp.read().decode("utf-8"))
            return True, body.get("access_token"), body.get("expires_in", 3599), None
    except urllib.error.HTTPError as e:
        err_msg = e.read().decode("utf-8", errors="ignore")
        return False, None, 0, f"HTTP_{e.code}_{err_msg}"
    except Exception as e:
        return False, None, 0, f"NET_{type(e).__name__}_{e}"


def test_antigravity_model(access_token, project_id="aicode-consumers"):
    url = "https://daily-cloudcode-pa.googleapis.com/v1internal:generateContent"
    body = {
        "project": project_id,
        "model": "gemini-3.8-flash-high",
        "request": {
            "contents": [{"role": "user", "parts": [{"text": "hi"}]}]
        }
    }
    req = urllib.request.Request(url, data=json.dumps(body).encode("utf-8"), headers={
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json",
        "User-Agent": "antigravity/ide/2.11.0 darwin/arm64"
    })
    try:
        with urllib.request.urlopen(req, timeout=12) as resp:
            return 200, "OK"
    except urllib.error.HTTPError as e:
        err_body = e.read().decode("utf-8", errors="ignore")
        return e.code, err_body
    except Exception as e:
        return 0, f"{type(e).__name__}: {e}"


def main():
    if not os.path.exists(DB_PATH):
        return

    con = sqlite3.connect(f"file:{DB_PATH}?mode=ro", uri=True)
    cur = con.cursor()
    cur.execute("SELECT id, name, email, isActive, data FROM providerConnections WHERE provider='antigravity'")
    rows = cur.fetchall()
    con.close()

    if not rows:
        return

    state = load_state()
    now = time.time()
    alerts = state.get("last_alert", {})
    if not isinstance(alerts, dict):
        alerts = {}

    recovered_accounts = []

    for conn_id, name, email, is_active, data_raw in rows:
        acct_label = email or name or conn_id[:8]
        data = json.loads(data_raw) if data_raw else {}

        error_code = data.get("errorCode")
        last_error = str(data.get("lastError") or "")
        test_status = data.get("testStatus")
        refresh_token = data.get("refreshToken")
        access_token = data.get("accessToken")
        project_id = data.get("projectId") or "aicode-consumers"

        has_locks = any(k.startswith("modelLock_") and v is not None for k, v in data.items())

        # JANGAN sentuh akun yang sengaja dimatiin user
        if is_active == 0:
            continue

        needs_check = (
            error_code is not None
            or last_error != ""
            or test_status != "active"
            or has_locks
        )
        if not needs_check:
            continue

        # Cooldown: batasi percobaan recovery per akun
        last_acct_check = alerts.get(f"check_{acct_label}", 0)
        if now - last_acct_check < 120:
            continue
        alerts[f"check_{acct_label}"] = now

        # 1. Probe access token sekarang
        status_code = 0
        err_msg = ""
        if access_token:
            status_code, err_msg = test_antigravity_model(access_token, project_id)

        # 2. Kalau gagal (401/0), coba refresh
        if status_code in (401, 0) and refresh_token:
            ok, new_token, expires_in, rf_err = refresh_google_token(refresh_token)
            if ok and new_token:
                access_token = new_token
                now_dt = datetime.datetime.now(datetime.timezone.utc)
                data["accessToken"] = new_token
                data["expiresIn"] = expires_in
                data["expiresAt"] = (now_dt + datetime.timedelta(seconds=expires_in)).isoformat().replace("+00:00", "Z")
                data["lastRefreshAt"] = now_dt.isoformat().replace("+00:00", "Z")
                status_code, err_msg = test_antigravity_model(access_token, project_id)
            elif rf_err and "invalid_grant" in rf_err:
                if now - alerts.get(f"revoked_{acct_label}", 0) > 3600:
                    print(f"🚨 [Antigravity] Akun {acct_label} token revoked/expired. Perlu login ulang di Dashboard 9Router.")
                    alerts[f"revoked_{acct_label}"] = now
                continue

        # 3. Handle hasil
        if status_code == 200:
            now_dt = datetime.datetime.now(datetime.timezone.utc)
            now_iso = now_dt.isoformat().replace("+00:00", "Z")

            data["lastError"] = None
            data["errorCode"] = None
            data["lastErrorAt"] = None
            data["testStatus"] = "active"

            for k in list(data.keys()):
                if k.startswith("modelLock_"):
                    data[k] = None

            con_rw = sqlite3.connect(DB_PATH, timeout=30)
            cur_rw = con_rw.cursor()
            cur_rw.execute(
                "UPDATE providerConnections SET isActive=1, data=?, updatedAt=? WHERE id=?",
                (json.dumps(data), now_iso, conn_id)
            )
            con_rw.commit()
            con_rw.close()

            recovered_accounts.append(acct_label)

        elif status_code == 403:
            if "Verify your account" in err_msg or "VALIDATION_REQUIRED" in err_msg:
                if now - alerts.get(f"verify_{acct_label}", 0) > 1800:
                    print(f"⚠️ [Antigravity Security] Akun Google {acct_label} butuh verifikasi keamanan (403: Verify your account). Buka Gmail/Google akun tersebut untuk konfirmasi.")
                    alerts[f"verify_{acct_label}"] = now
            else:
                if now - alerts.get(f"err403_{acct_label}", 0) > 1800:
                    print(f"⚠️ [Antigravity] Akun {acct_label} mengembalikan HTTP 403: {err_msg[:120]}")
                    alerts[f"err403_{acct_label}"] = now

        elif status_code == 404:
            if now - alerts.get(f"err404_{acct_label}", 0) > 1800:
                print(f"⚠️ [Antigravity] Akun {acct_label} mengembalikan HTTP 404 pada model test.")
                alerts[f"err404_{acct_label}"] = now

    state["last_alert"] = alerts
    if recovered_accounts:
        state["last_recovery"] = now
        print(f"🛡️ [Antigravity Guard] Berhasil auto-heal & aktifkan akun: {', '.join(recovered_accounts)}. Me-restart 9router...")
        subprocess.run(RESTART_CMD, check=False)

    save_state(state)


if __name__ == "__main__":
    main()
```

---

## 7. Langkah Pasang

1. Tulis script di atas ke `antigravity_guard.py`.
2. Ganti `CLIENT_ID` & `CLIENT_SECRET` (lihat bagian 5).
3. Sesuaikan `DB_PATH` & `RESTART_CMD` (systemd/docker/pm2 punya kamu).
4. `chmod +x antigravity_guard.py`
5. **TES MANUAL DULU** (jangan langsung cron):
   ```
   python3 antigravity_guard.py
   ```
   - Diam = semua akun sehat (hasil bener).
   - Error = perbaiki dulu.
6. Pasang cron tiap 5 menit:
   ```
   */5 * * * * /usr/bin/python3 /path/to/antigravity_guard.py >> /var/log/antigravity_guard.log 2>&1
   ```

---

## 8. Test Case yang Harus Lolos

| # | Skenario | Hasil Diharapkan |
|---|---|---|
| 1 | Semua akun sehat | Guard diam, gak update apa-apa |
| 2 | `isActive=0` (dimatiin) | Guard SKIP, akun tetap mati |
| 3 | `lastError` diisi dummy + token valid | Guard bersihin, akun tetep aktif |
| 4 | `modelLock_*` diisi dummy | Guard bersihin lock-nya |
| 5 | `refreshToken` dirusak | Guard notif "login ulang", GAK crash |
| 6 | DB gak ada / terkunci | Guard exit bersih, GAK crash |

---

## 9. Batasan (WAJIB TAHU)

Guard ini **TIDAK**:
- nyentuh akun yang dimatiin sendiri (`isActive=0`)
- bisa login ulang kalau token dicabut
- bisa lewatin verifikasi akun Google
- nambah kuota
- bikin akun aman dari razia Google
- nyentuh WA bot / sniper / provider lain

Guard ini **CUMA** nambal 3 hal: akibat gangguan jaringan, token basi, lock nyangkut.

---

## 10. Analogi

9Router = mesin mobil bagus, tapi gak punya dongkrak.
Guard = montir yang ngecek tiap 5 menit.

Ban bocor (jaringan blip) → guard nambal langsung.
Mesin rusak parah (token dicabut / perlu verifikasi) → guard cuma bisa nelpon:
*"Bos, ini gak bisa gue benerin, turun tangan dong."*

---

## 11. Yang Harus Diingat

- **Gak ada efek samping.** Sehat = diam total.
- **Restart 9Router = koneksi putus sebentar.** Kalau 9Router dipakai produksi (bot WA dll), request yang jalan bisa terputus.
- **GAK bikin akun aman dari razia Google.** Guard cuma benerin kerusakan teknis.
- **Rawan rusak kalau struktur DB 9Router berubah** saat update versi.
- **Gak ada garansi.** Script community-made buat setup spesifik, bukan produk resmi.

---

*Dokumen ini gak mengandung kredensial apa pun. Hermes kamu ambil sendiri dari 9Router kamu (lihat bagian 5).*

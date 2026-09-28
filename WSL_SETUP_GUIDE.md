# 🐧 WSL Ubuntu Setup Guide — Telegram Signal EA

Panduan instalasi & konfigurasi project **telegram-signal-mt5** di **WSL2 (Ubuntu 24.04)**.

> **MT5 Anda ada di Windows?** Lihat **Langkah 8 → Opsi C** (transparan: MT5 di Windows sebagai server RPC, Python di WSL) atau **Lampiran** (jalankan seluruh project langsung di Windows dengan paket `MetaTrader5` native — paling sederhana).

## Penting Sebelum Mulai

- Paket Python resmi `MetaTrader5` **hanya mendukung Windows** (berisi DLL Windows). Di WSL/Linux, `pip install MetaTrader5==5.0.5200` akan **gagal build**.
- Solusi yang digunakan di panduan ini: **`mt5linux`** — paket resmi komunitas "MetaTrader5 for Linux users" (berbasis Wine + RPyC + server binary), dikelola aktif, API-nya meniru API resmi.
- Project ini **tidak butuh EA (MQL5)** — script Python sudah menjadi robot (lihat README).
- Diperlukan: akun Telegram + `api_id`/`api_hash` (my.telegram.org), akun broker MT5 (login/password/server), API key OpenAI.

---

## Arsitektur Target

```
┌─────────────────────────────────────────────────────────┐
│ WSL2 Ubuntu                                             │
│  ┌────────────┐    ┌───────────────┐    ┌─────────────┐ │
│  │  Python 3.12│──►│ Signal EA code │──►│ shim          │ │
│  │  + venv     │    │ (Telegram,     │    │ MetaTrader5.py│ │
│  │             │    │  parser, DB)   │    └──────┬────── │ │
│  └────────────┘    └───────────────┘           RPyC      │
│                                                 │        │
│  ┌──────────────┐   wine mt5server.exe ──► TCP 18812     │
│  │ MT5 terminal │◄────────────────────────────────────── │
│  │ (via Wine/   │      (client terhubung ke server)      │
│  │  WSLg)       │                                        │
│  └──────────────┘                                        │
└─────────────────────────────────────────────────────────┘

Atau bila MT5 di Windows host (skenario Anda):
┌────────────────┐          ┌──────────────────────────────┐
│ WSL2 Ubuntu     │  TCP     │ Windows                      │
│ Signal EA code  │──18812──►│  mt5server.exe ──► MT5 terminal│
│ (Python+shim)   │          │  (RPC server)    (login+AT)   │
└────────────────┘          └──────────────────────────────┘
```

---

## Langkah 1 — Aktifkan WSL2 + Ubuntu 24.04

Di **PowerShell Windows (Admin)**:

```powershell
wsl --install -d Ubuntu-24.04
wsl --set-default-version 2
```

Buka Ubuntu, buat user, lalu **aktifkan systemd** (untuk auto-start nanti). Edit `/etc/wsl.conf`:

```bash
sudo nano /etc/wsl.conf
```

Isi:
```ini
[boot]
systemd=true
```

Tutup semua terminal WSL, lalu restart dari PowerShell:
```powershell
wsl --shutdown
wsl -d Ubuntu-24.04
```

## Langkah 2 — Update Sistem & Python 3.12

Ubuntu 24.04 sudah menyertakan Python 3.12:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3.12 python3.12-venv python3.12-dev python3-pip git build-essential
python3.12 --version   # harus >= 3.12.7
```

## Langkah 3 — Clone Project & Buat Virtual Environment

```bash
cd ~
git clone <url-repo> telegram-signal-mt5
cd telegram-signal-mt5
python3.12 -m venv venv
source venv/bin/activate
```

## Langkah 4 — Install Dependencies (PENTING: tanpa MetaTrader5)

Buat file requirements khusus WSL (hilangkan baris `MetaTrader5` yang Windows-only):

```bash
grep -vE "^MetaTrader5" requirements.txt > requirements-wsl.txt
pip install --upgrade pip
pip install -r requirements-wsl.txt
pip install mt5linux
```

> Catatan: versi `MetaTrader5==5.0.5200` tidak bisa diinstall di Linux. Semua modul lain (telethon, sqlalchemy, openai, dst.) tetap terinstall normal.

## Langkah 5 — Shim `MetaTrader5` → `mt5linux`

Kode project menggunakan `import MetaTrader5 as mt5` dan memanggil fungsi level modul (`initialize`, `login`, `positions_get`, `order_send`, dst.). Di WSL kita jembatani dengan **shim** yang diletakkan di dalam venv (tidak mengubah repo):

```bash
nano venv/lib/python3.12/site-packages/MetaTrader5.py
```

Isi:

```python
"""
Shim MetaTrader5 untuk Linux/WSL.
Menggantikan paket resmi (Windows-only) dengan klien mt5linux agar
'import MetaTrader5 as mt5' tetap berfungsi. Lihat WSL_SETUP_GUIDE.md.
"""
import os

try:
    from mt5linux import MetaTrader5 as _Client
    _HAS_MT5LINUX = True
except Exception:
    _Client = None
    _HAS_MT5LINUX = False

_client = None


def _get():
    global _client
    if _client is None:
        host = os.environ.get("MT5_HOST")
        port = os.environ.get("MT5_PORT")
        _client = _Client(host=host, port=port) if host else _Client()
    return _client


def initialize(path=None, login=None, password=None, server=None):
    if not _HAS_MT5LINUX:
        raise ImportError("mt5linux belum terinstall (pip install mt5linux)")
    kw = {}
    if login is not None:
        kw["login"] = login
    if password is not None:
        kw["password"] = password
    if server is not None:
        kw["server"] = server
    try:
        if kw:
            return bool(_get().initialize(**kw))
        return bool(_get().initialize())
    except Exception:
        return False


def login(login, password=None, server=None):
    kw = {"login": login}
    if password is not None:
        kw["password"] = password
    if server is not None:
        kw["server"] = server
    try:
        return bool(_get().initialize(**kw))
    except Exception:
        return False


def shutdown():
    try:
        return bool(_get().shutdown())
    except Exception:
        return True


def last_error():
    return [0, ""]


def terminal_info():
    return _get().terminal_info() if _HAS_MT5LINUX else None


def account_info():
    return _get().account_info() if _HAS_MT5LINUX else None


def positions_get(symbol=None, ticket=None):
    if not _HAS_MT5LINUX:
        return None
    try:
        if ticket is not None:
            return _get().positions_get(ticket=ticket)
        if symbol is not None:
            return _get().positions_get(symbol=symbol)
        return _get().positions_get()
    except Exception:
        return None


def orders_get(symbol=None):
    return _get().orders_get(symbol=symbol) if _HAS_MT5LINUX else None


def symbol_info(name):
    return _get().symbol_info(name) if _HAS_MT5LINUX else None


def symbol_info_tick(name):
    return _get().symbol_info_tick(name) if _HAS_MT5LINUX else None


def symbol_select(name, enable=True):
    return bool(_get().symbol_select(name, enable)) if _HAS_MT5LINUX else False


def order_send(request):
    return _get().order_send(request) if _HAS_MT5LINUX else None


def __getattr__(name):
    # Ambil konstanta (TRADE_ACTION_*, ORDER_TYPE_*, TRADE_RETCODE_*, dll.)
    # dari objek klien, lalu fallback ke modul mt5linux.
    if _HAS_MT5LINUX:
        try:
            return getattr(_get(), name)
        except AttributeError:
            import mt5linux
            if hasattr(mt5linux, name):
                return getattr(mt5linux, name)
    raise AttributeError(f"module 'MetaTrader5' has no attribute '{name}'")
```

> Kegunaan env: `MT5_HOST` & `MT5_PORT` mengarahkan ke server RPC (default `localhost:18812`). Jika server dijalankan via Docker/manual dengan 2 argumen (`host`, `port`), set kedua env ini.

## Langkah 6 — Konfigurasi `.env`

```bash
cp .env.example .env
nano .env
```

Isi sesuai environment Anda:

```bash
# Telegram
TELEGRAM_API_ID=12345
TELEGRAM_API_HASH=xxxxxxxxxxxxxxxxxxxxxxxx
PHONE_NUMBER=+62xxx
TELEGRAM_GROUPS=@signalgroup1,@signalgroup2

# OpenAI
OPENAI_API_KEY=sk-xxxx

# MetaTrader5
MT5_LOGIN=12345678
MT5_PASSWORD=your_mt5_password
MT5_SERVER=Broker-ServerName
# MT5_PATH biarkan kosong di WSL (terminal dibuka via Wine, bukan via path)
MT5_PATH=

# Database
DATABASE_URL=sqlite+aiosqlite:///data/signal_ea.db

# Opsional untuk shim
# MT5_HOST=localhost
# MT5_PORT=18812
```

## Langkah 7 — Setup Sesi Telegram

Script `scripts/setup_session.py` belum tersedia di repo ini, jadi buat sesi langsung dengan Telethon (file sesi tersimpan di `data/telegram.session`):

```bash
source venv/bin/activate
python - <<'EOF'
import asyncio
from telethon import TelegramClient
from config.settings import settings

async def main():
    client = TelegramClient(
        str(settings.data_dir / "telegram.session"),
        settings.get_telegram_api_id_int(),
        settings.telegram_api_hash,
    )
    await client.start(phone=settings.phone_number)
    me = await client.get_me()
    print("Session OK:", me.username or me.id)
    await client.disconnect()

asyncio.run(main())
EOF
```

Akan muncul prompt kode verifikasi Telegram (SMS/app). Setelah sukses, sesi tersimpan & tidak perlu login ulang.

## Langkah 8 — Siapkan Koneksi MT5 (pilih salah satu opsi)

### Opsi A — MT5 + Wine di dalam WSL (disarankan)

**1. Install WineHQ** (untuk GUI di WSL2, WSLg otomatis tersedia):

```bash
sudo dpkg --add-architecture i386
sudo mkdir -pm755 /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/winehq-archive.key https://dl.winehq.org/wine-builds/winehq.key
sudo wget -NP /etc/apt/sources.list.d/ https://dl.winehq.org/wine-builds/ubuntu/dists/noble/winehq-noble.sources
sudo apt update
sudo apt install -y --install-recommends winehq-stable
```

**2. Install MetaTrader 5 di Wine** — unduh installer `mt5setup.exe` dari broker Anda, lalu:

```bash
wine mt5setup.exe
```

- Login ke akun broker di terminal MT5.
- Aktifkan **AutoTrading**: `Tools → Options → Expert Advisors → Allow Algo Trading`.

**3. Jalankan server RPC `mt5server.exe`** — unduh dari rilis GitHub `lucas-campagna/mt5linux`:

```bash
wine mt5server.exe            # port default 18812
wine mt5server.exe --help     # lihat opsi lain
```

Biarkan terminal MT5 & server tetap terbuka di sesi WSL.

**4. Uji koneksi dari Python:**

```bash
source venv/bin/activate
python -c "import MetaTrader5 as mt5; print('init:', bool(mt5.initialize())); print(mt5.terminal_info()); mt5.shutdown()"
```

### Opsi B — MT5 via Docker (tanpa repot GUI/Wine)

`mt5linux` menyediakan container beserta UI noVNC:

```bash
source venv/bin/activate
pip install mt5linux
# Ikuti docker compose di repo mt5linux (folder docker/) atau:
docker compose up -d
```

Lalu arahkan shim ke container:

```bash
export MT5_HOST=localhost
export MT5_PORT=18812
python -c "import MetaTrader5 as mt5; print(bool(mt5.initialize())); print(mt5.terminal_info()); mt5.shutdown()"
```

### Opsi C — MT5 di Windows host, Python di WSL (cocok untuk Anda)

Karena terminal MT5 ada di **Windows**, gunakan Windows sebagai server RPC dan WSL sebagai klien.

**1. Di Windows — siapkan server RPC `mt5server.exe`**

Unduh `mt5server.exe` dari halaman rilis GitHub `lucas-campagna/mt5linux` (letakkan di folder mudah, mis. `C:\mt5linux\`):

```powershell
C:\mt5linux\mt5server.exe
```

Command prompt akan berjalan terus menerus (dibuka di port `18812` default). Terminal MT5 harus sudah login & **AutoTrading ON** (`Tools → Options → Expert Advisors → Allow Algo Trading`).

> Alternatif tanpa `mt5server.exe`: install Python di Windows lalu `pip install mt5linux` dan jalankan dari Windows Python `python -m mt5linux.server` (varian klien-server lama).

**2. Di Windows — buka port untuk WSL**

```powershell
# PowerShell (Admin)
New-NetFirewallRule -DisplayName "MT5 RPC 18812" -Direction Inbound -Protocol TCP -LocalPort 18812 -Action Allow
```

**3. Di Windows — cek IP host**

```powershell
ipconfig
# Catat IPv4 (misal 192.168.1.10)
```

**4. Di WSL — arahkan koneksi ke IP Windows tadi**

```bash
export MT5_HOST=192.168.1.10    # ganti dengan IP Windows
export MT5_PORT=18812
```

Uji:
```bash
python -c "import MetaTrader5 as mt5; print(bool(mt5.initialize())); print(mt5.terminal_info()); mt5.shutdown()"
```

> Agar permanen, tambahkan kedua `export` ke `~/.bashrc`.

## Langkah 9 — Verifikasi

```bash
cd ~/telegram-signal-mt5
source venv/bin/activate

# 1) Unit test (tidak butuh MT5/Telegram asli)
pytest tests/unit/ -v

# 2) Cek koneksi MT5 (pastikan terminal + server berjalan)
python -c "import MetaTrader5 as mt5; print(bool(mt5.initialize())); print(mt5.terminal_info()); mt5.shutdown()"

# 3) Jalankan aplikasi
python main.py

# 4) Atau gunakan CLI
python cli.py start
python cli.py status
python cli.py stop
```

Cek log: `logs/app.log`, `logs/error.log`, `logs/trades.log`.

## Langkah 10 — Auto-Start via systemd (opsional)

Pastikan systemd aktif (Langkah 1), lalu buat unit service:

```bash
sudo nano /etc/systemd/system/telegram-signal-ea.service
```

```ini
[Unit]
Description=Telegram Signal EA
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/<user>/telegram-signal-mt5
ExecStart=/home/<user>/telegram-signal-mt5/venv/bin/python main.py
Restart=always
RestartSec=10
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now telegram-signal-ea
sudo systemctl status telegram-signal-ea
```

> Pastikan MT5 terminal + `mt5server.exe` juga berjalan sebelum service (jalankan via Wine di sesi terpisah / sesuaikan unit).

---

## Lampiran — Menjalankan Langsung di Windows (Native)

Karena MT5 Anda berada di Windows, cara **paling sederhana** adalah menjalankan seluruh project langsung di Windows — paket resmi `MetaTrader5` berjalan native tanpa shim/WSL.

### A. Prasyarat Windows
- Windows 10/11 + MetaTrader 5 sudah terpasang & login ke akun broker
- Python 3.12.7+ (unduh dari python.org, centang **Add python.exe to PATH**)
- Akun Telegram + `api_id`/`api_hash` dan API key OpenAI

### B. Setup Project

```powershell
cd C:\path\to\telegram-signal-mt5
python -m venv venv
venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt    # MetaTrader5 native OK di Windows
```

### C. Konfigurasi `.env`

```powershell
copy .env.example .env
notepad .env
```

```bash
# Telegram
TELEGRAM_API_ID=12345
TELEGRAM_API_HASH=xxxxxxxxxxxxxxxxxxxxxxxx
PHONE_NUMBER=+62xxx
TELEGRAM_GROUPS=@signalgroup1,@signalgroup2

# OpenAI
OPENAI_API_KEY=sk-xxxx

# MetaTrader5
MT5_LOGIN=12345678
MT5_PASSWORD=your_mt5_password
MT5_SERVER=Broker-ServerName
MT5_PATH=C:\Program Files\MetaTrader 5\terminal64.exe   # isi bila ingin auto-launch
```

Pastikan **AutoTrading ON** di terminal MT5: `Tools → Options → Expert Advisors → Allow Algo Trading`.

### D. Setup Sesi Telegram

Script `scripts/setup_session.py` belum tersedia di repo ini. Buat sesi dengan Telethon:

```powershell
python -c @"
import asyncio
from telethon import TelegramClient
from config.settings import settings

async def main():
    client = TelegramClient(str(settings.data_dir / 'telegram.session'),
                            settings.get_telegram_api_id_int(),
                            settings.telegram_api_hash)
    await client.start(phone=settings.phone_number)
    me = await client.get_me()
    print('Session OK:', me.username or me.id)
    await client.disconnect()

asyncio.run(main())
"@
```

Masukkan kode verifikasi ketika diminta. Sesi tersimpan di `data\telegram.session`.

### E. Verifikasi & Menjalankan

```powershell
pytest tests/unit/ -v
python -c "import MetaTrader5 as mt5; print(bool(mt5.initialize())); print(mt5.terminal_info()); mt5.shutdown()"
python main.py
```

CLI: `python cli.py start`, `python cli.py status`, `python cli.py stop`.

### F. Auto-Start di Windows (opsional)

Gunakan **Task Scheduler** (task namanya mis. `TelegramSignalEA`):
- `Trigger`: At startup
- `Action`: `C:\path\to\telegram-signal-mt5\venv\Scripts\python.exe main.py`
- `Start in`: `C:\path\to\telegram-signal-mt5`

---

## Troubleshooting

| Gejala | Solusi |
|--------|--------|
| `pip install MetaTrader5==5.0.5200` gagal (`error: Metadata generation failed` / build) | Pakai `requirements-wsl.txt` (tanpa MetaTrader5) + `pip install mt5linux`. |
| `ModuleNotFoundError: mt5linux` | `pip install mt5linux` di dalam venv. |
| `MT5 initialize() failed` / `Not available` | Pastikan terminal MT5 (Wine/Docker/Windows) sudah login & `mt5server.exe` berjalan; cek `MT5_HOST`/`MT5_PORT`. |
| `Connection refused` pada port 18812 | Server RPC belum jalan; jalankan `mt5server.exe` di Windows/Wine atau `docker compose up -d`; cek firewall Windows. |
| `TRADE_RETCODE_REQUOTE / INVALID_STOPS` | Harga spread; normal, sistem punya retry. |
| Terminal MT5 tidak tampil GUI di WSL | WSLg harus aktif (`wsl --update`, pastikan WSL2). |
| `MetaTrader5` tidak ketemu di Windows native | Pastikan `pip install -r requirements.txt` berhasil (paket resmi hanya Windows). |
| `Wine exited with a non-zero status` (Winetricks) | Pastikan `wine` 64-bit: `wine --version`; coba `wineserver -k` lalu ulang. |
| `FloodWaitError` dari Telegram | Wajar di awal; sistem punya rate limiter (jeda 1–3 detik). |
| systemd tidak aktif | Periksa `[boot] systemd=true` di `/etc/wsl.conf`, lalu `wsl --shutdown`. |
| Ganda import `MetaTrader5` bentrok dengan package asli | Jangan menginstall `MetaTrader5` resmi di venv yang sama dengan shim. |

## Checklist Selesai

**Jalur A — Semua di WSL:**
- [ ] WSL2 + Ubuntu 24.04 + systemd aktif
- [ ] Python 3.12 + venv + dependencies terpasang
- [ ] Shim `MetaTrader5.py` di venv
- [ ] `.env` terisi (Telegram, OpenAI, MT5)
- [ ] Sesi Telegram berhasil dibuat (`data/telegram.session`)
- [ ] MT5 terminal (Wine/Docker) login + AutoTrading ON + server RPC berjalan
- [ ] `pytest tests/unit/` lolos
- [ ] `python main.py` berjalan & logs normal

**Jalur B — MT5 di Windows host (Opsi C / Lampiran):**
- [ ] `mt5server.exe` berjalan di Windows (port 18812) + firewall diizinkan
- [ ] `MT5_HOST`/`MT5_PORT` diatur di WSL (atau project dijalankan native di Windows)
- [ ] AutoTrading ON di MT5 Windows
- [ ] `python -c "import MetaTrader5 as mt5; print(mt5.initialize())"` mengembalikan `True`
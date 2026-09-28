# Windows Startup Guide - Telegram Signal EA

Panduan menjalankan project **langsung di Windows (native)** dengan paket resmi `MetaTrader5` — tanpa WSL, tanpa shim, tanpa mt5linux.

## Quick Reference

- **Project Type**: Telegram Signal Processing + MT5 Trading Automation
- **Bahasa**: Python 3.12.7+
- **Platform**: Windows + MetaTrader 5 (paket `MetaTrader5` resmi berjalan native hanya di Windows)
- **Database**: SQLite (file lokal di `data/signal_ea.db`)

---

## 1. Prasyarat Windows

- Windows 10/11
- **MetaTrader 5** sudah terpasang dan **login ke akun broker**
- **Python 3.12.7+** (unduh dari python.org, centang **Add python.exe to PATH**)
- Akun Telegram + `api_id`/`api_hash` (dari my.telegram.org) dan API key OpenAI

Pastikan **AutoTrading ON** di terminal MT5: `Tools → Options → Expert Advisors → Allow Algo Trading`.

---

## 2. Setup Project

```powershell
cd C:\path\to\telegram-signal-mt5
python -m venv venv
venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
```

`requirements.txt` mem-pin `MetaTrader5==5.0.5200` — instalasi ini **sah di Windows** (paket Python resmi), tidak perlu guci Wine/mt5linux seperti di WSL.

---

## 3. Konfigurasi `.env`

```powershell
copy .env.example .env
notepad .env
```

Isi minimal:

```bash
# Telegram
TELEGRAM_API_ID=12345
TELEGRAM_API_HASH=xxxxxxxxxxxxxxxxxxxxxxxx
PHONE_NUMBER=+62xxxxxxxxxxx
TELEGRAM_GROUPS=@signalgroup1,@signalgroup2

# OpenAI
OPENAI_API_KEY=sk-xxxx
OPENAI_MODEL_VARIANT=gpt-5-mini

# MetaTrader5
MT5_LOGIN=12345678
MT5_PASSWORD=your_mt5_password
MT5_SERVER=Broker-ServerName
MT5_PATH=C:\Program Files\MetaTrader 5\terminal64.exe   # isi bila ingin auto-launch terminal

# Database
DATABASE_URL=sqlite+aiosqlite:///data/signal_ea.db
```

Catatan:
- `MT5_PATH` opsional — diisi ke `terminal64.exe` bila ingin project men-launch terminal MT5 otomatis saat inisialisasi (`src/mt5_executor/connection.py`).
- Seluruh key lengkap tersedia di `.env.example` (rate limit, monitoring, trading parameters).

---

## 4. Setup Sesi Telegram

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

---

## 5. Verifikasi & Menjalankan

```powershell
pytest tests/unit/ -v
```

Cek koneksi MT5 (terminal harus terbuka & login):

```powershell
python -c "import MetaTrader5 as mt5; print(bool(mt5.initialize())); print(mt5.terminal_info()); mt5.shutdown()"
```

Jalankan aplikasi:

```powershell
python main.py
```

CLI: `python cli.py start`, `python cli.py status`, `python cli.py stop`.

---

## 6. Auto-Start di Windows (opsional)

Gunakan **Task Scheduler** (nama task mis. `TelegramSignalEA`):

- `Trigger`: At startup
- `Action`: `C:\path\to\telegram-signal-mt5\venv\Scripts\python.exe main.py`
- `Start in`: `C:\path\to\telegram-signal-mt5`

> Pastikan terminal MT5 + AutoTrading ikut aktif sebelum task berjalan (atur task MT5 dengan trigger login, atau start MT5 manual).

---

## 7. Troubleshooting

| Gejala | Solusi |
|--------|--------|
| `pip install MetaTrader5==5.0.5200` gagal | Pastikan di Windows native (bukan WSL) dan Python 64-bit. |
| `MT5 initialize() failed` / `Not available` | Terminal MT5 belum terbuka/login. Pastikan AutoTrading ON. |
| `ModuleNotFoundError: config.settings` | Pastikan `main.py`/`cli.py` dipanggil dari root project (`Start in` benar di Task Scheduler). |
| `MetaTrader5` error `Fatal` / DLL tidak ditemukan | Terminal MT5 harus versi 64-bit; tutup & buka ulang terminal, lalu jalankan terus `python main.py` dari folder project. |
| `FloodWaitError` dari Telegram | Wajar di awal; sistem punya rate limiter (jeda 1–3 detik). |
| Sesi Telegram corrupt/kedaluwarsa | Hapus `data\telegram.session`, ulangi Langkah 4. |
| `python cli.py status` tidak merespons | Service/process belum jalan; cek log di `logs/app.log` (set `DEV_MODE=true` untuk output console). |

Log aplikasi: `logs/app.log` (berputar, 10MB), `logs/error.log` (5MB), `logs/trades.log` (20MB).

---

## 8. Checklist Selesai

- [ ] Python 3.12 + venv + `pip install -r requirements.txt` berhasil
- [ ] `.env` terisi (Telegram, OpenAI, MT5 + `MT5_PATH` bila perlu)
- [ ] MT5 terminal login + AutoTrading ON
- [ ] Sesi Telegram berhasil dibuat (`data\telegram.session`)
- [ ] `pytest tests/unit/ -v` lolos
- [ ] `python -c "import MetaTrader5 as mt5; print(mt5.initialize())"` mengembalikan `True`
- [ ] `python main.py` berjalan & logs normal
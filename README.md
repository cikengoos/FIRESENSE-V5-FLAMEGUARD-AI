# 🤖 Bot Fleet Manager

A complete IoT system with:
- **Pico W** sensors sending telemetry (temperature, humidity, digital input)
- **Raspberry Pi 5** Flask server with real-time dashboard
- **14-digit protocol**: `SSSS CC AA TTT HHH` (Serial, Status, Digital, Temp, Humidity)

## 🎯 Features

- Real-time bot dashboard with glassmorphic UI
- Add / edit / delete bots from web interface
- Auto-offline detection (15 min no heartbeat)
- 4 status levels: `00` OFFLINE, `01` SAFE, `02` ALERT, `03` DANGER
- Persistent JSON storage
- Runs as systemd service on RPi5

## 🔧 Hardware

| Component | Purpose |
|-----------|---------|
| Raspberry Pi Pico W | Sensor node (WiFi) |
| Raspberry Pi 5 | Flask server + dashboard |
| DHT11 | Temperature & humidity |
| Digital sensor / button | Binary input (GPIO 15) |

## 🚀 Quick Start

### 1. Setup Raspberry Pi 5 Server

```bash
git clone https://github.com/YOUR_USERNAME/bot-fleet-manager.git
cd bot-fleet-manager
chmod +x setup.sh
./setup.sh
```

### 2. Configure Pico W

Edit `pico/main.py`:
```python
SSID = "your_wifi"
PASSWORD = "your_password"
SERVER_URL = "http://YOUR_RPI_IP:5000/api/telemetry/4385"
```

Upload to Pico W using Thonny.

### 3. Access Dashboard

Open browser: `http://YOUR_RPI_IP:5000`

## 📡 14-Digit Protocol

| Position | Field | Digits | Example |
|----------|-------|--------|---------|
| 1–4 | Serial | 4 | `4385` |
| 5–6 | Status Code | 2 | `01` |
| 7–8 | Digital Value | 2 | `00` or `01` |
| 9–11 | Temperature | 3 | `023` (23°C) |
| 12–14 | Humidity | 3 | `065` (65%) |

**Example:** `43850101023065`

## 🎨 Status Codes

| Code | Meaning | Trigger |
|------|---------|---------|
| `00` | OFFLINE | No sensor data |
| `01` | SAFE | Normal operation |
| `02` | ALERT | Temp > 40°C OR Hum < 30% |
| `03` | DANGER | Digital pin HIGH |

## 🛠️ Services

```bash
# Check status
sudo systemctl status botfleet

# Restart
sudo systemctl restart botfleet

# View logs
sudo journalctl -u botfleet -f

# Stop
sudo systemctl stop botfleet
```

## 📝 License

MIT — free to use, modify, share.

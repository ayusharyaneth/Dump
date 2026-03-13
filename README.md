
## 📦 Requirements

- Python 3.11+
- Git
- Linux / macOS / WSL (Windows via WSL recommended)

---

## ⚙️ Installation

### 1. Clone Repository

```bash
git clone https://github.com/ayusharyaneth/dexy.git
cd dexy
```

### 2. Create Virtual Environment

```bash
python3 -m venv venv
source venv/bin/activate
```

### 3. Upgrade pip (Recommended)

```bash
pip install --upgrade pip
```

### 4. Install Dependencies

```bash
pip install -r requirements.txt
```

---

### 5. Environment Configuration

Copy the example environment file:

```bash
cp .env.example .env
```

Edit `.env` and configure:

- `TELEGRAM_BOT_TOKEN`
- `TELEGRAM_CHAT_ID`
- Any additional required keys

If a value exists in both `.env` and `strategy.yaml`, `.env` takes priority.

---

### 6. Strategy Configuration

Edit:

```
strategy.yaml
```

Modify thresholds, filters, or risk parameters according to your trading logic.

---

### 7. Running Dexy

```bash
python3 main.py
```

Dexy will:

- Poll DexScreener  
- Apply strategy filters  
- Send alerts to configured Telegram chat  

---

### 🔄 Updating

```bash
git pull origin main
pip install -r requirements.txt
```

---

## ⚠️ Notes

- Ensure Python 3.11+ is installed  
- Always validate strategy parameters before running in production  
- Never expose your `.env` file publicly  

---

## Configuration

### Environment Variables (.env)

```bash
# Telegram Bot Tokens (required)
SIGNAL_BOT_TOKEN=your_signal_bot_token_here
ALERT_BOT_TOKEN=your_alert_bot_token_here

# Chat IDs (required)
SIGNAL_CHAT_ID=your_signal_chat_id
ALERT_CHAT_ID=your_alert_chat_id
ADMIN_CHAT_ID=your_admin_chat_id

# API Configuration
DEXSCREENER_API_BASE=https://api.dexscreener.com/latest
RPC_ENDPOINT=https://api.mainnet-beta.solana.com

# Polling Intervals
POLL_INTERVAL_SECONDS=30
WATCH_UPDATE_INTERVAL_SECONDS=60
HEALTH_CHECK_INTERVAL_SECONDS=300

# System Limits
MAX_MEMORY_MB=2048
MAX_CPU_PERCENT=80

# Feature Flags
ENABLE_SELF_DEFENSE=true
ENABLE_WATCH_MODE=true
ENABLE_WHALE_DETECTION=true
```

### Strategy Configuration (strategy.yaml)

```yaml
filters:
  stage1:
    min_liquidity_usd: 10000
    min_volume_24h_usd: 5000
    max_token_age_hours: 72
  
  stage2:
    min_buy_ratio: 0.55
    max_price_change_5m: 100

risk_scoring:
  weights:
    liquidity_risk: 0.25
    volume_risk: 0.20
    holder_concentration: 0.20

whale_detection:
  thresholds:
    min_single_buy_usd: 10000
    min_wallet_value_usd: 50000

self_defense:
  activation_thresholds:
    api_error_rate: 0.1
    avg_latency_ms: 5000
    memory_usage_mb: 1800
```

## Usage

### Running the System

```bash
# Activate virtual environment
source venv/bin/activate

# Run main system
python main.py

# Run with specific config
python main.py --config /path/to/strategy.yaml
```

### Running as Service (systemd)

```bash
# Create service file
sudo nano /etc/systemd/system/dex-intel.service
```

```ini
[Unit]
Description=DexScreener Intelligence System
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/dexscreener-intelligence-system
Environment=PATH=/home/ubuntu/dexscreener-intelligence-system/venv/bin
ExecStart=/home/ubuntu/dexscreener-intelligence-system/venv/bin/python main.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable dex-intel
sudo systemctl start dex-intel

# Check status
sudo systemctl status dex-intel
sudo journalctl -u dex-intel -f
```

## Telegram Commands

### Signal Bot Commands

| Command | Description |
|---------|-------------|
| `/start` | Welcome message and bot info |
| `/ping` | System status and health check |
| `/watchlist` | View all watched tokens |
| `/regime` | Current market regime analysis |

---

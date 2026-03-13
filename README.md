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

---


### 6. Running Dexy

```bash
python3 main.py
```

---

### 🔄 Updating

```bash
git pull origin main
pip install -r requirements.txt
```

---

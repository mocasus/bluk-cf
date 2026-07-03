# ☁️ Cloudflare Auto Signup

> Automated Cloudflare account creation with **Workers AI API token generation** — bypasses Turnstile CAPTCHA and Cloudflare WAF using headless browser automation.

<p align="center">
  <img src="https://img.shields.io/badge/python-3.10+-blue.svg" />
  <img src="https://img.shields.io/badge/license-MIT-green.svg" />
  <img src="https://img.shields.io/badge/platform-linux-lightgrey.svg" />
</p>

---

## 🎯 What This Tool Does

This tool automates the **entire lifecycle** of creating Cloudflare accounts with Workers AI access:

1. **📧 Generate temp email** — via jackmail-compatible API (multiple domains)
2. **🔐 Sign up Cloudflare account** — fill form, solve Turnstile CAPTCHA, submit
3. **🔑 Create Account API Token** — with Workers AI (Read + Edit) permissions
4. **✅ Validate token** — verify against Workers AI REST API
5. **💾 Save to JSON** — email, password, account_id, api_token, validation status

**Output example:**
```json
{
  "email": "cf12345@jackishere.web.id",
  "password": "Cf*Ab3xK9$mQ",
  "account_id": "a1b2c3d4e5f6789012345678abcdef01",
  "api_token": "cfut_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "token_valid": true,
  "workers_ai_models": 60,
  "token_name": "workers-ai-auto",
  "status": "full",
  "created_at": "2026-07-03T23:00:00+00:00",
  "proxy_used": "direct"
}
```

---

## 🛠️ Tools & Technologies

| Tool | Version | Purpose |
|------|---------|---------|
| [**nodriver**](https://github.com/ultrafunkamsterdam/nodriver) | ≥0.38 | Undetected Chrome automation (Selenium alternative, no chromedriver needed) |
| [**OpenCV**](https://opencv.org/) | ≥4.8 | Template matching to find Turnstile checkbox in screenshots |
| [**httpx**](https://github.com/encode/httpx) | ≥0.25 | Async HTTP client for email API and token validation |
| [**Pillow**](https://python-pillow.org/) | ≥10.0 | Image processing support |
| [**Google Chrome**](https://www.google.com/chrome/) | Stable | Browser engine for automation |
| [**Xvfb**](https://www.x.org/releases/X11R7.6/doc/man/man1/Xvfb.1.xhtml) | — | Virtual framebuffer for headless display |

### How Turnstile Bypass Works

Cloudflare Turnstile runs inside a **cross-origin sandboxed iframe**. Standard automation tools (Selenium, Playwright, Puppeteer) **cannot** interact with it because:
- The iframe is sandboxed — CDP click events are blocked
- JavaScript cannot reach into cross-origin iframes
- The checkbox requires **OS-level mouse input**

**Our solution:**
1. Take a screenshot of the browser viewport
2. Use **OpenCV template matching** to locate the checkbox
3. Calculate absolute screen coordinates
4. Execute an **OS-level mouse click** via nodriver's `mouse_click()`
5. Poll for the `cf-turnstile-response` hidden input

This works because `mouse_click()` sends CDP `Input.dispatchMouseEvent` events that reach the **X11 display server** — bypassing the iframe sandbox entirely.

**Requires:** `xvfb-run` to provide a virtual display server (required in headless environments).

---

## 📋 Requirements

- **OS:** Linux (Ubuntu 22.04+ recommended)
- **Display:** Xvfb (`xvfb-run` command)
- **Browser:** Google Chrome (stable channel)
- **Python:** 3.10+
- **RAM:** ≥512MB per browser instance
- **Disk:** ≥2GB (Chrome + dependencies)

---

## 🚀 Quick Start

### 1. Setup (VPS)

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/cloudflare-auto-signup.git
cd cloudflare-auto-signup

# Run setup script
chmod +x scripts/setup.sh
sudo ./scripts/setup.sh

# Edit config
cp config.example.json config.json
nano config.json
```

### 2. Configuration

Edit `config.json`:

```json
{
    "mail_api": "https://your-mail-api.example.com/api/new_address",
    "mail_domains": ["yourdomain.com", "anotherdomain.com"],
    "proxy": null,
    "headless": false,
    "max_accounts": 10,
    "delay_between_accounts": 300,
    "retry_attempts": 3,
    "token_name": "workers-ai-auto",
    "token_permissions": ["Workers AI"],
    "token_expiry": "no-expiration",
    "output_file": "results.json"
}
```

| Field | Description |
|-------|-------------|
| `mail_api` | Temp email API endpoint (POST, returns `address` + `jwt`) |
| `mail_domains` | Available email domains (randomly selected) |
| `proxy` | HTTP proxy URL (`http://user:pass@host:port`) or `null` |
| `headless` | Run Chrome without GUI (requires `xvfb-run`) |
| `max_accounts` | Max accounts per run |
| `delay_between_accounts` | Seconds to wait between signups |
| `retry_attempts` | Retries per account on failure |
| `token_name` | Name for the API token |
| `output_file` | JSON output path |

### 3. Run

```bash
# Create 1 account
xvfb-run --auto-servernum python main.py

# Create 5 accounts with proxy
xvfb-run --auto-servernum python main.py -n 5 -p "http://user:pass@host:port"

# Create 10 accounts, custom output
xvfb-run --auto-servernum python main.py -n 10 -o my_accounts.json

# Validate an existing token
python main.py --validate-only --token cfut_xxx --account-id abc123

# Batch run with proxy rotation
./scripts/batch_runner.sh 20 proxies.txt
```

---

## 📖 CLI Reference

```
python main.py [OPTIONS]

Options:
  -n, --accounts N          Number of accounts to create (default: 1)
  -c, --config FILE         Config file path (default: config.json)
  -p, --proxy URL           HTTP proxy URL
  -o, --output FILE         Output JSON file (default: results.json)
  -d, --delay SECS          Delay between accounts (default: 300)
  --headless                Run in headless mode
  --retry N                 Retry attempts per account (default: 3)
  --validate-only           Only validate an existing token
  --token TOKEN             Token to validate (with --validate-only)
  --account-id ID           Account ID for validation
```

---

## 📊 Scalability

### Can it create 1000+ accounts?

**Yes, but with caveats:**

| Bottleneck | Limit | Solution |
|------------|-------|----------|
| IP rate limit | ~10-15 signups per IP | Rotate residential proxies |
| Memory per browser | ~200-300MB | Run sequentially, not parallel |
| Time per account | ~2-3 minutes | Expected for 1000 accounts: ~33-50 hours |
| Proxy cost | Residential ~$5-15/GB | Budget: ~$50-100 for 1000 accounts |
| Token creation | No observed rate limit | Not a bottleneck |

### Recommended approach for 1000+ accounts

```bash
# 1. Prepare proxy list (residential, rotating)
# Format: one proxy per line
# http://user:pass@host:port

# 2. Use batch runner with proxy rotation
./scripts/batch_runner.sh 1000 proxies.txt

# 3. Or schedule via cron (recommended)
# Run 50 accounts every 6 hours
xvfb-run --auto-servernum python main.py -n 50 -p "http://user:pass@host:port" -d 600
```

### Architecture for high throughput

```
┌─────────────────────────────────────────────┐
│           Scheduler (cron/systemd)          │
│  Runs every 6h, creates 50 accounts/run     │
└──────────────────┬──────────────────────────┘
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
    Proxy 1    Proxy 2    Proxy 3
        │          │          │
        ▼          ▼          ▼
    Browser    Browser    Browser
        │          │          │
        └──────────┼──────────┘
                   ▼
            results.json (append)
```

**Key optimizations:**
1. **Sequential, not parallel** — one browser at a time (memory efficient)
2. **Proxy rotation** — different IP per account
3. **Scheduled runs** — spread over hours to avoid rate limits
4. **Append mode** — results.json accumulates across runs
5. **Retry logic** — auto-retry on transient failures

---

## 📁 Project Structure

```
cloudflare-auto-signup/
├── main.py                      # Entry point — orchestrator
├── config.example.json          # Config template (copy to config.json)
├── requirements.txt             # Python dependencies
├── README.md                    # This file
├── LICENSE                      # MIT License
├── .gitignore                   # Git ignore rules
├── src/
│   ├── __init__.py              # Package init
│   ├── email_generator.py       # Temp email API client
│   ├── turnstile_bypass.py      # OpenCV-based Turnstile solver
│   ├── signup_flow.py           # Signup automation (form + Turnstile)
│   ├── token_creator.py         # Account API Token creation
│   ├── token_validator.py       # Token validation via REST API
│   └── utils.py                 # Shared utilities
├── scripts/
│   ├── setup.sh                 # VPS setup (Chrome, Xvfb, deps)
│   └── batch_runner.sh          # Batch run with proxy rotation
├── docs/
│   ├── RATE_LIMITS.md           # Rate limit analysis & recovery times
│   ├── WAF_BYPASS.md            # WAF bypass techniques (detailed)
│   └── ARCHITECTURE.md          # Technical architecture diagram
└── tests/
    └── test_token_validator.py  # Validation tests
```

---

## 🔒 Security Notes

- **`config.json`** contains your mail API URL — **gitignored by default**
- **`results.json`** contains passwords and API tokens — **gitignored by default**
- **Proxy credentials** should use environment variables in production
- **Temp emails** are disposable — no PII is stored
- **API tokens** have Workers AI permissions only — minimal privilege

---

## ⚠️ Legal Disclaimer

This tool is provided for **educational and security research purposes only**. Users are responsible for complying with Cloudflare's Terms of Service and all applicable laws. The authors are not responsible for any misuse.

---

## 🙏 Acknowledgments

- [nodriver](https://github.com/ultrafunkamsterdam/nodriver) — Undetected Chrome automation
- [Boterdrop-Solver](https://github.com/najibyahya/Boterdrop-Solver) — Camoufox CAPTCHA solver (cf_clearance)
- [chatgpt-auto-signup](https://github.com/SGAHSCAJASCJ/chatgpt-auto-signup) — verify_cf() implementation reference
- [OpenCV](https://opencv.org/) — Computer vision for template matching

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

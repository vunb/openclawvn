# OpenClaw Vietnam — Hướng dẫn Docker & Môi trường

> **Phiên bản:** 2026.2.6 | **Ngôn ngữ:** Vietnamese/English

---

## Mục lục

1. [Tổng quan dự án](#1-tổng-quan-dự-án)
2. [Yêu cầu hệ thống](#2-yêu-cầu-hệ-thống)
3. [Chuẩn bị môi trường](#3-chuẩn-bị-môi-trường)
4. [Cấu hình biến môi trường (.env)](#4-cấu-hình-biến-môi-trường-env)
5. [Đóng gói Docker (Build)](#5-đóng-gói-docker-build)
6. [Triển khai với Docker Compose](#6-triển-khai-với-docker-compose)
7. [Onboard & Cấu hình kênh](#7-onboard--cấu-hình-kênh)
8. [Nginx Reverse Proxy (Production)](#8-nginx-reverse-proxy-production)
9. [Kiểm tra & Vận hành](#9-kiểm-tra--vận-hành)
10. [Tham khảo nhanh](#10-tham-khảo-nhanh)

---

## 1. Tổng quan dự án

**OpenClaw Vietnam Edition** là bản fork của [OpenClaw](https://github.com/openclaw/openclaw) — nền tảng AI assistant mã nguồn mở — được tối ưu cho người dùng Việt Nam.

### Kiến trúc tổng quan

```
┌──────────────────────────────────────────────────────┐
│                   Người dùng                          │
│   Telegram Bot │ Zalo OA │ Zalo Personal │ Web UI     │
└────────┬───────┴────┬────┴───────────────┴────────────┘
         │            │              (HTTP / WebSocket)
         ▼            ▼
┌──────────────────────────────────────────────────────┐
│              OpenClaw Gateway (Docker)                │
│                                                       │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Telegram  │  │   Zalo OA    │  │  ZaloUser    │  │
│  │  Extension │  │  Extension   │  │  Extension   │  │
│  └─────┬──────┘  └──────┬───────┘  └──────┬───────┘  │
│        └────────────────┼─────────────────┘          │
│                         │                             │
│  ┌──────────────────────▼──────────────────────────┐  │
│  │   AI Agent (Claude Sonnet / Haiku)               │  │
│  │   + Smart Routing + Cost Transparency            │  │
│  └──────────────────────┬──────────────────────────┘  │
│                         │                             │
│  ┌──────────────────────▼──────────────────────────┐  │
│  │   Skills: weather, summarize, coding-agent,      │  │
│  │   vibecode-build, openai-whisper, image-gen ...  │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │
│  │  Memory      │  │  TTS         │  │  Browser    │  │
│  │  (LanceDB)   │  │  (Edge TTS)  │  │  (Playwright│  │
│  └──────────────┘  └──────────────┘  └─────────────┘  │
└──────────────────────────────────────────────────────┘
         │
         ▼ (Anthropic API / OpenAI API)
┌──────────────────────────────────────────────────────┐
│                  AI Providers                         │
│  Anthropic Claude (chính) │ OpenAI (TTS, embeddings)  │
└──────────────────────────────────────────────────────┘
```

### Tính năng nổi bật (Vietnam Edition)

| Tính năng | Mô tả |
|-----------|-------|
| **Zalo OA** | Tích hợp Zalo Official Account Bot API |
| **Zalo Personal** | Tích hợp tài khoản Zalo cá nhân |
| **Telegram** | Bot Telegram đầy đủ tính năng |
| **TTS tiếng Việt** | Edge TTS miễn phí với giọng `vi-VN-HoaiMyNeural` |
| **Smart Routing** | Tự động chọn model AI phù hợp theo loại task |
| **Cost Transparency** | Ước tính và quản lý chi phí API theo ngân sách |
| **Memory (LanceDB)** | Nhớ ngữ cảnh dài hạn bằng vector store |
| **Eldercare (BÀ NỘI CARE)** | Module chăm sóc người cao tuổi qua Home Assistant |
| **Web UI** | Giao diện quản lý song ngữ Việt/Anh |

---

## 2. Yêu cầu hệ thống

| Thành phần | Tối thiểu | Khuyến nghị |
|-----------|-----------|-------------|
| CPU | 2 core | 4 core |
| RAM | 2 GB | 4 GB |
| Disk | 5 GB | 10 GB |
| OS | Linux/macOS/Windows (Docker) | Ubuntu 22.04 LTS |
| Docker | 24+ | 27+ |
| Docker Compose | v2.20+ | v2.27+ |
| Node.js (dev) | 22+ | 22 LTS |

> **Lưu ý:** Khi chạy bằng Docker, bạn **không cần** cài Node.js trực tiếp trên host — tất cả đều chạy bên trong container.

---

## 3. Chuẩn bị môi trường

### 3.1 Clone repository

```bash
git clone https://github.com/vunb/openclawvn.git
cd openclawvn
```

### 3.2 Kiểm tra Docker

```bash
docker --version          # >= 24.0
docker compose version    # >= 2.20
```

Nếu chưa có, cài Docker Desktop (macOS/Windows) hoặc Docker Engine (Linux):

```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com | bash
sudo usermod -aG docker $USER   # Thêm user vào group docker
newgrp docker                   # Áp dụng ngay
```

### 3.3 Tạo thư mục data

```bash
mkdir -p ~/.openclaw/workspace
```

---

## 4. Cấu hình biến môi trường (.env)

### 4.1 Tạo file .env từ template

```bash
cp .env.vietnam.example .env
```

### 4.2 Điền thông tin bắt buộc

Mở `.env` bằng editor và điền ít nhất các mục sau:

```dotenv
# ── AI Provider (BẮT BUỘC) ──────────────────────────────────
ANTHROPIC_API_KEY=sk-ant-api03-<lấy từ console.anthropic.com>

# ── Messaging (ít nhất 1 kênh) ──────────────────────────────
# Telegram: lấy token từ @BotFather
TELEGRAM_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz

# Zalo OA: lấy từ developers.zalo.me
ZALO_APP_ID=your_zalo_app_id
ZALO_SECRET_KEY=your_zalo_secret_key
```

### 4.3 Các biến môi trường tuỳ chọn

```dotenv
# OpenAI (cho TTS chất lượng cao, embeddings)
OPENAI_API_KEY=sk-...

# ElevenLabs (TTS tiếng Việt chất lượng cao)
ELEVENLABS_API_KEY=xi_...

# Brave Search (tìm kiếm web)
BRAVE_API_KEY=BSA...

# Zalo Personal account
ZALO_BOT_PHONE=+84xxxxxxxxx
```

### 4.4 Sinh gateway token tự động

Nếu để `OPENCLAW_GATEWAY_TOKEN=` trống, `docker-setup.sh` sẽ tự sinh token ngẫu nhiên và ghi vào `.env`. Bạn cũng có thể tự sinh:

```bash
openssl rand -hex 32
```

---

## 5. Đóng gói Docker (Build)

### 5.1 Build image cơ bản

```bash
docker build -t openclaw:local .
```

### 5.2 Build với packages bổ sung (ví dụ: ffmpeg cho media)

```bash
docker build \
  --build-arg OPENCLAW_DOCKER_APT_PACKAGES="ffmpeg" \
  -t openclaw:local \
  .
```

### 5.3 Dùng script tự động (khuyến nghị)

Script `docker-setup.sh` thực hiện toàn bộ quy trình: build image → onboard → start gateway:

```bash
bash docker-setup.sh
```

Script sẽ:
1. Kiểm tra Docker có sẵn
2. Tạo thư mục config/workspace nếu chưa có
3. Sinh `OPENCLAW_GATEWAY_TOKEN` nếu chưa có
4. Ghi các biến vào `.env`
5. Build Docker image
6. Chạy wizard onboard tương tác
7. Khởi động gateway

---

## 6. Triển khai với Docker Compose

### 6.1 Khởi động gateway (Vietnam edition)

```bash
# Dùng file compose riêng cho Vietnam
docker compose -f docker-compose.vietnam.yml up -d openclaw-gateway
```

### 6.2 Xem logs

```bash
docker compose -f docker-compose.vietnam.yml logs -f openclaw-gateway
```

### 6.3 Dừng / Khởi động lại

```bash
docker compose -f docker-compose.vietnam.yml stop
docker compose -f docker-compose.vietnam.yml restart openclaw-gateway
```

### 6.4 Cập nhật image và restart

```bash
# Build image mới
docker build -t openclaw:local .

# Recreate container với image mới
docker compose -f docker-compose.vietnam.yml up -d --force-recreate openclaw-gateway
```

### 6.5 Chạy lệnh CLI trong container

```bash
# Kiểm tra trạng thái kênh
docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli \
  channels status --probe

# Xem cấu hình
docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli \
  config get

# Xem usage / chi phí
docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli \
  usage --cost
```

### 6.6 Cấu trúc thư mục dữ liệu (trên HOST)

```
~/.openclaw/                    ← OPENCLAW_CONFIG_DIR
├── config.json                 # Cấu hình gateway
├── credentials/                # API keys (encrypted)
├── agents/
│   └── main/
│       └── sessions/           # Lịch sử hội thoại
├── memory/                     # LanceDB vector store
├── plugins/                    # Extensions đã cài
│   ├── zalo/
│   └── zalouser/
└── workspace/                  ← OPENCLAW_WORKSPACE_DIR
    └── skills/                 # Skill files
```

> **Dữ liệu được mount vào container** — không bị mất khi `docker compose down && up`.

---

## 7. Onboard & Cấu hình kênh

### 7.1 Onboard lần đầu

```bash
docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli \
  onboard --no-install-daemon
```

Khi được hỏi, chọn:
- **Gateway bind:** `lan`
- **Gateway auth:** `token`
- **Gateway token:** _(dán token từ `.env`)_
- **Install daemon:** `No` (trong Docker, không cần)

### 7.2 Thêm Telegram bot

```bash
docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli \
  providers add --provider telegram --token "$TELEGRAM_BOT_TOKEN"
```

### 7.3 Thêm Zalo OA

```bash
docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli \
  providers add --provider zalo \
    --app-id "$ZALO_APP_ID" \
    --secret-key "$ZALO_SECRET_KEY"
```

### 7.4 Áp dụng config Vietnam

```bash
# Copy config Vietnam vào thư mục config
cp openclaw.config.vietnam.json ~/.openclaw/config.json
```

Hoặc qua CLI:

```bash
docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli \
  config set agents.defaults.userTimezone "Asia/Ho_Chi_Minh"

docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli \
  config set messages.tts.edge.voice "vi-VN-HoaiMyNeural"
```

### 7.5 Cài extensions Zalo

```bash
docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli \
  plugins install zalo

docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli \
  plugins install zalouser
```

---

## 8. Nginx Reverse Proxy (Production)

Để expose gateway ra internet (cần thiết cho Zalo webhook HTTPS):

```nginx
# /etc/nginx/sites-available/openclaw-vietnam
server {
    listen 80;
    server_name your-domain.com;

    # Redirect HTTP → HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate     /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # Gateway proxy
    location / {
        proxy_pass         http://127.0.0.1:18789;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
    }

    # Zalo webhook endpoint
    location /webhook/zalo {
        proxy_pass       http://127.0.0.1:18789;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
# Enable site và lấy SSL với Let's Encrypt
sudo ln -s /etc/nginx/sites-available/openclaw-vietnam /etc/nginx/sites-enabled/
sudo certbot --nginx -d your-domain.com
sudo nginx -t && sudo systemctl reload nginx
```

---

## 9. Kiểm tra & Vận hành

### 9.1 Health check

```bash
# HTTP health check
curl http://localhost:18789/health

# Qua Docker
docker compose -f docker-compose.vietnam.yml exec openclaw-gateway \
  node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### 9.2 Kiểm tra kênh

```bash
docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli \
  channels status --probe
```

### 9.3 Theo dõi logs thực tế

```bash
# Logs gateway (follow)
docker compose -f docker-compose.vietnam.yml logs -f openclaw-gateway

# Lọc lỗi
docker compose -f docker-compose.vietnam.yml logs openclaw-gateway 2>&1 | grep -i error
```

### 9.4 Xử lý sự cố phổ biến

| Vấn đề | Nguyên nhân | Giải pháp |
|--------|------------|-----------|
| Gateway không start | Token chưa set | Kiểm tra `OPENCLAW_GATEWAY_TOKEN` trong `.env` |
| Telegram không phản hồi | Bot token sai | Chạy `channels status telegram` |
| Zalo webhook lỗi | Chưa có HTTPS | Cấu hình Nginx + SSL (xem mục 8) |
| TTS không hoạt động | Không có kết nối internet | Kiểm tra mạng container: `docker exec openclaw-gateway curl -s https://edge.microsoft.com` |
| Memory không lưu | Thiếu OPENAI_API_KEY | Set `OPENAI_API_KEY` trong `.env` |
| Container exit ngay | Lỗi build hoặc config | Xem `docker compose logs openclaw-gateway` |

---

## 10. Tham khảo nhanh

```bash
# ─── Build ────────────────────────────────────────────────────
docker build -t openclaw:local .

# ─── Start / Stop ─────────────────────────────────────────────
docker compose -f docker-compose.vietnam.yml up -d openclaw-gateway
docker compose -f docker-compose.vietnam.yml stop
docker compose -f docker-compose.vietnam.yml restart openclaw-gateway

# ─── Logs ─────────────────────────────────────────────────────
docker compose -f docker-compose.vietnam.yml logs -f openclaw-gateway

# ─── CLI ──────────────────────────────────────────────────────
# (thay <cmd> bằng lệnh cần chạy)
docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli <cmd>

# ─── Ví dụ CLI ────────────────────────────────────────────────
docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli channels status --probe
docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli config get
docker compose -f docker-compose.vietnam.yml run --rm openclaw-cli usage --cost

# ─── Onboard lần đầu ─────────────────────────────────────────
bash docker-setup.sh

# ─── Config Vietnam ───────────────────────────────────────────
cp openclaw.config.vietnam.json ~/.openclaw/config.json

# ─── Xem docs liên quan ───────────────────────────────────────
# README.vietnam.md         — Tổng quan tính năng VN
# DEPLOY_VIETNAM.md         — Hướng dẫn deploy (bare-metal + systemd)
# ELDERCARE-SETUP.md        — Cài module BÀ NỘI CARE
# .env.vietnam.example      — Template biến môi trường Docker
# docker-compose.vietnam.yml — Docker Compose Vietnam edition
```

---

## Chi phí vận hành ước tính

| Thành phần | Chi phí/tháng |
|------------|---------------|
| Anthropic API (nhẹ) | $10–30 |
| Anthropic API (nặng) | $50–150 |
| VPS 2 GB RAM (Docker) | $5–10 |
| Domain + SSL (Let's Encrypt) | $0–5 |
| Zalo OA | Miễn phí |
| Telegram Bot | Miễn phí |
| Edge TTS | Miễn phí |
| **Tổng (nhẹ)** | **$15–45/tháng** |
| **Tổng (nặng)** | **$55–165/tháng** |

---

## Tài liệu liên quan

- [README.vietnam.md](./README.vietnam.md) — Tổng quan Vietnam Edition
- [DEPLOY_VIETNAM.md](./DEPLOY_VIETNAM.md) — Deploy bare-metal + systemd
- [ELDERCARE-SETUP.md](./ELDERCARE-SETUP.md) — Module BÀ NỘI CARE
- [ELDERCARE-README.md](./ELDERCARE-README.md) — Kiến trúc module eldercare
- [.env.vietnam.example](./.env.vietnam.example) — Template biến môi trường
- [docker-compose.vietnam.yml](./docker-compose.vietnam.yml) — Docker Compose Vietnam
- [docs/](./docs/) — Tài liệu đầy đủ

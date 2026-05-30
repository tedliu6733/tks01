# Home Assistant + 語音助手 Docker Compose 部署指南

## 系統需求

- Linux 主機（Ubuntu/Debian 建議）
- Docker Engine + Docker Compose v2
- 4+ 核心 CPU、8GB+ RAM
- 網路連線（首次下載映像及模型）

## 架構概覽

```
┌─────────────────────────────────────────────────┐
│              Docker Compose Stack                │
│                                                  │
│  ┌──────────────────┐   ┌───────────────────┐   │
│  │  Home Assistant   │   │  Whisper (STT)    │   │
│  │  Port: 8123       │   │  Port: 10300      │   │
│  │  network: host    │   │  Wyoming protocol │   │
│  └──────────────────┘   └───────────────────┘   │
│                                                  │
│  ┌──────────────────┐   ┌───────────────────┐   │
│  │  Piper (TTS)      │   │  openWakeWord      │   │
│  │  Port: 10200      │   │  Port: 10400      │   │
│  │  Wyoming protocol │   │  Wyoming protocol │   │
│  └──────────────────┘   └───────────────────┘   │
└─────────────────────────────────────────────────┘
```

| 容器 | 映像 | Port | 用途 |
|------|------|------|------|
| homeassistant | `ghcr.io/home-assistant/home-assistant:stable` | 8123 | HA 核心 + Web UI |
| whisper | `rhasspy/wyoming-whisper` | 10300 | 語音轉文字 (STT) |
| piper | `rhasspy/wyoming-piper` | 10200 | 文字轉語音 (TTS) |
| openwakeword | `rhasspy/wyoming-openwakeword` | 10400 | 喚醒詞偵測 |

---

## 第一步：安裝 Docker

如果尚未安裝 Docker，執行以下指令：

```bash
# 安裝 Docker
curl -fsSL https://get.docker.com | sh

# 將當前使用者加入 docker 群組（免 sudo）
sudo usermod -aG docker $USER

# 重新登入後驗證
docker --version
docker compose version
```

## 第二步：建立專案目錄

```bash
mkdir -p ~/docker-HomeAsistant
cd ~/docker-HomeAsistant
```

## 第三步：建立環境變數檔 `.env`

```bash
cat > .env << 'EOF'
# Timezone — 依照你的地區修改
TZ=Asia/Taipei

# Whisper STT config
WHISPER_MODEL=base
WHISPER_LANGUAGE=zh

# Piper TTS voices
PIPER_VOICE=zh_CN-huayan-medium

# openWakeWord
OPENWAKEWORD_PRELOAD_MODEL=ok_nabu
EOF
```

### 可調整參數說明

| 變數 | 說明 | 可選值 |
|------|------|--------|
| `TZ` | 時區 | `Asia/Taipei`、`Asia/Tokyo`、`America/New_York` 等 |
| `WHISPER_MODEL` | Whisper 模型大小 | `tiny`（最快）、`base`（平衡）、`small`（較準） |
| `WHISPER_LANGUAGE` | 語音辨識語言 | `zh`（中文）、`en`（英文） |
| `PIPER_VOICE` | TTS 語音模型 | `zh_CN-huayan-medium`、`en_US-lessac-medium` |
| `OPENWAKEWORD_PRELOAD_MODEL` | 喚醒詞 | `ok_nabu`、`hey_jarvis`、`alexa` |

## 第四步：建立 `docker-compose.yml`

```bash
cat > docker-compose.yml << 'EOF'
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:stable
    volumes:
      - ./homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    environment:
      - TZ=${TZ}
    restart: unless-stopped
    network_mode: host
    depends_on:
      - whisper
      - piper
      - openwakeword

  whisper:
    container_name: whisper
    image: rhasspy/wyoming-whisper
    volumes:
      - ./whisper/data:/data
    ports:
      - "10300:10300"
    environment:
      - TZ=${TZ}
    command: --model ${WHISPER_MODEL} --language ${WHISPER_LANGUAGE}
    restart: unless-stopped

  piper:
    container_name: piper
    image: rhasspy/wyoming-piper
    volumes:
      - ./piper/data:/data
    ports:
      - "10200:10200"
    environment:
      - TZ=${TZ}
    command: --voice ${PIPER_VOICE}
    restart: unless-stopped

  openwakeword:
    container_name: openwakeword
    image: rhasspy/wyoming-openwakeword
    volumes:
      - ./openwakeword/data:/custom
    ports:
      - "10400:10400"
    environment:
      - TZ=${TZ}
    command: --preload-model ${OPENWAKEWORD_PRELOAD_MODEL}
    restart: unless-stopped
EOF
```

## 第五步：建立資料目錄

```bash
mkdir -p homeassistant/config whisper/data piper/data openwakeword/data
```

## 第六步：啟動服務

```bash
# 拉取映像
docker compose pull

# 背景啟動所有服務
docker compose up -d
```

首次啟動會自動下載模型檔案：
- Whisper base 模型：約 150MB
- Piper 語音模型：約 30-60MB

請等待 2-5 分鐘讓模型下載完成。

## 第七步：驗證服務狀態

```bash
# 檢查所有容器是否正常運行
docker compose ps

# 檢查各語音服務日誌（應顯示 Ready）
docker compose logs whisper --tail 5
docker compose logs piper --tail 5
docker compose logs openwakeword --tail 5
```

所有服務應顯示 `Up` 狀態，且日誌中出現 `Ready`。

## 第八步：設定 Home Assistant

打開瀏覽器訪問 **http://localhost:8123**（或 `http://<主機IP>:8123`）。

### 8.1 完成初始設定

首次訪問時會出現設定精靈：
1. 建立管理員帳號
2. 設定住家名稱與地理位置
3. 跳過自動發現的整合（之後再設定）

### 8.2 新增 Wyoming 語音整合

路徑：**Settings → Devices & Services → Add Integration**

搜尋 **Wyoming**，分別新增三個服務：

| 服務 | Host | Port |
|------|------|------|
| Whisper (STT) | `localhost` | `10300` |
| Piper (TTS) | `localhost` | `10200` |
| openWakeWord | `localhost` | `10400` |

### 8.3 建立語音助手管線

路徑：**Settings → Voice assistants → Add assistant**

| 設定項目 | 選擇 |
|---------|------|
| Name | `Home Voice`（自訂） |
| Conversation agent | Home Assistant |
| Speech-to-text | whisper (faster-whisper) |
| Text-to-speech | piper |
| Wake word | openwakeword → `ok nabu` |

### 8.4 測試語音助手

點擊 HA 介面右上角的麥克風圖示，說出 "ok nabu" 後下達語音指令測試。

---

## 常用維護指令

```bash
# 查看服務狀態
docker compose ps

# 查看特定服務日誌
docker compose logs <服務名> --tail 50

# 重啟所有服務
docker compose restart

# 重啟單一服務
docker compose restart whisper

# 停止所有服務
docker compose down

# 更新映像並重啟
docker compose pull && docker compose up -d

# 查看資源使用情況
docker stats --no-stream
```

## 疑難排解

### Whisper 啟動出現 float16 警告
```
WARNING: float16 is not supported on CPU, using float32 instead
```
這是正常現象，CPU 模式下會自動降級為 float32，不影響功能。

### HA 無法發現區網裝置
確認 Home Assistant 使用 `network_mode: host`，這樣才能透過 mDNS/SSDP 發現區網中的 IoT 裝置。

### 語音辨識不準確
可以在 `.env` 中將 `WHISPER_MODEL` 從 `base` 升級為 `small`，準確度更高但推理速度會變慢：
```bash
# 修改 .env
WHISPER_MODEL=small

# 重啟 whisper 服務
docker compose restart whisper
```

### 容器持續重啟
```bash
# 查看失敗日誌
docker compose logs <服務名> --tail 100

# 檢查磁碟空間
df -h
```

## 未來擴展

- **Wyoming Satellite**：接入 ESP32-S3 或 Raspberry Pi 做遠端語音節點
- **自訂喚醒詞**：訓練自己的喚醒詞模型
- **升級 Whisper 模型**：`base` → `small` → `medium`
- **本地 LLM**：加入 Ollama 等本地大語言模型做智慧對話

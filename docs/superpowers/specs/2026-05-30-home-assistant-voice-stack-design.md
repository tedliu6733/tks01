# Home Assistant + 語音助手 Docker Compose 部署設計

## 概述

使用 Docker Compose 部署 Home Assistant 及完整語音助手堆疊（STT/TTS/喚醒詞），全部本地運行，支援中英雙語。目標是個人智慧家庭用途，未來計劃接入 Wyoming Satellite（ESP32-S3 / Raspberry Pi）。

## 硬體前提

- 一般桌機或 NAS，4+ 核心 CPU、8GB+ RAM
- 無獨立 GPU，所有推理使用 CPU
- 預估資源佔用：整體約 2-3GB RAM

## 容器架構

採用 Wyoming 協議原生整合，四個獨立容器：

```
┌─────────────────────────────────────────────────┐
│              Docker Compose Stack                │
│                                                  │
│  ┌──────────────────┐   ┌───────────────────┐   │
│  │  Home Assistant   │   │  Whisper (STT)    │   │
│  │  Port: 8123       │◄──│  Port: 10300      │   │
│  │  network: host    │   │  Wyoming protocol │   │
│  └──────────────────┘   └───────────────────┘   │
│           │                                      │
│           │              ┌───────────────────┐   │
│           ├─────────────►│  Piper (TTS)      │   │
│           │              │  Port: 10200      │   │
│           │              │  Wyoming protocol │   │
│           │              └───────────────────┘   │
│           │                                      │
│           │              ┌───────────────────┐   │
│           └─────────────►│  openWakeWord      │   │
│                          │  Port: 10400      │   │
│                          │  Wyoming protocol │   │
│                          └───────────────────┘   │
└─────────────────────────────────────────────────┘
```

### 容器細節

| 容器 | 映像 | Port | 用途 |
|------|------|------|------|
| homeassistant | `ghcr.io/home-assistant/home-assistant:stable` | 8123 | HA 核心 + Web UI |
| whisper | `rhasspy/wyoming-whisper` | 10300 | 語音轉文字 (STT) |
| piper | `rhasspy/wyoming-piper` | 10200 | 文字轉語音 (TTS) |
| openwakeword | `rhasspy/wyoming-openwakeword` | 10400 | 喚醒詞偵測 |

### 網路配置

- Home Assistant 使用 `network_mode: host`，以便透過 mDNS/SSDP 自動發現區域網路中的 IoT 裝置
- Whisper、Piper、openWakeWord 使用 Docker bridge network，對外曝露各自的 Wyoming 協議端口
- HA 透過 `host.docker.internal` 或容器名稱 + 端口存取語音服務

## 資料持久化

| Volume 路徑 | 容器掛載點 | 內容 |
|-------------|-----------|------|
| `./homeassistant/config` | `/config` | HA 配置、自動化、資料庫 |
| `./whisper/data` | `/data` | Whisper 模型快取 |
| `./piper/data` | `/data` | Piper 語音模型快取 |
| `./openwakeword/data` | `/custom` | openWakeWord 自訂模型 |

## 語音模型配置

### Whisper (STT)
- 模型：`base`（平衡準確度與 CPU 速度，約 150MB）
- 語言：`zh`（中文優先，也能辨識英文）
- 可升級至 `small` 模型以提高準確度（需更多 CPU 時間）

### Piper (TTS)
- 中文語音：`zh_CN-huayan-medium`
- 英文語音：`en_US-lessac-medium`
- 每個語音模型約 30-60MB

### openWakeWord
- 預設喚醒詞：`ok_nabu`
- 未來可訓練自訂喚醒詞

## 啟動與重啟策略

- 所有容器設為 `restart: unless-stopped`
- HA 設定 `depends_on` 三個語音服務容器
- 首次啟動時會自動下載模型，需等待數分鐘

## 部署後手動設定（HA UI）

Docker Compose 啟動後，需在 HA Web UI 中完成以下步驟：

1. **新增 Wyoming 整合**：Settings → Devices & Services → Add Integration → Wyoming
   - Whisper：`whisper` / port `10300`（若 HA 用 host network 則填 `localhost`）
   - Piper：`piper` / port `10200`
   - openWakeWord：`openwakeword` / port `10400`

2. **設定語音助手管線**：Settings → Voice assistants
   - 建立新的 Assist pipeline
   - STT 選擇 Whisper
   - TTS 選擇 Piper
   - 喚醒詞選擇 openWakeWord

3. **測試語音**：在 HA UI 右上角點選語音助手圖示測試

## 未來擴展

- 接入 Wyoming Satellite（ESP32-S3 / Raspberry Pi）做遠端語音節點
- 自訂喚醒詞訓練
- 升級 Whisper 模型（base → small → medium）
- 加入本地 LLM 做更智慧的對話理解

## 產出檔案

| 檔案 | 說明 |
|------|------|
| `docker-compose.yml` | Docker Compose 主配置 |
| `.env` | 環境變數（時區等） |

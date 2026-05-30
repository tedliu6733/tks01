# Home Assistant + 語音助手堆疊 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy Home Assistant with a full local voice stack (Whisper STT, Piper TTS, openWakeWord) via Docker Compose, supporting Chinese and English.

**Architecture:** Four containers orchestrated by Docker Compose — HA Core on host network for IoT discovery, three Wyoming-protocol voice services on a shared bridge network with ports exposed to host. HA connects to voice services via `localhost:<port>`.

**Tech Stack:** Docker Compose, Home Assistant Core, Wyoming Whisper, Wyoming Piper, Wyoming openWakeWord

---

## File Structure

| File | Purpose |
|------|---------|
| `.env` | Environment variables (timezone) |
| `docker-compose.yml` | All four service definitions, volumes, network config |

---

### Task 1: Create .env file

**Files:**
- Create: `.env`

- [ ] **Step 1: Create the .env file**

```env
# Timezone — change to match your location
TZ=Asia/Taipei

# Whisper STT config
WHISPER_MODEL=base
WHISPER_LANGUAGE=zh

# Piper TTS voices
PIPER_VOICE=zh_CN-huayan-medium

# openWakeWord
OPENWAKEWORD_PRELOAD_MODEL=ok_nabu
```

- [ ] **Step 2: Verify the file**

Run: `cat .env`
Expected: The env file contents shown above.

- [ ] **Step 3: Commit**

```bash
git add .env
git commit -m "chore: add .env with timezone and voice model config"
```

---

### Task 2: Create docker-compose.yml with Home Assistant service

**Files:**
- Create: `docker-compose.yml`

- [ ] **Step 1: Create docker-compose.yml with HA service**

```yaml
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
```

- [ ] **Step 2: Validate the compose file syntax**

Run: `docker compose -f docker-compose.yml config --services`
Expected: Error about missing whisper/piper/openwakeword services (they don't exist yet), but the YAML parses. This is expected — we add the remaining services next.

---

### Task 3: Add Whisper (STT) service to docker-compose.yml

**Files:**
- Modify: `docker-compose.yml`

- [ ] **Step 1: Add whisper service after the homeassistant service**

Append the following service definition inside the `services:` block:

```yaml
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
```

- [ ] **Step 2: Validate syntax**

Run: `docker compose -f docker-compose.yml config --services`
Expected: Still errors about missing piper/openwakeword, but `whisper` and `homeassistant` are listed.

---

### Task 4: Add Piper (TTS) service to docker-compose.yml

**Files:**
- Modify: `docker-compose.yml`

- [ ] **Step 1: Add piper service**

Append the following service definition inside the `services:` block:

```yaml
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
```

- [ ] **Step 2: Validate syntax**

Run: `docker compose -f docker-compose.yml config --services`
Expected: Still errors about missing openwakeword, but `homeassistant`, `whisper`, `piper` are listed.

---

### Task 5: Add openWakeWord service to docker-compose.yml

**Files:**
- Modify: `docker-compose.yml`

- [ ] **Step 1: Add openwakeword service**

Append the following service definition inside the `services:` block:

```yaml
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
```

- [ ] **Step 2: Validate complete compose file**

Run: `docker compose -f docker-compose.yml config --services`
Expected output (all four services, no errors):
```
homeassistant
openwakeword
piper
whisper
```

- [ ] **Step 3: Commit the complete docker-compose.yml**

```bash
git add docker-compose.yml
git commit -m "feat: add docker-compose with HA, Whisper, Piper, openWakeWord"
```

---

### Task 6: Create volume directories and start the stack

**Files:**
- Create directories: `homeassistant/config/`, `whisper/data/`, `piper/data/`, `openwakeword/data/`

- [ ] **Step 1: Create persistent volume directories**

```bash
mkdir -p homeassistant/config whisper/data piper/data openwakeword/data
```

- [ ] **Step 2: Add .gitkeep files so empty dirs are tracked**

```bash
touch homeassistant/config/.gitkeep whisper/data/.gitkeep piper/data/.gitkeep openwakeword/data/.gitkeep
```

- [ ] **Step 3: Create .gitignore to exclude runtime data but keep structure**

Create `.gitignore` at project root:

```gitignore
# Home Assistant runtime data
homeassistant/config/*
!homeassistant/config/.gitkeep

# Model caches
whisper/data/*
!whisper/data/.gitkeep
piper/data/*
!piper/data/.gitkeep
openwakeword/data/*
!openwakeword/data/.gitkeep
```

- [ ] **Step 4: Commit directory structure**

```bash
git add .gitignore homeassistant/config/.gitkeep whisper/data/.gitkeep piper/data/.gitkeep openwakeword/data/.gitkeep
git commit -m "chore: add volume directories and gitignore for runtime data"
```

- [ ] **Step 5: Pull images and start the stack**

```bash
docker compose pull
docker compose up -d
```

Expected: All four containers start. First run will take several minutes as Whisper downloads the `base` model (~150MB) and Piper downloads voice models (~30-60MB each).

- [ ] **Step 6: Verify all containers are running**

Run: `docker compose ps`
Expected: All four containers show status `Up` or `running`.

- [ ] **Step 7: Verify HA is accessible**

Open `http://localhost:8123` in a browser. Expected: Home Assistant onboarding wizard (first run) or login page.

- [ ] **Step 8: Verify Wyoming services are listening**

```bash
docker compose logs whisper --tail 5
docker compose logs piper --tail 5
docker compose logs openwakeword --tail 5
```

Expected: Each service shows it is ready and listening on its respective port (10300, 10200, 10400).

---

### Task 7: Configure Wyoming integrations in Home Assistant UI

This task is manual — performed in the browser at `http://localhost:8123`.

- [ ] **Step 1: Complete HA onboarding** (first run only)

Create admin account, set location, skip any integrations discovery for now.

- [ ] **Step 2: Add Wyoming integration for Whisper (STT)**

Navigate: Settings → Devices & Services → Add Integration → search "Wyoming"
- Host: `localhost`
- Port: `10300`

Expected: Integration added, shows "Wyoming" with Whisper device.

- [ ] **Step 3: Add Wyoming integration for Piper (TTS)**

Add Integration → Wyoming again:
- Host: `localhost`
- Port: `10200`

Expected: Integration added, shows Piper device.

- [ ] **Step 4: Add Wyoming integration for openWakeWord**

Add Integration → Wyoming again:
- Host: `localhost`
- Port: `10400`

Expected: Integration added, shows openWakeWord device.

- [ ] **Step 5: Configure Voice Assistant pipeline**

Navigate: Settings → Voice assistants → Add assistant
- Name: `Home Voice`
- Conversation agent: Home Assistant
- Speech-to-text: `whisper` (faster-whisper)
- Text-to-speech: `piper`
- Wake word: `openwakeword` → select `ok nabu`

- [ ] **Step 6: Test the voice assistant**

Click the voice assistant icon (microphone) in the top-right of the HA UI. Say "ok nabu" followed by a command like "what time is it". Expected: Whisper transcribes speech, HA processes the command, Piper speaks the response.

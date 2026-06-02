# Hermes-ESP32 — Project Plan & Design Notes

> A from-scratch port of the [NousResearch Hermes Agent](https://github.com/NousResearch/hermes-agent)
> onto the ESP32 microcontroller. Built in the **Hermes way** (its loop, its
> concepts, its persona/skills/memory model), using existing embedded AI-agent
> projects (`esp-claw`, `zclaw`, etc.) **only as engineering reference** for the
> low-level basics — not as a codebase to copy.
>
> **Status:** Planning / pre-firmware. This document is the handoff for starting
> real work in Claude Code.
> **Last updated:** 2026-06-02
> **Branch:** `claude/hermes-esp32-analysis-XzxzX`

---

## 0. Read this first — the one mental model that matters

Hermes (like Claude Code and Codex CLI) is **NOT an AI model**. It is a ~30,000-line
Python **harness** — plumbing that:

1. builds a request (`messages[]` + `tools[]`),
2. POSTs it to a **cloud LLM** (OpenAI/Anthropic/OpenRouter/Nous Portal — *the
   intelligence lives here*),
3. parses the `tool_calls` the model returns,
4. **executes those tools locally**,
5. feeds the results back and loops until the model is done.

The "intelligence" is always a remote API. The harness is just the loop + the tools.

**That is the only part we port.** We cannot (and do not try to) run an LLM on the
ESP32. We run the *loop* on the ESP32 and let the cloud do the thinking.

### Why ESP32 and not "Claude Code on a chip"
- **ESP32 is a microcontroller (MCU), not a computer.** ~512 KB SRAM (+ a few MB
  PSRAM), no MMU, runs **FreeRTOS** (a real-time OS with threads/timers) — **not
  Linux**. There is no shell, no `fork`/`exec`, no filesystem of executable programs.
- Therefore the ESP32 agent's tools cannot be `bash`/`edit_file`/`git` (those need a
  full OS). They are **hardware tools** instead: GPIO, I2C, sensors, HTTP, memory.
- This makes our device a **physical/hardware-control agent** (think: a talking,
  thinking gadget that controls real things) — not a coding agent. That is the
  intended product.

> If we ever want a *true* Claude-Code-style coding agent on cheap hardware, the
> right target is a **Raspberry Pi Zero 2 W (~$15)** running real Linux, where
> Hermes/Claude Code/Codex run almost unmodified. That is a **separate, optional
> "Project B"** noted at the end — out of scope for the ESP32 build.

---

## 1. What we decided (the conversation, distilled)

| Decision | Choice |
|---|---|
| **Approach** | Build **from scratch**, set everything up ourselves. |
| **Language/SDK** | **ESP-IDF (C)** — finest control over heap/PSRAM, what every serious claw project uses. |
| **Reference, not copy** | Study `esp-claw` & `zclaw` for the *basics* (TLS, JSON, WiFi, agent loop wiring), then implement in **our own original code, modeled on Hermes's architecture** — not a fork, not copy-paste. |
| **Fidelity goal** | Port **as many Hermes features as physically possible** to the ESP32, staying as close to "real Hermes" as the hardware allows. |
| **v1 scope** | **Text + hardware tools first** (talk to it via serial/Telegram/web; it controls GPIO/I2C/sensors). |
| **v2 scope** | **Voice** (I2S mic + speaker, cloud STT→LLM→TTS) as a later milestone. |
| **Storage** | Use **microSD** (supported) for big stuff (skills, sessions, logs) + on-chip NVS/flash for config/secrets, and optionally **pull skills from GitHub** over HTTPS. |
| **Intelligence** | Remote LLM via OpenAI-compatible `/chat/completions` (OpenRouter / Nous Portal / OpenAI / self-hosted). No on-device model. |

---

## 2. Reference projects (study these, write our own)

All of these are the **same architecture** we're building (remote LLM + local tool
execution on ESP32). We read them to learn *how the embedded basics are wired*, then
write original Hermes-flavored code.

| Project | URL | Why it's useful to us |
|---|---|---|
| **ESP-Claw** (Espressif official) | https://github.com/espressif/esp-claw | The reference implementation. ~85% C/ESP-IDF. Native OpenAI **and** Anthropic API support. Event-driven agent loop. Caches confirmed behaviors as local **Lua scripts** to avoid re-calling the LLM (a pattern worth stealing conceptually). Acts as MCP server + client. Chips: S3/P4/C5. |
| **zclaw** | https://github.com/tnm/zclaw | **Best "minimal" reference.** C, total firmware **888 KiB** — app logic only **~38 KiB (~4.6%)**, WiFi ~44%, TLS/crypto ~16%. Shows exactly where the bytes go. Tools: GPIO, DHT, `i2c_scan`, cron (`daily`/`periodic`/`once`), persistent memory, user-defined tools. Providers: Anthropic/OpenAI/OpenRouter/Ollama. Control via Telegram/web/serial. Built-in rate limiting. |
| **espclaw** (atiti) | https://github.com/atiti/espclaw | ~91% C. "Remote models, local tool execution." Lua "apps"+"components", OTA updates, Web UI / UART / Telegram control, host-side **simulator** (great for dev without flashing). ESP32-CAM support. |
| **WireClaw** | https://github.com/M64GitHub/WireClaw | Persistent memory + **offline rule engine**, Telegram/Serial/NATS control. |
| **MimiClaw** | https://github.com/memovai/mimiclaw | "Run an agent on a $5 chip," all data on flash. |
| **KALO ESP32 Voice Chat** | https://github.com/kaloprojects/KALO-ESP32-Voice-Chat-AI-Friends | **Voice reference for v2.** All STT/LLM/TTS in the **cloud** (ESP32 only records/plays). STT: ElevenLabs/Deepgram. LLM: Groq/OpenAI. TTS: OpenAI. HW: ESP32-S3 + INMP441 mic + MAX98357A amp. |
| **DaveBben/esp32-llm** | https://github.com/DaveBben/esp32-llm | Proof that **on-device LLM is a toy** — 260K-param model, 19 tok/s, no tool-calling. Confirms we must use a cloud API. |

**House rule on originality:** we may read these for understanding (how to set up the
TLS client, how they frame the system prompt, how the loop is structured), but our
code is written fresh against the **Hermes** architecture and naming. No verbatim
copying. When in doubt, model the implementation on `hermes-agent/agent/` modules, not
on the claw sources.

---

## 3. Architecture — Hermes mapped onto the ESP32

```
┌─ ESP32-S3 (FreeRTOS, ESP-IDF, C) ─────────────────────────┐      ┌─ Cloud ──────────────┐
│                                                            │      │ OpenRouter /         │
│  WiFi  ──►  mbedTLS (keep-alive, 16KB RX / 2KB TX buffers) │─────►│ Nous Portal /        │
│                                                            │ JSON │ OpenAI / self-hosted │
│  ── Agent loop  (port of agent/conversation_loop.py) ──────│◄─────│ /chat/completions    │
│      build messages[] + tools[] → POST → parse tool_calls  │      └──────────────────────┘
│      → dispatch each → append tool results → repeat        │
│                                                            │   (v2 voice adds:)
│  ── Tool registry  (port of tools/registry.py) ────────────│   I2S mic  → cloud STT
│      gpio_write, gpio_read, i2c_scan, read_sensor,         │   cloud TTS → I2S speaker
│      http_get, memory_set/get, timer/cron, send_message    │
│                                                            │
│  ── System prompt  (trimmed SOUL.md persona, ~800 tokens) ─│
│  ── Memory / skills  (NVS + SD card + GitHub) ─────────────│
│  ── State: in-RAM messages[] + session log to SD ──────────│
│  ── Control channel: serial / Telegram / tiny web UI ──────│
└────────────────────────────────────────────────────────────┘
```

### The core loop (what we implement first)
Straight port of the Hermes loop essence (`agent/conversation_loop.py`):

```
loop:
  request  = build_request(system_prompt, messages, tool_schemas)
  response = https_post(provider_url, request)          # blocking, non-streaming
  msg      = parse_message(response)                    # content + tool_calls
  append(messages, msg)
  if msg.tool_calls:
      for call in msg.tool_calls:
          result = registry.dispatch(call.name, call.args)   # returns JSON string
          append(messages, {role:"tool", tool_call_id:call.id, content:result})
      continue
  else:
      return msg.content                                # final answer
```

- **Non-streaming** for v1 (we need the *complete* `tool_calls` before acting; simpler,
  less RAM). Streaming/SSE is a later optimization, mainly for voice latency.
- Message shape is OpenAI Chat Completions: roles `system|user|assistant|tool`,
  `tool_calls[]`, `tool_call_id`.

### Tool model (Hermes `tools/registry.py` ported)
A tool = `{ name, JSON-schema (params), handler() → JSON string }`. We keep Hermes's
exact contract:
- Handlers return a **JSON string** (`{"result": ...}` or `{"error": ...}`).
- A registry holds tools; we send their schemas to the model; we dispatch by name.

**Tool mapping (Hermes → ESP32):**

| Hermes tool | ESP32 equivalent | Notes |
|---|---|---|
| `terminal` / shell | `gpio_write`, `gpio_read`, `set_pin`, `set_pwm` | the "do something physical" verb |
| (n/a) | `i2c_scan`, `i2c_read`, `read_sensor` | discover/read hardware |
| `web_search` / web | `http_get`, `http_post` | call any HTTP/JSON API |
| `memory` | `memory_set` / `memory_get` | NVS- or SD-backed |
| `todo` | `todo` | pure logic, ports directly |
| `send_message` | `send_telegram` / serial print | reply out-of-band |
| cron (`cronjob_tools`) | `timer_create` / `schedule` | FreeRTOS timers |
| `skills_*` | `skill_list` / `skill_view` | read SKILL.md from SD/GitHub |

**Not portable (need a full OS — explicitly out of scope):** `browser_*` (Playwright),
`code_execution`, `delegate`, `computer_use`, file-editing/grep on a real FS.

---

## 4. Hermes feature port map (how close to "real Hermes" we can get)

| Hermes feature | Port to ESP32? | How |
|---|---|---|
| **Core agent loop** | ✅ Full | ~200-300 lines C. Direct port. |
| **OpenAI-compatible providers** | ✅ Full | `esp_http_client` + mbedTLS POST. Start with OpenRouter/Nous Portal. |
| **Tool calling + registry** | ✅ Full | Port the `{name, schema, handler}` contract + dispatch. |
| **SOUL.md persona / system prompt** | ✅ Trimmed | 3-tier prompt (`agent/system_prompt.py`) shrunk to ~600-800 tokens. Persona stored on SD/flash. |
| **Memory (user facts, MEMORY.md/USER.md)** | ✅ Subset | JSON in NVS (small) or on SD; injected into system prompt each turn. |
| **Skills (markdown SKILL.md)** | ✅ Subset | Store on **SD card** and/or **fetch from a GitHub repo** over HTTPS, cache to SD. Inject relevant skill text into context. |
| **Sessions / conversation history** | ✅ Subset | In-RAM `messages[]`; persist a session log to SD (JSON lines). No SQLite/FTS5. |
| **Context compression** | ⚠️ Minimal | Simple truncation / drop-oldest at a token budget. (Hermes's full compressor is too heavy.) |
| **Cron / scheduled automations** | ✅ | FreeRTOS timers → trigger an agent turn (the "event-driven agent loop" pattern). |
| **Gateway / multi-platform** | ⚠️ One channel | v1: serial **or** Telegram **or** tiny web UI. Not the full multi-platform gateway. |
| **Voice (STT/TTS)** | ✅ v2 | I2S mic → cloud STT; cloud TTS → I2S amp. Hermes's STT/TTS are already cloud-pluggable. |
| **MCP client/server** | 🔮 Stretch | ESP-Claw does it; consider after core is solid. |
| **Self-improving skill creation** | 🔮 Stretch | Hermes's killer feature. Hard but not impossible: have the agent write a SKILL.md to SD/GitHub. Long-term goal. |
| **Behavior caching (ESP-Claw Lua trick)** | 🔮 Optional | Cache confirmed action sequences as scripts to cut API calls/cost. |
| **On-device LLM** | ❌ Never | Physically infeasible for real models. Always remote. |

---

## 5. Storage strategy (4 tiers)

ESP32 supports all of these; we use the right one per data type.

1. **NVS (Non-Volatile Storage, on-chip flash key-value)** — WiFi creds, API keys,
   config, small memory facts. Encrypted-at-rest capable. *Secrets live here, never on
   SD in plaintext.*
2. **Flash filesystem (LittleFS/SPIFFS)** — the trimmed SOUL.md persona, a few baked-in
   skills, small state. Survives reboot, no extra hardware.
3. **microSD card** — **YES, ESP32 supports it.** Over SPI on any board with free pins,
   or faster **SDMMC (1-/4-bit)** on ESP32-S3, using FAT/LittleFS. Use for the bulk:
   the full skills library, session/conversation logs, cached GitHub skills, audio
   buffers (v2). **Get a board with a microSD slot** (see buying guide).
4. **GitHub (remote skills repo)** — keep the canonical skills as markdown in a GitHub
   repo; the device pulls `SKILL.md` files over HTTPS at boot / on demand and caches
   them to SD. Lets us update skills without reflashing. (We already have an
   OpenAI-compatible HTTPS client for the LLM — reuse it.)

Recommended default: **secrets in NVS, persona+core skills in LittleFS, everything
bulky on SD, skill updates from GitHub.**

---

## 6. Hardware buying guide — which ESP32 to get

### The one rule: **you need PSRAM. Do not buy a bare ESP32-WROOM (no PSRAM).**
TLS uses ~30–50 KB heap per live session and JSON parsing needs ~2× the response size;
without PSRAM you run out of RAM. PSRAM is non-negotiable for this project.

### ✅ Recommended (in priority order)

| Board | Why | Approx price | microSD | PSRAM | Notes |
|---|---|---|---|---|---|
| **ESP32-S3-DevKitC-1 (N16R8)** | **Top pick for dev.** 16 MB flash, 8 MB PSRAM, dual-core S3, native USB. | ~$10–15 | add via SPI module or pick a variant w/ slot | 8 MB | The clean, well-documented baseline. Plenty of GPIO for sensors. |
| **Freenove ESP32-S3-WROOM (CAM)** | Has **onboard microSD slot** + camera. 8 MB PSRAM. Great all-rounder. | ~$12–18 | ✅ onboard | 8 MB | SD slot saves wiring; camera is a bonus for future vision. |
| **Seeed XIAO ESP32-S3 (Sense)** | Tiny. **Sense variant has onboard mic + microSD slot.** 8 MB PSRAM. | ~$14 | ✅ (Sense) | 8 MB | Best if you want **voice (v2)** with minimal wiring — mic is built in. |
| **M5Stack CoreS3 / Cardputer** | All-in-one "device" with screen, battery, microSD, mic/speaker. Runs ESP-Claw already. | ~$30–45 | ✅ onboard | yes | Best if you want a **finished-feeling gadget** out of the box (case + screen + audio). |
| **ESP32-P4 board (e.g. Elecrow CrowPanel Advance)** | Most headroom: 32 MB PSRAM, fast RISC-V, screen. WiFi via companion ESP32-C6. | ~$40–70 | ✅ usually | 32 MB | Future-proof / voice + vision. More complex (WiFi over SDIO). What the XDA ESP-Claw demo used. |

### ❌ Avoid
- **ESP32-WROOM / DevKitC (classic, no PSRAM)** — too little RAM for TLS+JSON.
- Ultra-cheap clones with no documented PSRAM.
- ESP32-C3 *for the full build* — fine for zclaw-minimal, tight for our feature-rich port (single-core, less RAM). OK as a cheap experiment board only.

### My recommendation
- **For getting started + text v1:** **ESP32-S3-DevKitC-1 N16R8** (8 MB PSRAM, 16 MB
  flash) + a cheap **SPI microSD module** + a couple of sensors (e.g. a BME280 over
  I2C, an LED, a button). ~$20 all in.
- **If you know you want voice (v2) soon:** **Seeed XIAO ESP32-S3 Sense** (built-in mic
  + SD slot) **or** add an **INMP441 I2S mic + MAX98357A I2S amp** to the S3 DevKit.
- **If you want it to feel like a real product immediately:** **M5Stack CoreS3**
  (screen + audio + SD + battery in a case).

### Starter bill of materials (text v1, ~$20–25)
- 1× ESP32-S3 board with 8 MB PSRAM (DevKitC-1 N16R8) — *or* a board with onboard SD.
- 1× microSD module (SPI) + a microSD card (8–32 GB, FAT32) — if not onboard.
- 1× I2C sensor for the first real tool (e.g. BME280 temp/humidity/pressure).
- 1× LED + resistor + 1× push button (first `gpio_write` / `gpio_read` demos).
- Breadboard + jumper wires + USB-C cable.

### v2 voice add-ons
- INMP441 I2S MEMS microphone.
- MAX98357A I2S amplifier + small 4–8 Ω speaker.
- (Or just use a XIAO ESP32-S3 Sense / M5 CoreS3 that bundle audio.)

---

## 7. Constraints & gotchas (the real engineering)

- **Memory is the #1 risk.** Route big buffers (JSON bodies, skill text, audio) to
  **PSRAM explicitly** — the exact bug the XDA ESP-Claw test hit. Internal DRAM is
  precious.
- **TLS:** keep the connection **alive across turns**; use **asymmetric buffers**
  (16 KB RX — can't shrink, server controls fragment size; ~2 KB TX — our requests are
  small); **disable "keep peer cert after handshake."**
- **JSON:** don't fully deserialize big responses — **stream-parse** and pull only
  `content` / `tool_calls`. Prefer **cJSON** (ships with ESP-IDF) over ArduinoJson.
- **Power:** WiFi is the sink (~25 mA idle, hundreds of mA transmitting). Deep sleep
  kills WiFi and forces a costly TLS re-handshake on wake. **Pick a profile:**
  always-on (mains/desk gadget) **or** event-driven sleep+wake (battery) — not both.
  For battery: cache WiFi BSSID/channel in RTC memory + static IP to cut reconnect time.
- **Token budget:** keep the system prompt tiny (~600–800 tokens) so a 4–8K context
  model leaves room for conversation.
- **Rate limiting / cost:** add request rate limits (zclaw does 100/hr, 1000/day) — an
  agent loop can rack up API calls fast.

---

## 8. Roadmap / milestones

> Each milestone is independently demoable. Don't move on until the previous one works.

- [ ] **M0 — Toolchain & bring-up.** Install ESP-IDF, build & flash "hello world",
      confirm PSRAM is detected, mount LittleFS + microSD. WiFi connects.
- [ ] **M1 — First cloud round-trip.** `esp_http_client` + mbedTLS POST to OpenRouter
      `/chat/completions`, send a hardcoded prompt, parse and print the reply.
      *(Proves the hard part: TLS + JSON + memory.)*
- [ ] **M2 — Minimal agent loop + one tool.** Implement build→call→parse→dispatch→loop
      with a single `gpio_write` tool. Ask the model in plain English to blink an LED;
      watch it call the tool. *(Proves tool-calling end to end.)*
- [ ] **M3 — Tool registry.** Port the Hermes `{name, schema, handler}` registry +
      dispatch. Add `gpio_read`, `i2c_scan`, `read_sensor`, `http_get`,
      `memory_get/set`.
- [ ] **M4 — Hermes persona + memory.** Trimmed SOUL.md system prompt; persist memory
      facts (NVS) and a session log (SD). Multi-turn conversation with context.
- [ ] **M5 — Control channel.** Talk to it over serial **and** Telegram (or a tiny web
      UI) — the "gateway" concept, one channel.
- [ ] **M6 — Skills.** Read SKILL.md from SD; fetch/update skills from a GitHub repo;
      inject relevant skill text into context.
- [ ] **M7 — Cron / event triggers.** FreeRTOS timers + sensor events kick off an agent
      turn (event-driven agent loop).
- [ ] **M8 (v2) — Voice.** I2S mic capture → cloud STT → loop → cloud TTS → I2S amp.
- [ ] **M9+ (stretch)** — MCP client/server, behavior caching, self-authored skills.

---

## 9. Proposed repo layout

```
esp32-hermes/                ← project folder (named to dodge the repo's hermes-*/ gitignore)
├── PROJECT_PLAN.md          ← this file
├── README.md                ← quickstart once code exists
├── main/
│   ├── main.c               ← app entry, WiFi/storage init, REPL/control loop
│   ├── agent_loop.c/.h      ← the conversation loop (port of conversation_loop.py)
│   ├── provider_http.c/.h   ← mbedTLS + esp_http_client OpenAI-compatible client
│   ├── messages.c/.h        ← message list, JSON (de)serialize (cJSON)
│   ├── registry.c/.h        ← tool registry + dispatch (port of registry.py)
│   ├── tools/               ← gpio.c, i2c.c, sensor.c, http.c, memory.c, timer.c ...
│   ├── prompt.c/.h          ← system prompt builder (trimmed system_prompt.py)
│   ├── storage.c/.h         ← NVS + LittleFS + SD + GitHub skill fetch
│   └── control/             ← serial.c, telegram.c, webui.c
├── persona/SOUL.md          ← the Hermes persona (trimmed)
├── skills/                  ← local SKILL.md files (also mirrored to a GitHub repo)
├── reference/               ← cloned esp-claw / zclaw for STUDY ONLY (gitignored)
├── sdkconfig.defaults       ← PSRAM on, TLS buffers, partition table
└── partitions.csv           ← flash layout (app + nvs + littlefs)
```

> `reference/` is for reading the claw projects while we work. **Our code in `main/`
> is written fresh against the Hermes architecture.** Keep `reference/` out of our
> commits.

---

## 10. Open questions to resolve before/while building

1. **Which provider first?** OpenRouter (easiest, 200+ models, one key) vs Nous Portal
   (Hermes-native) vs a self-hosted Ollama/llama.cpp endpoint (free, local network, no
   per-token cost — great for dev). *Suggest: start with a self-hosted/Ollama endpoint
   for free iteration, then OpenRouter.*
2. **Which model?** Needs solid **tool-calling**. Candidates: a Hermes/Nous model, or
   any strong tool-calling model on OpenRouter. Smaller = cheaper + faster but worse at
   tools — test a few.
3. **Control channel for v1?** Serial-only is simplest to start; Telegram is the most
   satisfying ("text my gadget"). *Suggest: serial for M1–M4, add Telegram at M5.*
4. **Exact board purchased?** Pins/SD wiring depend on it — lock this from the buying
   guide so M0 wiring is concrete.
5. **GitHub skills repo:** public or private? (Private needs a token in NVS.)

---

## 11. "Project B" (parked, not now)

If the real desire is a **Claude-Code-style coding agent** (shell, files, git) rather
than a hardware gadget, the ESP32 is the wrong tool — use a **Raspberry Pi Zero 2 W
(~$15)** running Linux, where Hermes / Claude Code / Codex run almost unmodified. Could
later pair with the ESP32 (Pi = brain, ESP32 = hardware limbs). **Out of scope here;
noted so we don't conflate the two.**

> ⚠️ **Don't confuse the two near-identical names:**
> - **Raspberry Pi Zero 2 W** (~$15) = a *Linux computer* (quad-core A53, 512 MB RAM).
>   Runs real Hermes/Claude Code directly. **This is Project B.**
> - **Raspberry Pi Pico 2 W** (~$7) = a *microcontroller* (RP2350, 520 KB SRAM, no
>   Linux) — same class as the ESP32, **not** a Linux box. It can run the *embedded*
>   from-scratch agent, but it's a worse fit than an ESP32-S3 (no PSRAM out of the box,
>   far fewer agent/TLS reference projects). **Not** the way to run real Hermes.


---

## 12. Sources / further reading

- Hermes Agent: https://github.com/NousResearch/hermes-agent
- ESP-Claw (official): https://github.com/espressif/esp-claw
- zclaw: https://github.com/tnm/zclaw
- espclaw: https://github.com/atiti/espclaw
- WireClaw: https://github.com/M64GitHub/WireClaw
- KALO voice: https://github.com/kaloprojects/KALO-ESP32-Voice-Chat-AI-Friends
- ESP-Claw on ESP32-P4 (hands-on): https://www.xda-developers.com/ran-espressif-official-ai-agent-esp32-self-hosted-llm/
- On-device LLM reality check: https://github.com/DaveBben/esp32-llm
- ESP-IDF mbedTLS memory notes: https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/protocols/mbedtls.html
- ArduinoJson memory: https://arduinojson.org/v7/how-to/reduce-memory-usage/
- ESP32 sleep/power: https://lastminuteengineers.com/esp32-sleep-modes-power-consumption/

---

### Tomorrow-morning quick start
1. Buy/grab an **ESP32-S3 board with 8 MB PSRAM** (+ microSD). See §6.
2. Open Claude Code in this repo (this plan lives in `esp32-hermes/`) and start at
   **M0** (§8): install ESP-IDF, flash hello world, confirm PSRAM + SD + WiFi.
3. Then **M1**: one TLS POST to a `/chat/completions` endpoint, print the reply.
4. Keep `reference/` (esp-claw, zclaw) cloned for study; write our own code in `main/`.

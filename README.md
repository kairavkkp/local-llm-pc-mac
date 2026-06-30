# Local LLM Coding Assistant — Setup

Self-hosted code assistant: models run on the **PC** (AMD RX 9070 XT, 16GB), tools run on the **Mac** over the LAN.

## Architecture

```
  Mac (dev machine)                 PC (inference server)
  ┌──────────────────┐              ┌──────────────────────┐
  │ aider / Continue │── Wi-Fi ───▶ │ Ollama  :11434       │
  │ VS Code          │   LAN        │ RX 9070 XT (ROCm)    │
  └──────────────────┘              └──────────────────────┘
```

## Connection details (fill these in)

| Field        | Value                          |
|--------------|--------------------------------|
| PC hostname  | `__________.local`             |
| PC IP        | `192.168.___.___` (reserved)   |
| Ollama port  | `11434`                        |
| Base URL     | `http://__________:11434`      |

---

## PC setup (Windows)

### 1. Install Ollama
Installer: https://ollama.com/download/windows
Verify: browser → `http://localhost:11434` shows "Ollama is running".

### 2. Verify GPU acceleration  ← the step that matters
```powershell
ollama run qwen2.5-coder:3b      # then, in a second window:
ollama ps                        # PROCESSOR column should read 100% GPU
notepad $env:LOCALAPPDATA\Ollama\server.log   # search for: gfx1201
```
Good log line: `library=ROCm ... compute=gfx1201 ... AMD Radeon RX 9070 XT`
If it falls back to CPU, see Troubleshooting below.

### 3. Pull models
```powershell
ollama pull devstral            # agentic chat/edit (~14GB)
ollama pull qwen2.5-coder:3b    # fast autocomplete (FIM)
ollama pull nomic-embed-text    # codebase embeddings
```

### 4. Expose on the LAN
By default it only listens on localhost. Add a user environment variable so it binds to all interfaces:
Settings → search "environment variables" → Edit environment variables for your account → New → Name OLLAMA_HOST, Value 0.0.0.0:11434 → OK.
Then right-click the Ollama tray icon → Quit, and relaunch it so it picks up the change.


### 5. Firewall (admin PowerShell)
```powershell
New-NetFirewallRule -DisplayName "Ollama" -Direction Inbound -LocalPort 11434 -Protocol TCP -Action Allow
```

## Mac setup (next)

Test reachability first:
```bash
curl http://<PC>.local:11434/api/tags     # should return JSON list of models
```

Terminal agent (Claude Code style):
```bash
pipx install aider-chat
export OLLAMA_API_BASE=http://<PC>.local:11434
aider --model ollama/devstral
```

Editor autocomplete + chat (Copilot style): install the **Continue** VS Code
extension and point `apiBase` at `http://<PC>.local:11434` (see `continue-config.yaml`).

---

## Troubleshooting

**Ollama uses CPU instead of the GPU (gfx1201 not detected)**
RDNA 4 needs ROCm 7.x. If the bundled libs are older:
1. Update to the latest Ollama, restart, recheck `server.log`.
2. Still CPU → apply the gfx1201 ROCm-7 build from the `xnyzer/ollama-rocm` repo.
3. Or use LM Studio with its ROCm/Vulkan runtime as an alternative server.

**Mac can't reach the PC**
- `curl http://<IP>:11434/api/tags` directly (rules out `.local`/mDNS issues).
- Confirm `OLLAMA_HOST=0.0.0.0:11434` is set and Ollama was restarted.
- Confirm the firewall rule exists and both devices are on the same network/band.

**Models too slow / partly on CPU**
Keep models under ~14GB after quantization so they fit fully in 16GB VRAM.

# Local LLM Coding Assistant — Setup

Self-hosted code assistant: models run on the **PC** (AMD RX 9070 XT, 16GB),
tools run on the **Mac**. Connected over **Tailscale** (not plain Wi-Fi — see note).

> ⚠️ **Networking note:** Direct LAN (`.local` / raw IP over Wi-Fi) does **not** work
> on this network. Both machines sit on the same subnet (`192.168.29.x`) but the
> JioFiber router does **AP/client isolation**, so they can't reach each other —
> and the router admin panel is locked down, so isolation can't be disabled.
> **Tailscale** tunnels over this and is the working path. Keep using it.

## Architecture

```
  Mac (dev machine)            Tailscale mesh            PC (inference server)
  ┌──────────────────┐      (encrypted tunnel,          ┌──────────────────────┐
  │ aider / Continue │       bypasses router            │ Ollama  :11434       │
  │ + Tailscale      │── 100.x.x.x ───────isolation────▶│ + Tailscale          │
  └──────────────────┘                                  │ RX 9070 XT (ROCm)    │
                                                        └──────────────────────┘
```

## Connection details

| Field                 | Value                              |
|-----------------------|------------------------------------|
| PC LAN hostname       | `kairav` (does NOT work — router isolation) |
| PC Tailscale IP       | `100.78.136.106`  ← fill in (`tailscale ip -4`)  |
| PC Tailscale name     | `kairav` (if MagicDNS enabled)     |
| Ollama port           | `11434`                            |
| Base URL              | `http://100.78.136.106:11434`     |

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
ollama pull devstral              # best coding/agentic chat (~14GB Q4 — fills VRAM)
ollama pull qwen2.5-coder:14b    # lighter coding model, snappy edits (~9GB Q4)
ollama pull deepseek-r1:14b      # best reasoning at this size (~9GB Q4)
ollama pull qwen2.5-coder:7b     # fast FIM autocomplete (~5GB Q4)
ollama pull nomic-embed-text     # codebase embeddings (~0.3GB)
ollama list                       # note the EXACT tag for each, e.g. devstral:24b
```

### 4. Expose Ollama on all interfaces
By default it only listens on localhost. Add a user environment variable so it
binds to all interfaces (needed for Tailscale to reach it):
Settings → search "environment variables" → Edit environment variables for your
account → New → Name `OLLAMA_HOST`, Value `0.0.0.0:11434` → OK.
Then right-click the Ollama tray icon → Quit, and relaunch it.

Confirm it took effect:
```powershell
netstat -ano | findstr :11434    # must show 0.0.0.0:11434, not 127.0.0.1
```

### 5. Firewall (admin PowerShell)
Scope to ALL profiles so a Public/Private mislabel can't block it:
```powershell
New-NetFirewallRule -DisplayName "Ollama" -Direction Inbound -LocalPort 11434 -Protocol TCP -Action Allow -Profile Any
```

### 6. Install Tailscale
Download: https://tailscale.com/download/windows → run installer.
Tray icon → Log in → sign in with Google/Microsoft/GitHub.
**Remember which account** — the Mac must use the same one.
```powershell
tailscale ip -4                  # the 100.x.x.x address → put it in the table above
```

---

## Mac setup

### 1. Install Tailscale
Download: https://tailscale.com/download/mac → open the app.
Allow the VPN configuration prompt → menu-bar icon → Log in →
**sign in with the SAME account used on the PC.**
Verify both machines are listed at https://login.tailscale.com

### 2. Test reachability (use the PC's Tailscale IP)
```bash
curl http://100.78.136.106:11434/api/tags    # should return JSON list of models
```

### 3. Terminal agent (Claude Code style)
```bash
pipx install aider-chat
export OLLAMA_API_BASE=http://100.78.136.106:11434 OR export OLLAMA_API_BASE=http://kairav:11434
aider --model ollama_chat/devstral:24b   # ollama_chat/ prefix + EXACT tag from `ollama list`
```

### 4. Editor autocomplete + chat (Copilot style)
Install the **Continue** VS Code extension and point `apiBase` at
`http://100.78.136.106:11434` (see `continue-config.yaml`).

---

## Troubleshooting

**`model 'devstral' not found` (aider connects but model errors)**
The bare name only resolves to `devstral:latest`. Use the exact tag from
`ollama list` and the `ollama_chat/` prefix, e.g. `ollama_chat/devstral:24b`.

**Mac can't reach the PC — "No route to host" / curl hangs**
Diagnose in order:
1. `netstat -ano | findstr :11434` on PC → must be `0.0.0.0:11434`.
2. Compare IPs: PC `ipconfig | findstr IPv4` vs Mac `ipconfig getifaddr en0`.
   - Different subnets / Mac shows `169.254.x.x` → different Wi-Fi or band; rejoin same SSID.
   - Same subnet but still unreachable even with the **firewall fully off**
     (`Set-NetFirewallProfile -All -Enabled False`) → **router client isolation**.
     This is THIS network's situation. Don't fight the LAN — use **Tailscale** (above).
3. Over Tailscale still failing? Confirm both machines show "connected" (not paused)
   and are under the **same account** at https://login.tailscale.com.

**Ollama uses CPU instead of the GPU (gfx1201 not detected)**
RDNA 4 needs ROCm 7.x. If the bundled libs are older:
1. Update to the latest Ollama, restart, recheck `server.log`.
2. Still CPU → apply the gfx1201 ROCm-7 build from the `xnyzer/ollama-rocm` repo.
3. Or use LM Studio with its ROCm/Vulkan runtime as an alternative server.
Symptom from the Mac: replies crawl out ~1 word/sec instead of streaming fast.

**Models too slow / partly on CPU**
Keep models under ~14GB after quantization so they fit fully in 16GB VRAM.

---

## Quality-of-life (optional)

- **MagicDNS:** enable in the Tailscale admin console → use `http://kairav:11434`
  instead of the `100.x` number. Stable forever, survives everything.
- **aider context window:** Ollama defaults to a small context. For repo-scale work,
  set `OLLAMA_CONTEXT_LENGTH=16384` on the PC and use `--map-tokens 2048` in aider.

## Using Continue on VSCode

1. **Install the Continue extension**:
   - Open Visual Studio Code.
   - Go to the Extensions view by clicking on the Extensions icon in the Activity Bar or pressing `Ctrl+Shift+X`.
   - Search for "Continue" and install it.

2. **Configure Continue with your Ollama API Base URL**:
   - Once installed, open the Command Palette (`F1` or `Ctrl+Shift+P`).
   - Type `Continue: Open Settings` and select it.
   - In the settings file that opens, add or update the following lines to point to your Ollama server:
     ```yaml
     apiBase: "http://kairav:11434"
     ```
   - Save the changes. Refere `config.yaml` in the repo and adjust as needed.

3. **Start using Continue**:
   - Open a file in VSCode where you want code assistance.
   - Use the Command Palette to open the `Continue` chat or simply start typing, and the extension will provide suggestions and completions based on your configuration.

For further details, refer to the [official Continue documentation](https://github.com/continue-dev/continue-vscode).
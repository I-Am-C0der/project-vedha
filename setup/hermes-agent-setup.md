# Hermes Agent "Veda" — Staged Implementation Guide (v2.1.1)

| | |
|---|---|
| **Target host** | Windows 11 · GTX 1660 Super (6 GB) · 16 GB DDR4 · i3-10105F |
| **Guide revision** | v2.1.1 — v2.1 plus the consistency, recovery, and validation corrections recorded in Appendix K. Supersedes v1 (October 4, 2026). |
| **Release anchor** | Hermes Agent **v0.21.5** (`v2026.9.24`, commit `f97608f178d1ffeca59860195ab7da295f7c8e5f`) — taken from v1; re-confirm per Appendix J before installing. |
| **Models** | GLM-5.3-Flash (primary) and DeepSeek V4.1 Flash (delegated), both via OpenRouter, **CoreWeave-only routing**, no model-level fallback. |
| **Assistant name / project name** | The assistant is called **Veda**; the project, its folders and scripts are called **Vedha** (`/srv/vedha`, `vedha-*`). |

---

## Table of contents

| Part | Sections |
|---|---|
| Start here | [How to read this guide](#how-to-read-this-guide) · [Before you start: basics for first-time users](#before-you-start-basics-for-first-time-users) · [What you need before Stage 0](#what-you-need-before-stage-0) · [Architecture at a glance](#architecture-at-a-glance) · [Storage layout](#project-vedha-storage-layout-mandatory) |
| Build | [Stage 0](#stage-0--windows-host-preparation) · [Stage 1](#stage-1--wsl-foundation-storage-root-ops-toolkit--pinned-hermes-install) · [Stage 2](#stage-2--primary-model-complete-coreweave-routing-policy--basic-chat) · [Stage 3](#stage-3--execution-boundary-docker-sandbox-approvals--security-foundation) · [Stage 4](#stage-4--identity-engineering-context--persistent-memory) · [Stage 5](#stage-5--web-research--prompt-injectionexfiltration-hardening) · [Stage 6](#stage-6--skills-mcp--integrations) · [Stage 7](#stage-7--delegated-coding-model-deepseek-v41-flash) |
| Operate | [Stage 8](#stage-8--automation-cron--background-work) · [Stage 9](#stage-9--telegram-gateway--always-on-operation) · [Stage 10](#stage-10--additional-messaging-platforms-optional) · [Stage 11](#stage-11--reliability-cost-observability--alerting) · [Stage 12](#stage-12--local-voice-pipeline-stt--tts--wake-word--barge-in) · [Stage 13](#stage-13--proactive-jarvis-capabilities) · [Stage 14](#stage-14--failure-mode-drills) · [Final Stage](#final-stage--end-to-end-validation-backup-restore-drill--update-procedure) |
| Reference | [A Models](#appendix-a--model--architecture-reference) · [B Do's and Don'ts](#appendix-b--consolidated-dos-and-donts) · [C Troubleshooting](#appendix-c--troubleshooting-reference) · [D Resources](#appendix-d--resources) · [E Upgrades](#appendix-e--future-hardware-upgrade-path) · [F Quick reference](#appendix-f--quick-reference) · [G Bare metal](#appendix-g--bare-metal-linux-alternative-to-stages-01) · [H Cloud voice](#appendix-h--cloud-voice-alternatives-inactive) · [I Security model](#appendix-i--agent-security-model) · [J Ledger](#appendix-j--config-fragments-release-assumption-ledger--audits) · [K Changelog](#appendix-k--changelog-and-validation-record) · [L Resource budget](#appendix-l--resource-budget-16-gb-host-estimates-measure-and-adjust) |

---

## How to read this guide

### Markers

| Marker | Meaning |
|---|---|
| **[VERIFY]** | A release-specific fact (config key, CLI flag, file path, behavior, tag, digest) carried over from v1 that the v2 author could **not** independently confirm. Every [VERIFY] item has a matching row and a check command in **Appendix J — Release-Assumption Ledger**. Do not treat it as fact until that check passes on your machine. |
| ⚠️ **USER INPUT REQUIRED** | You must supply a value (key, token, ID, name). |
| 🪟 **PowerShell** / 🪟 **Windows GUI** | Run on Windows. Everything else runs in **Ubuntu (WSL2)**. |
| `vedha-*` | A Project Vedha ops script installed under `/srv/vedha/ops/bin` (created during the stages). |
| `[YOUR_…]` | A personal value you type in **your local copy only** (name, city, time zone, quiet hours). |
| `<…>` | A value you replace before running the line (e.g. `<value-from-4.1>`). Never paste the angle brackets. |

### The three rules that prevent most failures

1. **Never paste a multi-line script that contains `exit`, `set -e`, or `trap` into your interactive shell.** If it fails, your terminal closes. This guide puts every such procedure into a `vedha-*` script file, which you then **run**.
2. **Never replace `$HERMES_HOME/config.yaml` by hand.** Each stage adds a *fragment* under `ops/config/fragments/`, and `vedha-config-apply` deep-merges it, shows a diff, validates it, and commits a snapshot. Fragments only **add or override** keys; removing a key is a separate, explicit step (Stage 1.9e).
3. **One supervisor per long-lived service.** The Hermes gateway and STT service are managed **only** with `systemctl --user`. Do not use `hermes gateway install/start/stop/uninstall`, and do not launch a second foreground gateway while the unit is running.

---

## Before you start: basics for first-time users

You don't need Linux experience, but you need these six habits.

**1. Opening the two terminals**

| You need | How to open it |
|---|---|
| 🪟 **PowerShell (Administrator)** | Press **Win**, type `PowerShell`, right-click **Windows PowerShell** → **Run as administrator** → **Yes**. |
| 🪟 **PowerShell (normal)** | Same, but just click it. |
| **Ubuntu (WSL2) terminal** | After Stage 0.3: press **Win**, type `Ubuntu`, press **Enter**. (Or open **Terminal** and pick **Ubuntu** from the ⌄ menu.) The prompt ends in `$`. |

**2. Copy and paste**
- Copy a code block from this guide with the copy button or by selecting it.
- Paste into **Windows Terminal / Ubuntu** with **Ctrl+Shift+V** or a **right-click**. (Ctrl+V may not work in Linux shells.)
- Paste **one code block at a time**, then read the output before the next block.

**3. What the code blocks do**
- Blocks that start with `cat > /some/file <<'VEDHA_EOF'` **only create a file**. Nothing runs until you type the file's name.
- Lines starting with `#` are comments; they do nothing.
- `sudo …` runs one command as administrator and asks for **your Linux password** (typing shows nothing — that's normal).

**4. Editing a file with `nano`** (used rarely)
`nano <file>` opens an editor. Type or paste, then press **Ctrl+O**, **Enter** (save) and **Ctrl+X** (exit).

**5. Reading results**
- `PASS` lines are good. `WARN` means "check, may be expected at this stage". `FAIL` means **stop and fix before continuing** — use the stage's *Troubleshooting* section, then Appendix C.
- Every script ends with `Summary: FAIL=<n> WARN=<n>`.

**6. If you get lost**
- Run `. /srv/vedha/vedha-env.sh` (after Stage 1.7) to reload the environment, then `vedha-preflight` to see where you are.
- Each stage has a **Rollback / recovery** section. Nothing in this guide deletes your data without saying so in bold.

---

## What you need before Stage 0

| Item | Why | When |
|---|---|---|
| Windows 11 PC with administrator rights, F: drive (NTFS, ≥ 80 GB free) | Host for everything | Stage 0 |
| Internet connection | Downloads, OpenRouter, Telegram | All stages |
| **OpenRouter** account with credits (openrouter.ai) | Both LLMs | Stage 2 |
| **Telegram** account on your phone, with two-step verification | Remote access and alerts | Stage 0.7, Stage 9 |
| A web-search provider account (e.g. Tavily, Firecrawl or Exa) | Web research | Stage 5 |
| A **password manager** | OpenRouter key, bot token, backup private key | Stage 2, Final |
| A **USB stick** (or other offline storage) | Offline copy of the backup key; offsite backup copy | Final |
| A **headset** with microphone (recommended) | Voice without echo/self-triggering | Stage 12 |
| About 2–4 evenings | Each stage is 30–90 minutes; don't rush the verification steps | — |

---

## Architecture at a glance

| Layer | Choice | Notes |
|---|---|---|
| Primary LLM | `z-ai/glm-5.3-flash` via OpenRouter, `only: [coreweave]` | Conversation, research, automation, light coding |
| Delegated LLM | `deepseek/deepseek-v4.1-flash` via OpenRouter, `only: [coreweave]` | Coding/debugging/long tool loops via `delegate_task` |
| Auxiliary LLM calls | Every auxiliary route pinned to GLM on CoreWeave **from Stage 2** | Compression, titles, vision, memory rewrite, approvals, etc. |
| Fallback | None (fail-closed) | Outages are detected and alerted (Stage 11), never silently rerouted |
| Execution | Docker sandbox, no network by default, workspace bind mount only | Custom Java-capable image (Stage 3) |
| Memory | Hermes built-in `MEMORY.md` + `USER.md` (size-bounded) | External providers optional |
| Remote access | Telegram (primary), others optional | Per-platform tool restrictions |
| Voice | Local STT (faster-whisper) + local TTS (Kokoro-FastAPI) | Voice mode response style, latency budget |
| Supervision | systemd user units + linger + WSL keep-alive task | Health timer + Telegram alerting independent of the LLM |
| Storage | **WSL distro VHDX on F:**; all Vedha state on ext4 at `/srv/vedha` | Satisfies "everything on F:" without DrvFS/SQLite risk |
| Change control | `ops/` git repo (config fragments, persona, scripts, units) | Blue/green release switching, encrypted backups |

---

## Why this guide is structured in stages

Each stage adds one capability that you can verify on its own, on top of everything already proven. Finish a stage, pass its checklist, then move on. If something fails, it can only have been caused by the stage you just did.

### Overall progress

```text
[ ] Stage 0  — Windows Host Preparation (power, driver, WSL, distro on F:, Docker Desktop)
[ ] Stage 1  — WSL Foundation, Storage Root, Ops Toolkit & Pinned Hermes Install
[ ] Stage 2  — Primary Model, Complete CoreWeave Routing Policy & Basic Chat
[ ] Stage 3  — Execution Boundary (Docker Sandbox), Approvals & Security Foundation
[ ] Stage 4  — Identity, Engineering Context & Persistent Memory
[ ] Stage 5  — Web Research & Prompt-Injection/Exfiltration Hardening
[ ] Stage 6  — Skills, MCP & Integrations
[ ] Stage 7  — Delegated Coding Model (DeepSeek V4.1 Flash)
[ ] Stage 8  — Automation, Cron & Background Work
[ ] Stage 9  — Telegram Gateway & Always-On Operation
[ ] Stage 10 — Additional Messaging Platforms (Optional)
[ ] Stage 11 — Reliability, Cost, Observability & Alerting
[ ] Stage 12 — Local Voice Pipeline (STT + TTS + Wake Word + Barge-in)
[ ] Stage 13 — Proactive "Jarvis" Capabilities
[ ] Stage 14 — Failure-Mode Drills
[ ] Final    — End-to-End Validation, Backup, Restore Drill & Update Procedure
```

### Dependency map

```text
Stage 0 (Windows host) ─► Stage 1 (WSL + toolkit + Hermes)
                              │
                              ▼
                         Stage 2 (Model + full routing policy)
                              │
                              ▼
                         Stage 3 (Sandbox + approvals)
                              │
                              ▼
                         Stage 4 (Identity + memory + AGENTS.md)
                              │
               ┌──────────────┼──────────────┐
               ▼              ▼              ▼
           Stage 5        Stage 6        Stage 7
           (Web)          (Skills/MCP)   (Delegation)
               └──────────────┼──────────────┘
                              ▼
                         Stage 8 (Cron)
                              ▼
                         Stage 9 (Telegram + always-on)
                              ▼
                         Stage 10 (optional platforms)
                              ▼
                         Stage 11 (Reliability/alerting)
                              ▼
                         Stage 12 (Voice)
                              ▼
                         Stage 13 (Proactive features)
                              ▼
                         Stage 14 (Failure drills) ─► Final
```

**Why this order:**
- Routing policy is complete in Stage 2. Every LLM call is CoreWeave-only from the first request, not only after Stage 11.
- Identity and context (Stage 4) come before delegation, so child sessions get the right engineering rules.
- Alerting (Stage 11) comes before voice and proactive features, so the always-on paths are monitored from day one.
- The failure drills (Stage 14) come before you call the system reliable.

---

## Important notes before you begin

1. **WSL2 is the baseline.** v1 said Hermes documents a native Windows install **[VERIFY]**. Native Windows is a different deployment path. Do not mix it with this guide.
2. **The exact release matters.** v1 recorded `requires-python >=3.11,<3.14` for v0.21.5 **[VERIFY]**. This guide builds a dedicated Python 3.11 runtime per release and checks the requirement from `pyproject.toml` before installing.
3. **Install as your normal WSL user, never with `sudo`.** Only the commands explicitly prefixed with `sudo` need root.
4. **Built-in memory is always on and size-bounded.** Hermes's `MEMORY.md`/`USER.md` are curated, bounded system-prompt injections **[VERIFY limits]**. The persona files in Stage 4 are written to fit those limits.
5. **Wake word and barge-in are local-surface features.** They work in the CLI/TUI/desktop, not Telegram **[VERIFY row 61]**. Telegram voice notes are a separate path in.
6. **Delegation has one configured child model/provider.** `delegate_task` has no per-call model or effort argument **[VERIFY row 62]**.
7. **CoreWeave-only is fail-closed by design.** A CoreWeave outage means Veda cannot think. This guide does not hide that. Instead it **detects and alerts** on it without needing an LLM (Stage 11: endpoint listing **and** an authenticated hourly liveness probe, so a revoked key, exhausted credit or real outage all alert), and documents a manual, logged break-glass decision. The pin itself is proven with negative tests (Stages 2.7b, 7.2b, 11.7), not just by seeing CoreWeave in Activity.
8. **Auxiliary LLM calls do not inherit `provider_routing`** **[VERIFY row 5]**. That is why Stage 2 first lists **every** auxiliary route the pinned source has (2.4a) and pins all of them at the same time as the main model; `vedha-keycheck` fails on any unpinned route.
9. **Docker cross-process container reuse is disabled** (`docker_persist_across_processes: false`) for v0.21.5 **[VERIFY rationale: upstream #84969]**. Durable state is the host workspace.
10. **`docker_run_as_host_user: false`** keeps bundled skills predictable on v0.21.5 **[VERIFY rationale: upstream #34026]**. The cost is root-owned files in the workspace, handled by `vedha-repair-ownership`.
11. **Cron and the state database get extra verification.** v1 cited WSL-era reports of cron stalls and WAL problems **[VERIFY]**. v2 removes the main risk (DrvFS) by keeping state on ext4. It also adds a heartbeat that detects cron stalls, plus integrity checks.
12. **Never run `hermes update` on this install.** Updates are blue/green: install the new release side by side, test it against a copy of your state, switch a symlink, and keep the old release for rollback (Final stage).
13. **Personal use only.** Everything Veda reads is sent to OpenRouter/CoreWeave. **Never put employer or client code, data, logs, or credentials into the Vedha workspace, memory, or chats.** The persona files tell Veda to flag such material.
14. **Start coding sessions from the workspace** (`cd /srv/vedha/workspace && hermes`). Context files are discovered from the host working directory **[VERIFY row 16]**, so a session started elsewhere, and its delegated children, gets no `AGENTS.md` safety rules.

---

## Execution environment rule

**Unless a step is labeled 🪟 PowerShell or 🪟 Windows GUI, run it in the Ubuntu WSL2 terminal.**

| Environment | Use it for |
|---|---|
| **Ubuntu WSL2** | Everything about Project Vedha: toolkit, Hermes, config, Docker CLI, services, tests |
| **🪟 PowerShell** | `wsl --install/--update/--manage/--shutdown`, `.wslconfig` deployment, power settings, Task Scheduler registration |
| **🪟 Windows GUI** | NVIDIA driver, Docker Desktop install/settings, BotFather/OpenRouter web dashboards |

Do not paste PowerShell into Ubuntu or bash into PowerShell. The language tag on each code block tells you where it runs.

---

## Project Vedha storage layout (mandatory)

**Requirement:** everything lives on F: under `F:\project-vedha`.

**How v2 meets it without DrvFS risk:** v1 put live Hermes state on `/mnt/f`. That is DrvFS, a 9P-style network filesystem, where SQLite WAL is not reliable, Python imports and Docker bind mounts are 5–20× slower, and Linux permissions do not protect secrets from Windows processes. v2 **moves the Ubuntu WSL distro itself onto F:**. The Linux filesystem (ext4 inside a VHDX) physically sits in `F:\project-vedha\wsl\`, and all Vedha state lives on that ext4 filesystem at `/srv/vedha`.

> **Path invariant:**
> - `/srv/vedha` is the **live Project Vedha filesystem**. It is ext4 inside the WSL VHDX stored on `F:\project-vedha\wsl\Ubuntu`.
> - `/mnt/f` is the **DrvFS bridge to Windows F:** and is used only for explicit Windows-host integration and the encrypted backup mirror.
> - Do **not** put live Hermes state, SQLite databases, Python environments, or the active workspace on `/mnt/f`.
>
> The guide therefore keeps the runtime under `/srv/vedha` and uses `/mnt/f` only where Windows-visible files are intentionally required.

```text
WINDOWS (F:)                                         PURPOSE
F:\project-vedha\
├── wsl\Ubuntu\ext4.vhdx                             Ubuntu distro disk — contains /srv/vedha (all live state)
├── docker-desktop-data\                             Docker Desktop managed disk (images, containers, volumes)
├── host\wsl\.wslconfig                              Canonical .wslconfig (deployed to %USERPROFILE%)
├── host\wsl\swap.vhdx                               WSL swap file
├── host\windows\                                    Exported Task Scheduler XML, PowerShell helpers
└── backups\                                         Encrypted backup mirror (age-encrypted; safe to be Windows-visible)

LINUX (inside the VHDX on F:)                        PURPOSE
/srv/vedha/
├── vedha-env.sh                                     Canonical environment file
├── hermes\            ($HERMES_HOME)                Hermes config, .env, memory, skills, sessions, state.db
├── workspace\         ($VEDHA_WORKSPACE)            Agent workspace (bind-mounted to /workspace in the sandbox)
├── gateway-cwd\                                     Neutral working directory for the gateway (no AGENTS.md)
├── ops\               ($VEDHA_OPS)                  Git repo: config fragments, persona, scripts, units, compose, docs
│   ├── bin\                                         vedha-* scripts and the hermes wrapper
│   ├── lib\                                         Shared shell library
│   ├── config\fragments\                            Per-stage config fragments (single source of truth)
│   ├── config\                                      cli-assumptions.txt, aux-routes.txt, keycheck-allow.txt, rendered-config.yaml
│   ├── persona\                                     SOUL.md, USER.md seed, workspace AGENTS.md
│   ├── skills\                                      Your own version-controlled skills (Stage 6.3)
│   ├── systemd\                                     Unit and timer files
│   ├── compose\                                     Kokoro compose file
│   ├── sandbox\                                     Custom sandbox Dockerfile
│   └── docs\                                        Ledgers: validation, auxiliary audit, drills, break-glass
├── source\hermes-agent-<tag>\  + source\current →   Pinned source checkouts (blue/green)
├── runtime\hermes-<tag>\       + runtime\current →  Per-release Python runtimes (blue/green)
├── runtime\voice-venv\                              Separate STT runtime
├── services\stt\                                    STT server and client (included in backups)
├── models\hf\                                       Hugging Face model cache (HF_HOME)
├── tools\                                           uv binary, uv-managed Pythons, yq
├── cache\uv\                                        uv cache
├── state\                                           Health state, endpoint snapshots, heartbeats, config history
├── backups\                                         Local encrypted backups
└── tmp\                                             Vedha temp files (ext4)
```

**Host-integration exceptions** (the OS decides where these live). Their canonical copies are kept under F: or in `ops/`:

| Item | Live location | Canonical copy |
|---|---|---|
| `.wslconfig` | `%USERPROFILE%\.wslconfig` | `F:\project-vedha\host\wsl\.wslconfig` |
| `/etc/wsl.conf`, journald drop-in | `/etc/...` (inside the VHDX on F:) | Backed up by `vedha-backup` |
| systemd user registrations | `~/.config/systemd/user` (symlinks created by `systemctl --user enable <path>`) | `ops/systemd/` |
| Task Scheduler task | Windows task store | `F:\project-vedha\host\windows\*.xml` |
| WSL, Docker Desktop and NVIDIA driver **programs** and their settings/logs | `C:\Program Files\…`, `%APPDATA%`, `%LOCALAPPDATA%` | Reinstallable; no Vedha data lives there |

> **BitLocker recommendation:** the VHDX holds your API keys, tokens and memory. Enable BitLocker on F: (or put F: on a BitLocker-protected disk).

> **Opening Vedha files from Windows:** in Explorer use `\\wsl.localhost\Ubuntu\srv\vedha\workspace`. For editing code, use an IDE in WSL mode (e.g. VS Code with the WSL extension) or open that path from your Windows IDE. Run builds and tests inside WSL or the sandbox, not through the `\\wsl.localhost` path from Windows; that crossing is slow. Never edit `$HERMES_HOME/.env` with Windows tools that might sync or index it.

> **Live-state rule:** do not place Hermes state, SQLite databases, runtimes, or the active workspace on `/mnt/f`. Use `/mnt/f` only for explicit Windows-host integration and the encrypted backup mirror described above.

---

## Stage 0 — Windows Host Preparation

### Objective
Prepare Windows to host an always-on assistant: current GPU driver, no sleep, WSL installed and **relocated to F:**, resource limits set, and Docker Desktop configured. Nothing Hermes-specific yet.

### Prerequisites
```text
[ ] Windows 11 with administrator access (see "Before you start" for opening PowerShell as Administrator)
[ ] F: drive present, NTFS, ideally internal (not removable) and BitLocker-protected
[ ] At least ~80 GB free on F: (distro, Docker images, models, backups)
[ ] Virtualization enabled in the BIOS/UEFI (Task Manager → Performance → CPU shows "Virtualization: Enabled")
```

### Steps

**0.1 — 🪟 Windows GUI: install the current NVIDIA Windows driver**
1. Open https://www.nvidia.com/Download/index.aspx (or the NVIDIA App), choose **GeForce → GTX 16 Series → GTX 1660 SUPER → Windows 11**, and install the current **Game Ready** or **Studio** driver.
2. Reboot when asked.

**The Windows driver provides CUDA to WSL2.** **Never install a Linux NVIDIA driver inside WSL**, because it breaks GPU passthrough.

**0.2 — 🪟 PowerShell (Administrator): power settings for always-on**
```powershell
powercfg /change standby-timeout-ac 0
powercfg /change hibernate-timeout-ac 0
powercfg /change monitor-timeout-ac 15
```
Then in Settings → Windows Update → Advanced options → **Active hours**, set the hours Veda must be reachable. A Windows Update reboot is a planned outage, so Stage 9 makes sure everything comes back afterwards.

**0.3 — 🪟 PowerShell (Administrator): install and update WSL**
```powershell
wsl --install -d Ubuntu
```
Reboot if prompted. Launch **Ubuntu** once from the Start menu and create your Linux **username** (lowercase, no spaces) and **password**. Write the password into your password manager; `sudo` asks for it. Then, back in PowerShell:
```powershell
wsl --update
wsl --set-default-version 2
wsl --version
wsl -l -v
```
Record the WSL version. The distro-move command in 0.4 needs a recent WSL **[VERIFY: `wsl --manage <distro> --move` availability on your build]**.

Distro names and versions change over time, so don't hard-code an Ubuntu release. Check `wsl --list --online` before installing. Later, inside Ubuntu, record the exact release with `lsb_release -ds`.

**0.4 — 🪟 PowerShell: move the Ubuntu distro onto F:**
```powershell
New-Item -ItemType Directory -Force -Path 'F:\project-vedha\wsl' | Out-Null
wsl --shutdown
wsl --manage Ubuntu --move 'F:\project-vedha\wsl\Ubuntu'
wsl -l -v
```
**Fallback if `--manage --move` is not available** (export/import). Run these lines **one at a time** and check each result before continuing, because `--unregister` deletes the original:
```powershell
wsl --shutdown
wsl --export Ubuntu 'F:\project-vedha\wsl\ubuntu-export.tar'
# STOP: confirm the .tar exists and is several hundred MB+ before the next line.
Get-Item 'F:\project-vedha\wsl\ubuntu-export.tar' | Select-Object Length
wsl --unregister Ubuntu
wsl --import Ubuntu 'F:\project-vedha\wsl\Ubuntu' 'F:\project-vedha\wsl\ubuntu-export.tar' --version 2
```
After an import, the distro logs in as `root` until Stage 1.1 sets `[user] default=` in `/etc/wsl.conf`. Stage 1.1 detects this automatically and picks the user you created in 0.3. Delete the `.tar` once Stage 1 is verified, because it contains everything in the distro.

Verify the disk now lives on F:
```powershell
Get-ChildItem 'F:\project-vedha\wsl\Ubuntu'
```
Expected: an `ext4.vhdx` file.

Optional: let the VHDX return free space to F: **[VERIFY availability]**:
```powershell
wsl --shutdown
wsl --manage Ubuntu --set-sparse true
```

**0.5 — 🪟 PowerShell: create and deploy `.wslconfig`**

Memory budget (see Appendix L): with 16 GB total, give WSL 10 GB and leave about 6 GB for Windows. If Windows feels starved, drop to 9 GB and run Kokoro on CPU or on demand.

```powershell
$canonical = 'F:\project-vedha\host\wsl\.wslconfig'
New-Item -ItemType Directory -Force -Path (Split-Path $canonical) | Out-Null

@'
[wsl2]
# WSL2 VM budget on a 16 GB host. Docker Desktop (WSL2 backend) containers share this budget.
memory=10GB
# i3-10105F = 4 cores / 8 threads; expose 6 logical processors.
processors=6
swap=6GB
# Keep swap on F: (double backslashes are required in .wslconfig paths).
swapFile=F:\\project-vedha\\host\\wsl\\swap.vhdx
# Default NAT networking is the validated baseline.
# networkingMode=mirrored

[experimental]
autoMemoryReclaim=dropcache
'@ | Set-Content -Path $canonical -Encoding ascii

Copy-Item -LiteralPath $canonical -Destination (Join-Path $env:USERPROFILE '.wslconfig') -Force
wsl --shutdown
```
To redeploy after you change it:
```powershell
Copy-Item -LiteralPath 'F:\project-vedha\host\wsl\.wslconfig' -Destination (Join-Path $env:USERPROFILE '.wslconfig') -Force
wsl --shutdown
```

**0.6 — 🪟 Windows GUI: install and configure Docker Desktop**
1. Download Docker Desktop for Windows from https://www.docker.com/products/docker-desktop/ and run the installer. Keep "Use WSL 2 instead of Hyper-V" checked. Sign-in to a Docker account is optional; you can skip it.
2. Settings → General: **Use the WSL 2 based engine** ✔, **Start Docker Desktop when you sign in to your computer** ✔.
3. Settings → Resources → WSL Integration: enable **Ubuntu** → Apply & Restart. (If you moved or imported the distro after first enabling this, toggle it off and on again.)
4. Settings → Resources → Advanced → **Disk image location** = `F:\project-vedha\docker-desktop-data` → Apply. Let Docker move it; never move the disk image by hand.

**0.7 — 🪟 Windows GUI: Telegram account hardening (do it now, before you ever need it)**
Telegram → Settings → Privacy and Security → **Two-Step Verification**: set a cloud password. Your Telegram account becomes a remote control for an agent that can run tools.

### Verification
```powershell
wsl -l -v                      # Ubuntu, VERSION 2
Get-ChildItem 'F:\project-vedha\wsl\Ubuntu\ext4.vhdx'
Get-Content "$env:USERPROFILE\.wslconfig"
powercfg /query SCHEME_CURRENT SUB_SLEEP STANDBYIDLE
```
Docker Desktop shows "Engine running".

### Expected result
- `wsl -l -v` lists **Ubuntu** with **VERSION 2**.
- `F:\project-vedha\wsl\Ubuntu\ext4.vhdx` exists.
- `.wslconfig` shows `memory=10GB` and the swap path on F:.
- The power query shows the AC standby timeout as `0x00000000`.
- Docker Desktop shows "Engine running" with Ubuntu integration enabled.

### Troubleshooting
- **`wsl --install` says virtualization is disabled** — enable Intel VT-x in the BIOS/UEFI, then retry.
- **`wsl --manage … --move` is not recognized** — run `wsl --update`, then retry, or use the export/import fallback in 0.4.
- **Docker Desktop: "WSL 2 installation is incomplete"** — run `wsl --update` in PowerShell (Administrator) and restart Docker Desktop.
- **Ubuntu is missing in Docker Desktop → Resources → WSL Integration** — open Ubuntu once, then restart Docker Desktop and reopen that page.

### Rollback / recovery
- Distro move: `wsl --manage Ubuntu --move <old path>` moves it back.
- If export/import went wrong, the `.tar` export is your recovery copy. **Keep it until Stage 1 is verified.**
- `.wslconfig`: delete `%USERPROFILE%\.wslconfig` and run `wsl --shutdown`.

### Completion checklist
```text
[ ] Current NVIDIA Windows driver installed; no Linux NVIDIA driver planned
[ ] Sleep/hibernate disabled on AC; Windows Update active hours set
[ ] WSL updated; Ubuntu user created
[ ] Ubuntu distro VHDX located under F:\project-vedha\wsl\Ubuntu
[ ] Canonical .wslconfig on F: and deployed to %USERPROFILE%
[ ] Docker Desktop: WSL2 engine, starts at sign-in, Ubuntu integration on, disk image on F:
[ ] Telegram two-step verification enabled
[ ] (Recommended) BitLocker enabled on F:
```

---
## Stage 1 — WSL Foundation, Storage Root, Ops Toolkit & Pinned Hermes Install

### Objective
Prove the platform works: systemd, time, packages, Docker, and the GPU in containers. Then create the `/srv/vedha` root, the ops toolkit, and a **reproducible, release-pinned** Hermes install. No model, tools, or voice yet.

### Prerequisites
```text
[ ] Stage 0 complete
```

> **About the code blocks below.** Blocks that create files with `cat > file <<'VEDHA_EOF'` are safe to paste: they only *write* a script, they don't run it. Run the scripts afterwards by name.

### Steps

**1.1 — Configure `/etc/wsl.conf` (systemd, interop, default user, sane DrvFS permissions)**

Open the **Ubuntu** terminal and paste this whole block. It picks your normal user automatically, even if the distro currently logs in as `root` (after the 0.4 export/import fallback).
```bash
VEDHA_LINUX_USER="${SUDO_USER:-$USER}"
[[ "$VEDHA_LINUX_USER" == root ]] && VEDHA_LINUX_USER="$(getent passwd 1000 | cut -d: -f1)"
echo "Default WSL user will be: ${VEDHA_LINUX_USER:-<none found>}"
if [[ -n "$VEDHA_LINUX_USER" && "$VEDHA_LINUX_USER" != root ]]; then
sudo tee /etc/wsl.conf >/dev/null <<VEDHA_EOF
[boot]
systemd=true

[user]
default=${VEDHA_LINUX_USER}

[interop]
enabled=true
# Deterministic PATH: Windows tools are called by absolute path (e.g. /mnt/c/Windows/System32/...).
appendWindowsPath=false

[automount]
# Files without stored metadata: 0644; directories: 0755.
options = "metadata,umask=022,fmask=111"
VEDHA_EOF
echo "/etc/wsl.conf written."
else echo "No non-root user found: run 'adduser <name> && usermod -aG sudo <name>', then repeat 1.1."; fi
```
🪟 PowerShell: `wsl --shutdown`, then reopen Ubuntu.
```bash
ps -p 1 -o comm=          # expected: systemd
whoami                    # expected: your normal user, not root
```
If `whoami` still prints `root`, run 🪟 `wsl --shutdown` again and reopen Ubuntu. Don't continue as root: every later step assumes your normal user.
> v1 used `fmask=11`, which leaves DrvFS files group/other-writable. `appendWindowsPath=false` stops Windows `python.exe`/`node.exe` from shadowing Linux tools. If you want VS Code's `code` command, add an alias to its absolute path.

**1.2 — Time zone and time sync**
⚠️ **USER INPUT REQUIRED** — `[YOUR_TZ]` (IANA name, e.g. `Asia/Kolkata`).
```bash
sudo timedatectl set-timezone [YOUR_TZ]
sudo timedatectl set-ntp true
timedatectl
```
WSL2's clock can drift after the host sleeps. That breaks cron timing and TLS. `vedha-clock-check` (1.9) detects drift.

**1.3 — Base packages (including everything later stages need)**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl ca-certificates xz-utils jq sqlite3 age gzip zip unzip file bc nano \
  logrotate ffmpeg portaudio19-dev libopus0 espeak-ng pulseaudio-utils
```
| Package | Used by |
|---|---|
| `git`, `curl`, `ca-certificates`, `xz-utils` | Hermes install, downloads |
| `jq`, `sqlite3`, `bc`, `file` | Scripts, database checks, latency maths, file-type checks |
| `age`, `gzip`, `zip`, `unzip` | Encrypted backups |
| `logrotate` | Log rotation (Stage 11) |
| `ffmpeg`, `portaudio19-dev`, `libopus0`, `espeak-ng`, `pulseaudio-utils` | Voice and image probes (Stages 2, 12) |
| `nano` | Editing files when a step asks you to |

`yq` and `uv` are installed in 1.8. `docker` comes from Docker Desktop's WSL integration (Stage 0.6), never from `apt`.

**1.4 — Let user services run without an interactive login**
```bash
sudo loginctl enable-linger "$USER"
loginctl show-user "$USER" -p Linger      # expected: Linger=yes
```
Without linger, `systemctl --user` services do not start at boot and stop when your last session ends. v1 was missing this step.

**1.5 — Bound the system journal**
```bash
sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nSystemMaxUse=500M\nSystemMaxFileSize=50M\n' | sudo tee /etc/systemd/journald.conf.d/vedha.conf >/dev/null
sudo systemctl restart systemd-journald
```

**1.6 — Verify Docker and GPU passthrough**
```bash
docker --version
docker run --rm hello-world
docker run --rm --gpus all nvidia/cuda:12.6.0-base-ubuntu24.04 nvidia-smi
```
Expected: the last command prints the GTX 1660 Super. WSL2 + Docker Desktop needs **no** NVIDIA Container Toolkit inside Ubuntu. **[VERIFY]** that the `nvidia/cuda:12.6.0-base-ubuntu24.04` tag exists. If it doesn't, use any current `base-ubuntu24.04` tag.

If this fails: update Docker Desktop and the Windows NVIDIA driver, `wsl --shutdown`, and retry. Do not continue until it passes. Stage 12 depends on it.

**1.7 — Create the storage root and environment file**
```bash
sudo mkdir -p /srv/vedha
sudo chown "$USER:$USER" /srv/vedha
chmod 750 /srv/vedha

mkdir -p /srv/vedha/{hermes,workspace,gateway-cwd,source,runtime,services/stt,models/hf,tools/bin,tools/uv-python,cache/uv,state/config-history,state/endpoints,state/heartbeat,backups,tmp}
mkdir -p /srv/vedha/ops/{bin,lib,config/fragments,persona,skills,systemd,compose,sandbox,docs,logrotate,backup}

cat > /srv/vedha/vedha-env.sh <<'VEDHA_EOF'
# Project Vedha canonical environment. Sourced by shells, the hermes wrapper, and every vedha-* script.
export VEDHA_ROOT="/srv/vedha"
export VEDHA_OPS="$VEDHA_ROOT/ops"
# Overrides exist ONLY for the restore drill and update staging; never set them in your profile.
export HERMES_HOME="${VEDHA_HERMES_HOME_OVERRIDE:-$VEDHA_ROOT/hermes}"
export VEDHA_RUNTIME="${VEDHA_RUNTIME_OVERRIDE:-$VEDHA_ROOT/runtime/current}"
export VEDHA_SOURCE="${VEDHA_SOURCE_OVERRIDE:-$VEDHA_ROOT/source/current}"
export VEDHA_WORKSPACE="$VEDHA_ROOT/workspace"
export VEDHA_MODELS="$VEDHA_ROOT/models"
export VEDHA_STATE="$VEDHA_ROOT/state"
export VEDHA_BACKUPS="$VEDHA_ROOT/backups"
export VEDHA_HOST_ROOT="/mnt/f/project-vedha/host"
export VEDHA_BACKUP_MIRROR="/mnt/f/project-vedha/backups"   # Windows-visible encrypted backup mirror only
export VEDHA_TMP="$VEDHA_ROOT/tmp"

# Site settings. The health/digest/backup TIMERS read this file, not your interactive shell, so put
# overrides here (uncomment as needed). Each one is explained in the stage that introduces it.
# export VEDHA_HEARTBEAT_FILE="$VEDHA_ROOT/workspace/.heartbeat"   # Stage 8.4 fallback only
# export VEDHA_MAX_CRON_JOBS=15                                    # Stage 13.2 option B
# export VEDHA_DOCKER_GRACE=3600                                   # Stage 11.5: seconds Docker may be down before FAIL
# export VEDHA_BACKUP_KEEP=14                                      # Final B.2: archives kept locally and on F:

# Toolchain locations (kept inside the Vedha root for tidiness).
export UV_INSTALL_DIR="$VEDHA_ROOT/tools/bin"
export UV_PYTHON_INSTALL_DIR="$VEDHA_ROOT/tools/uv-python"
export UV_CACHE_DIR="$VEDHA_ROOT/cache/uv"
export HF_HOME="$VEDHA_MODELS/hf"

# PATH: ops scripts + tools only. The Hermes runtime's bin/ is deliberately NOT on PATH
# (that would silently activate the Hermes venv for every shell). TMPDIR is not changed globally.
case ":$PATH:" in
  *":$VEDHA_OPS/bin:"*) ;;
  *) export PATH="$VEDHA_OPS/bin:$VEDHA_ROOT/tools/bin:$PATH" ;;
esac
VEDHA_EOF
chmod 644 /srv/vedha/vedha-env.sh

for f in "$HOME/.profile" "$HOME/.bashrc"; do
  grep -qxF '[ -f /srv/vedha/vedha-env.sh ] && . /srv/vedha/vedha-env.sh' "$f" 2>/dev/null \
    || printf '\n# Project Vedha\n[ -f /srv/vedha/vedha-env.sh ] && . /srv/vedha/vedha-env.sh\n' >> "$f"
done
. /srv/vedha/vedha-env.sh
echo "$VEDHA_ROOT $HERMES_HOME"      # expected: /srv/vedha /srv/vedha/hermes
```

Create the ledgers (plain Markdown files the stages append to). Re-running this is safe: existing files are kept.
```bash
D=/srv/vedha/ops/docs
[ -f "$D/validation-ledger.md" ] || printf '# Validation ledger (release assumptions, pins, measurements)\n\nkey=value lines and dated notes; newest last.\n\n' > "$D/validation-ledger.md"
[ -f "$D/integrations.md" ] || printf '# Integrations (skills, MCP servers, platform toolsets)\n| Date | Type | ID/name | Version/commit | Why |\n|---|---|---|---|---|\n' > "$D/integrations.md"
[ -f "$D/break-glass.md" ] || printf '# Break-glass log (Stage 11.12)\n| Opened | Reason | Provider/routes changed | Expected duration | Closed |\n|---|---|---|---|---|\n' > "$D/break-glass.md"
ls -l "$D"
```

**1.8 — Install tools: uv and yq**
```bash
. /srv/vedha/vedha-env.sh
# uv (official installer, redirected into the Vedha root).
# Pin it: use the exact version validated for this guide [VERIFY row 29].
# Do not leave this empty in the canonical build; an unpinned bootstrap weakens reproducibility.
UV_VERSION="0.9.5"   # replace only after recording the validated version in Appendix J
curl -LsSf "https://astral.sh/uv/${UV_VERSION}/install.sh" | env UV_INSTALL_DIR="$VEDHA_ROOT/tools/bin" UV_NO_MODIFY_PATH=1 sh
uv --version
uv python dir     # expected: /srv/vedha/tools/uv-python
uv cache dir      # expected: /srv/vedha/cache/uv

# yq (mikefarah, pinned) — used to merge config fragments
YQ_VERSION="v4.44.3"   # [VERIFY row 29] pick the current v4 release
curl -fL -o "$VEDHA_ROOT/tools/bin/yq" "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/yq_linux_amd64"
sha256sum "$VEDHA_ROOT/tools/bin/yq"   # compare with the SHA-256 published on the release page; stop if different
chmod 755 "$VEDHA_ROOT/tools/bin/yq"
yq --version

# Record both versions (Appendix J "Version pinning record")
printf 'uv=%s\nyq=%s yq_sha256=%s\n' "$(uv --version)" "$YQ_VERSION" "$(sha256sum "$VEDHA_ROOT/tools/bin/yq" | cut -c1-64)" \
  >> /srv/vedha/ops/docs/validation-ledger.md
```

**1.9 — Initialize the ops repo and core toolkit**

*1.9a — Git repo (no secrets ever enter it):*
```bash
cd /srv/vedha/ops
git init -q
git config user.name "Vedha Ops"
git config user.email "vedha-ops@localhost"
cat > .gitignore <<'VEDHA_EOF'
.env
*.env
*.key
*identity*
backup/*.txt.private
VEDHA_EOF
cd - >/dev/null
```

*1.9b — Shared library:*
```bash
cat > /srv/vedha/ops/lib/vedha-common.sh <<'VEDHA_EOF'
# shellcheck shell=bash
# Shared helpers for vedha-* scripts. Source after /srv/vedha/vedha-env.sh.

VEDHA_FAIL=0
VEDHA_WARN=0
pass() { printf '  PASS  %s\n' "$*"; }
warn() { printf '  WARN  %s\n' "$*"; VEDHA_WARN=$((VEDHA_WARN + 1)); }
fail() { printf '  FAIL  %s\n' "$*"; VEDHA_FAIL=$((VEDHA_FAIL + 1)); }
summary() { printf '\nSummary: FAIL=%d WARN=%d\n' "$VEDHA_FAIL" "$VEDHA_WARN"; [[ $VEDHA_FAIL -eq 0 ]]; }

# Read KEY from a dotenv file WITHOUT sourcing it (no code execution, no export).
env_get() {
  local key="$1" file="${2:-$HERMES_HOME/.env}" line
  [[ -r "$file" ]] || return 1
  line="$(grep -E "^[[:space:]]*${key}[[:space:]]*=" "$file" | tail -n 1)" || return 1
  line="${line#*=}"
  line="${line#"${line%%[![:space:]]*}"}"; line="${line%"${line##*[![:space:]]}"}"
  line="${line%\"}"; line="${line#\"}"; line="${line%\'}"; line="${line#\'}"
  [[ -n "$line" ]] || return 1
  printf '%s' "$line"
}

# Insert or replace KEY in a dotenv file. The VALUE is read from stdin, so it never appears on a command line.
# Usage: read -rsp "Token: " T; echo; printf '%s\n' "$T" | env_set SOME_TOKEN; unset T
env_set() {
  local key="$1" file="${2:-$HERMES_HOME/.env}" val tmp
  IFS= read -r val || true
  [[ -n "$val" ]] || { echo "env_set: empty value for $key" >&2; return 1; }
  tmp="$(mktemp "$file.XXXXXX")" || return 1
  { grep -vE "^[[:space:]]*${key}[[:space:]]*=" "$file" 2>/dev/null; printf '%s=%s\n' "$key" "$val"; } > "$tmp"
  chmod 600 "$tmp" && mv -f -- "$tmp" "$file" && echo "$key stored in $file (600)"
}

# curl to OpenRouter with the key passed via a file descriptor (not visible in `ps`).
or_api() {
  local key
  key="$(env_get OPENROUTER_API_KEY)" || { echo "OPENROUTER_API_KEY not found in $HERMES_HOME/.env" >&2; return 1; }
  curl -sS --max-time 120 -H @<(printf 'Authorization: Bearer %s\n' "$key") "$@"
}

unit_enabled() { systemctl --user is-enabled -q "$1" 2>/dev/null; }
unit_active()  { systemctl --user is-active  -q "$1" 2>/dev/null; }
VEDHA_EOF
```

*1.9c — The `hermes` wrapper, with guards against a wrong or empty home:*
```bash
cat > /srv/vedha/ops/bin/hermes <<'VEDHA_EOF'
#!/usr/bin/env bash
# Project Vedha hermes wrapper: fixed environment, guarded home, pinned runtime.
set -euo pipefail
. /srv/vedha/vedha-env.sh
if [[ ! -f "$HERMES_HOME/.vedha-home" ]]; then
  echo "ERROR: $HERMES_HOME is not a Project Vedha Hermes home (.vedha-home marker missing). Refusing to start" >&2
  echo "       rather than letting Hermes create a fresh, empty home." >&2
  exit 78
fi
if [[ ! -x "$VEDHA_RUNTIME/bin/hermes" ]]; then
  echo "ERROR: Hermes runtime not found at $VEDHA_RUNTIME (is runtime/current set?)" >&2
  exit 78
fi
export TMPDIR="$VEDHA_TMP"
exec "$VEDHA_RUNTIME/bin/hermes" "$@"
VEDHA_EOF
chmod 755 /srv/vedha/ops/bin/hermes
```
> v1's wrapper hard-coded `HERMES_HOME`. Its restore drill (`HERMES_HOME=<restored> hermes ...`) therefore silently ran against the **live** home. v2 uses the explicit `VEDHA_HERMES_HOME_OVERRIDE` variable instead.

*1.9d — Filesystem smoke test (the test v1 promised but never included):*
```bash
cat > /srv/vedha/ops/bin/vedha-fs-smoke <<'VEDHA_EOF'
#!/usr/bin/env bash
# vedha-fs-smoke [DIR] — SQLite WAL, concurrent writers, locking, permissions, symlinks, rename.
set -uo pipefail
. /srv/vedha/vedha-env.sh; . "$VEDHA_OPS/lib/vedha-common.sh"
DIR="${1:-$HERMES_HOME}"
T="$(mktemp -d "$DIR/.fs-smoke.XXXXXX")" || { echo "cannot create temp dir in $DIR"; exit 1; }
trap 'rm -rf -- "$T"' EXIT
echo "Filesystem smoke test on: $DIR ($(stat -f -c %T "$DIR"))"
DB="$T/smoke.db"
[[ "$(sqlite3 "$DB" 'PRAGMA journal_mode=WAL;')" == "wal" ]] && pass "WAL journal mode" || fail "WAL journal mode unavailable"
sqlite3 "$DB" 'CREATE TABLE t(w INTEGER, i INTEGER);'
for w in 1 2 3 4; do
  ( for i in $(seq 1 250); do sqlite3 -cmd '.timeout 10000' "$DB" "INSERT INTO t VALUES($w,$i);" >/dev/null || exit 1; done ) &
done
wait
n="$(sqlite3 "$DB" 'SELECT count(*) FROM t;')"
[[ "$n" == "1000" ]] && pass "4 concurrent writers, 1000 rows" || fail "concurrent writers produced $n rows (expected 1000)"
[[ "$(sqlite3 "$DB" 'PRAGMA integrity_check;')" == "ok" ]] && pass "integrity_check ok" || fail "integrity_check failed"
: > "$T/lock"; exec 9>"$T/lock"
flock -n 9 && pass "flock acquired" || fail "flock unavailable"
( exec 8>"$T/lock"; flock -n 8 ) && fail "flock not exclusive" || pass "flock exclusive"
touch "$T/s"; chmod 600 "$T/s"; [[ "$(stat -c %a "$T/s")" == "600" ]] && pass "chmod 600 honored" || fail "chmod not honored"
ln -s s "$T/l" 2>/dev/null && [[ -L "$T/l" ]] && pass "symlinks" || fail "symlinks unsupported"
echo x > "$T/a" && mv "$T/a" "$T/b" && [[ -f "$T/b" ]] && pass "atomic rename" || fail "rename failed"
summary
VEDHA_EOF
chmod 755 /srv/vedha/ops/bin/vedha-fs-smoke
```

*1.9e — Config fragment merger (single source of truth):*
```bash
cat > /srv/vedha/ops/bin/vedha-config-apply <<'VEDHA_EOF'
#!/usr/bin/env bash
# vedha-config-apply [--yes] [fragment ...]
# Deep-merges fragments (default: all, in version order) ONTO the live config.yaml, preserving keys
# Hermes itself wrote. Maps merge recursively; lists are replaced. Shows a diff, validates, snapshots.
set -euo pipefail
. /srv/vedha/vedha-env.sh
YES=0; [[ "${1:-}" == "--yes" ]] && { YES=1; shift; }
CFG="$HERMES_HOME/config.yaml"; FR="$VEDHA_OPS/config/fragments"
files=()
if (($#)); then
  for f in "$@"; do [[ -f "$f" ]] && files+=("$f") || files+=("$FR/$f"); done
else
  while IFS= read -r f; do files+=("$f"); done < <(ls "$FR"/*.yaml 2>/dev/null | sort -V)
fi
((${#files[@]})) || { echo "No fragments to apply."; exit 0; }
for f in "${files[@]}"; do [[ -f "$f" ]] || { echo "Missing fragment: $f" >&2; exit 1; }; done
[[ -s "$CFG" ]] || echo '{}' > "$CFG"
ts="$(date +%Y%m%d-%H%M%S)"
cp -p "$CFG" "$VEDHA_STATE/config-history/config.$ts.yaml"
tmp="$(mktemp "$VEDHA_TMP/config.XXXXXX.yaml")"; trap 'rm -f -- "$tmp"' EXIT
yq eval-all '. as $item ireduce ({}; . * $item)' "$CFG" "${files[@]}" > "$tmp"
if diff -u "$CFG" "$tmp"; then echo "No changes."; exit 0; fi
if ((YES == 0)); then read -rp "Apply this change to $CFG? [y/N] " a; [[ "$a" == "y" ]] || { echo "Aborted."; exit 1; }; fi
install -m 600 "$tmp" "$CFG"
if ! hermes config check; then
  echo "config check FAILED — restoring previous config from $VEDHA_STATE/config-history/config.$ts.yaml" >&2
  install -m 600 "$VEDHA_STATE/config-history/config.$ts.yaml" "$CFG"; exit 1
fi
# Never commit secrets: wizards (e.g. `hermes gateway setup`) may write tokens into config.yaml.
if grep -qE 'sk-or-v1-[A-Za-z0-9]{16,}|[0-9]{8,12}:[A-Za-z0-9_-]{30,}|xox[abp]-[A-Za-z0-9-]+|xapp-[A-Za-z0-9-]+' "$CFG"; then
  echo "Applied, but NOT committed: config.yaml contains a secret-looking value. Move it to .env, then re-run." >&2; exit 1
fi
# Generic key-name scan: reject literal values beneath secret-like keys, but allow environment-variable references.
while IFS=$'	' read -r path value; do
  leaf="${path##*.}"
  case "$leaf" in
    api_key|token|secret|password|private_key|access_key|credential)
      [[ -z "$value" || "$value" == "null" ]] && continue
      [[ "$value" =~ ^\$\{?[A-Z][A-Z0-9_]*\}?$ ]] && continue
      echo "Applied, but NOT committed: literal value found under secret-like key $path. Use .env / *_env instead." >&2
      exit 1
      ;;
  esac
done < <(yq -r 'paths(scalars) as $p | [($p | map(tostring) | join(".")), (getpath($p) | tostring)] | @tsv' "$CFG")
cp "$CFG" "$VEDHA_OPS/config/rendered-config.yaml"
git -C "$VEDHA_OPS" add -A
git -C "$VEDHA_OPS" commit -qm "config: apply $(printf '%s ' "${files[@]##*/}")" || true
echo "Applied. Restart long-running Hermes processes (gateway, open sessions) to pick it up."
VEDHA_EOF
chmod 755 /srv/vedha/ops/bin/vedha-config-apply
```
> `yq` re-serializes the file, so comments in the live `config.yaml` are not preserved. The comments live in the fragments instead.

> **Removing a setting (important).** Fragments merge *onto* the live config, so they can only **add or override** keys. Deleting a key from a fragment, or deleting a whole fragment, does **not** remove it from `config.yaml`. To remove a key:
> ```bash
> # 1. delete the key from its fragment (or git rm the fragment), then:
> yq -i 'del(.<path.to.key>)' "$HERMES_HOME/config.yaml"
> hermes config check && cp "$HERMES_HOME/config.yaml" /srv/vedha/ops/config/rendered-config.yaml
> git -C /srv/vedha/ops add -A && git -C /srv/vedha/ops commit -qm "config: remove <path.to.key>"
> ```
> Likewise, if you restore an old `config.yaml` from `state/config-history/`, also revert the fragment (`git -C /srv/vedha/ops checkout <good-commit> -- config/fragments/<file>`). Otherwise the next `vedha-config-apply` re-applies it.

*1.9f — Release-assumption checker (config keys and CLI flags checked against the pinned source):*
```bash
cat > /srv/vedha/ops/config/cli-assumptions.txt <<'VEDHA_EOF'
# <hermes subcommand words>|<string that must appear in its --help output>
chat|--oneshot
gateway run|--external-supervisor
cron create|--script
cron create|--no-agent
tools|--summary
tools list|--platform
mcp add|--url
config|check
config|set
fallback|list
skills|inspect
skills|audit
memory|status
VEDHA_EOF

# Map keys that are names YOU choose (STT provider name, MCP server names); keycheck skips them.
[ -f /srv/vedha/ops/config/keycheck-allow.txt ] || printf '# one user-chosen config key name per line (Stage 6.4, 12.8)\n' > /srv/vedha/ops/config/keycheck-allow.txt

cat > /srv/vedha/ops/bin/vedha-keycheck <<'VEDHA_EOF'
#!/usr/bin/env bash
# vedha-keycheck [config.yaml] — every map key in the config must appear in the pinned Hermes source,
# and every CLI flag this guide relies on must appear in --help. Catches keys `config check` may ignore.
set -uo pipefail
. /srv/vedha/vedha-env.sh; . "$VEDHA_OPS/lib/vedha-common.sh"
CFG="${1:-$HERMES_HOME/config.yaml}"
[[ -d "$VEDHA_SOURCE" ]] || { echo "Source not found: $VEDHA_SOURCE"; exit 1; }
echo "== Config keys vs source ($VEDHA_SOURCE) =="
while IFS= read -r k; do
  [[ -z "$k" || "$k" == */* || "$k" =~ ^[0-9]+$ ]] && continue   # skip model slugs and indices
  grep -qxF -- "$k" "$VEDHA_OPS/config/keycheck-allow.txt" 2>/dev/null && continue   # names you chose (STT provider, MCP servers)
  if grep -rqsF --include='*.py' -- "$k" "$VEDHA_SOURCE"; then :; else fail "config key not found in source: $k"; fi
done < <(yq '.. | select(tag == "!!map") | keys | .[]' "$CFG" | sort -u)
echo "== Auxiliary route completeness (vs ops/config/aux-routes.txt, Stage 2.4a) =="
AUXL="$VEDHA_OPS/config/aux-routes.txt"
if [[ "$(yq '.auxiliary // {} | length' "$CFG")" == 0 ]]; then warn "no auxiliary block yet (expected before Stage 2.5)"
elif [[ -s "$AUXL" ]]; then
  missing="$(comm -23 <(grep -vE '^[[:space:]]*(#|$)' "$AUXL" | sort -u) <(yq '.auxiliary // {} | keys | .[]' "$CFG" | sort -u) | paste -sd, -)"
  [[ -z "$missing" ]] && pass "every source auxiliary route is pinned" || fail "UNPINNED auxiliary routes: $missing"
else fail "ops/config/aux-routes.txt missing — enumerate the routes from the source (Stage 2.4a)"; fi
echo "== CLI assumptions =="
while IFS='|' read -r sub needle; do
  [[ -z "$sub" || "$sub" == \#* ]] && continue
  # shellcheck disable=SC2086
  if hermes $sub --help 2>&1 | grep -qF -- "$needle"; then pass "hermes $sub ... $needle"; else fail "hermes $sub --help lacks: $needle"; fi
done < "$VEDHA_OPS/config/cli-assumptions.txt"
echo "(A key found in source is necessary, not sufficient: confirm semantics in docs/source for [VERIFY] items.)"
summary
VEDHA_EOF
chmod 755 /srv/vedha/ops/bin/vedha-keycheck
```

*1.9g — OpenRouter endpoint gate (replaces v1's `grep -qi coreweave`):*
```bash
cat > /srv/vedha/ops/bin/vedha-endpoint-check <<'VEDHA_EOF'
#!/usr/bin/env bash
# vedha-endpoint-check <model-slug> [provider-regex]
# Fails unless OpenRouter lists an endpoint from the provider that supports the parameters this build
# requires (require_parameters=true would otherwise reject every request). Saves a snapshot.
set -uo pipefail
. /srv/vedha/vedha-env.sh
MODEL="${1:?model slug}"; RE="${2:-coreweave}"
REQUIRED="${VEDHA_REQUIRED_PARAMS:-tools tool_choice reasoning}"
OPTIONAL="${VEDHA_OPTIONAL_PARAMS:-response_format structured_outputs}"
OUT="$VEDHA_STATE/endpoints/$(printf '%s' "$MODEL" | tr '/:' '__').json"
RAW="$(curl -fsS --max-time 20 "https://openrouter.ai/api/v1/models/$MODEL/endpoints")" \
  || { echo "FAIL: cannot query OpenRouter endpoints for $MODEL"; exit 2; }
EP="$(jq -c --arg re "$RE" '[.data.endpoints[]? | select(((.provider_name // "") | test($re; "i")) or ((.tag // "") | test($re; "i")))]' <<<"$RAW")"
[[ "$(jq 'length' <<<"$EP")" -gt 0 ]] || { echo "FAIL: no endpoint matching /$RE/ for $MODEL"; exit 1; }
jq '.[0] | {provider_name, tag, context_length, max_completion_tokens, quantization, status, supported_parameters}' <<<"$EP" | tee "$OUT"
[[ -f "${OUT%.json}.baseline.json" ]] || cp "$OUT" "${OUT%.json}.baseline.json"   # first snapshot kept as the baseline (health compares)
rc=0
for p in $REQUIRED; do
  jq -e --arg p "$p" '.[0].supported_parameters // [] | index($p) != null' <<<"$EP" >/dev/null \
    || { echo "FAIL: endpoint lacks required parameter: $p"; rc=1; }
done
for p in $OPTIONAL; do
  jq -e --arg p "$p" '.[0].supported_parameters // [] | index($p) != null' <<<"$EP" >/dev/null \
    || echo "WARN: endpoint does not advertise optional parameter: $p"
done
echo "Provider slug/tag to use in 'only:' → $(jq -r '.[0].tag // .[0].provider_name' <<<"$EP")"
[[ $rc -eq 0 ]] && echo "PASS: $MODEL usable on /$RE/ (context_length=$(jq '.[0].context_length' <<<"$EP"))"
exit $rc
VEDHA_EOF
chmod 755 /srv/vedha/ops/bin/vedha-endpoint-check
```
**[VERIFY]** the response field names (`data.endpoints[].provider_name/tag/context_length/supported_parameters`) against one live response: `curl -s https://openrouter.ai/api/v1/models/z-ai/glm-5.3-flash/endpoints | jq .`

*1.9h — Docker readiness wait and clock check:*
```bash
cat > /srv/vedha/ops/bin/vedha-wait-docker <<'VEDHA_EOF'
#!/usr/bin/env bash
# vedha-wait-docker [seconds] — wait for Docker Desktop; never fails (gateway can still chat without tools).
timeout="${1:-300}"; end=$((SECONDS + timeout))
until docker info >/dev/null 2>&1; do
  if (( SECONDS >= end )); then echo "WARN: Docker not ready after ${timeout}s; starting degraded (no sandbox tools)" >&2; exit 0; fi
  sleep 5
done
echo "Docker ready"
VEDHA_EOF

cat > /srv/vedha/ops/bin/vedha-clock-check <<'VEDHA_EOF'
#!/usr/bin/env bash
# vedha-clock-check [max-skew-seconds] — compares WSL clock with Windows clock.
# Exit codes: 0 = within limit, 1 = skew too large, 2 = could not check (callers report WARN, never PASS).
PS=/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe
[[ -x "$PS" ]] || { echo "SKIP: powershell.exe not reachable"; exit 2; }
win="$(timeout 20 "$PS" -NoProfile -Command '[DateTimeOffset]::UtcNow.ToUnixTimeSeconds()' 2>/dev/null | tr -d '\r')"
[[ "$win" =~ ^[0-9]+$ ]] || { echo "SKIP: could not read Windows clock (no interop in this context?)"; exit 2; }
now="$(date -u +%s)"; skew=$(( now > win ? now - win : win - now ))
echo "clock skew: ${skew}s"
(( skew <= ${1:-60} ))
VEDHA_EOF
chmod 755 /srv/vedha/ops/bin/vedha-wait-docker /srv/vedha/ops/bin/vedha-clock-check
```

*1.9i — Release installer and switcher (blue/green):*
```bash
cat > /srv/vedha/ops/bin/vedha-install-release <<'VEDHA_EOF'
#!/usr/bin/env bash
# vedha-install-release <tag> <expected-commit> [python-version] [extras]
# Side-by-side install: source/hermes-agent-<tag> + runtime/hermes-<tag>. Does NOT switch 'current'.
set -euo pipefail
. /srv/vedha/vedha-env.sh
TAG="${1:?tag}"; COMMIT="${2:?expected commit}"; PYVER="${3:-3.11}"; EXTRAS="${4:-all}"
SRC="$VEDHA_ROOT/source/hermes-agent-$TAG"; RT="$VEDHA_ROOT/runtime/hermes-$TAG"
if [[ -e "$SRC" || -e "$RT" ]]; then echo "ERROR: $SRC or $RT already exists. Remove deliberately or choose another tag." >&2; exit 1; fi
trap 'echo "Install failed — removing partial $SRC $RT" >&2; rm -rf -- "$SRC" "$RT"' ERR
git clone --quiet --branch "$TAG" --depth 1 https://github.com/NousResearch/hermes-agent.git "$SRC"
actual="$(git -C "$SRC" rev-parse HEAD)"
[[ "$actual" == "$COMMIT" ]] || { echo "ERROR: commit mismatch. expected=$COMMIT actual=$actual" >&2; false; }
cd "$SRC"
echo "requires-python: $(grep -E '^[[:space:]]*requires-python' pyproject.toml || echo 'not declared')"
if [[ -f uv.lock ]]; then
  echo "uv.lock present → reproducible install (uv sync --frozen)"
  UV_PROJECT_ENVIRONMENT="$RT" uv sync --frozen --no-dev --python "$PYVER" --extra "$EXTRAS"
else
  echo "WARN: no uv.lock — dependencies resolve at install time; a freeze is recorded below."
  uv venv "$RT" --python "$PYVER"
  VIRTUAL_ENV="$RT" uv pip install -e ".[$EXTRAS]"
fi
uv pip freeze --python "$RT/bin/python" > "$RT/vedha-lock.txt"
printf 'tag=%s\ncommit=%s\npython=%s\nextras=%s\ninstalled_at=%s\n' "$TAG" "$actual" \
  "$("$RT/bin/python" --version 2>&1)" "$EXTRAS" "$(date -u +%FT%TZ)" > "$RT/vedha-release.txt"
"$RT/bin/hermes" --version
trap - ERR
echo "Installed $TAG side-by-side. Activate with: vedha-switch-release $TAG"
VEDHA_EOF

cat > /srv/vedha/ops/bin/vedha-switch-release <<'VEDHA_EOF'
#!/usr/bin/env bash
# vedha-switch-release <tag> — atomically repoint source/current and runtime/current.
set -euo pipefail
. /srv/vedha/vedha-env.sh
TAG="${1:?tag}"
SRC="$VEDHA_ROOT/source/hermes-agent-$TAG"; RT="$VEDHA_ROOT/runtime/hermes-$TAG"
[[ -d "$SRC" && -x "$RT/bin/hermes" ]] || { echo "ERROR: release $TAG not installed" >&2; exit 1; }
prev="$(readlink "$VEDHA_ROOT/runtime/current" 2>/dev/null || echo none)"
ln -sfn "$SRC" "$VEDHA_ROOT/source/current.new" && mv -Tf "$VEDHA_ROOT/source/current.new" "$VEDHA_ROOT/source/current"
ln -sfn "$RT"  "$VEDHA_ROOT/runtime/current.new" && mv -Tf "$VEDHA_ROOT/runtime/current.new" "$VEDHA_ROOT/runtime/current"
echo "runtime/current: $prev → $RT"
echo "$(date -u +%FT%TZ) switch $prev -> $RT" >> "$VEDHA_STATE/release-history.log"
VEDHA_EOF
chmod 755 /srv/vedha/ops/bin/vedha-install-release /srv/vedha/ops/bin/vedha-switch-release
```

*1.9j — Stage-aware pre-flight:*
```bash
cat > /srv/vedha/ops/bin/vedha-preflight <<'VEDHA_EOF'
#!/usr/bin/env bash
# vedha-preflight — fast gate; checks only what exists at the current stage (WARN for not-yet-configured).
set -uo pipefail
. /srv/vedha/vedha-env.sh; . "$VEDHA_OPS/lib/vedha-common.sh"
echo "== Platform =="
[[ "$(ps -p 1 -o comm=)" == "systemd" ]] && pass "systemd is PID 1" || fail "systemd is not PID 1"
loginctl show-user "$USER" -p Linger 2>/dev/null | grep -q 'Linger=yes' && pass "linger enabled" || fail "linger disabled"
docker info >/dev/null 2>&1 && pass "Docker reachable" || fail "Docker not reachable"
docker run --rm --gpus all nvidia/cuda:12.6.0-base-ubuntu24.04 nvidia-smi -L >/dev/null 2>&1 && pass "GPU visible in containers" || fail "GPU passthrough failed"
vedha-clock-check 60 >/dev/null; rc_c=$?
case $rc_c in 0) pass "clock skew ≤ 60s";; 2) warn "clock check skipped (Windows clock unreadable)";; *) warn "clock skew > 60s (see Appendix C)";; esac
use="$(df -P "$VEDHA_ROOT" | awk 'NR==2{gsub("%","",$5);print $5}')"; (( use < 85 )) && pass "disk ${use}% used" || warn "disk ${use}% used"
echo "== Hermes =="
[[ -f "$HERMES_HOME/.vedha-home" ]] && pass "HERMES_HOME marker" || fail "HERMES_HOME marker missing"
if [[ -x "$VEDHA_RUNTIME/bin/hermes" ]]; then
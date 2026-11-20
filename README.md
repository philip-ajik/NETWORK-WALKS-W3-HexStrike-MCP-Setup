<div align="center">

# 🩸 Wiring an AI Agent to a Red Team Toolkit: HexStrike MCP + Claude Desktop

**Optional Week 3 lab — installing Claude Desktop and the HexStrike AI MCP server on Kali Linux, and everything that broke on the way**
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Tool-HexStrike%20AI%20MCP-1679A7?style=flat-square" />
  <img src="https://img.shields.io/badge/Role-AI%20Agent%20Tooling%20Setup-C00000?style=flat-square" />
  <img src="https://img.shields.io/badge/Result-MCP%20Server%20Healthy-C8570A?style=flat-square" />
  <img src="https://img.shields.io/badge/Phase-Recon%20%26%20Exploitation%20Tooling-238F89?style=flat-square" />
  <img src="https://img.shields.io/badge/NetworkWalks-B082-C00000?style=flat-square" />
</p>

---

## 📌 Project Overview

This is an **optional but important** Week 3 lab from the Networkwalks Cybersecurity internship: **How to set up the HexStrike MCP Server (Practice Lab)**. The task is to connect **Claude Desktop** to **HexStrike AI**, an MCP (Model Context Protocol) server that exposes a large set of offensive-security tools (recon, exploitation, password, network, forensics, and more) so an AI agent can call them directly, on a **Kali Linux** machine.

I followed the lab's own steps: install Claude Desktop from the `aaddrick/claude-desktop-debian` GitHub repo, clone `0x4m4/hexstrike-ai`, set up its Python virtual environment, and point Claude Desktop's MCP config at the HexStrike bridge script. I ran this on 19 November 2026, start to finish, across about two and a half hours.

Almost none of it worked on the first try. One real problem, running out of space while installing HexStrike's Python dependencies, cascaded into several more confusing errors before I reached a clean install, and the lab's own example configuration pointed at a username that wasn't mine. I used an AI assistant throughout to help read the errors and suggest the next command; I'm documenting what it told me to run and why, alongside what I actually typed.

| 🔎 Item | ✅ Result |
| ------- | --------- |
| Claude Desktop | Unofficial Debian build (`aaddrick/claude-desktop-debian`), installed via `apt` |
| HexStrike AI | Cloned from `0x4m4/hexstrike-ai`, Python virtual environment, Flask API on port 8888 |
| Root cause of most errors | `/tmp` filling up during `pip install`, which left several packages partially/incorrectly installed |
| Config issue | The lab's sample `claude_desktop_config.json` uses `/home/kali/...`; my actual home directory is `/home/philip/...` |
| Final state | HexStrike AI extension shows **Running** in Claude Desktop; a live health check reports the server healthy with 8/8 essential tools and 58/1000 tools installed overall |

---

## 🎯 Objectives

- Install Claude Desktop on Kali Linux from a community-maintained Debian package
- Clone and set up HexStrike AI's Python environment and dependencies
- Point Claude Desktop's MCP configuration at the HexStrike bridge script
- Diagnose and fix a disk-space problem that broke several unrelated-looking imports
- Correct a hardcoded path in the lab's sample configuration
- Confirm the integration actually works by asking Claude to check the server's own health

---

## 🛡️ Why This Matters for Security

MCP servers like HexStrike are becoming a real part of how AI-assisted offensive security work gets done: instead of a human typing every command, the AI agent can call recon, exploitation, and OSINT tools directly, reason about the output, and chain them together. Setting one up safely and correctly, including understanding exactly what gets installed, where it listens, and what it can reach, is itself a skill worth practicing before relying on it in real engagements.

This lab is also a good exercise in something less glamorous: reading a wall of Python tracebacks calmly, finding the one root cause underneath several different-looking symptoms, and fixing environment problems (disk space, package versions, hardcoded paths) that have nothing to do with "hacking" but stop every tool from working regardless.

> ⚠️ **Important:** HexStrike AI's own startup banner warns "This is a development server. Do not use it in a production deployment." This setup is for a personal practice lab only, running on `localhost`, with no tools pointed at any system I don't own or have permission to test.

---

## ⚙️ Lab Configuration

| 🧩 Component | ⚙️ Configuration |
| ------------ | ---------------- |
| 🖥️ Platform | Kali Linux VM (VirtualBox) |
| 🖥️ AI Client | Claude Desktop (`aaddrick/claude-desktop-debian`, unofficial build) |
| 🛠️ MCP Server | HexStrike AI (`0x4m4/hexstrike-ai`), Python virtual environment, Flask API on `127.0.0.1:8888` |
| 🔑 Config file | `~/.config/Claude/claude_desktop_config.json` |
| 🧰 Tools exposed | 1000 possible tools across categories (Essential, Network, Password, Web security, Forensics, OSINT, Binary, Exploitation, Vulnerability scanning, and more); 58 actually installed on this machine |
| 🏆 Scoring | None; success is a healthy, connected MCP server |

---

# 🪜 Lab Procedure

## Step 1. Install Claude Desktop
The lab guide installs the unofficial Debian build of Claude Desktop from `aaddrick/claude-desktop-debian` via a GPG key and apt repository.

![GitHub repo and README](screenshots/hex-01-claude-desktop-github-readme.png)

```bash
curl -fsSL https://pkg.claude-desktop-debian.dev/KEY.gpg | sudo gpg --dearmor -o /usr/share/keyrings/claude-desktop.gpg
echo "deb [signed-by=/usr/share/keyrings/claude-desktop.gpg arch=amd64,arm64] https://pkg.claude-desktop-debian.dev stable main" | sudo tee /etc/apt/sources.list.d/claude-desktop.list
sudo apt update
sudo apt install claude-desktop
```

![Repository added, apt update running](screenshots/hex-02-claude-desktop-apt-repo-added.png)

![Package installing](screenshots/hex-03-claude-desktop-installing.png)

I opened Claude Desktop for the first time and signed in.

![First launch](screenshots/hex-04-claude-desktop-first-launch.png)

![Signed in](screenshots/hex-05-claude-desktop-signed-in.png)

## Step 2. Clone HexStrike AI and set up its environment
```bash
git clone https://github.com/0x4m4/hexstrike-ai.git
cd hexstrike-ai
python3 -m venv hexstrike-env
source hexstrike-env/bin/activate
pip3 install -r requirements.txt
```

![HexStrike AI GitHub repo](screenshots/hex-06-hexstrike-github-repo.png)

![Repo cloned, venv created](screenshots/hex-07-clone-repo-and-venv.png)

## Step 3. The dependency install fails: "No space left on device"
The first `pip install -r requirements.txt` stopped partway through with `ERROR: Could not install packages due to an OSError: [Errno 28] No space left on device`, while a later line adds "This error originates from a subprocess, and is likely not a problem with pip."

![No space left on device](screenshots/hex-08-pip-install-no-space-left-error.png)

I asked an AI assistant to help read the error. It suggested checking actual disk usage first with `df -h` and `df -i`, since "no space left" can also mean the filesystem is out of inodes, not just bytes.

```bash
df -h
df -i
```

The disk itself had plenty of room (about 6.4 GB free, `/dev/sda1` at 72%), but the assistant pointed out that `/tmp` is its own small filesystem (a `tmpfs` capped well under 1 GB in this VM) and that `pip` extracts and builds large packages there by default; a big package like `angr` can fill a small `/tmp` even when the real disk is nowhere near full.

![Checking disk space with df](screenshots/hex-09-checking-disk-space-df.png)

## Step 4. Chasing the disk-space problem
Following the assistant's suggestions, I checked how much space the pip cache and `/tmp` were actually using, and cleared what I could.

```bash
du -sh ~/.cache/pip
du -sh /tmp
```

![Clearing pip cache and checking /tmp](screenshots/hex-10-clearing-pip-cache-and-tmp.png)

![pip cache size check](screenshots/hex-11-pip-cache-size-check.png)

Along the way, a `du -sh /tmp` as root (to see past some permission-denied systemd directories) needed a `quota -s` check the assistant suggested for a fuller picture of my disk limits, which wasn't installed.

```bash
quota -s
```

![quota: command not found](screenshots/hex-12-quota-not-found-prompt.png)

```bash
sudo apt install quota
```

![quota installed](screenshots/hex-13-quota-installed.png)

## Step 5. The real fix: redirect pip's temp directory
The assistant's actual fix, once the small `/tmp` was confirmed as the likely culprit, was to point `pip`'s temporary directory at my home folder instead, which has much more free space, and to install without using the pip cache:

```bash
grep -nE 'angr|pyvex|python' requirements.txt   # confirm what's actually pulling in the big packages
python -m pip check                              # "No broken requirements found" at this point
mkdir -p ~/pip-tmp
TMPDIR="$HOME/pip-tmp" python -m pip install --no-cache-dir -r requirements.txt
```

![Confirming the angr requirement; pip check clean](screenshots/hex-14-grep-angr-version-pip-check-clean.png)

![TMPDIR pip install retry](screenshots/hex-15-tmpdir-pip-install-retry.png)

![Packages downloading successfully](screenshots/hex-16-pip-downloading-packages.png)

This time the install ran all the way through, over a hundred packages including `angr`, `pyvex`, `cffi`, `mitmproxy`, and `pwntools`.

![Install completed successfully](screenshots/hex-17-pip-install-completed.png)

## Step 6. The same root cause, three more symptoms
With the install finally complete, I expected to just run the server. Instead, importing the tools failed three more times in a row, each with a different error. These were not three separate bugs: they were all fallout from packages that had been partially or incorrectly installed during the earlier disk-space failure, and each needed a specific package version reinstalled.

**First: `angr` itself failed to import**, with a long traceback through its dependency chain.

```bash
python -c "import angr, flask, fastmcp; print('HexStrike dependencies verified!')"
```

![angr import error](screenshots/hex-18-angr-import-error.png)

**Second: that traceback bottomed out in `bitstring`**, with `AttributeError: module 'bitstring' has no attribute 'ConstBitStream'`. The assistant explained that `bitstring` 5.x removed the older `ConstBitStream` API that `pyvex` (one of `angr`'s dependencies) still expects, and that the fix was to pin an older, compatible release.

![bitstring AttributeError](screenshots/hex-19-bitstring-attribute-error.png)

```bash
python -m pip install "bitstring>=4.2,<5"
```

![Installing a compatible bitstring version](screenshots/hex-20-installing-bitstring.png)

**Third: a new, different error appeared from `cffi`/`pycparser`**, `AttributeError: property 'filename' of 'CLexer' object has no setter`, after `bitstring` was fixed. The assistant identified this as a separate, unrelated version mismatch: a newer `pycparser` had changed how its lexer handles the `filename` property, which an older `cffi` (a `pyvex` dependency) wasn't expecting.

![bitstring fixed, cffi/pycparser error appears](screenshots/hex-21-bitstring-installed-still-failing.png)

![cffi/pycparser AttributeError](screenshots/hex-22-cffi-pycparser-attribute-error.png)

```bash
python -m pip install "pycparser==2.21"
```

![Reinstalling pycparser 2.21](screenshots/hex-23-reinstalling-pycparser.png)

With both `bitstring` and `pycparser` pinned to compatible versions, the verification line finally printed clean, and `hexstrike_server.py` started up with its full HEXSTRIKE banner.

```bash
cd ~/hexstrike-ai
source hexstrike-env/bin/activate
python3 hexstrike_server.py
```

![Dependencies verified, server starts](screenshots/hex-24-dependencies-verified-server-starts.png)

## Step 7. Configure Claude Desktop to use HexStrike as an MCP server
With the server running, I opened Claude Desktop's settings to add HexStrike as a local MCP server.

![Claude settings, no MCP servers yet](screenshots/hex-25-claude-settings-no-mcp-servers.png)

![Edit Config prompt](screenshots/hex-26-claude-settings-edit-config-prompt.png)

![Claude config folder](screenshots/hex-27-claude-config-folder.png)

The lab guide's sample `claude_desktop_config.json` uses the path `/home/kali/hexstrike-ai/...`, matching the lab's own demo VM where the user is literally called `kali`. My own Kali VM's user is `philip`, so that path doesn't exist on my machine.

```json
{
  "mcpServers": {
    "hexstrike-ai": {
      "command": "/home/kali/hexstrike-ai/hexstrike-env/bin/python",
      "args": [
        "/home/kali/hexstrike-ai/hexstrike_mcp.py",
        "--server",
        "http://localhost:8888"
      ]
    }
  }
}
```

![Config file with the lab's default /home/kali path](screenshots/hex-28-config-json-default-kali-path.png)

![Still showing the kali path before editing](screenshots/hex-29-config-json-still-kali-path.png)

I edited the paths to match my actual home directory.

![Config edited to /home/philip](screenshots/hex-30-config-json-edited-to-philip-path.png)

![Claude settings still showing no servers added](screenshots/hex-31-claude-settings-no-servers-added.png)

![Extensions panel](screenshots/hex-32-claude-extensions-panel.png)

## Step 8. Confirm everything actually connects
Before assuming the path fix alone would work, I checked that the bridge script genuinely existed where the config pointed, and that the config file itself now held the corrected paths.

```bash
ls -l /home/philip/hexstrike-ai/hexstrike_mcp.py
grep -nE '"command"|hexstrike_mcp.py|localhost:8888' ~/.config/Claude/claude_desktop_config.json
```

![Confirming hexstrike_mcp.py exists](screenshots/hex-33-confirming-hexstrike-mcp-py-exists.png)

![grep confirming the corrected config paths](screenshots/hex-34-grep-confirming-config-paths.png)

I then checked whether the Flask API was actually listening on port 8888, and what it returned.

```bash
ss -tln | grep 8888
curl -i http://localhost:8888
```

![Checking port 8888](screenshots/hex-35-checking-port-8888-listening.png)

`curl` returned `HTTP/1.1 404 NOT FOUND`, which looked alarming at first glance, but the assistant pointed out this actually confirms the Flask server **is** running and answering; it simply has no route defined at the bare root path `/`.

![curl to localhost:8888 returns 404, meaning the server is alive](screenshots/hex-36-curl-localhost-8888-404.png)

I double-checked the `hexstrike_server.py` process was genuinely running, then ran the MCP bridge script itself in the foreground to watch it connect live.

```bash
ps -fp <pid>
cd ~/hexstrike-ai
./hexstrike-env/bin/python hexstrike_mcp.py
```

![Confirming the server process is running](screenshots/hex-37-checking-server-process-running.png)

![Running hexstrike_mcp.py manually, connecting to the API](screenshots/hex-38-running-hexstrike-mcp-manually.png)

It connected successfully: "Successfully connected to HexStrike AI API server," reported a healthy status, and sat ready to serve requests.

![Full corrected config.json](screenshots/hex-39-full-config-json-contents.png)

![Claude's log directory](screenshots/hex-40-claude-logs-directory.png)

Claude Desktop's own logs then showed the MCP server registering successfully.

```bash
grep -i mcp ~/.config/Claude/logs/main.log | tail
```

![MCP server registered in Claude's logs](screenshots/hex-41-mcp-server-registered-in-logs.png)

![Confirming the registration line](screenshots/hex-42-grep-mcp-registered-confirmation.png)

## Step 9. Final checks and confirmation
I ran a few more sanity checks: confirming Claude Desktop's own processes, listing its installed binary, and running its built-in `--doctor` diagnostic command.

```bash
ps aux | grep -i claude
ls -la /usr/lib/claude-desktop-unofficial/
claude-desktop-unofficial --doctor
```

![Claude Desktop processes](screenshots/hex-43-ps-aux-claude-desktop-processes.png)

![Install directory listing](screenshots/hex-44-claude-desktop-install-dir-listing.png)

![--doctor diagnostics: install, display server, sandbox, MCP config all PASS](screenshots/hex-45-claude-desktop-doctor-diagnostics.png)

![Diagnostics continued: disk space, cowork mode (KVM not available, non-blocking)](screenshots/hex-46-doctor-diagnostics-continued.png)

I restarted Claude Desktop to pick up the corrected config.

![Restarting Claude Desktop](screenshots/hex-47-restart-claude-desktop.png)

The HexStrike AI extension now shows as **Running**, with the corrected command and arguments visible in its own settings panel.

![hexstrike-ai extension Running](screenshots/hex-48-hexstrike-ai-extension-running.png)

## Step 10. Ask Claude to check the server's own health
As a final, meaningful test rather than just a green status dot, I asked Claude directly to check the health of the HexStrike MCP server it was now connected to.

![Asking Claude for a health check](screenshots/hex-49-asking-claude-for-health-check.png)

Claude used the live MCP connection to report back: the server (v6.0.0) was healthy, had been running about 55 minutes, all 8 essential tools were available, and gave live resource and cache figures.

![Health check: resources and cache](screenshots/hex-50-health-check-resources-report.png)

It then broke down tool availability by category (58 of 1000 possible tools actually installed) and flagged a low overall command success rate as something worth investigating.

![Tool availability by category](screenshots/hex-51-health-check-tool-categories.png)

![Things to look at: low success rate, missing cloud/web tools](screenshots/hex-52-health-check-things-to-look-at.png)

**The HexStrike MCP server is confirmed connected, healthy, and genuinely usable by Claude Desktop.**

---

# 📊 Results

| # | 🎯 Step | ✅ Result |
| - | ------- | --------- |
| 1 | Install Claude Desktop | Installed and signed in (unofficial Debian build) |
| 2 | Install HexStrike AI | Cloned, venv created, dependencies eventually installed cleanly |
| 3 | Fix disk-space failure | `/tmp` was the real bottleneck, not the disk; fixed with `TMPDIR` redirected to home and `--no-cache-dir` |
| 4 | Fix knock-on dependency errors | Pinned `bitstring>=4.2,<5` and `pycparser==2.21` |
| 5 | Fix Claude Desktop config | Corrected `/home/kali/...` to `/home/philip/...` in `claude_desktop_config.json` |
| 6 | Confirm connection | `hexstrike-ai` extension shows **Running**; Claude's own health check confirms the server is up with 8/8 essential tools |

*Footnote: this is a setup lab with no scored task or flag; success is the HexStrike MCP server being connected, healthy, and genuinely callable by Claude Desktop, which the final health check confirms.*

### Timeline (19 November 2026)
| ⏱️ Time | 🔄 What happened |
| ------- | ---------------- |
| 05:00–05:11 | Claude Desktop installed and signed in |
| 05:11–05:26 | HexStrike cloned; first `pip install` fails with "No space left on device" |
| 05:26–06:17 | Disk-space diagnosis (`df`, `du`, `quota`); fixed with `TMPDIR` redirect; full install completes |
| 06:20–06:28 | Three chained import errors (`angr` → `bitstring` → `cffi`/`pycparser`), each fixed in turn |
| 06:31 | `hexstrike_server.py` starts cleanly with the full HEXSTRIKE banner |
| 06:34–07:00 | Claude Desktop MCP config edited from `/home/kali/...` to `/home/philip/...` |
| 07:00–07:20 | Verifying the fix: process checks, port checks, manual bridge run, `--doctor` diagnostics |
| 07:24 | `hexstrike-ai` extension shows **Running** |
| 07:26 | Claude's own health check confirms the server is healthy and usable |

---

# 🔭 Observations Beyond the Task Questions

1. **"No space left on device" did not mean the disk was full.** The real disk had 6.4 GB free; the actual bottleneck was a size-capped `/tmp` tmpfs that `pip` uses for temporary build files. This is worth remembering for any VM with a small `/tmp`.
2. **`pip check` reported "No broken requirements found" even while imports kept failing.** `pip check` only verifies that installed packages' declared version requirements are mutually satisfiable; it does not catch packages that were only partially written to disk or whose compiled extensions silently mismatch at import time.
3. **A `404 Not Found` from `curl` can mean success, not failure.** It confirmed the Flask server was alive and answering HTTP requests; it just had no route at `/`. Reading what an error code actually means for the specific server mattered more than reacting to "404" as a generic failure.
4. **The lab's own health check flagged real gaps worth knowing about.** Only 58 of 1000 possible tools are actually installed on this machine, cloud-security tooling (prowler, trivy, kube-hunter, scout-suite) is missing entirely, and Metasploit showing as unavailable while `msfconsole`/`msfvenom` show as present suggests it's just installed under a different binary name, not actually missing.
5. **HexStrike's own startup banner is explicit that this is a development server, not for production use.** That matches the lab's framing: a practice environment on `localhost`, not a deployed tool.

---

# 🐞 Challenges Encountered

* **Issue: `pip install -r requirements.txt` failed with "No space left on device", which then broke three unrelated-looking imports.** All of this traced back to one root cause: `/tmp` filling up mid-install, which left `angr` and several of its dependencies (`bitstring`, `pycparser`/`cffi`) partially or incorrectly installed. Fixing the actual disk problem first (`TMPDIR` pointed at a larger filesystem, `--no-cache-dir`) and then pinning the two packages whose APIs had drifted (`bitstring>=4.2,<5`, `pycparser==2.21`) resolved all three symptoms in turn.
  **What I'll do differently:** Check `df -h` and the size of `/tmp` specifically before running any large `pip install`, rather than after the first failure.

* **Issue: Claude Desktop's MCP config pointed at `/home/kali/...`, a path that doesn't exist on my machine.** The lab's sample config matches the instructor's own demo VM, where the Linux username happens to be `kali`. My VM's username is `philip`.
  **Resolution:** Edited `claude_desktop_config.json` to use my actual home directory, then verified the bridge script's real path with `ls -l` and the config's contents with `grep` before trusting it would work.
  **What I'll do differently:** Treat any sample config with a hardcoded path as a template, not a literal value, and check `whoami`/`$HOME` before copying it in.

* **No blockers once both issues were fixed.** The server started cleanly, the extension showed Running, and Claude's own health check confirmed the connection worked end to end.

---

# 💡 Key Learnings

1. One root cause can produce several different-looking error messages. Before chasing each traceback separately, it's worth asking whether they share a common origin.
2. "No space left on device" can mean a specific, small filesystem like `/tmp`, not the disk as a whole; `df -h` on the specific path matters more than a general sense of free space.
3. A clean `pip check` is not proof that every package actually imports correctly; partial installs from an earlier failure can still be sitting on disk.
4. Reading what an HTTP status code actually means for the server in front of you (a 404 at `/` rather than a connection refusal) avoids chasing a non-problem.
5. Any lab's sample configuration file is written for the instructor's own environment. Usernames, paths, and hostnames need to be checked against your own machine before use.
6. A green "Running" status is a good sign, but asking the AI agent to actually use the connected tool (here, a live health check) is a more meaningful confirmation than a status indicator alone.

---

# 🔐 Security & Ethical Use

This lab is a personal setup/practice exercise. HexStrike AI's server ran only on `localhost` for this session, and no tool it exposes was pointed at any system I don't own or don't have permission to test. HexStrike's own startup output explicitly warns it is a development server and should never be exposed in a production deployment. The same caution applies to any of the offensive-security tools it can call: they should only ever be used within an authorized, permitted scope.

---

# 🔗 Tools & Resources

- **Claude Desktop (unofficial Debian build):** https://github.com/aaddrick/claude-desktop-debian
- **HexStrike AI:** https://github.com/0x4m4/hexstrike-ai
- **NetworkWalks:** https://networkwalks.com

---

# 👤 Author
**Philip Ajik**
Cybersecurity Student (Batch B082) — Networkwalks Academy
LinkedIn: https://www.linkedin.com/in/philip-ajik-a22944384

---

## 📌 Project Information
**Program Name:** Cybersecurity at Networkwalks | **Task:** Week 3 Optional Lab | **Project:** HexStrike MCP Server Setup with Claude Desktop | **Repository:** GitHub

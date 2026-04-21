# Real Isolation on macOS: A General-Purpose Sandbox Model

> **What this is:** A controlled environment with limited blast radius.
> **What this isn't:** A formally secure system.
>
> The goal is to make safe behavior the path of least resistance.
>
> **One operational rule:** Run code only where you're comfortable deleting everything it can touch.

---------====--------====---------

## The Isolation Stack

| Layer | What it actually protects | What it does NOT protect | How you use it |
|-------|--------------------------|--------------------------|----------------|
| **Sandbox dir** | Organization, recoverability | System, network, other files | `cd ~/sandbox` |
| **venv** | Dependency pollution | System, network, other files | `python3 -m venv venv` |
| **Separate user (SSH)** | Your personal files, keychain, browser data | System-wide installs (`brew`), network, kernel | `ssh sandbox@localhost` |
| **VM (UTM/Parallels)** | System changes, filesystem, processes | Host via hypervisor escape or integration features | Boot VM, do everything inside |
| **Air-gapped VM** | System + network | Physical access | VM with networking disabled |

**The rule:** Each layer you add costs friction. Don't climb higher than the risk justifies.

---------====--------====---------

## Core Principles

### 1. Create the boundary

```bash
mkdir -p ~/sandbox
cd ~/sandbox
```

**Rule:** This is where experiments live. Nowhere else.

> This directory is the real sandbox - not Docker, not a VM. The boundary is the discipline.

### 2. Stateless by default

**Don't reuse environments.** The path of least resistance should be *fresh*, not *maintained*.

```bash
cd ~/sandbox
EXP_DIR=exp-$(date +%s)
mkdir "$EXP_DIR"
cd "$EXP_DIR"
python3 -m venv venv
source venv/bin/activate
```

Install what you need, run, then delete:

```bash
# when done
deactivate
cd ~/sandbox
rm -rf "$EXP_DIR"
```

**Key habits:**
- If you wouldn't recreate it from scratch, you shouldn't trust it
- Never leave credentials inside (no `.env` files, no tokens in the venv)
- Most experiments should end with deletion. Persistence is the exception, not the default.

### 3. Working directory discipline

Before running anything:

```bash
pwd
```

If it's not inside `~/sandbox`, stop.

**Optional helper:**

```bash
alias sandbox="cd ~/sandbox"
```

### 4. The "new experiment" ritual

```bash
cd ~/sandbox
EXP_DIR=exp-$(date +%s)
mkdir "$EXP_DIR"
cd "$EXP_DIR"
python3 -m venv venv
source venv/bin/activate
```

Now install deps, run, delete when done.

---------====--------====---------

## Setup Guide

### One-time setup

```bash
# 1. Create sandbox user
# System Settings → Users & Groups → Add user: sandbox (Standard)

# 2. Enable SSH
sudo systemsetup -setremotelogin on

# 3. Harden SSH configuration
sudo nano /etc/ssh/sshd_config

# Add or ensure these lines:
AllowUsers sandbox
PasswordAuthentication no
PermitRootLogin no
UsePAM no

# 4. Set up SSH key authentication for sandbox user
# Generate keys on your main user and copy the public key to the sandbox user (~/.ssh/authorized_keys)
# Then reload SSH:
sudo launchctl stop com.openssh.sshd
sudo launchctl start com.openssh.sshd

# 5. Add alias to main user
echo 'alias box="ssh sandbox@localhost"' >> ~/.zshrc
```

### Inside the SSH session (sandbox user)

```bash
# Auto-enter sandbox on login
echo 'cd ~/sandbox' >> ~/.zshrc

# Panic reset - actually cleans everything including hidden files
# With safety guardrail: only works when inside ~/sandbox or its subdirectories
cat << 'EOF' >> ~/.zshrc
function nuke() {
  if [[ "$PWD" != "$HOME/sandbox"* ]]; then
    echo "❌ Not in sandbox. Aborting."
    return 1
  fi
  find . -mindepth 1 -delete
}
EOF

# Guardrail function - runs command only if inside sandbox
cat << 'EOF' >> ~/.zshrc
function safe-run() {
  if [[ "$PWD" != $HOME/sandbox* ]]; then
    echo "❌ Not in sandbox. Aborting."
    return 1
  fi
  "$@"
}
EOF
```

### Daily workflow

```bash
# Normal experimentation
box                    # SSH into sandbox user
mkdir exp-$(date +%s) && cd $_
python3 -m venv venv && source venv/bin/activate
# ... do work ...
deactivate && cd .. && rm -rf exp-*

# Panic
nuke                   # back to clean slate (only works inside ~/sandbox)
```

---------====--------====---------

## Escalation Path

| Risk level | Action |
|------------|--------|
| Trusted tools, known code | Main user, sandbox dir |
| Unknown repo, agent/tooling | `box` → SSH into sandbox user |
| **Anything that modifies the system or runs untrusted code with side effects** | VM |
| Maximum paranoia | Air-gapped VM |

**The cleanest heuristic:**

> If it can modify anything outside the sandbox → escalate.

If it touches `brew`, system-level `pip install` (outside a venv), global `npm install -g`, modifies dotfiles (`~/.zshrc`, `~/.bashrc`, etc.), or changes anything outside the sandbox → VM.

**Build/install steps are execution:** `make`, `setup.py`, `postinstall` scripts, and Makefiles can execute arbitrary code - treat them as execution, not just installation.

---------====--------====---------

## Network Control

macOS networking is system-wide, not per-user.

| Approach | How | Tradeoff |
|----------|-----|----------|
| **Manual toggle** | `networksetup -setairportpower en0 off` | Simple, but global |
| **Discipline** | "This account = no network unless needed" | Free, requires habit |
| **Per-process firewall** | LuLu or Little Snitch | Best practical control |
| **VM without network** | Disable networking in VM settings | Strongest |

**Recommendation:** Pair SSH sandbox + LuLu. First time a process requests network access, **deny by default and observe what breaks.** If something breaks, selectively allow only what you understand - never blanket allow a process you don't recognize. Some tools will fail noisily without network access - this is expected. Only restore access if you understand why it's needed. This turns the firewall into a learning tool instead of a "click allow" reflex.

**Process visibility matters.** Watch what tries to connect out. Unknown behavior should be observable, not just contained.

---------====--------====---------

## Homebrew: The Global Problem

`/opt/homebrew` is shared across all users. A separate user doesn't fix this.

| Where you run `brew install` | Effect |
|------------------------------|--------|
| Main user | Global system change |
| `sandbox` via SSH | **Still global system change** |
| Inside VM | **Contained to VM only** |

**The deeper risk:** Some Homebrew formulas install background services (`brew services`), modify PATH-linked binaries, or pull in unexpected dependencies. Treat them as system-level changes, not just "packages."

**The same rule applies to:** system-level `pip install` (outside a venv) and global `npm install -g`.

**So the rule is:**
- Use `brew` only for tools you're willing to treat as part of your base system
- If you're experimenting with unknown packages → do it inside a VM
- If you want *zero* system contamination → VM is the only answer

---------====--------====---------

## VM Setup & Hardening

When you care about isolation, disable VM integrations:

In UTM / Parallels Desktop, turn off:
- Shared folders
- Clipboard sharing
- Drag & drop
- Time sync (if configurable)

**Also:** Disable shared iCloud / Apple ID login inside the VM. Otherwise you reintroduce personal data into the VM boundary, completely defeating the "separate failure domain" idea.

Otherwise you quietly punch holes through your isolation.

**Docker positioning:**

> Docker ≠ isolation tool. Docker = **workflow + reproducibility layer on top of a VM.**

On macOS, Docker Desktop runs containers inside a managed Linux VM. It has tighter workflow integration and built-in image reproducibility, but don't treat it as a security boundary.

---------====--------====---------

## Pre-run Checklist

Before you run anything, answer:

| Question | What to check |
|----------|---------------|
| **Filesystem** | Is it confined to `~/sandbox` with no home directory access? |
| **Network** | Does it need internet? If not, why is it on? |
| **Execution** | Can it spawn subprocesses or call shell? Am I implicitly allowing that via a framework? |
| **State** | Can I delete this instantly if it breaks? |
| **Provenance** | Am I pinning versions, or pulling latest? Could anything auto-update? |

> If any answer is unclear, tighten the boundary or don't run it yet.

---------====--------====---------

## What You Should Skip

| Thing | Why skip |
|-------|----------|
| Visible sandbox markers | UX clutter, not a security factor |
| `sudo -u sandbox` | Leaky, inherits environment weirdly, feels like isolation without delivering it |
| Docker as *primary* isolation | It's a workflow layer, not a security boundary |
| Chasing `sandbox-exec` replacements | Dead ecosystem, better alternatives exist |

---------====--------====---------

## Final Mental Model

Don't ask: *"Is this safe?"*

Ask: *"Where do I want this to be allowed to fail?"*

| Failure domain | Recovery action |
|----------------|-----------------|
| Sandbox dir | `nuke` |
| User account | Delete `/Users/sandbox` |
| VM | Delete VM file |

**Control the boundary, control the risk.**

---------====--------====---------

## What Success Looks Like

After following this guide, you should:

1. **Never run experimental code outside `~/sandbox`**
2. **Default to fresh environments** (create, run, delete)
3. **Know when to switch to a separate user** (unknown repos, agents, tools)
4. **Recognize when something needs stricter containment** (and reach for VM)

If you do those four things, the model is working.

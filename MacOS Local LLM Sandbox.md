# MacOS Local LLM Sandbox: macOS Setup Guide

> **What this is:** A controlled environment with limited blast radius.
> **What this isn't:** A formally secure system.
>
> The goal is to make safe behavior the path of least resistance.

---

## 0. What this entails

You'll build three things:

1. A **sandbox directory** — the only place LLM code runs
2. A **repeatable environment pattern** — fresh, disposable, stateless
3. **Optional isolation layers** — for when risk increases

This takes about 10 minutes. The habit takes longer, but the setup makes it easy.

---

## 1. Create the boundary

This is the foundation. Everything else depends on it.

```bash
mkdir -p ~/llm-sandbox
cd ~/llm-sandbox
```

**Rule:** This is where all experiments live. Nowhere else. No shortcuts like "I'll just run this from Downloads."

> This directory is the real sandbox—not Docker, not a VM. The boundary is the discipline.

---

## 2. Set up a clean Python workflow

**Don't reuse environments.** The path of least resistance should be *fresh*, not *maintained*.

Create a template you can copy, or just build fresh each time:

```bash
# Pattern: always create fresh
cd ~/llm-sandbox
EXP_DIR=exp-$(date +%s)
mkdir "$EXP_DIR"
cd "$EXP_DIR"
python3 -m venv venv
source venv/bin/activate
```

Install what you need, run your experiment, then delete it:

```bash
# when done
deactivate
cd ~/llm-sandbox
rm -rf "$EXP_DIR"
```

**Key habits:**
- If you wouldn't recreate it from scratch, you shouldn't trust it
- Never leave credentials inside (no `.env` files, no tokens in the venv)
- Most experiments should end with deletion. Persistence should be the exception, not the default.

---

## 3. Install a native runtime

**Ollama**

- Minimal setup
- Clean separation (models live in `~/.ollama`, not scattered)
- API-based usage (easy to call from scripts)
- 
```bash
# Install via Homebrew
brew install ollama

# Or download from https://ollama.com
```

**Note:** `brew install` introduces new code onto your system. Treat upgrades (`brew upgrade`) as intentional actions, not something to run casually.

**llama.cpp**

- More control
- Fully local inference without a daemon

```bash
brew install llama.cpp
```

Pick **one** (or research alternatives)

---

## 4. Enforce working directory discipline

Before running anything:

```bash
pwd
```

If it's not inside `~/llm-sandbox`, stop.

**Optional helper:** Add an alias to your `~/.zshrc`:

```bash
alias llmcd="cd ~/llm-sandbox"
```

Then `llmcd` always takes you home.

---

## 5. The "new experiment" ritual

This operationalizes *stateless by default*. Make it automatic:

```bash
cd ~/llm-sandbox
EXP_DIR=exp-$(date +%s)
mkdir "$EXP_DIR"
cd "$EXP_DIR"
python3 -m venv venv
source venv/bin/activate
```

Now install deps, run, delete when done.

This should feel as natural as `cd ~/Downloads` used to feel. That's the point.

---

## 6. Elevated tier: separate macOS user

For when you're running unknown repos or testing agents/tools.

**Setup:**

1. System Settings → Users & Groups
2. Add user: `llm-test` (Standard user, not admin)
3. Fast User Switch to this account when needed

This adds a small amount of friction (switching users), which is intentional—it's a signal that you're crossing a trust boundary.

**Why this matters:** This user has no access to your main files unless you explicitly give it. Photos, Contacts, Documents—none of it is visible. That's the real value.

**When to use:**
- Cloning a repo you haven't audited
- Running anything with tool/agent capabilities
- Testing MCP servers or LangChain chains

---

## 7. Optional: lightweight network control

Simple options:

- Turn off Wi-Fi temporarily
- Or just ask: "Does this actually need internet?"

Don't overbuild this. The habit is the control:

> If it doesn't need internet, consider running it without internet.

---

## 8. Paranoid mode: sandbox-exec

Use this for untrusted scripts or agent frameworks. Expect breakage—that's a signal, not a problem.

**Working example:**

```bash
sandbox-exec -p '
(version 1)
(deny default)
(allow file-read* (subpath "/System"))
(allow file-read* (subpath "/usr"))
(allow file-read* (subpath "/Users/yourname/llm-sandbox"))
(allow file-write* (subpath "/Users/yourname/llm-sandbox"))
(deny network*)
' python your_script.py
```

**Important:** Replace `yourname` with your actual username. Don't try to generalize profiles—keep it copy-paste usable.

If something fails under these constraints, that usually means it was trying to access something outside your defined boundary. That's diagnostic feedback, not confusion.

**When to use:**
- Running code from an untrusted source
- Agent frameworks with tool execution
- Anything that touches shell or filesystem broadly

---

## 9. Docker (optional, positioned correctly)

**Use when:**
- Workflows get complex (pipelines, eval harnesses, fine-tuning)
- You need reproducibility across machines
- Debugging environment-specific bugs ("works on my machine")
- If you find yourself debugging dependency issues for more than ~15–20 minutes, that's often a signal Docker might be worth it

**Skip when:**
- Running inference interactively
- Iterating quickly on one machine

> Docker controls environment consistency. It does not meaningfully reduce blast radius by default. Not required for this setup.

---

## 10. Close with the checklist

Setup is done. Safety is behavioral.

Before you run anything, answer:

| Question | What to check |
|----------|---------------|
| **Filesystem** | Is it confined to `~/llm-sandbox` with no home directory access? |
| **Network** | Does it need internet? If not, why is it on? |
| **Execution** | Can it spawn subprocesses or call shell? Am I implicitly allowing that via a framework (e.g. tool/agent abstractions)? |
| **State** | Can I delete this instantly if it breaks? |
| **Provenance** | Am I pinning versions, or pulling latest? Could anything auto-update? |

> If any answer is unclear, tighten the boundary or don't run it.

---

## What success looks like

After following this guide, you should:

1. **Never run LLM code outside `~/llm-sandbox`**
2. **Default to fresh environments** (create, run, delete)
3. **Know when to switch to a separate user** (unknown repos, agents, tools)
4. **Recognize when something needs stricter containment** (and reach for `sandbox-exec`)

If you do those four things, the framework is working.

---

**One-line takeaway:**

> *The blast radius question, not the "is this safe" question.*

**And if you can't clearly describe the boundary, you don't have one.**

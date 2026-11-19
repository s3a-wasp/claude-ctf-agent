# Claude Code as a CTF & Pentest Assistant (Kali VM)

Setting up [Claude Code](https://www.claude.com/product/claude-code) as an autonomous-ish
AI assistant for Capture The Flag challenges and authorized penetration testing, running
inside a Kali Linux VM on VMware — with vetted community security skills and tooling.

---

## TL;DR (short summary)

I wanted an AI agent that could help me work through CTF challenges and pentest labs,
similar to paid "deep agent" products but cheaper and under my control. I used **Claude Code**
(the Anthropic terminal agent) inside a disposable **Kali VM**.

What I did, in order:

1. Installed Claude Code in the Kali VM with the official installer.
2. Put it on my `PATH` and logged in (had to bump the VM's RAM to fix a browser freeze).
3. **Audited** two community security plugin/skill repos before trusting them.
4. Installed a set of security **skills** (wordlists, payloads, patterns) by copying folders.
5. Installed a tool-runner **plugin** (183+ pentest/OSINT tools) from its marketplace.
6. Added a `CLAUDE.md` that keeps the agent methodical and **in-scope**.
7. Verified the whole stack by solving a self-contained practice challenge.

Total model cost: a Claude Pro subscription ($20/mo). The free Claude plan does **not**
include Claude Code.

> **Authorized use only.** Everything here is for CTFs, your own labs, and systems you have
> explicit written permission to test. Many CTFs also restrict AI/automation — check the rules
> before entering an agent.

---

## Prerequisites

- A Kali Linux VM (I used VMware). **Give it at least 4 GB RAM** — see the troubleshooting note.
- Internet access from inside the VM (NAT is fine).
- A paid Claude account (Pro, $20/mo, is the minimum that includes Claude Code).
- Take a **VM snapshot** before you start so you can roll back.

---

## Step 1 — Install Claude Code

Run the official installer inside the Kali VM as your normal user (not root):

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

It installs to your home directory (typically `~/.local/bin/claude`).

## Step 2 — Put it on your PATH

The installer's location may not be on your `PATH` yet. Add it for the current shell and
confirm the binary works:

```bash
export PATH="$HOME/.local/bin:$PATH"
claude --version        # should print e.g. 2.1.327 (Claude Code)
```

Make it permanent so every new terminal finds `claude`:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
```

> `>>` appends to the file. Don't use a single `>`, which overwrites it.

## Step 3 — Log in

```bash
mkdir -p ~/ctf && cd ~/ctf
claude
```

On first launch you pick a theme, then log in with your **Claude account**: it opens a browser
to authorize, then hands a code back to the terminal. Type `/status` once you're at the prompt
to confirm you're authenticated.

### Troubleshooting: the login froze the whole VM

The browser login ran the VM out of memory and locked it up. **Fix: increase the VM's RAM**
(VMware → VM → Settings → Memory → 4 GB+), then retry. If a headless/SSH VM can't complete
the browser handoff, use the manual flow: copy the authorization **code** from the browser's
address bar (the value after `code=`) and paste it into the terminal with `Ctrl+Shift+V`.

## Step 4 — Have Claude audit the community repos before trusting them

A Claude Code plugin runs with the same access as Claude Code itself — file system, shell,
and any bundled MCP servers — so a malicious security plugin is a real supply-chain risk.
Before installing anything, I had Claude clone both repos into a scratch directory and read
through everything: every `SKILL.md`, any hooks, and all scripts. It then gave me a plain-English
verdict on each.

Here's what the audit looks for, and why each matters:

- **Unexpected network calls** (`requests`, `urllib`, `urlopen`, raw sockets, `curl`/`wget`
  inside scripts). *Why:* a security tool has no reason to phone home. Outbound connections that
  aren't part of the stated function are how a malicious plugin would exfiltrate your data or
  pull down a second-stage payload.
- **`eval` / `exec`, `base64`-decoded blobs, obfuscated strings.** *Why:* these are the classic
  way to hide executable code so a human skim-reader won't notice what it actually does.
- **Hardcoded tokens, webhooks, or API keys.** *Why:* they signal the code is wired to report
  to someone else's server.
- **How commands are executed.** *Why:* `subprocess.run([...])` with an argument list is safe;
  `shell=True` on an unsanitized string is a command-injection surface that could let crafted
  input run arbitrary commands on your machine.

> Why bother: a few hundred GitHub stars is not a security review, plugin-directory sites inflate
> numbers, and offensive-security tooling is exactly where someone would hide something nasty.
> Reading the code (or having Claude read it) is the only real check.

Claude's findings for the two I used:
- **Skills repo** — pure data (wordlists, payloads, web-shell samples). No scripts, no hooks,
  no network. (Note: it contains real web shells and an `nc.exe` sample — your host AV may
  flag the folder. They're inert unless deliberately deployed.)
- **Tool-runner plugin** — a command wrapper. Only network call was a connectivity check to
  Cloudflare. Commands run via `subprocess.run(argv, ...)` with no `shell=True`. No exfil,
  no eval, no telemetry.

> Pin to the commit you reviewed rather than tracking `main`, and re-audit after updates.

---

## Skills vs. plugins (and why I picked these two)

Claude Code can be extended in two ways, and the two repos here are one of each — which is
also why they install differently.

**A skill** is the simplest unit: a folder with a `SKILL.md` of instructions (plus optional
reference files) that Claude loads when your request matches it. No build step, no runtime —
it's on-demand knowledge. You install one by dropping the folder into `~/.claude/skills/`
(personal) or a project's `.claude/skills/`. Think of it as teaching Claude *a method or giving
it reference material*.

**A plugin** is a package. It's distributed through a *marketplace* (a git repo with a
`marketplace.json` catalog), installed with `/plugin install`, and versioned so it can be
updated. The difference that matters: a plugin can bundle several things at once — one or more
skills, plus subagents, hooks, slash commands, and MCP servers (tool integrations). It's the
wrapper you use when you need to ship actual **executable code** and wire it into Claude.

> In short: a skill is one piece of expertise; a plugin is a managed package that can contain
> skills *plus* executable tooling. Every plugin can include skills, but not every skill is a plugin.

Why each repo:

- **Eyadkelleh/awesome-skills-security** (installed as **skills**) — the most-cited CTF-oriented
  security repo, and almost entirely *inert data*: wordlists, fuzzing payloads, credential lists,
  detection patterns, web-shell samples. No executable code, which is why it ships as skills you
  copy in and why it was low-risk and quick to audit. It gives Claude ready reference material to
  pull from mid-challenge — passwords to try, SQLi payloads, and so on.
- **AKCodez/hackingtool-plugin** (installed as a **plugin**) — this one has to *do* something,
  not just know something. It wraps 183+ tools (nmap, nuclei, sqlmap, subfinder…) and ships
  scripts that run them and auto-pick a backend (native, WSL, or Docker). That executable
  machinery needs the plugin structure — a marketplace to install from and the
  `${CLAUDE_PLUGIN_ROOT}` path so its scripts find each other.

The pairing is deliberate: the skills give Claude the *knowledge and payloads*; the plugin gives
it the *ability to run the actual tools*.

## Step 5 — Install the security skills

Skills are just folders with a `SKILL.md`. This repo isn't a "marketplace," so you install it
by copying its skill folders into your personal skills directory:

```bash
mkdir -p ~/.claude/skills
git clone --depth 1 https://github.com/Eyadkelleh/awesome-skills-security /tmp/eyad-skills
cp -r /tmp/eyad-skills/skills/* ~/.claude/skills/
ls ~/.claude/skills
```

You should see: `security-fuzzing`, `security-passwords`, `security-usernames`,
`security-patterns`, `security-payloads`, `security-webshells`, `llm-testing`.

Skills load at startup — restart `claude` (`/exit`, then `claude`) to pick them up. Confirm by
asking the agent: *"What skills do you have available? List them."*

## Step 6 — Install the tool-runner plugin

This one **is** a marketplace, so add it and install the plugin:

```text
# inside a claude session:
/plugin marketplace add https://github.com/AKCodez/hackingtool-plugin.git
/plugin install hackingtool@hackingtool-marketplace
```

Pick **Install** and choose **user** scope so it's available in every project. Restart `claude`
afterward; you should then see a `pentest` skill in the skills list.

> Install one marketplace at a time. If the shorthand `owner/repo` form fails with an SSH
> host-key or auth error, use the full `https://...git` URL instead.

## Step 7 — Add a CLAUDE.md for method and scope control

`CLAUDE.md` is standing instructions the agent reads automatically. Mine makes it triage before
going deep, keep notes, write solve scripts, and — critically — **refuse any pentest target not
listed in that engagement's `scope.txt`**. Keep it at `~/ctf/CLAUDE.md` so it applies to every
challenge subfolder.

Minimal example:

```markdown
# CTF & Pentest Assistant — Standing Instructions

## Scope (pentesting)
- Before sending ANY traffic, read `scope.txt` in the current folder.
- Only touch hosts/URLs listed there. If it's missing or a target isn't listed, STOP and ask.
- No DoS, no out-of-scope brute forcing.

## Workflow
1. Read `challenge.md` (CTF) or `scope.txt` + `brief.md` (pentest). Note the flag format.
2. Triage first: `file`, `strings`, `exiftool`, `binwalk`, `checksec`; `nmap -sC -sV` for hosts.
3. Form a hypothesis, test it, record results (including dead ends) in `notes.md`.
4. Put reusable solve logic in `solve.py` (pwntools / requests).
5. Stop at the flag (write it to `flag.txt`) or after ~5 failed approaches — then summarize.
```

## Step 8 — Verify end to end

Drop a self-contained challenge into `~/ctf/` and let the agent solve it:

```
~/ctf/
  CLAUDE.md
  intro-crypto/
    challenge.md      # describes the task + flag format
    cipher.txt        # a flag wrapped in hex -> base64 -> ROT13
```

```bash
cd ~/ctf/intro-crypto
claude
# then: "Solve the challenge in this folder. The flag format is flag{...}."
```

If it reads the files, identifies the encoding layers, and returns the flag, the setup works.

---

## Day-to-day usage

- **New CTF challenge:** make a folder under `~/ctf/`, add the files + a short `challenge.md`,
  run `claude` from inside it. Use `/clear` between challenges.
- **Pentest engagement:** same, but add a `scope.txt` listing the authorized hosts.
- Snapshot the VM once everything works, labeled e.g. "claude + plugins working."

### What to expect
Strong on crypto, web, forensics, and encoding/stego; decent on easy reversing; weak on hard
binary exploitation (pwn) and long multi-step chains. Treat it as a fast first pass and a
force multiplier, not a replacement for your own skills.

---

## Notes & safety

- **Isolation:** run everything in a disposable, snapshotted VM. In hands-off mode the agent
  can read anything your user can — keep SSH keys, saved passwords, and personal files out.
- **Authorization:** only test systems you own or have written permission to test.
- **Competition rules:** many CTFs ban or restrict AI/automated solving. Check first.
- **Cost:** Claude Pro ($20/mo) includes Claude Code; the free plan does not.

*Not affiliated with Anthropic. Third-party repos referenced here are maintained by their
respective owners; audit them yourself before use.*

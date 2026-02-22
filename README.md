# OpenClaw Security Hardening Guide

A systematic walkthrough for hardening your OpenClaw agent's behavior on macOS. This guide helps you think through security decisions — it doesn't prescribe specific settings.

**Time required:** 1-2 hours for full walkthrough
**Prerequisites:** Working OpenClaw installation, basic familiarity with your agent's capabilities

---

## ⚡ Quick Start — The Easy Button

**Just paste this into your OpenClaw chat:**

```
Read the security hardening guide at https://github.com/rbfp/openclaw-security-guide and work through it with me one category at a time. For each category:
1. Explain the risks in plain language
2. Present my options and tradeoffs
3. Wait for my decision before moving on
4. Write the agreed rules into my AGENTS.md

Don't rush ahead. One category at a time, starting with Category 1.
```

Your agent will fetch the guide, explain each section, help you make decisions, and write the rules directly into your config. You stay in control at every step.

> **Note:** Your agent's behavior after hardening will depend on the choices *you* make — this guide doesn't impose a specific configuration. Every setup is different.

---

## Why This Matters

Your OpenClaw agent can:
- Read and write files on your machine
- Execute shell commands
- Send emails and messages
- Make web requests
- Access your calendar, notes, and other apps
- Run automated tasks while you're away

This power is useful — and exploitable. A malicious email, webpage, or file could contain instructions that hijack your agent's behavior. This guide helps you build defenses.

---

## Threat Model

Before hardening, understand what you're protecting against:

| Threat | Description | Risk Level |
|--------|-------------|------------|
| **Prompt Injection** | Malicious content in emails/web pages that instructs your agent to take harmful actions | High |
| **Data Exfiltration** | Sensitive data (credentials, files, client info) leaving your machine through the agent | High |
| **Destructive Commands** | Accidental or manipulated `rm`, infrastructure teardown, etc. | High |
| **Credential Exposure** | API keys, passwords appearing in logs, messages, or memory files | Medium |
| **Runaway Automation** | Cron jobs or sub-agents doing damage unsupervised | Medium |
| **Scope Creep** | Agent doing more than asked, touching things it shouldn't | Low-Medium |

**Your first decision:** Which of these matter most to you? Rank them. Your answers will shape which categories you prioritize.

---

## Category 1: Behavioral — Confirmation Tiers

The foundation. Define what your agent can do freely vs. what requires your approval.

### The Concept: Three Tiers

**Tier 1 — Just Do It**
Low-risk, read-only, or fully reversible actions. No confirmation needed.

**Tier 2 — Tell Me First**
Medium-risk actions. Agent describes what it's about to do and waits for your explicit "go."

**Tier 3 — Hard Gate**
High-risk actions. Requires a secondary verification (passphrase, token, or multi-channel confirmation).

### Decision Points

**Question 1:** What actions should be Tier 1 (no confirmation)?
- Reading files, emails, calendar?
- Web searches and research?
- Writing to workspace files?
- Creating folders and channels?

**Question 2:** What actions should be Tier 2 (tell me first)?
- Sending emails or messages?
- Deleting files (even to trash)?
- Installing software?
- Modifying config files?
- Running shell commands?

**Question 3:** What actions should be Tier 3 (hard gate)?
- Destructive shell commands (`rm`, `dd`)?
- Infrastructure changes (`terraform apply`)?
- Financial transactions?
- Publishing content publicly?

**Question 4:** How should Tier 3 verification work?
- **Option A: Passphrase** — Agent asks for a secret word you've pre-shared
- **Option B: Token** — Agent generates a one-time code, posts it to a separate channel (Discord, Slack), you confirm
- **Option C: Dual-channel** — Requires confirmation in two different places

Token-based is stronger (can't be faked by injection), but more friction.

### Template

```markdown
## Confirmation Tiers

### Tier 1 — Just Do It
- [list your Tier 1 actions]

### Tier 2 — Tell Me First
- [list your Tier 2 actions]
Confirmation must come from [your username/ID].

### Tier 3 — Hard Gate
- [list your Tier 3 actions]

**Verification method:** [passphrase / token / dual-channel]
**Verification channel:** [where confirmations happen]
```

---

## Category 2: File System Scope

What can your agent read and write?

### Decision Points

**Question 1:** What paths should be freely accessible?
- Workspace directory only?
- Your entire home folder?
- Specific project directories?

**Question 2:** What paths need extra protection?
- iCloud Documents?
- SSH keys (`~/.ssh/`)?
- Application data?
- Client/work files?

**Question 3:** How should protected paths work?
- **Hard block** — Agent can never access, period
- **Token gate** — Requires secondary confirmation each time
- **Read-only** — Can read but not write
- **Audit only** — Can access but every access is logged

### Template

```markdown
## File System Scope

### Free Access
- `~/path/to/workspace/` — full read/write

### Protected Paths (requires confirmation)
- `~/path/to/sensitive/` — [token gate / read-only / audit]

### Off-Limits (never access)
- `~/.ssh/`
- [other paths]
```

---

## Category 3: Network & Outbound

Control what leaves your machine.

### Sub-decisions

**3A: Web Fetching**
- Allow all URLs freely? (convenient but riskier)
- Warn on unknown domains?
- Maintain an allowlist of permitted domains?

**3B: Email Sending**
- Allow sending to any address?
- Maintain a domain allowlist? (start empty, add as needed)
- Require confirmation for every email?

**3C: Data Egress Rules**
- Scan outbound URLs for embedded credentials?
- Block file contents from being transmitted?
- Inspect shell commands that send data (`curl`, `scp`)?

**3D: Audit Logging**
- Log every outbound network call?
- How long to retain logs?
- What to include? (domain only vs. full URL)

### Template

```markdown
## Network & Outbound

### Web Fetching
[Your policy — open / warn on unknown / allowlist]

### Email
Domain allowlist: `config/email-allowlist.md`
Currently allowed: [domains or "none — all blocked"]
Adding domains requires: [Tier 2 / Tier 3]

### Data Egress
- Scan URLs for credential patterns: [yes/no]
- Block transmission of: [list sensitive data types]
- Shell egress inspection: [yes/no]

### Audit Log
Location: `memory/outbound-audit.log`
Retention: [X days]
Format: [domain only / full URL]
```

---

## Category 4: Command Execution

Shell commands are powerful. Constrain them.

### Decision Points

**Question 1:** What commands should be hard-blocked?
Patterns so dangerous they should never run, regardless of authorization:
- `rm -rf /` — filesystem wipe
- `dd` to disk devices — disk destruction
- `mkfs` — format filesystem
- Fork bombs
- System file overwrites

**Question 2:** What commands should require confirmation?
- Commands targeting system paths (`/etc/`, `/usr/`)?
- Privilege escalation (`sudo`, `su`)?
- Network listeners (`nc -l`, `socat`)?
- Background processes (`nohup`, `screen`)?

**Question 3:** How to handle dynamically-constructed commands?
If the agent builds a shell command from external input (filenames, fetched content), should it:
- Sanitize metacharacters automatically?
- Show you the full command before running?
- Reject external content in commands entirely?

### Template

```markdown
## Command Execution

### Hard Denylist (never execute)
- `rm -rf /`, `rm -rf ~`
- `dd if=... of=/dev/disk*`
- `mkfs`
- [add your own]

### Requires Confirmation
- Commands targeting: `/dev/`, `/etc/`, `/System/`, `/usr/`
- Privilege escalation: `sudo`, `su`
- Network listeners
- Background detachment

### Dynamic Commands
- Sanitize shell metacharacters: [yes/no]
- Show constructed command before running: [yes/no]
```

---

## Category 5: Credential Handling

Protect your API keys, tokens, and passwords.

### Decision Points

**Question 1:** Where are your credentials stored?
- OpenClaw config (`~/.openclaw/openclaw.json`)?
- Environment variables?
- Separate secrets file?
- OS keychain?

**Question 2:** What should the agent never do with credentials?
- Write to memory files or logs?
- Include in Discord/Slack messages?
- Pass to sub-agents in task prompts?

**Question 3:** What files should be off-limits for reading?
- Config files with embedded keys?
- `.env` files?
- Keychain access?

**Question 4:** What happens on suspected exposure?
- Alert you immediately?
- Log the incident?
- Treat credential as compromised until rotated?

### Template

```markdown
## Credential Handling

### Never Write Credentials To
- Memory files
- Logs
- Messages
- [other locations]

### Off-Limits for Direct Reads
- `~/.openclaw/openclaw.json`
- `~/.openclaw/credentials/`
- [other sensitive paths]

### Exposure Protocol
On suspected leak:
1. Alert in [channel]
2. Log incident (without the value)
3. Treat as compromised until [rotation confirmed / X hours]
```

---

## Category 6: Cron & Sub-Agent Limits

Automated and isolated sessions need constraints too.

### Decision Points

**Question 1:** Should all guardrails apply to sub-agents?
Recommended: Yes. Isolated sessions should not be exempt from any rule.

**Question 2:** Can cron/sub-agents perform Tier 3 actions?
Tier 3 typically requires interactive confirmation. Options:
- **Block** — Cron jobs can't do Tier 3 things; they must alert and stop
- **Pre-authorize** — Specific cron tasks can be pre-approved for specific actions
- **Allow** — Trust cron jobs fully (not recommended)

**Question 3:** How many cron jobs is too many?
A runaway setup could create dozens. Set a soft ceiling (e.g., 10).

**Question 4:** Can task prompts come from external content?
If a cron prompt is derived from a fetched webpage or email, it's a persistent injection vector. Require confirmation for externally-derived prompts.

### Template

```markdown
## Cron & Sub-Agent Limits

### Guardrail Inheritance
All rules apply to cron sessions and sub-agents: [yes/no]

### Tier 3 from Isolated Sessions
Policy: [block and alert / pre-authorize / allow]

### Cron Job Ceiling
Soft limit: [number] active jobs

### External Content in Prompts
Task prompts from external sources require: [Tier 2 / block entirely]
```

---

## Category 7: Audit Trail

Logs give you visibility and forensic capability.

### Decision Points

**Question 1:** What should be logged?
- All outbound network calls?
- Injection attempts (detected)?
- Tier 2 confirmations?
- Protected path access?

**Question 2:** What should NOT be in logs?
- Full URLs (may contain tokens)?
- Credential values?
- Specific injection patterns (could be a bypass map)?

**Question 3:** How long to retain?
Balance forensic value against disk space. 30-90 days is typical.

**Question 4:** Should logs ever leave the machine?
Logs contain operational patterns. Recommend: gitignore, don't sync to cloud.

**Question 5:** Daily summary?
Should the agent surface notable security events proactively?

### Template

```markdown
## Audit Trail

### Logs Maintained
- `memory/outbound-audit.log` — network calls
- `memory/injection-log.md` — detected attempts
- `memory/tier2-audit.log` — confirmed actions
- [others]

### Retention
[X] days, pruned during maintenance

### Git/Cloud
All security logs in `.gitignore`: [yes/no]

### Daily Summary
Post to [channel] if notable events occurred: [yes/no]
```

---

## Category 8: OS-Level Hardening

Some protections exist outside the agent.

### Checklist

Run these checks and note current state:

**FileVault (disk encryption)**
```bash
fdesetup status
```
- [ ] Enabled
- [ ] Disabled — *recommend enabling*

**SIP (System Integrity Protection)**
```bash
csrutil status
```
- [ ] Enabled
- [ ] Disabled — *security risk*

**macOS Firewall**
```bash
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
```
- [ ] Enabled
- [ ] Disabled — *recommend enabling*

**Gatekeeper**
System Settings → Privacy & Security
- [ ] App Store and identified developers
- [ ] App Store only
- [ ] Anywhere — *security risk*

**Directory Permissions**
```bash
ls -la ~/.openclaw/
ls -la ~/.openclaw/workspace/
```
- [ ] Both are `700` (owner only)
- [ ] World-readable — *run `chmod 700` on both*

### Optional Enhancements

- **Little Snitch** — Per-process outbound network filtering
- **Append-only logging** — OS-level tamper-proof audit
- **Full Disk Access audit** — Review which apps have FDA

---

## Prompt Injection Defense

This deserves special attention. Your agent processes content from external sources — any of it could contain attacks.

### Detection Approach

**Two-level detection:**

**Yellow — Suspicious, likely benign**
Content *discusses* AI manipulation (e.g., a security blog post about injection). Pause, flag, ask if you want to continue.

**Red — Active attack**
Content *attempts* to override your agent's behavior — references tools by name, claims new permissions, sets up exfiltration. Hard stop, alert, log.

### Patterns to Watch For

Don't enumerate exact phrases in your config (they become a bypass map). Instead, train your agent to recognize *categories*:
- Instruction overrides ("ignore previous...")
- Identity replacement ("you are now...")
- False permission claims ("you have been authorized...")
- Tool/file targeting (references to specific tools or paths)
- Exfiltration setup ("send this to...", "fetch with parameter...")

### Research Mode

If you do security research, you'll encounter malicious content intentionally. Create a way to temporarily suppress Yellow alerts while keeping Red alerts active.

Recommend: A keyword (like `sudo`) that only works when verified as coming from you (not from content you're analyzing).

---

## Implementation Checklist

Work through these in order:

- [ ] **Category 1:** Define your three tiers
- [ ] **Category 2:** Map your filesystem — free, protected, off-limits
- [ ] **Category 3:** Set outbound policies and create audit log
- [ ] **Category 4:** Define command denylist and confirmation patterns
- [ ] **Category 5:** Identify credential locations and set handling rules
- [ ] **Category 6:** Establish sub-agent inheritance and cron limits
- [ ] **Category 7:** Create log files and set retention policy
- [ ] **Category 8:** Run OS-level checks and fix any gaps
- [ ] **Injection Defense:** Implement two-level detection

After implementation:
- [ ] Test each tier with a real action
- [ ] Verify logs are being written
- [ ] Confirm gitignore covers sensitive logs
- [ ] Document your rollback procedure

---

## Maintenance

Security isn't set-and-forget.

**Weekly:**
- Glance at security logs for anomalies

**Monthly:**
- Review cron job list — still needed?
- Check for new credential files that should be protected
- Update OS and OpenClaw

**Quarterly:**
- Re-read your AGENTS.md — still accurate?
- Review tier assignments — anything need adjustment?
- Test the Tier 3 flow to make sure it works

---

## Final Notes

**No configuration survives a compromised machine.** These guardrails are behavioral — they constrain what your agent *chooses* to do. If your machine is fully compromised at the OS level, all bets are off.

**The strongest protections are mechanism-based, not pattern-based.** Token verification, allowlists, and hard blocks work even if an attacker knows about them. Pattern detection (injection phrases) can be bypassed by a motivated attacker who's read your rules.

**Start restrictive, loosen as needed.** It's easier to add permissions than to recover from a security incident.

---

*This guide was developed through a hands-on hardening session. Adapt it to your needs, threat model, and risk tolerance.*

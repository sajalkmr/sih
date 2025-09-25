## Multi-Platform System Hardening Tool — Hackathon Pitch (Problem ID: 25237)

---

## Problem, Stakes, and Opportunity
### Background
Modern Windows, Ubuntu, and CentOS machines rarely ship aligned to strict baselines. Misconfigurations in passwords, services, network, logging, or audit can create exploitable gaps leading to breaches and downtime.

### The problem
- Manual hardening is time-consuming, error-prone, inconsistent across platforms, and difficult to audit or verify at scale, even with CIS/industry guidance.
- Teams lack a single, automated tool to assess, enforce, report, and roll back policies across Windows and Linux.

### The opportunity
- A cross-platform, automated hardening tool that operationalizes Annexure ‘A’ (Windows) and Annexure ‘B’ (Linux) into practical, one-command enforcement with strong reporting and rollback. This reduces time-to-baseline from days to minutes.

---

## Solution Overview and Architecture
### Vision
One command to assess and harden Windows 10/11, Ubuntu 20.04+, and CentOS 7+, with detailed reports (previous/current/status/severity) and safe rollback.

### Core capabilities
- OS detection and modular engine
- Policy levels: basic, moderate, strict (Windows and Linux)
- Enforcement aligned to Annexure A/B
- Reports (HTML and PDF) with severity ratings; JSONL audit logs with timestamps
- Tamper-evident audit trail: hash-chained events and optional blockchain anchoring
- Rollback: safe restore of prior configurations
- CLI-first for automation; GUI optional later

### Architecture (local, agentless)
- Python CLI orchestrator: `hardener/cli.py`
- OS detection: `hardener/os_detect.py`
- Windows modules: secedit policy, UAC, Audit, Firewall, Services, Security Options, User Rights, WDAG
- Linux modules: sysctl, sshd, firewall (ufw/firewalld), services, auditd, AIDE, PAM (pwquality/faillock/pwhistory), sudo, login defaults, banners, package removal, kernel modules blacklist, wireless/BT disable, chrony/time sync, logrotate, fstab mount options
- Config-as-data: YAML under `configs/`
- Reporting: HTML/Jinja and PDF/ReportLab (`hardener/reporting.py`)
- Tamper-evidence (optional): hash chain + blockchain anchor adapter (`hardener/blockchain.py`)
- Backups/rollback: JSON/INF snapshots under `backups/`

---

## What’s Implemented (MVP) + Live Demo
### Implemented highlights
- Windows (Annexure A):
  - Password and account lockout via secedit levels (`configs/windows/levels.yml`)
  - UAC, Audit Policy, Firewall profiles, Services disable, Security Options, WDAG, Rename accounts
  - Rollback for secedit policies
- Linux (Annexure B):
  - Levels for sysctl and sshd (baseline/basic/moderate/strict)
  - sysctl and sshd hardening; ufw/firewalld; services disable; auditd rules; AIDE; mount options (/tmp, /dev/shm, /var/tmp, /var/log, /var/log/audit, /home)
  - New modules: PAM (pwquality/faillock/pwhistory), sudo policy, login.defs, banners, legacy client package removal, UFW guard (iptables-persistent), wireless/BT disable, chrony alternative, logrotate baseline
  - Rollback for sysctl and sshd (best effort), UFW reset
- Reporting: HTML + PDF (key, previous, current, status, severity) and JSONL audit logs
- Tamper-evident audit (beta): per-event hash chain; optional testnet anchor via flag

### Demo script (5–6 minutes)
1) Detect OS
   - `python -m hardener.cli detect-os`
2) Windows strict enforcement + report (run as Admin)
   - `python -m hardener.cli enforce windows --level strict --report reports\\win.html --pdf reports\\win.pdf`
   - Show `reports/win.html` and `reports/win.pdf`
3) Linux enforcement with levels (sudo)
   - `sudo python -m hardener.cli enforce-linux --level moderate --report reports/linux.html --pdf reports/linux.pdf`
4) Extended Linux hardening (sudo)
   - `sudo python -m hardener.cli enforce-linux-extended --report reports/linux_extended.html`
   - Show sections: services, auditd, filesystem, AIDE, logging, PAM, sudo, accounts, banners, packages, ufw guard, wireless/BT, chrony, logrotate
5) Rollback
   - Windows: `python -m hardener.cli rollback windows --latest`
   - Linux: `sudo python -m hardener.cli rollback-linux`
6) (Optional) Anchor report/log hash to blockchain
   - `python -m hardener.cli anchor --input logs\\audit.jsonl --anchor-file logs\\anchors.jsonl` (local hash chain)
   - `python -m hardener.cli anchor --input logs\\audit.jsonl --rpc <RPC_URL> --wallet <WALLET>` (testnet)

---

## Technical Deep Dive
### Modular policy engine
- Config-as-data in YAML under `configs/`; each module enforces a focused domain.
- Windows levels via secedit export/apply; Linux levels via keyed YAML sections (`baseline/basic/moderate/strict`).

### Enforcement and safety
- Windows uses `secedit`, `auditpol`, `reg`, `netsh` with state exports to `backups/` for rollback.
- Linux writes controlled config diffs (e.g., `/etc/security/*.conf`, `/etc/pam.d/*`, `/etc/login.defs`, `/etc/logrotate.d/*`, `/etc/modprobe.d/*`, `/etc/fstab`) and restarts services where needed.

### Reporting and auditing
- HTML and PDF reports include: Key, Previous, Current, Status (Success/Failure), Severity (from `configs/severity.yml`).
- All commands emit JSONL audit events with timestamps (`logs/audit.jsonl`).
- Each event includes a content hash and `prev_hash` to form a chain of custody.
- Optional anchoring writes periodic Merkle roots or report hashes on-chain; only hashes leave the host.

### Rollback design
- Windows: policy snapshot as INF/JSON, re-applied on rollback.
- Linux: sysctl values restored from JSON; sshd restored from on-disk backup; UFW reset (best effort).

---

## Compliance Mapping, Impact, and Differentiation
### Annexure coverage
- Windows Annexure ‘A’: strong coverage for Account Policies, Security Options, Audit, Firewall, Services, UAC, User Rights, WDAG, rename accounts.
- Linux Annexure ‘B’: broad coverage including networking/sysctl, SSH, firewall, services, auditd, PAM, sudo, password aging, banners, package hygiene, kernel modules blacklist, time sync, logrotate, wireless/BT, fstab mount options.
- Source: Annexure PDF ([Annexure A/B PDF](https://sih.gov.in/dataset/Annexure_A_B_NTRO_SIH25237.pdf)).

### Impact metrics
- Time-to-baseline reduced from days to minutes across mixed estates.
- Consistency via a single CLI across Windows and Linux; levels for fast rollout.
- Auditability through deterministic reports, PDF export, and JSONL audit logs.
- Safety with point-in-time backups and targeted rollbacks.

### Differentiation
- Practical OS-native enforcement (no agent) plus rich reporting and rollback.
- Config-as-data (YAML) enables rapid policy updates and extensions without code changes.
- Tamper-evident auditing through hash-chained logs with optional blockchain anchoring.

---

## Roadmap, Deployment, Risks, and The Ask
### Roadmap (post-hackathon)
- Expand Linux partition management checks/remediation and PAM variants across distros.
- Add GUI (PyQt) with dashboards, one-click enforce/rollback.
- SQLite-backed rollback snapshots and richer diff visualizations.
- Profile-based policies per server role; scheduling and dry-run simulation.
- Managed ledger connectors (Hyperledger, Ethereum L2), batch/gasless anchors, and ZK options.

### Deployment
- Local install: `pip install -r requirements.txt` then `python -m hardener.cli ...`
- Package for Windows (MSI) and Linux (DEB/RPM) to align with easy installation objective.
- Optional blockchain config via CLI/env (e.g., `--rpc`, `--wallet`) for anchoring.

### Risks and mitigations
- Distro variance: mitigate via gated, idempotent changes and optional dry-run.
- Service restarts: batch operations and explicit maintenance windows.
- Privilege requirements: clear prompts and preflight checks.
- On-chain cost/privacy/latency: hash-only anchors by default; opt-in and offline-first.

### The ask
- Evaluate on a mixed Windows/Linux VM set; provide sample baselines to validate coverage and reports.
- Scoring focus: speed-to-compliance, report clarity, and safe rollback.

### Contact & References
- Problem Statement ID: 25237 — Multi-Platform System Hardening Tool
- Annexure dataset: the Annexure A/B PDF (link above)
- Key files to review: `hardener/cli.py`, `hardener/reporting.py`, `configs/windows/*`, `configs/linux/*`



# 🔍 Sentinel Log Monitor — Linux Log Anomaly Detection & Alerting

> A lightweight Python daemon that continuously tails Linux system logs, applies configurable regex-based detection rules, and fires Slack or email alerts when suspicious patterns appear — no Elasticsearch, no Kibana, no cloud dependency.
>
> ---
>
> ## 🎯 The Problem This Solves
>
> Most log monitoring solutions (ELK, Splunk, Datadog) are heavy, expensive, and overkill for a single Linux server or small fleet. This tool gives you real-time log anomaly detection and alerting in a single Python script that runs as a `systemd` service — no agents, no SaaS subscriptions, no 8GB Java heap.
>
> **Use cases:**
> - SSH brute-force detection (monitors `/var/log/auth.log`)
> - - Disk-full and OOM killer events (`/var/log/syslog`)
>   - - Application error spikes (custom app log files)
>     - - Failed service restarts (`journald` output)
>      
>       - ---
>
> ## 🏗️ Architecture
>
> ```
> ┌─────────────────────────────────────────────────────┐
> │                     Linux Server                    │
> │                                                     │
> │  /var/log/auth.log  ──┐                             │
> │  /var/log/syslog    ──┤→  Sentinel Daemon (Python)  │
> │  /var/log/app.log   ──┘       ↓                     │
> │                        Pattern Matcher              │
> │                        (regex rules.yaml)           │
> │                               ↓                     │
> │                        Alert Dispatcher             │
> │                       ↙              ↘              │
> │               Slack Webhook      Email (SMTP)       │
> └─────────────────────────────────────────────────────┘
> ```
>
> ---
>
> ## 🔥 Features
>
> - **Real-time log tailing** — Uses `inotify` to tail files efficiently (no polling)
> - - **YAML-driven rules** — Add detection patterns without touching Python code
>   - - **Multi-file monitoring** — Watch multiple log files simultaneously in threads
>     - - **Deduplication** — Configurable cooldown period prevents alert storms
>       - - **Structured output** — JSON log format for easy downstream processing
>         - - **systemd integration** — Ships with a unit file for production deployment
>           - - **Dry-run mode** — Test your rules without sending real alerts
>             - - **Zero external dependencies** — Only stdlib + `PyYAML` + `requests`
>              
>               - ---
>
> ## 🛠️ Tech Stack
>
> | Component       | Technology                          |
> |-----------------|-------------------------------------|
> | Language        | Python 3.11                         |
> | File watching   | `inotify` (via `inotify-simple`)    |
> | Config format   | YAML (rules + settings)             |
> | Alerting        | Slack Webhooks, SMTP                |
> | Process mgmt    | systemd service unit                |
> | Logging         | Python `logging` → JSON output      |
> | OS              | Ubuntu 22.04 / RHEL 8+              |
>
> ---
>
> ## 📁 Project Structure
>
> ```
> sentinel-log-monitor/
> ├── sentinel.py              # Main daemon — tailing, matching, dispatching
> ├── config.yaml              # Global settings (log paths, cooldowns, channels)
> ├── rules/
> │   ├── auth.yaml            # SSH and authentication failure patterns
> │   ├── system.yaml          # Disk, memory, OOM killer patterns
> │   └── custom.yaml          # Add your own patterns here
> ├── alerters/
> │   ├── slack.py             # Slack webhook dispatcher
> │   └── email.py             # SMTP email dispatcher
> ├── tests/
> │   ├── test_matcher.py      # Unit tests for pattern matching logic
> │   └── test_alerters.py     # Mock-based alert dispatcher tests
> ├── systemd/
> │   └── sentinel.service     # systemd unit file for production
> └── README.md
> ```
>
> ---
>
> ## 🚀 Quick Start
>
> ### Install
>
> ```bash
> git clone https://github.com/ashllybr/sentinel-log-monitor.git
> cd sentinel-log-monitor
> pip install -r requirements.txt
> ```
>
> ### Configure
>
> ```yaml
> # config.yaml
> log_files:
>   - /var/log/auth.log
>   - /var/log/syslog
>
> alert_cooldown_seconds: 300   # Don't re-alert the same pattern within 5 min
>
> slack:
>   webhook_url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
>
> email:
>   smtp_host: "smtp.gmail.com"
>   smtp_port: 587
>   from: "alerts@yourdomain.com"
>   to: "you@yourdomain.com"
> ```
>
> ### Define Detection Rules
>
> ```yaml
> # rules/auth.yaml
> rules:
>   - name: "SSH Brute Force"
>     pattern: "Failed password for .* from (\d+\.\d+\.\d+\.\d+)"
>     threshold: 5          # Trigger after 5 matches within the window
>     window_seconds: 60
>     severity: "critical"
>     message: "Possible SSH brute force from {match_1}"
>
>   - name: "Root Login Attempt"
>     pattern: "Failed password for root"
>     threshold: 1
>     severity: "high"
>     message: "Root login attempt detected"
> ```
>
> ### Run
>
> ```bash
> # Test rules without sending alerts
> python sentinel.py --dry-run
>
> # Run as foreground process
> python sentinel.py
>
> # Install as systemd service
> sudo cp systemd/sentinel.service /etc/systemd/system/
> sudo systemctl enable --now sentinel
> sudo systemctl status sentinel
> ```
>
> ---
>
> ## 🔧 Sample Output
>
> ```json
> {
>   "timestamp": "2026-02-18T14:32:01Z",
>   "rule": "SSH Brute Force",
>   "severity": "critical",
>   "log_file": "/var/log/auth.log",
>   "matched_line": "Failed password for admin from 192.168.1.105 port 52341 ssh2",
>   "match_count": 7,
>   "alert_sent": true,
>   "channel": "slack"
> }
> ```
>
> ---
>
> ## 📊 Detection Rules Library
>
> | Rule Name             | Log File         | Detects                            |
> |-----------------------|------------------|------------------------------------|
> | SSH Brute Force       | auth.log         | Repeated failed SSH logins         |
> | Root Login Attempt    | auth.log         | Any failed root login              |
> | OOM Killer Triggered  | syslog           | Process killed due to memory       |
> | Disk Space Critical   | syslog           | Filesystem >90% full               |
> | Service Crashed       | syslog/journald  | systemd service entered failed state |
> | Sudo Privilege Abuse  | auth.log         | Unexpected sudo usage              |
>
> ---
>
> ## 💡 Design Decisions
>
> | Decision | Reasoning |
> |---|---|
> | Python over Bash | Complex pattern matching, threading, and HTTP requests are cleaner in Python |
> | inotify over polling | Event-driven file watching is more efficient — no wasted CPU cycles |
> | YAML rules file | Ops teams can add detection rules without touching application code |
> | Alert cooldown | Prevents alert fatigue during log storms (e.g., a service crash loop) |
> | systemd integration | Standard Linux process management — enables auto-restart and logging via journald |
>
> ---
>
> ## 🔮 What I'd Add Next
>
> - [ ] Prometheus metrics endpoint (`/metrics`) — expose match counts per rule
> - [ ] - [ ] Web dashboard — simple Flask UI to view recent alerts
> - [ ] - [ ] Geo-IP enrichment — Attach country/city info to IP addresses in alerts
> - [ ] - [ ] PagerDuty integration — Escalation paths for on-call rotation
> - [ ] - [ ] Regex tester CLI — `sentinel test-rule --rule auth.yaml --line "Failed password..."`
> - [ ] - [ ] Rate limiting per source IP — Detect but don't alert duplicate IPs repeatedly
>
> - [ ] ---
>
> - [ ] ## 🔑 Skills Demonstrated
>
> - [ ] - Linux system programming (file watching, systemd, log formats)
> - [ ] - Python daemon design (threading, signal handling, graceful shutdown)
> - [ ] - Security monitoring and detection engineering fundamentals
> - [ ] - YAML-driven configuration design
> - [ ] - Alert deduplication and rate-limiting patterns
> - [ ] - Unit testing with mocks for external services (Slack, SMTP)
> - [ ] - Infrastructure observability thinking
>
> - [ ] ---
>
> - [ ] ## 📜 License
>
> - [ ] MIT — Free to use and extend.
>
> - [ ] ---
>
> - [ ] ## 👤 Author
>
> - [ ] **Alex Brian** — DevOps & Linux Systems Engineer (RHCE)
>
> - [ ] 📧 ashllybr01@gmail.com | 💻 [GitHub](https://github.com/ashllybr)
>
> - [ ] ---
>
> - [ ] ⭐ *Star this repo if you've ever been woken up by a security incident you could have caught earlier!*

# 🔍 Sentinel Log Monitor — Linux Log Anomaly Detection & Alerting# Sentinel_log_monitor

Real-time log anomaly detection for Linux. Tails your system logs, matches patterns you define in YAML, and fires off a Slack or email alert when something looks wrong. No Elasticsearch, no Kibana, no cloud account needed.

Built for sysadmins who want to know when their server is getting hammered without standing up a full observability stack.

---

## What it does

Runs as a systemd daemon and watches whatever log files you point it at. When a line matches one of your rules it checks if it's crossed the threshold (e.g. 5 failed SSH logins in 60 seconds), and if so it sends an alert and backs off for a cooldown period so you don't get spammed.

Good for:
- Catching SSH brute force attempts before they become a problem
- - Knowing when a service crashes in the middle of the night
  - - Alerting on OOM kills or full disks before users notice
    - - Flagging unexpected sudo usage on shared servers
     
      - ---

      ## How it works

      ```
      /var/log/auth.log  --.
      /var/log/syslog    --+--> Sentinel daemon --> Pattern matcher --> Alert dispatcher
      /var/log/app.log   --'         (inotify)       (rules.yaml)      (Slack / email)
      ```

      Uses inotify to watch files so there's no polling and no wasted CPU. Each file gets its own thread. Matches run against a compiled regex so they stay fast even at high log volume.

      ---

      ## Tech used

      - Python 3.11
      - - inotify-simple (file watching)
        - - PyYAML (rule config)
          - - requests (Slack webhooks)
            - - systemd (process management)
              - - Runs on Ubuntu 22.04 and RHEL 8+
               
                - ---

                ## Project layout

                ```
                sentinel-log-monitor/
                |- sentinel.py          # main daemon, handles tailing + dispatching
                |- config.yaml          # which files to watch, cooldown settings, alert channels
                |- rules/
                |  |- auth.yaml         # SSH and login failure rules
                |  |- system.yaml       # disk, memory, OOM rules
                |  |- custom.yaml       # add your own here
                |- alerters/
                |  |- slack.py          # Slack webhook sender
                |  |- email.py          # SMTP sender
                |- tests/
                |  |- test_matcher.py
                |  |- test_alerters.py
                |- systemd/
                |  |- sentinel.service  # drop this in /etc/systemd/system/
                |- README.md
                ```

                ---

                ## Setup

                ```bash
                git clone https://github.com/ashllybr/sentinel-log-monitor.git
                cd sentinel-log-monitor
                pip install -r requirements.txt
                ```

                Edit config.yaml to point at your log files and add your Slack webhook or SMTP settings:

                ```yaml
                log_files:
                  - /var/log/auth.log
                  - /var/log/syslog

                alert_cooldown_seconds: 300

                slack:
                  webhook_url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

                email:
                  smtp_host: "smtp.gmail.com"
                  smtp_port: 587
                  from: "alerts@yourdomain.com"
                  to: "you@yourdomain.com"
                ```

                ---

                ## Writing rules

                Rules go in the `rules/` folder. Each one has a pattern, a threshold, and a message:

                ```yaml
                rules:
                  - name: "SSH brute force"
                    pattern: "Failed password for .* from (\\d+\\.\\d+\\.\\d+\\.\\d+)"
                    threshold: 5
                    window_seconds: 60
                    severity: "critical"
                    message: "Possible brute force from {match_1}"

                  - name: "Root login attempt"
                    pattern: "Failed password for root"
                    threshold: 1
                    severity: "high"
                    message: "Someone tried to log in as root"
                ```

                ---

                ## Running it

                ```bash
                # Test your rules without sending real alerts
                python sentinel.py --dry-run

                # Run in the foreground to see what's happening
                python sentinel.py

                # Install as a systemd service
                sudo cp systemd/sentinel.service /etc/systemd/system/
                sudo systemctl enable --now sentinel
                sudo systemctl status sentinel
                ```

                ---

                ## Sample alert output

                ```json
                {
                  "timestamp": "2026-02-18T14:32:01Z",
                  "rule": "SSH brute force",
                  "severity": "critical",
                  "log_file": "/var/log/auth.log",
                  "matched_line": "Failed password for admin from 192.168.1.105 port 52341 ssh2",
                  "match_count": 7,
                  "alert_sent": true,
                  "channel": "slack"
                }
                ```

                ---

                ## Detection rules included

                | Rule | Log file | What it catches |
                |---|---|---|
                | SSH brute force | auth.log | Multiple failed logins from same IP |
                | Root login attempt | auth.log | Any failed root login |
                | OOM kill | syslog | Process killed due to out of memory |
                | Disk full | syslog | Filesystem over 90% |
                | Service crash | syslog | systemd service entered failed state |
                | Unexpected sudo | auth.log | Sudo usage outside of expected patterns |

                ---

                ## Why not just use ELK / Grafana / Datadog

                Those tools are great at scale. For a single server or a small VPS fleet they're overkill. This script is ~300 lines of Python, runs as a service, uses almost no resources, and gets the job done.

                ---

                ## What's next

                - [ ] Prometheus /metrics endpoint so you can track match counts over time
                - [ ] - [ ] Geo-IP lookup on IP addresses in alerts
                - [ ] - [ ] Simple web UI to browse recent alerts
                - [ ] - [ ] PagerDuty support
                - [ ] - [ ] CLI tool to test a rule against a log file without running the daemon
               
                - [ ] ---
               
                - [ ] ## Skills this covers
               
                - [ ] Linux system programming, Python daemon patterns, security monitoring basics, YAML config design, unit testing with mocks, systemd integration
               
                - [ ] ---
               
                - [ ] ## License
               
                - [ ] MIT
               
                - [ ] ---
               
                - [ ] ## Author
               
                - [ ] Alex Brian - DevOps and Linux systems (RHCE)
               
                - [ ] ashllybr01@gmail.com | github.com/ashllybr

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

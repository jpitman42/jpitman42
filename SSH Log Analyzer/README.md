Hello, I'm Justin

<a href="https://linkedin.com/in/justin-pitman-347384267"><img src="https://img.shields.io/badge/-LinkedIn-0072b1?&style=for-the-badge&logo=linkedin&logoColor=white" /></a>

I am a data analyst interested in transitioning into the cybersecurity field

# SSH Log Analyzer

A lightweight, dependency-free Python tool that parses SSH authentication
logs and flags suspicious activity — brute-force attempts, username
scanning, and logins that follow a burst of failures.

Built as a beginner-friendly defensive security project: it doesn't attack
anything, it just helps you (or a blue team) notice when someone else is
trying to.

## Features

- **Brute-force detection** — flags any IP with N+ failed login attempts
  within a sliding time window (defaults: 5 attempts / 60 seconds, both
  configurable).
- **Username-probing detection** — flags IPs that try many different
  invalid usernames, a classic sign of automated scanning.
- **Suspicious success detection** — flags successful logins that
  immediately follow several failed attempts from the same IP, a possible
  sign a weak password was eventually guessed.
- **JSON export** for feeding results into other tools or dashboards.
- **Zero dependencies** — pure Python 3 standard library.

## Quick start

```bash
# Clone the repo
git clone https://github.com/yourusername/ssh-log-analyzer.git
cd ssh-log-analyzer

# (Optional) generate a realistic fake log to try it out on
python3 generate_sample_log.py > sample_logs/sample_auth.log

# Run the analyzer
python3 log_analyzer.py sample_logs/sample_auth.log
```

### Example output

```
============================================================
SSH LOG ANALYSIS REPORT
============================================================
Events parsed:       29
Failed logins:       19
Successful logins:   10

[!] Brute-force suspects: 2
    203.0.113.77     8 attempts in 60s (total failed: 8)
    198.51.100.23    7 attempts in 60s (total failed: 7)

[!] Username-probing IPs: 1
    198.51.100.23    tried 7 usernames: ftpuser, guest, oracle, pi, postgres...

[!] Suspicious successful logins: 1
    line 26: 192.0.2.44 logged in as 'admin' after 4 recent failures (2026-09-15T09:35:36)
============================================================
```

## Usage on a real server

Most Linux systems log SSH activity to `/var/log/auth.log` (Debian/Ubuntu)
or `/var/log/secure` (RHEL/CentOS/Fedora). You'll typically need `sudo` to
read it:

```bash
sudo python3 log_analyzer.py /var/log/auth.log
```

### Options

| Flag          | Default | Description                                            |
|---------------|---------|----------------------------------------------------------|
| `--window`    | `60`    | Time window (seconds) for brute-force detection          |
| `--threshold` | `5`     | Failed attempts within the window to trigger a flag       |
| `--json FILE` | —       | Also write the full report to `FILE` as JSON              |

Example with tighter, more sensitive thresholds:

```bash
python3 log_analyzer.py /var/log/auth.log --window 30 --threshold 3 --json report.json
```

## How it works

The analyzer parses standard `sshd` log lines with regular expressions,
classifying each into `failed` or `accepted` events. It then runs three
independent detectors over the parsed events:

1. **Brute force** — for each IP, uses a sliding-window scan over sorted
   failure timestamps to find the densest cluster of attempts.
2. **Username probing** — for each IP, counts the number of *distinct*
   invalid usernames attempted (a real user typically only tries one or
   two valid usernames — a scanner tries dozens).
3. **Suspicious success** — replays events in order, and for every
   successful login checks whether that same IP had several recent
   failures beforehand.

## Testing

The project includes unit tests for each detector, plus a sample log
generator so you can test without needing a real server:

```bash
python3 -m unittest test_log_analyzer.py -v
```

## Project structure

```
ssh-log-analyzer/
├── log_analyzer.py          # Main analyzer (parsing + detection + reporting)
├── generate_sample_log.py   # Generates a realistic fake log for testing
├── test_log_analyzer.py     # Unit tests for the detection logic
├── sample_logs/
│   └── sample_auth.log      # Example generated log
├── README.md
├── LICENSE
└── .gitignore
```

## Limitations & ideas for extending

This is intentionally scoped as a learning/portfolio project. Natural next
steps if you want to extend it:

- Support more log formats (nginx/Apache auth, Windows Event Logs, cloud
  provider audit logs).
- Add a `--live` mode that tails a log file in real time (like `tail -f`)
  and prints alerts as they happen.
- Send alerts to Slack/email/webhook when a new IP is flagged.
- Add geo-IP lookups to show where attempts are coming from.
- Persist state across runs (SQLite) so brute-force windows can span
  multiple invocations, not just a single file.
- Package it as a pip-installable CLI (`pip install ssh-log-analyzer`).

# CyberArk PAM – Intermediate Projects

A collection of hands-on automation projects for portfolio building, real-world use, and home lab practice.

---

## Project 1: Automated Account Onboarding Tool

**Purpose:** Automate bulk onboarding of privileged accounts from HR/IT provisioning data.  
**Use case:** Real work | Portfolio  
**Skills:** Python, REST API, CSV parsing, error handling

### Overview
When a new server or service account is provisioned, this tool reads a CSV export and automatically creates the account in the correct CyberArk safe with the right platform.

### Project Structure
```
account-onboarding/
├── onboard.py           # Main script
├── accounts_input.csv   # Input file (safe, platform, address, username)
├── onboard_report.csv   # Output report (success/failure per account)
├── config.py            # PVWA URL, CPM name, platform mappings
└── README.md
```

### Key Features to Build
- Read accounts from CSV
- Authenticate to PVWA via REST API
- Validate safe exists — create it if not
- Add each account with proper platform assignment
- Log success/failure per account to an output report
- Send summary email on completion (use `smtplib`)

### Sample Config (`config.py`)
```python
PVWA_URL = "https://pvwa.example.com"
CPM_NAME = "PasswordManager"
PLATFORM_MAP = {
    "windows": "WinServerLocal",
    "linux": "UnixSSH",
    "oracle": "Oracle"
}
```

### Stretch Goals
- Add a `--dry-run` flag to preview without making changes
- Slack/Teams webhook notification on completion
- Support for multiple PVWA environments (dev/prod)

---

## Project 2: Password Health Dashboard

**Purpose:** Audit all accounts and report on password health (last changed, failures, disabled CPM).  
**Use case:** Real work | Portfolio  
**Skills:** Python, REST API, pandas, HTML report generation

### Overview
Query all accounts across all safes, check their CPM status, and generate an HTML dashboard showing which passwords are overdue for rotation or have CPM errors.

### Project Structure
```
password-health-dashboard/
├── dashboard.py          # Main script
├── report_template.html  # Jinja2 HTML template
├── output/
│   └── report.html       # Generated dashboard
├── config.py
└── README.md
```

### Key Features to Build
- Fetch all accounts via paginated API calls
- For each account, retrieve:
  - `lastModifiedTime`
  - `secretManagement.status` (success, failure, disabled)
  - `secretManagement.lastModifiedTime`
- Flag accounts where password is older than X days
- Generate an HTML report with color-coded status table
- Schedule with cron or Windows Task Scheduler

### Sample Status Logic
```python
import datetime

def get_password_status(account):
    mgmt = account.get("secretManagement", {})
    status = mgmt.get("status", "unknown")
    last_change = mgmt.get("lastModifiedTime", 0)
    days_since = (datetime.datetime.now().timestamp() - last_change) / 86400

    if status == "failure":
        return "🔴 CPM Error"
    elif days_since > 90:
        return "🟡 Overdue (>90 days)"
    elif status == "success":
        return "🟢 Healthy"
    else:
        return "⚪ Unknown"
```

### Stretch Goals
- Export to Excel with color formatting using `openpyxl`
- Email the report as an HTML attachment
- Add filtering by safe name or platform

---

## Project 3: Safe Provisioning Automation

**Purpose:** Automatically create safes and assign members based on a request form or ITSM ticket.  
**Use case:** Real work | Lab  
**Skills:** Python, REST API, JSON config, role-based permissions

### Overview
When a new application team needs a safe, they fill out a JSON request file. This tool reads it, creates the safe, assigns the right members with least-privilege permissions, and logs the action.

### Project Structure
```
safe-provisioning/
├── provision_safe.py       # Main script
├── requests/
│   └── app_team_request.json  # Input request
├── permissions/
│   ├── admin.json           # Permission templates
│   ├── readonly.json
│   └── operator.json
├── logs/
│   └── provisioning.log
└── README.md
```

### Sample Request JSON
```json
{
  "safe_name": "AppTeam-PaymentService",
  "description": "Safe for Payment Service credentials",
  "cpm": "PasswordManager",
  "retention_days": 7,
  "members": [
    { "name": "svc_paymentapp", "role": "operator" },
    { "name": "john.doe", "role": "readonly" },
    { "name": "cyberark_admin", "role": "admin" }
  ]
}
```

### Permission Templates (`permissions/readonly.json`)
```json
{
  "useAccounts": false,
  "retrieveAccounts": false,
  "listAccounts": true,
  "viewAuditLog": true,
  "viewSafeMembers": true
}
```

### Stretch Goals
- Integrate with ServiceNow via its REST API to auto-close tickets
- Add a `--rollback` option to delete the safe if provisioning fails midway
- Support bulk provisioning from multiple JSON files in a folder

---

## Project 4: PSM Session Monitor & Auto-Terminator

**Purpose:** Monitor live PSM sessions and auto-terminate sessions that exceed a time limit or match a blacklist.  
**Use case:** Real work | Lab  
**Skills:** Python, REST API, scheduling, alerting

### Overview
A background service that polls active PSM sessions every N minutes, alerts on long-running or suspicious sessions, and optionally terminates them based on policy rules.

### Project Structure
```
session-monitor/
├── monitor.py           # Main loop
├── policy.json          # Rules (max duration, blacklisted targets)
├── alerts.py            # Slack/email alerting
├── config.py
└── README.md
```

### Sample Policy (`policy.json`)
```json
{
  "max_session_minutes": 120,
  "alert_at_minutes": 90,
  "auto_terminate": true,
  "blacklisted_targets": ["10.0.0.1", "dc01.corp.local"],
  "excluded_users": ["cyberark_admin"]
}
```

### Core Monitor Loop
```python
import time

def monitor_sessions(base_url, token, policy):
    while True:
        sessions = get_active_sessions(base_url, token)
        for session in sessions:
            duration_mins = session.get("Duration", 0) / 60
            target = session.get("RemoteMachine", "")
            user = session.get("User", "")

            if user in policy["excluded_users"]:
                continue

            if target in policy["blacklisted_targets"]:
                send_alert(f"⚠️ Blacklisted target accessed: {target} by {user}")
                if policy["auto_terminate"]:
                    terminate_session(base_url, token, session["SessionID"])

            elif duration_mins >= policy["max_session_minutes"]:
                send_alert(f"⏱️ Session exceeded {policy['max_session_minutes']}min: {user} → {target}")
                if policy["auto_terminate"]:
                    terminate_session(base_url, token, session["SessionID"])

        time.sleep(300)  # Poll every 5 minutes
```

### Stretch Goals
- Store session history in SQLite for trend analysis
- Build a simple web UI using Flask to view session stats
- Integrate with SIEM (Splunk/QRadar) via syslog

---

## Project 5: CyberArk DR Readiness Checker

**Purpose:** Validate that your DR (Disaster Recovery) vault is in sync and ready for failover.  
**Use case:** Lab | Portfolio  
**Skills:** Python, REST API, comparison logic, reporting

### Overview
Connects to both Primary and DR vaults, compares safe counts, account counts, and member lists, and generates a readiness report flagging any discrepancies.

### Project Structure
```
dr-readiness-checker/
├── dr_check.py           # Main comparison script
├── config.py             # Primary + DR PVWA URLs and credentials
├── output/
│   └── dr_report.html    # Generated report
└── README.md
```

### Core Comparison Logic
```python
def compare_vaults(primary_data, dr_data):
    discrepancies = []

    primary_safes = {s['SafeName'] for s in primary_data['safes']}
    dr_safes = {s['SafeName'] for s in dr_data['safes']}

    missing_in_dr = primary_safes - dr_safes
    for safe in missing_in_dr:
        discrepancies.append({
            "type": "Missing Safe",
            "detail": f"Safe '{safe}' exists in Primary but NOT in DR"
        })

    # Compare account counts per safe
    for safe in primary_safes & dr_safes:
        p_count = primary_data['account_counts'].get(safe, 0)
        d_count = dr_data['account_counts'].get(safe, 0)
        if p_count != d_count:
            discrepancies.append({
                "type": "Account Count Mismatch",
                "detail": f"Safe '{safe}': Primary={p_count}, DR={d_count}"
            })

    return discrepancies
```

### Stretch Goals
- Schedule daily and email the report to the security team
- Add vault component health checks (CPM, PSM, PVWA status)
- Track discrepancy trends over time in a database

---

## Project 6: Secrets Rotation Notifier

**Purpose:** Notify application teams before their shared credentials are rotated by CPM.  
**Use case:** Real work  
**Skills:** Python, REST API, email/Slack notifications, scheduling

### Overview
Some apps can't handle automatic password rotation without downtime. This tool identifies accounts scheduled for rotation soon and notifies the responsible team in advance.

### Project Structure
```
rotation-notifier/
├── notifier.py          # Main script
├── team_registry.json   # Maps safe names to team contacts
├── config.py
└── README.md
```

### Sample Team Registry
```json
{
  "AppTeam-PaymentService": {
    "team": "Payments Team",
    "email": "payments@company.com",
    "slack_channel": "#payments-alerts"
  },
  "AppTeam-HRSystem": {
    "team": "HR IT",
    "email": "hr-it@company.com",
    "slack_channel": "#hr-it-alerts"
  }
}
```

### Core Notification Logic
```python
def check_upcoming_rotations(accounts, days_ahead=3):
    upcoming = []
    now = datetime.datetime.now().timestamp()
    threshold = now + (days_ahead * 86400)

    for account in accounts:
        next_change = account.get("secretManagement", {}).get("lastModifiedTime", 0)
        # Estimate next rotation based on platform policy interval
        # (You'd pull platform details for actual interval)
        if next_change and next_change <= threshold:
            upcoming.append(account)

    return upcoming
```

### Stretch Goals
- Pull platform CPM interval to accurately predict next rotation time
- Allow teams to request a rotation delay via a self-service form
- Log all notifications to an audit trail

---

## Home Lab Setup Tips

To practice these projects without a production CyberArk environment:

1. **CyberArk Trial** — Request a 30-day trial at [cyberark.com](https://www.cyberark.com)
2. **CyberArk Developer Environment** — Free sandbox via [CyberArk Marketplace](https://cyberark-customers.force.com/mplace/s/)
3. **Mock PVWA API** — Build a simple Flask app that mimics PVWA REST endpoints for unit testing
4. **Use Postman** — Import CyberArk's official Postman collection to explore APIs before scripting

---

## Resume / Portfolio Tips

- Host each project in its own GitHub repo with a clear README
- Include a `demo/` folder with sample input/output files (no real credentials!)
- Add a architecture diagram showing how the tool fits into a PAM workflow
- Document what CyberArk components each project interacts with (PVWA, CPM, PSM)
- Tag repos with topics: `cyberark`, `pam`, `privileged-access`, `security-automation`
---

## ✅ Built Projects

These projects have been fully built out with complete source code:

| # | Project | Repo | Status |
|---|---------|------|--------|
| 2 | **Password Health Dashboard** | [cyberark-password-health-dashboard](https://github.com/satish2001-max/cyberark-password-health-dashboard) | ✅ Complete |

> More projects will be added here as they are built out.

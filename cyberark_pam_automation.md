# CyberArk PAM Scripts & Automation Guide

## Table of Contents
1. [REST API Authentication](#rest-api-authentication)
2. [Account Management](#account-management)
3. [Safe Management](#safe-management)
4. [Session Management](#session-management)
5. [Bulk Operations](#bulk-operations)
6. [Reporting & Auditing](#reporting--auditing)
7. [PowerShell Module (psPAS)](#powershell-module-pspas)

---

## REST API Authentication

### Get Auth Token (CyberArk Authentication)
```python
import requests
import json

def get_cyberark_token(base_url, username, password):
    """
    Authenticate to CyberArk PVWA and retrieve a session token.
    """
    url = f"{base_url}/PasswordVault/API/auth/Cyberark/Logon"
    headers = {"Content-Type": "application/json"}
    payload = {
        "username": username,
        "password": password,
        "concurrentSession": True
    }

    response = requests.post(url, headers=headers, json=payload, verify=False)
    response.raise_for_status()
    return response.text.strip('"')

# Usage
BASE_URL = "https://pvwa.example.com"
token = get_cyberark_token(BASE_URL, "admin", "Password123!")
print(f"Token: {token}")
```

### LDAP Authentication
```python
def get_ldap_token(base_url, username, password):
    url = f"{base_url}/PasswordVault/API/auth/LDAP/Logon"
    headers = {"Content-Type": "application/json"}
    payload = {"username": username, "password": password}

    response = requests.post(url, headers=headers, json=payload, verify=False)
    response.raise_for_status()
    return response.text.strip('"')
```

### Logoff
```python
def logoff(base_url, token):
    url = f"{base_url}/PasswordVault/API/auth/Logoff"
    headers = {
        "Authorization": token,
        "Content-Type": "application/json"
    }
    requests.post(url, headers=headers, verify=False)
    print("Logged off successfully.")
```

---

## Account Management

### List All Accounts
```python
def list_accounts(base_url, token, safe_name=None, keyword=None):
    url = f"{base_url}/PasswordVault/API/Accounts"
    headers = {"Authorization": token}
    params = {}

    if safe_name:
        params["filter"] = f"safeName eq {safe_name}"
    if keyword:
        params["search"] = keyword

    response = requests.get(url, headers=headers, params=params, verify=False)
    response.raise_for_status()
    return response.json().get("value", [])

# Usage
accounts = list_accounts(BASE_URL, token, safe_name="Windows-Servers")
for account in accounts:
    print(f"ID: {account['id']} | Username: {account['userName']} | Address: {account['address']}")
```

### Add a New Account
```python
def add_account(base_url, token, safe_name, platform_id, address, username, secret):
    url = f"{base_url}/PasswordVault/API/Accounts"
    headers = {
        "Authorization": token,
        "Content-Type": "application/json"
    }
    payload = {
        "safeName": safe_name,
        "platformId": platform_id,
        "address": address,
        "userName": username,
        "secretType": "password",
        "secret": secret,
        "platformAccountProperties": {},
        "secretManagement": {
            "automaticManagementEnabled": True
        }
    }

    response = requests.post(url, headers=headers, json=payload, verify=False)
    response.raise_for_status()
    return response.json()

# Usage
new_account = add_account(
    BASE_URL, token,
    safe_name="Windows-Servers",
    platform_id="WinServerLocal",
    address="192.168.1.100",
    username="Administrator",
    secret="InitialP@ss123!"
)
print(f"Created Account ID: {new_account['id']}")
```

### Delete an Account
```python
def delete_account(base_url, token, account_id):
    url = f"{base_url}/PasswordVault/API/Accounts/{account_id}"
    headers = {"Authorization": token}

    response = requests.delete(url, headers=headers, verify=False)
    response.raise_for_status()
    print(f"Account {account_id} deleted successfully.")
```

### Retrieve (Get) a Password
```python
def get_password(base_url, token, account_id, reason="Automation script"):
    url = f"{base_url}/PasswordVault/API/Accounts/{account_id}/Password/Retrieve"
    headers = {
        "Authorization": token,
        "Content-Type": "application/json"
    }
    payload = {"reason": reason}

    response = requests.post(url, headers=headers, json=payload, verify=False)
    response.raise_for_status()
    return response.text.strip('"')
```

### Verify / Change / Reconcile Password
```python
def verify_password(base_url, token, account_id):
    url = f"{base_url}/PasswordVault/API/Accounts/{account_id}/Verify"
    headers = {"Authorization": token}
    requests.post(url, headers=headers, verify=False)
    print(f"Verification triggered for account {account_id}.")

def change_password(base_url, token, account_id):
    url = f"{base_url}/PasswordVault/API/Accounts/{account_id}/Change"
    headers = {"Authorization": token}
    requests.post(url, headers=headers, verify=False)
    print(f"Password change triggered for account {account_id}.")

def reconcile_password(base_url, token, account_id):
    url = f"{base_url}/PasswordVault/API/Accounts/{account_id}/Reconcile"
    headers = {"Authorization": token}
    requests.post(url, headers=headers, verify=False)
    print(f"Reconciliation triggered for account {account_id}.")
```

---

## Safe Management

### List All Safes
```python
def list_safes(base_url, token):
    url = f"{base_url}/PasswordVault/API/Safes"
    headers = {"Authorization": token}

    response = requests.get(url, headers=headers, verify=False)
    response.raise_for_status()
    return response.json().get("value", [])
```

### Create a New Safe
```python
def create_safe(base_url, token, safe_name, description, cpm_name, retention_days=7):
    url = f"{base_url}/PasswordVault/API/Safes"
    headers = {
        "Authorization": token,
        "Content-Type": "application/json"
    }
    payload = {
        "SafeName": safe_name,
        "Description": description,
        "OLACEnabled": False,
        "ManagingCPM": cpm_name,
        "NumberOfVersionsRetention": retention_days
    }

    response = requests.post(url, headers=headers, json=payload, verify=False)
    response.raise_for_status()
    return response.json()

# Usage
safe = create_safe(BASE_URL, token, "Linux-Prod-Servers", "Production Linux servers", "PasswordManager")
print(f"Safe created: {safe['SafeName']}")
```

### Add Safe Member
```python
def add_safe_member(base_url, token, safe_name, member_name, permissions):
    url = f"{base_url}/PasswordVault/API/Safes/{safe_name}/Members"
    headers = {
        "Authorization": token,
        "Content-Type": "application/json"
    }
    payload = {
        "memberName": member_name,
        "memberType": "User",
        "permissions": permissions
    }

    response = requests.post(url, headers=headers, json=payload, verify=False)
    response.raise_for_status()
    return response.json()

# Full permissions example
FULL_PERMISSIONS = {
    "useAccounts": True,
    "retrieveAccounts": True,
    "listAccounts": True,
    "addAccounts": True,
    "updateAccountContent": True,
    "updateAccountProperties": True,
    "initiateCPMAccountManagementOperations": True,
    "specifyNextAccountContent": True,
    "renameAccounts": True,
    "deleteAccounts": True,
    "unlockAccounts": True,
    "manageSafe": True,
    "manageSafeMembers": True,
    "backupSafe": True,
    "viewAuditLog": True,
    "viewSafeMembers": True,
    "requestsAuthorizationLevel1": True,
    "accessWithoutConfirmation": True,
    "createFolders": True,
    "deleteFolders": True,
    "moveAccountsAndFolders": True
}
```

---

## Session Management

### Get Active Sessions
```python
def get_active_sessions(base_url, token):
    url = f"{base_url}/PasswordVault/API/LiveSessions"
    headers = {"Authorization": token}

    response = requests.get(url, headers=headers, verify=False)
    response.raise_for_status()
    return response.json().get("LiveSessions", [])

# Usage
sessions = get_active_sessions(BASE_URL, token)
for s in sessions:
    print(f"Session ID: {s['SessionID']} | User: {s['User']} | Target: {s['RemoteMachine']}")
```

### Terminate a Session
```python
def terminate_session(base_url, token, session_id):
    url = f"{base_url}/PasswordVault/API/LiveSessions/{session_id}/Terminate"
    headers = {"Authorization": token}

    response = requests.post(url, headers=headers, verify=False)
    response.raise_for_status()
    print(f"Session {session_id} terminated.")
```

---

## Bulk Operations

### Bulk Add Accounts from CSV
```python
import csv

def bulk_add_accounts_from_csv(base_url, token, csv_file):
    """
    CSV format: safe_name, platform_id, address, username, secret
    """
    results = []
    with open(csv_file, newline='') as f:
        reader = csv.DictReader(f)
        for row in reader:
            try:
                account = add_account(
                    base_url, token,
                    safe_name=row['safe_name'],
                    platform_id=row['platform_id'],
                    address=row['address'],
                    username=row['username'],
                    secret=row['secret']
                )
                results.append({"status": "success", "id": account['id'], "address": row['address']})
                print(f"✅ Added: {row['username']}@{row['address']}")
            except Exception as e:
                results.append({"status": "failed", "address": row['address'], "error": str(e)})
                print(f"❌ Failed: {row['username']}@{row['address']} — {e}")
    return results

# Usage
# bulk_add_accounts_from_csv(BASE_URL, token, "accounts.csv")
```

### Bulk Trigger Password Change
```python
def bulk_change_passwords(base_url, token, safe_name):
    accounts = list_accounts(base_url, token, safe_name=safe_name)
    for account in accounts:
        try:
            change_password(base_url, token, account['id'])
            print(f"✅ Change triggered: {account['userName']}@{account['address']}")
        except Exception as e:
            print(f"❌ Failed: {account['userName']}@{account['address']} — {e}")
```

---

## Reporting & Auditing

### Export All Accounts to CSV
```python
def export_accounts_to_csv(base_url, token, output_file="accounts_report.csv"):
    accounts = list_accounts(base_url, token)
    fieldnames = ["id", "userName", "address", "safeName", "platformId", "createdTime"]

    with open(output_file, "w", newline='') as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames, extrasaction='ignore')
        writer.writeheader()
        writer.writerows(accounts)

    print(f"Exported {len(accounts)} accounts to {output_file}")
```

### Get Audit Log for a Safe
```python
def get_safe_audit_log(base_url, token, safe_name):
    url = f"{base_url}/PasswordVault/API/Safes/{safe_name}/Activities"
    headers = {"Authorization": token}

    response = requests.get(url, headers=headers, verify=False)
    response.raise_for_status()
    return response.json().get("Activities", [])
```

---

## PowerShell Module (psPAS)

### Install & Connect
```powershell
# Install psPAS module
Install-Module -Name psPAS -Force

# Import and connect
Import-Module psPAS
New-PASSession -Credential (Get-Credential) -BaseURI "https://pvwa.example.com" -AuthType CyberArk
```

### Common psPAS Commands
```powershell
# List all safes
Get-PASSafe

# List accounts in a safe
Get-PASAccount -safeName "Windows-Servers"

# Add a new account
Add-PASAccount -safe "Windows-Servers" `
               -platformID "WinServerLocal" `
               -address "192.168.1.50" `
               -userName "Administrator" `
               -secret (ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force)

# Trigger password change
Invoke-PASCPMOperation -AccountID "12_3" -ChangeTask

# Get active PSM sessions
Get-PASSession

# Close session
Close-PASSession
```

### Bulk Safe Creation from CSV (PowerShell)
```powershell
Import-Module psPAS
New-PASSession -Credential (Get-Credential) -BaseURI "https://pvwa.example.com" -AuthType CyberArk

$safes = Import-Csv "safes.csv"  # Columns: SafeName, Description, CPM
foreach ($safe in $safes) {
    try {
        Add-PASSafe -SafeName $safe.SafeName `
                    -Description $safe.Description `
                    -ManagingCPM $safe.CPM `
                    -NumberOfVersionsRetention 7
        Write-Host "✅ Created: $($safe.SafeName)"
    } catch {
        Write-Host "❌ Failed: $($safe.SafeName) — $_"
    }
}
```

---

## Tips & Best Practices

- **Always logoff** after API sessions to avoid token leaks.
- **Use `verify=False` only in dev/test** — enable SSL verification in production.
- **Store credentials securely** — use environment variables or a secrets manager, never hardcode.
- **Handle pagination** — large vaults may return paginated results; check `nextLink` in responses.
- **Rate limiting** — add delays in bulk operations to avoid overwhelming the PVWA.
- **Use service accounts** — create dedicated API accounts with least-privilege permissions.
- **Log everything** — maintain audit trails for all automation actions.


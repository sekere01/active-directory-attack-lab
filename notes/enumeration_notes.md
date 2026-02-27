# Enumeration Notes — corp.local

## Network Scanning

**Tool:** Nmap 7.95  
**Date:** 2026-02-27  
**Target:** 192.168.10.10 (DC01)

### Command
```bash
nmap -sV -sC -p 88,135,139,389,445,3268,3389 192.168.10.10
```

### Open Ports

| Port | Service | Notes |
|------|---------|-------|
| 88   | Kerberos | Confirms Domain Controller |
| 135  | MSRPC | Windows RPC |
| 139  | NetBIOS | Legacy Windows networking |
| 389  | LDAP | Revealed domain: corp.local |
| 445  | SMB | SMB signing enabled and required |
| 3389 | RDP | Filtered — blocked by firewall |

### Key Observations
- SMB signing is **enabled and required** — prevents SMB relay attacks
- Domain name **corp.local** leaked via LDAP without authentication
- Clock skew: 21 seconds — within Kerberos tolerance (max 5 minutes)
- OS: Windows Server 2025 Build 26100 x64
- Host confirmed as **DC01**

---

## Active Directory Enumeration

**Tool:** SharpHound + BloodHound  
**Collected as:** corp\john.doe (standard domain user)

### SharpHound Collection (run on CORP-PC)
```powershell
Set-ExecutionPolicy Bypass -Scope Process
. .\SharpHound.ps1
Invoke-BloodHound -CollectionMethod All
```

**Output file:** `20260227053244_BloodHound.zip`

### BloodHound Findings

**Query:** Shortest Paths to Domain Admins

**Attack path identified:**
```
john.doe → [domain user] → svc.sql [has SPN] → attack path to Domain Admins
```

**Notable findings:**
- `jane.smith` — Kerberos pre-authentication disabled (AS-REP Roastable)
- `svc.sql` — SPN assigned: `MSSQLSvc/dc01.corp.local:1433` (Kerberoastable)
- `helpdesk` — local admin on CORP-PC (lateral movement vector)
- All users have logged into CORP-PC — credentials cached on workstation

### Users Enumerated

| Username | Privilege Level | Notable Flag |
|----------|----------------|--------------|
| administrator | Domain Admin | Weak password |
| john.doe | Standard User | — |
| jane.smith | Standard User | DoesNotRequirePreAuth = True |
| svc.sql | Service Account | SPN: MSSQLSvc/dc01.corp.local:1433 |
| helpdesk | Standard User + Local Admin on CORP-PC | Cached on workstation |

### SPNs Identified
```
MSSQLSvc/dc01.corp.local:1433  →  corp\svc.sql
```

Verified with:
```bash
python3 GetUserSPNs.py corp.local/john.doe:Password123! -dc-ip 192.168.10.10
```

---

## Key Attack Surface Summary

1. **AS-REP Roasting** — jane.smith has pre-auth disabled, hash requestable with no credentials
2. **Kerberoasting** — svc.sql has SPN, TGS ticket requestable with any domain credentials
3. **Credential caching** — all users have logged into CORP-PC, hashes stored in LSASS
4. **Weak passwords** — all accounts use simple passwords present in rockyou.txt
5. **Local admin** — helpdesk has local admin on CORP-PC, enabling remote secretsdump

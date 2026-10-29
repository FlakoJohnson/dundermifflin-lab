# Dunder Mifflin AD Lab

An Active Directory attack lab themed after *The Office*, built for [Ludus](https://ludus.cloud). Two-domain forest with parent/child trust, MSSQL Server with impersonation chains, ADCS (ESC1–16), DACL abuse chains, constrained/unconstrained delegation, RBCD, cross-domain ACL abuse, and a loaded Kali attacker VM.

## Lab Design

```
                    ┌─────────────────────────────────┐
                    │        dundermifflin.local       │
                    │         (Corporate HQ)           │
                    │                                  │
                    │  DM-DC01  (.10)  Root DC         │
                    │  DM-SQL01 (.11)  MSSQL Server    │
                    └──────────────┬───────────────────┘
                                   │ parent/child trust
                    ┌──────────────┴───────────────────┐
                    │  scranton.dundermifflin.local     │
                    │       (Scranton Branch)           │
                    │                                   │
                    │  SC-DC01  (.20)  Child DC + ADCS  │
                    │  SC-WS01  (.31)  Win 11 Wkst     │
                    │  SC-KALI  (.99)  Attacker         │
                    └───────────────────────────────────┘
```

All VMs on VLAN 10 (`10.x.10.0/24`). Router at `.254`.

## Quick Start

```bash
# 0. Clone the repo
git clone https://github.com/FlakoJohnson/dundermifflin-lab.git
cd dundermifflin-lab

# 1. Add the win2019 template (not in the default Ludus source)
ludus templates add -d /opt/ludus/sources/*/templates/win2019-server-x64
ludus templates build -n win2019-server-x64-template
# Wait for the build to complete (~45 min)
ludus templates status

# 2. Install roles (custom + community)
ludus ansible role add -d roles/dundermifflin_parent_dc
ludus ansible role add -d roles/dundermifflin_child_dc
ludus ansible role add -d roles/dundermifflin_member
ludus ansible role add -d roles/dundermifflin_workstation
ludus ansible role add badsectorlabs.ludus_adcs
ludus ansible role add badsectorlabs.ludus_mssql

# 3. Set the config
ludus range config set -f dundermifflin.yml

# 4. Deploy
ludus range deploy

# 5. Monitor
ludus range logs -f
```

> **Note:** Ludus 2.3+ no longer ships `win2019-server-x64-template` by default. Step 1 adds it from the BSL community source. If the glob path doesn't match, find it with `find /opt/ludus/sources -name "win2019-server-x64" -type d`.

## Users

### dundermifflin.local (Corporate)

| User | sAMAccountName | Role | Password | Groups | Notes |
|------|---------------|------|----------|--------|-------|
| David Wallace | dwallace | CFO | Suck1tDevice! | Domain Admins, Enterprise Admins, Corporate-Executives | DA+EA, cached creds on SQL01 |
| Jan Levinson | jlevinson | VP Sales | Candles2026! | Corporate-Executives | |
| Ryan Howard | rhoward | Temp | WUPHF2026! | Corporate-IT | AS-REP roastable, SQL login with sa impersonation |
| svc-sql | svc-sql | SQL Service | SQLSvc2026! | — | Kerberoastable (MSSQLSvc SPN), SQL sysadmin |
| svc-backup | — | — | — | — | GPP cpassword in SYSVOL |

### scranton.dundermifflin.local (Scranton Branch)

| User | sAMAccountName | Role | Password | Groups | Notes |
|------|---------------|------|----------|--------|-------|
| Michael Scott | mscott | Regional Manager | Password1 | Scranton-Management | AS-REP roastable, local admin on WS01 |
| Dwight Schrute | dschrute | Asst. Regional Mgr | Beets2026! | Scranton-Management, Scranton-Sales | GenericWrite target (from jhalpert), ForceChangePassword on tflenderson, GenericWrite on parent Corporate-IT |
| Jim Halpert | jhalpert | Sales Rep | BigTuna01 | Scranton-Sales | GenericWrite on dschrute (DACL chain entry) |
| Pam Beesly | pbeesly | Receptionist | Art2026! | Scranton-Reception | GenericWrite on SC-WS01 (RBCD) |
| Angela Martin | amartin | Head of Accounting | Cats4Ever! | Scranton-Accounting | GenericWrite on shudson (targeted kerberoast) |
| Kevin Malone | kmalone | Accountant | Kevin1! | Scranton-Accounting | AS-REP roastable, password in LDAP description |
| Oscar Martinez | omartinez | Accountant | Actually2026! | Scranton-Accounting | |
| Stanley Hudson | shudson | Sales Rep | Pretzel1 | Scranton-Sales | GenericWrite target (from amartin, for targeted kerberoast) |
| Toby Flenderson | tflenderson | HR Rep | Divorced3x | Scranton-HR, DnsAdmins | ForceChangePassword target (from dschrute), DACL chain endpoint |
| Creed Bratton | cbratton | Quality Assurance | Cr33dBr4tt0n | Scranton-QA, Backup Operators, Server Operators | WriteDacl on Domain Admins |
| svc-print | svc-print | Print Service | Pr1ntSvc2026! | — | Kerberoastable (HTTP SPN), constrained delegation → CIFS/SC-DC01 |
| svc-deploy | — | — | — | — | GPP cpassword in SYSVOL |

### ADCS ESC Users (created by `badsectorlabs.ludus_adcs`)

| User | sAMAccountName | Character | ESC Role |
|------|---------------|-----------|----------|
| Andy Bernard | abernard | The Nard Dog | ESC5 — CA object rights |
| Robert California | rcalifornia | The Lizard King | ESC7 — CA manager |
| Angela (cert) | amartin_cert | Angela's cert admin | ESC7 — Certificate manager |
| Kelly Kapoor | kellykapoor | Customer Service | ESC9 — GenericAll target |
| Meredith Palmer | meredithr | Supplier Relations | ESC10 — Weak cert binding |
| Darryl Philbin | darryljr | Warehouse | ESC13 — Issuance policy |
| Phyllis Vance | phyllisvance | Sales | ESC14 — attacker |
| Todd Packer | todpacker | Travelling Salesman | ESC14 — target |
| Gabe Lewis | glewis | Corporate IT | ESC16 — NTDS CA security |

Template names are realistic (WorkstationAuth, SubordinateCA, EnrollmentAgent, etc.) instead of ESC1/ESC2.

### MSSQL Accounts (on DM-SQL01)

| Login | Type | Password | Role | Notes |
|-------|------|----------|------|-------|
| dundermifflin\svc-sql | AD | SQLSvc2026! | sysadmin | Kerberoast → SQL sysadmin → xp_cmdshell |
| dundermifflin\rhoward | AD | WUPHF2026! | public | EXECUTE AS LOGIN = 'sa' (impersonation → sysadmin) |
| appuser | SQL | App2026! | public | EXECUTE AS USER = 'dbo' in msdb (Agent job access) |
| sa | SQL | sa_P@ssw0rd! | sysadmin | Brute-force target |

## Attack Surface

### Credential Access

| Vulnerability | Details |
|--------------|---------|
| AS-REP Roast | rhoward (parent), mscott + kmalone (child) |
| Kerberoast | svc-sql/MSSQLSvc (parent), svc-print/HTTP (child) |
| GPP cpassword | Corporate-ServiceAccount-Deploy (svc-backup), Scranton-LocalAdmin-Deploy (svc-deploy) |
| Cached DA Credentials | dwallace logged into DM-SQL01 — LSASS dump recovers DA hash |
| Password in Description | kmalone LDAP description contains password |
| Credentials in Shares | \\\\SC-DC01\IT\backup_sql.ps1 (SQL creds), \\\\SC-DC01\HR\salary_review.txt (dwallace password) |
| Weak Passwords | Multiple sprayable: Password1, Pretzel1, BigTuna01, Kevin1! |

### MSSQL Attack Paths

| Path | Chain |
|------|-------|
| Kerberoast → xp_cmdshell | Kerberoast svc-sql → crack → SQL auth as sysadmin → xp_cmdshell |
| AS-REP → impersonation → xp_cmdshell | AS-REP roast rhoward → SQL auth → EXECUTE AS LOGIN = 'sa' → xp_cmdshell |
| SQL brute → dbo impersonation | appuser (App2026!) → EXECUTE AS USER = 'dbo' in msdb → Agent job access |
| SA brute force | sa (sa_P@ssw0rd!) → direct sysadmin → xp_cmdshell |
| Coerced authentication | Any SQL access → xp_dirtree to attacker → NTLMv2 (machine account, pairs with unconstrained delegation) |

### DACL Abuse

| Path | Chain |
|------|-------|
| 3-hop DACL chain | jhalpert → GenericWrite → dschrute → ForceChangePassword → tflenderson (DnsAdmins → DLL injection on DC) |
| Targeted kerberoast | amartin → GenericWrite → shudson (set SPN, roast, clean up) |
| WriteDACL → DA | cbratton → WriteDACL on Domain Admins |
| Cross-domain ACL | dschrute (child) → GenericWrite on parent Corporate-IT group |

### Delegation

| Path | Details |
|------|---------|
| Unconstrained delegation | DM-SQL01 — coerce + capture TGT |
| Constrained delegation (S4U) | svc-print → CIFS/SC-DC01 with protocol transition |
| RBCD | pbeesly → GenericWrite on SC-WS01 (create machine account, configure delegation) |

### Certificate Abuse

| Path | Details |
|------|---------|
| ADCS ESC1–16 | Full Certified Pre-Owned attack surface via [`badsectorlabs.ludus_adcs`](https://github.com/badsectorlabs/ludus_adcs) |
| ADCS ESC8 | HTTP-based web enrollment on SC-DC01 — NTLM relay target |

### Privilege Escalation

| Path | Details |
|------|---------|
| Backup Operators | cbratton — backup SAM/SYSTEM for offline hash extraction |
| Server Operators | cbratton — service control for code execution on DCs |
| DnsAdmins | tflenderson — DLL injection on domain controller |
| Enterprise Admin | dwallace — forest-wide privilege |
| Local Admin | mscott on SC-WS01 |

## Requirements

- Ludus 2.3+ (needed for correct license activation and role support)
- Templates: `win2019-server-x64-template`, `win11-22h2-x64-enterprise-template`, `kali-x64-desktop-template`, `debian-12-x64-server-template`
- Minimum 24 GB RAM for the range (5 Windows VMs @ 4GB + Kali @ 4GB + router @ 2GB)

## Credentials

| Account | Password | Scope |
|---------|----------|-------|
| localuser | password | All VMs (Ludus default) |
| domainadmin | password | dundermifflin.local (Ludus default) |
| Administrator | password | dundermifflin.local (Ludus default) |
| kali | kali | Kali VM |

## Kali Tools

The Kali VM comes pre-installed with: nmap, netexec, impacket-scripts, bloodhound, certipy-ad, responder, evil-winrm, smbclient, ldap-utils, john, hashcat, chisel.

## Architecture Notes

- **Parent DC** (DM-DC01): Promoted by Ludus natively via the `domain` config block. Creates corporate users, groups, GPP cpassword GPO, and configures AS-REP roastable accounts.
- **Child DC** (SC-DC01): Promoted by the `dundermifflin_child_dc` Ansible role using `Install-ADDSDomain -DomainType ChildDomain`. Creates Scranton users/groups, configures DACL abuse chain, constrained delegation, targeted kerberoast path, cross-domain ACL, DnsAdmins membership, GPP cpassword, and SMB shares. Skips promotion on re-runs (checks if domain exists first).
- **MSSQL Server** (DM-SQL01): Domain-joined by Ludus natively. SQL Server 2019 installed by [`badsectorlabs.ludus_mssql`](https://github.com/badsectorlabs/ludus_mssql) with mixed-mode auth, xp_cmdshell, impersonation chains, and SA enabled. Post-config by `dundermifflin_member` enables WDigest and simulates DA logon for LSASS credential caching. Unconstrained delegation set by parent DC role.
- **Workstations** (SC-WS01): Domain-joined to scranton by the `dundermifflin_workstation` role using parent domain credentials. RBCD path configured (pbeesly GenericWrite on computer object).
- **ADCS**: Installed by [`badsectorlabs.ludus_adcs`](https://github.com/badsectorlabs/ludus_adcs) on SC-DC01 with `role_vars` overrides (child DC has no native Ludus domain block). ESC1 through ESC16 vulnerable templates with Office-themed users and realistic template names.

## Troubleshooting

- **Server 2022 template**: Do NOT use — Edge Appx blocks sysprep, causing SID duplication. Use `win2019-server-x64-template` instead.
- **Debian 11 router**: Stale apt repos. The config overrides to `debian-12-x64-server-template`.
- **WinRM timeouts**: The R610 (or similar constrained hardware) may struggle with 6 VMs sysprepping simultaneously. Ludus handles retries; a resume deploy usually completes what the first pass missed.
- **Async wrapper timeout on IP change**: Ansible's `WIN_ASYNC_STARTUP_TIMEOUT` defaults to 5 seconds. On slower hardware, the fire-and-forget IP change task fails before the namedpipe connects, leaving the VM at its DHCP address. Subsequent plays silently skip it (unreachable at the expected static IP), and the child DC never gets promoted. **Fix:** add `win_async_startup_timeout = 60` under `[defaults]` in `/etc/ansible/ansible.cfg` on the Ludus host. Note: the `win_async_startup_timeout: 3600` in Ludus's `group_vars/windows.yml` creates a variable, not the config setting — it has no effect.
- **ADCS on child DC**: The `badsectorlabs.ludus_adcs` role handles Enterprise Admins via `ludus_adcs_domain_username` (set to `dundermifflin\domainadmin`). If ADCS fails with `ERROR_DS_RANGE_CONSTRAINT`, verify the domain admin account has Enterprise Admins membership.
- **Workstation domain join**: Uses parent domain creds (`dundermifflin.local\domainadmin`) to join the child domain — the child domain doesn't have its own `domainadmin` account.
- **ludus_adcs idempotency**: The upstream role is not fully idempotent on re-runs. A [UPN fix PR](https://github.com/badsectorlabs/ludus_adcs/pull/12) addresses user creation failures on repeated deploys.

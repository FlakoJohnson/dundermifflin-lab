# Dunder Mifflin AD Lab

An Active Directory attack lab themed after *The Office*, built for [Ludus](https://ludus.cloud). Two-domain forest with parent/child trust, ADCS, vulnerable GPOs, kerberoastable and AS-REP roastable accounts, and a loaded Kali attacker VM.

## Lab Design

```
                    ┌─────────────────────────────────┐
                    │        dundermifflin.local       │
                    │         (Corporate HQ)           │
                    │                                  │
                    │  DM-DC01  (.10)  Root DC         │
                    │  DM-SQL01 (.11)  SQL Server      │
                    └──────────────┬───────────────────┘
                                   │ forest trust
                    ┌──────────────┴───────────────────┐
                    │  scranton.dundermifflin.local     │
                    │       (Scranton Branch)           │
                    │                                   │
                    │  SC-DC01  (.20)  Child DC + ADCS  │
                    │  SC-WS01  (.31)  Win 11 Wkst      │
                    │  SC-WS02  (.32)  Win 11 Wkst      │
                    │  SC-KALI  (.99)  Attacker          │
                    └───────────────────────────────────┘
```

All VMs on VLAN 10 (`10.x.10.0/24`). Router at `.254`.

## Quick Start

```bash
# 0. Clone the repo
git clone https://github.com/FlakoJohnson/dundermifflin-lab.git
cd dundermifflin-lab

# 1. Install the roles
ludus ansible role add -d roles/dundermifflin_parent_dc
ludus ansible role add -d roles/dundermifflin_child_dc
ludus ansible role add -d roles/dundermifflin_member
ludus ansible role add -d roles/dundermifflin_workstation

# 2. Set the config
ludus range config set -f dundermifflin.yml

# 3. Deploy
ludus range deploy

# 4. Monitor
ludus range logs -f
```

## Users

### dundermifflin.local (Corporate)

| User | sAMAccountName | Role | Password | Groups | Notes |
|------|---------------|------|----------|--------|-------|
| David Wallace | dwallace | CFO | Suck1tDevice! | Domain Admins, Corporate-Executives | DA in parent domain |
| Jan Levinson | jlevinson | VP Sales | Candles2026! | Corporate-Executives | |
| Ryan Howard | rhoward | Temp | WUPHF2026! | Corporate-IT | |
| svc-sql | svc-sql | SQL Service | SQLSvc2026! | — | Kerberoastable (MSSQLSvc SPN) |

### scranton.dundermifflin.local (Scranton Branch)

| User | sAMAccountName | Role | Password | Groups | Notes |
|------|---------------|------|----------|--------|-------|
| Michael Scott | mscott | Regional Manager | Password1 | Scranton-Management | AS-REP roastable, local admin on WS01/WS02 |
| Dwight Schrute | dschrute | Asst. Regional Mgr | Beets2026! | Scranton-Management, Scranton-Sales | |
| Jim Halpert | jhalpert | Sales Rep | BigTuna01 | Scranton-Sales | |
| Pam Beesly | pbeesly | Receptionist | Art2026! | Scranton-Reception | |
| Angela Martin | amartin | Head of Accounting | Cats4Ever! | Scranton-Accounting | |
| Kevin Malone | kmalone | Accountant | Kevin1! | Scranton-Accounting | AS-REP roastable, password in LDAP description |
| Oscar Martinez | omartinez | Accountant | Actually2026! | Scranton-Accounting | |
| Stanley Hudson | shudson | Sales Rep | Pretzel1 | Scranton-Sales | |
| Toby Flenderson | tflenderson | HR Rep | Divorced3x | Scranton-HR | |
| Creed Bratton | cbratton | Quality Assurance | Cr33dBr4tt0n | Scranton-QA, Backup Operators, Server Operators | WriteDacl on Domain Admins |

## Attack Surface

| Vulnerability | Type | Details |
|--------------|------|---------|
| AS-REP Roast | Credential | mscott (Password1), kmalone (Kevin1!) |
| Kerberoast | Credential | svc-sql (SQLSvc2026!) — MSSQLSvc/DM-SQL01:1433 |
| GPP cpassword | Credential | Two GPOs: Corporate-ServiceAccount-Deploy (svc-backup), Scranton-LocalAdmin-Deploy (svc-deploy) |
| ADCS ESC1 | Privilege Escalation | User template with ENROLLEE_SUPPLIES_SUBJECT — any user can request a cert as any other user |
| ADCS ESC8 | Privilege Escalation | HTTP-based web enrollment on SC-DC01 — NTLM relay target |
| Unconstrained Delegation | Privilege Escalation | DM-SQL01 — coerce + relay for TGT capture |
| Excessive Privileges | Privilege Escalation | cbratton in Backup Operators + Server Operators + WriteDacl on Domain Admins |
| Password in Description | Credential | kmalone LDAP description contains password |
| Credentials in Shares | Credential | \\SC-DC01\IT\backup_sql.ps1 (SQL SA password), \\SC-DC01\HR\salary_review.txt (dwallace password) |
| Local Admin | Lateral Movement | mscott is local admin on SC-WS01 and SC-WS02 |
| Domain Admin (parent) | Privilege Escalation | dwallace is DA in dundermifflin.local — cross-forest escalation via trust |
| Weak Passwords | Credential | Multiple sprayable passwords (Password1, Pretzel1, BigTuna01, etc.) |

## Requirements

- Ludus 2.3+ with NFR/Pro license (for router roles)
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

The Kali VM comes pre-installed with: nmap, netexec, impacket-scripts, bloodhound, certipy-ad, responder, evil-winrm, smbclient, ldap-utils, john, hashcat, chisel, crackmapexec.

## Architecture Notes

- **Parent DC** (DM-DC01): Promoted by Ludus natively via the `domain` config block
- **Child DC** (SC-DC01): Promoted by the `dundermifflin_child_dc` Ansible role using `Install-ADDSDomain -DomainType ChildDomain`
- **SQL Server** (DM-SQL01): Domain-joined by Ludus natively, post-config by `dundermifflin_member` role
- **Workstations**: Domain-joined to scranton by the `dundermifflin_workstation` role
- **ADCS**: Enterprise Root CA installed by the child DC role

## Troubleshooting

- **Server 2022 template**: Do NOT use — Edge Appx blocks sysprep, causing SID duplication. Use `win2019-server-x64-template` instead.
- **Debian 11 router**: Stale apt repos. The config overrides to `debian-12-x64-server-template`.
- **WinRM timeouts**: The R610 (or similar constrained hardware) may struggle with 6 VMs sysprepping simultaneously. Ludus handles retries; a resume deploy usually completes what the first pass missed.
- **ADCS on child DC**: Requires Enterprise Admins membership. The `dundermifflin_child_dc` role adds `localuser` to Enterprise Admins on the parent DC before installing ADCS. If ADCS fails with `ERROR_DS_RANGE_CONSTRAINT`, check Enterprise Admins membership.
- **Workstation domain join**: Uses parent domain creds (`dundermifflin.local\domainadmin`) to join the child domain — the child domain doesn't have its own `domainadmin` account.

# Dunder Mifflin AD Lab

An Active Directory attack lab themed after *The Office*, built for [Ludus](https://ludus.cloud). Two-domain forest with parent/child trust, ADCS (ESC1–16), vulnerable GPOs, kerberoastable and AS-REP roastable accounts, cached DA credentials, and a loaded Kali attacker VM.

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

# 1. Add the win2019 template (not in the default Ludus source)
ludus templates add -d /opt/ludus/sources/*/templates/win2019-server-x64
ludus templates build -n win2019-server-x64-template
# Wait for the build to complete (~45 min)
ludus templates status

# 2. Install roles (custom + community ADCS)
ludus ansible role add -d roles/dundermifflin_parent_dc
ludus ansible role add -d roles/dundermifflin_child_dc
ludus ansible role add -d roles/dundermifflin_member
ludus ansible role add -d roles/dundermifflin_workstation
ludus ansible role add badsectorlabs.ludus_adcs

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
| David Wallace | dwallace | CFO | Suck1tDevice! | Domain Admins, Corporate-Executives | DA in parent domain, cached creds on SQL01 |
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

## Attack Surface

| Vulnerability | Type | Details |
|--------------|------|---------|
| AS-REP Roast | Credential | mscott (Password1), kmalone (Kevin1!) |
| Kerberoast | Credential | svc-sql (SQLSvc2026!) — MSSQLSvc/DM-SQL01:1433 |
| GPP cpassword | Credential | Two GPOs: Corporate-ServiceAccount-Deploy (svc-backup), Scranton-LocalAdmin-Deploy (svc-deploy) |
| ADCS ESC1–16 | Privilege Escalation | Full Certified Pre-Owned attack surface via [`badsectorlabs.ludus_adcs`](https://github.com/badsectorlabs/ludus_adcs) with Office-themed users and realistic template names |
| ADCS ESC8 | Privilege Escalation | HTTP-based web enrollment on SC-DC01 — NTLM relay target |
| Unconstrained Delegation | Privilege Escalation | DM-SQL01 — coerce + relay for TGT capture |
| Excessive Privileges | Privilege Escalation | cbratton in Backup Operators + Server Operators + WriteDacl on Domain Admins |
| Cached DA Credentials | Credential | dwallace (DA) logged into DM-SQL01 — LSASS dump recovers DA hash |
| Password in Description | Credential | kmalone LDAP description contains password |
| Credentials in Shares | Credential | \\SC-DC01\IT\backup_sql.ps1 (SQL SA password), \\SC-DC01\HR\salary_review.txt (dwallace password) |
| Local Admin | Lateral Movement | mscott is local admin on SC-WS01 and SC-WS02 |
| Domain Admin (parent) | Privilege Escalation | dwallace is DA in dundermifflin.local — cross-forest escalation via trust |
| Weak Passwords | Credential | Multiple sprayable passwords (Password1, Pretzel1, BigTuna01, etc.) |

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

- **Parent DC** (DM-DC01): Promoted by Ludus natively via the `domain` config block. After deployment, simulates a DA logon to SQL01 to leave cached credentials in LSASS.
- **Child DC** (SC-DC01): Promoted by the `dundermifflin_child_dc` Ansible role using `Install-ADDSDomain -DomainType ChildDomain`. Skips promotion on re-runs (checks if domain exists first).
- **SQL Server** (DM-SQL01): Domain-joined by Ludus natively, post-config by `dundermifflin_member` role. Contains cached DA credentials from the simulated logon.
- **Workstations**: Domain-joined to scranton by the `dundermifflin_workstation` role using parent domain credentials.
- **ADCS**: Installed by [`badsectorlabs.ludus_adcs`](https://github.com/badsectorlabs/ludus_adcs) on SC-DC01 with `role_vars` overrides (child DC has no native Ludus domain block). ESC1 through ESC16 vulnerable templates with Office-themed users and realistic template names.

## Troubleshooting

- **Server 2022 template**: Do NOT use — Edge Appx blocks sysprep, causing SID duplication. Use `win2019-server-x64-template` instead.
- **Debian 11 router**: Stale apt repos. The config overrides to `debian-12-x64-server-template`.
- **WinRM timeouts**: The R610 (or similar constrained hardware) may struggle with 6 VMs sysprepping simultaneously. Ludus handles retries; a resume deploy usually completes what the first pass missed.
- **ADCS on child DC**: The `badsectorlabs.ludus_adcs` role handles Enterprise Admins via `ludus_adcs_domain_username` (set to `dundermifflin\domainadmin`). If ADCS fails with `ERROR_DS_RANGE_CONSTRAINT`, verify the domain admin account has Enterprise Admins membership.
- **Workstation domain join**: Uses parent domain creds (`dundermifflin.local\domainadmin`) to join the child domain — the child domain doesn't have its own `domainadmin` account.
- **ludus_adcs idempotency**: The upstream role is not fully idempotent on re-runs. A [UPN fix PR](https://github.com/badsectorlabs/ludus_adcs/pull/12) addresses user creation failures on repeated deploys.

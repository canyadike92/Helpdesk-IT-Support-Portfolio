# Project 2: Help Desk Ticketing System (osTicket)

**Focus:** Tier 1 Service Desk
**Lab:** helpdesk.lab (Windows Server 2022 Active Directory domain, built in Project 1)

## Overview

I deployed osTicket, an open source help desk ticketing system, on a domain-joined Windows Server 2022 web server. I configured it like a real service desk, with departments, SLA plans, priorities, help topics, and role-based agent accounts. I then worked 12 simulated support tickets from submission to closure, including real fixes in Active Directory, DNS, and file share permissions.

## Skills Demonstrated

- Windows Server 2022 deployment and domain join
- IIS web server configuration (FastCGI, handler mappings, URL Rewrite)
- PHP 8.4 and MySQL 8.4 LTS installation and configuration
- Service desk design: departments, SLA plans, priorities, help topics, agent roles
- Ticket lifecycle management: triage, assignment, internal notes, user communication, closure
- Active Directory administration with PowerShell (unlock, password reset, new user, group membership)
- DNS troubleshooting and record management
- SMB file share and NTFS permission management
- Least-privilege access and post-install security hardening
- Troubleshooting from logs and error messages

## Lab Architecture

| Host | Role | OS | IP |
|---|---|---|---|
| DC01 | Domain controller, DNS | Windows Server 2022 | 10.0.2.10 (static) |
| WEB01 | Web server: IIS, PHP, MySQL, osTicket | Windows Server 2022 | 10.0.2.20 (static) |
| CLIENT01 | End user workstation | Windows 11 Enterprise | DHCP |

- **Network:** VirtualBox NAT Network "HelpdeskLab", 10.0.2.0/24
- **Domain:** helpdesk.lab
- **Portal URL:** http://web01.helpdesk.lab/osticket/

## Software Stack

| Component | Version |
|---|---|
| osTicket | v1.18.4 |
| PHP | 8.4.26 NTS x64 (FastCGI) |
| MySQL | 8.4.11 LTS |
| IIS | 10 (with CGI and URL Rewrite 2.1) |
| Visual C++ Redistributable | v14 x64 (required by PHP) |

**Why MySQL 8.4 instead of the newer 9.7 LTS:** osTicket requires MySQL 5.5 or newer, so both qualify. I chose 8.4 because it has a longer production track record with osTicket, more troubleshooting resources, and remains supported well beyond the life of this lab.

## Build Steps

### Phase 1: Server
1. Created WEB01 in VirtualBox (4 GB RAM, 2 vCPU, 60 GB disk, BIOS firmware).
2. Installed Windows Server 2022 (Desktop Experience).
3. Set a static IP (10.0.2.20) with DNS pointing to DC01 (10.0.2.10).
4. Verified domain controller discovery with an SRV record lookup, then joined WEB01 to helpdesk.lab.

### Phase 2: Web Stack and osTicket
5. Installed IIS with the CGI role service.
6. Installed the Visual C++ Redistributable and extracted PHP 8.4 to C:\PHP.
7. Created php.ini and enabled the extensions osTicket uses (mysqli, gd, intl, mbstring, fileinfo, gettext, zip, OPcache).
8. Mapped *.php to php-cgi.exe through FastCGI in IIS and verified with a phpinfo test page, then deleted the test page.
9. Installed MySQL 8.4 LTS and set the root password.
10. Created a dedicated `osticket` database and an `osticket_user` account with rights to that database only.
11. Installed the IIS URL Rewrite module (an osTicket prerequisite on IIS).
12. Deployed osTicket and ran the web installer.
13. Hardened the install: removed write access from ost-config.php and deleted the setup folder.

### Phase 3: Service Desk Configuration

**SLA Plans**

| Plan | Grace Period |
|---|---|
| Critical | 4 hours |
| High | 8 hours |
| Normal | 24 hours |
| Low | 72 hours |

**Departments**
- Service Desk (Tier 1)
- Desktop Support (Tier 2)
- Network & Infrastructure

**Help Topics**

| Help Topic | Department | Priority | SLA |
|---|---|---|---|
| Password Reset / Account Lockout | Service Desk (Tier 1) | High | High |
| New User Account Request | Service Desk (Tier 1) | Normal | Normal |
| Printer Issue | Service Desk (Tier 1) | Normal | Normal |
| Hardware Issue | Desktop Support (Tier 2) | Normal | Normal |
| Software Installation | Desktop Support (Tier 2) | Low | Low |
| Network / Connectivity | Network & Infrastructure | Emergency | Critical |

A locked-out user cannot work at all, so lockouts get a tight SLA. A network outage can affect many users at once, so it gets top priority and the 4-hour SLA. Software requests can usually wait, so they get 72 hours. I disabled the installer's default help topics so users only see topics that route to a real department and SLA.

**Agents**

| Agent | Department | Role |
|---|---|---|
| Jordan Lee | Service Desk (Tier 1) | Expanded Access |
| Maria Gomez | Service Desk (Tier 1) | Expanded Access |
| Sam Patel | Desktop Support (Tier 2), plus Network & Infrastructure | All Access |

### Phase 4: Client Access
14. Added a DNS A record for web01 on DC01 so CLIENT01 could reach the portal by name.
15. Updated the osTicket Helpdesk URL from localhost to web01.helpdesk.lab.

## Ticket Log

Requesters are the Active Directory users from Project 1. Tickets were submitted from CLIENT01 through the customer portal and worked by agents on WEB01.

| # | Ticket No. | Requester | Issue | Agent | Resolution | Type |
|---|---|---|---|---|---|---|
| 1 | 735071 | Linda Park | Account locked out | Jordan Lee | Unlocked account in AD with PowerShell | Real |
| 2 | | Grace Liu | Forgot password | Jordan Lee | Reset password in AD, forced change at next logon | Real |
| 3 | | Omar Haddad | New hire account | Maria Gomez | Created kshaw in HR OU, added to HR-Staff | Real |
| 4 | | Daniel Brooks | Printer offline | Jordan Lee | Restarted Print Spooler on CLIENT01 | Real |
| 5 | | Priya Nair | Printer added | Maria Gomez | Confirmed no network printer with Get-Printer; added to print server rollout | Real check |
| 6 | | Grace Liu | Laptop keys not working | Sam Patel | Hardware fault; loaner keyboard issued, warranty replacement ordered | Simulated |
| 7 | | Marcus Reed | Monitor flickering | Sam Patel | Replaced faulty display cable | Simulated |
| 8 | | Priya Nair | 7-Zip install | Sam Patel | Installed 7-Zip on CLIENT01 | Real |
| 9 | | Daniel Brooks | Excel crashes on launch | Sam Patel | Isolated faulty add-in with safe mode, disabled it | Simulated |
| 10 | | Linda Park | No internet | Sam Patel | Reproduced wrong DNS server on CLIENT01, reset to DC01, verified resolution | Real |
| 11 | | Omar Haddad | Can't reach HR share | Sam Patel | Created HR share on DC01 with HR-Staff share and NTFS permissions | Real |
| 12 | | Daniel Brooks | Printer still not working | Jordan Lee | Closed as duplicate of ticket #4 | Real |

"Simulated" tickets involve physical hardware or software not present in the lab. They demonstrate the ticket workflow and documentation, not a hands-on fix.

## Troubleshooting Log

| Problem | Cause | Fix |
|---|---|---|
| nslookup showed "DNS request timed out" before answering | No reverse lookup zone for DC01's IP | Not an error; forward lookup succeeded |
| Domain join failed: AD DC "could not be contacted" | DNS SRV lookup timed out | Confirmed SRV record with nslookup, retried join successfully |
| WEB01 froze during IIS install | Undetermined | Reset VM; Get-WindowsFeature showed the install rolled back cleanly; reinstalled |
| MySQL service would not start after a VM freeze | ibdata1 corrupted (error log: header page of zero bytes) | Uninstalled, deleted the data folder, reinstalled |
| MySQL Configurator reported failure on "Starting the server" | Configurator gave up while MySQL was still starting; error log showed "ready for connections" | Confirmed service was running, set root password manually |
| osTicket installer: access denied for osticket_user | Wrong database password entered in the form | Tested the login in PowerShell, retried installer |
| Agent "Set Password" pop-up loaded blank | Pop-up content failed to load | Refreshed the page and retried |
| Agent could not log in | Password not saved on the agent account | Reset the agent password hash directly in the database |
| CLIENT01 could not resolve web01.helpdesk.lab | No DNS A record for WEB01 | Added the A record on DC01, flushed DNS cache on CLIENT01 |

## Security Practices

- osTicket connects to MySQL with a dedicated account limited to its own database, never as root.
- Agents receive role-based access by tier (least privilege).
- ost-config.php write access removed and the setup folder deleted after install, per osTicket guidance.
- The phpinfo test page was removed after testing, since it exposes server details.
- The admin account does not use the name "admin".
- Passwords are never sent in ticket replies; temporary passwords are delivered out of band with forced change at next logon.
- Lab only: the three agents share one password and MySQL's Windows Firewall port was left open. In production, each agent would have a unique password, and the database port would stay closed when the app and database share a server.

## Screenshots

Screenshots are in the `screenshots` folder and named `p2-##-description.png`, in build order.

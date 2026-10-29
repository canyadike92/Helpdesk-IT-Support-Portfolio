# Project 2: Building a Help Desk Ticketing System with osTicket

In Project 1, I built a small Active Directory domain called helpdesk.lab. For this project, I wanted to give that domain a working help desk: somewhere users could report problems, and somewhere agents could pick those problems up, fix them, and close them out.

I installed osTicket, an open source ticketing system, on a new Windows Server 2022 machine, set it up the way a real service desk would run, and then worked through 12 support tickets from start to finish. Several of those tickets involved real fixes in Active Directory, DNS, and file share permissions, not just clicking "close."

## What I Built

The lab has three machines on an isolated VirtualBox network (10.0.2.0/24):

| Machine | What it does | IP |
|---|---|---|
| DC01 | Domain controller and DNS (from Project 1) | 10.0.2.10 |
| WEB01 | New for this project: runs IIS, PHP, MySQL, and osTicket | 10.0.2.20 |
| CLIENT01 | Windows 11 workstation where "users" submit tickets | DHCP |

Users open tickets at http://web01.helpdesk.lab/osticket/, and agents work them from the staff panel on the same server.

**The software stack:** osTicket v1.18.4, PHP 8.4.26, MySQL 8.4.11 LTS, and IIS 10 with the URL Rewrite module.

One decision worth explaining: MySQL 9.7 is also an LTS release and is newer than 8.4. I went with 8.4 anyway. osTicket only needs MySQL 5.5 or newer, so both would work, but 8.4 has been around longer, most osTicket troubleshooting guides are written against it, and it is still supported for years. For a system people depend on, I would rather pick the version with the longer track record.

## How I Set It Up

### Getting the server ready

I created WEB01, installed Windows Server 2022, gave it a static IP, and pointed its DNS at DC01. Before joining it to the domain, I checked that it could actually find the domain controller by looking up the SRV record that domain joins rely on. Then I joined it to helpdesk.lab.

### Building the web stack

osTicket is a PHP application, so on Windows it needs IIS, PHP, and a database working together:

- Installed IIS with the CGI feature, which lets IIS hand PHP files off to PHP
- Installed PHP 8.4 and turned on the extensions osTicket needs
- Connected PHP to IIS through FastCGI, then confirmed it worked with a test page (and deleted that page afterward, since it exposes server details)
- Installed MySQL 8.4 and created a database just for osTicket, with its own account that can only touch that one database. osTicket never connects as root.
- Installed the IIS URL Rewrite module, which osTicket requires on IIS

Then I ran the osTicket installer. Once it finished, I locked down the configuration file so the web server can no longer write to it, and deleted the setup folder so nobody can rerun the installer. Both steps come straight from osTicket's own guidance.

### Designing the service desk

This was the part I enjoyed most, because it is less about software and more about how a support team actually works.

**Departments.** I set up three, to mirror a typical support structure:
- Service Desk (Tier 1): first point of contact for everything
- Desktop Support (Tier 2): hardware and software escalations
- Network & Infrastructure: connectivity and server issues

**SLA plans.** These set how long a ticket can sit before it's considered overdue:

| Plan | Time to resolve |
|---|---|
| Critical | 4 hours |
| High | 8 hours |
| Normal | 24 hours |
| Low | 72 hours |

**Help topics.** These are what users pick when they submit a ticket, and each one decides where the ticket goes and how urgent it is:

| Help topic | Goes to | Priority | SLA |
|---|---|---|---|
| Password Reset / Account Lockout | Service Desk | High | High |
| New User Account Request | Service Desk | Normal | Normal |
| Printer Issue | Service Desk | Normal | Normal |
| Hardware Issue | Desktop Support | Normal | Normal |
| Software Installation | Desktop Support | Low | Low |
| Network / Connectivity | Network & Infrastructure | Emergency | Critical |

My reasoning: someone who is locked out can't do any work, so that gets a tight deadline. A network problem can stop a whole team at once, so it gets the highest priority. A software install request can usually wait a few days. I also turned off the default help topics osTicket ships with, so users only see options that actually route somewhere.

**Agents.** I created three agent accounts and gave each one only the access their role needs:

| Agent | Team | Access level |
|---|---|---|
| Jordan Lee | Tier 1 | Expanded Access |
| Maria Gomez | Tier 1 | Expanded Access |
| Sam Patel | Tier 2 and Network | All Access |

### Letting users in

When I first tried opening the portal from CLIENT01, it couldn't find WEB01 at all. The problem was DNS: there was no record telling the network which IP belonged to "web01." I added that record on DC01, cleared the DNS cache on CLIENT01, and the portal loaded. I also updated osTicket's own URL setting so it stopped referring to itself as "localhost."

## Working the Tickets

With everything in place, I submitted 12 tickets from CLIENT01, using the six Active Directory users from Project 1 as the people reporting problems. Then I signed in as each agent and worked their queue: assigning tickets, fixing the issue, writing internal notes for other techs, replying to the user in plain language, and closing the ticket.

| # | Ticket No. | Reported by | Problem | Worked by | What I did | Hands-on? |
|---|---|---|---|---|---|---|
| 1 | 735071 | Linda Park | Locked out of account | Jordan Lee | Unlocked her account in AD with PowerShell | Yes |
| 2 | | Grace Liu | Forgot password | Jordan Lee | Reset her password in AD and required a new one at next sign-in | Yes |
| 3 | | Omar Haddad | New hire needs an account | Maria Gomez | Created Kevin Shaw's account in the HR OU and added him to HR-Staff | Yes |
| 4 | | Daniel Brooks | Printer shows offline | Jordan Lee | Restarted the Print Spooler service on his PC | Yes |
| 5 | | Priya Nair | Needs a printer added | Maria Gomez | Checked installed printers with Get-Printer, confirmed the Finance printer isn't on the network yet, added her to the rollout list | Partly |
| 6 | | Grace Liu | Laptop keys not working | Sam Patel | Diagnosed a hardware fault, issued a loaner keyboard, ordered a warranty replacement | Simulated |
| 7 | | Marcus Reed | Monitor flickering | Sam Patel | Replaced a faulty display cable | Simulated |
| 8 | | Priya Nair | Needs 7-Zip installed | Sam Patel | Installed 7-Zip on CLIENT01 | Yes |
| 9 | | Daniel Brooks | Excel crashes on launch | Sam Patel | Opened Excel in safe mode to isolate a bad add-in, then disabled it | Simulated |
| 10 | | Linda Park | No internet | Sam Patel | Recreated a bad DNS setting on CLIENT01, traced it, and pointed DNS back to DC01 | Yes |
| 11 | | Omar Haddad | Can't open the HR shared folder | Sam Patel | Set up the HR share on DC01 with access for the HR group only | Yes |
| 12 | | Daniel Brooks | Printer still not working | Jordan Lee | Recognized it as a duplicate of ticket #4 and closed it with a note | Yes |

A few of these are marked "simulated" because my lab doesn't have a physical laptop, a second monitor, or Microsoft Office. For those, I worked the ticket the way I would in a real job (notes, reply, closure), but the fix itself is described rather than performed. I'd rather be upfront about that than pretend otherwise.

A couple of habits I followed on every ticket:
- **Passwords never go in a ticket.** For the reset and the new hire, the reply says the temporary password will be shared in person, and the user has to change it at first sign-in.
- **Internal notes explain the fix for the next tech.** The user gets a short, friendly reply. The technical details go in a note only staff can see.

## What Went Wrong (and How I Fixed It)

Honestly, this was the most useful part of the project. Not everything worked the first time:

**The domain join failed.** Windows said it couldn't contact a domain controller. I looked up the SRV record directly, got a clean answer on the second try, and the join went through.

**WEB01 froze completely during the IIS install.** I reset the VM and checked the feature status with PowerShell before doing anything else. Windows had rolled the install back cleanly, so I just ran it again. That freeze taught me to snapshot before every big install.

**MySQL wouldn't start after another freeze.** The error log showed that a core data file had been corrupted, with the first page of the file completely empty. Since the database didn't have any data in it yet, the fastest fix was to uninstall, delete the data folder, and reinstall.

**The MySQL setup tool said it failed, but it hadn't.** The log showed the server was actually up and "ready for connections." The setup tool had just given up waiting before MySQL finished starting, which meant it never applied my root password. I confirmed the service was running and set the password myself.

**The osTicket installer couldn't log in to the database.** I tested the database account directly in PowerShell, found the password I'd typed into the form was wrong, and reran the installer.

**An agent couldn't sign in.** The password pop-up hadn't saved properly. I fixed it by updating the agent's password directly in the database.

**CLIENT01 couldn't find the portal.** Covered above: a missing DNS record on DC01.

## Security Choices

- osTicket uses its own database account, limited to its own database
- Each agent only gets the access their tier needs
- Config file locked and installer removed after setup
- Test pages deleted once they'd done their job
- The admin account isn't named "admin," since that's the first name attackers try
- No passwords in tickets, ever

A couple of shortcuts I took because this is a lab, and wouldn't take in production: the three agents share one password, and the database port was left open in Windows Firewall. In a real environment, every agent would have their own password, and the database port would stay closed since the app and database live on the same server.

## What I'd Add Next

- Show an SLA breach by letting a ticket run past its deadline
- Reopen a closed ticket when a user reports the problem came back
- Connect osTicket to email so users can open tickets by sending a message

## Screenshots

All screenshots are in the `screenshots` folder, named `p2-##-description.png` in the order I took them.

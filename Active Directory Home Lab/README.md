# Project 1: Active Directory Home Lab

## Why I Built This
Almost every help desk ticket touches Active Directory in some way: a locked
account, a forgotten password, a new hire who needs access, a laptop that
won't join the domain. I wanted hands-on experience with all of that, so I
built a small company network from scratch: one domain controller, one
Windows 11 workstation, and a set of users and groups spread across three
departments.

## What's in the Lab

| VM | Operating System | Role | IP Address |
|---|---|---|---|
| DC01 | Windows Server 2022 Standard Evaluation (Desktop Experience) | Domain Controller, DNS | 10.0.2.10 (static) |
| CLIENT01 | Windows 11 Enterprise Evaluation | Domain workstation | DHCP (10.0.2.100 to .200) |

Everything runs in Oracle VirtualBox 7.2.10 on a NAT Network I called
**HelpdeskLab** (10.0.2.0/24, gateway 10.0.2.1). The domain is
**helpdesk.lab**, and its short name is **HELPDESK**.

I organized Active Directory the way a small company might:
- **_ADMINS** holds my admin account (a-angel), which I keep separate from everyday use
- **_USERS** has an OU for each department (IT, HR, and Finance) with two users each
- **_GROUPS** has a security group for each department: IT-Staff, HR-Staff, and Finance-Staff
- **_COMPUTERS** is where domain-joined workstations live

## How I Built It

### Setting up the network
First, I created a dedicated NAT Network so this lab wouldn't interfere with
my other projects.

![NAT Network](../screenshots/p1-01-nat-network-helpdesklab.png)

The DHCP server was handing out nearly every address in the subnet, which
left no safe spot for a server with a fixed IP. I narrowed the range to
10.0.2.100 through 10.0.2.200 to free up room.

![DHCP range](../screenshots/p1-06-helpdesklab-dhcp-range.png)

### Building the domain controller
I installed Windows Server 2022 with the full desktop, renamed the machine
DC01, and checked that it activated.
![Activation](../screenshots/p1-07-dc01-activation.png)

Next, I gave it a static IP (10.0.2.10) and pointed its DNS at itself, since
a domain controller needs an address that never changes.

![Static IP](../screenshots/p1-08-dc01-static-ip.png)

Then I installed Active Directory Domain Services and promoted DC01 to a
domain controller in a brand-new forest called helpdesk.lab. DNS was
installed automatically as part of that step.

![AD DS role](../screenshots/p1-09-adds-role-selected.png)
![New forest](../screenshots/p1-11-promote-new-forest.png)
![Prerequisites](../screenshots/p1-12-prereq-check-passed.png)
![AD DS and DNS](../screenshots/p1-13-server-manager-adds-dns.png)

### Creating users and groups
With the domain up, I built out the OU structure, created my admin account,
added six department users, and put each user into their department's group.

![OU structure](../screenshots/p1-14-ou-structure.png)
![Domain Admins](../screenshots/p1-15-admin-domain-admins.png)
![Users](../screenshots/p1-16-dept-users.png)
![Group members](../screenshots/p1-17-group-members.png)

Before moving on, I signed in with my admin account and made sure DNS could
resolve both the domain and outside websites.

![whoami](../screenshots/p1-17b-admin-login-whoami.png)

### Setting up the workstation
CLIENT01 is a Windows 11 Enterprise machine on the same network.

![CLIENT01 settings](../screenshots/p1-20-client01-system-settings.png)
![Network adapter](../screenshots/p1-21-client01-network-adapter.png)

A computer can't join a domain it can't find, so I pointed CLIENT01's DNS at
DC01 and tested it with nslookup before going any further.

![DNS setting](../screenshots/p1-22a-client01-dns-setting.png)
![ipconfig](../screenshots/p1-22-client01-ipconfig.png)
![nslookup](../screenshots/p1-23-client01-nslookup.png)

### Joining the domain
The join went through on the first try.

![Welcome](../screenshots/p1-24-domain-join-welcome.png)

I logged in as a regular domain user, mreed. Running `whoami` and
`echo %logonserver%` confirmed that DC01 was the one handling the login.

![Verified](../screenshots/p1-25-client01-domain-user-verified.png)

Finally, I moved CLIENT01 into the _COMPUTERS OU so it sits with the other
workstations instead of the default container.

![Computers OU](../screenshots/p1-26-client01-in-computers-ou.png)

### Saving my progress
I took snapshots of both VMs so I can roll back to a clean, working state
if a later project breaks something.

![DC01 snapshot](../screenshots/p1-18-dc01-snapshot.png)
![CLIENT01 snapshot](../screenshots/p1-27-client01-snapshot.png)

## What Went Wrong (and How I Fixed It)
This part taught me the most.

**No room for a static IP.** My first look at the DHCP settings showed the
server handing out almost every address in the subnet. Rather than guess at
an IP and risk a conflict, I used `VBoxManage dhcpserver` to shrink the range
to .100 through .200, then gave DC01 10.0.2.10.

**DC01 froze mid-command.** The VM stopped responding while I was testing
DNS. I used VirtualBox's Machine > Reset to restart it. A hard reset is risky
on a domain controller, so once it came back up, I checked that all my OUs,
users, and groups were still there, then took a snapshot right away.

**CLIENT01 booted to a black screen.** No logo, no error, nothing. Instead of
changing settings at random, I compared CLIENT01 to DC01, which was working
fine, using `VBoxManage showvminfo`. The only firmware difference was that
CLIENT01 used EFI and DC01 used BIOS. The VirtualBox log also showed it was
running in NEM mode, because Memory Integrity is turned on for my laptop.
Switching CLIENT01 to BIOS fixed it.

**Windows 11 refused to install.** Without UEFI, Setup said the PC needed
TPM 2.0 and Secure Boot. I used the LabConfig registry bypass during setup
(BypassTPMCheck and BypassSecureBootCheck) to get past the check. This is
only appropriate for a lab: it skips hardware security features, and future
major Windows updates may check the requirements again.

![Requirements check](../screenshots/p1-20a-win11-requirements-check.png)
![LabConfig](../screenshots/p1-20b-labconfig-bypass.png)

**"Disk file name is not unique."** An earlier attempt at creating CLIENT01
had left its virtual disk behind. I removed it, and the new VM created
without issues.

**Setup wanted a work or school account.** Windows 11 Enterprise expects a
company account at sign-in. I used Sign-in options, then Domain join instead,
to create a local account, and joined the real domain later.

**CLIENT01 couldn't find the domain.** `ipconfig /all` showed it was using
192.168.0.1 for DNS, which has no idea what helpdesk.lab is. I pointed its
DNS at DC01 (10.0.2.10), confirmed with nslookup, and the domain join worked.

## What I Practiced
Active Directory Domain Services, DNS, DHCP scoping, static IP configuration,
joining computers to a domain, OU design, user and group management,
VirtualBox administration with VBoxManage, and troubleshooting by comparing
a working system against a broken one.

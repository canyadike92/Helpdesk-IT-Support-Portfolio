# Project 1: Active Directory Home Lab

## Why
Help desk and desktop support technicians work inside Active Directory every day:
creating accounts, resetting passwords, joining computers to the domain, and
troubleshooting logins. I built this lab to practice those tasks in a small
company domain that I designed, built, and troubleshot from scratch.

## Lab Architecture

| VM | Operating System | Role | IP Address |
|---|---|---|---|
| DC01 | Windows Server 2022 Standard Evaluation (Desktop Experience) | Domain Controller, DNS | 10.0.2.10 (static) |
| CLIENT01 | Windows 11 Enterprise Evaluation | Domain workstation | DHCP (10.0.2.100 to .200) |

| Setting | Value |
|---|---|
| Hypervisor | Oracle VirtualBox 7.2.10 |
| Network | VirtualBox NAT Network "HelpdeskLab" (10.0.2.0/24, gateway 10.0.2.1) |
| DHCP range | 10.0.2.100 to 10.0.2.200 (narrowed to reserve addresses for servers) |
| Domain | helpdesk.lab (NetBIOS: HELPDESK) |

### Active Directory Structure
- `_ADMINS`: separate admin account (a-angel) in Domain Admins
- `_USERS`: department OUs for IT, HR, and Finance, with 2 users each
- `_GROUPS`: IT-Staff, HR-Staff, Finance-Staff (Global Security groups)
- `_COMPUTERS`: domain-joined workstations

## How I Built It

### 1. Network
Created a dedicated NAT Network so the lab is isolated from my other labs.

![NAT Network](../screenshots/p1-01-nat-network-helpdesklab.png)

Narrowed the DHCP range to 10.0.2.100 to 10.0.2.200 so the domain controller's
static IP could not conflict with DHCP.

![DHCP range](../screenshots/p1-06-helpdesklab-dhcp-range.png)

### 2. Domain Controller (DC01)
Installed Windows Server 2022 with Desktop Experience, renamed it DC01,
and confirmed activation.
![Activation](../screenshots/p1-07-dc01-activation.png)

Assigned a static IP and pointed DNS to itself.

![Static IP](../screenshots/p1-08-dc01-static-ip.png)

Installed Active Directory Domain Services and promoted DC01 to a domain
controller in a new forest, helpdesk.lab.

![AD DS role](../screenshots/p1-09-adds-role-selected.png)
![New forest](../screenshots/p1-11-promote-new-forest.png)
![Prerequisites](../screenshots/p1-12-prereq-check-passed.png)
![AD DS and DNS](../screenshots/p1-13-server-manager-adds-dns.png)

### 3. AD Structure, Users, and Groups

![OU structure](../screenshots/p1-14-ou-structure.png)
![Domain Admins](../screenshots/p1-15-admin-domain-admins.png)
![Users](../screenshots/p1-16-dept-users.png)
![Group members](../screenshots/p1-17-group-members.png)

Verified my admin login and DNS resolution for both the domain and the internet.

![whoami](../screenshots/p1-17b-admin-login-whoami.png)

### 4. Client Workstation (CLIENT01)

![CLIENT01 settings](../screenshots/p1-20-client01-system-settings.png)
![Network adapter](../screenshots/p1-21-client01-network-adapter.png)

Pointed CLIENT01's DNS to the domain controller and confirmed it could resolve
the domain before joining.

![DNS setting](../screenshots/p1-22a-client01-dns-setting.png)
![ipconfig](../screenshots/p1-22-client01-ipconfig.png)
![nslookup](../screenshots/p1-23-client01-nslookup.png)

### 5. Domain Join and Verification

![Welcome](../screenshots/p1-24-domain-join-welcome.png)

Logged in as domain user mreed. `whoami` and `%logonserver%` confirm that
DC01 authenticated the login.

![Verified](../screenshots/p1-25-client01-domain-user-verified.png)

Moved CLIENT01 into the `_COMPUTERS` OU.

![Computers OU](../screenshots/p1-26-client01-in-computers-ou.png)

### 6. Snapshots
Took snapshots of both VMs as clean restore points for future projects.

![DC01 snapshot](../screenshots/p1-18-dc01-snapshot.png)
![CLIENT01 snapshot](../screenshots/p1-27-client01-snapshot.png)

## Problems I Hit and How I Fixed Them

**1. The DHCP range covered almost the whole subnet**
- Problem: My other NAT Network's DHCP handed out .3 to .254, leaving no safe
  address for a static server IP.
- Fix: Used `VBoxManage dhcpserver` to set HelpdeskLab's range to .100 to .200,
  then gave DC01 the static IP 10.0.2.10.

**2. DC01 froze while I was running commands**
- Fix: Used VirtualBox's Machine > Reset (a hard restart) after the VM stopped
  responding. Afterward, I confirmed my OUs, users, and groups were intact and
  took a snapshot right away.

**3. CLIENT01 showed only a black screen at startup**
- Diagnosis: Compared CLIENT01's settings with DC01, which worked, using
  `VBoxManage showvminfo`. The only firmware difference was EFI on CLIENT01
  versus BIOS on DC01. The VirtualBox log showed it was running in NEM mode
  because Memory Integrity was enabled on my host.
- Fix: Switched CLIENT01 to BIOS firmware.

**4. Windows 11 Setup blocked the install**
- Problem: Without UEFI, Setup reported that the PC must support TPM 2.0 and
  Secure Boot.
- Fix: Used the LabConfig registry bypass (BypassTPMCheck and
  BypassSecureBootCheck) during setup.
- Trade-off: This is for a lab only. It skips hardware-backed security
  features, and future major Windows updates may recheck hardware requirements.

![Requirements check](../screenshots/p1-20a-win11-requirements-check.png)
![LabConfig](../screenshots/p1-20b-labconfig-bypass.png)

**5. "Disk file name is not unique" when creating CLIENT01**
- Fix: A leftover virtual disk from an earlier attempt was still registered.
  Removed it before recreating the VM.

**6. Windows 11 Enterprise required a work or school account**
- Fix: Chose Sign-in options, then Domain join instead, to create a local
  account. The machine was joined to the domain afterward.

**7. CLIENT01 could not find the domain**
- Problem: `ipconfig /all` showed DNS server 192.168.0.1, which knows nothing
  about helpdesk.lab.
- Fix: Set CLIENT01's preferred DNS server to DC01 (10.0.2.10), then confirmed
  with `nslookup helpdesk.lab` before joining.

## Skills Demonstrated
Active Directory Domain Services, DNS, DHCP scoping, static IP configuration,
domain join, OU design, user and group management, VirtualBox administration
(VBoxManage), systematic troubleshooting by comparing working and broken systems

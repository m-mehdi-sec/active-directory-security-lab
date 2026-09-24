# Active Directory Security Lab – Technical Documentation

This document contains the detailed implementation record, verification procedures, commands, troubleshooting notes, and observed results from the Active Directory Security Lab.

[← Back to project overview](../README.md)

The purpose of this file is to provide technical evidence of the work performed in the lab. It intentionally goes deeper than the repository README and focuses on configuration details, validation, problems encountered, and the methods used to troubleshoot them.

---

## 1. Environment Record

The lab was built using Microsoft Hyper-V with one Windows Server 2025 Domain Controller and one Windows 11 domain client.

| System | Hostname | IP Address | Role |
|---|---|---:|---|
| Windows Server 2025 | `WIN-SRV2025.lab.local` | `192.168.10.10` | Domain Controller, AD DS, DNS |
| Windows 11 | `W11-CLIENT.lab.local` | `192.168.10.20` | Domain client |

Additional network information:

```text
Domain:        lab.local
NetBIOS:       LAB
Network:       192.168.10.0/24
Gateway:       192.168.10.1
Domain DNS:    192.168.10.10
```

The environment currently contains a single Domain Controller.

---

# 2. Initial Domain Controller Verification

Before making additional Active Directory changes, the Domain Controller and its core services were verified.

## 2.1 AD DS, DNS and Netlogon Services

PowerShell was opened with administrative privileges.

```powershell
Get-Service NTDS,DNS,Netlogon
```

The command was used to verify the status of:

```text
NTDS       Active Directory Domain Services
DNS        DNS Server
Netlogon   Netlogon
```

The services were available and running during the initial verification.

---

## 2.2 Domain Information

The Active Directory domain configuration was queried using:

```powershell
Get-ADDomain
```

The environment was confirmed as:

```text
DNSRoot:       lab.local
NetBIOSName:   LAB
```

The command also provides information about the current domain configuration and domain controllers.

---

## 2.3 Forest Information

The forest was queried using:

```powershell
Get-ADForest
```

The lab contains one Active Directory forest with:

```text
Forest: lab.local
Domain: lab.local
```

No additional domains exist in the forest.

---

# 3. DNS Validation

DNS was treated as a core dependency before domain client configuration and Group Policy testing.

Active Directory depends on DNS for Domain Controller discovery and service location.

## 3.1 DNS Server Configuration

The Domain Controller hosts the DNS role for:

```text
lab.local
```

The Windows 11 client was configured to use:

```text
192.168.10.10
```

as its DNS server.

Using an external resolver such as `8.8.8.8` directly on the domain client would prevent normal discovery of the internal Active Directory DNS records.

---

## 3.2 DNS Client Configuration Check

DNS settings were inspected with:

```powershell
Get-DnsClientServerAddress |
Select-Object InterfaceAlias,AddressFamily,ServerAddresses
```

A more targeted check can be performed with:

```powershell
Get-DnsClientServerAddress -InterfaceAlias "Ethernet"
```

During DNS configuration work, the IPv4 DNS server was explicitly configured where required:

```powershell
Set-DnsClientServerAddress `
  -InterfaceAlias "Ethernet" `
  -ServerAddresses 192.168.10.10
```

The result was verified again using:

```powershell
Get-DnsClientServerAddress -InterfaceAlias "Ethernet"
```

---

## 3.3 Forward DNS Resolution

The domain was tested using:

```powershell
Resolve-DnsName lab.local
```

and:

```cmd
nslookup lab.local
```

The domain resolved to:

```text
192.168.10.10
```

A test performed on the Domain Controller also returned:

```text
Server:  UnKnown
Address: ::1

Name:    lab.local
Address: 192.168.10.10
```

The important part of this result was that `lab.local` resolved correctly to the Domain Controller.

The `Server: UnKnown` output did not mean that forward DNS resolution had failed. The DNS query itself completed successfully.

---

## 3.4 Active Directory SRV Records

Active Directory service discovery was checked using SRV records.

```powershell
Resolve-DnsName `
  -Type SRV `
  _ldap._tcp.dc._msdcs.lab.local
```

The DNS zone was also inspected through DNS Manager for Active Directory-related records including:

```text
_ldap
_kerberos
_msdcs
```

These records allow domain clients to locate Active Directory services.

---

## 3.5 Domain Controller Discovery

Domain Controller discovery was additionally verified using:

```cmd
nltest /dsgetdc:lab.local
```

The command successfully located the Domain Controller.

This confirmed that the domain could be discovered using the normal Active Directory locator mechanism.

---

# 4. Directory Structure Implementation

Active Directory Users and Computers was used to create the organizational structure.

The final OU hierarchy was:

```text
lab.local
└── LAB
    ├── Users
    │   ├── IT
    │   ├── Finance
    │   └── Sales
    ├── Workstations
    ├── Servers
    ├── Groups
    ├── Admin
    └── Helpdesk-Test
```

The structure contains ten OUs when the top-level `LAB` OU is included.

The domain itself, `lab.local`, is not an OU.

The OU design provides separate policy and administration boundaries for:

- Standard users
- Departments
- Workstations
- Servers
- Security groups
- Administrative accounts
- Delegation testing

Important OUs were protected from accidental deletion where appropriate.

---

# 5. User Object Implementation

Users were created through Active Directory Users and Computers and placed in their appropriate departmental OUs.

## IT

```text
Erik Ek
```

## Finance

```text
Sara Svensson
Lisa Lind
Anna Andersson
```

## Sales

```text
Johan Jansson
Oskar Olsson
```

Anna Andersson was initially used in another OU and later moved to Finance as part of Group Policy scope testing.

Moving an object between OUs was important because OU placement affects which linked Group Policies can apply.

Group membership should not be assumed to change automatically when an account is moved between OUs.

---

# 6. Separate Administrative Identity

A separate administrative account was created:

```text
adm-anna.andersson
```

The account was placed under:

```text
LAB
└── Admin
```

The purpose was to separate normal user activity from administrative activity.

The standard account and administrative account therefore represent different security contexts.

This avoids using a highly privileged identity for ordinary desktop activity.

---

# 7. Security Group Implementation

The following security groups were created:

```text
GG-IT-Users
GG-Finance-Users
GG-Sales-Users
GG-Helpdesk
DL-Finance-Read
```

The naming convention identifies the intended group scope.

```text
GG = Global Group
DL = Domain Local Group
```

Known department membership included:

```text
GG-Finance-Users
├── Sara Svensson
└── Lisa Lind
```

and:

```text
GG-Sales-Users
├── Johan Jansson
└── Oskar Olsson
```

Anna Andersson was moved between OUs during policy testing. Her final departmental group membership was not separately revalidated as part of that test and is therefore not documented here as a confirmed final state.

---

# 8. AGDLP Implementation

A simple AGDLP structure was created.

```text
Accounts
   ↓
GG-Finance-Users
   ↓
DL-Finance-Read
   ↓
Permission
```

The implemented nesting was:

```text
GG-Finance-Users
        ↓
DL-Finance-Read
```

The purpose of the model is to separate:

```text
User membership
        ↓
Business role
        ↓
Resource permission group
        ↓
Actual permission
```

The Domain Local group was created and the Global group was nested inside it.

However, an actual NTFS or SMB file-share Read permission was not configured during this lab.

Therefore:

```text
GG-Finance-Users → DL-Finance-Read
```

was implemented, while:

```text
DL-Finance-Read → Production file share Read permission
```

remained a design example rather than an implemented control.

---

# 9. Windows 11 Domain Join

Before joining Windows 11 to the domain, the client network configuration was checked.

```cmd
ipconfig /all
```

The important configuration was:

```text
IP:       192.168.10.20
Gateway:  192.168.10.1
DNS:      192.168.10.10
```

DNS resolution was tested before attempting the domain join.

```powershell
Resolve-DnsName lab.local
```

The client was then joined to:

```text
lab.local
```

After the domain join, Windows was restarted.

---

## 9.1 Computer Object Placement

The computer account was initially visible in Active Directory and was moved to:

```text
LAB
└── Workstations
    └── W11-CLIENT
```

This step was necessary before testing the workstation-specific Group Policies.

A computer object located outside the intended OU would not receive policies linked specifically to `LAB\Workstations`.

---

## 9.2 Domain User Verification

A domain user was used to sign in to Windows 11.

The security context was verified with:

```cmd
whoami
```

A domain account returned a result in the form:

```text
LAB\username
```

rather than a local identity such as:

```text
W11-CLIENT\username
```

This confirmed that the session was using an Active Directory account.

---

# 10. Group Policy – Computer Policy Test

The first custom workstation GPO was:

```text
SEC-W11-Basic-Security
```

It was linked to:

```text
LAB
└── Workstations
```

The configured setting was:

```text
Computer Configuration
└── Policies
    └── Windows Settings
        └── Security Settings
            └── Local Policies
                └── Security Options
                    └── Interactive logon:
                       Do not display last signed-in
```

The setting was enabled.

---

## 10.1 Policy Refresh

The client was updated using:

```cmd
gpupdate /force
```

Group Policy processing was then inspected with:

```cmd
gpresult /r
```

The applied policy list was checked for:

```text
SEC-W11-Basic-Security
```

---

## 10.2 HTML Group Policy Report

A more detailed Group Policy report was generated:

```cmd
gpresult /h C:\gpresult.html /f
```

The HTML report was used to distinguish between:

```text
A GPO that exists in GPMC
```

and:

```text
A GPO that was actually processed by the client
```

This was an important troubleshooting and verification step throughout the lab.

---

# 11. Group Policy – User Policy Test

A separate user-based policy was created:

```text
CFG-Users-Basic
```

The configured setting was:

```text
User Configuration
└── Policies
    └── Administrative Templates
        └── Control Panel
            └── Prohibit access to Control Panel and PC settings
```

The policy was enabled.

It was initially linked higher in the user structure and was later targeted to the Finance OU to test scope.

---

## 11.1 Scope Test

Two users in different OUs were used.

```text
Anna Andersson → Finance
Erik Ek        → IT
```

The user policy was linked to:

```text
Finance
```

After policy refresh and sign-in testing:

```text
Anna → Settings blocked
Erik → Settings accessible
```

This verified that the policy followed the user object's OU scope.

The test also demonstrated that simply having a GPO in Group Policy Management does not cause it to affect every domain account.

---

# 12. Security Filtering Test

A separate test policy was created:

```text
SEC-Finance-Test
```

The policy was linked to:

```text
Finance
```

Security Filtering was configured using:

```text
GG-Finance-Users
```

The test was designed to investigate how GPO scope can be restricted further than OU placement alone.

The important permissions involved in GPO filtering are:

```text
Read
Apply Group Policy
```

No separate production security setting was configured inside `SEC-Finance-Test`.

For that reason, this GPO is documented as a filtering configuration test rather than as an implemented security control.

---

# 13. Group Policy Inheritance and Conflict Test

Two GPOs were created specifically to test precedence.

```text
TEST-Domain-Setting
TEST-OU-Setting
```

Both configured:

```text
Computer Configuration
└── Policies
    └── Administrative Templates
        └── System
            └── Display highly detailed status messages
```

The values were intentionally different.

### Domain-level GPO

```text
TEST-Domain-Setting
Setting: Disabled
Link: lab.local
```

### OU-level GPO

```text
TEST-OU-Setting
Setting: Enabled
Link: LAB\Workstations
```

---

## 13.1 Processing Model

The normal Group Policy processing order is:

```text
Local
  ↓
Site
  ↓
Domain
  ↓
OU
```

The test client was located in:

```text
LAB\Workstations
```

The OU policy therefore processed after the domain-level policy in the normal processing order.

---

## 13.2 Verification

The resulting client registry state was inspected.

The relevant value was:

```text
VerboseStatus : 1
```

A PowerShell check for the value can be performed with:

```powershell
Get-ItemProperty `
  -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" `
  -Name VerboseStatus
```

The resulting value matched the enabled OU policy.

This confirmed the effective result of the policy conflict on the test client.

A Resultant Set of Policy report was also generated during conflict testing:

```cmd
gpresult /h C:\conflict.html /f
```

The purpose was to verify the result rather than relying only on what appeared in GPMC.

---

# 14. Default Domain Policy Review

The Default Domain Policy was inspected but was not used as a general-purpose configuration policy.

The following password-related settings were observed during inspection:

```text
Password history:              24 passwords
Maximum password age:          42 days
Minimum password age:          1 day
Minimum password length:       7 characters
Password complexity:           Enabled
Reversible encryption:         Disabled
Account lockout threshold:     0
```

These values document what was inspected in the policy configuration.

The complete effective password policy was not independently validated as part of this specific check.

For that reason, the values are not presented as a complete proof of the final effective authentication policy.

---

# 15. Privileged Group Review

The following Active Directory groups were inspected:

```text
Domain Admins
Enterprise Admins
Schema Admins
```

At the time of inspection, only:

```text
Administrator
```

was observed as a member of these highly privileged groups.

No ordinary user account was intentionally added to them.

This provided a baseline before delegated Helpdesk permissions were tested.

---

# 16. Helpdesk Delegation Implementation

A Global Security Group was created:

```text
GG-Helpdesk
```

The test account:

```text
helpdesk.test
```

was added to this group.

Delegation was applied to:

```text
LAB\Users
```

using the Active Directory Delegation of Control Wizard.

The delegated tasks were limited to:

```text
Reset user passwords
Force password change at next logon
```

The account was not made a Domain Admin.

---

# 17. RSAT Administration from Windows 11

During delegation testing, it became clear that the Helpdesk test should not be performed by allowing the Helpdesk account to administer the Domain Controller interactively.

Instead, Active Directory administration tools were installed on the Windows 11 client.

PowerShell was opened as Administrator and RSAT was installed using:

```powershell
Add-WindowsCapability `
  -Online `
  -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0
```

The Active Directory administration tools could then be used from the domain-joined workstation.

This created a more realistic administration model:

```text
Helpdesk account
      ↓
Windows 11 admin workstation
      ↓
RSAT
      ↓
Active Directory
```

rather than:

```text
Helpdesk account
      ↓
Interactive logon to Domain Controller
```

---

# 18. Delegation Validation

Two separate actions were tested while operating as the Helpdesk user.

## Test 1 – Password Reset

The Helpdesk account attempted to reset Sara Svensson's password.

Result:

```text
SUCCESS
```

The delegated permission worked.

---

## Test 2 – Privileged Group Modification

The Helpdesk account attempted an operation requiring higher privileges by trying to modify Domain Admin membership.

Result:

```text
ACCESS DENIED
```

The Helpdesk account could therefore perform the delegated task but could not perform a domain-wide privileged administration task.

This provided practical evidence of least privilege.

---

# 19. Helpdesk Logon Troubleshooting

One issue encountered during the test was related to where and how the Helpdesk account should be used.

A normal Helpdesk identity was not intended to receive interactive administrative logon rights to the Domain Controller.

When direct use of the DC was restricted, this was not treated as a configuration failure.

Instead, administration was moved to the Windows 11 workstation using RSAT.

This preserved the intended security model.

---

## 19.1 Hyper-V Enhanced Session Issue

Another issue was related to Hyper-V session modes.

Enhanced Session Mode relies on Remote Desktop-related functionality and can require rights that a normal domain user does not have.

This can make a valid domain account appear unable to use the VM session even when domain authentication itself is working.

For testing normal domain users, Hyper-V Basic Session was used when necessary.

The important distinction was:

```text
Failed Enhanced Session access
≠
Failed Active Directory authentication
```

The issue was therefore not solved by granting unnecessary Remote Desktop or Domain Admin rights.

---

# 20. FSMO Role Verification

FSMO role placement was checked using:

```cmd
netdom query fsmo
```

All five FSMO roles were located on:

```text
WIN-SRV2025.lab.local
```

The roles were:

```text
Schema Master
Domain Naming Master
PDC Emulator
RID Master
Infrastructure Master
```

Because the environment contains only one Domain Controller, this role placement was expected.

---

# 21. Domain Controller Health Check

A Domain Controller diagnostic test was performed using:

```cmd
dcdiag
```

The output was reviewed instead of treating a completed command as proof of a healthy DC.

Several tests passed, including:

```text
Connectivity
SysVolCheck
NetLogons
Replications
Services
```

However, the following tests reported failures:

```text
Advertising
SystemLog
LocatorCheck
```

This required additional investigation.

---

# 22. Time Synchronization Problem

The most important issue found during `dcdiag` involved time service availability.

The diagnostic output reported conditions including:

```text
not advertising as time server
```

and:

```text
A Time Server could not be located
```

Because this Domain Controller also owns the PDC Emulator role, time configuration is particularly important.

The Windows Time service status was investigated with:

```cmd
w32tm /query /status
```

The result included:

```text
Leap Indicator: 3 (not synchronized)
Stratum: 1
Source: VM IC Time Synchronization Provider
```

The last successful synchronization timestamp was also displayed by the command.

The important finding was:

```text
Leap Indicator 3 = clock not synchronized
```

The server was using the Hyper-V integration time provider but did not report a properly synchronized state.

This issue was documented and remained unresolved at the end of this lab.

It should not be reported as fixed.

---

# 23. dcdiag System Log Findings

The `SystemLog` test also reported problems because Windows had recorded warnings and errors.

Examples observed during the broader investigation included events related to:

```text
DNS SRV lookup timeouts
Disk write cache
Secure Boot attestation
WinRM SPN registration
DCOM warnings
```

These events were not all current failures.

Some represented earlier system activity.

For that reason:

```text
Historical Event Viewer warning
≠
Current service failure
```

Each event needs to be evaluated based on its timestamp and current system state.

---

# 24. Replication Verification

Replication was checked using:

```cmd
repadmin /replsummary
```

The command displayed the Source and Destination sections without replication partners.

This initially looks unusual if the command is expected to display another server.

However, this lab contains only:

```text
1 Domain Controller
```

Therefore there is no second DC with which Active Directory can perform DC-to-DC replication.

The empty replication summary was therefore expected for this topology.

It should not be documented as proof that multi-DC replication is healthy.

A future environment with:

```text
DC01
DC02
```

would provide an actual replication relationship to verify.

---

# 25. SYSVOL and NETLOGON Verification

The Domain Controller shares were tested through:

```text
\\WIN-SRV2025\SYSVOL
```

and:

```text
\\WIN-SRV2025\NETLOGON
```

Both were accessible.

Inside SYSVOL, the domain structure included:

```text
lab.local
└── Policies
```

and the scripts structure was also available.

SYSVOL was inspected only.

No files were manually edited.

Manual modification of SYSVOL was intentionally avoided because Group Policy should be managed using the supported Group Policy management tools.

---

# 26. Event Viewer Review

The Domain Controller Event Viewer was reviewed across several logs:

```text
Directory Service
DNS Server
Security
System
DFS Replication
```

Examples observed included:

| Log | Event ID | Observation |
|---|---:|---|
| Directory Service | 3027 | Online defragmentation started |
| DNS Server | 4 | DNS zone loading completed |
| Security | 4688 | New process created |
| System | 7036 | Windows Error Reporting service stopped |
| DFS Replication | 1210 | RPC listener successfully started |

These events were used to practice distinguishing normal service activity from events that require investigation.

An event existing in Event Viewer does not automatically indicate a security incident.

Context, timestamp, source, and related events must also be reviewed.

---

# 27. Active Directory Audit Test

A temporary user account was created specifically to produce Active Directory audit events.

The account was:

```text
logg.test
```

The test workflow was:

```text
Create logg.test
        ↓
Add logg.test to GG-IT-Users
        ↓
Delete logg.test
        ↓
Inspect Security log
```

---

## 27.1 User Creation

The Security log contained:

```text
Event ID 4720
```

Meaning:

```text
A user account was created
```

---

## 27.2 Group Membership Change

After the account was added to `GG-IT-Users`, the log contained:

```text
Event ID 4728
```

Meaning:

```text
A member was added to a security-enabled global group
```

---

## 27.3 User Deletion

After the test account was removed, the Security log contained:

```text
Event ID 4726
```

Meaning:

```text
A user account was deleted
```

---

## 27.4 SID Correlation

The events were correlated using the same account SID:

```text
S-1-5-21-308717170-3868153215-1720923902-1117
```

The matching SID demonstrated that the creation, group membership change, and deletion events referred to the same temporary account.

The events also showed that the actions were performed by:

```text
LAB\Administrator
```

This demonstrated how Windows Security auditing can be used to reconstruct identity-related administrative activity.

---

# 28. Troubleshooting Summary

The following issues or potentially confusing results were encountered during the lab.

## 28.1 DNS Server Displayed as "UnKnown"

Observed:

```text
Server:  UnKnown
Address: ::1

Name:    lab.local
Address: 192.168.10.10
```

Interpretation:

The forward lookup succeeded.

The `UnKnown` server name did not mean that `lab.local` DNS resolution had failed.

---

## 28.2 GPO Created but Not Applied

Creating:

```text
SEC-W11-Basic-Security
```

inside Group Policy Objects did not automatically affect the Windows 11 client.

The GPO first needed to be:

```text
Created
   ↓
Configured
   ↓
Linked
   ↓
Within target scope
   ↓
Allowed by security filtering
   ↓
Processed by client
   ↓
Verified
```

Verification was performed with:

```cmd
gpupdate /force
gpresult /r
gpresult /h C:\gpresult.html /f
```

This became one of the key troubleshooting methods used throughout the lab.

---

## 28.3 User Policy Applied to Anna but Not Erik

This initially appears inconsistent if only the GPO name is considered.

The reason was OU placement.

```text
Anna → Finance
Erik  → IT
```

The policy was linked to Finance.

Result:

```text
Anna → affected
Erik  → not affected
```

The policy was therefore working according to its scope.

---

## 28.4 Conflicting GPO Values

Two policies deliberately configured the same setting differently.

The effective value was not determined by guessing from GPMC.

Instead, the final client state was verified.

```text
VerboseStatus = 1
```

This matched the OU-level policy.

The test demonstrated why effective configuration should be verified on the endpoint.

---

## 28.5 Helpdesk Could Not Perform Privileged Action

The Helpdesk account was able to reset a password but could not modify Domain Admin membership.

This was expected.

The denied action demonstrated that delegation was correctly more limited than Domain Admin privileges.

The solution was not to elevate the Helpdesk account.

---

## 28.6 Helpdesk Administration from the DC

Direct interactive use of the Domain Controller was not required for normal Helpdesk administration.

RSAT on Windows 11 was used instead.

This avoided granting unnecessary server logon rights.

---

## 28.7 Hyper-V Enhanced Session

Enhanced Session behavior caused additional access restrictions for ordinary domain users.

Basic Session was used where appropriate.

No unnecessary RDP or Domain Admin permissions were added simply to make the test session work.

---

## 28.8 `dcdiag` Was Not Fully Clean

The following failures required interpretation:

```text
Advertising
SystemLog
LocatorCheck
```

The main unresolved technical issue was time synchronization.

The Domain Controller reported:

```text
Leap Indicator: 3
```

and was not correctly advertising a usable time source.

This remains a known issue.

---

## 28.9 `repadmin /replsummary` Returned No Partners

This was expected because the environment contains only one Domain Controller.

No attempt was made to "fix" replication when no second replication partner existed.

---

# 29. Commands Used During Validation

The following command set represents the main command-line validation performed throughout the lab.

## Active Directory Services

```powershell
Get-Service NTDS,DNS,Netlogon
```

## Domain Information

```powershell
Get-ADDomain
```

## Forest Information

```powershell
Get-ADForest
```

## DNS Configuration

```powershell
Get-DnsClientServerAddress |
Select-Object InterfaceAlias,AddressFamily,ServerAddresses
```

```powershell
Get-DnsClientServerAddress -InterfaceAlias "Ethernet"
```

## DNS Resolution

```powershell
Resolve-DnsName lab.local
```

```powershell
Resolve-DnsName `
  -Type SRV `
  _ldap._tcp.dc._msdcs.lab.local
```

```cmd
nslookup lab.local
```

## Domain Controller Discovery

```cmd
nltest /dsgetdc:lab.local
```

## Windows Network Configuration

```cmd
ipconfig /all
```

## Logged-On Identity

```cmd
whoami
```

## Group Policy Refresh

```cmd
gpupdate /force
```

## Group Policy Result

```cmd
gpresult /r
```

```cmd
gpresult /h C:\gpresult.html /f
```

## Policy Conflict Report

```cmd
gpresult /h C:\conflict.html /f
```

## GPO Registry Verification

```powershell
Get-ItemProperty `
  -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" `
  -Name VerboseStatus
```

## RSAT Installation

```powershell
Add-WindowsCapability `
  -Online `
  -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0
```

## FSMO Roles

```cmd
netdom query fsmo
```

## Domain Controller Diagnostics

```cmd
dcdiag
```

## Active Directory Replication Summary

```cmd
repadmin /replsummary
```

## Windows Time Status

```cmd
w32tm /query /status
```

---

# 30. Verified Security Controls

The following controls were directly demonstrated in the lab.

| Control | Verification |
|---|---|
| AD DS availability | AD DS services checked |
| DNS resolution | Domain and AD SRV records resolved |
| Domain Join | Windows 11 joined `lab.local` |
| OU targeting | Workstation and user policies tested |
| Computer GPO | Applied and verified on Windows 11 |
| User GPO | Finance user affected, IT user unaffected |
| GPO precedence | OU policy won controlled conflict test |
| Security Filtering | `GG-Finance-Users` configured as target |
| Privileged group review | Domain/Enterprise/Schema Admins inspected |
| Least Privilege | Helpdesk reset allowed; privileged modification denied |
| FSMO roles | All five roles identified |
| SYSVOL | Accessible |
| NETLOGON | Accessible |
| DC diagnostics | `dcdiag` executed and findings reviewed |
| Replication tooling | `repadmin` executed and topology interpreted |
| Account auditing | Events 4720, 4728 and 4726 verified |

---

# 31. Items Not Fully Implemented or Verified

The following items should not be presented as completed controls.

## Second Domain Controller

A second DC was discussed as an improvement but was not deployed.

Proposed design:

```text
WIN-SRV2025-DC02
192.168.10.11
AD DS
DNS
```

Status:

```text
NOT IMPLEMENTED
```

---

## AGDLP Resource Permission

The group nesting:

```text
GG-Finance-Users
        ↓
DL-Finance-Read
```

was implemented.

An actual file share or NTFS Read permission was not assigned as part of this lab.

Status:

```text
GROUP DESIGN IMPLEMENTED
RESOURCE PERMISSION NOT IMPLEMENTED
```

---

## Effective Password Policy

Password policy settings were inspected in Group Policy.

A complete independent verification of the final effective password policy was not completed.

Status:

```text
INSPECTED
NOT FULLY VERIFIED
```

---

## Active Directory Backup and Recovery

No complete Active Directory backup and restore test was documented.

Status:

```text
NOT VERIFIED
```

---

## Time Synchronization

The Domain Controller reported:

```text
Leap Indicator: 3
```

Status:

```text
ISSUE IDENTIFIED
NOT RESOLVED
```

---

## Multi-DC Replication

The environment contains only one Domain Controller.

Status:

```text
NOT APPLICABLE TO CURRENT TOPOLOGY
```

---

# 32. Final Technical State

At the completion of the lab, the environment had:

```text
Windows Server 2025 Domain Controller
        │
        ├── AD DS
        ├── DNS
        ├── lab.local
        ├── Structured OUs
        ├── Users
        ├── Security Groups
        ├── AGDLP group nesting
        ├── Group Policy
        ├── Delegated administration
        ├── SYSVOL / NETLOGON
        └── Security auditing
                │
                ↓
        Windows 11 Domain Client
```

The main functional path was successfully demonstrated:

```text
DNS
 ↓
Domain Discovery
 ↓
Domain Join
 ↓
Identity
 ↓
OU Placement
 ↓
Group Membership
 ↓
Group Policy
 ↓
Delegated Access
 ↓
Audit Events
 ↓
Verification
```

The lab also identified real operational issues rather than presenting the environment as fully production-ready.

The most important unresolved finding is Domain Controller time synchronization.

The environment is therefore documented as a working Active Directory security lab with verified identity, Group Policy, delegation, and auditing functionality, together with clearly documented limitations and troubleshooting findings.

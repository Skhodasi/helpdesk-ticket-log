# IT Help Desk Ticket Log & Troubleshooting Runbook

Tier 1 service desk tickets resolved end to end in a home lab that combines on-premises Active Directory with Microsoft 365, Entra ID and Intune. Each ticket documents the issue, the investigation, the root cause, the fix, and how the fix was verified, with screenshots of every step.

The aim is to show how I approach support work: confirm who I'm helping, find the actual cause instead of guessing, fix it with the least access needed, and verify before closing. My CompTIA Security+ background shapes each decision.

## Lab environment

| Machine | Role | OS |
| --- | --- | --- |
| DC01 | Domain controller for `lab.local` (Server Core, no GUI) | Windows Server 2025 |
| WIN10-01 | Domain-joined help desk workstation with RSAT, enrolled in Intune | Windows 11 Pro |
| Hodasi Lab tenant | Microsoft 365 Business Premium: Entra ID users and MFA, Intune device management | Cloud |

Hosted in Oracle VirtualBox. The same lab supports my [password-spray detection project](https://github.com/Skhodasi/detection-lab-password-spray).

## Tickets

| # | Ticket | Summary | Key skills |
| --- | --- | --- | --- |
| 1 | [Account lockout](ticket-01-account-lockout/) | Traced a lockout to its source and unlocked the account | ADUC, Fine-Grained Password Policy, event ID 4740, PowerShell |
| 2 | [Password reset](ticket-02-password-reset/) | Reset a forgotten password with a forced change at next sign-in | ADUC, password policy, secure credential handling |
| 3 | [New hire onboarding](ticket-03-new-hire-onboarding/) | Provisioned a new employee with role-based group access | OUs, security groups, ADAC, group-based policy |
| 4 | [Employee offboarding](ticket-04-employee-offboarding/) | Removed a leaver's access and verified it, uncovering a cached-credential sign-in gap | Account disable, access removal, Kerberos (`klist`), cached logons |
| 5 | [Shared folder access denied](ticket-05-shared-folder-access/) | Built a two-layer-secured department share and resolved an access-denied ticket | SMB shares, NTFS permissions, security groups, least privilege |
| 6 | [Mapped drive via Group Policy](ticket-06-mapped-drive-gpo/) | Mapped a department share with a GPO and resolved a DC-locator DNS fault | GPO Drive Maps, gpresult, DNS troubleshooting, nltest |
| 7 | [Microsoft 365 new user](ticket-07-m365-new-user/) | Created and licensed a cloud user, then verified first sign-in and mailbox | Microsoft 365 admin center, Entra ID, licensing |
| 8 | [MFA reset after lost phone](ticket-08-mfa-reset/) | Removed a lost phone's Authenticator, revoked sessions and re-registered MFA | Entra ID authentication methods, session revocation, MFA |
| 9 | [Intune enrollment and compliance](ticket-09-intune-compliance/) | Enrolled a device, built a compliance policy, and found BitLocker unprotected despite a "Compliant" result | Intune, compliance policies, BitLocker, manage-bde, TPM |

**In progress:** ServiceNow ticketing, troubleshooting runbook.

## Approach

- **GUI first, PowerShell second.** Each task uses the tools a Tier 1 analyst uses daily (ADUC, ADAC, Event Viewer), with the PowerShell equivalent for scale and automation.
- **Verify identity before any account change.** It's the main defence against social engineering at the help desk.
- **Least privilege.** Access is granted through security groups, never directly to users.
- **Confirm the fix.** Every ticket is checked from both the admin and the user side before closing.

---

Selorm Hodasi · CompTIA Security+ · [GitHub](https://github.com/Skhodasi)

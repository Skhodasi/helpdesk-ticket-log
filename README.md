# Ticket #4: Employee Offboarding

> **Request:** HR reports that Kevin Hodasi (test user, Accounting Clerk) leaves today. Remove his access by 5 PM.
>
> **Result:** Disabled the account, removed all group access, documented the change, and moved the account to a Disabled Users OU. Verification uncovered a **cached-credential gap**: the disabled user could still sign in locally once, though with no network access. A later sign-in, checked by the domain controller, was blocked.

| Ticket | Date | Type | Status |
| --- | --- | --- | --- |
| REQ-0004 | Nov 19, 2026 | Service request – offboarding | Completed |

## Why disable instead of delete

Deleting an account is permanent. It loses the account's history and can cut off access to files the person owned. Standard practice is to **disable** the account straight away, keep it for a set period (often 30–90 days), and delete it later.

## Steps

**1. Create a holding OU.** I added **Staff → Disabled Users** to keep leavers separate from active staff.

![Create Disabled Users OU](screenshots/01-new-ou-disabled-users.png)

Starting point: Kevin is an active user in the Accounting OU.

![Kevin in Accounting before offboarding](screenshots/02-before-in-accounting.png)

**2. Disable the account.** This comes first, because it stops new sign-ins immediately. (Right-click → **Disable Account**. PowerShell: `Disable-ADAccount khodasi`)

![Account disabled](screenshots/03-account-disabled.png)

**3. Remove group access.** I removed **Accounting** and **All Staff**, leaving only *Domain Users*, which is the required primary group. If the account were ever re-enabled by mistake, it would come back with no access, and access reviews no longer list a former employee.

![Removing group memberships](screenshots/04-remove-groups.png)

```powershell
Remove-ADGroupMember -Identity "Accounting" -Members khodasi -Confirm:$false
Remove-ADGroupMember -Identity "All Staff" -Members khodasi -Confirm:$false
```

**4. Document the change.** I set the account description to `Disabled 2026-11-19 - left company - REQ-0004`, so anyone who finds the account later knows when, why, and which ticket.

![Account description](screenshots/05-description.png)

**5. Move to Disabled Users.** This keeps the Accounting OU to current staff only, and stops department-level Group Policy (such as drive mappings) from applying to the old account.

![Move to Disabled Users](screenshots/06-move-to-disabled-users.png)

## Verification and an unexpected finding

**Test 1: sign-in succeeded.** Signing in as Kevin with his correct password **worked**, even though the account was disabled. I investigated rather than assume:

| Check | Result | Meaning |
| --- | --- | --- |
| `whoami` | `lab\khodasi` | It really was the disabled account |
| `Test-Connection DC01` | Replies received | The domain controller was reachable |
| `query user` | Logon time 6:32 AM, after the 6:26 disable | A fresh sign-in, not an old session |
| `klist` | **Cached Tickets: (0)** | No Kerberos tickets. The domain controller never approved this sign-in. |
| `dir \\DC01\NETLOGON` | Access failed | **No access to network resources** |

![Investigation in PowerShell](screenshots/07-cached-signin-checks.png)

**Root cause:** Windows signed Kevin in using his **cached credentials**, the copy of his password saved from his last successful sign-in. Windows keeps these so laptops still work offline. The workstation never asked the domain controller, so it never learned the account was disabled. Without Kerberos tickets, though, Kevin couldn't reach any domain resource.

**Test 2: sign-in blocked.** After signing out, a new sign-in was checked against DC01 and refused:

![Your account has been disabled](screenshots/08-signin-blocked.png)

## Key takeaways

- **Disabling an account blocks network access immediately, but not always local sign-in.** A device with cached credentials, especially an offline laptop, can still let a former employee onto the desktop.
- **Complete offboarding also includes:** resetting the password (the cached copy then becomes useless), collecting or remotely wiping devices, and revoking remote access and cloud sessions.
- **Hardening option:** reduce or remove cached logons on shared or high-risk machines via Group Policy (*Interactive logon: Number of previous logons to cache*).
- **Test, don't assume.** The settings looked correct, but only an actual sign-in test exposed the gap.

**Skills:** Active Directory Users and Computers · account lifecycle management · access removal · Kerberos troubleshooting (`klist`) · cached credentials · PowerShell · security-focused verification

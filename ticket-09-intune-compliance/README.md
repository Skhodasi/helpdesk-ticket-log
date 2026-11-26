# Ticket #9: Enroll a Laptop in Intune and Enforce a Compliance Policy

> **Request:** Ama Tay's (test user, Marketing) work laptop needs to be managed by Microsoft Intune, and IT needs to confirm it meets the company's security requirements.
>
> **Result:** Removed an enrollment blocker, enrolled the device, corrected its ownership and created a compliance policy. Intune then reported the device fully compliant, but a check on the device itself showed BitLocker encryption was **suspended with no key protectors**, leaving the data unprotected. Traced the cause to a leftover installation disc, fixed it, and verified protection was on.

| Ticket | Date | Type | Status |
| --- | --- | --- | --- |
| REQ-0009 | Nov 24–25, 2026 | Service request (+ security issue found during work) | Completed |

## What Intune is

[Ticket #6](../ticket-06-mapped-drive-gpo/) used **Group Policy**, which manages computers through the on-site domain controller. **Microsoft Intune** manages devices from the cloud, so it works on any laptop with internet access, in or out of the office. A **compliance policy** is a list of security rules (encryption, firewall, antivirus). Intune checks each device against it and marks it *Compliant* or *Not compliant*.

## Part 1: Enrollment

**1. Find and fix the blocker.** In the Intune admin center (**Devices → Enrollment → Windows → Automatic Enrollment**), **MDM user scope** was set to **None**, so no user could enroll any device. Changed it to **All**.

![MDM scope None](screenshots/01-mdm-scope-none.jpg)

![MDM scope All](screenshots/02-mdm-scope-all.png)

**2. Enroll the device.** On WIN10-01: **Settings → Accounts → Access work or school**. The device was joined only to the on-prem `lab.local` domain. Selected **Enroll only in device management** and signed in as Ama (MFA approved).

![Before enrollment](screenshots/03a-access-work-before.png)

![Registering](screenshots/03b-registering.png)

![Setting up the device](screenshots/03c-setting-up.png)

**3. Verify on the device.** Settings showed **Managed by Hodasi Lab**, a successful sync, and Intune's management server address.

![Managed by Hodasi Lab](screenshots/04-managed-by-hodasi-lab.png)

**4. Verify in Intune.** WIN10-01 appeared as managed by Intune, with Ama as primary user.

![Intune device list](screenshots/04b-intune-device-list.jpg)

**5. Correct the ownership.** This enrollment method marks devices as **Personal**. Changed it to **Corporate**, since it's a company laptop. Intune allows more management actions on corporate devices (such as a full wipe if stolen) and notifies the user of the change.

![Ownership Personal](screenshots/05a-ownership-personal.png)

![Changing to Corporate](screenshots/05b-ownership-corporate.png)

![Ownership Corporate](screenshots/05c-properties-corporate.png)

## Part 2: Compliance policy

**6. A misleading "Compliant."** The device list showed *Compliant*, but no compliance policies existed. Intune's built-in policy confirmed it: **"Has a compliance policy assigned: Error (65001)."**

![No policies](screenshots/06-no-policies.png)

![Default policy error](screenshots/08a-default-policy-error.jpg)

**7. Create the policy.** **Devices → Compliance → Create policy → Windows 10 and later**, named `Windows - Baseline Security`:

| Section | Setting | Value |
| --- | --- | --- |
| Device Health | BitLocker | Require |
| System Security | Firewall | Require |
| System Security | Antivirus | Require |
| System Security | Antispyware | Require |
| Actions | Mark device noncompliant | Immediately |
| Assignment | Included groups | All users |

![BitLocker setting](screenshots/07a-policy-bitlocker.png)

![System security settings](screenshots/07b-policy-system-security.png)

![Policy review](screenshots/07c-policy-review.png)

**8. Sync and check.** Triggered a sync from the device (**Access work or school → Info → Sync**). The built-in error cleared, and the new policy reported **all four settings Compliant**.

![Default policy fixed](screenshots/08b-default-policy-fixed.jpg)

![Baseline all compliant](screenshots/09-baseline-all-compliant.png)

## Part 3: Verifying on the device

Intune's result was checked against the device itself, starting with BitLocker.

**9. Encryption suspended.** **Settings → Privacy & security → Device encryption** showed encryption **On**, but with a warning: *"Device encryption is temporarily suspended."*

![Encryption suspended](screenshots/10-encryption-suspended.png)

**10. Confirmed with the command line.** `manage-bde -status C:` (run as administrator):

| Field | Value | Meaning |
| --- | --- | --- |
| Percentage Encrypted | 100.0% | The disk is encrypted |
| **Protection Status** | **Protection Off** | The data is not protected |
| **Key Protectors** | **None Found** | The unlock key is stored unprotected on the disk, and no recovery key exists |

![Protection off](screenshots/11-manage-bde-protection-off.png)

A restart (which Windows said would resume protection) changed nothing.

**11. Ruled out a missing TPM.** BitLocker normally locks its key with the **TPM** security chip. Windows Security → Device security showed a TPM 2.0 with **Storage: Ready**, so the chip wasn't the problem. (*Attestation: Not ready* affects boot health reporting, not BitLocker.)

![TPM details](screenshots/12-tpm-details.jpg)

**12. Root cause.** Starting BitLocker manually failed: *"BitLocker Drive Encryption detected bootable media (CD or DVD) in the computer."* The Windows installation disc image was still inserted in the virtual DVD drive. BitLocker won't seal the key to the TPM while bootable media is present, so encryption had been left without any protector.

![Bootable media error](screenshots/13-bootable-media-error.png)

**13. Fix.** Removed the disc (VirtualBox **Devices → Optical Drives → Remove disk**), restarted, and ran **Turn on BitLocker** again. The recovery key was saved to an admin-only location off the device, then BitLocker was activated.

![Activate BitLocker](screenshots/14-activate-bitlocker.png)

## Verification

`manage-bde -status C:` now shows **Protection On**, with **TPM** and **Numerical Password** (recovery key) protectors.

![Protection on](screenshots/15-manage-bde-protection-on.png)

## Issues found

| Issue | Cause | Resolution |
| --- | --- | --- |
| No device could enroll | MDM user scope set to None | Set to All |
| Company laptop listed as Personal | "Enroll only in device management" default | Changed ownership to Corporate |
| Device "Compliant" with no policy | Intune treats devices without a policy as compliant | Created and assigned a baseline policy |
| BitLocker reported compliant but unprotected | Leftover bootable disc prevented TPM key protector | Removed disc, activated BitLocker, saved recovery key |

## Notes

- **Recovery key storage.** The wizard offered only *Save to a file* or *Print*, because the device is Intune-enrolled but not **Entra ID-joined**. On Entra-joined devices the key backs up to Entra automatically, where help desk can retrieve it.
- **Timestamps.** The VM clock runs one hour behind the host, so device times differ from the admin center.

## Key takeaways

- **A green dashboard isn't proof.** Intune marked BitLocker compliant while the drive's key had no protection. Checking the device directly with `manage-bde` found the real state.
- **"Compliant" can mean "not checked."** With no policy assigned, Intune reports devices as compliant by default.
- **Encrypted ≠ protected.** Encryption with no key protectors gives no protection against a stolen device.
- **Every BitLocker deployment needs a retrievable recovery key**, stored where IT can access it and not on the same drive.
- **Common real-world cause:** removable media (a USB stick or disc) left in a laptop can block BitLocker setup.

**Skills:** Microsoft Intune · MDM enrollment · compliance policies · device ownership · BitLocker / `manage-bde` · TPM · Windows Security · root cause analysis

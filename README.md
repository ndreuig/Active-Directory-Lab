# Active Directory Lab — techcorp.local

A self-built home lab simulating a small corporate Active Directory environment on Windows Server 2016. The goal was to practice the core sysadmin tasks encountered in a real helpdesk-to-sysadmin transition: domain setup, user and group management, Group Policy enforcement, shared folder permissions, and account troubleshooting.

---

## Scenario

A three-department company (IT, HR, Finance) needs a centralized directory service. As the sole administrator, I was responsible for standing up the domain controller, organizing users, locking down access based on department role, setting up shared network storage, and handling a real account lockout incident end-to-end.

---

## Environment

| Component | Details |
|-----------|---------|
| OS | Windows Server 2016 Standard Evaluation |
| Domain | techcorp.local |
| NetBIOS Name | TECHCORP |
| Server IP | 10.0.2.15 (static) |
| DNS | 127.0.0.1 (loopback — DC resolves itself) |
| Forest / Domain Level | Windows Server 2016 |

---

## What Was Built

- Promoted a Windows Server 2016 machine to a Domain Controller
- Structured the domain with three OUs: IT, HR, Finance
- Created users and security groups per department
- Applied a GPO to restrict Control Panel access for HR users
- Enforced a domain-wide password and account lockout policy
- Created department file shares with role-based NTFS and Share permissions
- Mapped a network drive for IT users
- Diagnosed and resolved a locked user account using both PowerShell and the GUI

---

## Part 1 — Domain Controller Setup

### Installing AD DS and Promoting the Server

AD DS was installed via the Add Roles and Features Wizard. Before promotion, a static IP (`10.0.2.15`) was assigned and DNS was pointed to loopback (`127.0.0.1`) — a required step so the DC can resolve its own domain after reboot.

The domain was configured as a new forest with the following settings:

| Setting | Value |
|---------|-------|
| Domain Name | techcorp.local |
| Forest / Domain Functional Level | Windows Server 2016 |
| Global Catalog | Yes |
| DNS Server | Yes |

All prerequisite checks passed. The DNS delegation warning is expected in an isolated lab with no parent zone — no action required.

![01 - AD DS Feature Installation](screenshots/01-adds-feature-installation.PNG)

---
![04 - Prerequisites Check Passed](screenshots/04-prerequisites-check-passed.PNG)

---

![05 - Domain Promotion Review](screenshots/05-domain-promotion-review.PNG)

---

![06 - Server Joined to Domain](screenshots/06-server-joined-domain.PNG)

> **Why this matters:** Static IP and loopback DNS are the two most common misconfigurations that break AD promotion. Getting these right before running the wizard is a habit worth building early.

---

## Part 2 — OU Structure, Users, and Groups

### Organizational Unit Design

Three OUs were created under `techcorp.local` to reflect the company's department structure. Separating users by OU is what makes Group Policy targeting and delegation of control possible later.

![07 - OU Structure](screenshots/07-ou-structure.PNG)

### Users and Security Groups

Users were created inside their respective OUs and added to department security groups. Groups — not individual users — are assigned to permissions and policies. This is the correct approach for any environment that will grow.

| Department | Users | Group |
|-----------|-------|-------|
| HR | John Carter, Anna Hill | HR_Group |
| IT | Mike Reed, Sara Lane | IT_Group |

![08 - HR Users](screenshots/08-hr-users.PNG)

---

![09 - HR Group Members](screenshots/09-hr-group-members.PNG)

---

![15 - IT Group Members](screenshots/15-it-group-members.PNG)

---

## Part 3 — Group Policy

### Restricting Control Panel for HR Users

A GPO named `HR-ControlPanel-Restriction` was created and linked to the HR OU. The setting **"Prohibit access to Control Panel and PC settings"** was enabled under:

`User Configuration > Administrative Templates > Control Panel`

This prevents HR users from opening Control Panel or the Windows Settings app entirely.

![10 - GPO Linked to HR OU](screenshots/10-gpo-linked-hr-ou.PNG)

---

![11 - Control Panel GPO Enabled](screenshots/11-gpo-controlpanel-enabled.PNG)

### Password Policy

A domain-wide password policy was enforced via the Default Domain Policy:

| Policy | Value |
|--------|-------|
| Enforce password history | 24 passwords |
| Maximum password age | 42 days |
| Minimum password age | 1 day |
| Minimum password length | 8 characters |
| Complexity requirements | Enabled |
| Reversible encryption | Disabled |

![12 - Password Policy](screenshots/12-password-policy.PNG)

### Account Lockout Policy

To protect against brute-force attempts, an account lockout policy was configured:

| Policy | Value |
|--------|-------|
| Account lockout threshold | 3 invalid logon attempts |
| Account lockout duration | 30 minutes |
| Reset lockout counter after | 30 minutes |

![21 - Account Lockout Policy](screenshots/21-account-lockout-policy.PNG)

Policy changes were applied immediately using:

```cmd
gpupdate /force
```

![22 - gpupdate /force](screenshots/22-gpupdate-force.PNG)

---

## Part 4 — Shared Folders and Permissions

### Folder Structure

Two department folders were created under `C:\CompanyShares` and shared over the network:

```
C:\CompanyShares\
├── HR_Files\
└── IT_Files\
```

![14 - Company Shared Folders](screenshots/14-company-shares-folders.PNG)

### Permission Design

Permissions were applied at two layers — Share and NTFS — following the principle of least privilege. Groups, not individual users, were assigned access.

**HR_Files — Read only (HR_Group)**

| Layer | Permission |
|-------|-----------|
| Share | Read |
| NTFS | Read, Read & Execute, List Folder Contents |

![16 - HR Files Share Permissions](screenshots/16-hr-files-share-permissions.PNG)

---

![17 - HR Files Security Permissions](screenshots/17-hr-files-security-permissions.PNG)

**IT_Files — Full Control (IT_Group)**

| Layer | Permission |
|-------|-----------|
| Share | Full Control, Change, Read |
| NTFS | Full Control, Modify, Read & Execute, List Folder Contents, Read |

![18 - IT Files Share Permissions](screenshots/18-it-files-share-permissions.PNG)

---

![19 - IT Files Security Permissions](screenshots/19-it-files-security-permissions.PNG)

> **Why two permission layers:** Share permissions apply only over the network. NTFS permissions apply always, including local access. Best practice is to grant Full Control at the share level and control access precisely through NTFS. Here HR received Read at both layers intentionally, since they should not modify department files.

### Mapped Network Drive

The `IT_Files` share was mapped as drive `I:` on a client machine, confirming the share is accessible over the network with the correct permissions applied.

![20 - Mapped Network Drive](screenshots/20-mapped-network-drive.PNG)

---

## Part 5 — Account Troubleshooting

### Scenario

User `john.carter` was locked out after 3 failed login attempts — exactly the lockout threshold set in policy. This was treated as a real helpdesk ticket: verify the issue first, then resolve it.

### Step 1 — Verify with PowerShell

Before touching anything in the GUI, the account status was confirmed using PowerShell:

```powershell
Get-ADUser -identity "john.carter" -Properties LockedOut, BadLogonCount `
  | Select Name, LockedOut, BadLogonCount
```

Output confirmed: `LockedOut: True`, `BadLogonCount: 3`

![23 - Account Locked PowerShell](screenshots/23-account-locked-powershell.PNG)

### Step 2 — Unlock via ADUC

The account was unlocked through Active Directory Users and Computers by navigating to the user's **Account** tab and checking **"Unlock account"**.

![24 - Unlock Account via GUI](screenshots/24-unlock-account-gui.PNG)

### Step 3 — Reset the Password

The password was reset via ADUC. Confirmation dialog verified the change was applied successfully.

![25 - Password Reset Success](screenshots/25-password-reset-success.PNG)

> **Why verify with PowerShell first:** In a production environment, always confirm the actual state of an account before making changes. PowerShell gives a precise, auditable view — important if the issue escalates or needs to be documented.

---

## Domain Login

Successful authentication as `TECHCORP\Administrator` confirms the domain controller is fully operational and domain logins are working as expected.

![13 - Domain Login](screenshots/13-domain-login.PNG)

---

## What I Learned

- The order of operations matters: static IP and DNS must be correct *before* AD DS promotion, not after
- GPOs only apply correctly when linked to the right OU — testing with `gpupdate /force` and then verifying the result is essential
- NTFS and Share permissions are two separate layers; understanding how they interact is critical for file server work
- Checking account status in PowerShell before touching the GUI is the professional approach — it gives you evidence and avoids mistakes
- Assigning permissions to groups instead of individual users is not just best practice, it is the only approach that scales

## What I Would Add Next

- A second Windows 10 client VM joined to the domain to test GPOs and share access from a real user session
- Fine-Grained Password Policies (PSOs) to apply stricter rules to admin accounts
- Audit policy configuration and reviewing Security Event logs (Event ID 4740 for lockouts)
- Automating user creation with a PowerShell script instead of the GUI

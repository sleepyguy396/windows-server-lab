## Lab 02: Building a Windows Server Domain Controller Integrated with Internal DNS

### 📌 Overview & Objective
This lab documents the end-to-end installation and configuration of an **Active Directory Domain Services (AD DS)** forest integrated with an internal **DNS Server**. It covers IPv6 interface remediation, Active Directory installation, Organizational Unit (OU) structuring, domain joining a Windows 10 workstation, and enforcing domain-wide Group Policy Object (GPO) security controls.

---

### 🌐 Environment & Topology

> **OPSEC / IP Sanitization Note:** Network parameters use RFC 5737 TEST-NET-1 documentation IP ranges (`192.0.2.0/24`) to obscure internal lab infrastructure details.

| Parameter | Domain Controller / DNS (`DC1`) | Client Workstation (`PC1`) |
| :--- | :--- | :--- |
| **OS** | Windows Server 2016 / 2019 / 2022 | Windows 10 Enterprise |
| **Hypervisor** | VMware Workstation | VMware Workstation |
| **Domain Name** | `qtm.local` | `qtm.local` |
| **IPv4 Address** | `192.0.2.101/24`| `192.0.2.103/24`|
| **Subnet Mask** | `255.255.255.0` | `255.255.255.0` |
| **Default Gateway**| `192.0.2.1` | `192.0.2.1` |
| **Preferred DNS** | `192.0.2.101` (Self) | `192.0.2.101` (`DC1` IP) |

---

### ⚙️ Step-by-Step Implementation Guide

#### Phase 1: Internal DNS Server Setup & Network Remediation

##### Step 1: Disable IPv6 Binding
1. Open Network Connections: `Win + R` → `ncpa.cpl` → **Enter**.
2. Right-click the active Ethernet adapter → **Properties**.
3. Uncheck **Internet Protocol Version 6 (TCP/IPv6)** → Click **OK**.
> *Reasoning:* Prevents unconfigured IPv6 DNS resolution fallback during local active directory lookup calls.

##### Step 2: Configure Static IPv4 Parameters
1. Open IPv4 properties on `DC1`:
   * **IP Address:** `192.0.2.101`
   * **Subnet Mask:** `255.255.255.0`
   * **Default Gateway:** `192.0.2.1`
   * **Preferred DNS:** `192.0.2.101`
2. Verify configuration via Command Prompt:
   ```cmd
   ipconfig /all
   ```

##### Step 3: Install DNS Server Role
1. Open Server Manager → Manage → Add Roles and Features.
2. Select Role-based or feature-based installation.
3. Target local server → Select DNS Server → Accept default features → Click Install.

##### Step 4: Configure Forward & Reverse Lookup Zones
1. Open DNS Manager (dnsmgmt.msc).
2. Forward Lookup Zone:
    - Right-click Forward Lookup Zones → New Zone...
    - Zone Type: Primary zone.
    - Zone Name: qtm.local
    - Dynamic Updates: Allow both nonsecure and secure dynamic updates.
3. Reverse Lookup Zone:
    - Right-click Reverse Lookup Zones → New Zone...
    - Zone Type: Primary zone → IPv4 Reverse Lookup Zone.
    - Network ID: 192.0.2
    - Dynamic Updates: Allow both nonsecure and secure dynamic updates.
4. Create Pointer (PTR) Record:
    - Open 2.0.192.in-addr.arpa zone → Right-click empty area → New Pointer (PTR).
    - Host IP Number: 101
    - Host Name: Browse and select DC1.qtm.local.

#### Phase 2: Active Directory Domain Services (AD DS) Promotion

##### Step 1: Install AD DS Role

1. Open Server Manager → Add Roles and Features.
2. Select Active Directory Domain Services → Add Required Features → Complete wizard.

##### Step 2: Promote Server to Domain Controller

1. Click the flag notification in Server Manager → Promote this server to a domain controller.
2. Deployment Configuration: Select Add a new forest.
3. Root Domain Name: qtm.local
4. Domain Functional Level: Windows Server 2016.
5. DSRM Password: Enter a secure Directory Services Restore Mode password.
6. Accept default NetBIOS name (QTM) and database folder paths (NTDS / SYSVOL).
7. Click Install. Server will reboot automatically upon completion.

#### Phase 3: Organizational Units (OUs), Groups, & Users

##### Step 1: Create OU & User Accounts
1. Launch Active Directory Users and Computers (dsa.msc).
2. Right-click qtm.local → New → Organizational Unit → Name: Phong Nhan Su.
3. Inside Phong Nhan Su, create User Accounts:
    - User 1: a.np (Full Name: Nguyen Van A)
    - User 2: b.nv (Full Name: Nguyen Van B)
4. Uncheck "User must change password at next logon" and check "Password never expires" for testing purposes.

##### Step 2: Group Creation & Membership Assignment

1. Right-click Phong Nhan Su → New → Group.
2. Name: G_NHANSU (Scope: Global, Type: Security).
3. Add a.np and b.nv as members of G_NHANSU.

#### Phase 4: Joining Client Workstation to Domain

1. Boot PC1 and assign static network configuration (192.0.2.103/24, Preferred DNS: 192.0.2.101).
2. Uncheck TCP/IPv6 on PC1's Ethernet interface.
3. Test connectivity and name resolution:

     ```cmd
     ping qtm.local
     ```
     
     ```cmd
     nslookup qtm.local
     ```
     
4. Open System Properties (sysdm.cpl) → Click Change...
5. Under Member of, select Domain → Enter qtm.local.
6. Authenticate using domain credentials (QTM\Administrator).
7. Restart PC1 when prompted.

#### Phase 5: Domain Security Hardening via Group Policy (GPO)
Launch Group Policy Management (gpmgmt.msc) on DC1 and edit the Default Domain Policy:

1. Logon Hours Control
    - In dsa.msc, open properties for a.np and b.nv.
    - Under Account tab → Logon Hours: Restrict access to Mon–Fri, 08:00–17:00.

2. Account & Password Policies
    - Navigate to: Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies
    - Password Policy:
          - Minimum password length: 8 characters
          - Password must meet complexity requirements: Enabled
          - Maximum password age: 30 days
          - Enforce password history: 1 password remembered
    - Account Lockout Policy:
          - Account lockout threshold: 5 invalid logon attempts
          - Account lockout duration: 30 minutes

3. Interactive Logon Policies
    - Navigate to: Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options
          - Interactive logon: Do not display last user name → Enabled
          - Interactive logon: Do not require CTRL+ALT+DEL → Disabled
    - Enforce policy changes immediately across the domain:
      ```cmd
      gpupdate /force
      ```

#### Verification & Security Testing
Perform validation testing from PC1 (192.0.2.103):

1. Domain DNS Resolution Test:
   ```cmd
   nslookup DC1.qtm.local
   ```
> Expected Result: Resolves correctly to 192.0.2.101.

2. Interactive Logon Test
    - Log on to PC1 using QTM\a.np. Confirm the system authenticates against the domain controller without local account fallbacks.
   
3. Account Lockout Enforcer Test
    - Attempt 5 consecutive incorrect passwords for QTM\b.nv.
    - Expected Result: System locks the account and displays: "The referenced account is currently locked out and may not be logged on to."

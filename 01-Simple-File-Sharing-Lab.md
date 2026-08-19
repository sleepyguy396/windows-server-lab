## Lab 01: Simple File Sharing & Guest Access Model (Windows 10 / Windows Server)

### 📌 Overview & Objective
This lab documents the step-by-step configuration of **Simple File Sharing** using the **Guest-Only Authentication Model** across virtualized Windows endpoints. 

The primary objective is to demonstrate how network shares function when password-protected sharing is disabled and local group policies force non-authenticated network users into the restricted `Guest` security context.

---

### 🌐 Environment & Topology

> **Note on OPSEC / IP Sanitization:** All network parameters below utilize RFC 5737 TEST-NET-1 documentation IP ranges (`192.0.2.0/24`) to obscure internal lab infrastructure details.

| Parameter | Host / Target 1 (PC1 - Server/Host) | Host / Target 2 (PC2 - Client) |
| :--- | :--- | :--- |
| **OS** | Windows 10 Enterprise / Windows Server | Windows 10 Enterprise / Windows Server |
| **Hypervisor** | VMware Workstation | VMware Workstation |
| **Network Adapter** | Custom (`VMnet1` - Host-Only) | Custom (`VMnet1` - Host-Only) |
| **IP Address** | `192.0.2.10/24`  | `192.0.2.15/24` |

---

### ⚙️ Step-by-Step Implementation Guide

#### Step 1: Verify Virtual Network & Interface IP Binding
> **Goal:** Ensures layer 2/3 connectivity between VMs

1. Open VM Settings for both virtual machines in VMware Workstation.
2. Set network adapters to the same isolated virtual switch (e.g., `Custom: VMnet1 (Host-Only)`).
3. Verify connectivity via Command Prompt on both nodes:
   ```cmd
   ipconfig /all
   ```
4. Confirm PC1 (192.0.2.10) and PC2 (192.0.2.15) reside on the same broadcast domain.

#### Step 2: Enable & Unblock the Local Guest Account
> Goal: Required for anonymous network mapping

1. Open Computer Management: Win + R → compmgmt.msc → Enter.
2. Expand System Tools → Local Users and Groups → Users.
3. Right-click the Guest user account → Select Properties.
4. Uncheck Account is disabled.
5. Click Apply and OK.

#### Step 3: Configure Local Group Policy (Security Model)
> Goal: Force incoming SMB sessions to authenticate as Guest

1. Open Local Group Policy Editor: Win + R → gpedit.msc → Enter.
2. Navigate to:
  Computer Configuration → Windows Settings → Security Settings → Local Policies → Security Options
3. Locate: Network access: Sharing and security model for local accounts.
4. Set policy to: Guest only - local users authenticate as Guest.
5. Force policy update via Command Prompt:
```cmd
gpupdate /force
```

#### Step 4: Configure Sharing Wizard & Turn Off Password Protection
> Goal: Disables credential prompts for SMB network access

1. Open File Explorer → View tab → Options (Folder Options).
2. Under View tab, ensure Use Sharing Wizard (Recommended) is checked.
3. Open Network and Sharing Center (control /name Microsoft.NetworkAndSharingCenter).
4. Click Change advanced sharing settings.
5. Expand All Networks → Select Turn off password protected sharing.
6. Save changes.

#### Step 5: Create Share & Assign Read-Only Guest Permissions
> Goal: Configuring folder-level SMB share permissions

1. Create directory C:\Data on PC1.
2. Create test file C:\Data\Data.txt.
3. Right-click C:\Data → Properties → Sharing (or Give access to → Specific people...).
4. Add the Guest account.
5. Set Permission Level to Read.
6. Confirm SMB share creation (\\<HOST_NAME>\Data or \\192.0.2.10\Data).

#### Step 6: Configure Windows Defender Firewall Rules
> Goal: Allows inbound ICMP and File/Printer Sharing traffic

1. Open Windows Defender Firewall with Advanced Security (wf.msc).
2. Enable Inbound Rules for File and Printer Sharing (SMB-In) across Private and Public profiles.
3. Alternatively, enable File and Printer Sharing via control firewall.cpl → Allow an app or feature through Windows Defender Firewall.

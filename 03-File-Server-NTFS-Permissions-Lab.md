## Lab 03: Deploying Windows File Server with Shared Folders, NTFS Permissions & Network Drive Mapping

### 📌 Overview & Objective
This lab documents the deployment of an enterprise **File Server** running on Windows Server. It covers configuring dynamic share and NTFS permissions (Role-Based Access Control), automating individual **Home Folder** creation via Active Directory environment variables, and deploying **Logon Scripts** via Group Policy Objects (GPO) to map shared network drives automatically based on user department OUs.

---

### 🌐 Infrastructure & Network Topology

> **OPSEC / IP Sanitization Note:** Network parameters use RFC 5737 TEST-NET-1 documentation IP ranges (`192.0.2.0/24`) to obscure internal lab infrastructure details.

| Endpoint | Role | OS | IP Address | Domain |
| :--- | :--- | :--- | :--- | :--- |
| **`DC1`** | Domain Controller / DNS / File Server | Windows Server | `192.0.2.101/24` | `qtm.local` |
| **`PC1`** | Domain Workstation Client | Windows 10 Enterprise | `192.0.2.103/24` | `qtm.local` |

#### Domain OU & Group Architecture
* **Organizational Units (OUs):** `Phong Nhan Su` (HR), `Phong Ke Toan` (Accounting), `Phong CNTT` (IT)
* **Global Security Groups:** `G_NHANSU`, `G_KETOAN`, `G_CNTT`
* **Department Head Naming Convention:** User login ends with `tv` (e.g., `a.tv` = Department Manager)
* **Standard Employee Naming Convention:** User login ends with `nv` (e.g., `b.nv` = Staff)

---

### 📂 Directory & Network Drive Mapping Matrix

| Shared Folder Path | Target Users | Permissions Level | Mapped Drive Letter |
| :--- | :--- | :--- | :--- |
| `D:\Public` | All Domain Users | Read/Execute, Write (Create Files/Folders), Modify **own** files only | `G:` |
| `D:\<Department>` | Department Staff (`G_<DEPT>`) | Read/Execute, List Folder Contents (This folder & files only) | `F:` |
| `D:\<Department>` | Department Head (`*.tv`) | Full Control / Modify across Department Share | `F:` |
| `D:\<Department>\%USERNAME%` | Account Owner Only | **Full Control** (Isolated Home Directory) | `B:` |

---

### ⚙️ Step-by-Step Implementation Guide

#### Phase 1: Storage Directory Structure Setup
On Server `DC1` (`192.0.2.101`), create the root directory layout on drive `D:\`:
D:

├── Public

├── NhanSu

├── KeToan

└── CNTT\

---

#### Phase 2: Public Folder Share & Advanced NTFS Permissions

##### Step 1: Configure Share Permissions
1. Right-click `D:\Public` → **Properties** → **Sharing** → **Advanced Sharing**.
2. Check **Share this folder** (Share Name: `Public`).
3. Click **Permissions**:
   * Remove `Everyone`.
   * Add `QTM\Domain Users`.
   * Grant **Full Control** *(NTFS will restrict actual permissions)*.

##### Step 2: Configure Advanced NTFS Permissions
1. Switch to **Security** tab → Click **Advanced**.
2. Click **Disable inheritance** → Select **Convert inherited permissions into explicit permissions on this object**.
3. Remove `Users` / `Domain Users` explicit read entries.
4. Click **Add** → **Select a principal** → `QTM\Domain Users`:
   * **Type:** Allow
   * **Applies to:** This folder, subfolders, and files
   * **Basic Permissions:** Read & execute, List folder contents, Read, Write.
5. Click **Show advanced permissions**:
   * Uncheck `Create files / write data` and `Create folders / append data` from the standard write set if needed, but ensure **Creator Owner** retains **Full Control**.
   * *Mechanism:* Default `CREATOR OWNER` entry ensures users can modify/delete files **they personally upload**, but cannot alter or delete files uploaded by other domain members.

---

#### Phase 3: Department Folder Share & Role-Based Access Control (RBAC)

Perform the following for each department folder (`NhanSu`, `KeToan`, `CNTT`):

##### Step 1: Share Configuration
1. Right-click `D:\NhanSu` → **Properties** → **Sharing** → **Advanced Sharing**.
2. Share Name: `NhanSu`.
3. Share Permissions: Add `QTM\Administrators` (**Full Control**) and `QTM\G_NHANSU` (**Full Control**).

##### Step 2: Granular NTFS Permissions
1. Open **Security** tab → **Advanced** → **Disable inheritance** (Convert to explicit).
2. Remove standard domain `Users` groups.
3. **Configure Department Group (`G_NHANSU`):**
   * Principal: `QTM\G_NHANSU`
   * Applies to: **This folder and files only** *(Crucial: Prevents group access from bleeding down into isolated individual Home Folders)*.
   * Permissions: Read, List folder contents, Read & execute.
4. **Configure Department Head Account (`a.tv`):**
   * Principal: `QTM\a.tv`
   * Applies to: **This folder, subfolders, and files**
   * Permissions: Modify, Read & execute, List folder contents, Write.

---

#### Phase 4: Automated Home Directory Provisioning

1. Launch **Active Directory Users and Computers** (`dsa.msc`).
2. Select target user accounts (e.g., `a.tv`, `b.nv` under `Phong Nhan Su`).
3. Right-click → **Properties** → **Profile** tab.
4. Under **Home folder**, select **Connect**:
   * **Drive Letter:** `B:`
   * **To Path:** `\\DC1\NhanSu\%USERNAME%`
5. Click **Apply**. 
   > *System Action:* Active Directory resolves `%USERNAME%` automatically (e.g., `\\DC1\NhanSu\b.nv`), creates the directory on disk, and sets NTFS owner/permissions exclusively to `b.nv` with Full Control.

---

#### Phase 5: GPO Automated Network Drive Mapping

##### Step 1: Draft Batch Script
Create logon batch script files for each OU (e.g., `map_nhansu.bat`):
```bat
@echo off
:: Map Department Share to F: Drive
net use F: \\DC1\NhanSu /persistent:no

:: Map Public Share to G: Drive
net use G: \\DC1\Public /persistent:no
```

##### Step 2: Deploy Logon Script via GPO
1. Open Group Policy Management (gpmgmt.msc).
2. Expand domain qtm.local → Right-click target OU Phong Nhan Su → Create a GPO in this domain, and Link it here...
3. Name: GPO_DriveMapping_NhanSu.
4. Right-click GPO → Edit.
5. Navigate to: User Configuration → Policies → Windows Settings → Scripts (Logon/Logoff) → Double-click Logon.
6. Click Show Files... and save map_nhansu.bat into that exact directory path.
7. Click Add... → Select map_nhansu.bat → Click OK → Apply.

#### Verification & Audit Procedures
1. Log into domain workstation PC1 (192.0.2.103) as standard staff user QTM\b.nv.
2. Open File Explorer and inspect This PC:
    - B: Drive resolves to \\DC1\NhanSu\b.nv (Full read/write isolated folder).
    - F: Drive resolves to \\DC1\NhanSu (Department shared drive).
    - G: Drive resolves to \\DC1\Public (Company-wide shared drive).
3. Access Enforcement Test:
    - Attempt to open \\DC1\CNTT or \\DC1\KeToan.
    - Expected Result: Access Denied (Error 0x80070005).
4. Public Folder Modify Enforcement Test:
    - Attempt to edit or delete a file uploaded by a.tv within G:\Public.
    - Expected Result: Action Denied (Only original creator or Administrator can alter files).

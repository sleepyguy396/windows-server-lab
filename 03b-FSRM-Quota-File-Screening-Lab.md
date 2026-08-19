## Lab 03b: Implementing File Server Resource Manager (FSRM) for Storage Quotas & File Screening

### 📌 Overview & Objective
This lab extends enterprise file server capabilities by deploying the **File Server Resource Manager (FSRM)** role service on Windows Server. It covers configuring hard storage quotas across departmental directories and enforcing file screening rules to block unauthorized file extensions (e.g., executables, scripts) from being uploaded to server shares.

---

### 🌐 Infrastructure Overview

| Hostname | Role | IP Address | OS |
| :--- | :--- | :--- | :--- |
| **`DC1`** | Primary File Server / FSRM Role | `192.0.2.101/24` | Windows Server |
| **`PC1`** | Client Workstation | `192.0.2.103/24` | Windows 10 Enterprise |

---

### ⚙️ Step-by-Step Implementation Guide

#### Phase 1: Install FSRM Role Service

1. On `DC1`, open **Server Manager** → **Manage** → **Add Roles and Features**.
2. Under **Server Roles**, expand **File and Storage Services** → **File and iSCSI Services**.
3. Check **File Server Resource Manager** → Accept required management tools → Complete wizard.

---

#### Phase 2: Storage Quota Management

##### Policy Requirement
Total storage limit per employee across home directory and departmental shares:
* **HR Department (`NhanSu`):** `2 GB` Hard Quota
* **Accounting (`KeToan`):** `3 GB` Hard Quota
* **IT Department (`CNTT`):** `5 GB` Hard Quota

##### Step 1: Create Custom Quota Template
1. Open **File Server Resource Manager** (`fsrm.msc`).
2. Expand **Quota Management** → Right-click **Quota Templates** → **Create Quota Template...**
3. Template Name: `2GB Hard Quota Limit`.
4. Space Limit: `2 GB`.
5. Select **Hard Quota** *(Do not allow users to exceed limit)*.
6. Click **OK**.

##### Step 2: Apply Quotas to Directories
1. Right-click **Quotas** → **Create Quota...**
2. Quota Path: Browse to `D:\NhanSu`.
3. Select **Auto apply template and create quotas on existing and new subfolders** *(Enforces quota across individual home directories underneath)*.
4. Derive properties from template: Select `2GB Hard Quota Limit`.
5. Click **Create**.
6. Repeat step for `D:\KeToan` (3 GB) and `D:\CNTT` (5 GB).

---

#### Phase 3: File Screening & Extension Blocking

##### Policy Requirement
Block users from uploading executable and script files (`.exe`, `.dll`, `.bat`, `.cmd`, `.vbs`, `.msi`, `.ps1`) to server shares.

##### Step 1: Define Custom File Group
1. In `fsrm.msc`, expand **File Screening Management** → Right-click **File Groups** → **Create File Group...**
2. File Group Name: `Executable and Script Files`.
3. **Files to include:** *.exe, *.com, *.bat, *.cmd, *.vbs, *.vbe, *.js, *.jse, *.wsf, *.wsh, *.msi, *.ps1
4. Click **OK**.

##### Step 2: Create Custom File Screen Template
1. Right-click **File Screen Templates** → **Create File Screen Template...**
2. Template Name: `Block Executables and Scripts`.
3. Select **Active screening** *(Do not allow users to save blocked files)*.
4. Select File Groups: Check `Executable and Script Files`.
5. Click **OK**.

##### Step 3: Apply File Screen to Department Shares
1. Right-click **File Screens** → **Create File Screen...**
2. File Screen Path: `D:\NhanSu` (and repeat for `D:\KeToan`, `D:\CNTT`, `D:\Public`).
3. Select `Block Executables and Scripts` template.
4. Click **Create**.

---

### 🧪 Verification & Security Auditing

Perform verification tests from client workstation **PC1** (`192.0.2.103`):

#### 1. Quota Enforcement Test
1. Connect to mapped home drive `B:` (`\\DC1\NhanSu\b.nv`).
2. Attempt to copy a file exceeding the `2 GB` quota limit.
3. *Expected Result:* Windows displays an error prompt: *"There is not enough space on B:\. You need an additional X GB to copy these files."*

#### 2. File Screen Block Test
1. Attempt to drag and drop or create an executable file (`test_app.exe` or `script.bat`) inside `F:\` or `G:\Public`.
2. *Expected Result:* Windows blocks the file creation with the error message: *"You need permission to perform this action / Access is denied."*

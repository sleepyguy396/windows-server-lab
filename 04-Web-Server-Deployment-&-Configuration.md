## Lab 4: Web Server Deployment & Configuration (Windows Server 2016 & IIS)
> **OPSEC / IP Sanitization Note:** Network parameters use RFC 5737 TEST-NET-1 documentation IP ranges (`192.0.2.0/24`) to obscure internal lab infrastructure details.

**Environment Requirements:**
- **Server 1 (SV1):** Windows Server 2016 R2 (Services: Internal DNS Server, IIS Web Server) — IP: `192.0.2.101`
- **Server 2 (SV2):** Windows Server 2016 R2
- **Client (PC1):** Windows 11 — IP: `192.0.2.103`
- **Internal Domain:** `qtm.local`

## ⚙️ Step-by-Step Implementation Guide

#### Phase 1: Installing & Configuring Internal DNS Server

##### Step 1: Install DNS Server Role
  1. Open **Server Manager** -> **Manage** -> **Add Roles and Features**.
  2. Select **Role-based or feature-based installation**.
  3. Choose the local server from the pool (`SV1`).
  4. Select **DNS Server**, click **Add Features**, and proceed through the wizard.
  5. Click **Install** and wait for completion.

##### Step 2: System Renaming & Domain Assignment
  1. Open **Server Manager** -> **Local Server**.
  2. Change the **Computer Name** to `SV1` and append the domain `qtm.local`.
  3. Restart the server when prompted.

##### Step 3: Configure Forward Lookup Zone [00:03:45]
  1. Open **DNS Manager** (via `Server Manager -> Tools -> DNS`).
  2. Right-click **Forward Lookup Zones** -> **New Zone**.
  3. Choose **Primary Zone**.
  4. Enter Zone Name: `qtm.local`.
  5. Select **Allow both nonsecure and secure dynamic updates**.

##### Step 4: Configure Reverse Lookup Zone
  1. Right-click **Reverse Lookup Zones** -> **New Zone**.
  2. Choose **Primary Zone** -> **IPv4 Reverse Lookup Zone**.
  3. Enter Network ID: `192.0.2`.
  4. Select **Allow both nonsecure and secure dynamic updates**.

#####  Step 5: Create Host (A) and PTR Records
  1. In `qtm.local`, create a **New Host (A)** record:
     - **Name:** `SV1`
     - **IP Address:** `192.0.2.101`
     - Check **Create associated pointer (PTR) record**.
  2. Right-click `SV1` -> **Clear Cache** & restart DNS service via **All Tasks -> Restart**.
  3. Flush DNS in command prompt:
     ```cmd
     ipconfig /flushdns
     ```
#####  Step 6: Verify DNS operation
  - using nslookup:
      ```cmd
      nslookup google.com.vn
      ```

##### Step 7: Client Network Configuration (PC1)
  1. Assign static IPv4 settings on Windows 11 client:
        - IP Address: 192.0.2.103
        - Subnet Mask: 255.255.255.0
        - Gateway: 192.0.2.1
        - Preferred DNS Server: 192.0.2.101
  2. Disable IPv6 on client and server adapters to avoid DNS lookup conflicts.

#### Phase 2: Installing Internet Information Services (IIS)
  1. Open Server Manager -> Add Roles and Features.
  2. Select Web Server (IIS) role and click Add Features.
  3. Under Role Services, enable additional required features:
        - Basic Authentication
        - IP and Domain Restrictions
        - .NET Extensibility 4.6 / ASP.NET 4.6

  4. Click Install.

#### Phase 3: Website Creation & Document Configuration

##### Step 1: Default Web Site Verification

1. Open IIS Manager (Tools -> Internet Information Services (IIS) Manager).
2. Test default site locally: navigate to http://localhost or http://192.0.2.101.

##### Step 2: Create a Custom Web Site

1. In IIS Manager, right-click Sites -> Add Website.
2. Configure settings:
        - Site name: qtm.local
        - Physical path: C:\qtm
        - IP address: 192.0.2.101
        - Port: 80

##### Step 3: Index Document Creation

1. In C:\qtm, create an index page named index.html with basic HTML content.
2. In IIS Manager, select qtm.local -> open Default Document.
3. Add index.html to the top of the list and remove unnecessary entries.

#### Phase 4: DNS Binding & Host Name Configuration

1. In IIS Manager, right-click qtm.local site -> Edit Bindings....
2. Edit HTTP binding: Set Host name to www.qtm.local.
3. In DNS Manager -> Forward Lookup Zones -> qtm.local:
      - Right-click -> New Host (A).
      - Name: www
      - IP Address: 192.0.2.101
      - Test website access from client browser using: http://www.qtm.local

#### Phase 5: IIS Advanced Settings & Access Control (IP Restrictions)
##### Step 1: Limits & Connection Limits

1. In IIS Manager, select qtm.local -> open Limits.
2. Configure runtime parameters:
        - Bandwidth limit: 1 GB
        - Connection timeout: 120 seconds
        - Maximum concurrent connections: 200

##### Step 2: IP Address Restriction

1. Select qtm.local -> open IP Address and Domain Restrictions.
2. Click Edit Feature Settings... and set Unspecified clients to Allow.
3. Click Add Deny Entry...:
        - Specific IP address: 192.0.2.103 (PC1)
4. Verify from PC1: Attempting to browse http://www.qtm.local now returns an HTTP 403.6 - Forbidden error.

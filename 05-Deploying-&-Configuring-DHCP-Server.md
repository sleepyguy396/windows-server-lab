# Lab 5: Deploying & Configuring DHCP Server (Windows Server 2016)

> **OPSEC / IP Sanitization Note:** Network parameters use RFC 5737 TEST-NET-1 documentation IP ranges (`192.0.2.0/24`) to obscure internal lab infrastructure details.
> 
**Environment Requirements:**
- **Server (SV1):** Windows Server 2016 Datacenter (Domain Controller / Active Directory Integrated) — IP: `192.0.2.101/24`
- **Client (PC1):** Windows 10 Pro — Dynamic IP Assignment via DHCP
- **Default Gateway:** `192.0.2.1` [00:06:54]
- **Hypervisor Setup:** Internal Virtual Network (`VMnet1` / Host-Only) — Ensure VMware Built-in DHCP is disabled to avoid IP allocation conflicts
---

#### Phase 1: Installing the DHCP Server Role
1. Open **Server Manager** on `SV1`.
2. Click **Manage** -> **Add Roles and Features**.
3. Select **Role-based or feature-based installation** and click **Next**.
4. Select the destination server (`SV1`) from the server pool.
5. Check **DHCP Server**, click **Add Features** on the pop-up prompt, and click **Next**.
6. Leave default features unchanged and click **Install**.
7. Wait for the installation to complete.

#### Phase 2: Authorizing the DHCP Server in Active Directory
1. In **Server Manager**, click the **Notification Flag** (yellow exclamation icon) at the top right.
2. Click **Complete DHCP configuration**.
3. In the Authorization Wizard:
   - Use default Domain Admin credentials to authorize the server in Active Directory.
   - Click **Commit**, then **Close**.
4. Open **DHCP Manager** (`Server Manager -> Tools -> DHCP`).
5. Expand `IPv4` and verify authorization status:
   - Right-click the server name. If **Unauthorize** appears in the menu, the server is properly authorized in Active Directory.
#### Phase 3: Creating and Configuring a DHCP Scope

##### Step 1: Define Scope Range and Exclusions
1. In **DHCP Manager**, expand **SV1**, right-click **IPv4**, and select **New Scope...**.
2. **Scope Name:** Enter `NV_Scope_20` (e.g., Scope for Staff Workstations) and click **Next**.
3. **IP Address Range:**
   - **Start IP address:** `192.0.2.1` 
   - **End IP address:** `192.0.2.254`
   - **Length:** `24` / **Subnet Mask:** `255.255.255.0`
4. **Add Exclusions and Delay:**
   - Exclude Gateway IP: `192.0.2.1` -> Click **Add**.
   - Exclude Server Static IP: `192.0.2.101` -> Click **Add**.
   - Click **Next**.

##### Step 2: Lease Duration & Scope Options
1. **Lease Duration:** Set to **6 days** (standard for fixed internal workstation environments) and click **Next**.
2. Choose **Yes, I want to configure these options now**.
3. **Router (Default Gateway):** Enter `192.0.2.1` -> Click **Add** -> Click **Next**.
4. **Domain Name and DNS Servers:**
   - **Parent domain:** `qtm.local`
   - **IP Address:** `192.0.2.101` -> Click **Add** -> Click **Next**.
5. **WINS Servers:** Leave default (blank) and click **Next**.
6. Select **Yes, I want to activate this scope now** and click **Finish**.


#### Phase 4: Verifying DHCP Client Allocation & Renewal

##### Step 1: Obtain Dynamic IP on Client (PC1)
1. Switch to **PC1** (Windows 10).
2. Ensure network adapter settings are set to **Obtain an IP address automatically** and **Obtain DNS server address automatically**.
3. Open **Command Prompt** (`cmd`) and verify IP configuration:
   ```cmd
   ipconfig /all
   ```
    - Confirm details received from DHCP.
    - IPv4 Address: 192.0.2.3 (or next available IP in scope)
    - Subnet Mask: 255.255.255.0
    - Lease Obtained / Expires: 6 days duration
    - Default Gateway: 192.0.2.1
    - DHCP / DNS Server: 192.0.2.101


##### Step 2: Test IP Release and Renewal
1. Release the current IP binding:
   ```cmd
   ipconfig /release
   ```

2. Request a new lease from the DHCP server:
   ```cmd
   ipconfig /renew
   ```
3. Verify that PC1 retains or re-acquires its leased address.
4. On SV1, open DHCP Manager -> expand Scope -> click Address Leases to view active lease entries (PC1 hostname and MAC address).

#### Phase 5: Setting Up Static DHCP Reservations
DHCP Reservation ensures a specific device (e.g., management PC, dedicated printer) always receives the exact same IP address dynamically based on its Network Interface MAC address.

##### Step 1: Retrieve Client MAC Address
1. On PC1, view the MAC address in Command Prompt:
   ```DOS
   getmac
   ```
> Example MAC: 00-0C-29-96-5B-5A

##### Step 2: Create Reservation on Server
1. On SV1, inside DHCP Manager, expand scope NV_Scope_20.
2. Right-click Reservations -> New Reservation....
3. Configure the reservation parameters:
        - Reservation name: PC1_Reserved_100
        - IP address: 192.0.2.100
        - MAC address: 000C29965B5A (enter without dashes or colons)
        - Supported types: Both (DHCP and BOOTP)
4. Click Add, then Close.

##### Step 3: Verify Reservation Enforcement
1. Switch to PC1 and force a DHCP lease renewal:
   ```DOS
   ipconfig /release
   ipconfig /renew
   ```

2. Check the updated network profile using ipconfig:
        - IPv4 Address: 192.0.2.100
3. Confirm that the client successfully bound to the reserved static IP address.

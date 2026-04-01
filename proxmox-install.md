# Proxmox VE 9 Installation and Initial Configuration

## Hardware setup:

Repurposing old workstation for Proxmox:

- <u>Motherboard:</u> *Asus Z9PA-D8*
  - <u>Network:</u> *(2) Intel WG82574L Gigabit Adapters + (1) Management Port*

- <u>Processors:</u> *(2) Intel Xeon E5-2620 v2 2.1GHz LGA 2011*

- <u>Memory:</u> *Kingston ValueRAM 64GB (4 x 16GB) ECC Registered DDR3 1600*

- <u>Video:</u> *NVIDIA GeForce GTX 1650*

- <u>Disks</u>
    - SATA0 - *WD Blue HDD 500GB ---> System*
    - SATA1 - *Crucial MX300 SSD 1TB ---> VMs and Containers*
    - SATA2 - *WD Red Plus HDD 4TB ---> TrueNAS*
    - SATA3 - *WD Red Plus HDD 4TB ---> TrueNAS*



## Proxmox Installation

Proxmox VE 9.1

[Proxmox VE Wiki](https://pve.proxmox.com/wiki/Main_Page)
- [System Requirements](https://pve.proxmox.com/wiki/System_Requirements)
- [Prepare Installation Media](https://pve.proxmox.com/wiki/Prepare_Installation_Media)
- [Installation](https://pve.proxmox.com/wiki/Installation)

[Proxmox VE Administrator Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)


## Adding *nomodeset* Kernel Parameter

Due to old video hardware, the installation hangs during boot, trying to switch to graphic mode.

By adding the ***nomodeset*** parameter, the Linux kernel is prevented from loading any graphics drivers, and the installation continues in text mode.

1. In the GRUB menu, select *"Install Proxmox VE (Terminal UI)"*. Do not press enter.
2. Press `e` to edit the entry
3. Add the parameter `nomodeset` at the end of the line starting with `linux`
4. Press `Ctrl-X` or `F10` to boot the configuration.



## Proxmox VE No-Subscription Repository

Proxmox offers two main types of repositories for its virtualization platform: the No-Subscription Repository and the Enterprise Repository

| Feature   | No-Subscription Repository                | Enterprise Repository           |
| --------- | ----------------------------------------- | ------------------------------- |
| Access    |	Free for all users                      | Requires an active subscription |
| Stability | Community-tested, may have less stability | Heavily tested, recommended for production use |
| Updates   | Regular updates, but may not be as timely | Immediate access to important security fixes |
| Support   | Community support only                    | Enterprise-grade technical support available |

Additional repositories:

- Base Debian repositories - *Required*
- Ceph Repositories - *For users utilizing Ceph. Accessible if the Proxmox VE node has a valid subscription of any level*
- Promox VE Test Repository - *Contains the latest packages and is primarily used by developers to test new features*

### Adding No-Subscription Repository in Proxmox 9 Using Web Interface

1. Access the Proxmox Web Interface (`https://<Node>:8006/`)

2. Navigate to *Datacenter > Node > Updates > Repositories*

3. Disable the Proxmox VE Enterprise Repository
   - Select the entry for the enterprise repository - *Components* column with value `pve-enterprise`
   - Click on the *Disable* button at the top

4. Add the Proxmox VE No-Subscription Repository
   - Click on the *Add* button at the top
   - In the dialog that appears, from the *Repository:* drop-down list select `No-Subscription`

5. Additionally, disable Ceph Repository
   - Select the entry for the ceph repository - *Components* column with value `enterprise`
   - Click on the *Disable* button at the top



## Disabling High Availability (HA) Logging

When running a single-node setup, it is often recommended to disable unused services to reduce unnecessary resource usage. Disabling the following services is generally safe if you are not using High Availability (HA) features.

Disabling `pve-ha-lrm.service` and `pve-ha-crm.service` in Proxmox:

1. Open the Terminal: Access your Proxmox server via SSH or directly through the console.

2. Stop the Services: Run the following commands to stop the services:

    ```bash
    systemctl stop pve-ha-lrm
    systemctl stop pve-ha-crm
    ```

3. Disable the Services: To prevent them from starting on boot, execute:

    ```bash
    systemctl disable pve-ha-lrm
    systemctl disable pve-ha-crm
    ```
    
    To check if the services are disabled, use:

    ```bash
    systemctl status pve-ha-lrm
    systemctl status pve-ha-crm
    ```
    


## Add LVM-Thin Volume for Virtual Machines 

VMs and containers will be installed on the 1TB SSD disk.

1. Access the Proxmox Web Interface (`https://<Node>:8006/`)

2. Initialize disk
   - Navigate to *Datacenter > Node > Disks*
   - Select the SSD disk in the list of available disks (e.g. /dev/sdb)
   - Click on the *Wipe* button at the top
   - Alternatively, in the Proxmox shell run:
     ```bash
     wipefs -a /dev/sdb
     ```

3. Create a Physical Volume
   - In the Proxmox shell execute: 
     ```bash
     pvcreate /dev/sdb
     ```

4. Creating a Volume Group
   - To create a volume group named *pve-data*, execute the following command:
     ```bash
     vgcreate pve-data /dev/sdb
     ```

5. Creating a Thin Pool Logical Volume
   - To create a logical volume named *vm-data* using 100% of the available storage and with maximum size of metadata, execute the following command:
     ```bash
     lvcreate -l100%FREE --poolmetadatasize=16G -T -n vm-data pve-data
     ```

2. Add storage to Proxmox
   - Navigate to *Datacenter > Storage*
   - Click the *Add* button, and pick `LVM-Thin`
   - In the dialog set:
       - ID: `data-lvm`
       - Volume Group: `pve-data`
       - Thin Pool: `vm-data`
       - Content: `Disk image`, `Container`
       - Enable: `Checked`




## Delete local-lvm and Expand Root


Deleting local-lvm

1. In the Proxmox GUI, navigate to *Datacenter > Storage*

2. Select `local-lvm` and click on the *Remove* button.

Expanding Root Partition

1. Click on the Proxmox node and then select *Shell*

2. Enter the following commands to remove the local-lvm and resize the root partition:
   
| Command                  | Description                  |
| ------------------------ | ---------------------------- |
| `lvremove /dev/pve/data` | Removes the local-lvm volume |
| `lvresize -l +100%FREE /dev/pve/root` | Resizes the root partition to use all available free space |
| `resize2fs /dev/mapper/pve-root` | Resizes the filesystem to match the new partition size |



## Configure Network Link Aggregation

Why Configure Link Aggregation in Proxmox?

- Increase Bandwidth: Aggregate multiple Gigabit Ethernet connections (e.g., 2 x 1Gbps = 2Gbps)

- Redundancy: If one link fails, others remain operational

- Traffic Load Balancing: Distribute traffic efficiently across links


Prerequisites

- Physical Interfaces: At least two physical network interfaces on the Proxmox server
    - 2 x Intel WG82574L Gigabit.

- Switch Configuration: Netgear GS305EP switch 
    - Supports a single static link aggregation group (LAG) with up to four ports
    - The switch **does not** support Link Aggregation Control Protocol (LACP)


### Configuration using the Web UI

1. In the Proxmox GUI, navigate to *Datacenter > Node > Network*

2. Open the Linux Bridge `vmbr0` 
   - Delete the configured nic in the *Bridge ports:* field
   - Click *OK*. Do not Apply Configuration.

3. Create Linux Bond
   - Click on *Create* and select *Linux Bond* from the dropdwon list
   - In the *Create: Linux Bond* dialog:
      - Name: `bond0`
      - Slaves: `nic1` `nic2`
      - Mode: `balance-xor`
      - Hash policy: `layer2+3`
   - Click *Create*. Do not Apply Configuration

4. Go back to Linux Bridge `vmbr0`
   - In the *Edit: Linux Bond* dialog:
      - Bridge ports: `bond0`
   - Click *OK*

5. Configure and enable Link Aggregation Group in the switch

6. Go back to the Network settings in Proxmox
   - Click on the **Apply Configuration** button

### Configuration using the shell

1. Open the Terminal: Access your Proxmox server via SSH or directly through the console.

2. Backup current network configuration
    ```bash
    cp /etc/network/interfaces /etc/network/interfaces.bak
    ```

3. Update configuration in `/etc/network/interfaces`
   - Configure an interface called bond0 and setup the bond interface with corresponding slaves, mode, hashing policy, etc
   - On `vmbr0` bridge, define the bridge-ports to be `bond0`
   
    ```bash
    auto lo
    iface lo inet loopback
    
    iface enp7s0 inet manual
    
    iface enp8s0 inet manual
    
    auto bond0
    iface bond0 inet manual
            bond-slaves enp7s0 enp8s0
            bond-miimon 100
            bond-mode balance-xor
            bond-xmit-hash-policy layer2+3
    
    auto vmbr0
    iface vmbr0 inet static
            address 10.1.1.88/24
            gateway 10.1.1.1
            bridge-ports bond0
            bridge-stp off
            bridge-fd 0
    
    source /etc/network/interfaces.d/*
    ```

4. Configure and enable Link Aggregation Group in the switch

5. Go back to the Proxmox shell
   - Apply the new interface settings by executing:
   
    ```bash
    ifreload -a
    ``` 


## Disable Proxmox Subscription Warning

The Proxmox subscription warning appears if you use the platform without a paid license.

THE OFFICIAL METHOD TO REMOVE THE POPUP IS TO BUY A SUBSCRIPTION

1. Open the Terminal: Access your Proxmox server via SSH or directly through the console.

2. Change Directory:
    ```bash
    cd /usr/share/javascript/proxmox-widget-toolkit
    ```

3. Back Up Original File:
    ```bash
    cp proxmoxlib.js proxmoxlib.js.bak
    ```

4. Open `proxmoxlib.js` with an editor
    ```bash
    vi proxmoxlib.js
    ```

5. Disable popup logic
   - Search for "Ext.Msg.show({"
   - Replace with "void({"

6. Save changes

7. Restart `pveproxy` to apply changes
    ```bash
    systemctl restart pveproxy
    ```


Note: Every time Proxmox VE upgrade packages (apt upgrade), new versions may overwrite proxmoxlib.js and/or file paths



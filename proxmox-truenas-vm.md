# Installing TrueNAS on Proxmox


## Download TrueNAS Community Edition Installer

1. Go to TrueNAS Community Edition page
   
   `https://www.truenas.com/truenas-community-edition/`

2. Follow the link to Download Community Edition

3. You'll be asked to subscribe. Skip and click on `No Thanks, I’ve Already Signed Up.`

4. Under Install TrueNAS, copy the link of the version to install (e.g. 25.10.2.1)
   
   `https://download.sys.truenas.net/TrueNAS-SCALE-Goldeye/25.10.2.1/TrueNAS-SCALE-25.10.2.1.iso`

5. Switch to Proxmox web interface
    - Navigate to *Datacenter > Node > local > ISO Images*
    - Click on `Download from URL`

6. In the *Download from URL* dialog
    - Enter the download URL in the `URL` field
    - Click on the `Query URL` button to obtain metadata
    - Click on `Download`



## Minimum Hardware Requirements

|             |                                                           |
| ----------- | --------------------------------------------------------- |
| Processor   | *2-Core Intel 64-Bit or AMD x86_64 processor*             |
| Memory      | *8 GB memory*                                             |
| Boot Device | *16 GB SSD*                                               |
| Storage     | *Two identically-sized devices for a single storage pool* |
|             |                                                           |


## Create TrueNAS virtual machine

1. Access the Proxmox Web Interface (`https://<Node>:8006/`)

2. Click on the `Create VM` button at the top right

3. In the *Create: Virtual Machine* dialog:
   - General section:
      - Node: `<Node>`
      - VM ID: `101`
      - Name: `<VM_Name>`
      - Start at boot: `Checked`

   - OS section:
      - Storqge: `local`
      - ISO Image: `<Select downloaded TrueNAS ISO>`

   - System section:
      - Leave defauts

   - Disks section:
      - Storage: `data-lvm`
      - Disk size: `32GB`
      - SSD Emulation: `Checked`

   - CPU section:
      - Sockets: `1`
      - Cores: `2`

   - Memory section:
      - Memory: `8192MB`

   - Network section:
      - Leave defaults



## Install TrueNAS

1. Start the VM

2. Open the console

3. After the system boots into the installer, follow the steps in the documentation

    [Using the TrueNAS Installer in a Virtual Machine](https://www.truenas.com/docs/scale/25.10/gettingstarted/install/installingscale/#expand-20)

4. The VM will restart and boot directly into TueNAS

5. The console will display the URL for the web interface
   
   ```
   https:<ip_address
   https://ip_address
   ```
   
6. Open a browser and enter the IP address shown and login
   - Username: `truenas_admin`
   - Password: `<set during installation>`



## Configuring TrueNAS network settings

1. Login to TrueNAS GUI

2. Navigate to *System > Network*

3. In the *Network Configuration* section, click on *Settings*:
   - In *Edit Global Configuration*, set/update *Hostname*, *Domain Name*, *DNS Servers* and *Default Gateway*
   - Click on *Save*

4. In the *Interfaces* section, click on the *Actions* menu to the right of the network interface, and select *Edit*
   - In the *DHCP* section, select *Define Static IP Addresses*
   - In *Static IP Addresses* select *Add*
   - Enter IP address and network mask
   - Click on *Save*

5. Back to Network Settings, you are prompted if you want to test the changes:
   - Click on *Test Changes* and confirm in the next window
    Changes are applied for 60 seconds and reverted if the new configuration is not saved by connecting to the new IP address

6. Connect to the TrueNAS interface with the new IP address
   - Login with TrueNAS admin credentials
   - Go to Network Settings
   - Click on *Save Changes* and confirm in the next window
 
Note: If access is lost after changing IP address, you may need to revert to DHCP or access the system directly to correct the settings. 



## Attach disks to TrueNAS

TrueNAS requires direct access to the hard disks

1. In the Proxmox GUI, navigate to *Datacenter > <Node> > Disks*

2. Take note of the model and serial number of the disks to be assigned to TrueNAS VM

3. Retrieve the disk IDs
   - Open the Proxmox shell
   - Execute the following command to list all disk IDs
      ```bash
      root@chaos:~# ls /dev/disk/by-id

      ata-Crucial_CT1050MX300SSD1_164114392C1A
      ata-WDC_WD40EFRX-68WT0N0_WD-WCC4E0462725    <---
      ata-WDC_WD40EFRX-68WT0N0_WD-WCC4E0478458    <---
      ata-WDC_WD5000AAKS-00UU3A0_WD-WCAYU7607646
      ```
   - Save the corresponding disk IDs (matching model and serial previously obtained)

4. In Proxmox, open *Hardware* settings of TrueNAS VM
   - Verify there's one disk in scsi0
   - Remove *CD/DVD Drive*

5. In Proxmox shell, attach disks to TrueNAS VM
    ```bash
    qm set 101 -scsi1 /dev/disk/by-id/ata-WDC_WD40EFRX-68WT0N0_WD-WCC4E0478458
    
    qm set 101 -scsi2 /dev/disk/by-id/ata-WDC_WD40EFRX-68WT0N0_WD-WCC4E0462725
    ```

6. Back to *Hardware* settings of TrueNAS VM
   - Verify disks were added (scsi1 and scsi2)

7. In TrueNAS GUI, go to the *Storage* section and click on *Disks*

    | Name | Serial      | Disk Size | Pool      | Disk Type |
    | ---- | ----------- | --------- | --------- | --------- |
    | sda  | drive-scsi0 | 32 GiB    | boot-pool | SSD       |
    | sdb  | drive-scsi1 | 3.64 TiB  | N/A       | HDD       |
    | sdc  | drive-scsi2 | 3.64 TiB  | N/A       | HDD       |


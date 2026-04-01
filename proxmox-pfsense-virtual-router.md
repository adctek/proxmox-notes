# Create a Virtual Router on Proxmox with pfSense 

## Download pfSense Installation ISO

1. Go to pfSense download page
   
   `https://www.pfsense.org/download/`

2. Follow the link to Download the latest stable version

3. Redirected to Netgate site
   - Under *Installation Image* select `AMD64 ISO IPMI/Virtual Machines`
   - Quantity: `1`
   - Click on *Add to Cart*

4. Go to the Cart page and click on *Checkout*
   - You'll be asked to login, or create an account to continue
   - Complete the checkout process
   - Wait for the download link to be generated, then download the compressed file

5. Extract the ISO file to a local folder

6. In to Proxmox web interface
    - Navigate to *Datacenter > Node > local > ISO Images*
    - Click on `Upload`

7. In the *Upload* dialog
    - Click on `Select File` and locate the extracted pfSense ISO file
    - Click on `Upload`


## Minimum Hardware Requirements

|             |                                                  |
| ----------- | ------------------------------------------------ |
| Processor   | *64-bit amd64 (x86-64) compatible CPU*           |
| Memory      | *1GB RAM*                                        |
| Disk Size   | *8GB*                                            |
| Network     | *One or more compatible network interface cards* |
|             |                                                  |



## Product Documentation

[pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/)

[Virtualizing pfSense with Proxmox VE](https://docs.netgate.com/pfsense/en/latest/recipes/virtualize-proxmox-ve.html)



## Prepare Proxmox VE Networking

Two Linux Bridges should be configured on Proxmox VE, which will be used for LAN and WAN on the pfSense VM

- vmbr0 - WAN interface
- vmbr1 - LAN / VLANs interface

1. In the Proxmox GUI, navigate to *Datacenter > Node > Network*

2. Create new Linux Bridge `vmbr1`
   - Click on *Create* and select *Linux Bridge* from the dropdwon list
   - In the *Create: Linux Bridge* dialog:
      - Name: `vmbr1`
      - VLAN aware: `Checked`
   - Click *Create*

3. Click on **Apply Configuration** to configure the new interface, and confirm by selexting *Yes*



## Create pfSense Virtual Machine

1. In the Proxmox GUI, Click on the `Create VM` button at the top right

2. In the *Create: Virtual Machine* dialog:
   - General section:
      - Node: `<Node>`
      - VM ID: `102`
      - Name: `<VM_Name>`
      - Start at boot: `Checked`

   - OS section:
      - Storqge: `local`
      - ISO Image: `<Select pfSense ISO>`

   - System section:
      - Graphic card: `SPICE`

   - Disks section:
      - Storage: `data-lvm`
      - Disk size: `32GB`
      - SSD Emulation: `Checked`

   - CPU section:
      - Sockets: `1`
      - Cores: `1`

   - Memory section:
      - Memory: `1024MB`

   - Network section:
      - Bridge: `vmbr0`
      - Model: `VirtIO (paravirtualized)`

3. Review the settings in the *Confirm* section and click *Finish*

    Wait for the VM creation process to finish

4. Expand *Datacenter > Node* and select newly created virtual machine

4. Add second network interface
   - Click *Hardware* in the right pane
   - Click *Add*, and from the dropdown list select `Network Device`
      - Bridge: `vmbr1`
      - Model: `VirtIO (paravirtualized)`



## Install pfSense

1. Start the VM

2. Open the console

3. After the system boots into the installer, follow the steps in the documentation

    [Installation Walkthrough](https://docs.netgate.com/pfsense/en/latest/install/install-walkthrough.html)

    - WAN: `vtnet0`
      - Interface Mode: `STATIC`
      - VLAN Settings: `VLAN Tagging disabled`
      - IP Address: `<IP_Address>`
      - Default Gateway: `<Default_Gateway>`
      - DNS Server: `<DNS_Server>`
    - LAN: `vtnet1`
      - Interface Mode: `STATIC`
      - VLAN Settings: `VLAN Tagging disabled`
      - IP Address: `<Internal_IP_Address>`
      - DHCPD Enabled: `true`
      - DHCPD Range Start: `<Internal_DHCP_Range_Start>`
      - DHCPD Range End: `<Internal_DHCP_Range_End>`

4. Reboot when finished



## Initial Configuration

1. Login through the GUI¶
   - Connect a client computer to the same network as the LAN interface.
   - Open a web browser and navigate to `Internal_IP_Address` specified during installation (`https://192.168.1.1`)
   - Enter the default credentials in the login page:
      - Username: `admin`
      - Password: `pfsense`

2. Complete the Setup Wizard. Follow the steps in the documentation

    [Setup Wizard](https://docs.netgate.com/pfsense/en/latest/config/setup-wizard.html)

3. Configure interfaces

    [Interface Configuration](https://docs.netgate.com/pfsense/en/latest/config/interface-configuration.html)

    - Since WAN interface is on `vmbr0`, and was configured with a private IP address:
      - IPv4 Upstream Gateway: `<Default_Gateway_IP_Address>`
      - Block private networks and loopback addresses: `Uncheck`

4. Configure firewall rules (Optional)
   
    To allow ping from the local network to the firewall and the subnet to which the LAN interface is connected:
      - Navigate to *Firewall > Rules*
      - Select the `WAN` tab 
      - Add rule to allow ICMP traffic from the local network

        ```bash
        # WAN Interface
        $ ping 10.1.1.10 -c 3
        PING 10.1.1.10 (10.1.1.10) 56(84) bytes of data.
        64 bytes from 10.1.1.10: icmp_seq=1 ttl=64 time=3.38 ms
        64 bytes from 10.1.1.10: icmp_seq=2 ttl=64 time=1.93 ms
        64 bytes from 10.1.1.10: icmp_seq=3 ttl=64 time=3.32 ms
        
        --- 10.1.1.10 ping statistics ---
        3 packets transmitted, 3 received, 0% packet loss, time 2002ms
        rtt min/avg/max/mdev = 1.932/2.876/3.379/0.668 ms
        
        # VM in the LAN subnet
        $ ping 192.168.1.100 -c 3
        PING 192.168.1.100 (192.168.1.100) 56(84) bytes of data.
        64 bytes from 192.168.1.100: icmp_seq=1 ttl=63 time=2.45 ms
        64 bytes from 192.168.1.100: icmp_seq=2 ttl=63 time=3.92 ms
        64 bytes from 192.168.1.100: icmp_seq=3 ttl=63 time=4.56 ms

        --- 192.168.1.100 ping statistics ---
        3 packets transmitted, 3 received, 0% packet loss, time 2003ms
        rtt min/avg/max/mdev = 2.446/3.639/4.555/0.883 ms
        ```
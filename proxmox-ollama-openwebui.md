
# Ollama with NVIDIA GPU in Proxmox VM and LXC container

The setup has two components:

- Headless VM with Ollama for AI model serving [--->](#ollama_vm)

- LXC container with OpenWebUI for management and user access



<a id="ollama_vm"></a>
# Run VM with Ollama using GPU Passthrough

Prerequisites:

- IOMMU enabled in the Proxmox server BIOS
- GPU card(s) supported by Ollama

**Note:** *The ASUS Z9PA-D8 motherboard used is not listed as supporting IOMMU, but the setup worked just the same*  `¯\_(ツ)_/¯`

## Configure GPU Passthrough in Proxmox

Step 1 - Enable IOMMU in GRUB:

```bash
## Open GRUP Configuration File
nano /etc/default/grub

## Add or alter the CMDLINE line to include the following
GRUB_CMDLINE_LINUX_DEFAULT="quiet iommu=on iommu=pt pcie_acs_override=downstream,multifunction"

## Save the file then update GRUB
update-grub

## Don't reboot yet. Wait after adding VFIO Kernel Modules
```

Step 2 - Load the VFIO Kernel Modules

If the GPU drivers are already installed on the Proxmox server, you'll need to blacklist then to keep Proxmox from using them:

```bash
## Create the blacklist file
nano /etc/modprobe.d/blacklist.conf

## Add relevant drivers
## For AMD
blacklist radeon
## For NVIDIA
blacklist nouveau
blacklist nvidia
blacklist nvidiafb
blacklist rivafb
```

Add the VFIO modules

```bash
## Open the modules file
nano /etc/modules

## Add all four lines below
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd

## Save the file then reboot to update both GRUB and kernel modules
```

Step 3 - Verify IOMMU configuration status

```bash
root@chaos:~# dmesg | grep -e IOMMU -e DMAR
[    0.017209] ACPI: DMAR 0x000000007C9F6300 0000E0 (v01 A M I  OEMDMAR  00000001 INTL 00000001)
[    0.017259] ACPI: Reserving DMAR table memory at [mem 0x7c9f6300-0x7c9f63df]
[    0.207585] DMAR: IOMMU enabled
[    0.207613] DMAR: IOMMU enabled
[    0.536593] DMAR: Host address width 46
[    0.536595] DMAR: DRHD base: 0x000000fbffe000 flags: 0x0
[    0.536616] DMAR: dmar0: reg_base_addr fbffe000 ver 1:0 cap d2078c106f0466 ecap f020da
[    0.536621] DMAR: DRHD base: 0x000000cfffc000 flags: 0x1
[    0.536627] DMAR: dmar1: reg_base_addr cfffc000 ver 1:0 cap d2078c106f0466 ecap f020da
[    0.536630] DMAR: RMRR base: 0x0000007e96b000 end: 0x0000007e981fff
[    0.536634] DMAR: RHSA base: 0x000000fbffe000 proximity domain: 0x1
[    0.536636] DMAR: RHSA base: 0x000000cfffc000 proximity domain: 0x0
[    0.536640] DMAR-IR: IOAPIC id 3 under DRHD base  0xfbffe000 IOMMU 0
[    0.536643] DMAR-IR: IOAPIC id 0 under DRHD base  0xcfffc000 IOMMU 1
[    0.536645] DMAR-IR: IOAPIC id 2 under DRHD base  0xcfffc000 IOMMU 1

```

Step 4 - Identify GPU and additional functions 

```bash
root@chaos:~# lspci -nn | grep -i nvidia
03:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU116 [GeForce GTX 1650 SUPER] [10de:2187] (rev a1)
03:00.1 Audio device [0403]: NVIDIA Corporation TU116 High Definition Audio Controller [10de:1aeb] (rev a1)
03:00.2 USB controller [0c03]: NVIDIA Corporation TU116 USB 3.1 Host Controller [10de:1aec] (rev a1)
03:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU116 USB Type-C UCSI Controller [10de:1aed] (rev a1)


## Take note the addresses and check all devices are in the same IOMMU group and it's not shared with other devices
find /sys/kernel/iommu_groups/ -type l


## Another way to find the IOMMU group
lspci -v
.
.
.
03:00.3 Serial bus controller: NVIDIA Corporation TU116 USB Type-C UCSI Controller (rev a1)
        Subsystem: Micro-Star International Co., Ltd. [MSI] Device 3850
        Flags: bus master, fast devsel, latency 0, IRQ 65, NUMA node 0, IOMMU group 24
        Memory at c3084000 (32-bit, non-prefetchable) [size=4K]
        Capabilities: [68] MSI: Enable+ Count=1/1 Maskable- 64bit+
        Capabilities: [78] Express Endpoint, IntMsgNum 0
        Capabilities: [b4] Power Management version 3
        Capabilities: [100] Advanced Error Reporting
        Kernel driver in use: vfio-pci
        Kernel modules: i2c_nvidia_gpu
```


## Create Ollama VM

1. Create a Linux VM in Proxmox. My VM has the following settings:

    - BIOS: `Default(SeaBIOS)`  <-- Since the VM will not use the GPU as the main display adapter
    - Processors: `1 CPU, 8 cores`
    - Memory: `16GB`
    - Hard Disk: `100GB`
    - Display: `Default`
    - Network: `Default`

2. Add the PCI passthrough device
   - In Proxmox, go to *Datacenter > Node > VM > Hardware*
   - Click *Add*, the select `PCI Device` from the dropdown
   - In the *Add: PCI Device* dialog, select `Raw Device`
   - In the *Device* dropdown, look for the GPU ID and select it.
   - Check `All Functions` 
   - Click on *Add*

3. Start the VM and install the operating system
   
   - Note: *Pop_OS! failed, installed Kubuntu*

4. Install NVIDIA drivers
   
   - Some installations may detect and install the NVIDIA drivers during install process

    ```bash
    ## Determine your NVIDIA graphics card model and the recommended driver
    ubuntu-drivers devices

    == /sys/devices/pci0000:00/0000:00:10.0 ==
    modalias : pci:v000010DEd00002187sv00001462sd00003850bc03sc00i00
    vendor   : NVIDIA Corporation
    model    : TU116 [GeForce GTX 1650 SUPER]
    driver   : nvidia-driver-590-open - distro non-free
    driver   : nvidia-driver-590-server-open - distro non-free
    driver   : nvidia-driver-535-open - distro non-free
    driver   : nvidia-driver-535-server-open - distro non-free
    driver   : nvidia-driver-580-server - distro non-free
    driver   : nvidia-driver-570 - distro non-free
    driver   : nvidia-driver-570-server - distro non-free
    driver   : nvidia-driver-580 - distro non-free
    driver   : nvidia-driver-580-server-open - distro non-free
    driver   : nvidia-driver-470 - distro non-free
    driver   : nvidia-driver-570-server-open - distro non-free
    driver   : nvidia-driver-580-open - distro non-free recommended
    driver   : nvidia-driver-590-server - distro non-free
    driver   : nvidia-driver-535 - distro non-free
    driver   : nvidia-driver-470-server - distro non-free
    driver   : nvidia-driver-590 - distro non-free
    driver   : nvidia-driver-570-open - distro non-free
    driver   : nvidia-driver-535-server - distro non-free
    driver   : xserver-xorg-video-nouveau - distro free builtin
    
    ## Install NVIDIA driver
    sudo apt update
    sudo apt install -y nvidia-driver-590
    
    ## Reboot
    sudo reboot
    
    ## Verify driver installation
    nvidia-smi
        
    +-----------------------------------------------------------------------------------------+
    | NVIDIA-SMI 590.48.01              Driver Version: 590.48.01      CUDA Version: 13.1     |
    +-----------------------------------------+------------------------+----------------------+
    | GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
    | Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
    |                                         |                        |               MIG M. |
    |=========================================+========================+======================|
    |   0  NVIDIA GeForce GTX 1650 ...    Off |   00000000:00:10.0 Off |                  N/A |
    |  0%   38C    P8              5W /  100W |      14MiB /   4096MiB |      0%      Default |
    |                                         |                        |                  N/A |
    +-----------------------------------------+------------------------+----------------------+
    
    +-----------------------------------------------------------------------------------------+
    | Processes:                                                                              |
    |  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
    |        ID   ID                                                               Usage      |
    |=========================================================================================|
    |    0   N/A  N/A            1177      G   /usr/lib/xorg/Xorg                        4MiB |
    +-----------------------------------------------------------------------------------------+
    ```

5. Install Ollama
   
    ```bash
    ## From Ollama official website
    curl -fsSL https://ollama.com/install.sh | sh

    ## Test installation
    ollama run llama3

    >>> Hello
    Hello! It's nice to meet you. Is there something I can help you with, or would you like to chat?
    
    >>> 'Send a message (/? for help)'
    
    >>> /bye
    ```

6. Make the Ollama API available to LAN clients

    By default Ollama only listens on `127.0.0.1:11434`. To make it available for network clients to connect in, you need to modify the `OLLAMA_HOST` environment variable of the service.

    ```bash
    ## Edit the Ollama service file
    sudo vi /etc/systemd/system/ollama.service
    
    ## Modify the following line
    [Service]
    Environment="OLLAMA_HOST=0.0.0.0:11434"
    
    ## Restart the service
    sudo systemctl daemon-reload
    sudo systemctl restart ollama
    
    ## Verify the service is listening on all interfaces
    ss -tulnp | grep 11434
    tcp   LISTEN 0      4096               *:11434            *:*  
    
    ## Allow connections to Ollama API port (if needed)
    sudo ufw allow 11434/tcp
    ```




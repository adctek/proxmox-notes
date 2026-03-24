# Reset Proxmox Root Password via GRUB

1. Reboot the Proxmox Server

    Restart the Proxmox machine from the terminal or via remote management.

2. Interrupt the GRUB Bootloader

    In the boot menu screen, instead of pressing enter, press `e` to edit the default boot entry.

3. Modify the Boot Configuration

    At the end of the line starting with linux (e.g., linux /boot/vmlinuz...), append the following, leaving a space from the previous parameter:

    ```bash
    init=/bin/bash
    ```

4. Boot the System

    Press `Ctrl + X` or `F10` to boot with the added parameters.

5. Remount Root Filesystem as Read/Write

    The system will boot in console mode.

    At the shell prompt, execute:

    ```bash
    mount -o remount,rw /
    ```

6. Change the Root Password

    Execute the following command:

    ```bash
    passwd
    ```

    You will be asked to enter your new password and confirm it.

7. Reboot the System

    To reboot your system, execute:

    ```bash
    reboot
    ```

    If you get an error, try

    ```bash
    exec /sbin/init
    ```

    Then login using the newly set password.



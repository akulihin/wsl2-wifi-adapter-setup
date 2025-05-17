# WSL2 WiFi Adapter Setup Guide

> **Disclaimer**: This README was edited and restructured by an AI model to improve readability and organization. While the technical content remains accurate, the formatting and structure have been enhanced for better user experience.

This guide will help you set up a WiFi adapter in WSL2 by building the official Microsoft WSL2 kernel with USB/IP support enabled and configuring USB device sharing between Windows and WSL2.

## Table of Contents

1. [Building WSL2 Kernel with USB/IP Support](#building-wsl2-kernel-with-usbip-support)
2. [Installing USB/IP Tools](#installing-usbip-tools)
3. [Sharing USB Device with WSL2](#sharing-usb-device-with-wsl2)
4. [Verifying Setup](#verifying-setup)
5. [Installing WiFi Adapter Driver](#installing-wifi-adapter-driver)
6. [Using WiFi Adapter](#using-wifi-adapter)

## Building WSL2 Kernel with USB/IP Support

> **Reference**: For official Microsoft documentation on building the WSL2 kernel, visit [Microsoft Learn: WSL User MSFT Kernel v6](https://learn.microsoft.com/en-us/community/content/wsl-user-msft-kernel-v6)

This section guides you through building the official Microsoft WSL2 kernel from source with USB/IP support enabled. We're using Microsoft's official kernel repository, not a custom or modified kernel.

### Step 1: Download the Kernel Source

You have two options:
1. Download the kernel version you're currently using (run `uname -r` in WSL2)
2. Download the latest kernel version (Recommended)

To download the latest kernel:
> **Note**: It is recommended to run git within WSL2 filesystem, not in Windows. `/mnt/c/` is the Windows filesystem. `/home/` is the WSL2 filesystem.

```bash
# Navigate to your home directory
mkdir ~/wifi
cd ~/wifi

# Clone the kernel repository
git clone https://github.com/microsoft/WSL2-Linux-Kernel.git --depth=1
```

> **Note**: If you need a specific kernel version, use:
> ```bash
> git clone https://github.com/microsoft/WSL2-Linux-Kernel.git --depth=1 -b linux-msft-wsl-6.6.y
> ```
> Replace `linux-msft-wsl-6.6.y` with your desired branch name.

### Step 2: Install Build Dependencies

Install the required packages:
```bash
sudo apt update && sudo apt upgrade && sudo apt install build-essential flex bison libssl-dev libelf-dev bc python3 pahole cpio
```

### Step 3: Build the Kernel

Build the kernel:
```bash
# Navigate to the kernel directory
cd WSL2-Linux-Kernel

# Build the kernel (this may take some time)
make -j$(nproc) KCONFIG_CONFIG=Microsoft/config-wsl

# Install kernel modules and headers
sudo make modules_install headers_install

# Copy the kernel image to Windows (replace 'myusername' with your Windows username)
cp arch/x86/boot/bzImage /mnt/c/Users/myusername/
```

### Step 4: Configure WSL2 to Use Custom Kernel

#### In PowerShell:

1. Open the WSL2 configuration file:
   ```powershell
   notepad %USERPROFILE%\.wslconfig
   ```

2. Add the following configuration (replace 'myusername' with your Windows username):
   ```ini
   [wsl2]
   kernel=C:\\Users\\myusername\\bzImage
   ```

3. Restart WSL2:
   ```powershell
   wsl --shutdown
   ```

#### In WSL2:

4. Verify the new kernel is active:
   ```bash
   uname -r
   ```
   Example output:
   ```
   6.6.87.1-microsoft-standard-WSL2+
   ```

## Installing USB/IP Tools

> **Reference**: [usbipd-win GitHub Repository](https://github.com/dorssel/usbipd-win)

### Step 1: Install usbipd-win

#### In PowerShell (as Administrator):

Install the USB/IP daemon for Windows:
```powershell
# Install usbipd using winget
winget install usbipd
```

## Sharing USB Device with WSL2

### Step 1: List Available USB Devices

#### In PowerShell (as Administrator):

List all USB devices:
```powershell
# List all connected USB devices
usbipd list
```

Example output:
```
Connected:
BUSID  VID:PID    DEVICE                                                        STATE
2-17   abcd:0001  Realtek USB Audio, USB Input Device                           Not shared
3-12   ehfc:0002  Realtek 8812AU Wireless LAN 802.11ac USB NIC                  Not shared
```

### Step 2: Bind USB Device

#### In PowerShell (as Administrator):

Bind your WiFi adapter to WSL2:
```powershell
# Bind the device using its VID:PID
usbipd bind --hardware-id <VID:PID>
```

Replace `<VID:PID>` with the VID:PID of your WiFi adapter from the previous step.

Example:
```powershell
usbipd bind --hardware-id ehfc:0002
```

### Step 3: Attach Device to WSL2

#### In PowerShell (as Administrator):

Attach the device to WSL2:
```powershell
# Attach the device to WSL2
usbipd attach --hardware-id <VID:PID> --wsl -a
```

> **Note**: If you have multiple WSL distributions, you can specify which one to use:
> ```powershell
> usbipd attach --hardware-id <VID:PID> --wsl <distribution-name> -a
> ```

Example:
```powershell
usbipd attach --hardware-id ehfc:0002 --wsl -a
```

## Verifying Setup

### Check USB Device Connection

#### In WSL2:

Verify the USB device is properly attached:
```bash
# List USB devices
lsusb
```

You should see your WiFi adapter listed in the output. Example output:
```
$ lsusb
Bus 002 Device 001: ID abcd:0001 Linux Foundation 3.0 root hub
Bus 001 Device 003: ID ehfc:0002 Realtek Semiconductor Corp. RTL8812AU 802.11a/b/g/n/ac 2T2R DB WLAN Adapter
Bus 001 Device 001: ID abcd:0003 Linux Foundation 2.0 root hub
```

## Installing WiFi Adapter Driver

### Initial Check

#### In WSL2:

Verify that WiFi adapter is working:
```bash
# Check network interfaces
ifconfig -a
```

If you see only "lo" and "eth0" interfaces, you need to install the WiFi adapter driver.

> **Important**: These driver installation steps are specific to the RTL8812AU adapter using [aircrack-ng/rtl8812au repository](https://github.com/aircrack-ng/rtl8812au). For other adapters, you'll need to find and install the appropriate drivers for your specific hardware.

### Driver Installation Steps (RTL8812AU Example)

#### In WSL2:

1. Update system:
   ```bash
   # Update package lists and upgrade existing packages
   sudo apt update && sudo apt upgrade
   ```

2. Install required packages:
   ```bash
   # Install git and DKMS (Dynamic Kernel Module Support)
   sudo apt install git dkms
   ```

3. Clone and install the driver:
   ```bash
   # Clone the driver repository
   git clone https://github.com/aircrack-ng/rtl8812au.git
   cd rtl8812au
   
   # Install the driver using DKMS
   sudo make dkms_install
   ```

4. Physically detach and reattach the WiFi adapter to your USB port.

5. Verify if the WiFi adapter is now visible in WSL2:
   ```bash
   # Check network interfaces again
   ifconfig -a
   ```
   You should see a new interface! In my case it was `wlxREDACTED` (interface name contains the MAC address of the adapter).

## Using WiFi Adapter

### Setting Up Monitor Mode with aircrack-ng

> **Important**: Make sure you have permission to perform wireless network scanning in your jurisdiction. Some activities might be illegal in certain regions.

#### In WSL2:

1. Install aircrack-ng:
   ```bash
   # Install aircrack-ng suite
   sudo apt install aircrack-ng
   ```

2. Start monitor mode:
   ```bash
   # Enable monitor mode on your WiFi interface
   sudo airmon-ng start wlxREDACTED
   ```
   > **Note**: Usually the command renames the interface to `wlan0mon`, but in WSL2 it stays the same.

3. Scan for available WiFi networks:
   ```bash
   # Start scanning for WiFi networks
   airodump-ng wlxREDACTED
   ```


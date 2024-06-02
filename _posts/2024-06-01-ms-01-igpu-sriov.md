---
title: Minisforum MS-01 SR-IOV iGPU passthrough
date: 2024-06-01 17:00:00 +0000
categories: [Proxmox]
tags: [proxmox, gpu, passthrough, sriov, vm] 
---

A compilation of the notes/resources I used to virtualize the Intel iGPU and share it with multiple virtual machines. These notes work at the time of writing. I'd recommend checking the references at the bottom for any updated information. Credits go to the respective authors listed in the references.

## Environment
- Minisforum MS-01
- CPU: Intel® Core™ i9-13900H (14C/20T)
- RAM: 96GB Crucial SODIMM DDR5 
- GPU: Intel® Iris® Xe Graphics (96 Execution units)
- Host: Proxmox 8.2.2 (Kernel 6.5.13-3)
- Guest: Ubuntu 22.04.4 (Kernel 6.5.0-35)

## Proxmox Setup [^1]

### Downgrading and Pinning the Kernel
```
apt install proxmox-kernel-6.5.13-3-pve-signed
apt install proxmox-headers-6.5.13-3-pve
proxmox-boot-tool kernel pin 6.5.13-3-pve
```

To unpin the Kernel in the future:
```
proxmox-boot-tool kernel unpin 6.5.13-3-pve
```

### DKMS Setup
1. Update your system and install required packages:
```bash
apt update && apt install pve-headers-$(uname -r)
apt update && apt install git pve-headers mokutil
rm -rf /var/lib/dkms/i915-sriov-dkms*
rm -rf /usr/src/i915-sriov-dkms*
rm -rf ~/i915-sriov-dkms
KERNEL=$(uname -r); KERNEL=${KERNEL%-pve}
```

2. Proceed to clone the DKMS repository and adjust its configuration:
```bash
cd ~
git clone https://github.com/strongtz/i915-sriov-dkms.git
cd ~/i915-sriov-dkms
cp -a ~/i915-sriov-dkms/dkms.conf{,.bak}
sed -i 's/"@_PKGBASE@"/"i915-sriov-dkms"/g' ~/i915-sriov-dkms/dkms.conf
sed -i 's/"@PKGVER@"/"'"$KERNEL"'"/g' ~/i915-sriov-dkms/dkms.conf
sed -i 's/ -j$(nproc)//g' ~/i915-sriov-dkms/dkms.conf
cat ~/i915-sriov-dkms/dkms.conf
```

3. Install the DKMS module:
```bash
apt install --reinstall dkms -y
dkms add .
cd /usr/src/i915-sriov-dkms-$KERNEL
dkms status
```

4. Build the i915-sriov-dkms
```bash
dkms install -m i915-sriov-dkms -v $KERNEL -k $(uname -r) --force -j 1
dkms status
```

### GRUB Bootloader Setup
Backup and update the GRUB configuration:
```bash
cp -a /etc/default/grub{,.bak}
sudo sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT/c\GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt i915.enable_guc=3 i915.max_vfs=7"' /etc/default/grub
update-grub
update-initramfs -u -k all
apt install sysfsutils -y
```

### Create vGPU
Identify the PCIe bus number of the VGA card, usually **00:02.0**: 
```bash
lspci | grep VGA
```

Edit the sysfs configuration to enable the vGPUs. In this case I’m using **00:02.0**. To verify the file was modified, cat the file and ensure it was modified.
```bash
echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7" > /etc/sysfs.conf
cat /etc/sysfs.conf
```

### If you have EFI  **Enabled** and Secure Boot **Enabled**
You need to install mokutill for Secure Boot
```bash
apt update && apt install mokutil
mokutil --import /var/lib/dkms/mok.pub
```
**Reboot Proxmox Host**, monitor the boot process and wait for the **Perform MOK management** window (screenshot below). If you miss the first reboot you will need to re-run the mokutil command and reboot again. The DKMS module will NOT load until you step through this setup. 

Secure Boot MOK Configuration (Proxmox 8.1+)

Select **Enroll MOK, Continue, Yes, (password), Reboot.**


## Ubuntu Guest Setup

### Guest Configuration
- CPU: host
- BIOS: OVMF (UEFI)
- Machine: q35
- Disk: Cache - Write Through, Discard
- GPU: default
- PCI Devices: Don't add the VF just yet. We'll add it later.

### Updating the Kernel to 6.5.0-35
```
sudo apt install linux-generic-hwe-22.04
```

#### Holding the Kernel version on Ubuntu
```
sudo apt-mark hold linux-generic-hwe-22.04
sudo apt-mark unhold linux-generic-hwe-22.04
```

### DKMS Setup
1. Update your system and install required packages:
```
sudo apt update -y
sudo apt install git dkms build-* -y
sudo apt install build-* dkms
```

2. Proceed to clone the DKMS repository and adjust its configuration:
``` 
git clone https://github.com/strongtz/i915-sriov-dkms
cd ~/i915-sriov-dkms
cp -a ~/i915-sriov-dkms/dkms.conf{,.bak}
KERNEL=$(uname -r);
sed -i 's/"@_PKGBASE@"/"i915-sriov-dkms"/g' ~/i915-sriov-dkms/dkms.conf
sed -i 's/"@PKGVER@"/"'"$KERNEL"'"/g' ~/i915-sriov-dkms/dkms.conf
sed -i 's/ -j$(nproc)//g' ~/i915-sriov-dkms/dkms.conf
cat ~/i915-sriov-dkms/dkms.conf
```

3. Install the DKMS module:
```
apt install --reinstall dkms -y
dkms add .
cd /usr/src/i915-sriov-dkms-$KERNEL
dkms status
```

4. Build the i915-sriov-dkms
```
dkms install -m i915-sriov-dkms -v $KERNEL -k $(uname -r) --force -j 1
dkms status
```

### GRUB Bootloader Setup [^2]
Backup and update the GRUB configuration:
```
sudo nano /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="i915.enable_nuc=3"
sudo update-grub
sudo update-initramfs -u
```

> We don't need to include `i915.max_vfs=7` in the command line default for grub. We also don't need `sysfsutils` to create VFs since it's already been created.
You might also be prompted to do the MOK Configuration when rebooting. Repeat as you did with the Proxmox setup.
{: .prompt-info }

### Add the VF to the VM
1. Shutdown the VM. 
2. Set Display to none
3. Add PCI-E device. Select the VF and not the iGPU. Will have a device ID of 0000:00:02:*01*
4. Set it as the Primary GPU.

## Setting up Plex for Hardware Transcoding [^3]
I'm using Plex on docker, so will need to bind `/dev/dri` to the container.
In my Ansible Playbook, I include the below in the task:
```
    devices:
      - /dev/dri:/dev/dri
```
You can verify that it's using hardware transcoding in the Dashboard. When hardware acceleration is being used, you should see (hw) next to the Video format as shown above. 

## References
[^1]: [Upinel - PVE-Intel-vGPU](https://github.com/Upinel/PVE-Intel-vGPU)
[^2]: [Strongtz - i915 SRIOV DKMS](https://github.com/strongtz/i915-sriov-dkms?tab=readme-ov-file#linux-guest-installation-steps-tested-kernel-62)
[^3]: [Plex Docker - Hardware Transcoding](https://github.com/plexinc/pms-docker?tab=readme-ov-file#intel-quick-sync-hardware-transcoding-support)
# Local Environment

## Helpers

### Fix docker access on ubuntu
[Link](https://docs.docker.com/engine/install/linux-postinstall/)

Execute:
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
reboot
```

### Proxmox disk passthrough
[Video](https://www.youtube.com/watch?v=U-UTMuhmC1U)

1. Find the disk Id you need and copy it: 
```bash
ls -n /dev/disk/by-id/
```
Passthrough the disk with ID=DISK_ID to the virtual machine with ID=VM-ID: 
```bash
/sbin/qm set [VM-ID] -virtio2 /dev/disk/by-id/[DISK-ID]
```

### Mount disk 
[Video](https://www.youtube.com/watch?v=a3QTaV4Cg7M)

1. Made a directory to mount disk to: 
```bash
mkdir /mnt/disk_name
```

2. List all disks: 
```bash
ls -l /dev/disk/by-uuid/*
```
3. Copy UUID of the disk you want to mount

4. Edit file `/etc/fstab`: 
```bash
nano /etc/fstab
```
5. Add line to the end:
```
/dev/disk/by-uuid/$DISK_UUID /mnt/$MOUNT_DIRECTORY ext4 defaults 0
```

6. Mount the disk: 
```bash
mount -a
```

7. Check the mount by: 
```bash
lsblk
```

### Updating of Linux Kernel

[Update Linux Kernel version](https://askubuntu.com/questions/1388115/how-do-i-update-my-kernel-to-the-latest-one)

Check Kernel version:

```bash
uname -r
```

Install the shell script which automatically checks and install the latest kernel:
```bash
wget https://raw.githubusercontent.com/pimlie/ubuntu-mainline-kernel.sh/master/ubuntu-mainline-kernel.sh
sudo install ubuntu-mainline-kernel.sh /usr/local/bin/
```

Check if a newer kernel version is available:

```bash
sudo ubuntu-mainline-kernel.sh -c
```

List available Kernel versions:

```bash
sudo ubuntu-mainline-kernel.sh -r
```

Install the latest Kernel:

```bash
sudo ubuntu-mainline-kernel.sh -i
```

Install the needed Kernel version:
```bash
sudo ubuntu-mainline-kernel.sh -i 6.5.11
```



### Installing i915 driver to the Proxmox VE

#### Preparation
Make sure that your proxmox is not in enterprise mode or you have subscription
Go to you datacenter->Updates->Repositories
Add new "No subscription"
Disable pve-enterprise

#### Install all helpers
```bash
apt update
apt upgrade
apt install git
apt install build-* dkms
```

#### Build and install driver

1. Clone repo: 
```bash
git clone https://github.com/strongtz/i915-sriov-dkms.git
```

2. Edit dkms.conf: 
```bash
nano i915-sriov-dkms/dkms.conf
```

Set PACKAGE_NAME to `i915-sriov-dkms` and PACKAGE_VERSION to `6.6` or whatever version you want, we will force driver installation anyway. The rows should look like this:
```bash
PACKAGE_NAME="i915-sriov-dkms"
PACKAGE_VERSION="6.6"
```

3. Make the directory for source files to build driver inside of `/usr/src`. The directory name should be formatted $PACKAGE_NAME-$PACKAGE_VERSION specified in the previous step:
```bash
mkdir /usr/src/i915-sriov-dkms-6.6
```

4. Copy all files from the git repo folder to the source directory:
```bash
cp -r i915-sriov-dkms/* /usr/src/i915-sriov-dkms-6.6
```

5. Build and install the driver. Version should be the save as we specified earlier:
```bash
dkms install --force -m i915-sriov-dkms -v 6.6
```

6. When that’s done, use the following command to verify the installation:
```bash
dkms status
```

The correct output should look something like this:
```bash
i915-sriov-dkms/6.6, 6.5.11-7-pve, x86_64: installed
```

#### Host Kernel Command Line Modification
1. Edit GRUB:
```bash
nano /etc/default/grub
```

2. Look for “GRUB_CMDLINE_LINUX_DEFAULT” line, and add the following:
```bash
intel_iommu=on i915.enable_guc=3 i915.max_vfs=7
```

3. Your line could look like this:
```bash
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on i915.enable_guc=3 i915.max_vfs=7 quiet"
```

4. Update initramfs and grub:
```bash
update-grub
update-initramfs -u
```

5. Install sysfsutils:
```bash
apt install sysfsutils -y
```

6. Check PCI address
```bash
lspci | grep "VGA"
```

Output should look like this:
```bash
00:02.0 VGA compatible controller: Intel Corporation Alder Lake-S GT1 [UHD Graphics 730] (rev 0c)
```

7. Configure sysfs.conf. Make sure that PCI address is the same as in previous step, in example it is `00:02:0`:
```bash
echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7" > /etc/sysfs.conf
```

8. *Optional* Download iGPU Firmware
- Check if the firmware exists on your system:
```bash
ls /lib/firmware/i915/tgl_guc_70.1.1.bin
```

- If the output is empty, download the firmware from here:
```bash
wget https://mirrors.apqa.cn/d/proxmox-edge/intel_gpu_sriov/i915-firmware.tar.gz
```

- Then: 
```bash
tar -xf i915-firmware.tar.gz
```

- Finally, copy to the firmware folder: 
```bash
cp ./firmware/* /lib/firmware/i915/
```

- Note: this is the firmware for 12th/13th gen iGPUs (Alder Lake, Raptor Lake). If you are using 11th gen, you have to source your own firmware.

9. Reboot the system:
```bash
reboot
```

#### Verification
After a system reboot, check `dmesg` to see if your iGPU has virtual functions enabled, or use the following:
```bash
lspci  | grep VGA
```

You should see a similar output under PCIe address your iGPU is assigned to:
```bash
00:02.0 VGA compatible controller: Intel Corporation Alder Lake-S GT1 [UHD Graphics 730] (rev 0c)
```

If you don't see any VFs, do `dmesg | grep i915` to see what kind of error messages you're getting.

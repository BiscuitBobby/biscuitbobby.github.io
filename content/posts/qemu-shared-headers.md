+++
title = "QEMU Shared Headers & Module Development Setup"
date = "2025-06-15T20:05:14+05:30"
author = "BiscuitBobby"
cover = ""
tags = ["QEMU", "Linux Kernel"]
keywords = ["QEMU", "kernel development", "virtio-9p", "kernel modules", "shared directory", "headers", "SELinux", "IIO dummy"]
description = "A quick guide to setting up a QEMU VM for kernel development, including shared directories, networking, SELinux tuning, and managing headers/modules."
showFullContent = false
readingTime = true
hideComments = false
+++

# QEMU VM Setup for Kernel Development & Testing

This guide outlines how to set up a QEMU virtual machine for kernel development, including sharing directories with the host, configuring network support, building kernel headers, and managing kernel modules.
## Sharing a Host Directory with the QEMU Guest
To share a directory from your host with your QEMU virtual machine, you can use the "virtio-9p" filesystem. This allows you to mount a host directory inside the guest VM. Here's how to set it up:

1. First, modify your QEMU command to include the shared directory:

```bash
qemu-system-x86_64 \
-m 2G \
-smp 2 \
-kernel "$BZIMAGE" \
-append "root=/dev/sda rootfstype=ext4 console=ttyS0 earlyprintk=serial net.ifnames=0" \
-drive file="$ROOTFS",format=raw \
-net user,host=10.0.2.10,hostfwd=tcp::20021-:22 \
-net nic,model=e1000 \
-fsdev local,id=shared_dev,path=/path/to/host/directory,security_model=mapped-xattr \
-device virtio-9p-pci,fsdev=shared_dev,mount_tag=shared_dir \
-enable-kvm \
-nographic
```

Replace `/path/to/host/directory` with the actual path of the directory you want to share.
(**Dont forget to enable ext4**)

2. Once your VM has booted, you need to mount the shared directory inside the guest. You can do this with:

```bash
# First, create a mount point
mkdir -p /mnt/shared

# Then mount the 9p filesystem
mount -t 9p -o trans=virtio,version=9p2000.L shared_dir /mnt/shared

cd /mnt/shared/
```

3. If you want to make this mount permanent, add it to your guest's `/etc/fstab`:

```
shared_dir /mnt/shared 9p trans=virtio,version=9p2000.L 0 0
```

Notes:

- Your kernel needs to have [9P filesystem support](https://wiki.qemu.org/Documentation/9psetup) enabled
- The `security_model` parameter can be `none`, `mapped`, or `mapped-xattr` depending on your security needs
- You might need to adjust permissions or use different security models based on your specific requirements
- The `mount_tag` (in this case `shared_dir`) is an arbitrary name that you use to identify the share inside the VM

If your kernel doesn't have 9P support, you'll need to either rebuild it with that feature or use an alternative approach like NFS or SSH for file transfers.

## Network Support
you need:
```
CONFIG_E1000=y
CONFIG_E1000E=y
CONFIG_VIRTIO_NET=y
CONFIG_NETDEVICES=y
CONFIG_ETHERNET=y
CONFIG_NET_VENDOR_INTEL=y
```

If CONFIG_E1000 is n, your kernel lacks support for the e1000 network card.
If CONFIG_VIRTIO_NET is n, VirtIO networking won’t work.

### Build Headers on shared Directory

If the headers are missing, you'll need to generate them:

1. Create a symbolic link:
```bash
rm -rf /usr/lib/modules/$(uname -r)
mkdir /mnt/shared/modules/$(uname -r)
```

```bash
sudo ln -sfn /mnt/shared/modules/$(uname -r) /usr/lib/modules/
```
2. Go to the Source Directory
```bash
make INSTALL_HDR_PATH=/mnt/shared/modules/$(uname -r) headers_install
```

## SELinux Configuration (If Applicable)

If your guest distribution uses SELinux (e.g., Fedora, RHEL, CentOS), it might interfere with operations like mounting 9P shares or loading certain modules if policies are not permissive enough.

To check or change SELinux mode:
1.  Edit the configuration file: `sudo nano /etc/selinux/config` (or `vi`, etc.)
2.  Find the `SELINUX=` directive.
    *   `SELINUX=enforcing`: SELinux security policy is enforced.
    *   `SELINUX=permissive`: SELinux prints warnings but does not enforce. Useful for debugging.
    *   `SELINUX=disabled`: SELinux is completely off.
3.  Change the value as needed. For development or troubleshooting, you might temporarily set it to `permissive`.
    ```
    SELINUX=permissive
    ```
4.  **Reboot the guest VM for the change to take full effect if you changed from `enforcing`/`permissive` to `disabled` or vice-versa.**
    Alternatively, you can temporarily change the mode without rebooting (if not disabling completely):
    ```bash
    sudo setenforce 0 # Sets to permissive mode
    sudo setenforce 1 # Sets to enforcing mode
    getenforce        # Checks current mode
    ```


## Loading modules
Let us try loading the dummy driver iio driver. It needs to be built as a loadable kernel module rather than being compiled directly into the kernel. Currently, your system has the IIO dummy driver built into the kernel itself (not as a module), which is why it doesn't appear in `lsmod`.

Here's what you need to do:

1. Change your kernel configuration to build the IIO dummy driver as a module instead of built-in:
    
    - Run `make menuconfig` in your kernel source directory
    - Navigate to Device Drivers → Industrial I/O support
    - Find the dummy device entries (IIO dummy driver and IIO dummy event generator)
    - Change them from `<*>` (built-in) to `<M>` (module)
    - Save the configuration and exit
2. Rebuild the kernel modules:
    
```bash
make modules
make modules_install
```
    
3. Load the module manually after reboot:
    
```bash
modprobe iio_dummy
```
    
4. Then check with `lsmod` and you should see it listed
    

The exact module name might vary depending on your kernel version, but after loading it should appear in the `lsmod` output.

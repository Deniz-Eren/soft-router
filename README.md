# soft-router

Home router setup and associated server hardware and software configurations.

## Outline

We want to run pfsense within a QEmu VM on the N100 Computer running Ubuntu server.

QEmu VM is a very lightweight system, that allows easy pass-through of the host CPU and it's features, together with PCI devices. In our case, we wish to pass-through 2 of the 2.5G I226-V LAN devices to be owned and managed within the VM running pfsense.

Furthermore, the host Ubuntu server gives us control over changes we would like to make to the VM.

Containers were considered, however since pfsense runs on a different kernel, QEmu VM was selected.

## Hardware

Hardware used from AliExpress:

- 12th Gen Intel N100 Computer with 4 x 2.5G I226-V LAN NVMe Industrial Fanless Mini PC (16GB Ram 256GB NVMe)
- HORACO 2.5G Managed Switch 8 Port 2.5GBASE-T Fanless

## Architecture

## BIOS Setup

Ensure you enable Intel Virtualization (VT-x) and VT-d (direct I/O) from the BIOS.

## Server Setup

Configuring the N100 Computer.

### Install Host Ubuntu Server

Installed Ubuntu 24.04 onto the N100 Computer.

### Enable Linux IOMMU Support

Edit `sudo vim /etc/default/grub` and navigate to `GRUB_CMDLINE_LINUX_DEFAULT=""` and make the following changes:

    GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on kvm.ignore_msrs=1"

For it to take effect run:

    sudo grub-mkconfig -o /boot/grub/grub.cfg

After rebooting the server, check that Linux IOMMU Support is enabled by running:

    sudo dmesg | grep -i -e DMAR -e IOMMU

You should see `DMAR: IOMMU enabled` in the output dump.

## Enable Linux Kernel Module VFIO - “Virtual Function I/O”

Copy `/etc/modules-load.d/vfio-pci.conf` to your server and run:

    sudo chmod 644 /etc/modules-load.d/vfio-pci.conf

## Create pfsense disk image

First make a 32G disk image:

    dd if=/dev/zero of=disk-raw iflag=fullblock bs=1G count=32

Download your copy of `netgate-installer-amd64.iso` (pfsense installer).

Figure out the PCI device addresses of the two I226-V LAN devices you wish to use with pfsense. To do this, you use commands like `lspci -nnk` and `lshw -short -c network`. In my case these address were `0000:02:00.0` for ETH1 and `0000:03:00.0` ETH2.

Then start QEmu mounting your disk image, ISO installer file and LAN path-through devices:

    qemu-system-x86_64 \
        -enable-kvm \
        -cpu host \
        -smp 4 \
        -k en-us \
        -cdrom netgate-installer-amd64.iso \
        -drive id=disk,file=./disk-raw,format=raw,if=none \
        -device ahci,id=ahci \
        -device ide-hd,drive=disk,bus=ahci.1 \
        -boot d \
        -device vfio-pci,host=0000:02:00.0 \
        -device vfio-pci,host=0000:03:00.0 \
        -m size=4096

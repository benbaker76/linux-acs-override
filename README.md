# ACS Override Patch

A common use case in virtualization is the need to passthrough a PCI or VGA device to a guest VM. However, because all devices within a single IOMMU group must be passed through at once, it can pose challenges to passthrough if there are multiple devices in the same IOMMU group.

To address the issue, Alex Williamson created a kernel patch that allows each device to be placed into its own IOMMU group, easing passthrough:

[https://lkml.org/lkml/2013/5/30/513](https://lkml.org/lkml/2013/5/30/513)

This repository provides patch files for various kernel versions from 4.10 and updates when a new kernel release breaks patch compatibility.

Note: The ACS override patch is typically considered a 'last resort' for PCIe passthrough if other methods don't work. It's not meant for production systems (and neither are custom kernels), but may be useful for dev environments or homelabs.

References and ACS Override Alternatives:

[https://heiko-sieger.info/iommu-groups-what-you-need-to-consider/](https://heiko-sieger.info/iommu-groups-what-you-need-to-consider/) [https://forum.level1techs.com/t/the-pragmatic-neckbeard-3-vfio-iommu-and-pcie/111251](https://forum.level1techs.com/t/the-pragmatic-neckbeard-3-vfio-iommu-and-pcie/111251) [https://bugzilla.redhat.com/show_bug.cgi?id=1113399](https://bugzilla.redhat.com/show_bug.cgi?id=1113399) [https://bugzilla.redhat.com/attachment.cgi?id=913028&action=diff](https://bugzilla.redhat.com/attachment.cgi?id=913028&action=diff)

## Installation

### Confirm Current IOMMU Groups

Run the iommu-groups.sh helper script to view the current IOMMU groups:

```sh
# ./iommu-groups.sh
...
IOMMU Group 2:
	00:08.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Renoir PCIe Dummy Host Bridge [1022:1632]
	00:08.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Renoir Internal PCIe GPP Bridge to Bus [1022:1635]
	00:08.2 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Renoir Internal PCIe GPP Bridge to Bus [1022:1635]
	05:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir [1002:1636] (rev d8)
	05:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Device [1002:1637]
	05:00.2 Encryption controller [1080]: Advanced Micro Devices, Inc. [AMD] Family 17h (Models 10h-1fh) Platform Security Processor [1022:15df]
	05:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Renoir USB 3.1 [1022:1639]
	05:00.4 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Renoir USB 3.1 [1022:1639]
	05:00.6 Audio device [0403]: Advanced Micro Devices, Inc. [AMD] Family 17h (Models 10h-1fh) HD Audio Controller [1022:15e3]
	06:00.0 SATA controller [0106]: Advanced Micro Devices, Inc. [AMD] FCH SATA Controller [AHCI mode] [1022:7901] (rev 81)
...
```

In the above example, two USB controllers and two PCI bridges are in the same group.

### Build the Kernel Source

In this example we'll build the Kernel into the Debian .deb file format for installation.

```sh
# ./build-debian.sh
```

### Install the Kernel

```sh
# ls *.deb
linux-headers-6.3.0-acso_6.3.0-1_amd64.deb  linux-image-6.3.0-acso-dbg_6.3.0-1_amd64.deb
linux-image-6.3.0-acso_6.3.0-1_amd64.deb    linux-libc-dev_6.3.0-1_amd64.deb
# dpkg -i *.deb
# reboot
```

### Boot System Using the New Custom Kernel

```sh
# vi /etc/default/grub
```

Add `pcie_acs_override=downstream,multifunction` to the `GRUB_CMDLINE_LINUX_DEFAULT` entry like so:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash pcie_acs_override=downstream,multifunction"
```

You can also change this line to boot directly into the ACS patched kernel on boot:

```
GRUB_DEFAULT="1>Ubuntu, with Linux 6.3.0-acso"
```

### Update GRUB

```sh
# update-grub
# reboot
```

Confirm the grub config updates are applied:

```sh
# cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-6.3.0-acso root=UUID=0706649e-a7c1-4631-973f-cd650f863f00 ro quiet splash systemd.unified_cgroup_hierarchy=0 amd-pstate=active pcie_acs_override=downstream,multifunction vt.handoff=7
```

### Confirm Devices Separated

Run the iommu-groups.sh script again to confirm devices are now separated as expected:

```sh
# ./iommu-groups.sh
IOMMU Group 0:
	00:01.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Renoir PCIe Dummy Host Bridge [1022:1632]
IOMMU Group 1:
	00:01.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Renoir PCIe GPP Bridge [1022:1633]
IOMMU Group 2:
	00:02.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Renoir PCIe Dummy Host Bridge [1022:1632]
IOMMU Group 3:
	00:02.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Renoir PCIe GPP Bridge [1022:1634]
IOMMU Group 4:
	00:02.3 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Renoir PCIe GPP Bridge [1022:1634]
IOMMU Group 5:
	00:02.4 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Renoir PCIe GPP Bridge [1022:1634]
IOMMU Group 6:
	00:08.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Renoir PCIe Dummy Host Bridge [1022:1632]
IOMMU Group 7:
	00:08.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Renoir Internal PCIe GPP Bridge to Bus [1022:1635]
IOMMU Group 8:
	00:08.2 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Renoir Internal PCIe GPP Bridge to Bus [1022:1635]
IOMMU Group 9:
	00:14.0 SMBus [0c05]: Advanced Micro Devices, Inc. [AMD] FCH SMBus Controller [1022:790b] (rev 51)
	00:14.3 ISA bridge [0601]: Advanced Micro Devices, Inc. [AMD] FCH LPC Bridge [1022:790e] (rev 51)
IOMMU Group 10:
	00:18.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Renoir Device 24: Function 0 [1022:1448]
	00:18.1 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Renoir Device 24: Function 1 [1022:1449]
	00:18.2 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Renoir Device 24: Function 2 [1022:144a]
	00:18.3 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Renoir Device 24: Function 3 [1022:144b]
	00:18.4 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Renoir Device 24: Function 4 [1022:144c]
	00:18.5 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Renoir Device 24: Function 5 [1022:144d]
	00:18.6 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Renoir Device 24: Function 6 [1022:144e]
	00:18.7 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Renoir Device 24: Function 7 [1022:144f]
IOMMU Group 11:
	01:00.0 Non-Volatile memory controller [0108]: Samsung Electronics Co Ltd NVMe SSD Controller SM981/PM981/PM983 [144d:a808]
IOMMU Group 12:
	02:00.0 Non-Volatile memory controller [0108]: Samsung Electronics Co Ltd NVMe SSD Controller SM981/PM981/PM983 [144d:a808]
IOMMU Group 13:
	03:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller [10ec:8168] (rev 15)
IOMMU Group 14:
	04:00.0 Network controller [0280]: Intel Corporation Wi-Fi 6 AX200 [8086:2723] (rev 1a)
IOMMU Group 15:
	05:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir [1002:1636] (rev d8)
IOMMU Group 16:
	05:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Device [1002:1637]
IOMMU Group 17:
	05:00.2 Encryption controller [1080]: Advanced Micro Devices, Inc. [AMD] Family 17h (Models 10h-1fh) Platform Security Processor [1022:15df]
IOMMU Group 18:
	05:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Renoir USB 3.1 [1022:1639]
IOMMU Group 19:
	05:00.4 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Renoir USB 3.1 [1022:1639]
IOMMU Group 20:
	05:00.6 Audio device [0403]: Advanced Micro Devices, Inc. [AMD] Family 17h (Models 10h-1fh) HD Audio Controller [1022:15e3]
IOMMU Group 21:
	06:00.0 SATA controller [0106]: Advanced Micro Devices, Inc. [AMD] FCH SATA Controller [AHCI mode] [1022:7901] (rev 81)
```

## PCI passthrough

Now the patch has been applied to the kernel and the USB controller, for example, is in its own group we can pass it through to the virtual machine.

```sh
IOMMU Group 18:
	05:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Renoir USB 3.1 [1022:1639]
```

```sh
# vi /etc/modprobe.d/vfio.conf
```

```
install xhci-pci echo vfio-pci >/sys/bus/pci/devices/0000:05:00.3/driver_override; echo 0000:05:00.3 >/sys/bus/pci/drivers/vfio-pci/bind; modprobe -i xhci-pci
```

```sh
# update-initramfs -u
# reboot
```

## Links

- https://github.com/ecspangler/centos-7-acs-override-kernel-patch
- [ACS Override Kernel Builds](https://queuecumber.gitlab.io/linux-acs-override)
- [Ensuring that the groups are valid](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Ensuring_that_the_groups_are_valid)

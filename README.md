# GPU PCI-Passthrough on Secure Blue

## Overview

When you have hardware burning a hole in your desk, and you want to run a VM that can utilize physical hardware such as a Graphics Processing Unit, or GPU, this is how you set it up.

### References

Docs and Blogs on how-to

- [ArchLinux Wiki: PCI Passthrough via OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)
- [GitHub: BigOakley/gpu-passthrough](https://github.com/BigOakley/gpu-passthrough/blob/main/outline.md)
- [mrzk.io Blog: Fedora 32 and GPU Passthrough VFIO](https://mrzk.io/posts/fedora-32-and-gpu-passthrough-vfio/)
- [N00b security Blog: Easy GPU Passthrough using KVM on Fedora](https://n00bsecurityblog.wordpress.com/2019/11/03/easy-gpu-passthrough-using-kvm-on-fedora/)

Fedora Discussions

- [Fedora Discussions: GPU Pass-through on Silverblue](https://discussion.fedoraproject.org/t/gpu-pci-passthrough-with-silverblue-kinoite/45271)

Secure Blue Discussions

- [[Working] GPU Passthrough with Virt - Manager](https://discord.com/channels/1202086019298500629/1439321824251875619)


## Hardware

You will need to have a _second monitor_ with this setup. You could try to setup _Looking Glass_ or _Sunshine+Moonlight_ if you wanted to.

### Primary GPU for Host

- [Asus Prime GeForce RTX 5060 8 GB OC Edition](https://www.asus.com/es/motherboards-components/graphics-cards/prime/prime-rtx5060-o8g/) - 145w

### Secondary GPU for VM

* [Palit KalmX Fanless GeForce RTX 3060 6 GB](https://www.palit.com/palit/vgapro.php?id=5147&lang=esla)

## Preliminary Checks

> [!Info]
> Make sure you have all updates installed before doing any of this. Makes it go faster!

Run this grep command to find if your CPU has virtualization enabled.

`run0 grep --color --regexp vmx --regexp svm /proc/cpuinfo`

If there is no result, make sure to enable `VT-d` for Intel or `AMD-V` for AMD based motherboards. Consult your hardware's instructions on how to do that.

## Steps for Intel CPU

### Ensure CPU has IOMMU enabled

Make sure to set:

`iommu=force intel_iommu=on iommu.passthrough=1 iommu.strict=1 rd.driver.pre=vfio_pci`

as a Kernel parameter.

Run, edit and save with:

`rpm-ostree kargs --editor`


## Steps for AMD CPU (Not tested on my side)

### Ensure CPU has IOMMU enabled

This should already be enabled for AMD CPUs. Either way, you'll want to do this.

`iommu=force amd_iommu=on iommu.passthrough=1 iommu.strict=1 rd.driver.pre=vfio_pci`

## Check PCI Bus Groups

You can get the device IDs using `lspci`. In this case, I'm looking for my NVIDIA card to pass-through.

```
➜  ~ 
> lspci -vnn | grep -i --regexp NVIDIA
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GB206 [GeForce RTX 5060] [10de:2d05] (rev a1) (prog-if 00 [VGA controller])
	Kernel driver in use: nvidia
	Kernel modules: nouveau, nvidia_drm, nvidia
01:00.1 Audio device [0403]: NVIDIA Corporation GB206 High Definition Audio Controller [10de:22eb] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:0000]
06:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA107 [GeForce RTX 3050 6GB] [10de:2584] (rev a1) (prog-if 00 [VGA controller])
	Subsystem: NVIDIA Corporation GA107 [GeForce RTX 3050 6GB] [10de:2584]
	Kernel modules: nouveau, nvidia_drm, nvidia
06:00.1 Audio device [0403]: NVIDIA Corporation GA107 High Definition Audio Controller [10de:2291] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:2584]
```

Now we'll setup the Kernel Args for disabling the PCI Buses for the GPU.

```
rpm-ostree kargs --editor \
  "vfio-pci.ids=10de:2584,10de:2291"
```

Then perform a `run0 nano /etc/dracut.conf.d/10-vfio.conf`and add these lines:

`force_drivers=" vfio-pci "`
`install_optional_items+=" /usr/lib64/libno_rlimit_as.so "`

Save and exit.


to make sure the _initramfs_ has the Kernel module loaded.

```bash
rpm-ostree initramfs --enable
```

And reboot.

You should now see when you perform

```bash
sudo lspci -nnv
```

Should show something similar to...

```bash
➜  ~ 
> lspci -vnn
[...lots of output...]

06:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA107 [GeForce RTX 3050 6GB] [10de:2584] (rev a1) (prog-if 00 [VGA controller])
	Subsystem: NVIDIA Corporation GA107 [GeForce RTX 3050 6GB] [10de:2584]
	Flags: fast devsel, IRQ 255, IOMMU group 17
	Memory at 79000000 (32-bit, non-prefetchable) [size=16M]
	Memory at 40000000 (64-bit, prefetchable) [size=256M]
	Memory at 50000000 (64-bit, prefetchable) [size=32M]
	I/O ports at 3000 [size=128]
	Expansion ROM at 7a000000 [disabled] [size=512K]
	Capabilities: <access denied>
	Kernel driver in use: vfio-pci
	Kernel modules: nouveau, nvidia_drm, nvidia

06:00.1 Audio device [0403]: NVIDIA Corporation GA107 High Definition Audio Controller [10de:2291] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:2584]
	Flags: fast devsel, IRQ 255, IOMMU group 17
	Memory at 7a080000 (32-bit, non-prefetchable) [disabled] [size=16K]
	Capabilities: <access denied>
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel

[...lots of output...]
```

> [!Warning] If it doesn't say `vfio-pci`
> Then you're doing something wrong.

You can also check if the _vfio_ Kernel modules made it in the _initramfs_ by using:

```bash
➜  ~ 
> sudo lsinitrd /boot/ostree/fedora-af7516f20c0bee3df3aa77532024d9fb7c207d1b7246d1d4949b047d7ab916d9/initramfs-6.0.15-300.fc37.x86_64.img| grep -i --regexp vfio
Arguments:  --reproducible -v --add 'ostree' --tmpdir '/tmp/dracut' -f --no-hostonly --kver '6.0.15-300.fc37.x86_64' --reproducible -v --add 'ostree' --tmpdir '/tmp/dracut' -f --no-hostonly --add-drivers ' vfio-pci' --kver '6.0.15-300.fc37.x86_64'
drwxr-xr-x   3 root     root            0 Dec 31  1969 usr/lib/modules/6.0.15-300.fc37.x86_64/kernel/drivers/vfio
drwxr-xr-x   2 root     root            0 Dec 31  1969 usr/lib/modules/6.0.15-300.fc37.x86_64/kernel/drivers/vfio/pci
-rw-r--r--   1 root     root        32140 Dec 31  1969 usr/lib/modules/6.0.15-300.fc37.x86_64/kernel/drivers/vfio/pci/vfio-pci-core.ko.xz
-rw-r--r--   1 root     root         8596 Dec 31  1969 usr/lib/modules/6.0.15-300.fc37.x86_64/kernel/drivers/vfio/pci/vfio-pci.ko.xz
-rw-r--r--   1 root     root        19884 Dec 31  1969 usr/lib/modules/6.0.15-300.fc37.x86_64/kernel/drivers/vfio/vfio_iommu_type1.ko.xz
-rw-r--r--   1 root     root        14880 Dec 31  1969 usr/lib/modules/6.0.15-300.fc37.x86_64/kernel/drivers/vfio/vfio.ko.xz
-rw-r--r--   1 root     root         3552 Dec 31  1969 usr/lib/modules/6.0.15-300.fc37.x86_64/kernel/drivers/vfio/vfio_virqfd.ko.xz
```

The `/boot/ostree/fedora-???/initramfs-x.y.z-aaa.fc??.x86_64.img` depends on what image is live. You may have two or three depending on how many backup images you keep of your _ostree_ setup.


## Setup the Virtual Machine

In this we're using `qemu-kvm` with `virt-manager` as a GUI to help us.

As of this writing, I'm using `qemu-kvm` version `7.0.1`. This should be sufficient.

```bash
➜  ~ 
> qemu-kvm --version
QEMU emulator version 10.1.2 (qemu-10.1.2-1.fc43)
```

### Download Windows 10 ISO

At the time of this writing I'm using _Windows 10_ and I am downloading it from this link. Verify it before clicking.

- [Microsoft Windows 10 ISO Download](https://www.microsoft.com/en-us/software-download/windows10ISO)

> [!Warning] Verify your ISO Download
> Remember to verify the ISO download you're performing!

### Use Virtual Machine Manager

This is going to use `virt-manager` or Virtual Machine Manager. Either way it's the GUI of the command-line application _virt-install_.

- Open `virt-manager`.
- Create a new Virtual Machine. Before installing, be sure to customize the configuration.
	- Provide at least 8GB of RAM.
	- Provide at least 4 CPUs.
	- Provide at least 500GB of Storage.
		- This is because Windows is stupid and huge.
- Ensure the following settings are set.
	- Overview -> Chipset: Q35
	- Overview -> Firmware: UEFI/OVMF
	- Add Hardware -> PCI Host Device -> NVIDIA GPU
	- Add Hardware -> PCI Host Device -> NVIDIA High Def Audio.
- Enable the SATA CD Drive.
- Change the boot order so that SATA CD Drive is first.
- Set CPUs -> Topology -> Sockets 1, Cores 4, Threads 2.
	- This makes 8 Virtual Processors.
- Apply the changes.
- Begin Installation

Setup Windows as you can do the best. I had to move my monitor, then adjust the position of screens, update a bunch and then I could finally move on.

### Modify the Domain XML

I am using Virt Manger.

Here, we'll insert the few elements.

```xml
<domain type='kvm'>
  ...
  <features>
    <kvm>
      <hidden state='on' />
    </kvm>
    <hyperv>
      <vendor_id state='on' value='whatever' />
    </hyperv>
  </features>
  ...
</domain>
```

You'll need to have the VM powered off for these changes to take effect.

> [!Warning] Secure Boot UEFI
> I disabled this in the VM's Bios. I had to enable the boot menu, then press F12.

### Install virtio-win-guest-tools

This is from:

- [Fedorapeople.org - virt-win-guest-tools.exe](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/virtio-win-guest-tools.exe)

This will need to be downloaded on the Windows PC. Then reboot.

### Install and configure NVIDIA drivers

Download the _official_ NVIDIA drivers.

### Install Steam

Download and install SteamSetup.exe from:

- [SteamPowered](https://steampowered.com)

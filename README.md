# Asrock NUC 4x4 4300U - Passthrough AMD 4300u iGPU to Windows 10

How to passthrough an AMD 4300U iGPU to Windows 10 on Proxmox


# References

- Proxmox 8.1 or later.
- Kernel 6.1 or later.
- UEFI Bios, Q35.
- Windows 10 or 11 VM, I didn't test on Linux VM.
- VBIOS file: Renoir_Generic_VBIOS_updGOP.bin
- GOP file for audio: AMDGopDriver.rom

## Tools

- VBiosFinder - https://github.com/coderobe/VBiosFinder

## How to do that

Find your hardware info:
> root@pve:~# lspci -nn | grep VGA
05:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir [1002:1636] (rev c4)
> root@pve:/etc/modprobe.d# lspci -nns 05:00
05:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir [1002:1636] (rev c4)
05:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir Radeon High Definition Audio Controller [1002:1637]
05:00.2 Encryption controller [1080]: Advanced Micro Devices, Inc. [AMD] Family 17h (Models 10h-1fh) Platform Security Processor [1022:15df]
05:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Renoir/Cezanne USB 3.1 [1022:1639]
05:00.4 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Renoir/Cezanne USB 3.1 [1022:1639]
05:00.5 Multimedia controller [0480]: Advanced Micro Devices, Inc. [AMD] ACP/ACP3X/ACP6x Audio Coprocessor [1022:15e2] (rev 01)
05:00.6 Audio device [0403]: Advanced Micro Devices, Inc. [AMD] Family 17h/19h HD Audio Controller [1022:15e3]
root@pve:/etc/modprobe.d# cd ~
root@pve:~# lspci -nn | grep VGA
05:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir [1002:1636] (rev c4)
root@pve:~# lspci -nns 05:00
05:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir [1002:1636] (rev c4)
05:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Renoir Radeon High Definition Audio Controller [1002:1637]
05:00.2 Encryption controller [1080]: Advanced Micro Devices, Inc. [AMD] Family 17h (Models 10h-1fh) Platform Security Processor [1022:15df]
05:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Renoir/Cezanne USB 3.1 [1022:1639]
05:00.4 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Renoir/Cezanne USB 3.1 [1022:1639]
05:00.5 Multimedia controller [0480]: Advanced Micro Devices, Inc. [AMD] ACP/ACP3X/ACP6x Audio Coprocessor [1022:15e2] (rev 01)
05:00.6 Audio device [0403]: Advanced Micro Devices, Inc. [AMD] Family 17h/19h HD Audio Controller [1022:15e3]

- Copy bios files [Renoir_Generic_VBIOS_updGOP.bin, AMDGopDriver.rom]to your proxmox host folder "/usr/share/kvm/"
- Edit line "" in proxmox host grub file at "/etc/defaults/grub":
> GRUB_CMDLINE_LINUX_DEFAULT="amd-pstate=passive quiet video=efifb:off initcall_blacklist=sysfb_init textonly iommu=pt pcie_acs_override=downstream,multifunction"

- Edit your modprob config files
	- blacklist.conf
	> blacklist radeon
		blacklist amdgpu
		blacklist snd_hda_intel
	- vfio.conf
	> 	options vfio-pci ids=1002:1636,1002:1637,1022:15df,1022:1639,1022:15e2,1022:15e3 disable_vga=1
		softdep radeon pre: vfio-pci
		softdep amdgpu pre: vfio-pci
		softdep snd_hda_intel pre: vfio-pci

After all configurations we need to update module and grub
> update-initramfs -u -k all ; update-grub

Forward PCI device to VM, here is my VM config file
> agent: 1
args: -cpu 'host,-hypervisor,kvm=off'
balloon: 8192
bios: ovmf
boot: order=scsi0;ide2;net0;ide0
cores: 2
cpu: host
efidisk0: local-lvm:vm-100-disk-0,efitype=4m,size=4M
hostpci0: 0000:05:00.0,pcie=1,x-vga=1,romfile=Renoir_Generic_VBIOS_updGOP.bin
hostpci1: 0000:05:00.1,pcie=1,romfile=AMDGopDriver.rom
hostpci2: 0000:05:00.2,pcie=1
ide0: none,media=cdrom
machine: q35
memory: 8192
meta: creation-qemu=8.1.5,ctime=1714375794
name: windows10
net0: virtio=XX:XX:XX:XX:XX:XX bridge=vmbr1
numa: 0
ostype: l26
scsi0: local-lvm:vm-100-disk-1,backup=0,cache=writeback,discard=on,iothread=1,size=100G,ssd=1
scsihw: virtio-scsi-single
smbios1: uuid=f0361f56-75ca-4c07-ab86-d0d64d5d6bab
sockets: 1
tpmstate0: local-lvm:vm-100-disk-2,size=4M,version=v2.0
vga: none
vmgenid: af021217-86b5-4e49-a2a2-9bedf6511e3f

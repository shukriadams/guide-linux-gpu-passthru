# Hardware passthrough from Linux host

WORK IN PROGRESS

This guides explains how to provision a Linux host machine with one or more Windows guests using hardware passthrough. Hardware passthrough with disks allows you to transplant
an existing bootable Windows or Linux disk into your host system and run it as a guest. These disks can be removed and run independently again, or even set to be the primary 
boot disk on your host system via the BIOS. This gives you a great deal of flexibility for experimenting with virtualization and system consolidation without overcommitting.

## Distro

This guide assumes you're running Ubuntu 20.04 host.

## Hardware

Hardware passthrough requires that you have IOMMU supported and enabled on your motherboard. You will likely have to manually enable this in your BIOS.

## Install IOMMU

On your host system install the following apps

    sudo apt install qemu-kvm qemu-utils libvirt-daemon-system libvirt-clients bridge-utils virt-manager ovmf -y

To pass through a GPU, get its device id

    sudo lspci -nnk

look for `VGA`, find all ids associated with it, egs `[10de:1b06]`. There will normally be multiple.

modify grub 

    sudo nano /etc/default/grub

set content to look like, replace the pci.ids list with those from your gpu. force gfxmode to lower resolution as on high res displays grub iteself can be extremely slow 

    GRUB_GFXMODE=1280x1024x32,auto
    GRUB_CMDLINE_LINUX_DEFAULT="quiet splash iommu=1 amd_iommu=on vfio_pci.ids=10de:128b,10de:0e0f vfio_iommu_type1.allow_unsafe_interrupts=1"
    GRUB_CMDLINE_LINUX=""

rebuild 

    sudo update-grub

reboot system.

Confirm IOMMU is working with 

    dmesg | grep AMD-Vi

which should report a bunch of IOMMU things.

Confirm that the device has been successfully assigned to vfio with 

    sudo lspci -nnk
    
this should list the `kernel driver` in use for that device as `vcio-pci`.

Ensure that the device you are passing through is in its own iommu group

    find /sys/kernel/iommu_groups/ -type l

If there are other devices in that group, move the devices to another PCI slot, or assign them to vfio as well. If you can't, your motherboard unfortunately can't be used for this.

- open virt manager 
- create vm
- Select "import existing disk image" under the sources for operating system
- for storage, point to the ENTIRE disk to pass through, ex `/dev/sdb`. It's far better to use a fixed id that never changes though, such as `/dev/disk/by-id/<your-disk-id-here>`
  You can get the fixed disk id by querying its device name

        udevadm info /dev/sda

  Find your devices with 
        
        lsblk

- choose Windows 10 as OS (check allow end-of-life OSes)
- ensure that Customize before install is checked
- chipset must be q35
- firmware must be uefi, likely OVMF_CODE.fd

Under CPUs uncheck "copy host CPU config" and in the textbox/dropdown that becomes available type in "host-passthrough". Entering the correct.
Select "Manually set CPU topology" and configure how many cores/threads you want to pass though. This can be a bit tricky, the logical host cpus field at the top 
of the CPU config list is actually threads from the host.

Sockets is normally 1.
Cores is the nr of cores that will appear in the guest, and is normally half the nr of threads passed in from host.
Threads is the number of threads per core, and is normally 2. So guest cores x guest threads = nr of logical host cpus.

To pass through GPUs and USB devices, click

"add hardware", for GPUs/audio click "pci host device", the GPU you blocked in GRUB should be available here as both a video and audio device.

USB devices should appear under "USB host devices" and should be plug-and-play, both from host and to guest. That is to say, you can allocate them to the guest
after starting it. 

Start your PC and confirm that it loads. You should see your Nvidia GPU passed through, and if you look for it under device maanger, it will still list "error 43", 
just install modern drivers and it should appear as a functional device.

To use the GPU as your primary device, shut down your guest and under its hardware config, edit "Video QXL" and under model set  

## How to get into VM?

Figuring out how to get into a guest can be a bit tricky.
Use RDP. Don't know about looking glass yet.

## Networking

see https://www.cyberciti.biz/faq/how-to-add-network-bridge-with-nmcli-networkmanager-on-linux/

to get the vm online you'll need a bridge connection, this connection lets you piggy back onto the connection of a host connection.

Get a list of active connections with 

    nmcli connection show --active 

This should return "wired connection 1" and probably a few more. QEMU normally already installs a vbri0 or similarly-named connected, you can't use that. Note the device name `enp8s0` or `eth0` etc, this is needed.

Create a bridge called `br0` with

    sudo nmcli con add ifname br0 type bridge con-name br0

Then bind it your device with

    sudo nmcli con add type bridge-slave ifname enp8s0 master br0

confirm with

    nmcli connection show
    
you should get something like

    br0                  ee05dabb-cd7d-4b9c-ba9b-7012b10cb47e  bridge    br0    
    bridge-slave-enp8s0  3be4dd44-46a1-43c3-a00f-4e0154baad24  ethernet  enp8s0 
    vnet0                c4a1b159-9e32-4ebf-ac1d-a604742fc3f9  tun       vnet0  
    Wired connection 1   ed78e1ad-43a4-3486-b944-77b0ec9fb432  ethernet  --     
    Wired connection 2   9b2c4b07-94cf-3dbe-a066-4cf65d3b4013  ethernet  --   
    
    
disable wired connection 1 - warning, this will take you offline

    sudo nmcli con down "Wired connection 1"
    
and replace it by bringing your bridge online

    sudo nmcli con up br0
    
To add your bridge to QEMU use

    nano /tmp/br0.xml

add this text to it

    <network>
      <name>br0</name>
      <forward mode="bridge"/>
      <bridge name="br0" />
    </network>

then run the following to add bridge 

    virsh net-define /tmp/br0.xml
    virsh net-start br0
    virsh net-autostart br0
    virsh net-list --all
    
You should now see `br0` as a network option on your VM's nic - point to that. When you start your guest, VMM will likely not show an IP, you need to console into your VM and ipconfig to get its IP. Ideally, get the MAC nr as well and make them static.

Sometimes you need to remove a device and start again, try

    sudo nmcli connection delete <guid>
    

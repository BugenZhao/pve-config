# Proxmox VE

## Host Configuration

- `/etc/default/grub`

  ```bash
  GRUB_CMDLINE_LINUX="intel_iommu=on pcie_acs_override=downstream,multifunction pci=assign-busses nofb nomodeset video=efifb:off,vesafb:off"
  ```

- `/etc/modules/`

  ```bash
  vfio
  vfio_iommu_type1
  vfio_pci
  vfio_virqfd
  ```


- `/etc/modprobe.d/blacklist.conf`

  ```bash
  blacklist amdgpu
  blacklist snd_hda_intel
  blacklist radeon
  ```

- `/etc/modprobe.d/igb.conf` (SR-IOV ethernet)

  ```bash
  options igb max_vfs=7
  ```

- `/etc/modprobe.d/kvm-intel.conf`

  ```bash
  options kvm-intel nested=Y
  ```
  
- `/etc/modprobe.d/kvm-intel.conf` (SR-IOV)

  ```bash
  options vfio_iommu_type1 allow_unsafe_interrupts=1
  options vfio-pci ids=1002:731f,1002:ab38 disable_vga=1
  ```
  
  Run `lspci -nn` to get the ids.

- Update kernel and initramfs

  ```bash
  update-initramfs -u
  update-grub
  ```

## Guest Configuration

- `/etc/pve/qemu-server/1001.conf`

  ```
  agent: 1
  bios: ovmf
  boot: cd
  bootdisk: sata0
  cores: 6
  cpu: cputype=host
  efidisk0: local-lvm:vm-1001-disk-1,size=4M
  hostpci0: 03:00,pcie=1,x-vga=1
  ide2: s416-images:iso/virtio-win-0.1.185.iso,media=cdrom,size=402812K
  machine: q35
  memory: 12288
  name: Windows
  net0: virtio=D6:58:50:1B:00:68,bridge=vmbr0,firewall=1
  numa: 1
  ostype: win10
  sata0: /dev/disk/by-id/nvme-WDS500G2X0C-00L350_184510803654,backup=0,replicate=0,size=488386584K,ssd=1
  scsihw: virtio-scsi-pci
  smbios1: uuid=9bb9d862-3fd0-4246-bef1-5f6f14d3f564
  sockets: 1
  startup: up=10
  usb0: host=1-10,usb3=1
  usb1: host=1-11,usb3=1
  usb2: host=1-12,usb3=1
  vga: none
  vmgenid: 8d3e6dd9-19f1-4141-8b3f-c3f2d39bd5d0
  ```

## Tricks

- Use `fdisk -l` to check all disks, then `ls -l /sys/block/nvme0n1` to check the device's PCIe port

- Add a physical disk as virtual disk to vm

  `qm set 1001 -sata0 /dev/disk/by-id/nvme-WDS500G2X0C-00L350_184510803654`

- Windows cannot recognize disk on VirtIO SCSI buses, insert the driver disc while installing.

  For already installed Windows, follow this:

  https://superuser.com/questions/1057959/windows-10-in-kvm-change-boot-disk-to-virtio/1200899#1200899

- Add a CIFS storage for samba sharing, and remember to follow the directory structure.

- To disable hypervisor (hide vm status), append the line to `1001.conf`

  `args: -cpu 'host,-hypervisor'`

## Navi Reset Bug

- Patch at https://forum.level1techs.com/t/navi-reset-kernel-patch/147547/46

- Patch and compile the kernel

  - Prereqs

    ```bash
    apt install git-core lintain build-essential automake autoconf libtool
    apt install dh-make libtool sphinx-common dh-python
    apt install asciidoc-base bison flex libdw-dev libelf-dev libiberty-dev libnuma-dev libslang2-dev libssl-dev lz4 xmlto zlib1g-dev
    ```

  - Clone and patch

    ```bash
    git clone git://git.proxmox.com/git/pve-kernel.git --depth 1
    cd pve-kernel
    cp /path/to/patch.patch ./patches/kernel/
    ```

  - Make and install

    ```bash
    make
    dpkg -i *.deb
    ```

    
# Linux-Based-Diskless-Virtual-Machine-Implementations
## Proxmox, VM Linux completely into RAM About one of the ways to implement a diskless virtual machine based on Linux for clusters, running only in RAM

<div align="center">
<img src="Virtual-Machine-Implementations.png" style="width: 700px;height:500px" alt="Linux-Based-Diskless-Virtual-Machine-Implementations">
</div>
<br>
<br>

## The following solution, in my opinion, is good for implementing a virtual machine that does not use storage, can run in HA mode, live migration and perform various tasks and provide fault tolerance, made on Linux Debian in Proxmox cluster.  

## 1. We make a virtual machine so that it works only in RAM without a disk. We put it in HA.  

### 1.1. We make a Linux virtual machine in a standard way and configure it according to the tasks that will be assigned to it. We place the disk on a shared or local storage. This disk is used for the first start. And then the machine will be disconnected from this disk and can "roam" across nodes.  

### 1.2. In the file /usr/share/initramfs-tools/scripts/local, look for lines 179-185 (after making a backup of the file):  
```
   checkfs "${ROOT}" root "${FSTYPE}"  
	# Mount root
	# shellcheck disable=SC2086
	if ! mount ${roflag} ${FSTYPE:+-t "${FSTYPE}"} ${ROOTFLAGS} "${ROOT}" "${rootmnt?}"; then
		panic "Failed to mount ${ROOT} as root file system."
	fi  
```
**And we change this code to the following:**  
```
   #checkfs "${ROOT}" root "${FSTYPE}"

	# Mount root
	# shellcheck disable=SC2086
	mkdir /ramboottmp
	mount ${roflag} -t ${FSTYPE} ${ROOTFLAGS} ${ROOT} /ramboottmp
	mount -t tmpfs -o size=100% none ${rootmnt}
	cd ${rootmnt}
	cp -rfa /ramboottmp/* ${rootmnt}
	umount /ramboottmp  
```
### 1.3. Save the file. And enter the command in the terminal as root:  
```
mkinitramfs -o /boot/initrd.img-ramboot  
```
### 1.4. Check that the file is created in the /boot folder and return the old local to its place in the /usr/share/initramfs-tools/scripts/local folder (or delete all our changes that we made in step 1).  

### 1.5. Go to the folder: /etc and find the file: fstab, save a copy of it and edit it, look for something like this in the first lines:  
```
UUID= 321dba83-9a22-442b-b06b-185d7afe1088 / ext4 defaults 1 1  
```
**and change to:**  
```
none / tmpfs defaults 0 0  
```
### 1.6 Let's make a suitable menu at boot:  
**file:**  
```
nano /etc/grub.d/40_custom  
```
**Content:**  
```
#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.
menuentry 'RAM-Debian GNU/Linux' --class debian --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-321dba83-9a22-442b-b06b-185d7afe1088' {
        load_video
        insmod gzio
        insmod part_msdos
        insmod ext2
        set root='hd0,msdos1'
        echo    'Loading Linux 6.1.0-28-amd64 ...'
        linux   /boot/vmlinuz-6.1.0-28-amd64 root=UUID=321dba83-9a22-442b-b06b-185d7afe1088 ro  quiet splash toram
        echo    'Loading initial ramdisk ...'
        initrd  /boot/initrd.img-ramboot
}  
```
```
update-grub  
```
We get the grub menu.
1.6. Now let's create ram.tar.gz, turn off the virtual machine, boot a new virtual machine in liveCD mode and connect the disk of this virtual machine.

Mount it to /mnt. Let's do it:

# cd /mnt
# tar -czf /mnt/boot/ram.tar.gz .

1.7. Now when loading the virtual machine, select the appropriate RAM boot menu. After loading, unlock the disk:
1.7.1. Arrêtons l'accès au disque:

To do this, use the command:

echo 1 > /sys/block/sda/device/delete

echo 1 > /sys/block/sda/device/delete

This will disable the /dev/sda device at kernel level. The player will no longer be visible in the system.
1.7.2 Let's check the status:

Let's make sure the drive is no longer showing up in the list of devices:

lsblk

It will look like this:

root@debvsan:/home/vov# lsblk
NAME                            MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda                               8:0    0     8G  0 disk  
└─sda1                            8:1    0     8G  0 part  
root@debvsan:/home/vov#

echo 1 > /sys/block/sda/device/delete

lsblk

1.7.3. Detaching a disk from a virtual machine via the QEMU monitor:
1.7.3.1 Connect to the QEMU monitor for a specific virtual machine:

qm monitor 105

Check all devices connected to the virtual machine:

info block  

All connected disks will be displayed. You will see something like this:

root@pve1:~# qm monitor 105 
Entering QEMU Monitor for VM 105 - type 'help' for help
qm> info block
drive-scsi0 (#block190): /dev/pve/vm-105-disk-1 (raw)
    Attached to:      scsi0
    Cache mode:       writeback, direct
    Detect zeroes:    unmap
qm>

Disassemble the player:

device_del scsi0

Check what devices are connected:

info pci

You will see a list of PCI devices, including the SCSI controller.

For example:

Bus  9, device   1, function 0:
    SCSI controller: PCI device 1af4:1004
      PCI subsystem 1af4:0008
      IRQ 10, pin A
      BAR0: I/O at 0x1000 [0x103f].
      BAR1: 32 bit memory at 0xfd800000 [0xfd800fff].
      BAR4: 64 bit prefetchable memory at 0xfc000000 [0xfc003fff].
      id "virtioscsi0"

Recall:

qm> device_del virtioscsi0

To exit:

q

1.7.4.2. Insert into the configuration file:

root@pve1:~# nano /etc/pve/qemu-server/105.conf

Next:

disabled=1

Example:

scsi0: local-lvm:vm-105-disk-1,disabled=1,aio=native,backup=0,discard=on,iothread=1,size=8G
scsihw: virtio-scsi-single,disabled=1

1.7.4.3. And during migration we will see:

()
Task viewer: VM 105 - Migrate
OutputStatus
Stop
Download
task started by HA resource agent
2025-01-04 00:34:24 use dedicated network address for sending migration traffic (10.10.1.1)
2025-01-04 00:34:24 starting migration of VM 105 to node 'pve1' (10.10.1.1)
2025-01-04 00:34:24 starting VM 105 on remote node 'pve1'
2025-01-04 00:34:28 start remote tunnel
2025-01-04 00:34:29 ssh tunnel ver 1
2025-01-04 00:34:29 starting online/live migration on unix:/run/qemu-server/105.migrate
2025-01-04 00:34:29 set migration capabilities
2025-01-04 00:34:29 migration downtime limit: 100 ms
2025-01-04 00:34:29 migration cachesize: 512.0 MiB
2025-01-04 00:34:29 set migration parameters
2025-01-04 00:34:29 start migrate command to unix:/run/qemu-server/105.migrate
2025-01-04 00:34:30 migration active, transferred 357.9 MiB of 4.0 GiB VM-state, 586.6 MiB/s
2025-01-04 00:34:31 migration active, transferred 735.9 MiB of 4.0 GiB VM-state, 543.8 MiB/s
2025-01-04 00:34:32 migration active, transferred 1.1 GiB of 4.0 GiB VM-state, 399.1 MiB/s
2025-01-04 00:34:33 migration active, transferred 1.3 GiB of 4.0 GiB VM-state, 502.5 MiB/s
2025-01-04 00:34:34 migration active, transferred 1.6 GiB of 4.0 GiB VM-state, 243.0 MiB/s
2025-01-04 00:34:35 migration active, transferred 1.8 GiB of 4.0 GiB VM-state, 320.5 MiB/s
2025-01-04 00:34:36 migration active, transferred 2.2 GiB of 4.0 GiB VM-state, 462.0 MiB/s
2025-01-04 00:34:37 migration active, transferred 2.5 GiB of 4.0 GiB VM-state, 443.7 MiB/s
2025-01-04 00:34:38 average migration speed: 457.0 MiB/s - downtime 73 ms
2025-01-04 00:34:38 migration status: completed
2025-01-04 00:34:42 migration finished successfully (duration 00:00:18)
TASK OK

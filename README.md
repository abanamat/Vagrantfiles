# Vagrantfile-pvesoftraid

A Vagrant file used to create second drive with exactly the same size as the first drive and attach it to the same controller in VirtualBox

<details>
<summary>Proxmox VE: convert a single drive system to RAID1</summary>
Many thanks for the [article](https://www.prado.lt/how-to-set-up-software-raid1-on-a-running-lvm-system-incl-grub2-configuration-ubuntu-18-04)

Make sure the following packages are installed
```shell
apt-get update && apt install -y initramfs-tools mdadm
```
Show current situation
```shell
lsblk && blkid
fdisk -l /dev/sda /dev/sdc
pvdisplay && vgdisplay &&lvdisplay
```
Load a few kernel modules (to avoid a reboot):
```shell
modprobe linear
modprobe multipath
modprobe raid1
cat /proc/mdstat
```
Replicate  the  main  device's partition table on the specified second device
```shell
sgdisk -R /dev/sdc /dev/sda
```
Randomize  the  disk's GUID and all partitions' unique GUIDs
```shell
sgdisk -G /dev/sdc
```
> It is possible to make the ESP part of a RAID1 array, but doing so brings the risk
> of data corruption, and further considerations need to be taken when creating the ESP
> More at <https://outflux.net/blog/archives/2018/04/19/uefi-booting-and-raid1/>
> Also at <https://askubuntu.com/questions/355727/how-to-install-ubuntu-server-with-uefi-and-raid1-lvm>

Setting up a new partition for use as synced ESP <https://pve.proxmox.com/wiki/Host_Bootloader>
Format an empty partition `/dev/sdc2` as ESP
```shell
pve-efiboot-tool format /dev/sdc2
```
Make shure we have a visible FAT32 filesystem on the ESP
```shell
dd if=/dev/sdc2 bs=1K count=5 status=none | hexdump -C
```
To setup an existing, unmounted ESP located on `/dev/sdc2` for inclusion in Proxmox VE
kernel update synchronization mechanism, use the following:
```shell
pve-efiboot-tool init /dev/sdc2
```
To copy and configure all bootable kernels and keep all ESPs listed in
/etc/kernel/pve-efiboot-uuids in sync you just need to run:
```shell
pve-efiboot-tool refresh
```
Change the partition type for LVM partition on the new Drive
(`FD` for MBR and `A19D880F-05FC-4D3B-A006-743F0F84911E` for GPT)
```shell
sfdisk --part-type /dev/sdc 3 A19D880F-05FC-4D3B-A006-743F0F84911E
```
Check the microphone
```shell
fdisk -l /dev/sda /dev/sdc
```
Create LVM RAID array and use the placeholder missing for `/dev/sda3`
because the system is currently running on them
```shell
mdadm --create /dev/md3 -e 1.2 --level=1 --raid-disks=2 missing /dev/sdc3
```
Check the microphone
```shell
cat /proc/mdstat
```
Prepare LVM RAID array for LVM
```shell
pvcreate /dev/md3 && pvdisplay
```
Add LVM RAID array to existing volume group
```shell
vgextend pve /dev/md3 && pvdisplay && vgdisplay
```
##If got the error: Insufficient free space ..
> If the old Allocated PE is larger than the new Free PE
> And if it's all used – it can't fit on the new Drive
> So we can reduce the swap size with the following commands:
```shell
swapoff /dev/pve/swap
```
> Compute resize command (assume that PE Size: 4.00 MiB)
> ```shell
> pvs -q -o pv_name,pv_pe_count,pv_pe_alloc_count \
> |awk '{if($1=="/dev/sda3"){ope=$3}else if($1=="/dev/md3")\
> {fpe=$2}}END{if(int(ope)>int(fpe)){rs=(int(ope-fpe)*4);\
> print("lvresize -L-"rs"m /dev/pve/swap")}}'
> ```
> Run computed resize command
> ```shell
> mkswap /dev/pve/swap
> swapon /dev/pve/swap
> ```
Adjust `mdadm.conf` to the new situation
```shell
cp /etc/mdadm/mdadm.conf /etc/mdadm/mdadm.conf_orig
mdadm --examine --scan >> /etc/mdadm/mdadm.conf
cat /etc/mdadm/mdadm.conf
```
Update GRUB2 bootloader configuration
```shell
update-grub
# Next we adjust our ramdisk to the new situation
update-initramfs -u
```
Move the contents of LVM partition `/dev/sda3` to LVM RAID array
```shell
pvmove --atomic -i 2 /dev/sda3 /dev/md3
```
Remove `/dev/sda3` from the existing volume group
```shell
vgreduce pve /dev/sda3
```
And tell the system to not use `/dev/sda3` anymore for LVM
```shell
pvremove /dev/sda3 && pvdisplay
```
Change the partition type of `/dev/sda3` to Linux raid autodetect
```shell
sfdisk --part-type /dev/sda 3 A19D880F-05FC-4D3B-A006-743F0F84911E
```
Add `/dev/sda3` to LVM RAID array
```shell
mdadm --add /dev/md3 /dev/sda3
```
You should see that the LVM RAID array is being synchronized
```shell
watch cat /proc/mdstat
```
Afterwards we must make sure that the GRUB2 bootloader is installed on both hard drives
```shell
grub-install /dev/sda
grub-install /dev/sdc
```
Update GRUB2 bootloader configuration
```shell
update-grub
```
# Adjust ramdisk to the new situation
```shell
update-initramfs -u
```
Failed to send WATCHDOG=1 notification message: Transport endpoint is not connected
When trying to reboot for the first time. Workaround:
```shell
echo 1 > /proc/sys/kernel/sysrq
sync && echo b > /proc/sysrq-trigger
```
# TESTING
```shell
mdadm --manage /dev/md3 --fail /dev/sda3
mdadm --manage /dev/md3 --remove /dev/sda3
sgdisk -o /dev/sda
sync && reboot

sgdisk -R /dev/sda /dev/sdc
sgdisk -G /dev/sda
pve-efiboot-tool format /dev/sda2
pve-efiboot-tool init /dev/sda2
pve-efiboot-tool refresh

fdisk -l /dev/sdc /dev/sda
mdadm --zero-superblock /dev/sda3
mdadm -a /dev/md3 /dev/sda3
watch cat /proc/mdstat
grub-install /dev/sdc
grub-install /dev/sda
update-initramfs -u
```
</details>

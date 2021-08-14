# Arch Linux Daily Usage

## Package Maintenance

```bash
# Upgrade packages
sudo pacman -Syu

# Upgrade AUR packages only
sudo paru -Sua

# Print available AUR updates
sudo paru -Sua
```





## Anti-Virus

```bash
# Scan a file
clamscan myfile

# Scan home directories
clamscan --recursive --infected /home

# Scan entire system
sudo clamscan --recursive --infected --exclude-dir='^/sys|^/dev' /
```





## LVM Snapshot

Create LVM snapshot(s)

```bash
# Review existing LVM volumes
sudo lvs

# Review unallocated (free) space in volume group (i.e VFree)
sudo vgs

# Create the snapshot
sudo lvcreate -s -n snappy_root -L 50GB volgroup0/lv_root
sudo lvcreate -s -n snappy_home -L 50GB volgroup0/lv_home
# WARNING: Do not using 'snapshot' anywhere in the name!
# -s snapshot
# -n name
# -L size
```



Verify LVM snapshot(s)

```bash
# Verify
sudo lvs

# More details
sudo lvdisplay volgroup0/snappy_root
sudo lvdisplay volgroup0/snappy_home
```



Do work, i.e. Arch update, any major changes...

```bash
sudo pacman -Syu
```



Go forwards or backwards?

- Keep changes

```bash
# Drop snapshots
sudo lvremove volgroup0/snappy_root
sudo lvremove volgroup0/snappy_home
```

- Rollback system

```bash
# Boot to archiso

# Restore snapshot
sudo lvconvert --mergesnapshot volgroup0/snappy_root
sudo lvconvert --mergesnapshot volgroup0/snappy_home
   
# Restart - shutdown is better than reboot
poweroff
```



Extend snapshot size

```bash
# Review unallocated (free) space in volume group (i.e VFree)
sudo vgs

# Extend
sudo lvextend -L +2G volgroup0/snappy_root
sudo lvextend -L +2G volgroup0/snappy_home

# Verify
sudo lvs
sudo lvdisplay volgroup0/snappy_root
sudo lvdisplay volgroup0/snappy_home
```





## Rescue process

Pre-requisite steps to boot

1. Insert RescueCD USB flash drive
2. Boot
3. Hold down `F12` during boot, you will be taken to the BIOS boot manager, then select the Archiso USB



Mount file systems

```bash
# Find the SSD# E.g. the disk with the following/similair partitions#	/dev/sdx1  83  Linux#	/dev/sdx2  8e  Linux LVMfdisk -l | less# Open the (encrypted) LVM partition to work with itcryptsetup open --type luks /dev/sdx2 lvmEnter passphrase:# Mount rootmount /dev/volgroup0/lv_root /mntmount /dev/volgroup0/lv_home /mnt/homemount /dev/sdx1 /mnt/boot
```



Chroot

```bash
# Change root into systemarch-chroot /mnt
```



Fix problem

```bash
# Put tips here over time# Issue 1: device /dev/volgroup0/lv_root not found# Rebuild the initial ramdiskmkinitcpio -p linux-lts
```



Cleanup and leave

```bash
# exit chrootexit# unmount all partitionsumount -R /mntreboot# Remove the RescueCD USB flash drive
```





## Backup

Automated hourly snapshots of home

```bash
sudo touch /usr/local/bin/home_snapshots.shsudo chmod +x /usr/local/bin/home_snapshots.shsudo vim /usr/local/bin/home_snapshots.sh#!/bin/bash# configSOURCE="/home/peterg/" #dont forget trailing slash!DESTINATION="/mnt/samsung_ssd_evo_250gb/snapshots"OPTIONS="-rltgoiv --delay-updates --delete --chmod=a-w"MINIMUM_CHNAGES=20# run this process with real low priorityionice -c 3 -p $$renice +12  -p $$# syncecho $'\n\n\n*********************************************************************************\nBegin: '$(date)$'\n*********************************************************************************' >> $DESTINATION/rsync.logrsync $OPTIONS $SOURCE $DESTINATION/latest >> $DESTINATION/rsync.logecho $'*********************************************************************************\nEnd: '$(date)$'\n*********************************************************************************' >> $DESTINATION/rsync.log# check if enough has changed and if so make a hardlinked copy named as the dateCOUNT=$( wc -l $DESTINATION/rsync.log|cut -d" " -f1 )if [ $COUNT -gt $MINIMUM_CHNAGES ] ; then        DATETAG=$(date +%Y-%m-%d)        if [ ! -e $DESTINATION/$DATETAG ] ; then                cp -al $DESTINATION/latest $DESTINATION/$DATETAG                chmod u+w $DESTINATION/$DATETAG                mv $DESTINATION/rsync.log $DESTINATION/$DATETAG               chmod u-w $DESTINATION/$DATETAG         fifi
```

```bash
sudo vim /etc/systemd/system/home_snapshots.service[Unit]Description=Home snapshots with rsync[Service]Type=oneshotExecStart=/usr/local/bin/home_snapshots.sh
```

```bash
sudo vim /etc/systemd/system/home_snapshots.timer[Unit]Description=Run home snapshots with rsync every hour[Timer]OnCalendar=0/1:00:00Persistent=true[Install]WantedBy=timers.target
```

```bash
sudo systemctl enable home_snapshots.timersudo systemctl start home_snapshots.timer
```



Automated daily full backup of system

```bash
sudo touch /usr/local/bin/system_backup.shsudo chmod +x /usr/local/bin/system_backup.shsudo vim /usr/local/bin/system_backup.sh#!/bin/bash# configSOURCE="/"DESTINATION="/mnt/samsung_ssd_evo_250gb/backup"# syncecho $'\n\n\n*********************************************************************************\nBegin: '$(date)$'\n*********************************************************************************' >> $DESTINATION/rsync.logrsync -aAXv \        --delete \        --exclude=/dev/* \        --exclude=/proc/* \        --exclude=/sys/* \        --exclude=/tmp/* \        --exclude=/run/* \        --exclude=/mnt/* \        --exclude=/media/* \        --exclude=/lost+found \        --exclude=/var/lib/dhcpcd/* \        --exclude=/home/peterg/Downloads/* \        --exclude=/home/peterg/.thumbnails/* \        --exclude=/home/peterg/.cache/mozilla/* \        --exclude=/home/peterg/.cache/chromium/* \        --exclude=/home/peterg/.local/share/Trash/* \        $SOURCE \        $DESTINATION/archangel >> $DESTINATION/rsync.logecho $'*********************************************************************************\nEnd: '$(date)$'\n*********************************************************************************' >> $DESTINATION/rsync.log
```

```bash
sudo vim /etc/systemd/system/system_backup.service[Unit]Description=Full system backup with rsync[Service]Type=oneshotExecStart=/usr/local/bin/system_backup.sh
```

```bash
sudo vim /etc/systemd/system/system_backup.timer[Unit]Description=Run full system backup with rsync every day at 12:00pm[Timer]OnCalendar=*-*-* 12:00:00Persistent=true[Install]WantedBy=timers.target
```

```
sudo systemctl enable system_backup.timersudo systemctl start system_backup.timer
```



Destination

```bash
sudo mkdir -p /mnt/samsung_ssd_evo_250gb/backup/archangelsudo mkdir -p /mnt/samsung_ssd_evo_250gb/snapshots/latests
```



Restore

```bash
# Swap the destination and source in the following scripts/usr/local/bin/system_backup.sh/usr/local/bin/home_snapshots.sh
```





## Firmware update

https://wiki.archlinux.org/index.php/Lenovo_ThinkPad_T480#UEFI/Firmware_Updates

```bash
# To display all devices detected by fwupdfwupdmgr get-devices# Download the latest metadata from LVFSfwupdmgr refresh# List updates available for any devices on the systemfwupdmgr get-updates# Install updatesfwupdmgr update
```



UEFI upgrade

Warning: An update to your UEFI firmware may discard the current bootloader installation. It may be necessary to recreate the NVRAM entry (for example using efibootmgr) after the firmware update has been installed successfully.

https://wiki.archlinux.org/index.php/Fwupd

After the UEFI upgrade, you will have to restart the fwupd service

```bash
systemctl restart fwupd.service
```





## systemd-boot update

Manual update

```bash
bootctl update
```

Automatic update
Refer to pacman hook above (in installation guide)





## Add encrypted (LVM/Non LVM) disk to existing system

Find dev name(s) for new disk(s)

```bash
# Find main storage devicesudo fdisk -l | lessq
```



Partition disk

```bash
# Create new partition table and partitionsudo fdisk /dev/sdd# Choose ONE of these onlyo		# Dos partition tableg		# GPT partition tablenPartition Type (default p): [Enter]Partition number (default 1): [Enter]First sector (default xyz): [Enter]Last sector (default xyz): [Enter]# Review partitionp#/dev/sdd1			465.8GB	Linux# Save changes and exitw
```

(Optional) Repeat above step for each new disk, e.g. /dev/sde



Encrypt the partition

```bash
# Encrypt partitionsudo cryptsetup luksFormat /dev/sdd1Are you sure? (Type uppercase yes): YESEnter passphrase for /dev/sdd1: [enter passphrase here]Verify passphrase: [enter passphrase here]# Open the (encrypted) LVM partition to work with itsudo cryptsetup open --type luks /dev/sdd1 lvm2LVMEnter passphrase:# The decrypted container is now available at /dev/mapper/lvm2
```

(Optional) Repeat above step for each disk

```bash
# Encrypt LVM partitionsudo cryptsetup luksFormat /dev/sde1Are you sure? (Type uppercase yes): YESEnter passphrase for /dev/sde1: [enter passphrase here]Verify passphrase: [enter passphrase here]# Open the (encrypted) LVM partition to work with itsudo cryptsetup open --type luks /dev/sde1 lvm3Enter passphrase:# The decrypted container is now available at /dev/mapper/lvm3
```



Create key file

```bash
# Create a keyfile (must not have a newline in it)echo -n 'your_passphrase' > /path/to/keyfile.keysudo chown root:root /path/to/keyfile.keysudo chmod 400 /path/to/keyfile.key
```



Choose one of the following options ONLY

1. Setup LVM on new disk(s)

```bash
# Setup LVMpvcreate --dataalignment 1m /dev/mapper/lvm2# Create volume groupvgcreate volgroup1 /dev/mapper/lvm2# Create a logical volume for /lvcreate -l 100%FREE volgroup1 -n lv_data1# Format LVM logical volumesmkfs.ext4 /dev/volgroup1/lv_data1
```

(Optional) Repeat above step for each disk

```bash
# Setup LVMpvcreate --dataalignment 1m /dev/mapper/lvm3# Create volume groupvgcreate volgroup2 /dev/mapper/lvm3# Create a logical volume for /lvcreate -l 100%FREE volgroup2 -n lv_data1# Format LVM logical volumesmkfs.ext4 /dev/volgroup2/lv_data1
```

Setup auto-mount of filesystem(s)

```bash
# Get UUID of partitionsudo lsblk -f# Configure crypttab to open luks on bootsudo vim /etc/crypttabdata_disk_2	UUID={UUID value}	/path/to/keyfile.keydata_disk_3	UUID={UUID value}	/path/to/keyfile.key# Configure fstab to auto-mount diskssudo mkdir /mnt/samsung_ssd_evo_500gbsudo mkdir /mnt/samsung_ssd_evo_250gbsudo vim /etc/fstab/dev/volgroup1/lv_data_1	/mnt/samsung_ssd_evo_500gb	ext4	defaults	0 2/dev/volgroup2/lv_data_1	/mnt/samsung_ssd_evo_250gb	ext4	defaults	0 2
```

(Optional) Manual mount of filesystems

```bash
sudo mount /dev/volgroup1/lv_data_1 /mnt/samsung_ssd_evo_500gbsudo mount /dev/volgroup2/lv_data_1 /mnt/samsung_ssd_evo_250gb
```



2. Non-LVM prepare disk

```bash
# Format diskmkfs.ext4 /dev/sdd1
```

Setup auto-mount of filesystem(s)

```bash
# Get UUID of partition
sudo lsblk -f

# Configure crypttab to open luks on bLVM Snapshot
Create LVM snapshot(s)

￼
# Review existing LVM volumes
sudo lvs
​
# Review unallocated (free) space in volume group (i.e VFree)
sudo vgs
​
# Create the snapshot
sudo lvcreate -s -n snappy_root -L 50GB volgroup0/lv_root
sudo lvcreate -s -n snappy_home -L 50GB volgroup0/lv_home
# WARNING: Do not using 'snapshot' anywhere in the name!
# -s snapshot
# -n name
# -L size
Verify LVM snapshot(s)

# Verify
sudo lvs

# More details
sudo lvdisplay volgroup0/snappy_root
sudo lvdisplay volgroup0/snappy_home
Do work, i.e. Arch update, any major changes...

sudo pacman -Syu
Go forwards or backwards?

Keep changes

# Drop snapshots
sudo lvremove volgroup0/snappy_root
sudo lvremove volgroup0/snappy_home
Rollback system

# Boot to archiso

# Restore snapshot
sudo lvconvert --mergesnapshot volgroup0/snappy_root
sudo lvconvert --mergesnapshot volgroup0/snappy_home
   
# Restart - shutdown is better than reboot
poweroff
Extend snapshot size

# Review unallocated (free) space in volume group (i.e VFree)
sudo vgs

# Extend
sudo lvextend -L +2G volgroup0/snappy_root
sudo lvextend -L +2G volgroup0/snappy_home

# Verify
sudo lvs
sudo lvdisplay volgroup0/snappy_root
sudo lvdisplay volgroup0/snappy_home
Firmware update
https://wiki.archlinux.org/index.php/Lenovo_ThinkPad_T480#UEFI/Firmware_Updates

# To display all devices detected by fwupd
fwupdmgr get-devices

# Download the latest metadata from LVFS
fwupdmgr refresh

# List updates available for any devices on the system
fwupdmgr get-updates

# Install updates
fwupdmgr update
UEFI upgrade

Warning: An update to your UEFI firmware may discard the current bootloader installation. It may be necessary to recreate the NVRAM entry (for example using efibootmgr) after the firmware update has been installed successfully.

https://wiki.archlinux.org/index.php/Fwupd

After the UEFI upgrade, you will have to restart the fwupd service

systemctl restart fwupd.service
systemd-boot update
Manual update

bootctl update
Automatic update
Refer to pacman hook above (in installation guide)

Add encrypted (LVM/Non) disk to existing system
Find dev name(s) for new disk(s)

# Find main storage device
sudo fdisk -l | less
q
Partition disk

# Create new partition table and partition
sudo fdisk /dev/sdd

# Choose ONE of these only
o		# Dos partition table
g		# GPT partition table

n
Partition Type (default p): [Enter]
Partition number (default 1): [Enter]
First sector (default xyz): [Enter]
Last sector (default xyz): [Enter]

# Review partition
p
#/dev/sdd1			465.8GB	Linux

# Save changes and exit
w
(Optional) Repeat above step for each new disk, e.g. /dev/sde

Encrypt the partition

# Encrypt partition
sudo cryptsetup luksFormat /dev/sdd1
Are you sure? (Type uppercase yes): YES
Enter passphrase for /dev/sdd1: [enter passphrase here]
Verify passphrase: [enter passphrase here]

# Open the (encrypted) LVM partition to work with it
sudo cryptsetup open --type luks /dev/sdd1 lvm2LVM
Enter passphrase:

# The decrypted container is now available at /dev/mapper/lvm2
(Optional) Repeat above step for each disk

# Encrypt LVM partition
sudo cryptsetup luksFormat /dev/sde1
Are you sure? (Type uppercase yes): YES
Enter passphrase for /dev/sde1: [enter passphrase here]
Verify passphrase: [enter passphrase here]

# Open the (encrypted) LVM partition to work with it
sudo cryptsetup open --type luks /dev/sde1 lvm3
Enter passphrase:

# The decrypted container is now available at /dev/mapper/lvm3
Create key file

# Create a keyfile (must not have a newline in it)
echo -n 'your_passphrase' > /path/to/keyfile.key
sudo chown root:root /path/to/keyfile.key
sudo chmod 400 /path/to/keyfile.key
Choose one of the following options ONLY

Setup LVM on new disk(s)

# Setup LVM
pvcreate --dataalignment 1m /dev/mapper/lvm2

# Create volume group
vgcreate volgroup1 /dev/mapper/lvm2

# Create a logical volume for /
lvcreate -l 100%FREE volgroup1 -n lv_data1

# Format LVM logical volumes
mkfs.ext4 /dev/volgroup1/lv_data1
(Optional) Repeat above step for each disk

# Setup LVM
pvcreate --dataalignment 1m /dev/mapper/lvm3

# Create volume group
vgcreate volgroup2 /dev/mapper/lvm3

# Create a logical volume for /
lvcreate -l 100%FREE volgroup2 -n lv_data1

# Format LVM logical volumes
mkfs.ext4 /dev/volgroup2/lv_data1
Setup auto-mount of filesystem(s)

# Get UUID of partition
sudo lsblk -f

# Configure crypttab to open luks on boot
sudo vim /etc/crypttab
data_disk_2	UUID={UUID value}	/path/to/keyfile.key
data_disk_3	UUID={UUID value}	/path/to/keyfile.key

# Configure fstab to auto-mount disks
sudo mkdir /mnt/samsung_ssd_evo_500gb
sudo mkdir /mnt/samsung_ssd_evo_250gb

sudo vim /etc/fstab
/dev/volgroup1/lv_data_1	/mnt/samsung_ssd_evo_500gb	ext4	defaults	0 2
/dev/volgroup2/lv_data_1	/mnt/samsung_ssd_evo_250gb	ext4	defaults	0 2
(Optional) Manual mount of filesystems

sudo mount /dev/volgroup1/lv_data_1 /mnt/samsung_ssd_evo_500gb
sudo mount /dev/volgroup2/lv_data_1 /mnt/samsung_ssd_evo_250gb
Non-LVM prepare disk

# Format disk
mkfs.ext4 /dev/sdd1
Setup auto-mount of filesystem(s)

# Get UUID of partition
sudo lsblk -f

# Configure crypttab to open luks on boot
sudo vim /etc/crypttab
data_disk_2	UUID={UUID value}	/path/to/keyfile.key
data_disk_3	UUID={UUID value}	/path/to/keyfile.key

# Configure fstab to auto-mount disks
sudo mkdir /mnt/samsung_ssd_evo_500gb
sudo mkdir /mnt/samsung_ssd_evo_250gb

sudo vim /etc/fstab
/dev/sdd1	/mnt/samsung_ssd_evo_500gb	ext4	defaults	0 2
/dev/sdd1	/mnt/samsung_ssd_evo_250gb	ext4	defaults	0 2
(Optional) Manual mount of filesystems

sudo mount /dev/sdd1 /mnt/samsung_ssd_evo_500gb
sudo mount /dev/sdd1 /mnt/samsung_ssd_evo_250gb
 oot
sudo vim /etc/crypttab
data_disk_2	UUID={UUID value}	/path/to/keyfile.key
data_disk_3	UUID={UUID value}	/path/to/keyfile.key

# Configure fstab to auto-mount disks
sudo mkdir /mnt/samsung_ssd_evo_500gb
sudo mkdir /mnt/samsung_ssd_evo_250gb

sudo vim /etc/fstab
/dev/sdd1	/mnt/samsung_ssd_evo_500gb	ext4	defaults	0 2
/dev/sdd1	/mnt/samsung_ssd_evo_250gb	ext4	defaults	0 2
```

(Optional) Manual mount of filesystems

```bash
sudo mount /dev/sdd1 /mnt/samsung_ssd_evo_500gb
sudo mount /dev/sdd1 /mnt/samsung_ssd_evo_250gb
```

 

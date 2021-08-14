# Arch Linux Installation Guide

## Notes

This installation guide was designed for the following system:

**Laptop: Lenovo T480**

- UEFI firmware

- RAM: 32 GB

- Disk: 500 GB NvME



The following where used extensively:

Non-official installation guide (on YouTube) - https://www.youtube.com/watch?v=Lq4cbp5AOZM
Official installation guide -  https://wiki.archlinux.org/index.php/installation_guide
BIOS specific instructions - https://gist.github.com/mjnaderi/28264ce68f87f52f2cabb823a503e673






## Requirements

| Topic              | Solution\Category   | Sub-category             |
| ------------------ | ------------------- | ------------------------ |
| Storage Encryption | LUKS                |                          |
| Boot loader        | UEFI - systemd-boot |                          |
| Security           | MAC - AppArmor      |                          |
|                    | Firewall - UFW      |                          |
| Backup             | Home                | rsync - hourly snapshots |
|                    | System              | rsync - daily backups    |
|                    | Point-in-time       | LVM Snapshots            |
|                    | System              | ddrescue                 |





## Pre-install

Create Arch Linux live media

Download site: https://www.archlinux.org/download/

1. Download the Arch Linux iso from an Australian mirror sites
2. Download the PGP signature and Checksums (MD5, SHA1)
   Note, for security reasons, do **NOT** download these file from the mirror site!

Note: keep the Archiso USB flash drive as this will be used as a rescue USB tool!



Verify the image

```bash
gpg --keyserver-options auto-key-retrieve --verify archlinux-version-x86_64.iso.sig
```



Verify the (MD5 and/or SHA1) checksum(s)

```bash
# From Ubuntu
md5sum archlinux-version-x86_64.iso------
```



Using Etcher, burn the iso image onto the USB flash drive



Prepare Firmware settings

Verify the following UEFI Firmware settings:

1. Boot UEFI only: ON

2. Secure Boot: OFF

------





## Install

### Arch

Boot Arch Linux Live USB flash drive.
Note you will need to hold down `F12` during boot to go directly to the firmware boot manager



Verify boot mode

```bash
# Verify system booted in UEFI mode and UEFI variables are accessible
ls /sys/firmware/efi/efivars
# if the directory exists, the system is booted in UEFI mode
```



Connect to internet

```bash
# Check NIC is listed and enabled
ip link

# Verify connectivity
ping archlinux.org

# If you do not have internet access
dhcpcd
```



Update system clock

```bash
timedatectl set-ntp true
```



##### Disks

Layout

```
+-----------------------------------------------------------------------+ +----------------+ 
| Logical volume 1      | Logical volume 2      | Logical volume 3      | | Boot partition |
|                       |                       |                       | |                |
| /                     | /home                 | /[SWAP]               | | /boot          |
|                       |                       |                       | |                |
| /dev/volgroup0/lv_root| /dev/volgroup0/lv_home| /dev/volgroup0/lv_swap| |                |
|_ _ _ _ _ _ _ _ _ _ _ _|_ _ _ _ _ _ _ _ _ _ _ _|_ _ _ _ _ _ _ _ _ _ _ _| |                |
|                                                                       | |                |
|                         LUKS2 encrypted partition                     | |                |
|                           /dev/nvme0n1p2                              | | /dev/nvme0n1p1 |
+-----------------------------------------------------------------------+ +----------------+ 
```

Partition

```bash
# Find main storage device
fdisk -l | less
q

# # Create new partition table and partitions (on /dev/nvme0n1)
fdisk /dev/nvme0n1
p
g			# GPT partition table
p
ctr-l

# Partition 1: Boot
# Arch Linux suggested size: 260–512 MiB
# Therefore, 512 MiB = approx. 537 MB
n
Partition number (default 1): [Enter]
First sector (default 2048): [Enter]
Last sector (default xyz): 537M
(optional) Do you want to remove the signature?: y
t
Partition number (default 1): [Enter]
Partition type: 1
ctr-l
# ignore signature error message (if any)

# Partition 2: LVM
n
Partition number (default 2): [Enter]
First sector (default xyz): [Enter]
Last sector (default xyz): [Enter]
t
Partition number (default 2): [Enter]
Partition type: 30
ctr-l

# Review partitions 
p
#/dev/nvme0n1p1		537MB	EFI System
#/dev/nvme0n1p2		476GB	Linux LVM

# Save changes and exit
w
```

Format boot partition

```bash
# Review partitions again (before continuing)
lsblk

# Format (with FAT32)
mkfs.fat -F32 /dev/nvme0n1p1
```

Encrypt a partition for LVM

```bash
# Encrypt LVM partitioncryptsetup luksFormat /dev/nvme0n1p2Are you sure? (Type uppercase yes): YESEnter passphrase for /dev/nvme0n1p2: [enter passphrase here]Verify passphrase: [enter passphrase here]# Open the (encrypted) LVM partition to work with itcryptsetup open --type luks /dev/nvme0n1p2 lvmEnter passphrase:# The decrypted container is now available at /dev/mapper/lvm
```

Setup LVM

```bash
# Notes on sizing# ---------------# Total volgroup0 size: 476GB# Used:# 	1. lv_root - 50GB# 	2. lv_home - 250GB# 	3. lv_swap - 38GB## Remaining volgroup0 free space: 138GB# Setup LVMpvcreate --dataalignment 1m /dev/mapper/lvmvgcreate volgroup0 /dev/mapper/lvm# Create a logical volume for /lvcreate -L 50GB volgroup0 -n lv_root# Create a logical volume for /home# Note: using LVM snapshots so leave free space in volume grouplvcreate -L 500GB volgroup0 -n lv_home# Create a logical volume for swap partition# Ubuntu swap size recommendations for 16GB RAM:# 	Without hiberation: 4GB# 	With hiberation: 	20GBlvcreate -L 20GB volgroup0 -n lv_swap# Format LVM logical volumesmkfs.ext4 /dev/volgroup0/lv_rootmkfs.ext4 /dev/volgroup0/lv_homemkswap /dev/volgroup0/lv_swap
```

Mount file systems

```bash
# Mount rootmount /dev/volgroup0/lv_root /mnt# Create directory for the home partition (under root) and mount itmkdir /mnt/homemount /dev/volgroup0/lv_home /mnt/home# Create directory for the boot partition (under root) and mount itmkdir /mnt/bootmount /dev/sdc1 /mnt/boot# Create /etc directory to hold settings filesmkdir /mnt/etc# Activate swap logical volumeswapon /dev/volgroup0/lv_swap
```



##### Select mirrors

```bash
# Move the geographically closest mirrors to the top of the list (i.e. Australia)nano /etc/pacman.d/mirrorlist# Update package repository indexpacman -Syyy
```



##### Install essential packages

```bash
pacstrap /mnt base linux linux-headers linux-lts linux-lts-headers linux-firmware lvm2 dkms
```



##### Fstab

```bash
# Generate an fstab filegenfstab -U -p /mnt >> /mnt/etc/fstab# Check the resulting filecat /mnt/etc/fstab
```



##### Chroot

```bash
# Change root into the new systemarch-chroot /mnt
```



##### Install networking (and related) packages

```bash
pacman -S networkmanager plasma-nm wpa_supplicant wireless_tools netctl dhcpcd dialog ufw# accept all defaults# Enable networkingsystemctl disable dhcpcdsystemctl enable NetworkManager
```



##### Install Firmware tools

```bash
pacman -S intel-ucode fwupd efibootmgr
```



##### Install filesystems support

```bash
pacman -S ntfs-3g exfat-utils
```



##### Install optional packages

```bash
pacman -S base-devel openssh util-linux usbutils gvim nano# accept all defaults
```



##### Set Time zone and clock

```bash
# Set the time zoneln -sf /usr/share/zoneinfo/Australia/Sydney /etc/localtime# Set the hardware clockhwclock --systohc --utc
```



##### Set localisation

```bash
# Edit locale filevim /etc/locale.gen# Uncomment your localeen_AU.UTF-8 UTF-8# Generate localelocale-gen# Set the localevim /etc/locale.confLANG=en_AU.UTF-8
```



##### Create hostname

```bash
echo myhostname > /etc/hostname
```



##### Create hosts file

```bash
vim /etc/hosts127.0.0.1	localhost::1			localhost127.0.1.1	myhostname.localdomain	myhostname
```



##### Enable periodic TRIM

```bash
# Verify TRIM supportlsblk --discard# proceed if supported# LVM# Change the value of issue_discards option from 0 to 1vim /etc/lvm/lvm.confissue_discards = 1# Enable periodic trim weeklysystemctl enable fstrim.timer
```



##### Install HP printer drivers

```bash
# Printer managerssudo pacman -S print-manager system-config-printer cups# HP printer driverssudo pacman -S hplip# Printer servicesudo systemctl enable cups.servicesudo systemctl enable cups.socket# Setup printer via command linehp-setup -i
```



##### Install graphics driver

Install both nouveau and nvidia

```bash
sudo pacman -S mesa nvidia nvidia-lts nvidia-utils
```



##### Initramfs

Configure

```bash
vim /etc/mkinitcpio.conf# The MODULES line should read:#---With Nouveau--------------------------------------------------------MODULES=(ext4 nouveau atkbd)#-----------------------------------------------------------------------# OR#---With NVIDIA---------------------------------------------------------MODULES=(ext4 nvidia atkbd)# Notes:# 1. Lenovo ThinkPad T480 specific instructions: 'atkbd'#    https://wiki.archlinux.org/index.php/Lenovo_ThinkPad_T480#Encryption_and_keyboard# 2. nvidia related modules refer to:#    https://wiki.archlinux.org/index.php/NVIDIA#DRM_kernel_mode_setting# On the HOOKS line, include the following:#	1. 'encrypt' and 'lvm2' to go between 'block filesystems'#	2. 'resume' to go AFTER 'udev' and 'lvm2'#	3. Move 'keyboard' to go BEFORE autodetect#	4. Add 'keymap' and 'consolefont' after 'keyboard'## The HOOKS line should look like the following:HOOKS=(base udev keyboard autodetect keymap consolefont modconf block lvm2 encrypt resume filesystems fsck)# IMPORTANT: Plug-in any specific devices, e.g. wireless keyboard... BEFORE regenerating ramdisk, so devices can be recognised!# Re-generate ramdiskmkinitcpio -p linux-lts
```

Add firmware

```bash
# If warnings are reported for missing firmware (i.e. below), then review and if not a false positive then install firmware.#	`==> WARNING: Possibly missing firmware for module: wd719x`#	`==> WARNING: Possibly missing firmware for module: aic94xx`cd ~/AURgit clone https://aur.archlinux.org/aic94xx-firmware.gitcd aic94xx-firmware/makepkg -sigit clone https://aur.archlinux.org/wd719x-firmware.gitcd wd719x-firmware/makepkg -si
```

Re-generate ramdisk:

```bash
# Re-generate ramdiskmkinitcpio -p linux-lts
```



##### Lenovo ThinkPad T480 specific

Thermal Throttling prevention

```bash
# https://wiki.archlinux.org/index.php/Lenovo_ThinkPad_T480#Power_management/Throttling_issuespacman -S throttledsystemctl enable lenovo_fix.service
```

TrackPoint and Trackpad

```bash
# https://wiki.archlinux.org/index.php/Lenovo_ThinkPad_T480#TrackPoint_and_Touchpadvim /etc/modprobe.d/psmouse.confoptions psmouse synaptics_intertouch=1
```

Special buttons - Re-map unsupported keys

```bash
# https://wiki.archlinux.org/index.php/Lenovo_ThinkPad_T480#Special_buttons# Edit udev rulevim /etc/udev/hwdb.d/90-thinkpad-keyboard.hwdbevdev:name:ThinkPad Extra Buttons:dmi:bvn*:bvr*:bd*:svnLENOVO*:pn* KEYBOARD_KEY_45=prog1 KEYBOARD_KEY_49=prog2 # Update hwdbudevadm hwdb --updateudevadm trigger --sysname-match="event*"# Button names will be `XF86Launch2` (KEY_KEYBOARD) and `XF86Launch1` (KEY_FAVORITES)
```

Screen backlight

```bash
# https://wiki.archlinux.org/index.php/Lenovo_ThinkPad_T480#Screen_backlight# Install a backward-compatible xbacklight replacement based on ACPIsudo pacman -S acpilight#Add current user to the video groupusermod -a -G video myusername# Add the following udev rule:vim /etc/udev/rules.d/90-backlight.rulesSUBSYSTEM=="backlight", ACTION=="add", \  RUN+="/bin/chgrp video /sys/class/backlight/%k/brightness", \  RUN+="/bin/chmod g+w /sys/class/backlight/%k/brightness"SUBSYSTEM=="leds", ACTION=="add", KERNEL=="*::kbd_backlight", \  RUN+="/bin/chgrp video /sys/class/leds/%k/brightness", \  RUN+="/bin/chmod g+w /sys/class/leds/%k/brightness"
```



##### Set root password

```bash
passwd
```



##### Create non-root user

```bash
useradd -m -g users -G wheel myusernamepasswd myusername
```



##### Setup sudo

```bash
# If not installedpacman -S sudo# ConfigureEDITOR=vim visudo# Uncomment the following line:%wheel ALL=(ALL) ALL
```



##### Bootloader

Setup and configure systemd-boot

```bash
# Install bootloaderbootctl --path=/boot/ install# This will copy the systemd-boot boot loader to the EFI partition.# Two identical binaries will be transferred to the ESP:#	esp/EFI/systemd/systemd-bootx64.efi and#	esp/EFI/BOOT/BOOTX64.EFI# Configure boot loadervim /boot/loader/loader.confdefault  arch-lts.conftimeout  4console-mode maxeditor   no# Configure boot loader entriesvim /boot/loader/entries/arch.conftitle   Arch Linuxlinux   /vmlinuz-linuxinitrd  /intel-ucode.imginitrd  /initramfs-linux.img#---With Nouveau--------------------------------------------------------options cryptdevice=UUID={UUID}:lvm:allow-discards root=/dev/volgroup0/lv_root resume=/dev/volgroup0/lv_swap apparmor=1 lsm=lockdown,yama,apparmor rw quiet# OR#---With NVIDIA---------------------------------------------------------options cryptdevice=UUID={UUID}:lvm:allow-discards root=/dev/volgroup0/lv_root resume=/dev/volgroup0/lv_swap nvidia-drm.modeset=1 apparmor=1 lsm=lockdown,yama,apparmor rw quiet#-----------------------------------------------------------------------vim /boot/loader/entries/arch-lts.conftitle   Arch Linux LTSlinux   /vmlinuz-linux-ltsinitrd  /intel-ucode.imginitrd  /initramfs-linux-lts.img#---With Nouveau--------------------------------------------------------options cryptdevice=UUID={UUID}:lvm:allow-discards root=/dev/volgroup0/lv_root resume=/dev/volgroup0/lv_swap apparmor=1 lsm=lockdown,yama,apparmor rw quiet#-----------------------------------------------------------------------# OR#---With NVIDIA---------------------------------------------------------options cryptdevice=UUID={UUID}:lvm:allow-discards root=/dev/volgroup0/lv_root resume=/dev/volgroup0/lv_swap nvidia-drm.modeset=1 apparmor=1 lsm=lockdown,yama,apparmor rw quiet#-----------------------------------------------------------------------# In order to get the {UUID} run the following command in vim and paste above::r! blkid /dev/nvmen0p2# Notes:# If problems, use the following device name# /dev/nvmen0p2:volgroup0
```



##### XDG

```bash
sudo pacman -S xdg-user-dirssudo vim /etc/profile# Add the following line at the end of the file# XDGexport XDG_CONFIG_HOME="$HOME/.config"
```



##### Post-reboot review

```bash
# exit chrootexit# unmount all partitionsumount -R /mnt# Disable swapswapoff -apoweroff# Remove Arch Linux Live USB# Restart PC# 1. Verify boot# 2. Verify swap volume is enabled with `top`# 3. Verify fstab in `/etc/fstab`
```





### X Windows System

Install X

```bash
sudo pacman -S xorg-server xorg-xinit
```



Nvidia Graphics driver

Start X

```bash
startx# If X does not start and complains about config, create configsudo nvidia-xconfig
```

Configure graphics

```bash
sudo sunvidia-settings# save config file
```



​	OR



Nouveau driver

Check that you do not have Nouveau disabled using any modprobe blacklisting technique within
`/etc/modprobe.d/`

`/usr/lib/modprobe.d/nvidia.conf`

`/usr/lib/modprobe.d/nvidia-lts.conf`



Check if `/etc/X11/xorg.conf` exists and is referencing `nvidia` driver. If so, rename the file.



Reboot

```bash
reboot
```

Check graphics driver used by X

```bash
lspci -v | less
```

Look at what driver is in use under your VGA controller





## Post-install

### Security

Firewall

```bash
# Installsudo pacman -S ufw# Start servicesudo systemctl enable ufw.service# One-off enablesudo ufw enable# Querysudo ufw status
```



Mandatory Access Control (AppArmor)

```bash
# NO ACTION REQUIRED HERE! For information purposes only!# Bootloader configure for kernel to use AppArmor has been included in the above Bootloader section for both UEFI and BIOS.# E.g. UEFI#vim /boot/loader/entries/arch-lts.conf#vim /boot/loader/entries/arch.conf#options cryptdevice=UUID={UUID}:lvm:allow-discards root=/dev/volgroup0/lv_root resume=/dev/volgroup0/lv_swap nvidia-drm.modeset=1 apparmor=1 lsm=lockdown,yama,apparmor rw quiet# E.g. BIOS section#vim /etc/default/grub#GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 cryptdevice=UUID={device-UUID}:volgroup0:allow-discards root=/dev/volgroup0/lv_root resume=/dev/volgroup0/lv_swap nvidia-drm.modeset=1 apparmor=1 lsm=lockdown,yama,apparmor quiet"#grub-mkconfig -o /boot/grub/grub.cfg
```

Userspace

```bash
# Install tools and libraries to control AppArmorsudo pacman -S apparmor# Enable profiles on startupsudo systemctl enable apparmor.service# Reboot# Display statussudo aa-enabled# should display 'Yes'# Verify statussudo aa-status
```



Anti-Virus

```bash
# Installsudo pacman -S clamav# Run (before starting the service)freshclam# Enable virus definition servicesudo systemctl enable clamav-freshclam.servicesudo systemctl start clamav-freshclam.service# Testcurl https://secure.eicar.org/eicar.com.txt | clamscan -# The output must include:#stdin: Win.Test.EICAR_HDB-1 FOUND# Add more databases/signatures repositories# Install from AURpython-fangfrisch# Create database structuresudo -u clamav /usr/bin/fangfrisch --conf /etc/fangfrisch/fangfrisch.conf initdb# Enablesudo systemctl enable fangfrisch.timer
```



### General

WiFi

```bash
wifi-menu
```

Mirrorlist

```bash
# 1. Visit official mirrorlist https://www.archlinux.org/mirrorlist/# 2. Generate list# 3. Paste list into the following file:sudo vim /etc/pacman.d/mirrorlist# 4. Un-comment all servers
```

Pacman disable compression

```bash
sudo vim /etc/makepkg.conf#PKGEXT='.pkg.tar.xz'#PKGEXT='.pkg.tar.gz'PKGEXT='.pkg.tar'SRCEXT='.src.tar.gz'
```

Pacman hooks

arch-audit

```bash
sudo vim /usr/share/libalpm/hooks/arch-audit.hook[Trigger]Operation = InstallOperation = UpgradeOperation = RemoveType = PackageTarget = *[Action]Depends = curlDepends = opensslDepends = arch-auditWhen = PostTransactionExec = /usr/bin/arch-audit -c -u -C 'always'
```

orphans

```bash
sudo vim /usr/share/libalpm/hooks/orphans.hook[Trigger]Operation = InstallOperation = UpgradeOperation = RemoveType = PackageTarget = *[Action]Description = Searching for orphaned packages...When = PostTransactionExec = /usr/bin/bash -c "/usr/bin/pacman -Qtd || /usr/bin/echo '==> no orphans found.'"#Exec = /usr/bin/bash -c "[[ if $(/usr/bin/pacman -Qtdq) = 0 ]] && /usr/bin/pacman -Rns $(/usr/bin/pacman -Qtdq) || /usr/bin/echo '==> No orphaned packages found.'"
```

systemd-boot automatic update

```bash
sudo vim /etc/pacman.d/hooks/100-systemd-boot.hook[Trigger]Type = PackageOperation = UpgradeTarget = systemd[Action]Description = Updating systemd-bootWhen = PostTransactionExec = /usr/bin/bootctl update
```

nvidia

```bash
sudo vim /usr/share/libalpm/hooks/nvidia.hook[Trigger]Operation=InstallOperation=UpgradeOperation=RemoveType=PackageTarget=nvidiaTarget=nvidia-ltsTarget=linuxTarget=linux-lts[Action]Description=Update Nvidia module in initcpioDepends=mkinitcpioWhen=PostTransactionNeedsTargetsExec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
```

OneDrive Free Client

```bash
# Install dependenciessudo pacman -S curl sqlite dmd libnotify# Install from AURonedrive-abraunegg# Create log directorysudo mkdir /usr/log/onedrive# Synchronize from cloud to your local ~/OneDriveonedrive --synchronize --verbose --enable-logging# Choose one of the following ONLY# 1. OFFICIAL# Setup servive (does NOT works with libnotify)# Non-root, so run WITHOUT sudosystemctl enable onedrive@peterg.servicesystemctl start onedrive@peterg.service# View status on running servicesystemctl status onedrive@peterg.service# 2. NON-OFFICIAL# Setup service (works with libnotify)systemctl --user enable onedrive.servicesystemctl --user start onedrive.service# View status on running servicesystemctl --user status onedrive.service
```



### Development

Postgresql

```bash
# Installsudo pacman -S postgresql pgadmin4# Switch to the PostgreSQL usersudo -iu postgresinitdb -D /var/lib/postgres/dataexit# Start the postgres servicesudo systemctl start postgresql.service sudo systemctl enable postgresql.service
```



Anaconda
AUR

```bash
git clone https://aur.archlinux.org/anacondacd anacondamakepkg -si# To activate and deactivate the anaconda enviroment.source /opt/anaconda/bin/activate rootsource /opt/anaconda/bin/deactivate root
```

​	or

Binary

```bash
# Downloadhttps://www.anaconda.com/distribution/#download-section# Install anacondabash ~/Downloads/Anaconda3-2020.02-Linux-x86_64.sh
```

Workaround & Update

```bash
# Workaround for bug in Archvim ~/anaconda3/lib/python3.7/lib/python3.7/site-packages/anaconda_navigator/api/external_apps/vscode.py	def _find_linux_install_dir:		DISTRO_NAME = None  ## <---- add this line only!		for distro in self.distro_map.keys		...		...# Exit terminalexit# Re-launch terminalsudo conda update anaconda-navigator
```

Fix Anaconda path (if installed in /opt/anaconda)

```bash
echo 'export PATH="/opt/anaconda/bin:$PATH"'>>~/.bashrc source .bashrc
```



Java

```bash
sudo pacman -S jre-openjdk java-openjfx# Note: Required by some applications, e.g. Cryptomator.
```





### Virtualisation

Step 1: Install KVM packages

First step is installing all packages needed to run KVM:

```bash
sudo pacman -S qemu dnsmasq vde2 bridge-utils openbsd-netcat
```



Also install *ebtables*  and *iptables* packages:

```
sudo pacman -S ebtables iptables
```



Install aquemu from AUR

```bash
cd AURgit clone https://aur.archlinux.org/aqemu.gitcd aqemumakepkg -si
```

Note, not installing: virt-manager virt-viewer



Step 2: Install libguestfs

Install dmidecode

```bash
sudo pacman -S dmidecode
```

Install libguestfs from AUR

```bash
cd AURgit clone https://aur.archlinux.org/libguestfs.gitcd libguestfsmakepkg -si
```



Step 3: Start KVM libvirt service

Once the installation is done, start and enable libvirtd service to start at boot:

```
sudo systemctl enable libvirtd.servicesudo systemctl start libvirtd.service
```



Verify service is running:

```bash
sudo systemctl status libvirtd.service
```



Step 4: Enable normal user account to use KVM

Since we want to use our standard Linux user account to manage KVM, let’s configure KVM to allow this.

Open the file */etc/libvirt/libvirtd.conf* for editing.

```
sudo pacman -S vimsudo vim /etc/libvirt/libvirtd.conf
```

Set the UNIX domain socket group ownership to libvirt, (around line **85**)

```
unix_sock_group = "libvirt"
```

Set the UNIX socket permissions for the R/W socket (around line **102**)

```
unix_sock_rw_perms = "0770"
```

Add your user account to libvirt group.

```
sudo usermod -a -G libvirt $(whoami)newgrp libvirt
```

Restart libvirt daemon.

```
sudo systemctl restart libvirtd.service
```



Step 5: Enable Nested Virtualization (Optional)

Nested Virtualization feature enables you to run Virtual Machines  inside a VM. Enable Nested virtualization for kvm_intel by enabling  kernel module as shown.

```bash
sudo modprobe -r kvm_intel
```

Reboot

```bash
sudo modprobe kvm_intel nested=1
```

To make this configuration persistent,run:

```
echo "options kvm-intel nested=1" | sudo tee /etc/modprobe.d/kvm-intel.conf
```

Confirm that Nested Virtualization is set to Yes:

```bash
$ systool -m kvm_intel -v | grep nested
    nested              = "Y"
    nested_early_check  = "N"
$ cat /sys/module/kvm_intel/parameters/nested 
Y
```



For vm guests

1. Install spice-vdagent

   ```bash
   sudo pacman -S spice-vdagent
   sudo apt install spice-vdagent
   sudo yum install spice-vdagent
   ```

2. Ensure virtio devices (for network and disks) are used

3. If the *Video QXL* hardware prevents X Windowss from launching, change video hardware to *Video Virtio*

4. Add new Input hardware : EvTouch USB Graphics Tablet, for mouse integration



To Do

```bash
# Create a storage pool in home
# Launch **virt-manager**
# Menu /Edit Connection Details/Storage tab

# Update existing guests location to home
# Update existing VMs to reside in storage pool in home
```



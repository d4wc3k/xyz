
- ##  Steps

___

1. #### Install required packages


	Update packages database:

	```bash
	# switch to root
	sudo -i
	# update package database
	apt update
	```

	Install packages:

	```bash
	apt install neovim gdisk debootstrap arch-install-scripts cryptsetup lvm2 dosfstools
	```

___

2. #### Partition disk

	```bash
	gdisk /dev/sda
	```

____

3. #### Prepare encrypted partition and lvm

   Encryption:

   ```bash
  	cryptsetup --type luks2 -v --verify-passphrase --cipher aes-xts-plain64 --key-size 512 --key-slot 0 --hash sha256 --iter-time 11654 --pbkdf argon2id  --pbkdf-memory 524288 --pbkdf-parallel 4 --use-random --label xyz-locked --timeout 60 luksFormat /dev/sda3
   ```

	Open encrypted container:

	``` bash
	cryptsetup open /dev/sda3 xyz-unlocked
	```

	LVM:

	```bash
	pvcreate /dev/mapper/xyz-unlocked
	vgcreate vg /dev/mapper/xyz-unlocked
	lvcreate -L 30G vg --name root
	lvcreate -L 4G vg --name swap
	lvcreate -L 40G vg --name home
	```

___

4. #### Format and mount partitions

   ```bash
   #
   # dir for chroot
   mkdir -p /target
   # root partition
   mkfs.ext4 -L RootFs /dev/mapper/vg-root
   mount /dev/mapper/vg-root /target
   # boot partition
   mkdir -p /target/boot
   mkfs.ext2 -L BootFs /dev/sda2
   mount /dev/sda2 /target/boot
   # efi partition
   mkdir -p /target/boot/efi
   mkfs.vfat -F 32 -n EFI /dev/sda1
   mount /dev/sda1 /target/boot/efi
   # home partition
   mkdir -p /target/home
   mkfs.ext4 -L HomeFs /dev/mapper/vg-home
   mount /dev/mapper/vg-home /target/home
   # swap partition
   mkswap -L SWAP /dev/mapper/vg-swap
   swapon /dev/mapper/vg-swap
   #
   ```

____

5. #### Install base system with deboostrap

	```bash
	debootstrap --verbose --arch=amd64 --include=ca-certificates,bash-completion trixie /target http://ftp.pl.debian.org/debian/
	```

____

6. #### Prepare apt sources file for new installed system

	```bash
	#
	vim /target/etc/apt/sources.list
	#
	# Content of file:
	########################################################################################################################
	##														      #
	## trixie													      #
	deb http://ftp.pl.debian.org/debian trixie main contrib non-free non-free-firmware				      #
	# deb-src http://ftp.pl.debian.org/debian trixie main contrib non-free non-free-firmware			      #
	#														      #	
	## trixie-backports												      #
	# deb http://ftp.pl.debian.org/debian trixie-backports main contrib non-free non-free-firmware			      #
	# deb http://ftp.pl.debian.org/debian trixie-backports main contrib non-free non-free-firmware			      #
	#														      #
	## trixie-security												      #
	deb https://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware		      #
	# deb https://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware 	      #
	#														      #
	## trixie-updates												      #
	deb http://ftp.pl.debian.org/debian trixie-updates main contrib non-free non-free-firmware			      #
	# deb-src http://ftp.pl.debian.org/debian trixie-updates main contrib non-free non-free-firmware		      #
	#														      #
	## trixie-proposed												      #
	# deb http://ftp.pl.debian.org/debian trixie-proposed main contrib non-free non-free-firmware			      #
	# deb-src http://ftp.pl.debian.org/debian trixie-proposed main contrib non-free non-free-firmware		      #
	##														      #
	#######################################################################################################################
	```

___

7. #### Chroot

	```bash
	# mounting additional filesystems
	mount --bind /dev /target/dev
	mount --bind /dev/pts /target/dev/pts
	mount --bind /proc /target/proc
	mount --bind /sys /target/sys
	mount --bind /run /target/run
	mount --bind /sys/firmware/efi/efivars /target/sys/firmware/efi/efivars
	# chroot
	chroot /target /bin/bash -l
	# commands in chroot
	export PS1="|chroot| ${PS1}"
	alias vim="vim.tiny"
	```

___

8. #### Update new system

	```bash
	# update package database
	apt update
	#
	source /etc/profile
	export PS1="|chroot| ${PS1}"
	alias vim="vim.tiny"
	
	```

___

9. #### Configure locale,keyboard,timezone,language

	```bash
	# Install packages
	apt install tzdata locales keyboard-configuration console-setup
	# configure locales
	dpkg-reconfigure locales
	# configure keyboard
	dpkg-reconfigure keyboard-configuration
	# configure console-setup
	dpkg-reconfigure console-setup
	# configure timezone
	dpkg-reconfigure tzdata
	#
	source /etc/profile
	export PS1="|chroot| ${PS1}"
	alias vim="vim.tiny"
	#
	```

____

10. #### Kernel and required tools

	```bash
	# install kernel and firmware
	apt install linux-image-amd64 linux-headers-amd64 dkms firmware-linux firmware-linux-free firmware-linux-nonfree
	# install other packages needed for base system
	apt install initramfs-tools cryptsetup cryptsetup-initramfs lvm2 man-db manpages manpages-pl filevi
	
	```

____

11. #### Set hostname and configure hosts file

	```bash
	vim.tiny /etc/hostname
	vim.tiny /etc/hosts
	```

___

12. #### Configure user

	```bash
	# create new user 
	useradd -m -G users,sudo -s /bin/bash -c "Charlie" charlie
	# set password for new user
	passwd charlie
	# lock root account
	passwd -l root
	#
	```

____

13. #### Configure fstab

	```bash
	# comand for getting UUID of partition
	blkid -s UUID -o value $DEV
	#
	# File content:
	#################################################################################################################################
	##                                                                                                                              #
	#<file system>                                  <mount point>   <type>  <options>                       <dump>  <pass>          #
	## root LABEL=RootFs                                                                                                            #
	UUID=0875236a-0afe-4b46-8df3-d9eceae911b2       /               ext4    errors=remount-ro               0       1               #
	#                                                                                                                               #
	## boot LABEL=BootFs                                                                                                            #
	UUID=beb2e793-2e70-4fac-a898-fdbc0171913d       /boot           ext2    defaults                        0       2               #
	#                                                                                                                               #
	## efi LABEL=EFI                                                                                                                #
	UUID=7DA2-F47E                                  /boot/efi       vfat    umask=0077                      0       1               #
	#                                                                                                                               #
	## home LABEL=HomeFs                                                                                                            #
	UUID=797f3a26-9f24-4ae1-b7f0-0f4881c17b4e       /home           ext4    defaults                        0       2               #
	#                                                                                                                               #
	## swap LABEL=SWAP                                                                                                              #
	UUID=c026a602-bbf4-4fa1-9d15-4b353330c969       none            swap    sw                              0       0               #
	#                                                                                                                               #
	##                                                                                                                              #
	#################################################################################################################################
	```

____

14. #### configure cryptab

	```bash
	# get UUID for lukscrypt partition
	blkid -s UUID -o value /dev/sda3
	#
	# Example crypttab file
	# <target name> <source device>                                 <key file>      <options>
	xyz-unlocked   UUID=7f1bcc62-a1c2-4567-93d5-d8183e2da7c1      none            luks,discard
	#
	# prepare resume file for initramfs
	blkid -s UUID -o value /dev/mapper/vg-swap >> /etc/initramfs-tools/conf.d/resume
	vim /etc/initramfs-tools/conf.d/resume
	# Content
	RESUME=UUID=c026a602-bbf4-4fa1-9d15-4b353330c969
	#
	```

____

15. #### Configure network

	```bash
	# install network-manager
	apt install network-manager
	# check network interfaces
	ip addr
	# add following configuration to /etc/network/interfaces file
	vim /etc/network/interfaces
	#
	# Content to add:
	## loopback
	#
	auto lo
	iface lo inet loopback
	#
	## ens33
	auto ens33
	allow-hotplug ens33
	iface ens33 inet dhcp
	#
	```

____

16. #### Install grub bootloader

	```bash
	# install packages
	apt install grub-efi-amd64 efibootmgr os-prober
	# install grub bootloader
	grub-install --target=x86_64-efi --efi-directory=/boot/efi
	# add resume=UUID=xxxx... to /etc/default/grub 
	# example:
	GRUB_CMDLINE_LINUX_DEFAULT="resume=UUID=c026a602-bbf4-4fa1-9d15-4b353330c969 quiet"
	# update grub config and recreate initramfs
	update-initramfs -ck all;update-grub2
	#
	```

____

17. #### finish thing

	```bash
	# exit from chroot
	exit
	# deactivate swap
	swapoff /dev/dm-2
	# deactivate swap partition
	swapoff /dev/dm-2
	# umount all filesystems mounted at /target
	#
	umount /target/sys/firmware/efi/efivars
	umount /target/run
	umount /target/sys
	umount /target/proc
	umount /target/dev/pts
	umount /target/dev
	umount /target/home
	umount /target/boot/efi
	umount /target/boot
	umount /target
	vgchange -a n vg
	cryptsetup close xyz-unlocked
	reboot
	#
	```

____

18. #### Install basic packages after restart to newly installed system.

	```bash
	 apt install neovim htop tmux iftop iotop build-essential gdb git cmake wget wget2 curl rsync tcpdump net-tools xclip xsel cryfs age mc fzf aptitude gdisk dosfstools mtools ntfs-3g btrfs-progs xfsprogs jfsutils exfatprogs squashfs-tools command-not-found 7zip rar unrar zip unzip gzip bzip2 xz-utils zstd pigz btrfs-progs exfatprogs jfsutils mdadm xfsprogs aptitude diffutils lsof bc pwgen xkcdpass xxd pv jq yq xq trash-cli speedtest-cli links tasksel
	 # install standrad task with tasksel
	 tasksel install standard
	```

_____

19. #### Configure silent boot
	```bash
	# configure to /etc/default/grub
	GRUB_CMDLINE_LINUX_DEFAULT="quiet splash loglevel=3 systemd.show_status=auto rd.udev.log_level=3 vt.global_cursor_default=0"
	```

____

20. #### Install some GUI

	```bash
	# balanced kde plasma desktop
	apt install kde-standard
	# set default target to newly installed GUI enviroment
	systemctl set-default graphical.target
	#
	```

____
- ##  Reference

- a) [https://morfikov.github.io/post/instalacja-debiana-z-wykorzystaniem-debootstrap/](https://morfikov.github.io/post/instalacja-debiana-z-wykorzystaniem-debootstrap/)
- b) [https://gist.github.com/varqox/42e213b6b2dde2b636ef](https://gist.github.com/varqox/42e213b6b2dde2b636ef)
- c) [https://gist.github.com/starquake/856b05dc88d68e7509e23f8995f7ac5e](https://gist.github.com/starquake/856b05dc88d68e7509e23f8995f7ac5e)

________
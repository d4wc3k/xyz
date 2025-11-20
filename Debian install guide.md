
### 1. Reference

- a) [https://morfikov.github.io/post/instalacja-debiana-z-wykorzystaniem-debootstrap/](https://morfikov.github.io/post/instalacja-debiana-z-wykorzystaniem-debootstrap/)
- b) [https://gist.github.com/varqox/42e213b6b2dde2b636ef](https://gist.github.com/varqox/42e213b6b2dde2b636ef)

### 2. Steps

___

1. #### Install required packages


	Update packages database:

	```bash
	apt update
	```

	Install packages:

	```bash
	apt install neovim gdisk debootstrap arch-install-scripts
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
  	cryptsetup --debug --type luks2 -v --verify-passphrase --cipher aes-xts-plain64 \
	--key-size 512 --key-slot 0  --key-description "Basic key" --hash sha256 \
	--iter-time 11654 --pbkdf argon2id  --pbkdf-memory 524288 --pbkdf-parallel 4 \
	--use-random --label xyz-locked --timeout 60 luksFormat /dev/sda3
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
	debootstrap --verbose --arch=amd64 trixie /target http://ftp.pl.debian.org/debian/
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
	mount --make-rslave --rbind /proc /target/proc
	mount --make-rslave --rbind /sys /target/sys
	mount --make-rslave --rbind /dev /target/dev
	mount --make-rslave --rbind /run /target/run
	mount --make-rslave --rbind /sys/firmware/efi/efivars /target/sys/firmware/efi/efivars
	# chroot
	chroot /target /bin/bash -l
	# commands in chroot
	export PS1="|chroot| ${PS1}"
	```

___

8. #### Update new system

	```bash
	# update package database
	apt update
	# install ca-certificate and bash-completion
	apt install ca-certificates bash-completion
	# 
	source /etc/profile
	export PS1="|chroot| ${PS1}"
	
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
	#
	```

____

10. #### Kernel and required tools

	```bash
	apt install linux-image-amd64 linux-headers-amd64 dkms firmware-linux firmware-linux-nonfree initramfs-tools cryptsetup cryptsetup-initramfs lvm2 man-db manpages manpages-pl
	
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
	
	#
	```

____

13. #### Configure fstab

	```bash
	# comand for getting UUID of partition
	blkid -s UUID -o value $DEV
	#
	#################################################################################################################################
	##																#
	#<file system>					<mount point>	<type>	<options>			<dump>	<pass>		#
	## root																#
	UUID=effba5dc-1da7-4e19-8b36-ffbc033c6ed4	/         	ext4	errors=remount-ro		0	1		#
	#																#
	## boot																#
	UUID=7794c5c6-570e-4ea9-b2fd-f218cd7fe678	/boot		ext2	defaults			0	2		#
	#																#
	## efi																#
	UUID=25A5-03B2					/boot/efi 	vfat	umask=0077			0	1		#
	#																#
	## home																#
	UUID=0bfc47ff-8ab8-4286-9551-80e1b7d7eaa7	/home     	ext4	defaults			0	2		#
	#																#
	## swap																#
	UUID=56ece0ce-fe20-4628-a736-5e55ba784f30	none		swap	sw				0	0		#
	#																#
	##																#
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
	```

____

15. #### Configure network

	```bash
	# install network-manager
	apt install network-manager
	# check network interfaces
	ip addr
	# add following configuration to /etc/network/interfaces file
	vim
	```

____

16. #### Install grub bootloader

	```bash
	# install packages
	apt install grub-efi-amd64 efibootmgr os-prober
	# install grub bootloader
	grub-install --target=x86_64-efi --efi-directory=/boot/efi
	# update grub config and recreate initramfs
	
	#
	```

____

17. #### finish thing

	```bash
	# exit from chroot
	exit
	# umount all filesystems mounted at /target
	umount -R /target
	# deactivate swap partition
	swapoff /dev/dm-2
	#
	reboot
	#
	```

____

18. #### Install basic packages after restart to newly installed system.

	```bash
	 apt install neovim htop tmux iftop iotop build-essential gdb git cmake wget wget2 curl rsync tcpdump net-tools xclip xsel cryfs age mc fzf aptitude gdisk dosfstools mtools ntfs-3g btrfs-progs xfsprogs jfsutils exfatprogs squashfs-tools
	```
____
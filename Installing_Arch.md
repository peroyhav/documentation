# Procedure for installing ARCH Linux

1.  ## Pre installation
    1.  ### Download and verify the ISO
        First, download the ARCH Linux ISO from [the official website](https://archlinux.org/download/), it's a good practice to verify the signature as well, in order to ensure you have an authentic file.
    1.  ### Create a bootable USB
        You can create a bootable USB by using Ventoy, or by using the dd command
        > Example using dd to /dev/sda
        ```bash
        $ sudo dd if=/path/to/arch.iso of=/dev/sda bs=1M status=progress oflag=direct
        ```
    
    1.  ### Boot into the Live environment
        Reboot your computer and boot from the USB device, You may need to press a key like *Del*, *F12* *F8* or *F2* during startup to select the USB for boot
    1.  ## Connect to the internet
        If you are not using a wired connection, you probably need to connect to a Wifi now
        ```bash
        # Network commands
        $ iwctl device list
        # powe on the adapter if powered off
        $ iwctl adapter phy0 set-property Powered on
        $ iwctl station wlan0 show
        $ iwctl station wlan0 get-networks
        $ iwctl station wlan0 connect {SSID}
        ```
        Check that your connection is working
        ```bash
        $ ping archlinux.org -c 3
        ```
    1.  ### Update the system clock
        ```bash
        $ timedatectl set-ntp true
        ```
1.  ## Disk preparation
    1.  ### Erase the disk securelly and format to the desired partition layout
        If your disk contains sensitive data, you can securely erase it. This step is optional for new drives.
        * For NVME SSDs
          ```bash
          $ nvme format /dev/nvme0n1 --ses=2 --reset
          ```
        * For SATA SSDs
          Check if the drive is frozen, if it is, you have to suspend and resume your system to unfreeze it.
          ```bash
          $ hdparm -I /dev/sda | grep "frozen"
          ```
          Set a temporary password and erase the disk
          ```bash
          $ hdparm --user-master u --security-set-pass password /dev/sda
          $ hdparm --user-master u --security-erase-enhanced password /dev/sda
          ```
        * For traditional Harddrives using the shred command to overwrite with random data
          ```bash
          shred -n 1 -vz /dev/sda
          ```
    
    1.  ### Partition the disk
        We will create 2 partitions in this guide, where the first is teh EFI partition, and the second is the encrypted btrfs partition where the remainding files are stored
        > Example partition layout
        > Id  Size              Type              Filesystem
        > 1   1GiB              EFI(ef00)         FAT 32
        > 2   Remainding space  Linux LUKS(8309)  LUKS
        ```bash
        $ gdisk /dev/nvme0n1
        ```
        > Create a new GPT partition header with `o` option
        > Create a new 1GB *EFI* partion with the `n` option and set code to `ef00`
        > Create a new Linux LUKS partition for the remainding disk space with the `n` option and set code to `8309`
        > Save the partition layout with the `w` option, this will exit gdisk simultaneously.
    
    1.  ### Format the EFI partition
        > NOTE: The FAT partition is labeled EFI, but we will use the UUID for mount points.
        ```bash
        $ mkfs.fat -F 32 -n EFI /dev/nvme0n1p1
        ```
    
    1.  ### Randomize the partition that will be encrypted
        > NOTE: This step is optional, and will add wear to the disk, but will add security by hiding data boundaries for the next step
        ```bash
        $ cryptsetup luksFormat /dev/nvme0n1p2 --header /tmp/cryptdisk.img
        $ cryptsetup open /dev/nvme0n1p2 --header /tmp/cryptdisk.img cryptdisk
        $ dd if=/dev/zero of=/dev/mapper/cryptdisk bs=1M status=progress
        $ cryptsetup close cryptdisk
        ```
    
    1.  ### Create and mount the encrypted partition
        ```bash
        $ cryptsetup lumsFormat /dev/nvme0n1p2 --pbkdf pbkdf2 --hash sha256
        $ cryptsetuo open /dev/nvme0n1p2 arch
        ```
    1.  ### Create the btrfs file system and set up subvolumes
        ```bash
        $ mkfs.btrfs -L ARCH /dev/mapper/arch
        $ mount /dev/mapper/arch /mnt
        $ btrfs subvolume create /mnt/@            # Linux ROOT
        $ btrfs subvolume create /mnt/@boot        # Optional /boot subvolume to keep boot separate from /
        $ btrfs subvolume create /mnt/@root        # Optional subvolume for home for the /root user
        $ btrfs subvolume create /mnt/@home        # Optional, but recommended subvolume for /home
        $ btrfs subvolume create /mnt/@.snapshots  # Optional subvolume for keeping snapshots
        $ btrfs subvolume create /mnt/@log         # Optional subvolume for logs, recommended if snapshots is enabled, to avoid copies of old logs
        $ btrfs subvolume create /mnt/@pkg         # Optional subvolume for pacman cache information, recommended to keep outside snapshots
        $ umount /mnt
        ```
    1.  ### Mount the root file structure under /mnt
        > NOTE: some mount options like noatime and ssd should be omitted if you use a normal drive, compression can 
        > in addition be tuned if you want a different compression level than the default of 3
        ```bash
        $ mount --mkdir -o ssd,noatime,compress=zstd,subvol=@ /dev/mapper/root /mnt
        $ mount --mkdir -o ssd,noatime,compress=zstd,subvol=@boot /dev/mapper/root /mnt/boot
        $ mount --mkdir -o ssd,noatime,compress=zstd,subvol=@root /dev/mapper/root /mnt/root
        $ mount --mkdir -o ssd,noatime,compress=zstd,subvol=@home /dev/mapper/root /mnt/home
        $ mount --mkdir -o ssd,noatime,compress=zstd,subvol=@.snapshots /dev/mapper/root /mnt/.snapshots
        $ mount --mkdir -o ssd,noatime,compress=zstd,subvol=@log /dev/mapper/root /mnt/var/log
        $ mount --mkdir -o ssd,noatime,compress=zstd,subvol=@pkg /dev/mapper/root /mnt/var/cache/pacman/pkg
        $ mount --mkdir /dev/nvme0n1p1 /mnt/efi
        ```
1.  ## System Installation
    1.  ## Select the best mirrors based on your location
        > NOTE: I live in Norway, so I chose Norway and Sweden
        ```bash
         $ reflector --protocol https --country Norway,Sweden --latest 5 --save /etc/pacman.d/mirrorlist
        ```
    1.  ## Install base system
        > NOTE: If you wish to use labels instead of UUIDS in /etc/fstab, thats completely possible by replacing the -U flag with -L for genfstab
        ```bash
        # Install either intel-ucode or amd-ucode depending on CPU in the device
        $ pacstrap -K /mnt base linux linux-headers linux-firmware intel-ucode neovim bash-completion
        $ genfstab -U /mnt >> /mnt/etc/fstab
        ```
    1.  ## Chroot into the environment
        ```bash
        $ arch-chroot /mnt
        ```
    1.  ## Set up timezone and hostname
        ```bash
        $ ln -sf /usr/share/zoneinfo/Europe/Oslo /etc/localtime
        $ hwclock --systohc
        $ echo {hostname} > /etc/hostname
        ```
    1.  ## Install necessary software to be able to set up boot configuration
        ```bash
        $ pacman -S btrfs-progs dosfstools grub grub-btrfs efibootmgr usbutils
        ```
    1.  ## Create keyfile to ensure we don't need to enter password twice
        ```bash
        $ dd if=/dev/random iflag=fullblock bs=4k count=1 | install -m 0600 /dev/sdtin /etc/cryptsetup-keys.d/arch.key
        $ cryptsetup luksAddKey /dev/nvme0n1p2 /etc/cryptsetup-keys.d/arch.key
        ```
    1.  ## Set up mkinitcpip to configure initramfs
        ```bash
        $ nvim /etc/mkinitcpio.conf
        ```
        Here is my mkinitcpio.conf file
        ``` /etc/mkinitcpio.conf
        MODULES=(btrfs hid_apple usbhid xhci_hcd)
        BINARIES=(/usr/bin/btrfs)
        FILES=(/etc/cryptsetup-keys.d/rootfs.key)
        HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems fsck)
        ```
        > NOTE: The modules, with exception of btrfs are set up because I have a Keycron USB keyboard
        >       usbhid to support USB keyboard
        >       hid_apple because the keyboard is reporting as an apple keyboard to the OS
        >       xhci_hcd to support USB 3.0 and newer
    
        Run mkinitcpio to generate a correct initramfs file.
        ```bash
        $ mkinitcpio -P
        ```
    1.  ## Set up GRUB bootloader
        Because GRUB is one of the few bootloaders that support a fully encrypted /boot, we need to set up grub EFI boot
        ```
        $ nvim /etc/default/grub
        ```
        ### Go to the line GRUB_CMDLINE_LINUX(line 6) and edit to use an encrypted disk
        ```
        GRUB_CMDLINE_LINUX="loglevel=3 quiet cryptdevice=UUID=$((blkid -s UUID -o value /dev/nvme0n1p2)):arch cryptkey=rootfs:/etc/cryptsetup-keys.d/arch.key"
        ```
        ### Go to the line `# GRUB_ENABLE_CRYPTDISL=y`(line 13) and uncomment
    
        ### Set up EFI boot
        ```bash
        $ grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
        ```
        ### Create GRUB config
        ```
        $ grub-mkconfig -o /boot/grub/grub.cfg
        ```
    1.  ## Install userland tools
        ```
        $ pacman -S base-devel git bash-completion sudo tmux neofetch ripgrep networkmanager wireless_tools ufw usbutils wget
        ```
    1.  ## Set up so members of group wheel have sudo rights
        ```bash
        $ EDITOR=nvim visudo
        ```
        > Search for the line `@ %wheel ALL=(ALL:ALL) ALL` and remove `# ` to uncomment.
    1.  ## Set up user with sudo privileges
        ```bash
        $ useradd -m -G wheel -U {username}
        $ passwd {username}
        ```
    1.  ## Fetch this repo and install the rest of the tools reqired
        ```bash
        $ su {username}
        $ mkdir .git
        $ cd .git
        $ git clone https://github.com/peroyhav/documentation.git
        $ cd documentation
        $ sudo pacman -S --needed - arch.packages
        $ sudo systemctl enable gdm
        $ sudo systemctl enable gdm
        ```
    1. ## Exit fakeroot, unmount disk and reboot
        ```bash
        $ logout
        $ exit
        $ umount -R /mnt
        $ reboot
        ```
1. ## Congratulations, you should now have a graphical UI booted with gnome, and are able to switch to hyprland after configuring terminal tools and such.

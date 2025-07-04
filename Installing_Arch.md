# Procedure for installing ARCH Linux

1.  Download and verify ARCH Linux ISO
2.  Move ISO to a disk e.g. Ventoy
3.  Boot into the ISO
4.  Connect to network and prepare for setup
    ```bash
    $ ip link
    $ iwctl device list
    $ iwctl adapter wlan0 set-property Powered on
    $ iwctl station wlan0 show
    $ iwctl station wlan0 get-networks
    $ iwctl station wlan0 connect {SSID}
    $ ping archlinux.org
    $ timedatectl
    ```
5.  Erase the disk securelly and format to the desired partition layout
    ```bash
    # This step may be skipped if the disk previously did not contain any sensitive data,
    # and you don't need or require to have a strong encryption.
    $ nvme format /dev/nvme0n1 --ses=2 --reset
    # Partition and format the drive
    # For EFI boot, the following partition scheme could be used:
    # Id  Size              Type              Filesystem
    # 1   1GiB              EFI(ef00)         FAT 32
    # 2   Remainding space  Linux LUKS(8309)  LUKS
    $ gdisk /dev/ng0nX
    Command (? for help): n
    Partition number (1-128, default 1):
    First sector (34-1998307982, default = 2048) or {+-}size{KMGTP}:
    Last sector (4096-1998307982, default = 1998307327) or {+-}size{KMGTP}: +1G
    Current type is 8300 (Linux filesystem)
    Hex code or GUID (L to show codes, Enter = 8300): ef00
    Changed type of partition to 'EFI system partition'

    Command (? for help): n
    Partition number (2-128, default 2):
    First sector (34-1998307982, default = 2099200) or {+-}size{KMGTP}:
    Last sector (2099200-1998307982, default = 1998307327) or {+-}size{KMGTP}:
    Current type is 8300 (Linux filesystem)
    Hex code or GUID (L to show codes, Enter = 8300): 8309
    Changed type of partition to 'Linux LUKS'

    Command (? for help): w

    Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
    PARTITIONS!!

    Do you want to proceed? (Y/N): y
    ...
    # Write random data to the LUKS partition to hide data boundaries
    # An alternative way, would be to use cryptsetup, open the encrypted disk
    # write alll zeroes to the encrypted disk,
    # then reformat using cryptsetup and a secure key.
    $ dd if=/dev/urandom of=/dev/nvme0n1p1 bs=1G oflag=direct status=progress
    # Format the partitions
    $ mkfs.fat -F 32 -n EFI /dev/nvme0n1p1
    $ cryptsetup luksFormat --pbkdf pbkdf2 /dev/nvme0n1p2
    WARNING!
    ========
    This will overwrite data on /dev/nvme0n1p2 irrevocably.

    Are you sure? (Type 'yes' in capital letters): YES
    Enter passphrase for /dev/nvme0n1p1:
    Verify passphrase:
    $ cryptsetup open /dev/nvme0n1p2 root
    Enter passphrase for /dev/nvme0n1p2:
    $ mkfs.btrfs -L ROOT /dev/mapper/root
    $ mount -o compress=zstd /dev/mapper/root /mnt
    $ cd /mnt
    $ btrfs subvolume create @
    $ btrfs subvolume create @boot
    $ btrfs subvolume create @root
    $ btrfs subvolume create @home
    $ btrfs subvolume create @.snapshots
    $ btrfs subvolume create @log
    $ btrfs subvolume create @pkg
    $ cd
    $ umount -R /mnt
    $ mount --mkdir -o compress=zstd,subvol=@ /dev/mapper/root /mnt
    $ mount --mkdir -o compress=zstd,subvol=@boot /dev/mapper/root /mnt/boot
    $ mount --mkdir -o compress=zstd,subvol=@root /dev/mapper/root /mnt/root
    $ mount --mkdir -o compress=zstd,subvol=@home /dev/mapper/root /mnt/home
    $ mount --mkdir -o compress=zstd,subvol=@.snapshots /dev/mapper/root /mnt/.snapshots
    $ mount --mkdir -o compress=zstd,subvol=@log /dev/mapper/root /mnt/var/log
    $ mount --mkdir -o compress=zstd,subvol=@pkg /dev/mapper/root /mnt/var/cache/pacman/pkg
    $ mount --mkdir /dev/nvme0n1p1 /mnt/efi
    ```
6.  Select mirrors
    ```bash
     $ reflector --protocol https --country Norway,Sweden --latest 5 --save /etc/pacman.d/mirrorlist
    ```
7.  Install base packages and add fstab
    ``` bash
    # Install either intel-ucode or amd-ucode depending on CPU in the device
    $ packstrap -K /mnt base linux linux-lts linux-firmware intel-ucode
    $ genfstab -U /mnt >> /mnt/etc/fstab
    ```
8.  Chroot into the environment and set up the environment
    ``` bash
    $ arch-chroot /mnt
    $ pacman -Sy
    $ pacman -S btrfs-progs dosfstools base-devel git bash-completion sudo tmux neovim neofetch ripgrep grub-btrfs
    # Create the user you will be using
    $ useradd -m -U {username}
    # Enable sudo group to give sudo privileges 
    $ EDITOR=nvim visudo
    $ groupadd sudo
    $ usermod -aG sudo {username}
    # Set password for the user
    $ passwd {username}
    New password:
    Retype new password:
    # Set local timezone to Europe/Oslo
    $ ln -sf /usr/share/zoneinfo/Europe/Oslo /etc/localtime
    $ hwclock --systohc
    $ echo {hostname} >> /etc/hostname
    # Set boot config and install grub2, grub2-btrfs, and efibootmgr and set up boot environment
    # Install grub and tools for connecting to the internet
    $ pacman -S grub grub-btrfs efibootmgr networkmanager wireless_tools ufw usbutils wget
    # Generate a boot key and add to LUKS volume
    $ dd bs=512 count=4 if=/dev/random iflag=fullblock | install -m 0600 /dev/stdin /etc/cryptsetup-keys.d/cryptbtrfs.key
    $ cryptsetup luksAddKey /dev/nvme0n1p2 /etc/cryptsetup-keys.d/cryptbtrfs.key
    # prepare setup for decrypting the disk
    $ nvim /etc/mkinitcpio.conf
    # update line with HOOKS to the following
    ...
    HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems fsck)
    ...
    FILES=(/etc/cryptsetup-keys.d/cryptbtrfs.key)
    $ mkinitcpio -P
    $ nvim /etc/default/grub
    # Edit the line for GRUB_CMDLINE_LINUX
    # to get the UUID of the partitions, you can run the blkid command on a new blank line:
    :!blkid /dev/nvme0n1p1
    GRUB_CMDLINE_LINUX="... cryptdevice=UUID=device-UUID:root rd.luks.name=device-UUID=root /etc/cryptsetup-keys.d/cryptbtrfs.key"
    Uncomment the line # GRUB_ENABLE_CRYPTODISK=y
    GRUB_ENABLE_CRYPTODISK=y
    $ grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
    $ grub-mkconfig -o /boot/grub/grub.cfg
    $ logout
    $ umount -R /mnt
    $ reboot
    # eject the install media while iso environment is shutting down
    ```
9.  Reboot the computer and see that you can log in with the user you created
    ```bash
    login: {username}
    password: {password}
    # connect to the internet for the "first time"
    $ sudo syustemctl enable --now NetworkManager
    $ sudo nmcli device wifi connect {NetworkName} --ask
    # install desired window manager, e.g. gnome and xfce4-terminal and other tools that is still missing
    $ sudo pacman -S gdm alacritty alsa-tools hypridle hyprland hyprlock hyprpaper hyprpolkitagent swaylock wayland dolphin bluez-utils blueman brightnessctl firefox nvidia-dkms pavucontrol ttf-nerd-fonts-symbols ttf-nerd-fonts-symbols-mono wireless_tools wofi network-manager-applet ttf-font-awesome loupe wl-clipboard swaync vlc totem
    $ sudo systemctl enable --now gdm
    ```
10. Congratulations, you should now have a graphical UI booted with gnome, and are able to switch to hyprland after configuring terminal tools and such.

# Install on QEMU

## 🛠 Requirements
Make sure you have the following packages installed:
* qemu
* qemu-system-x86_64
* ovmf (for UEFI support — this provides OVMF_CODE.fd and OVMF_VARS.fd)
* virt-manager (optional GUI)
* qemu-utils (for qemu-img)

### Install on Arch-based system:
```bash
sudo pacman -S qemu ovmf virt-manager
```

## ✅ QEMU Script for Installing Arch Linux with UEFI, NVMe, and VirtIO
```bash start-arch-install.sh
#!/bin/bash

# Path to your downloaded Arch Linux ISO
ARCH_ISO="$HOME/Downloads/archlinux-x86_64.iso"

# Path to your virtual disk image
DISK_IMG="$HOME/VMs/archlinux/archlinux.qcow2"

# Create a 32GB virtual disk image if it doesn't exist
if [ ! -f "$DISK_IMG" ]; then
  mkdir -p "$(dirname "$DISK_IMG")"
  qemu-img create -f qcow2 "$DISK_IMG" 32G
fi

# Start QEMU with UEFI firmware and NVMe disk
qemu-system-x86_64 \
  -enable-kvm \
  -m 4096 \
  -smp 4 \
  -cpu host \
  -machine type=q35,accel=kvm \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE.fd \
  -drive if=pflash,format=raw,file=/usr/share/OVMF/OVMF_VARS.fd \
  -boot d \
  -cdrom "$ARCH_ISO" \
  -drive file="$DISK_IMG",if=none,id=disk0,format=qcow2 \
  -device nvme,drive=disk0,serial=nvme0 \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net,netdev=net0 \
  -display default,show-cursor=on \
  -usb -device usb-tablet \
  -soundhw hda \
  -vga virtio \
  -name "Arch Linux Install" \
  -monitor stdio
```

### 🧠 Notes
This VM boots using UEFI (OVMF) — matching your install instructions exactly.

Uses an NVMe device so your partition names will be /dev/nvme0n1, just like in your guide.

VirtIO network and graphics — good performance and compatibility.

Port forwarding is enabled (host port 2222 → guest port 22) in case you want to SSH in later.

## 📹 Recording Tip with OBS
When you run the script:

OBS will see the QEMU display window as a regular application window.

Just use “Window Capture” or “Screen Capture” mode to record the install session.

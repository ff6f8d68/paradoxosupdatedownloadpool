Overview
Write a Bootloader: A simple bootloader in Assembly that loads the kernel and transfers control to it.
Prepare a Minimal Linux Kernel: Download and compile a basic Linux kernel.
Create an Initrd Image: Include a shell script and necessary binaries in the initrd.
Combine Everything: Use GRUB to create a bootable ISO.
Step-by-Step Guide
1. Write the Bootloader

bootloader.asm:
´´´
BITS 16
ORG 0x7C00

start:
    ; Set up the stack
    xor ax, ax
    mov ss, ax
    mov sp, 0x7C00

    ; Print loading message
    mov si, msg
    call print_string

    ; Load kernel (this is simplified; real bootloaders would do this differently)
    ; For demonstration, we assume the kernel is located at 0x1000:0
    jmp 0x1000:0

print_string:
    lodsb
    or al, al
    jz .done
    mov ah, 0x0E
    int 0x10
    jmp print_string
.done:
    ret

msg db 'Loading kernel...', 0

TIMES 510-($-$$) db 0
DW 0xAA55
´´´
Compile the Bootloader:

´´´
nasm -f bin bootloader.asm -o bootloader.bin
´´´
2. Prepare a Minimal Linux Kernel

Download and Compile the Linux Kernel:
You can use a precompiled kernel for simplicity. Download a Linux kernel, e.g., from Alpine Linux:

´´´
curl -O https://dl-cdn.alpinelinux.org/alpine/latest-stable/main/x86_64/linux-virt-5.10.104-r0.apk
´´´
Extract the Kernel:

´´´
mkdir -p kernel_pkg
cd kernel_pkg
ar x ../linux-virt-5.10.104-r0.apk
tar -xzf data.tar.gz
´´´
Copy the Kernel Image:
´´´
cp boot/vmlinuz-virt ../kernel/
cd ..
´´´
3. Create Initrd with a Shell Script

Create Directory Structure for Initrd:

´´´
mkdir -p initrd/{bin,dev,etc,proc,sys,tmp}
´´´
Copy Required Binaries:
For simplicity, copy a minimal set of binaries to initrd/bin. Here, we assume you have a compatible busybox binary that includes sh.

Write the Shell Script:
script.sh:
´´´
#!/bin/bash

sudo apt-get update
sudo apt-get install -y xvfb matchbox-window-manager chromium-browser whiptail

# URL of the website to download
URL="https://paradoxosupdatedownloadpool.netlify.app"

# Local directory to store the downloaded website
LOCAL_DIR="/html"

# Path to the index.html file
INDEX_FILE="$LOCAL_DIR/index.html"

# Create the local directory if it does not exist
mkdir -p "$LOCAL_DIR"

# Function to download the website
download_website() {
    # Show Whiptail progress dialog
    (
        echo "10" ; sleep 1
        wget --mirror --no-parent --quiet --show-progress --progress=bar:force:noscroll --directory-prefix="$LOCAL_DIR" "$URL" 2>&1 | \
        while read line; do
            echo "XXX $line"
        done
    ) | whiptail --gauge "Downloading update..." 10 60 0

    # Check if wget completed successfully
    if [ $? -eq 0 ]; then
        whiptail --title "Update Complete" --msgbox "Website update completed successfully." 10 60
    else
        whiptail --title "Update Failed" --msgbox "Website update failed." 10 60
        exit 1
    fi
}

# Download the website if necessary
if [ ! -d "$LOCAL_DIR" ] || [ "$(find "$LOCAL_DIR" -type f -exec md5sum {} + | md5sum)" != "$(curl -s "$URL/md5sum" | md5sum)" ]; then
    download_website
fi

# Check if the index.html file exists
if [ ! -f "$INDEX_FILE" ]; then
    whiptail --title "File Not Found" --msgbox "The file $INDEX_FILE does not exist." 10 60
    exit 1
fi

# Start Xvfb (virtual framebuffer)
Xvfb :99 -screen 0 1024x768x16 &

# Set DISPLAY environment variable to use the virtual framebuffer
export DISPLAY=:99

# Start the matchbox window manager without window decorations
matchbox-window-manager -use_cursor no &

# Start Chromium in kiosk mode to display the HTML content
chromium-browser --no-sandbox --disable-infobars --kiosk "file://$INDEX_FILE" &

# Wait for Chromium to close
wait $!

´´´
Make the Script Executable:

´´´
chmod +x initrd/tmp/script.sh
Create Init Script:
initrd/init:
´´´
´´´
#!/bin/sh
/tmp/script.sh
´´´
Make the Init Script Executable:

´´´
chmod +x initrd/init
´´´
Create the Initrd Image:

´´´
cd initrd
find . | cpio -o -H newc | gzip > ../initrd.img
cd ..
´´´
4. Set Up GRUB Configuration

Create a GRUB Configuration File:
grub.cfg:

´´´
menuentry 'Custom Linux' {
    set root='(hd0,1)'
    linux /boot/vmlinuz root=/dev/ram0
    initrd /boot/initrd.img
}
´´´
Prepare the ISO Directory Structure:

´´´
mkdir -p iso/boot/grub
cp bootloader.bin iso/boot/
cp kernel/vmlinuz iso/boot/
cp initrd.img iso/boot/
cp grub.cfg iso/boot/grub/
´´´
Create the Bootable ISO:

´´´
grub-mkrescue -o custom_linux.iso iso/
´´´
5. Write the ISO to a USB Drive (Linux Environment)

If you are on Linux, you can use dd to write the ISO to a USB drive:

´´´
sudo dd if=custom_linux.iso of=/dev/sdX bs=4M
´´´
Replace /dev/sdX with the appropriate USB device identifier.

Summary
Bootloader: The Assembly code initializes the system and loads the kernel.
Kernel: Downloaded or compiled, and placed in the boot directory.
Initrd: Contains the shell script and necessary binaries.
GRUB Configuration: Directs GRUB to use the kernel and initrd.
ISO Creation: Combines everything into a bootable ISO using GRUB.
This setup assumes basic knowledge of creating bootable media and working with Linux tools. For real-world applications, additional steps might be required based on specific needs and hardware compatibility.

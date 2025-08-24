# Nvidia GeForce RTX 50 series driver failure and black screen of death problem
  
Currently there are driver incompatibility issues with Nvidia 50 series Blackwell architechture GPUs particularly for Linux OS (tsted Ubuntu, POP OS, Alma Linux, Rocky Linux and RHEL)

## Complicated issues:
  - Simply there are many- Nvidia transitioning to open source kernel while there are proprietary versions in the distro
  - Even there are many versions of open source drivers in the distro repos
  - Not all the open source drivers from repo works or works/fails equally well!
  - And even if you have found the working open source driver it doesn't perform well as the proprietary drivers

Previously, I was able to find a solution for mixed Nvidia GPUs with 30 series and 50 series dual GPUs using POP-OS (debian). Here is the link for that thread- https://unix.stackexchange.com/questions/798137/nvidia-rtx-5070-ti-causes-black-screen-and-very-slow-boot
But that lasted until the GPU was under stress with full load for long work! During hours of 100% load, the driver was unloading randomly and my work was stopped. nvidia-smi was showing driver loaded but they did allow running programs until I restarted the machine!

After experimenting with multiple distros, various driver versions, powering off GPUs one after the other, Bios settings checking one by one and PCIe disabling at various steps and permutation/combination, it seems, finally I understood the problem and got a work around solution!

## Problem: In a word "many"
  - You need a proper nvidia driver (version),
  - Nouveau driver loaded during UEFI boot/Kernel loading fails with black screen when GPUs connected
  - Some bios settings interfere with the GPU (PCIe v5 of RTX 5070ti/ 5080)
  - Latest OS kernel doesn't have X server / Xorg display (wayland doesn't work with GeFroce 50 series balckwell architechture GPUs)

## Solution: Multi-step and requires every step to complete without an error

1. Disconnect the power cable of Nvidia gpus (don't need to take it off from PCIe slot, I know this is a pain and risky to damaging the PCIe connectors even for pros)
   
2. Bios settings (depends of chipset maker) - Above 4 G decoding "ENABLED"; PCIe spread spectrum "ENABLED"; CSM "DISABLED" (was not needed for my MB though); you might also have to "DISABLED" secure boot (was not needed for my MB)
 
3. If you have a running Linux, restart and/or start with a freshly installed distro
   
4. Check if you have any nvidia driver/library/configuration, cuda driver in your system and uninstall and/or purge from nvidia driver or cuda packages and restart
   
$locate "nvidia" (shows where are the "nvidia" file/folders)
remove the nvidia drivers/ cuda packages

For debian: 
>sudo apt-get purge '*nvidia*'
>
>sudo apt remove --purge '^nvidia-.*'

For fedora/rocky Linux/RHEL:
>sudo dnf remove --purge '^nvidia-.*'
>
restart:
>sudo reboot
>
5. Update & restart

debian:
>sudo apt update
>
>sudo apt upgrade

fedora:
>sudo dnf update -y
>
>sudo dnf upgrade --refresh -y

>sudo reboot
>
6. Nouveau is the default GPU driver loaded with kernel through GRUB. Disable it and update grub
   
6.a. Check which system boot type - bios or UEFI your system is running with. Almost all modern linux systems use UEFI now-a-days
* Check your BIOS/UEFI setting or use CLI
>sudo df -h (if you have a mount point /boot/efi, then it is UEFI)
>
6.b. Cafefully modify default GRUB boot loader configuration with nano or vi. Be extra careful!
First, copy the target grub configuration as grub.backup so that you can recover boot loader in case it gets messed up
>sudo cp /etc/default/grub /etc/default/grub.backup
>
Edit the grub configuration. Careful that you don't edit any other grub file with suffix or extensions
>sudo nano /etc/default/grub
>
Locate the line with 'GRUB_CMDLINE_LINUX_DEFAULT ' or ' GRUB_CMDLINE_LINUX ' followed by "=" and go the end of that line and add 'nouveau.modeset=0' before the " mark. Don't touch anything else or give any extra space before or after the last ". As example- 
>GRUB_CMDLINE_LINUX="crashkernel=****** rhgb quiet nouveau.modeset=0"
>
Save it (nano> Ctl+O, enter, Ctl+X or vi> :x)

6.c. Update GRUB

For BIOS:
>sudo grub2-mkconfig -o /boot/grub2/grub.cfg

For UEFI: replace ???? with ubuntu or rocky or redhat or check whatever distro name folder you have
>sudo grub2-mkconfig -o /boot/efi/EFI/????/grub.cfg
>
Restart
>sudo reboot now
>

7. Identify and download the right (RIGHT and RIGHT version!) NVIDIA GPU driver
Go to https://www.nvidia.com/en-us/drivers/ and find the right GPU driver .run file in ~/Download

I suggest not to get the latest Nvidia driver unless you know it works.

I chose a driver that is compatible for both RTX 3060ti and RTX 5070ti and supports compatible cuda (12.8) for my work. Latest driver will support latest cuda, but many work related programs yet to catch up the latest cuda, even essential GCC or CMAKE. and both the latest drivers and cuda were proven to be more buggy that the one served well for longer time afte release. I selected v570.172.08
<img width="1595" height="1650" alt="image" src="https://github.com/user-attachments/assets/596b3c86-2878-4139-955a-308d5e8bd86c" />

make the NVIDIA-Linux****.run file an executable one
>sudo chmod +x NVIDIA-Linux***.run

8. Disable the running desktop GUI display (if the terminal is stuck, call another terminal using any of the F1-F7 key, as example Ctl+Alt+F3)
>sudo init 3
>
9. Install Nvidia driver (Select MIT license, 32-bit support) and check that your display driver is nvidia module after restart
>cd ~/Download
>sudo ./NVIDIA-Linux-******.run

If the installation fails for some minor reason. repeat step 8 & 9 to reinstall it complete.
>sudo reboot now

>nvidia-smi (should show that system doesn't have connected Nvidia device)

10. shutdown and connect the power cable(s) of your GPU as you did in step-1 and start your PC.
Check that nvidia-smi shows your GPUs like this-
<img width="1466" height="888" alt="Screenshot from 2025-08-24 01-40-54" src="https://github.com/user-attachments/assets/984a89da-46ab-4968-928e-1b7c1bd1103b" />

# Enjoy!






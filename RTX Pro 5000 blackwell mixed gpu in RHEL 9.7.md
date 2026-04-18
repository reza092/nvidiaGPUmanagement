# Nvidia mixed blackwell GPUs (RTX Pro 5000 Blackwell and RTX 5080) in RHEL 9.7 broke the Graphical Display Manager  (GDM) giving blackscreen

Display issue came back after I swapped one of the two GPUs (RTX 5070ti and RTX 5080) with RTX Pro 5000 blackwell

***
** Start the system in console text mode (tty)

** NVIDIA driver

** X display

** Fix the driver loading failure and GDM 
***

## Start the system in console text mode (tty)

Case-1: If the screen only shows blinking cursor, do Ctl+Alt+F3 (or F1/F2/F4 try if required). That will bring the text log in mode (tty)

Case-2: If the screen is only black and you can't initiate tty mode as case-2> Follow these steps
Edit the GRUB menu to start in console text mode (tty)
1. Interrupt Boot- Reboot your RHEL machine and interrupt the boot process (usually by holding Shift or pressing Esc in bios or immediately after) to bring up the GRUB menu.
2. Edit Entry carefully- Use the arrow keys to highlight your default kernel and press 'e' to edit.
3. Locate Line to edit- by finding the line starting with linux, or linuxefi
4. Modify Parameter to text mode- Go to the end of the line (use End key) and add a space followed by "systemd.unit=multi-user.target" (for RHEL 7+ to stop GUI), carefully make sure the spaces in between texts are right, and No additional space after the text Check if that is applicable for RHEL 10 as well.
5. Boo- Press Ctrl+X or F10 to boot with these temporary settings

## Install proper NVIDIA driver or check nvidia driver status
In the tty mode, check if the system has running nvidia driver handling the gpus-
>nvidia-smi

If it doesn't run, follow the instructions here https://github.com/reza092/nvidiaGPUmanagement/blob/main/Nvidia_GeForce_5080_GPU_driver_linux.md and comeback in tty mode

##  X display
You need to find out and make sure that X display runs with Xorg. In tty consule, run this
```bash
sudo systemctl stop gdm # stops the GDM if it is running in the background
startx # this should bring the display or log in screen. If it doesn't and goes to blackscreen,
#hit Ctl+Alt+F1/F2/F3/F4. Otherwise, get in a terminal window or work in tty console to stop the Xorg with GDM
sudo systemctl stop gdm
```


## Fix the driver loading failure and GDM because of conflict with nouveau driver loading

1. You need to know what gets loaded in RAM during OS boot? Do-

```bash
ls /etc/dracut.conf.d/

#If it gives nouveau.conf and nvidia.conf or no nvidia.conf, then that is significant. `nouveau.conf` in `/etc/dracut.conf.d/` is almost certainly written by the `.run` installer
— and it may be doing the opposite of what you need, or doing it incompletely.
```

## First: Check What's Actually Inside It
```bash
cat /etc/dracut.conf.d/nouveau.conf

#The `.run` installer typically writes one of these:

#Common .run installer output — omits nouveau from initramfs
omit_drivers+=" nouveau "
#or just:
blacklist nouveau
```
** Why This May Still Be Broken

There are three possible problems depending on what the file contains:

**Scenario A — File only omits nouveau but doesn't add NVIDIA modules**
Nouveau is excluded from initramfs, but `nvidia`, `nvidia_drm`, `nvidia_modeset`, `nvidia_uvm` are never added. This means at early boot there is **no GPU driver at all**, and GDM starts against a DRM device that's uninitialized.[^3]

**Scenario B — File was written but `dracut -f` was never re-run**
The config exists but the actual initramfs on disk (`/boot/initramfs-$(uname -r).img`) was never rebuilt, so it still contains `nouveau`.[^1]

**Scenario C — File omits nouveau but `modprobe.blacklist` is missing from kernel cmdline**
`omit_drivers` only removes nouveau from the initramfs image — it doesn't prevent it from being loaded later from the rootfs. Without `modprobe.blacklist=nouveau` on the kernel cmdline, nouveau can still load from `/lib/modules/` after initramfs hands off to systemd

***

## The Complete Fix

Handle all three scenarios at once:

```bash
# 1. Replace/augment nouveau.conf with a complete config
sudo tee /etc/dracut.conf.d/nouveau.conf << 'EOF'
omit_drivers+=" nouveau "
EOF

# 2. Add NVIDIA modules (create separately for clarity)
sudo tee /etc/dracut.conf.d/nvidia.conf << 'EOF'
add_drivers+=" nvidia nvidia_modeset nvidia_uvm nvidia_drm "
EOF

# 3. Ensure modprobe blacklist exists
sudo tee /etc/modprobe.d/blacklist-nouveau.conf << 'EOF'
blacklist nouveau
options nouveau modeset=0
EOF

# 4. Set modesetting on kernel cmdline: be careful here because all the kernels in grub list will be updated. Be sure that's what you want to do
sudo grubby --update-kernel=ALL \
  --args="modprobe.blacklist=nouveau nouveau.modeset=0 nvidia-drm.modeset=1 nvidia-drm.fbdev=1"

# 5. Rebuild initramfs
sudo dracut -f --kver $(uname -r)

# 6. Verify nouveau is gone from initramfs
lsinitrd /boot/initramfs-$(uname -r).img | grep nouveau   # should return nothing
lsinitrd /boot/initramfs-$(uname -r).img | grep nvidia    # should list nvidia modules
```


***

## After Reboot, Confirm

```bash
lsmod | grep nouveau          # must be empty
lsmod | grep nvidia_drm       # must show nvidia_drm
cat /sys/module/nvidia_drm/parameters/modeset   # must print Y
systemctl status gdm          # should be active (running)
```

The key - `nouveau.conf` in dracut only handles the **initramfs** layer. Without the kernel cmdline `modprobe.blacklist=nouveau` and the separate `modprobe.d` config, nouveau can still load from the rootfs after boot handoff — which is exactly what the `lsmod` output showed at the beginning.


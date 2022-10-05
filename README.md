# zram-hibernate

Allows dynamic swap changes to activate disk-based storage as swap for hibernation support when a system typically uses only zram swap during normal operation.

Some applications keep growing until they fill RAM and swap (Google Chrome is a great example).  This can significantly slow down a system by "swapping it to a crawl" - filling memory and swap to the point that the system isn't doing anything but swapping.

To avoid this problem entirely, one can simply disable swap so it can't slow down the system, but then virtual memory isn't available for the application to use during normal operation either.  Using ZRAM instead of a swap disk is a great alternative since it doesn't have as much of a negative performance impact.  A system with ZRAM, however, cannot hibernate to disk, which is a significant portability limitation for systems with a short battery life, no battery at all, or broken/missing suspend/standby functionality.

That is why I created this tool - to allow one to use ZRAM during normal operation for reasonable virtual memory expansion, while also reasonably constraining application memory use, but also allowing the system to activate disk swap when needed for hibernation.

## Linux Hibernation High-Level Summary
Linux Hibernation has 2 main phases: Shutdown and Resume.

- Shutdown is a matter of pausing applications, storing memory contents and a hibernation signature to disk-based swap, and powering off the system.
- Resume is similar to a normal system boot process, except that the kernel `resume=` parameter identifies a swap device to attempt to use to restore the system state upon bootup.  When this option is present on boot, the kernel checks the swap device specified by `resume` for the hibernation signature, and if present, it restores the system state from the swap device into memory.  Upon successful restoration, the kernel then clears the hibernation signature from the resume device and resumes execution of the system state as it was previous to Shutdown.

Resume without a hibernation signature (such as after a normal system shutdown) will result in the standard system bootup process - running an instance of init/systemd/openrc as PID 1, etc.

It is highly recommended to use a dedicated swap partition for hibernation.  Swap files are less safe for hibernation as it could trigger race conditions or caching bugs in device drivers, filesystems or kernel cache.

## Principle of Operations / How it works

This script finds and activates a disk-based swap, moves all ZRAM content to the swap device, disables ZRAM, and hibernates the system to the swap device.  Upon resume from hibernation (after the system reboots and the kernel resumes successfully) it restores the original swap configuration, restoring ZRAM, and disabling any disk swap it added for during the hibernation/shutdown phase.

Part of this process is detecting where your system wants to store it's hibernation data.  This is primarily done by analyzing your system's kernel command line for the `resume=` parameter.  This is the safest method as your system would need this to resume from hibernation after shutdown anyhow.  Assuming this is not present, however, the system can use /etc/fstab swap entries for more obscure configurations.

This script also detects if existing swap devices are already active and simply uses them if it believes there is already enough swap to contain your hibernation data.  In this case, it would also keep them active after resume, since that was the original state.

## Usage

zram-hibernate supports a number of arguments that affect its operation and provide troubleshooting information:

```
-h --help        show help text
-v --verbose     increase output verbosity
-n --dry-run     show what would be done without making changes
-t --test        do everything except actually affect the system power state
-d --debug       debug data parsing logic only
```

To safely test swap detection and operation before system-wide installation you can simply call:

```
zram-hibernate -tn
```

System-wide installation is performed by hooking into systemd's hibernation callback, placing the script or a symlink to it in `/usr/lib/systemd/system-sleep/`.

Normal usage is generally transparent after installation by executing a system power command like:

```
systemctl hibernate
```

To manually invoke hibernation without systemwide installation you can also call:

```
zram-hibernate
```

Verbosity defaults to level 2 when running manually, and 0 when called by systemctl.

If detection fails to work as expected, or you need to manually specify the hibernation device to use, you can create a configuration file `/etc/zram-hibernate.conf` with a line specifying the desired swap device.  The file contents should be similar to the following lines:

```
KERNEL_SWAP_DEVICE=/dev/mapper/swap-device
KERNEL_SWAP_DEVICE=/dev/disk/by-label/My_Swap_Disk
KERNEL_SWAP_DEVICE=/dev/disk/by-uuid/1234567890ABCDE
KERNEL_SWAP_DEVICE=/swapfile.swp
KERNEL_SWAP_DEVICE=/swapfile.swp
```

This file is also necessary if you want to use a swap file or the `resume_offset=` kernel option, as they often mean runtime detection of the `resume=` kernel option will not properly detect the actual resume device.  Note that swap files can have unknown race conditions, and it is recommended to use a dedicated swap partition whenever possible.  If you use the configuration file, be aware that many of the sanity checks done during the detection process are no longer possible, so it's your responsibility to ensure the kernel command line arguments and configuration file contents are correctly configured to work properly.

---
Background reference information for the extremely curious follows...

Information about integration with systemd:

https://blog.christophersmart.com/2016/05/11/running-scripts-before-and-after-suspend-with-systemd/

Short summary: add script to /usr/lib/systemd/system-sleep/ that supports pre/post arguments.  More details available in systemd-suspend.service manpage.

There's likely a way to integrate with openrc as well, possibly using apcid or something like that, but I haven't needed to do so yet.

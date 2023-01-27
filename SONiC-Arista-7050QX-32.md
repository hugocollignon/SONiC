# How to run SONiC on Arista 7050QX-32

This device no longer works out of the box with SONiC, we have some changes to do.

# History and context

The internal flash of this device is only 2 GB.

- In 2017, an improvement to save space uses pipe: https://github.com/sonic-net/sonic-buildimage/commit/b968cf73a199f485dccacfd5353ec55dfd36c0c0
- In 2019, a second trick uses a tmpfs to have enough space again: https://github.com/sonic-net/sonic-buildimage/commit/2b28d5585338da28ba3988e8ef25b2289227fbb0

But now, both of the flash and the tmpfs are too short because SONiC images have grown. By enlarging the tmpfs, we probably not have enough RAM to start properly SONiC.

Even if it could have another optimisations to manage that, this way is not encouraged:
- It takes time to develop for only one device
- SONiC images will continue to grow
- Updates will not be possible (to store both of the running and next version)

The easier and quicker way is to change the internal flash (and it's really easy).

# Bill of materials

This device uses an USB DOM (Disk on Module), which is like an USB key directly on an USB internal header. It looks like that:  
![image](/uploads/b3cc8f23fe01ead68b14f85b3b997f3a.png)

Regarding size, 8 GB is the minimum and 16 GB (or more) is recommended.

## Solution 1

The best way is to find a similar component with a bigger size. In practice, it could be hard to find (in particular for larger capacity) and is quite expensive. If you find something which fits, don't hesitate to share a link.

### Hardware needed:

- USB DOM 8-16+ GB

## Solution 2

Alternatively, we can add an adapter `USB internal header male to USB A (or C) receptacle` and plug any USB key inside. This is cheaper, availability would be better, it works like a charm, but it is not the "most elegant" solution.

Some adapters exist on PCB without cable : we didn't find one with 90° angle. If you find something interesting, again don't hesitate to share.  
![image](/uploads/fa0f7ff07c153566caf23335f4c47ea9.png)

Notes:
- A manufacturer is not more recommended than another, you will probably not have issues with any decent chip
- It's useless to have an adapter with 2 USB ports
- It could be a good idea to stay away from exposed metal / conductive elements to avoid any electrical issue

### Hardware needed:

- USB internal header male to USB A (or C) receptacle
- USB Key 8-16+ GB
- Glue

For informations only, this is what was used here. Reminder: SONiC is not related to a manufacturer, do not consider the following as promotion.
- Adapter  
![image](/uploads/a3e42fd01628e7d7847db38eba5cb800.png)  
https://www.amazon.fr/gp/product/B00DKWY0S8/
- USB Key  
![image](/uploads/a47f387960a2c678eb523abb396a7817.png)  
https://www.amazon.fr/gp/product/B077VXV323/

# Hardware "hack"

- Remove lateral rails (if any)
- Remove lateral screws and slide the top cover to the frond to open the switch
- Look at the USB DOM, already taken off on this picture  
![image](/uploads/f4f7a65b483b6de5f3c3934ed65ee4a7.jpg)
- Unplug it (keep the plastic screw)  
![image](/uploads/d6ee034594f5f199bc019e4810cebb5c.jpg)
- The best location for the USB connector is on PSU2 slot, glued on it  
![image](/uploads/fa44bc6409d95542dda7852128fd2344.jpg)
- When it's dry, plug exactly like this the adapter (only one row of the header is wired on the motherboard)  
![image](/uploads/2deca9980fc7340b163c076bb115e43d.jpg)
- Plug the USB key, test if your device recognize it and if all goes well, reassemble everything

# Software patch

At this date (20221110), the size of the flash is not automatically discovered for this device. We will enable this feature by hand. To do this, we have to edit `boot0` file but it is located in two places.

- Get the SONiC image you want
  - On https://github.com/sonic-net/SONiC/wiki/Supported-Devices-and-Platforms, you can find the link to pipelines: https://sonic-build.azurewebsites.net/ui/sonic/Pipelines
  - Follow Platform: `broadcom` > BranchName: `your choice` > Builds: `Build History` > Result: `last succeeded build` > Artifacts: `Artifacts` > Name: `sonic-buildimage.broadcom` > Name: `target/sonic-aboot-broadcom.swi`
- Extract `.sonic-boot.swi` and `boot0` from the downloaded file  
```
unzip sonic-aboot-broadcom.swi .sonic-boot.swi boot0
```
- Search `raven` (which is the platform name in the bootloader), line 478 (may change)  
```
    if [ "$platform" = "raven" ]; then
        # Assuming sid=Cloverdale
        aboot_machine=arista_7050_qx32
        flash_size=2000                                        # Delete
        cmdline_add modprobe.blacklist=radeon,sp5100_tco
        cmdline_add acpi=off
    fi
```
- Then delete the line containing `flash_size`
- Update the modified file in both locations  
```
zip -u .sonic-boot.swi boot0
zip -u sonic-aboot-broadcom.swi .sonic-boot.swi boot0
```
- Now you can use this .swi file

# USB recovery key

Because we changed the internal flash, Aboot bootloader is the only prompt we have on the switch.

Following https://www.arista.com/en/um-eos/eos-recovery-procedures#xx1129071:
- Format an USB key with FAT
- Create a file named `boot-config` containing
```
SWI=flash:sonic-aboot-broadcom.swi
```
- Copy your `sonic-aboot-broadcom.swi` image

Do not plug it in the switch yet.

# First boot

Aboot does not automatically create partitions on an empty media.

- Boot the switch, only with the **internal** USB key you installed before but not for now the **external** one (if you do this, the switch will consider the **external** as the **internal** and will install SONiC on the external one)
- You should get a prompt on Aboot
- To ensure that there are nothing (from manufacturer for example) on the **internal**
```
dd if=/dev/zero of=/dev/sda
```
- Create table partition and one partition  
```
fdisk /dev/sda
n p 1 [default] [default]
p # Print
w # Write
```
/!\ We got strange errors like `unzip short read` at the next boot when the partition was too large. To get it works, you should set `Last cylinder <= 24588`. Don't worry, the SONiC installer will properly clean that at first boot.
- Create filesystem
```
mkfs.vfat /dev/sda1
```
- Plug the **external**
- Reboot
- You should have `/mnt/flash` and `/mnt/usb1` automatically mounted
- Copy content of **external** to **internal**
```
cp /mnt/usb1/boot-config /mnt/usb1/sonic-aboot-broadcom.swi /mnt/flash
```
- Unmount the **external** then unplug it
```
umount /mnt/usb1
```
- Reboot
- SONiC boots and installs itself
```
Arista Networks Inc...


Aboot 2.1.0-3037058


Press Control-C now to enter Aboot shell
Booting flash:sonic-aboot-broadcom.swi
6.61: Cleaning flash content /mnt/flash
6.63: Generating boot-config, machine.conf and cmdline
6.76: Installing image under /mnt/flash/image-202205.161660-cfc9af71e
6.76: Moving swi to a tmpfs
39.28: Extracting swi content
77.56: Extracting platform.tar.gz
77.75: Extracting dockerfs.tar.gz from swi
123.65: Unpacking dockerfs.tar.gz delayed to initrd because /mnt/flash is vfat or docker_inram is on
123.65: Remove installer
df: /tmp/tmp.9C8swD/sonic-aboot-broadcom.swi: can't find mount point
125.02: Next reboot will use flash:image-202205.161660-cfc9af71e/.sonic-boot.swi
125.91: Kexecing[  125.913597] Starting new kernel
...
�2048+0 records in
2048+0 records out
Checking that no-one is using this disk right now ... OK

Disk /dev/sda: 28.65 GiB, 30765219840 bytes, 60088320 sectors
Disk model:  SanDisk 3.2Gen1
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

>>> Created a new DOS disklabel with disk identifier 0x1e0688c0.
/dev/sda1: Created a new partition 1 of type 'Linux' and of size 28.7 GiB.
/dev/sda2: Done.

New situation:
Disklabel type: dos
Disk identifier: 0x1e0688c0

Device     Boot Start      End  Sectors  Size Id Type
/dev/sda1        2048 60088319 60086272 28.7G 83 Linux

The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
mke2fs 1.46.2 (28-Feb-2021)
Creating filesystem with 7510784 4k blocks and 1880480 inodes
Filesystem UUID: e2889a7e-b64a-4ba5-b8ab-44007d64cbd4
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
        4096000

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done   

tune2fs 1.46.2 (28-Feb-2021)
Setting reserved blocks percentage to 0% (0 blocks)
Setting reserved blocks count to 0
[  297.710554] rc.local[471]: + cat /etc/sonic/sonic_version.yml
[  297.787436] rc.local[472]: + grep build_version
[  297.932964] rc.local[484]: + sed -e s/build_version: //g;s/'//g
[  298.013865] rc.local[466]: + SONIC_VERSION=202205.161660-cfc9af71e
[  298.090427] rc.local[466]: + FIRST_BOOT_FILE=/host/image-202205.161660-cfc9af71e/platform/firsttime
[  298.202342] rc.local[466]: + SONIC_CONFIG_DIR=/host/image-202205.161660-cfc9af71e/sonic-config
[  298.308487] rc.local[466]: + SONIC_ENV_FILE=/host/image-202205.161660-cfc9af71e/sonic-config/sonic-environment
[  298.438321] rc.local[466]: + [ -d /host/image-202205.161660-cfc9af71e/sonic-config -a -f /host/image-202205.161660-cfc9af71e/sonic-config/sonic-environment ]
[  298.614432] rc.local[466]: + logger SONiC version 202205.161660-cfc9af71e starting up...
[  298.721012] rc.local[466]: + grub_installation_needed=
[  298.794274] rc.local[466]: + [ ! -e /host/machine.conf ]
[  298.867151] rc.local[466]: + . /host/machine.conf
[  298.926892] rc.local[466]: + aboot_version=2.1.0-3037058
[  298.998397] rc.local[466]: + aboot_vendor=arista
[  299.060371] rc.local[466]: + aboot_platform=x86_64-arista_7050_qx32
[  299.147135] rc.local[466]: + aboot_machine=arista_7050_qx32
[  299.218451] rc.local[466]: + aboot_arch=x86_64
[  299.286562] rc.local[466]: + aboot_build_date=2016-03-14T19:11:31.000000000
[  299.375159] rc.local[466]: + program_console_speed
[  299.443752] rc.local[498]: + cat /proc/cmdline
[  299.504105] rc.local[500]: + cut -d , -f2
[  299.560284] rc.local[499]: + grep -Eo console=ttyS[0-9]+,[0-9]+
[  299.634502] rc.local[466]: + speed=
[  299.679040] rc.local[466]: + [ -z  ]
[  299.722358] rc.local[466]: + CONSOLE_SPEED=9600
[  299.782951] rc.local[502]: + grep keep-baud
[FAILED] Failed to start OpenBSD Secure Shell server.
[  299.842737] rc.local[501]: + grep agetty /lib/systemd/system/serial-getty@.service
[  300.034661] rc.local[502]: ExecStart=-/sbin/agetty -o '-p -- \\u' --keep-baud 115200,57600,38400,9600 %I $TERM
[  300.164506] rc.local[466]: + [ 0 = 0 ]
[FAILED] Failed to start OpenBSD Secure Shell server.
[  300.224600] rc.local[466]: + sed -i s|\-\-keep\-baud .* %I| 9600 %I|g /lib/systemd/system/serial-getty@.service
[  300.452826] rc.local[466]: + systemctl daemon-reload
[  300.522336] rc.local[466]: + [ -f /host/image-202205.161660-cfc9af71e/platform/firsttime ]
[FAILED] Failed to start OpenBSD Secure Shell server.
[  300.630731] rc.local[466]: + echo First boot detected. Performing first boot tasks...
[  300.822426] rc.local[466]: First boot detected. Performing first boot tasks...
[  300.916849] rc.local[466]: + [ -n x86_64-arista_7050_qx32 ]
[  300.986435] rc.local[466]: + platform=x86_64-arista_7050_qx32
[  301.062611] rc.local[466]: + [ -d /host/old_config ]
[  301.122515] rc.local[466]: + [ -f /host/minigraph.xml ]
[FAILED] Failed to start OpenBSD Secure Shell server.
[  301.192686] rc.local[466]: + [ -n  ]
[  301.342400] rc.local[466]: + touch /tmp/pending_config_initialization
[  301.430406] rc.local[466]: + touch /tmp/notify_firstboot_to_platform
[  301.510424] rc.local[466]: + [ ! -d /host/reboot-cause/platform ]
[  301.598428] rc.local[466]: + mkdir -p /host/reboot-cause/platform
[  301.674382] rc.local[466]: + [ -d /host/image-202205.161660-cfc9af71e/platform/x86_64-arista_7050_qx32 ]
[FAILED] Failed to start OpenBSD Secure Shell server.
[  301.800550] rc.local[466]: + sync
[  301.945422] rc.local[466]: + [ -n  ]
[  301.995122] rc.local[466]: + mkdir -p /var/platform
[  302.054356] rc.local[466]: + [ -f /etc/default/kdump-tools ]
[  302.130436] rc.local[466]: + sed -i -e s/__PLATFORM__/x86_64-arista_7050_qx32/g /etc/default/kdump-tools
[  302.250544] rc.local[466]: + firsttime_exit
[  302.306413] rc.local[466]: + rm -rf /host/image-202205.161660-cfc9af71e/platform/firsttime
[FAILED] Failed to start OpenBSD Secure Shell server.
[  302.421546] rc.local[466]: + exit 0
[  302.559770] kdump-tools[458]: Starting kdump-tools:
[  302.619567] kdump-tools[549]: no crashkernel= parameter in the kernel cmdline ...
[FAILED] Failed to start OpenBSD Secure Shell server.
[  302.715069] kdump-tools[606]:  failed!

Debian GNU/Linux 11 sonic ttyS0

sonic login: [  311.224810] arista: waiting for switch chip
[  311.475814] arista: switch chip is ready
[  313.524978] arista: yielding...
```

## openssh issue

You may encounter an issue when `sshd` tries to start: `[FAILED] Failed to start OpenBSD Secure Shell server.`.

Can be solved with:

```
sudo apt remove openssh-client openssh-server
sudo apt update
sudo apt install openssh-server
```

# Source

https://github.com/sonic-net/SONiC/issues/592

Special thanks to @Staphylo for his help to debug this issue.

# CIA Lab 2 - Booting
#### Artem Abramov SNE19

May I suggest viewing this document in your browser at address: 
https://github.com/temach/innopolis_university_reports/
Unfortunately rendering the document to PDF breaks some long lines and crops images.

## Task 1 - Loading the OS

### 1. What is an UEFI OS loader and where does the Ubuntu OS loader reside on the system? Hint: See the UEFI specification.

What is a "UEFI OS loader"? "OS loader" is program to load the OS. Then at a higher level of abstraction "UEFI OS loader" can be called a "UEFI program (to load the OS)". 

Now lets understand what is a "UEFI program". UEFI firmware is capable of executing programs that conform to the UEFI specification. To conform to the specification the program must:
1. Be stored on the ESP (EFI System Partition) partition that is formatted as FAT32 (source: https://wiki.osdev.org/UEFI) and has partition type set to C12A7328-F81F-11D2-BA4B-00A0C93EC93B.
2. Be stored in a PE (Portable Executable) format (an alternative to, for example, the ELF format). (source: https://wiki.osdev.org/UEFI and https://wiki.osdev.org/PE).

Such programs may be EFI drivers, EFI scripts, bootloaders, bootmanagers and finally just user applications such as a Hello World shell (source: https://github.com/tianocore/edk2/blob/master/ShellPkg/Application/ShellCTestApp/ShellCTestApp.c). Most common ways to create such programs is using "EDK II" from TianoCore project or using the GNU-EFI project (source: https://www.rodsbooks.com/efi-programming/prepare.html).

To sum up a "UEFI program" is a program that UEFI firmware can find, load and execute. The UEFI specification requires that the UEFI firmware can run UEFI programs.

To be completely clear, the term "UEFI firmware" refers to the code that is (normally) developed by the motherboard manufacturer and comes pre-installed in the motherboard ROM. In older motherboards this ROM would contain BIOS code. Another interesting note is that UEFI firmware acts as a half-baked OS. It provides a standard library with quite a few functions, it can load and execute user programs, user programs can finish execution and return control back to UEFI firmware. The conclusion to draw from the above is that UEFI is a reasonably comlicated piece of software with a different implementation by each manufacturer! UEFI specification allows the manufacturers to include many more interesting functions than what is required by the UEFI specification.

Now lets look back at the "UEFI OS loader" term and focus on the "OS loader" part. 
An OS loader is a program that loads the kernel image into memory and begins its execution. 
The typical case for a GNU/Linux distribution is to keep the kernel as an ELF (Executable and Linkable Format) file stored in the /boot/ directory on an ext4 partition. The kernel is actually compressed, but will self-extract on execution, more info on kernel booting can be found here: https://blog.lse.epita.fr/cat/sustem/system-linux/index.html. 

The filename used for the kernel image is normally `bzImage` or `vmlinuz` (source with description: https://unix.stackexchange.com/questions/5518/what-is-the-difference-between-the-following-kernel-makefile-terms-vmlinux-vml and http://www.linfo.org/vmlinuz.html).
The bootloader normally handles loading the initrd as well, but that is not strictly necessary as it depends on how the kernel was build (source: https://stackoverflow.com/questions/6405083/is-it-possible-to-boot-the-linux-kernel-without-creating-an-initrd-image). 
To get an overview of the typical steps to load the kernel in GRUB see the link (its for MBR, but applies to GPT too): https://www.unix-ninja.com/p/Manually_booting_the_Linux_kernel_from_GRUB

The UEFI specification does not require that UEFI firmware know anything about ELF or how to mount and search ext4 filesystem for the kernel image. This is the job of the bootloader (OS loader).

Therefore (in the GNU/Linux world) the "UEFI OS loader" is a UEFI program that knows how to mount the ext4 filesystem, search it for a linux kernel image, load the image and execute it.

When a new OS is installed on a GPT disk, one of the steps during the installation is making sure a UEFI OS loader program is present and correctly configured to load the newly installed kernel. The UEFI OS loader is placed in a subdirectory under the EFI directory on the ESP partition. To avoid name collisions organisations should register a subdirectory name with the UEFI Forum, the current list of registered subdirectories can be seen here: https://uefi.org/registry. 

In an attempts to pinpoint the location of the UEFI bootloader lets examine the contents of the ESP partition. We can find an "ubuntu" folder there.
Mounting the ESP partition and listing files in `\EFI\ubuntu\` is shown below:
```
# mount /dev/sda1 /mnt/esp
# ls -la /mnt/esp/EFI/ubuntu
total 3732
drwx------ 3 root root    4096 Aug 19 13:22 .
drwx------ 5 root root    4096 Aug 31 00:51 ..
-rwx------ 1 root root     108 Aug 19 13:22 BOOTX64.CSV
drwx------ 2 root root    4096 Aug 19 13:21 fw
-rwx------ 1 root root   75992 Aug 19 13:21 fwupx64.efi
-rwx------ 1 root root     201 Aug 19 13:22 grub.cfg
-rwx------ 1 root root 1116024 Aug 19 13:22 grubx64.efi
-rwx------ 1 root root 1269496 Aug 19 13:22 mmx64.efi
-rwx------ 1 root root 1334816 Aug 19 13:22 shimx64.efi
```

From the above we can guess that probably grubx64.efi is responsible for bootloading Ubuntu. This program is GRUB ported to UEFI platform (source: https://wiki.osdev.org/GRUB#GRUB_for_UEFI) 

The last piece of the puzzle is understanding how UEFI firmware decides which UEFI OS loader to execute (when there are multiple OS installed and hence multiple UEFI OS loaders present). This question is addressed by a bootmanager. 

There is one bootmanager that is part of the UEFI firmware (the default UEFI bootmanager). Although it is called a bootmanager, it is in fact a more general mechanism that decided which ".efi" binary to execute based on variables saved in NVRAM. This is an important mechanism to understand. See https://mjg59.livejournal.com/138188.html and https://blog.uncooperative.org/blog/2014/02/06/the-efi-system-partition/ for much more detailed explanations of how the selection mechanism works.

In short, the NVRAM variables can be manipulated by user space tools: efivar, efivarfs and efibootmgr.

The NVRAM variables can be listed with the command below:
```
$ efivar --list
8be4df61-93ca-11d2-aa0d-00e098032b8c-VendorKeys
8be4df61-93ca-11d2-aa0d-00e098032b8c-dbDefault
8be4df61-93ca-11d2-aa0d-00e098032b8c-PKDefault
8be4df61-93ca-11d2-aa0d-00e098032b8c-KEKDefault
8be4df61-93ca-11d2-aa0d-00e098032b8c-SecureBoot
8be4df61-93ca-11d2-aa0d-00e098032b8c-SignatureSupport
8be4df61-93ca-11d2-aa0d-00e098032b8c-SetupMode
36d08fa7-cf0b-42f5-8f14-68df73ed3740-PreviousBoot
8be4df61-93ca-11d2-aa0d-00e098032b8c-Boot0000
8be4df61-93ca-11d2-aa0d-00e098032b8c-Boot0007
8be4df61-93ca-11d2-aa0d-00e098032b8c-Boot0005
8be4df61-93ca-11d2-aa0d-00e098032b8c-Boot0004
8be4df61-93ca-11d2-aa0d-00e098032b8c-Boot0003
8be4df61-93ca-11d2-aa0d-00e098032b8c-Boot0002
8be4df61-93ca-11d2-aa0d-00e098032b8c-Boot0001
8be4df61-93ca-11d2-aa0d-00e098032b8c-ErrOut
8be4df61-93ca-11d2-aa0d-00e098032b8c-ConOut
-- output cropped --
```

The output of efibootmgr showing the default boot programs and their locations is shown below:
```
$ efibootmgr -v
BootCurrent: 0008
Timeout: 0 seconds
BootOrder: 0008,0000,0001,0002,0003,0004,0005,0006,0007
Boot0000* ubuntu	HD(1,GPT,f57ce334-0038-4c3b-8094-d3d2639871ae,0x800,0x100000)/File(\EFI\ubuntu\shimx64.efi)
Boot0001* DTO UEFI USB Floppy/CD	VenMedia(b6fef66f-1495-4584-a836-3492d1984a8d,0500000001)..BO
Boot0002* DTO UEFI USB Hard Drive	VenMedia(b6fef66f-1495-4584-a836-3492d1984a8d,0200000001)..BO
Boot0003* DTO UEFI ATAPI CD-ROM Drive	VenMedia(b6fef66f-1495-4584-a836-3492d1984a8d,0300000001)..BO
Boot0004* CD/DVD Drive 	BBS(CDROM,,0x0)..GO..NO{.......+.S.A.T.A. . .P.M.:. .h.p. . . . . . . .D.V.D.R.A.M. .G.T.B.0.........................rN.D+..,.\...........BO
Boot0005* DTO Legacy USB Floppy/CD	VenMedia(b6fef66f-1495-4584-a836-3492d1984a8d,0500000000)..BO
Boot0006* Hard Drive	BBS(HD,,0x0)..GO..NO?.........F.a.k.e. .U.s.b. .O.p.t.i.o.n.................BO..NOc.......+.T.O.S.H.I.B.A. .D.T.0.1.A.C.A.1.0.0.........................rN.D+..,.\...........BO
Boot0007* IBA GE Slot 00C8 v1550	BBS(Network,,0x0)..BO
Boot0008* rEFInd	HD(1,GPT,f57ce334-0038-4c3b-8094-d3d2639871ae,0x800,0x100000)/File(\EFI\refind\refind_x64.efi)
```

On my machine the BootOrder starts with entries 0008 and then 0000. 

1. Boot option 0000 points to the path `\EFI\ubuntu\shimx64.efi` which is the OS loader for Ubuntu. The file is called `shimx64.efi`, it is actually a wrapper that deals with some Secure Boot issues before loading the second stage image: GRUB or MokManager (source: https://wiki.ubuntu.com/UEFI/SecureBoot). More info about the UEFI shim can be found here: http://www.rodsbooks.com/efi-bootloaders/secureboot.html#initial_shim). 

2. Boot option 0008 points to `\EFI\refind\refind_x64.efi` which is a UEFI program "rEFInd" (https://www.rodsbooks.com/refind/index.html), this program is a custom bootmanager. On my machine I have configured the default UEFI bootmanager to "boot" (i.e. execute) rEFInd using the configuration steps from https://www.rodsbooks.com/refind/installing.html.

See http://www.rodsbooks.com/efi-bootloaders/index.html for a deeper discussion of the differences between a bootloader and a bootmanager.

Finally returning to the original question: the UEFI OS loader for Ubuntu is the shim program that is located on the ESP partition in the path `\EFI\ubuntu\shimx64.efi`, after the shim handles the SecureBoot procedure correctly it loads GRUB `\EFI\ubuntu\grubx64.efi` that does the actual loading of the linux kernel for Ubuntu. If Secure Boot is disabled the shim goes directly to loading GRUB.


### 2. Describe in order all the steps required for booting the computer (until the OS loader starts running.)





### 3. What is the purpose of the GRUB boot loader in a UEFI system?

As already state above, the UEFI specification does not require that UEFI firmware know anything about EFL or how to mount and search ext4 filesystem for the kernel image. In the GNU/Linux world the booloader is a UEFI program that knows how to mount the ext4 filesystem, search it for a linux kernel image, load the image and execute it. 

In the case of the linux kernel the UEFI bootloader can be provided in at least two  ways:

1. A `*.efi` bootloader program in a subdirectory on the `EFI System Partition` (source https://www.happyassassin.net/2014/01/25/uefi-boot-how-does-that-actually-work-then/). This is the approach used by Ubuntu. The possible bootloader programs include GRUB (which is also a bootmanager), SYSLINUX, ELILO.

2. The kernel can be configured with a built-in UEFI bootloader. Officially called the EFISTUB (source: http://www.rodsbooks.com/efi-bootloaders/efistub.html and https://lkml.org/lkml/2011/10/17/81). In this case the linux kernel becomes a normal UEFI application and must reside on the ESP partition. Then it can be loaded by the default UEFI bootmanager directly.

Another source discussing the two methods of providing the UEFI OS loader: https://unix.stackexchange.com/questions/83744/why-do-most-distributions-chain-uefi-and-grub.

GRUB however is more than just a bootloader, because it is also a bootmanager. We can edit its configuration to add other boot options (other kernels or same kernel with different parameters).


### 4. How does the Ubuntu OS loader load the GRUB boot loader?

The word "Ubuntu" in the question seems to be a mistake. Without it the question becomes: `How does the OS loader load the GRUB boot loader?` which makes much more sense to answer.

To understand how the OS loader loads the GRUB boot loader we must look at the whole chain of events that starts from the default UEFI bootmanager configuration (that is stored in NVRAM).

Below is the output of efibootmgr command on my system.
```
$ efibootmgr -v
BootCurrent: 0008
Timeout: 0 seconds
BootOrder: 0000,0008,0001,0002,0003,0004,0005,0006,0007
Boot0000* ubuntu	HD(1,GPT,f57ce334-0038-4c3b-8094-d3d2639871ae,0x800,0x100000)/File(\EFI\ubuntu\shimx64.efi)
Boot0001* DTO UEFI USB Floppy/CD	VenMedia(b6fef66f-1495-4584-a836-3492d1984a8d,0500000001)..BO
Boot0002* DTO UEFI USB Hard Drive	VenMedia(b6fef66f-1495-4584-a836-3492d1984a8d,0200000001)..BO
Boot0003* DTO UEFI ATAPI CD-ROM Drive	VenMedia(b6fef66f-1495-4584-a836-3492d1984a8d,0300000001)..BO
Boot0004* CD/DVD Drive 	BBS(CDROM,,0x0)..GO..NO{.......+.S.A.T.A. . .P.M.:. .h.p. . . . . . . .D.V.D.R.A.M. .G.T.B.0.........................rN.D+..,.\...........BO
Boot0005* DTO Legacy USB Floppy/CD	VenMedia(b6fef66f-1495-4584-a836-3492d1984a8d,0500000000)..BO
Boot0006* Hard Drive	BBS(HD,,0x0)..GO..NO?.........F.a.k.e. .U.s.b. .O.p.t.i.o.n.................BO..NOc.......+.T.O.S.H.I.B.A. .D.T.0.1.A.C.A.1.0.0.........................rN.D+..,.\...........BO
Boot0007* IBA GE Slot 00C8 v1550	BBS(Network,,0x0)..BO
Boot0008* rEFInd	HD(1,GPT,f57ce334-0038-4c3b-8094-d3d2639871ae,0x800,0x100000)/File(\EFI\refind\refind_x64.efi)
```

In this configuration the boot entry 0000 is the ubuntu entry. The UEFI program that will be executed by the UEFI firmware for this entry is `\EFI\ubuntu\shimx64.efi`. 

The details of how exactly the UEFI firmware executes this program are not 100% clear, because each UEFI implementation is different. However they all must mount the ESP, find the program file, load it into memory according to the PE (Portable Executable) specification, and then start its execution. 


The UEFI firmware also provides some arguments to the running program.
Most importantly the UEFI firmware must provide the program with a pointer to EFI_SYSTEM_TABLE stuructue, which contains pointers to the EFI_RUNTIME_SERVICES and EFI_BOOT_SERVICES tables (source: https://uefi.org/sites/default/files/resources/UEFI_Spec_2_8_final.pdf). These tables describe the capabilities of the system and serve as a standard library of functions. If the program was build using the GNU-EFI framework then its main function will be of the form:
```
EFI_STATUS efi_main (EFI_HANDLE image_handle, EFI_SYSTEM_TABLE *systab);
```
source: https://blog.lse.epita.fr/cat/sustem/system-linux/index.html

In my case UEFI firmware executes `shimx64.efi`. This is a Shim program. It is used to handle Secure Boot correctly. After it has done its job (which may be nothing if Secure Boot is disabled) it proceeds to load the "second-stage image" as described in the official Ubuntu documentation (source: https://wiki.ubuntu.com/UEFI/SecureBoot) this is normally GRUB.

When GRUB binary is executed it presents a menu to the user, so to be pedantic the OS loader never actually loads the "GRUB bootloader", it loads just GRUB which is a bootmanager and a bootloader in one executable.

### 5. Explain how the GRUB boot loader, in turn, loads and run the kernel by answering these 3 questions:

#### (a) What type of filesystem is the kernel on?

The linux kernel is normally stored either on the same partition as the user's home partition or it can be stored on a dedicated /boot partition. The filesystem depends on user choice, however the user would be smart to choose a filesystem that is supported by the GRUB bootloader. Otherwise his system might not boot. The most commonly used filesystem is ext4 or its older variant ext3.

However since linux version 3.3, the kernel has a EFI boot stub availiable. This means that the kernel can be a UEFI application, i.e. it has a stub that will be interpreted correctly by the UEFI firmware and that will bootload the kernel (source: https://blog.lse.epita.fr/cat/sustem/system-linux/index.html). In this case the kernel can reside on the ESP which uses a FAT32 filesystem. 

#### (b) What type(s) of filesystem does UEFI support?

According to the UEFI specification every compliant UEFI firmware must support at least the FAT32 filesystem with which the ESP is formatted. The UEFI firmware must also support CD-ROM filesystems to boot from removable disks. The CD-ROM is assumed to contain an ISO-9660 file system and follow the CD-ROM "El Torito" format (source: https://uefi.org/sites/default/files/resources/UEFI_Spec_2_8_final.pdf).

However the specification does not limit the namufacturers from supporting more filesystems than is necessary. The prime example being Apple, their UEFI firmware comes with support for the HFS+ filesystem. Additional drivers for other filesystems can be installed as UEFI programs. For example rEFInd boot manager can make use of additional drivers when searching for kernels (source: https://rodsbooks.com/refind/drivers.html).

#### \(c\) What does the GRUB boot loader therefore have to do to load the kernel?

The first problem GRUB has to solve when started as a UEFI program is finding the grub config file. It turns out that grubx64.efi binary can read the grub.cfg file that is in the same directory as grubx64.efi on the ESP (source: https://askubuntu.com/questions/721970/multiple-grub-installs-with-uefi-how-do-they-find-their-config).
There are a couple of different ways to pass the config to GRUB, but ubuntu uses the grub.cfg approach as can be seen from the listing of the ubuntu folder on the ESP partiton that is shown below.
```
# mount /dev/sda1 /mnt/esp
# ls -la /mnt/esp/EFI/ubuntu
total 3732
drwx------ 3 root root    4096 Aug 19 13:22 .
drwx------ 5 root root    4096 Aug 31 00:51 ..
-rwx------ 1 root root     108 Aug 19 13:22 BOOTX64.CSV
drwx------ 2 root root    4096 Aug 19 13:21 fw
-rwx------ 1 root root   75992 Aug 19 13:21 fwupx64.efi
-rwx------ 1 root root     201 Aug 19 13:22 grub.cfg
-rwx------ 1 root root 1116024 Aug 19 13:22 grubx64.efi
-rwx------ 1 root root 1269496 Aug 19 13:22 mmx64.efi
-rwx------ 1 root root 1334816 Aug 19 13:22 shimx64.efi
```

Grub parses the config file, in fact its a very small file. Below are its contents:
![](https://i.imgur.com/pNdW3xL.png)

Essentially it tells GRUB to go look for the real config and specifies the partition and path of the real grub config. In this case it is `/boot/grub/grub.cfg` on my primary partition. A side note: as can be seen from the screenshot, grub refers to "lvmid/" which is LVM (Logical Volume Manager) id instead of device partition, because I actually have LVM configured on this system and the config takes that into account.

Then GRUB has to mount the filesystem on the hard disk, find and parse its real config.
The filesystem is typically ext4 and GRUB comes with support for ext4.

The configuration file will tell GRUB what kernels are present on the system and their location. Then GRUB presents a bootmanager menu to me (to the user). After a certain kernel is choosen GRUB runs the commands specified in its config for this particular kernel that are necessary to load the kernel, the initrd and to execute it.

The typical case for a GNU/Linux distribution is to keep the kernel as an ELF file stored in the /boot/ directory on the same filesystem where the grub config is stored (the actual kernel is compressed, but will decompress itself when executed). The filename used for the image is normally `bzImage` or `vmlinuz`. More detailes on the types of file used for the kernel image can be found at:

1. https://unix.stackexchange.com/questions/5518/what-is-the-difference-between-the-following-kernel-makefile-terms-vmlinux-vml
2. http://www.linfo.org/vmlinuz.html

The GRUB bootloader normally performs a load of the initrd as well, but that is not strictly necessary as it depends on how the kernel was build (source: https://stackoverflow.com/questions/6405083/is-it-possible-to-boot-the-linux-kernel-without-creating-an-initrd-image). 

GRUB will perform the following commands to load ubuntu on my machine, as taken from the config, shown below:

```
menuentry 'Ubuntu' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-a31450be-66ad-441d-b4d4-3a0b4f61d683' {
        recordfail
        load_video
        gfxmode $linux_gfx_mode
        insmod gzio
        if [ x$grub_platform = xxen ]; then insmod xzio; insmod lzopio; fi
        insmod part_gpt
        insmod lvm
        insmod ext2
        set root='lvmid/qSD3Zf-mcjh-Eidg-fRgT-HVuZ-s3e6-EaBCMS/sNe5fw-Rqmq-R3Cd-VdXx-PSR9-kLv6-9vutDG'
        if [ x$feature_platform_search_hint = xy ]; then
          search --no-floppy --fs-uuid --set=root --hint='lvmid/qSD3Zf-mcjh-Eidg-fRgT-HVuZ-s3e6-EaBCMS/sNe5fw-Rqmq-R3Cd-VdXx-PSR9-kLv6-9vutDG'  a31450be-66ad-441d-b4d4-3a0b4f61d683
        else
          search --no-floppy --fs-uuid --set=root a31450be-66ad-441d-b4d4-3a0b4f61d683
        fi
        linux   /boot/vmlinuz-5.0.0-25-generic root=/dev/mapper/ubuntu--vg-root ro  quiet splash $vt_handoff
        initrd  /boot/initrd.img-5.0.0-25-generic
}
```

The key parts are:
1. load the necessary GRUB modules: `gzio`, `part_gpt`, `lvm`, `ext2`. The ext2 GRUB module covers ext3 and ext4 as well (source: https://ubuntuforums.org/showthread.php?t=1460756)
1. `linux` command finds and loads the linux kernel image into memory, and also loads the kernel parameters for it to consume.
2. `initrd` command that finds and loads the compressed filesystem image that contains an initial rootfs with kernel modules.
4. After all the loading is done, GRUB executes the `boot` command to handle control off to the kernel.

To get more details on the steps see this link (its for BIOS/MBR, but applies to UEFI/GPT as well): https://www.unix-ninja.com/p/Manually_booting_the_Linux_kernel_from_GRUB

**An interesting note**: linux kernel used in Ubuntu comes with EFI_STUB support.
The command to check the kenrel for EFI_STUB support is shown below:
```
# cat /boot/config-5.0.0-23-generic | grep EFI_STUB
CONFIG_EFI_STUB=y
```
(source for how to check kernel config: https://superuser.com/questions/287371/obtain-kernel-config-from-currently-running-linux-system)

The kernel can be its own bootloader if executed by the UEFI firmware. In this configuration, GRUB would act as a bootmanager, but after the user has decided what kernel to load, GRUB would simply trigger the UEFI firmware to execute the linux kernel `.efi` program. 

There are some existing bootmanagers such as rEFInd and systemd-boot, that are simply bootmanagers and specifically exclude the bootloader functionallity, expecting the kernel to have a working EFI_STUB. 

Another source discussing the two methods of providing the UEFI OS loader: https://unix.stackexchange.com/questions/83744/why-do-most-distributions-chain-uefi-and-grub.



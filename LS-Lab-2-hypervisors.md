# LS Lab 1 - Hypervisors & Clouds

#### Artem Abramov SNE19

I decided to use the Xen hypervisor additionally compiling it from source and using a PVH for dom0 (going for the bonus task).

Source code:
```
https://wiki.xenproject.org/wiki/Xen_Project_Repositories
```

I wanted to run Xen as EFI binary, instructions for compilation: 
```
https://wiki.xenproject.org/wiki/Xen_EFI
```

I decided to build xen in a container, so keep the main system clean from build dependencies.
Get docker archlinux container:
```
docker pull archlinux
```

Run it
```
docker run --name build-xen -it archlinux /bin/bash 
```

I choose to build Xen version 4.12.2 because 4.13.0 looked a bit too raw and had same release date as 4.12.2. Anyway this is way ahead of what comes with ubuntu - 4.9.

The next multiple steps  are mandatory  to setup the compilation environment (in no particular order):

- Prepare the container: enable multilib, install multiple build packages and multiple basic packages (because the container is truly minimal):
```
pacman -Syu vim base-devel multilib-devel sudo inetutils binutils python2
```

- Make an important link that is needed during compilation:
```
sudo ln -s /usr/bin/python2 /usr/bin/python
```

- Adjust PATH to get pod2man, and other perl utils:
```
export PATH="$PATH:/usr/bin/core_perl" 
```

- Add non root user for building and enable sudo:
```
useradd -m build && visudo && su build
```

Install perl, haskel (I cant imagine where its used, but its used!), and a number of other tools:

```
  "bin86"
  "binutils"
  "bridge-utils"
  "brltty"
  "cmake"
  "curl"
  "dev86"
  "fig2dev"
  "figlet"
  "ghostscript"
  "git"
  "gnutls"
  "iasl"
  "iproute2"
  "lib32-glibc"
  "libaio"
  "libcap-ng"
  "libepoxy"
  "libiscsi"
  "libnl"
  "libpng"
  "lzo"
  "markdown"
  "nasm"
  "ocaml-findlib"
  "pandoc"
  "pciutils"
  "perl"
  "python2"
  "sdl"
  "spice"
  "spice-glib"
  "spice-protocol"
  "usbredir"
  "vde2"
  "wget"
  "yajl"
```

An annoying to resolve source code error was:
```
xentoollog_stubs.c: In function 'stub_xtl_ocaml_progress':
xentoollog_stubs.c:123:16: error: initialization discards 'const' qualifier from pointer target type [-Werror=discarded-qualifiers]
  123 |  value *func = caml_named_value(xtl->progress_cb) ;
      |                ^~~~~~~~~~~~~~~~
```
Solution to error was to disable -Werror in `tools/ocaml/common.make`.

I used the default configuration, it was pretty general, but changed install path from `/boot/efi` to `/boot`.

The only good part of compiling is the ascii logo (with high version numbers):

```
make[2]: Entering directory '/home/build/xen/src/xen-4.12.2/xen'
make -C tools
make[3]: Entering directory '/home/build/xen/src/xen-4.12.2/xen/tools'
make symbols
make[4]: Entering directory '/home/build/xen/src/xen-4.12.2/xen/tools'
make[4]: 'symbols' is up to date.
make[4]: Leaving directory '/home/build/xen/src/xen-4.12.2/xen/tools'
make[3]: Leaving directory '/home/build/xen/src/xen-4.12.2/xen/tools'
make -f /home/build/xen/src/xen-4.12.2/xen/Rules.mk include/xen/compile.h
make[3]: Entering directory '/home/build/xen/src/xen-4.12.2/xen'
 __  __            _  _    _ ____    ____  
 \ \/ /___ _ __   | || |  / |___ \  |___ \ 
  \  // _ \ '_ \  | || |_ | | __) |   __) |
  /  \  __/ | | | |__   _|| |/ __/ _ / __/ 
 /_/\_\___|_| |_|    |_|(_)_|_____(_)_____|
                                           
make[3]: Leaving directory '/home/build/xen/src/xen-4.12.2/xen'
[ -e include/asm ] || ln -sf asm-x86 include/asm
[ -e arch/x86/efi ] && for f in boot.c runtime.c compat.c efi.h;\
	do test -r arch/x86/efi/$f || \
	   ln -nsf ../../../common/efi/$f arch/x86/efi/; \
	done; \
	true
make -f /home/build/xen/src/xen-4.12.2/xen/Rules.mk -C include
make[3]: Entering directory '/home/build/xen/src/xen-4.12.2/xen
```



Then we extract the files from docker with `docker cp build-xen:/home/build/xen ~/Downloads/xen ` and run make install on the actual host own system. Risky trick, but it works and does not pollute the host with a ton of build dependencies.
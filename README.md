<div align="center">

# Ft_Linux

**Build a complete, bootable Linux distribution from source — 42 School project.**

*Linux From Scratch (LFS 13.0 / systemd branch) — kernel, toolchain, init, userspace and bootloader assembled from upstream tarballs, without any package manager.*

</div>

---

<p align="center"><i>LFS booting in Vbox — multi window, minimaliste GUI and web navigation</i></p>

[Screencast from 05-13-2026 08:18:59 PM.webm](https://github.com/user-attachments/assets/adf5e4a8-9860-4958-87c4-220c52dce59c)

---

## Table of contents
1. [What is an OS](#what-is-an-os)
2. [Build pipeline](#build-pipeline)
3. [The bootstrap problem & cross-toolchain](#the-bootstrap-problem--cross-toolchain)
4. [Pass 1 vs Pass 2](#pass-1-vs-pass-2)
5. [Why `chroot`](#why-chroot)
6. [Mandatory part — completed](#mandatory-part--completed)
7. [Bonuses — completed](#bonuses--completed)
8. [Standards respected](#standards-respected)
9. [Defense — verification commands](#defense--verification-commands)
10. [Build notes](#build-notes)
11. [References](#references)

---

## What is an OS

An operating system, stripped to its parts, is:

| Layer | Component | This project |
|---|---|---|
| Boot | Bootloader | **GRUB** (MBR, custom `grub.cfg`) |
| Kernel | Linux | Built from source (`make menuconfig`) |
| Userspace ABI | C library + ELF | **glibc**, ELF binaries |
| PID 1 | Init system | **systemd** (chose over SysVinit) |
| Storage | Filesystem | **ext4** for `/`, ext2 historically suited for `/boot` |
| Shell | Bourne-compatible | **bash** |
| Tools | GNU coreutils | tar, grep, sed, awk, find, … |
| Permissions | UNIX DAC | `umask 022`, 644/755 defaults |
| Graphical | Display server | **Xorg** + **dwm** (bonus) |

LFS deliberately ships **no package manager** — `apt`, `pacman`, etc. are themselves userspace programs maintained by the distro authors. Here, every binary is built and placed by hand.

---

## Build pipeline

```
                Host (Arch live ISO — minimal, has compiler)
                                │
   Ch.2  Partition target disk         root + /boot + swap, mkfs.ext4
   Ch.3  Fetch tarballs                wget-list + md5sums verification
   Ch.4  Prepare $LFS environment      user `lfs`, /tools symlink, .bash_profile
   Ch.5  Cross-toolchain — Pass 1      binutils + gcc(minimal) + glibc + libstdc++
   Ch.6  Temporary userspace           bash, coreutils, grep, sed, tar … built with the cross-toolchain
   Ch.7  chroot — Pass 2               re-enter $LFS as new "/", recompile cleanly
   Ch.8  Final build (~80 packages)    full GCC, full userspace, then strip + cleanup
   Ch.9  System configuration          fstab, hostname, systemd-networkd, locale, console, inputrc
   Ch.10 Kernel + GRUB                 vmlinuz-x.y.z-throbert installed to /boot, GRUB to MBR
                                │
                                ▼
                       Boot LFS ✓
```

---

## The bootstrap problem & cross-toolchain

To compile a compiler, you need a compiler. To compile a libc, you need a libc. The historical UNIX bootstrap took years (punched cards → ASM → linker → C compiler). We sidestep that by starting from a host that already has GCC — but that host’s GCC hard-codes paths into every ELF it produces:

```
Ubuntu/Arch GCC  →  /lib/x86_64-linux-gnu/libc.so.6   (host path baked in)
LFS expects      →  /lib/libc.so.6
```

So we rebuild **binutils + gcc + glibc** with a target triplet of our own, isolating the future system from the host:

```sh
export LFS_TGT=$(uname -m)-lfs-linux-gnu     # x86_64-lfs-linux-gnu
```

"Cross" here is technically a *false cross* (x86_64 → x86_64), but the triplet rename is what severs the host’s dynamic-linker path from the future LFS binaries.

---

## Pass 1 vs Pass 2

|  | **Pass 1** (Ch.5, host) | **Pass 2** (Ch.6 in host, Ch.8 in chroot) |
|--|---|---|
| Where it runs | Host distro | Host → then inside chroot |
| Linked against | Host libc | LFS libc |
| GCC capability | Bootstrap — no libc, can’t fully link | Full, self-hosting |
| Why it exists | GCC depends on libc, libc must be built by GCC → chicken/egg | Erase every trace of the host compiler from the final binaries |

`gcc -print-search-dirs` after Pass 1 still shows host paths — Pass 2 is what scrubs them.

---

## Why `chroot`

A running system has exactly **one** `/`. Multiple disks, multiple partitions, still one root. While the LFS partition is just mounted at `/mnt/lfs`, every absolute path in a build (e.g. `--prefix=/usr`) resolves to the *host’s* `/usr`, not LFS’s.

`chroot $LFS /bin/bash` tells the kernel: *for this process tree, `/mnt/lfs` is now `/`*. Side-effect: the kernel-managed virtual filesystems (`/dev`, `/proc`, `/sys`, `/run`) live in memory, not on disk — they must be bind-mounted into the new root before chrooting:

```sh
mount --bind /dev      $LFS/dev
mount -t devpts devpts $LFS/dev/pts -o gid=5,mode=0620
mount -t proc   proc   $LFS/proc
mount -t sysfs  sysfs  $LFS/sys
mount -t tmpfs  tmpfs  $LFS/run

chroot "$LFS" /usr/bin/env -i \
    HOME=/root TERM="$TERM" \
    PS1='(lfs chroot) \u:\w\$ ' \
    PATH=/usr/bin:/usr/sbin /bin/bash --login
```

`env -i` is intentional — strip every host environment variable so nothing leaks into the build.

---

## Mandatory part — completed

- [x] Disk partitioning (`cfdisk`), ext4 format, label `lfs`
- [x] Full LFS 13.0 build, **systemd branch**
- [x] Cross-toolchain (binutils, gcc pass 1, glibc, libstdc++)
- [x] Temporary tools + chroot re-entry
- [x] ~80 final packages built, stripped (`8.83 / 8.84`), cleaned
- [x] Orphan-UID prevention: `chown --from lfs -R root:root $LFS/{usr,var,etc,tools,lib64}`
- [x] `/etc/fstab`, hostname, locale, console, `/etc/inputrc`, `/etc/shells`
- [x] Network — `systemd-networkd` + DHCP via `10-eth-dhcp.network`
- [x] Linux kernel compiled from source (`make menuconfig` → `make` → `make modules_install`)
- [x] GRUB installed to MBR (`grub-install /dev/sda --target=i386-pc`), custom `grub.cfg`
- [x] Bootable distribution — `vmlinuz-<ver>-throbert`

---

## Bonuses — completed

| Bonus | Purpose | Why it matters for the defense |
|---|---|---|
| **Xorg** | Display server | Proves the build is complete enough to run a graphical stack |
| **dwm** (suckless) | Tiling window manager | Configured + recompiled from source — no package manager involved |
| **st** | Simple Terminal (suckless) | Minimal terminal emulator, also built from source |
| **xclock** | Reference X11 client | Smoke-test that Xorg + fonts + event loop work |
| **feh** | Image viewer | Validates image-format libraries (`libpng`, `libjpeg`) |
| **lynx** | Text-mode web browser | End-to-end check of networking + TLS userspace |
| **systemd** | PID 1 (instead of SysVinit) | Modern, parallelized service start, journald, auto-restart on crash |

### dwm / st key bindings

| Action | Shortcut |
|---|---|
| Launch X session | `startx` |
| New terminal (st) | `Alt + Shift + Enter` |
| Close terminal | `Alt + Shift + C` |
| Toggle fullscreen | `Ctrl + Alt + Enter` |
| Switch workspace | `Alt + [1-9]` |
| Move window to workspace | `Alt + Shift + [1-9]` |
| Quit X | `Ctrl + Shift + Q` |

---

## Standards respected

- **POSIX** — libc + standard headers (`<stdio.h>`, `<unistd.h>`, `<pthread.h>`), `fork`/`exec`/`wait`, Bourne-compatible shell, mandatory commands (`ls`, `cp`, `mv`, `grep`, `sed`, `awk`, `find`, `pax`), `PATH` / `HOME` / `LOGNAME` env vars
- **LSB** — ELF executable format, glibc + zlib + PAM, FHS layout
- **FHS** — `/usr` consolidated; `/bin`, `/lib`, `/sbin` are symlinks into `/usr/{bin,lib,sbin}`; `/lib64` for the dynamic linker (path is hard-coded in every dynamically-linked ELF)
- **umask 022** — files `666 - 022 = 644`, dirs `755`

---

## Defense — verification commands

```sh
# Kernel
uname -r
uname -m
cat /proc/version
dmesg | head -1
ls /usr/src/

# Storage
lsblk
lsblk -f
cat /etc/fstab

# udev / devices (eudev, Gentoo fork)
ps aux | grep udevd
udevadm --version
lsmod

# init system
systemctl --version
ps aux | grep systemd

# Bootloader
cat /boot/grub/grub.cfg

# Userland + network
vim --version | head -1
ping -c 3 8.8.8.8
curl -I http://google.com
```

### Canonical install loop (every package in Ch.8 and BLFS)

```sh
wget <url>
tar -xf <tarball>
cd <dir>
./configure --prefix=/usr
make
make install
<binary> --version
```

If `./configure` is missing or fails on modern GCC strictness:

```sh
../configure CFLAGS="-Wno-implicit-int -Wno-implicit-function-declaration -O2"
```

---

## Build notes

- **Disk layout** — single `/dev/sda` partitioned in place: `[ LFS ][ swap ][ Debian ]`. Debian is deleted at the end; `resize2fs` extends LFS rightward (so LFS must sit on the left).
- **Kernel ≥ 5.4** — older kernels force glibc to ship compatibility shims.
- **No swap on SSDs** — page-out churn destroys flash cells.
- **SBU** (Standard Build Unit) — `binutils-pass1` build time = 1 SBU; every other package time is given relative to that.
- **Sticky bit on `/sources`** — `chmod a+wt`: both `root` and `lfs` write here during the build; sticky prevents either user from deleting the other’s files.
- **`/lib` vs `/usr/lib`** — at early boot only `/` is mounted; `/usr` is mounted later via `fstab`. Anything needed before `/usr` is reachable (init, bash, ls + their `.so`) must live under `/lib`.
- **Mirror EU** (fallback if upstream is unreachable): `https://mirror.easyname.at/gnu/`

### Boot chain — what runs at power-on

```
Power button
  └─ CPU jumps to firmware (BIOS / UEFI) at a fixed reset vector
       └─ Firmware loads GRUB stage 1 from MBR (first 512 bytes of sda)
            └─ GRUB stage 2 reads /boot/grub/grub.cfg, shows menu
                 └─ Loads vmlinuz (compressed kernel) + initramfs
                      └─ Kernel decompresses, sets up GDT / paging
                           └─ Kernel execs PID 1 → systemd
                                └─ systemd brings up services (network, tty, sshd…)
                                     └─ login prompt
```

---

## References

- [Linux From Scratch (stable)](https://www.linuxfromscratch.org/lfs/view/stable/)
- [LFS 13.0 systemd — wget-list](https://www.linuxfromscratch.org/lfs/downloads/13.0-systemd/wget-list)
- [Beyond Linux From Scratch (BLFS)](https://www.linuxfromscratch.org/blfs/view/svn/index.html)
- [Host requirements check script](https://www.linuxfromscratch.org/lfs/view/stable/chapter02/hostreqs.html)
- [suckless.org](https://suckless.org/) — `dwm`, `st`

---

<div align="center">

*Built by **thomasrbm** — 42 School*

</div>

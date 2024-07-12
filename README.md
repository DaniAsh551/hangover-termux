![stars](https://img.shields.io/github/stars/AndreRH/hangover?style=flat-square)
![forks](https://img.shields.io/github/forks/AndreRH/hangover?style=flat-square)
![Release](https://img.shields.io/github/v/release/AndreRH/hangover?style=flat-square)

Make sure to leave a :star: :)

## Hangover
This is Hangover, a project started by André Zwing and Stefan Dösinger in 2016 that currently can
run x86_32 Windows applications on aarch64 Wine.

### How it works
Hangover uses various emulators as DLLs (pick one that suits your needs, e.g. works for you) to only emulate the application you want to run instead of emulating a complete Wine installation.

As soon as the application does a Windows/Wine system call, say NtUserCreateWindowEx, it's executed outside the emulator (read non-emulated, fast, native). Even better, everything Unix related is never emulated.

In short, we break out of emulation at the win32 syscall or wine unix call level for performance reasons, which is enabled by the WoW64 support in Wine.

### Status
While the overall stability was improved, expect issues.

For Benchmarks see [here](benchmarks/readme.md). They show that the Hangover approach works as expected, as only emulating the application instead of a complete Wine installation has benefits. It's especially visible with box64cpu vs. Wine running under Box64.

Current main focus is to run i386 Windows applications on ARM64 Linux, but it's also possible to run ARM32 Windows applications on x86_64 Linux.
I also started working on a branch for [RISC-V 64-bit Linux](https://github.com/AndreRH/hangover/tree/riscv64).

PPC64le isn't supported anymore and won't be added back in the near future.
Same for running x86_64 applications, though it might be added back as soon as the ARM64EC support in Wine is ready.
If you need those features, have a look at older releases before 0.8.x.

Emulator integrations:

- [QEMU](https://gitlab.com/qemu-project/qemu): Has the most issues and is by far the [slowest](https://github.com/AndreRH/hangover/tree/master/benchmarks) option
- [FEX](https://github.com/FEX-Emu/FEX): Upstream PE version plus some conveniences
- [Box64](https://github.com/ptitSeb/box64/): Mostly done, but depends on the early 32-bit emulation of Box64
- [Blink](https://github.com/jart/blink): started, not part of this repository yet

### Discord
A Discord Server is available for contributors and previous financial supporters (see "Financial Contributiors" below).
It provides advanced user support, development discussions and more.

### Packages
__Debian__ 11 & 12 (also usable for Raspbian, Armbian, ...) and __Ubuntu__ 22.04 & 23.10 & 24.04 are attached to the Github Release.

__Termux__ packages can be found in the [Termux User Repository](https://github.com/termux-user-repository/tur).

__Alpine__ package can be found in the [Alpine Testing Repository](https://gitlab.alpinelinux.org/alpine/aports/-/tree/master/testing/hangover-wine).
It's only hangover-wine without box64cpu.dll for now, but you can copy over box64cpu.dll and/or libwow64fex.dll from extracted debian packages or compile them yourself.

- The dependencies to [build](https://wiki.winehq.org/Building_Wine#Satisfying_Build_Dependencies) a 64 bit Wine
- [llvm-mingw](https://github.com/mstorsjo/llvm-mingw) for PE cross-compilation (download & unpack a release, but don't use the .zip files, they are for Windows)
- [Android Studio](https://developer.android.com/studio/index.html), [Android NDK](https://developer.android.com/ndk/index.html), Gradle
- Libraries and headers from Termux ($PREFIX/lib and $PREFIX/include) to support X11, Pulseaudio, etc.
- About 10GB of disk space

First of all, you need to build Wine Native using the script "build_wine_native.sh". This is required to obtain wine-tools.
Then you can start building Wine Hangover using the "build_wine_termux.sh" script. Before building, the "Fix_build.patch" must be applied to fix errors.

#### 6.2) QEMU (optional)
To build QEMU as a library you need:

- The dependencies to build QEMU (in particular glib)
- About 1GB of disk space

Build it like (from the Hangover repository):
```bash
$ mkdir -p qemu/build
$ cd qemu/build
$ ../configure --disable-werror --target-list=arm-linux-user,i386-linux-user
$ make -j$(nproc)
```

In case the compiler complains about something in linux-user/ioctls.h remove the corresponding line and run make again.

Place resulting libraries (build/libqemu-arm.so and/or build/libqemu-i386.so) in your library path (e.g /usr/lib) or set HOLIB to the full path of the resulting library.

Depreciation note: Placing the libraries under /opt will still work, but is deprecated. Until it is removed the load order is HOLIB, library path, /opt.

#### 6.3) FEX, Unix (optional)
To build FEXCore from FEX you need:

- The dependencies to [build](https://wiki.fex-emu.com/index.php/Development:Setting_up_FEX) FEX (in particular clang, libepoxy and libsdl2)
- About 1.4GB of disk space

Build it like (from the Hangover repository):
```bash
$ mkdir -p fex/build_unix
$ cd fex/build_unix
$ CC=clang CXX=clang++ cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo -DENABLE_LTO=True -DBUILD_TESTS=False -DENABLE_ASSERTIONS=False ..
$ make -j$(nproc) FEXCore_shared
```

Place resulting library (build_unix/FEXCore/Source/libFEXCore.so) in your library path (e.g /usr/lib) or set HOLIB to the full path of the resulting library.

Depreciation note: Placing the libraries under /opt will still work, but is deprecated. Until it is removed the load order is HOLIB, library path, /opt.

#### 6.4) FEX, PE (optional)
To build wow64fex from FEX you need:

- [llvm-mingw](https://github.com/mstorsjo/llvm-mingw) for PE cross-compilation (downlaod & unpack a release, but don't use the .zip files, they are for Windows)
- About 1.5GB of disk space

Build it like (from the Hangover repository):
```bash
$ mkdir -p fex/build_pe
$ cd fex/build_pe
$ export PATH=/path/to/llvm-mingw/bin:$PATH
$ cmake -DCMAKE_TOOLCHAIN_FILE=../toolchain_mingw.cmake -DENABLE_JEMALLOC=0 -DENABLE_JEMALLOC_GLIBC_ALLOC=0 -DMINGW_TRIPLE=aarch64-w64-mingw32 -DCMAKE_BUILD_TYPE=RelWithDebInfo -DBUILD_TESTS=False -DENABLE_ASSERTIONS=False ..

$ make -j$(nproc) wow64fex
```

Place resulting library (build_pe/Bin/libwow64fex.dll) in your wine prefix under drive_c/windows/system32/.

### 7) Running in Termux

Before starting you need to install dependencies in Termux:
```bash
$ pkg update -y && pkg upgrade -y 
$ pkg install -y x11-repo tur-repo 
$ pkg install -y freetype git gnutls libandroid-shmem-static libx11 xorgproto libdrm libpixman libxfixes libjpeg-turbo mesa-demos osmesa pulseaudio termux-x11-nightly vulkan-tools xtrans libxxf86vm xorg-xrandr xorg-font-util xorg-util-macros libxfont2 libxkbfile libpciaccess xcb-util-renderutil xcb-util-image xcb-util-keysyms xcb-util-wm xorg-xkbcomp xkeyboard-config libxdamage libxinerama libxshmfence 
```

You can add the following environment variables:

* HODLL to select the emulator dll:
    * wow64cpu.dll for "native" i386 mode on x86_64
    * wowarmhw.dll for ARM emulation (Qemu)
    * xtajit.dll for i386 emulation (Qemu)
    * libwow64fex.dll for i386 emulation (FEX)
    * box64cpu.dll for i386 emulation (Box64)
* HOLIB to set full path of the library, e.g. HOLIB=/path/to/libqemu-i386.so
* QEMU_LOG to set QEMU log channels, find some options [here.](https://github.com/AndreRH/qemu/blob/v5.2.0/util/log.c#L297)

#### Box64
box64cpu.dll currently is the default for i386 emulation, so it's simply:

```bash
$ wine your_x86_application.exe
```

If you have issues with the default, please try one of the other emulators below.

#### QEMU
Until the critical section issue is solved it is highly recomended to limit execution to 1 core with
"taskset -c 1" for Qemu emulation:

```bash
$ HODLL=xtajit.dll   taskset -c 1 wine your_x86_application.exe
$ HODLL=wowarmhw.dll taskset -c 1 wine your_arm_application.exe
```

#### FEX
```bash
$ HODLL=libwow64fex.dll wine your_x86_application.exe
```

### Known issues

* QEMU: CriticalSection doesn't work reliably and other instabilities
* FEX: Doesn't support CLI applications, as it can't handle writing to the console

### Financial Contributors

I have decided to end my activities on Patreon and other platforms.
It won't be the end of the project, my plan is to keep working on it,
delivering new releases and updates. However, I will probably invest less time,
except for the RISC-V port.

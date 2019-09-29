# Building emulator for Windows on Linux

The following instructions are for building the Commander X16 8-bit computer emulator for Windows using Linux. This was tested using Ubuntu Windows Subsystem for Linux along with a Ubuntu installed on a computer. The following steps are base on Ubuntu 18.

## Prepare build environment

Make sure Linux is update to date

``` shell
sudo apt-get update
sudo apt-get upgrade
```

Install development tools and zip (zip is used for packaging).

``` shell
sudo apt-get install build-essential
sudo apt-get install zip
```

Install [mingw-w64](http://mingw-w64.org/doku.php)

``` shell
sudo apt-get install mingw-w64
```

The Commander X16 emulator requires [SDL2](https://www.libsdl.org/). The following steps will install SDL2.

``` shell
wget https://www.libsdl.org/release/SDL2-devel-2.0.10-mingw.tar.gz
tar -xf SDL2-devel-2.0.10-mingw.tar.gz
cd SDL2-2.0.10
sudo make native
cd ..
```

To build the ROMs for Commander X16 emulator requires the [cc65 compiler](https://cc65.github.io/)

``` shell
git clone https://github.com/cc65/cc65.git
cd cc65
make bin
sudo make avail
cd ..
```

(optional) To package the build [pandoc](https://pandoc.org/) is required. For Ubuntu `pandoc` will need to be downloaded directly from the developer. For other linux distros confirm the `pandoc` package is 2.0 version.

``` shell
wget https://github.com/jgm/pandoc/releases/download/2.7.3/pandoc-2.7.3-1-amd64.deb
sudo dpkg -i pandoc-2.7.3-1-amd64.deb
```

## Building Emulator

Create a directory to organize the pieces of the Commander X16 emulator.

``` shell
mkdir commander-x16
cd commander-x16
```

Clone the ROMs from Commander X16 repository.

``` shell
git clone https://github.com/commanderx16/x16-rom.git
```

Clone the Commander x16 emulator. The instructions below clones the source from this repository. This might be outdated. If cloning from the Commander X16 repository the Makefile and main.c file will need to by patched, see below.

``` shell
git clone https://github.com/y2k4life/x16-emulator.git
```

(optional) To package the build the Commander X16 documentation is required.

``` shell
git clone https://github.com/commanderx16/x16-docs.git
```

To build the emulator two environment variables need to be set

``` shell
export CROSS_COMPILE_WINDOWS=1
export CROSS_COMPILE_WITH_LINUX=1
```

For sound using the YM2151 set the following environment variable

``` shell
export WITH_YM2151=1
```

To build the ROMs

``` shell
cd x16-rom
make
cd ..
```

The following command will build the emulator. This will create `x16emu.exe` which can then be copied to a Windows computer e.g Using WSL `cp x16emu.exe /mnt/d/x16emu.exe`.

``` shell
cd x16-emulator
make
cd ..
```

To package the emulator. This will create the x16emu_win.zip which can be copied to the host Windows computer e.g `cp x16emu.exe /mnt/d/x16emu_win.zip`.

``` shell
cd x16-emulator
make package_win
cd ..
```

## Changes made to original repository

### Compile error

How to fix the following error when compiling.

``` output
main.c: In function ‘SDL_main’:
main.c:457:13: error: implicit declaration of function ‘tolower’
[-Werror=implicit-function-declaration]
     switch (tolower(*p)) {
             ^~~~~~~
cc1: all warnings being treated as errors
Makefile:59: recipe for target 'main.o' failed
make: *** [main.o] Error 1
```

Run the following command:

``` shell
cd x16-emulator
sed -i '/#include <limits.h>/a #include <ctype.h>' main.c
cd ..
```

### Makefile

A new environment variable is created `CROSS_COMPILE_WITH_LINUX` to indicate the building process is using Linux. This along with `CROSS_COMPILE_WINDOWS` will indicate building the Windows version on a Linux system and not building on a Mac.

Change the location of where Ming32 and SDL2 are located.

``` Makefile
ifeq ($(CROSS_COMPILE_WITH_LINUX),1)
        MINGW32=/usr/i686-w64-mingw32
        WIN_SDL2=/usr
else
        # the mingw32 path on macOS installed through homebrew
        MINGW32=/usr/local/Cellar/mingw-w64/6.0.0_2/toolchain-i686/i686-w64-mingw32
        # the Windows SDL2 path on macOS installed through ./configure --prefix=... && make && make install
        WIN_SDL2=~/tmp/sdl2-win32
endif
```

Change the output file name to include `.exe`.

``` Makefile
ifeq ($(CROSS_COMPILE_WINDOWS),1)
        OUTPUT=x16emu.exe
else
        OUTPUT=x16emu
endif
```

Comment command that copies files that don't exist. The processes still works without these files

``` Makefile
package_win:
        (cd ../x16-rom/; make clean all)
        CROSS_COMPILE_WINDOWS=1 make clean all
        rm -rf $(TMPDIR_NAME) x16emu_win.zip
        mkdir $(TMPDIR_NAME)
        cp x16emu.exe $(TMPDIR_NAME)
#       cp $(MINGW32)/lib/libgcc_s_sjlj-1.dll $(TMPDIR_NAME)/
#       cp $(MINGW32)/bin/libwinpthread-1.dll $(TMPDIR_NAME)/
        cp $(WIN_SDL2)/bin/SDL2.dll $(TMPDIR_NAME)/
        $(call add_extra_files_to_package)
        (cd $(TMPDIR_NAME)/; zip -r "../x16emu_win.zip" *)
```

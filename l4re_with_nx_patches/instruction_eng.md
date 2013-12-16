# building L4Re and L4Linux with NX memory support

## getting the compiler
L4Re for ARM does not work when compiled with a hard-float compiler. Besides, during the linking stage L4Re needs the crtendS.o and some other CRT (C Runtime) files from the GCC. Therefore, one needs to use the "linux" toolchain as opposed to a "bare-metal" one. We have used the Sourcery Lite toolchain from Mentor

https://sourcery.mentor.com/GNUToolchain/package11447/public/arm-none-linux-gnueabi/arm-2013.05-24-arm-none-linux-gnueabi-i686-pc-linux-gnu.tar.bz2

## fixing L4Re makefile
By default L4Re does not provide a way to override the toolchain used to compile it. Therefore, you need to manually edit the file **L4Reap/l4/mk/Makeconf** and replace the line containing the `SYSTEM_TARGET_arm` variable with the following.
```
./mk/Makeconf:SYSTEM_TARGET_arm     = arm-none-linux-gnueabi-
```

## preparing build directory
Go to the directory where you want to do put the source code and the build directories for fiasco, L4Re and L4Linux.

> **Note:** In the course of this manual, we assume that the directory is `/mnt/disk2/l4_snapshot`. 

Get source code from:
https://github.com/Ksys-labs/L4Reap
htpps://github.com/Ksys-labs/l4linux

```bash
git clone https://github.com/Ksys-labs/L4Reap.git
git clone htpps://github.com/Ksys-labs/l4linux.git
```

Check out the branch with the support for the NX memory:
```bash
git checkout test-nx
```

Export the environment variables pointing to the build directories
```bash
export FOC_ROOT="$PWD"
export FOC_PATH="${FOC_ROOT}/L4Reap"
export FOC_BUILDDIR="${FOC_ROOT}/build"
export L4RE_BUILDDIR="${FOC_ROOT}/build_l4re"
export L4LINUX_BUILDDIR="${FOC_ROOT}/build_l4linux"
export MAKE_OPTS="-j10"
```
Prepare the build directories
```bash
# for fiasco
pushd .
	cd "${FOC_PATH}/kernel/fiasco/"
	make BUILDDIR="$FOC_BUILDDIR"
popd

# for L4Re
pushd .
	cd "${FOC_PATH}/l4"
	make B="$L4RE_BUILDDIR"
popd

```

## building L4Re
### building fiasco
Go to the fiasco build directory, execute `make config` and set up architecture options. Sample configs for ARM and AMD64 are provided *TODO*.
```bash
pushd .
	cd "$FOC_BUILDDIR"
	make config
	make "$MAKE_OPTS"
popd

```

Now, we can configure the L4Re.
```bash
pushd .
	cd "${FOC_PATH}/l4"
	make O="$L4RE_BUILDDIR" config
popd

```
### creating Makeconf.boot
It is inconvenient to export the shell variables and PATH each time. To avoid doing so, we can put the needed variables and QEMU arguments int a config file. It must be located at `conf/Makeconf.boot` in the L4Re build directory. We can copy the one from the L4Re examples.

```bash
pushd .
	cd "$L4RE_BUILDDIR"
	mkdir conf
	cp ../l4/conf/Makeconf.boot.example conf/Makeconf.boot
popd
```

When building a custom module, we may want to add the directory with the configuration scripts to the `MODULE_SEARCH_PATH` variable. 

```make
MODULE_SEARCH_PATH += /mnt/disk2/l4_snapshot/build/:/mnt/disk2/l4_snapshot/L4Reap/l4/pkg/ksys_mem_tests/conf:/mnt/disk2/l4_snapshot/tudos/l4/pkg/io/config
qemu: MODULE_SEARCH_PATH += /mnt/disk2/l4_snapshot/build/:/mnt/disk2/l4_snapshot/L4Reap/l4/pkg/ksys_mem_tests/conf:/mnt/disk2/l4_snapshot/tudos/l4/pkg/io/config
```

Now, you can either build all packages inside the L4Re build dir by just issuing the `make` command or build only the needed packages. L4Re is known to sometimes fail to build due to package dependencies when doing parallel make. Issue the `make` command until it succeeds. Alternatively, you can build only the needed packages using the `S=` parameter. Here is a command to build everything you need to run the `map_rwx` test application from the `ksys_mem_tests` package:

```bash
pushd .
	cd "$L4RE_BUILDDIR"
	make "$MAKE_OPTS" S=libirq,input,libvbus,libio,libio-io,drivers,drivers-frst,bootstrap,libsupc++,libgcc-pure,libgcc,crtn,libkproxy,libsigma0,libsupc++-minimal,uclibc-minimal,lua,ned,ksys_mem_tests,log,libc_backends,cxx,cxx_libc_io,l4re,l4re_c,l4re_kernel,l4re_vfs,l4sys,l4util,ldscripts,ldso,libloader,loader,uclibc,moe
popd
```

### creating the u-boot image for the target device
```bash
	make MODULES_LIST=/mnt/disk2/l4_snapshot/L4Reap/l4/pkg/ksys_mem_tests/conf/modules.list uimage E=map_rwx

	# copy the uimage to the device or the TFTP server or whatever
	cp images/bootstrap.uimage /srv/tftp/panda/uImage
```

## building l4linux
You will need to additionally build the following L4Re packages:

> io,libstdc++-v3,libvcpu,shmc

to compile l4linux
```bash
# create the l4linux build directory
pushd .
	cd "$FOC_ROOT/l4linux"
	make O="$L4LINUX_BUILDDIR" arm_defconfig
popd

# actually compile
pushd .
	cd "$L4LINUX_BUILDDIR"
	#you can configure the kernel now
	make menuconfig

	#edit the config and set up the path to the fiasco build directory
	echo CONFIG_L4_OBJ_TREE=\\"$L4_BUILDDIR\\" >> $L4BUI/l4reap/.config

	make O=$PWD/../build_l4re L4ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabi- -j10
	mkdir -p "$L4RE_BUILDDIR/bin/arm_armv7a"
	cp vmlinuz.arm "$L4RE_BUILDDIR/bin/arm_armv7a/l4f/vmlinuz.arm" 
popd
```

### building initial RAMdisk
You may get a prebuilt ramdisk or create one using buildroot.
Get the prebuilt image from:

```bash
pushd .
	cd "$L4RE_BUILDDIR"
	wget http://genode.org/files/release-11.11/l4lx/initrd-arm.gz -O bin/arm_armv7a/ramdisk-arm.rd
popd
```

You may want to customize the contents of the ramdisk. To do so, you need to do the following:
```bash
#extract the ramdisk
cpio -i /path/to/ramdisk.img

#make a new image
find . | cpio -o -H newc > /path/to/new_ramdisk.img

```

## building PaX tests for L4Linux
```
	tar xvf pax_tests_l4linux.tar.gz
	cd pax_tests_l4linux
	make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabi-
```

Done, the binaries are in the pax_tests_l4linux directory.

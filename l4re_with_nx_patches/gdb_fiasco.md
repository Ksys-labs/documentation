# using GDB to debug L4Re and L4Linux
GDB in conjunction with qemu can be used to debug L4Re applications and L4Linux

## configuring QEMU
First, configure the qemu to automatically start the GDB server. To do this, add the “-s” switch to the parameters. When building L4Re, you can use the Makeconf.boot file to set up qemu. It must be located at `conf/Makeconf.boot` in the L4Re build directory. We can copy the one from the L4Re examples.

You need to uncomment or add the following line to the Makeconf.boot:
> QEMU_OPTIONS += -s

## connecting the debugger
Next, launch the gdb. If your host machine has the architecture different to the one being debugged, set the correct architecture via “set architecture i386:x86_64”. Also, I had to install “gdb-multiarch” on my i386 debian system to be able to debug an amd64 kernel.

> A good frontend for the GDB is CGDB (which uses ncurses) which displays both the command prompt and the source code.

Now, when qemu is started, we can connect to it from gdb:
target remote tcp::1234

## loading symbols
To allow gdb to display functions and variable names, we need to add the debugging symbols. To know the addresses for them, use objdump and look for the address of the first **LOAD** section. Luckily for us, by default each module is loaded to a different address, and physical addresses match virtual ones, so we can even debug across context switches.

```
add-symbol-file /mnt/disk2/work/l4linux/2git/git/build/fiasco.image 0
add-symbol-file /mnt/disk2/work/l4linux/2git/git/build_l4linux/vmlinux 0x400000
```

## using GDB
Now, you can debug L4Re binaries and L4Linux as ordinary executables. For example, set up a breakpoint at some function and continue execution when it is triggered.

```
break __schedule

cont
```

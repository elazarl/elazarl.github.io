# How does OSv boot?

What happens after the computer power is on? How do you
write the very first lines of code the CPU would execute
right after the computer is loaded?

Here, again, OSv is supplying a nice, self contained answer.
Let's limit our answer to the `x86_64` architecture, also OSv currently
supports ARM as well.

What happens right after an CPU is loaded? The CPU instruction pointer is
initialized to its _reset vector_. The x86 CPU sets their `CS:EIP` address
to the fixed address `0xF000:0xFFF0`, this physical address contains some
code leading you eventually to the BIOS.
See [coreboot](http://www.coreboot.org/Coreboot_v3#How_coreboot_starts_after_Reset)'s
documentation for additional information.

Let's see that in action

    $ # Start paused (-S) QEMU with debugger (-s)
    $ qemu-system-x86_64 -s -S -nographic
    $ # on another tab
    $ gdb
    ...
    (gdb) target remote localhost:1234
    Remote debugging using localhost:1234
    0x0000fff0 in ?? ()
    (gdb) i r eip
    eip            0xfff0	0xfff0
    (gdb) i r cs
    cs             0xf000	61440
    (gdb) 

As we start, the CPU start on `0xf000:0xfff0`. To see what the CPU is about to execute,
we'll have to translate the segmented address `0xf000:0xfff0` to a
linear address. Since we start in real mode, we simply have to multiply the segment
address by `0x10`, and add it to the IP, see [Wikipedia](http://wiki.osdev.org/Segmentation#Real_mode)
for additional details.

The BIOS then loads the MBR from the disk at address `0x7c00`, and executes
it.

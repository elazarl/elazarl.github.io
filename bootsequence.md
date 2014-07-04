# How does OSv boot?

What happens after the computer power is on? How do you
write the very first lines of code the CPU would execute
right after the computer is loaded?

Here, again, OSv is supplying a nice, self contained answer.
Let's limit our answer to the `x86_64` architecture, even though OSv
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
    (gdb) set architecture i8086 
    warning: A handler for the OS ABI "GNU/Linux" is not built into this configuration
    of GDB.  Attempting to continue with the default i8086 settings.

    The target architecture is assumed to be i8086
    (gdb) target remote localhost:1234
    Remote debugging using localhost:1234
    0x0000fff0 in ?? ()
    (gdb) info registers eip cs
    eip            0xfff0	0xfff0
    cs             0xf000	61440
    (gdb) 

As we start, the CPU start on `0xf000:0xfff0`. To see what the CPU is about to execute,
we'll have to translate the segmented address `0xf000:0xfff0` to a
linear address. Since we start in real mode, we simply have to multiply the segment
address by `0x10`, and add it to the IP, see [Wikipedia](http://wiki.osdev.org/Segmentation#Real_mode)
for additional details. Let's print the first instruction the CPU which should be
in `0x10*0xf000+0xfff0=0xffff0`:

    (gdb) x/i 0xffff0
    0xffff0:	ljmp   $0xf000,$0xe05b

It's probably a jump to the BIOS code at `0xf000:0xe05b`. Let's step to the next instruction

    (gdb) si
    0x0000e05b in ?? ()
    (gdb) i r eip cs
    eip            0xe05b	0xe05b
    cs             0xf000	61440

Indeed, we jumped to `0xf000:0xe05b` as expected.

The BIOS then loads the MBR from the disk at address `0x7c00`, and executes
it. Let's see that in action.

Let's start the virtual machine, and tell it to stop at load time:

    $ ./scripts/run.py -d --wait

On a different terminal, let's connect with gdb, and break at `0x7c00`, the
address the BIOS loads the MBR to. Note that we're specifying commands that `gdb`
should run at startup with the `-ex` switch, so that when `gdb`'s prompt shows
up, it'll already be loaded, and the correct architecture is set:
    
    $ gdb -ex 'set architecture i8086' -ex 'target remote localhost:1234'
    ...
    (gdb) hbr *0x7c00
    Hardware assisted breakpoint 1 at 0x7c00
    (gdb) c
    Continuing.

    Breakpoint 1, 0x00007c00 in ?? ()

Now we finally started to run OSv code, the Master Boot Record, or the MBR.
The MBR is in `build/debug/boot.bin`. Let's verify that this is the case:

    (gdb) x/32b $eip
    0x7c00:	0xea	0x5e	0x7c	0x00	0x00	0x00	0x00	0x00
    0x7c08:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
    0x7c10:	0xdb	0x00	0x10	0x00	0x40	0x00	0x00	0x00
    0x7c18:	0x00	0x80	0x80	0x00	0x00	0x00	0x00	0x00
    $ hexdump -n 32 -e '"0x%04_ax: " 8/1 "0x%02x\t" "\n" ' \
    build/release/boot.bin 
    0x0000: 0xea    0x5e    0x7c    0x00    0x00    0x00    0x00    0x00
    0x0008: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
    0x0010: 0x00    0x10    0x10    0x00    0x40    0x00    0x00    0x00
    0x0018: 0x00    0x80    0x80    0x00    0x00    0x00    0x00    0x00

Looks like indeed we're running `boot.bin`. How is `boot.bin` generated?
The answer is in `build.mk`. First `boot16.S` is compiled to `boot16.o`
with regular compilation, by the `%.o: %.S` rule. Then, `boot.bin`
is being linked from `boot16.o` and `boot16.ld` linker script.

What does `boot16.ld` do? At first it defines a memory section of the
available memory at boot time. The first 1MB.

    MEMORY { BOOTSECT : ORIGIN = 0, LENGTH = 0x10000 }

Then, it'll take all text sections from the input, and relocate them to
`0x7c00`, where the MBR would load it. It would constraint the text
section to the first megabyte of memory we defined previously. That
way, if `boot16.S` accidently surpass 1MB, the linker would complain.

    SECTIONS { .text 0x7c00 : { *(.text) } > BOOTSECT }

For example, if we'll add a megabyte of data at the end of `boot16.S`,
we'll get the following:

    $ echo .fill 0x10000, 1, 0 >> arch/x64/boot16.S
    $ make mode=debug
    ...
    make[1]: Entering directory `/home/elazar/dev/osv/build/debug'
      GEN gen/include/osv/version.h
      AS arch/x64/boot16.o
      LD boot.bin
    ld: address 0x17e00 of boot.bin section `.text' is not within region `BOOTSECT'

Finally, we'll instruct `ld` to output the raw assembly instructions,
without any ELF headers. The BIOS expect the MBR to contain the actual
assembly commands:

    OUTPUT_FORMAT(binary)

What does `boot16.S` do?

At first we can see some x86 bookkeeping. Setting up the
[A20](http://www.win.tue.nl/~aeb/linux/kbd/A20.html) line, and set up the segment
registers.

Then, it'll try to actually load the OSV loader from disk, using
[interrupt 13](http://wiki.osdev.org/ATA_in_x86_RealMode_%28BIOS%29#LBA_in_Extended_Mode)

    int1342_boot_struct:
    .byte 0x10 # size of packet (16 bytes)
    .byte 0 # should always be 0
    .short 0x3f   # fetch 0x3f sectors = 31.5k
    .short cmdline # fetch to address $cmdline
    .short 0 # fetch to segment 0
    .quad 1 # start at LBA 1.
    # That is, fetch the first 31.5k from the disk
    ...
    lea int1342_boot_struct, %si
    mov $0x42, %ah
    mov $0x80, %dl
    int $0x13

Indeed after the interrupt, we can see the system is loaded.

# How does OSv boot?

What happens after the computer power is on? How do you
write the very first lines of code the CPU would execute
right after the computer is loaded?

Here, again, OSv is supplying a nice, self contained answer.
Let's limit our answer to the `x86_64` architecture, also OSv currently
supports ARM as well.

What happens right after an CPU is loaded? The CPU sets the `EIP` address
to the fixed address `0xF000:0xFFF0`, this physical address contains some
code leading you eventually to the BIOS. See [coreboot](http://www.coreboot.org/Coreboot_v3#How_coreboot_starts_after_Reset)'s documentation.

The BIOS then loads the MBR from the disk at address `0x7c00`, and executes
it.

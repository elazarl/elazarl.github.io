# OSv loader

OSv loader is composed of two parts. The LZ loader, which simply uncompresses the "real"
loader. And the actual loader who starts up OSv.

The first part of the loader of OSv is pretty simple, a C [LZ77 implementation](http://fastlz.org/faq.htm)
that uncompress the loader. It is compiled to a standard ELF file, and is called by `boot16.S`. The
`lzloader.ld` linker script specify `uncompress_loader` to be `lzloader.elf`'s entry point, and
`boot16.S` calls the function at the address of `lzloader.elf start + entry_offset`.
The function itself is fairly simple:

```C
fastlz_decompress(&_binary_loader_stripped_elf_lz_start,
        (size_t) &_binary_loader_stripped_elf_lz_size,
        BUFFER_OUT/*(char *)0x200000*/, MAX_BUFFER);
```

We decompress the stream at `_binary_loader_stripped_elf_lz_start` to the hardcoded address
`0x200000`. Obviously, `_binary_loader_stripped_elf_lz_start` contains the compressed loader,
but where does it come from? The `objcopy` utility comes to the rescue,
enabling us package a file inside an object file. From the manual:

> You can access this binary data inside a program by
> referencing the special symbols that are created by the conversion
> process. These symbols are called _binary_objfile_start,
> _binary_objfile_end and _binary_objfile_size.  e.g. you can
> transform a picture file into an object file and then access it in
> your code using these symbols.

For example

```
$ echo hello world > hello-1.txt
$ cat ->main.c
#include <stdio.h>
extern char _binary_hello_1_txt_start;
extern char _binary_hello_1_txt_end;
extern char _binary_hello_1_txt_size;
int main(int argc, char** argv) {
    size_t sz = (size_t)&_binary_hello_1_txt_size;
    (&_binary_hello_1_txt_start)[sz-1] = '\0';
    printf("packaged %s\n", &_binary_hello_1_txt_start);
    return 0;
}
$ #       out ELF arch   input format   output format
$ objcopy -B i386:x86-64 -I binary    -O elf64-x86-64 hello-1.txt hello-1.o
$ gcc main.c hello-1.o
$ ./a.out
packaged hello world
```

Indeed the makefile includes:

```
loader-stripped.elf.lz.o: loader-stripped.elf fastlz/lz
    # compress the loader's code with LZ
	$(call quiet, fastlz/lz loader-stripped.elf, LZ $@)
    # convert to object file, exporting _binary_loader_stripped_elf_lz_{start,end,size}
	$(call quiet, objcopy -B i386 -I binary -O elf32-i386 loader-stripped.elf.lz $@, OBJCOPY $@)
```

After lzloader decompress the loader, `boot16.S` calls the ELF file that now appears in `0x200000`
by calling the address in its entry header. This time, we're running a full fledged C++ program.

But wait. One cannot just issue `call main` and expect their C++ program to start.
Usually the OS sets a proper environment for your `C++` program, loads different sections to different
location in the virtual memory. Zeros the `BSS` section, sets the `argv` and `argc` command line variables
etc. Who does all those things? In OSv _you_ are the OS.

The entry point of `loader.elf` is `start32` in `boot.S`. Let's remember that `boot16.S` left the CPU
in 32 bit mode before calling `lzloader.elf` and then `loader.elf`, so before we start we have some
bookkeeping to make:

Set GDT to flat segments for 64 and 32 bits, initialize segment registers to this flat segment
and start using the new GDT.

```
gdt = . - 8
    //    base flag limit type  base    limit
    .quad 0x00 a     f    9b    000000  ffff # 64-bit code segment
    // first descriptor special to enable long mode, see
    // http://wiki.osdev.org/X86-64#How_do_I_enable_Long_Mode_.3F
    .quad 0x00 c     f    93    000000  ffff # 64-bit data segment
    .quad 0x00 c     f    9b    000000  ffff # 32-bit code segment
gdt_end = .
lgdt gdt_desc
mov $0x10, %eax
mov %eax, %ebp
mov $0x10, %eax
mov %eax, %ds
mov %eax, %es
mov %eax, %fs
mov %eax, %gs
mov %eax, %ss
// first, we enable 32-bit mode on the new GDT table
ljmp $0x18, $1f
```

Now, we need to [enable 64 bit mode](http://wiki.osdev.org/X86-64#How_do_I_enable_Long_Mode_.3F)

  1. Disable paging - they were never enabled.
  2. Set the PAE enable bit on CR4

```
mov $BOOT_CR4, %eax
mov %eax, %cr4
```

  3. Load CR3 with the physical address of [Page Map Level 4](http://www.pagetable.com/?p=14)

```
lea ident_pt_l4, %eax
mov %eax, %cr3
```

  4. Enable long mode by setting the EFER.LME flag in MSR `0xC0000080`

```
mov $0xc0000080, %ecx
mov $0x00000900, %eax
xor %edx, %edx
wrmsr
```

  5. Enable paging, by setting the relevant bit in `CR0`

```
mov $BOOT_CR0, %eax
mov %eax, %cr0
```

  6. Now the CPU will be in compatibility mode, jump to the special 64-bit GDT code descriptor,
     and we're in 64 bit mode.

```
ljmpl $8, $start64
```

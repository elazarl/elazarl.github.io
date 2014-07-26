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
and start using the new GDT:

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
// set all segment registers to 0x10, the second segment
...
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

Some more bookkeeping to do when in 64 bit mode:

  1. Zero the BSS section:
    ```
    lea .bss, %rdi
    lea .edata, %rcx
    sub %rdi, %rcx
    xor %eax, %eax
    rep stosb
    ```

  2. Init global variables with information from `boot16.S`. Note that
     `boot16.S` keeps the ELF header and multiboot addresses in `%ecx`
     and `ebx` respectively. `boot.S` saves `%ebx` to `ebp`.
    ```
    mov %rbp, elf_header
    mov %rbx, osv_multiboot_info
    ```

  3. Set the stack pointer. At first, for `main`, it is simply set
     to an empty `16K` region in the image, remember that in x64 stack
     grows down:
    ```
    .align 16
    . = . + 4096*4
    init_stack_top = .
    lea init_stack_top, %rsp
    ```
  4. Call the `premain` function.
    ```
    call premain
    ```

Now we're in `premain`, and it _looks_ like we're running a regular C++ program.
There are however a couple of differences:

  1. Global variables are not initialized, since the
     [`.init` section](http://refspecs.linuxbase.org/LSB_3.1.1/LSB-Core-generic/LSB-Core-generic/specialsections.html)
     in `loader.elf` yet.
  2. The stack is limited to `16K`.
  3. We haven't initialized the other CPUs, so we're running on the
     [BSP CPU](http://stackoverflow.com/questions/14261612/which-core-initializes-first-when-a-system-boots)
     that started the system.

What does `premain` do?

  1. Init terminal

    ```C++
    arch_init_early_console();
    ```

  2. We need to use the newer APIC interrupt controller
     Hence we disable [PIC](http://wiki.osdev.org/8259_PIC)

    ```C++
    disable_pic();
    auto inittab = elf::get_init(elf_header);
    ```

  3. Setup thread local storage. This is an interesting topic worthy of a discussion alone.

    ```
    setup_tls(inittab);
    ```

  4. Run .init functions from loader.elf allowing us to use
     global variables.

    ```C++
    for (auto init = inittab.start; init < inittab.start + inittab.count; ++init) {
        (*init)();
    }
    ```

We are now ready to enter `main` and start running the actual loader.
Except of one last thing - initializing SMP.

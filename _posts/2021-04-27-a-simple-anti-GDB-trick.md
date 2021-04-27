After reading [Alex O'Mara's](https://alexomara.com/blog/a-silly-anti-disassembly-trick/) post about preventing disassembly with Hopper by including the Hopper self-protection signature in the binary, I remembered this trick I discovered a while back to prevent files from being opened by gdb.

Say we have this basic program:;

    #include <stdio.h>
    int main(void) {
        printf("hello, world\n");
    }

Compile it with:

    $ gcc main.c -o main

It should run ordinarily:

    $ ./main 
    hello, world

And it works in gdb:

    $ gdb ./main 
    GNU gdb (Debian 8.2.1-2+b3) 8.2.1
    ...
    Reading symbols from ./main...(no debugging symbols found)...done.
    (gdb) 

But if we change the [e_machine field](https://refspecs.linuxfoundation.org/elf/gabi4+/ch4.eheader.html) in the ELF header to an invalid value:

    printf '\xFF' | dd of=main bs=1 seek=5 count=1 conv=notrunc

The program still runs:

    $ ./main 
    hello, world

But fails to open in gdb:

    GNU gdb (Debian 8.2.1-2+b3) 8.2.1
    ...
    "/home/christopher/./main": not in executable format: file format not recognized
    (gdb) 

This even prevents objdump from being able to disassemble it.

    $ objdump -d ./main
    objdump: ./main: file format not recognized

The reason this works is because Linux parses the ELF header much more generously than these tools.  This of course is just as easily circumvented as it is implemented but is probably effective in detering "noob" analysts.  Read more about other creative uses of the ELF header at the [ELF Kickers](https://www.muppetlabs.com/~breadbox/software/elfkickers.html) page on muppetlabs.com.

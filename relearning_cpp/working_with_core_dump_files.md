---
layout: default
---
# Working with core dump files

- Enabling and troubleshooting core dump files in Ubuntu
  - Set max core dump file size to unlimited
    - ```
      $ ulimit -c unlimited
      ```
  - Check how core dumps are handled by the kernel
    - Alternative commands
      - ```bash
        $ cat /proc/sys/kernel/core_pattern
        ```
      - ```bash
        $ sysctl kernel.core_pattern
        ```
    - A possible value is like the following
      - ```bash
        | /usr/share/apport/apport %p %s %c
        ```
    - The `apport` service sets the `core_pattern` to the above value
    - Kernel pipes the core dump to the command
    - Even if core dump files are disabled by `ulimit` `apport` still gets the core dump
  - To modify the kernel `core_pattern`
    - ```bash
      $ sysctl -w kernel.core_pattern=core.%u.%p.%t
      ```
      - Percent formatters - UID, PID, timestamp
      - More formatters defined [here](https://man7.org/linux/man-pages/man5/core.5.html)
    - Starting `apport` service sets `core_pattern` to the piped value
    - Stopping `apport` service sets `core_pattern` to “core”
      - This generates a core dump file in current directory
  - Check and modify `apport` service status
    - ```bash
      $ sudo service apport status
      ```
    - ```bash
      $ sudo service apport start
      ```
    - ```bash
      $ sudo service apport stop
      ```
  - Look for core dumps under `apport` coredump folder
    - ```bash
      $ ll /var/lib/apport/coredump
      ```
  - Check for `apport` errors and messages
    - ```bash
      $ cat /var/log/apport.log
      ```
- Working with GDB and core files
  - Start GDB with a core file
    - ```bash
      $ gdb path/to/executable path/to/core
      ```
- Running executable in GDB to observe crash behaviour
  - Start executable in GDB
    - ```bash
      $ gdb path/to/executable
      ```
- Useful GDB commands
  - ```bash
    (gdb) set environment LD_LIBRARY_PATH=./out/
    ```
  - ```bash
    (gdb) break mylib_func
    ```
    - Add breakpoint at function `mylib_func`
  - ```bash
    (gdb) break *0x1234abcd
    ```
    - Add breakpoint at address `0x1234abcd`
  - ```bash
    (gdb) run
    ```
    - Run to next breakpoint 
  - ```bash
    (gdb) step
    ```
    - Step into function
  - ```bash
    (gdb) finish
    ```
    - Run till end of stack frame (current function call)
  - ```bash
    (gdb) up
    ```
    - Step out of function
  - ```bash
    (gdb) next
    ```
    - Run next line
  - ```bash
    (gdb) set disassembly-flavor intel
    ```
  - ```bash
    (gdb) disas mylib_func
    ```
    - disassemble instructions in function `mylib_func`
  - ```bash
    (gdb) disas 0xf7fbb040
    ```
    - disassemble instructions at address
  - ```bash
    (gdb) i registers eax 
    ```
    - inspect register eax
  - ```bash
    (gdb) x 0xF7FBD1B0
    ```
    - display memory content at address
  - ```bash
    (gdb) x/wx 0xF7FBD1CC
    ```
    - display 1 word at address as hex
  - ```bash
    (gdb) x/3i 0xF7FBD1CC
    ```
    - display 3 machine instructions at address
  - ```bash
    (gdb) x/300bx 0x400b40
    ```
    - display 300 bytes in hex starting at address
  - ```bash
    (gdb) p &myglob
    ```
    - print address of variable `myglob`
  - ```bash
    (gdb) info symbol 0x400aa0
    ```
    - print debug info of symbol at address 



### References:

1. [Where do I find core dump files](https://askubuntu.com/questions/1349047/where-do-i-find-core-dump-files-and-how-do-i-view-and-analyze-the-backtrace-st)
1. [Where do I find the core dump in ubuntu 16.04LTS?](https://askubuntu.com/questions/966407/where-do-i-find-the-core-dump-in-ubuntu-16-04lts)
1. [Linux manual page core(5)](https://man7.org/linux/man-pages/man5/core.5.html)
1. [How to Set the Core Dump File Path](https://www.baeldung.com/linux/core-dumps-path-set)
1. [GDB reference card](https://users.ece.utexas.edu/~adnan/gdb-refcard.pdf)

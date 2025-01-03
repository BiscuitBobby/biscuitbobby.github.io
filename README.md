# My microblog to get into kernel debugging
In this blog I will share my experience diving into kernel debugging and some of the essential tools I’ve used along the way. If you’re just starting out, there are some great blogs to get you started with debugging the kernel such as [this one](https://marliere.net/posts/1/) by Javier, which covers the basics of using QEMU, Syzkaller, and GDB for kernel debugging.

The linux kernel is the first major open-source project I've contributed to, and it’s been an incredibly rewarding experience. Although I’ve been a daily Linux user for years, I had little to no prior exposure to kernel development before joining this program. Looking back, I’m glad I invested the time and effort rather than letting the initial complexity intimidate me.

Admittedly, navigating the Linux kernel codebase for the first time can feel overwhelming. However, diving into kernel debugging is an excellent way to start understanding its intricacies and get familiar with it.

---

## Getting started with debugging
First and foremost, make sure `CONFIG_DEBUG_INFO` is enabled so that the kernel is compiled with debugging information. This allows tools like GDB to access symbol information and line numbers.<br>
Next, you will want to set up a debugging environment. I primarily use vng, cscope and a whole lot of printk's...

You might also want to adjust the kernel’s log level to ensure that all debugging messages are displayed in the terminal. You can do this by setting the console_loglevel to 7 or 8, which allows more verbose output. To modify the current console_loglevel, use the following command:
```bash
echo 8 > /proc/sys/kernel/printk
```

### Debugging techniques
I happened to come across a great write-up on kernel debugging techniques by joelagnel [here](https://gist.github.com/joelagnel/ae15c404facee0eb3ebb8aff0e996a68). 
My go to method is usually shot-gun debugging, where I sprinkle a bunch printk()'s around the code to pinpoint the origin of the problem. I use statements like these: 

```
printk("Lemon %s: (%s) (%d)", __func__, __FILE__, __LINE__);
```

This is of course after going through the traceback of a crash to figure out the general location of the issue.

Let us look at the following message:
```
[ 1147.588443][    C0]  #0: ffffffff8c38be20 (rcu_read_lock){....}-{1:2}, at: timerfd_clock_was_set+0x4/0x2e0
```

You can break this down into these basic components:
- **[ 1147.588443]**: Timestamp indicating when the kernel event occurred.
- **[ C0]**: The CPU core number (Core 0) where the panic happened.
- **#0**: The stack frame number. `#0` usually means the current frame.
- **ffffffff8c38be20**: The memory address where the issue occurred.
- **rcu_read_lock**: The lock held at the time of the panic.
- **timerfd_clock_was_set+0x4/0x2e0**: This points to the function and offset within the function where the issue occurred

System.map:
The `System.map` file is a symbol table used by the kernel that maps kernel symbols (like functions) to specific memory addresses.
```
ffffffff81ee7920 T timerfd_clock_was_set
```
- **ffffffff81ee7920**: This is the starting memory address of the function `timerfd_clock_was_set` 
- **T**: This symbol type `T` usually stands for a text (code) symbol, which means it's part of the executable code in the kernel.

Hex value:  
We can find the specific location of where crash occured by adding the offset 0x4 to the starting address 81ee7920
```
81ee7920 + 0x4 = 81EE7924
```

### Using GDB to find the location of kernel panic
We can use the list command in GDB to list the source code around a specific location. GDB will output the lines of source code around this point (I am in pwndbg split screen tui mode):
```
(gdb) list *(timerfd_clock_was_set+0x4)
```
![Pasted image 20240831000553](https://github.com/user-attachments/assets/ef9ce5ac-b11f-4529-9c13-daffc730f677)

I am using pwndbg here, a gdb plug-in that provides some extra functionality, but normal gdb will be more than enough.
### objdump
We can disassemble the kernel binary using the following command:
```
objdump -d vmlinux > vmlinux_disassembly.txt
```

Once we generate the file, we can search for the memory address 81EE7924, which corresponds to the function `timerfd_clock_was_set+0x4/0x2e0` (81ee7920 + 0x4) 
![[Pasted image 20240831013039.png]]
You can analyse the assembly code and its corresponding Kernel C function to identify potential issues, such as incorrect register usage, stack corruption, or unexpected control flow.

# Other tools
## virtme-ng
This tool particularly has saved me a lot of time with a noticable speed up the compilation time. My biggest cause of frustration with debugging the kernel was not fixing issues, but waiting for it to finish compiling after adding a couple debug statement.

Generate a minimal kernel .config in the current kernel directory
```
vng --kconfig
```
Build a kernel from local kernel source directory
```
vng --build
```
Run compiled kernel in a virtualized environment
```
vng
```
The vm is an exact copy-on-write copy of your live system, which means that any changes made to the virtualized environment do not affect the host system

## Cscope
At the kernel root directory, In order to generate a Cscope database that includes all symbols across the directory tree, use the `-R` flag to search recursively we can run:
```
cscope -R
```
This creates a database of all the symbols, functions, macros, and other identifiers within the directory and its subdirectories.

After the database is generated, you can launch the Cscope browser using:
```
cscope -d
```

Presenting us with a list of search options such as this which we can navigate using tab and the arrow keys

![Pasted image 20240831011426](https://github.com/user-attachments/assets/9c6122c1-a690-430b-800f-35cb73d84177)


## kw - Kernel Workflow Tool

**kw** is a tool designed to reduce the overhead of setting up and managing the development environment for Linux kernel contributions. It simplifies kernel debugging by providing interfaces for various tools and scripts.

### Some of my most used Commands

Identify the maintainers for a file or directory:  
```bash
kw m path_to_file_or_dir
```

Finding trivial style violations in your patches:  
```bash
kw c path_to_file_or_dir
```

Submit patches to the kernel mailing list with a single command:  
```bash
kw send-patch --send --to=<recipient> --cc=<recipient>
```
(Dont forget to line wrap - max of 80 char per line in the commit)<br>
Make sure `linux-kernel@vger.kernel.org` and the relevant maintainers are included in the recipient or CC list. While **kw** handles this automatically, it’s good to double-check. Use the `--simulate` parameter to preview the process without sending the email.

### Some useful parameters for `kw send-patch`
- `HEAD~n..` → Include the last **n** commits.  
- `-v n` → Specify patch version **n**.  
- `--private` → Send only to manually specified recipients.  
- `--simulate` → Perform all steps without actually sending emails (similar to Git’s `--dry-run`).  
- `--cover-letter` → Add a cover letter explaining the purpose of the patch series.

You can see the full set of parameters available to you [here](https://kworkflow.org/man/kw.html).

---

I hope you found some useful tips or pointers from my blog, I wish you the very best on your kernel development journy.

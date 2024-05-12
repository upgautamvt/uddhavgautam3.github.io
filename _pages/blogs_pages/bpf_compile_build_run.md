# BPF compile, load, run (Includes libbpf skeleton generation concept)
## <a name="_wozkdroeokc4"></a>What are traceable functions?
We can find all the list of functions in /sys/kernel/debug/tracing/available\_filter\_functions

## <a name="_ml0aq6wgxgbs"></a>What is vmlinux.h?
vmlinux.h is generated code. it contains all of the type definitions that your running Linux kernel uses in it’s own source code. When you build Linux one of the output artifacts is a file called vmlinux (there is vmlinz also, but vmlinux is uncompressed file while vmlinuz is compressed one). It’s also typically packaged with major distributions. This is an [**ELF**](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) binary that contains the compiled kernel inside it. The vmlinux.h contains every type-definition -- as listed below -- that the installed kernel uses, it’s a very large header file.

#### Kernel Data Structures
- `task_struct`: Represents a process or task in the kernel.
- `file`: Represents an open file descriptor.
- `inode`: Represents an inode in the filesystem.
- `super_block`: Represents a filesystem superblock.
- `net_device`: Represents a network interface device.


#### Memory Management Structures
- `page`: Represents a physical page frame.
- `vm_area_struct`: Represents a memory mapping area.
- `mm_struct`: Represents a process's memory map.


#### Device and Driver Structures
- `device`: Represents a hardware device.
- `platform_device`: Represents a device on a platform bus.
- `pci_dev`: Represents a PCI device.

#### Networking Structures
- `sk_buff`: Represents a network packet buffer.
- `net_device`: Represents a network interface device.
- `sock`: Represents a socket.

#### Filesystem Structures
- `inode`: Represents an inode in the filesystem.
- `file_operations`: Represents operations on a file.
- `address_space`: Represents the address space of a file.

#### Kernel Constants and Macros
- Configuration constants controlling various kernel features.
- Macros for handling bit manipulation, memory allocation, and more.
- Constants for error codes, system call numbers, and other kernel interfaces.

#### Other Miscellaneous Structures
- Structures related to timers, interrupts, locks, and synchronization primitives.
- Structures related to virtual memory management, scheduling, and process control.

## <a name="_9jdnk8bhj7pn"></a>What is the use/benefits of using vmlinux.h?
Since the vmlinux.h file is generated from your installed kernel, your bpf program could break if you try to run it on another machine without recompiling if it’s running a different kernel version. This is because, from version to version, definitions of internal structs change within the linux source code. So, the best practice is always recompile your bpf program before running it. It means including vmlinux.h, usually demands, recompiling your bpf program. Then why do you want to include this vmlinux.h at all?

But wait, the CO:RE (Compile Once, Run Everywhere) of libbpf allows you to analyze kernel data structures defined inside vmlinux.h.

analysis output = core\_analyze\_function(vmlinux.h)

Now, for your different kernel version, where you are now running, can automatically adapt based on analysis\_output. This brought portability. The system, especially libbpf system, will adapt based on analysis\_output.

There are several other benefits of using vmlinux.h in your bpf program. Please google yourself.
## <a name="_nxyybsouxnkv"></a>What is ELF binary?
An ELF (Executable and Linkable Format) binary is a common file format used for executables, object code, shared libraries, and core dumps in Unix-like operating systems such as Linux, BSD, and others.

Here's a breakdown of what each component typically contains:

- **Executable Code**: This section contains machine code that the CPU can execute directly.
- **Data**: This section contains initialized data that the program needs during execution. This can include global variables and static variables that have **explicit initialization.**
- **Uninitialized Data (BSS)**: This section contains **uninitialized** data that the program needs during execution. It is typically zero-initialized.
- **Symbol Table**: This table holds information about various symbols used in the program, including function names, global variable names, and so on.
- **Program Header Table**: This table contains information required by the system to map the binary into memory during execution. It includes details like which parts of the binary are executable, which parts are read-only, etc.
- **Section Header Table:** This table contains information about various sections within the binary, such as their names, sizes, and file offsets.

ELF binaries are flexible and versatile, supporting features like position-independent code, shared libraries, and more, making them a fundamental part of Unix-like operating systems' executable and linking infrastructure.
## <a name="_3k9hqm7hn6jv"></a>libbpf skeleton
libbpf is a C-based library containing a BPF loader that takes compiled BPF object files and prepares and loads them into the Linux kernel. libbpf takes the heavy lifting of loading, verifying, and attaching BPF programs to various kernel hooks, allowing BPF application developers to focus only on BPF program correctness and performance.

The following are the high-level features supported by libbpf:

- Provides high-level and low-level APIs for user space programs to interact with BPF programs. The low-level APIs wrap all the bpf system call functionality, which is useful when users need more fine-grained control over the interactions between user space and BPF programs.
- Provides overall support for the BPF object skeleton generated by bpftool. The skeleton file simplifies the process for the user space programs to access global variables and work with BPF programs.
- Provides BPF-side APIS, including BPF helper definitions, BPF maps support, and tracing helpers, allowing developers to simplify BPF code writing.
- Supports BPF CO-RE mechanism, enabling BPF developers to write portable BPF programs that can be compiled once and run across different kernel versions.

A BPF application consists of one or more BPF programs (either cooperating or completely independent), BPF maps, and global variables. The global variables are shared between all BPF programs, which allows them to cooperate on a common set of data. libbpf provides APIs that user space programs can use to manipulate the BPF programs by triggering different phases of a BPF application lifecycle.

The following section provides a brief overview of each phase in the BPF life cycle:

- **Open phase**: In this phase, libbpf parses the BPF object file and discovers BPF maps, BPF programs, and global variables. After a BPF app is opened, user space apps can make additional adjustments (setting BPF program types, if necessary; pre-setting initial values for global variables, etc.) before all the entities are created and loaded.
- **Load phase**: In the load phase, libbpf creates BPF maps, resolves various relocations, and verifies and loads BPF programs into the kernel. At this point, libbpf validates all the parts of a BPF application and loads the BPF program into the kernel, but no BPF program has yet been executed. After the load phase, it’s possible to set up the initial BPF map state without racing with the BPF program code execution.
- **Attachment phase**: In this phase, libbpf attaches BPF programs to various BPF hook points (e.g., tracepoints, kprobes, cgroup hooks, network packet processing pipeline, etc.). During this phase, BPF programs perform useful work such as processing packets, or updating BPF maps and global variables that can be read from user space.
- **Tear down phase**: In the tear down phase, libbpf detaches BPF programs and unloads them from the kernel. BPF maps are destroyed, and all the resources used by the BPF app are freed.

BPF skeleton is an alternative interface to libbpf APIs for working with BPF objects. Skeleton code abstract away generic libbpf APIs to significantly simplify code for manipulating BPF programs from user space. Skeleton code includes a bytecode representation of the BPF object file, simplifying the process of distributing your BPF code. With BPF bytecode embedded, there are no extra files to deploy along with your application binary.

You can generate the skeleton header file (.skel.h) for a specific object file by passing the BPF object to the bpftool. The generated BPF skeleton provides the following custom functions that correspond to the BPF lifecycle, each of them prefixed with the specific object name:

- <name>\_\_open() – creates and opens BPF application (<name> stands for the specific bpf object name)
- <name>\_\_load() – instantiates, loads,and verifies BPF application parts
- <name>\_\_attach() – attaches all auto-attachable BPF programs (it’s optional, you can have more control by using libbpf APIs directly)
- <name>\_\_destroy() – detaches all BPF programs and frees up all used resources

Using the skeleton code is the recommended way to work with bpf programs. Keep in mind, BPF skeleton provides access to the underlying BPF object, so whatever was possible to do with generic libbpf APIs is still possible even when the BPF skeleton is used. It's an additive convenience feature, with no syscalls, and no cumbersome code.

Several other advantages of skeleton

- BPF skeleton provides an interface for user space programs to work with BPF global variables. The skeleton code memory maps global variables as a struct into user space. The struct interface allows user space programs to initialize BPF programs before the BPF load phase and fetch and update data from user space afterward.
- The skel.h file reflects the object file structure by listing out the available maps, programs, etc. BPF skeleton provides direct access to all the BPF maps and BPF programs as struct fields. This eliminates the need for string-based lookups with bpf\_object\_find\_map\_by\_name() and bpf\_object\_find\_program\_by\_name() APIs, reducing errors due to BPF source code and user-space code getting out of sync.
- The embedded bytecode representation of the object file ensures that the skeleton and the BPF object file are always in sync.


## <a name="_8qitevxmanxj"></a>Step by Step compile, build, run your eBPF program
```
**upgautam@ubuntu205:~/CLionProjects/fastpathtestBpf/bpf$** bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

Now, we include vmlinux.h in ThousandLoopBpf.c, and compile ThousandLoopBpf.c to generate ThousandLoopBpf.o

```
**upgautam@ubuntu205:~/CLionProjects/fastpathtestBpf/bpf$** clang -g -I`realpath ~/CLionProjects/decoupling/linux/tools/lib` -I`realpath ~/CLionProjects/decoupling/linux/usr/include` -target bpf -Wall -O2 -c ThousandLoopBpf.c -o ThousandLoopBpf.o`
```


Then we utilize the genrated ThousandLoopBpf.o in our next command to generate skeleton header file (i.e., ThousandLoopBpf.skel.h) inside output directory.
```
**upgautam@ubuntu205:~/CLionProjects/fastpathtestBpf/bpf$** bpftool gen skeleton ThousandLoopBpf.o name ThousandLoopBpf | tee ./output/ThousandLoopBpf.skel.h > /dev/null 2>&1
```

//Then, we include ThousandLoopBpf.skel.h in our corresponding ThousandLoopBpf.skel.c file. ThousandLoopBpf.skel.c is like our normal user-space C program, that we compile to finally output ThousandLoopBpf binary.
```
**upgautam@ubuntu205:~/CLionProjects/fastpathtestBpf/bpf$** clang -g -I`realpath ~/CLionProjects/decoupling/linux/tools/lib` -I`realpath ~/CLionProjects/decoupling/linux/usr/include` -L`realpath ~/CLionProjects/decoupling/linux/tools/lib/bpf` -lbpf ThousandLoopBpf.skel.c -o ThousandLoopBpf
```

Finally, we run ThousandLoopBpf from inside QEMU (Note that QEMU is not docker container) as explained below.

__Note__: We will soon create another page for BPF developer machine setup.

I have volume mounted the whole CLionProjects directory inside my docker container, where I have configured everything that I need to run a bpf program. Now, from inside container, I can access all the Clion cloned projects.

From Clion, I cloned https://github.com/rosalab/decoupling and switched to milo-reloc branch, and created my separate branch off of that milo-reloc branch.

From host machine, I ran to start qemu
```
sudo make qemu-run //starts qemu
//sudo make qemu-ssh //then you can ssh as many shells as you need
```

Now, I am inside QEMU shell where I am installed latest linux kernel and all needed libraries for my BPF development. And I run my

```
root@q:/CLionProjects/fastpathtestBpf/bpf# ./ThousandLoopBpf
```
then I am inside qemu where I am installed latest linux kernel and all needed libraries for my BPF development. And I run my

`
root@q:/CLionProjects/fastpathtestBpf/bpf# ./ThousandLoopBpf //this is QEMU, not docker
root@q:~# cat /sys/kernel/debug/tracing/trace_pipe
`

and from another ssh sehll,
`
root@q:~# clear //or any command that should trigger kprobe/do_exit
`


Now, create corresponding loader implementation file for ThousandLoopBpf.skel.h, named that to ThousandLoopBpf.skel.c, which should be as below. How did I know this code? I just copy and pasted from some official site and then all I needed to change is name of the struct that I highlight in Yellow below. I am using Clion IDE, which does auto-completion, so much easy to complete this source code. (Note: This whole process is automated using Makefile. Keep reading below..)

## <a name="_1k97u1fu4v97"></a>Source Code
### <a name="_jl0riix6va0h"></a>ThousandLoop.c
```c
#include <stdio.h>
#include <unistd.h>
#include <time.h>

int main() {
   struct timespec start, end;
   long long delta_ns;

   clock_gettime(CLOCK_MONOTONIC, &start);

   int i;
   for (i=0; i<10000; i++) {
       //getpid(); //vDSO optimization gives false positive
       printf(""); //leads to system call
   }

   clock_gettime(CLOCK_MONOTONIC, &end);

   delta_ns = (end.tv_sec - start.tv_sec) * 1000000000LL + (end.tv_nsec - start.tv_nsec);
   printf("Elapsed time: %lld nanoseconds\n", delta_ns);

   return 0;
}
```


The equivalent BPF program for ThousandLoop.c is ThousandLoopBpf.c
### <a name="_hwqfl9tszkc4"></a>ThousandLoopBpf.c

```c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>


char LICENSE[] SEC("license") = "GPL";


//this gets triggered something like clear command from terminal
SEC("kprobe/do_exit")
int bpf_prog(struct pt_regs *ctx) {
   //start time
   __u64 start = bpf_ktime_get_ns();


   int i;


   for (i = 0; i < 1000; i++) {
       bpf_printk("");
   }


   //end time
   __u64 end = bpf_ktime_get_ns();
   __u64 delta_ns = end - start;
   bpf_printk("Elapsed time: %lld nanoseconds\n", delta_ns);


   return 0;
}
```


Compile ThousandLoopBpf.c to ThousandLoopBpf.o as,
```
**upgautam@ubuntu205:~/CLionProjects/fastpathtestBpf/bpf$** bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

Now, we include vmlinux.h in ThousandLoopBpf.c, and compile ThousandLoopBpf.c to generate ThousandLoopBpf.o
```
**upgautam@ubuntu205:~/CLionProjects/fastpathtestBpf/bpf$** clang -g -I`realpath ~/CLionProjects/decoupling/linux/tools/lib` -I`realpath ~/CLionProjects/decoupling/linux/usr/include` -target bpf -Wall -O2 -c ThousandLoopBpf.c -o ThousandLoopBpf.o
```

Also, generate skeleton header file
```
**upgautam@ubuntu205:~/CLionProjects/fastpathtestBpf/bpf$** bpftool gen skeleton ThousandLoopBpf.o name ThousandLoopBpf | tee ./output/ThousandLoopBpf.skel.h > /dev/null 2>&1
```

Then we write ThousandLoopBpf.skel.c as

```c
#include <stdio.h>
#include <unistd.h>
#include <signal.h>
#include <string.h>
#include <errno.h>
#include <bpf/libbpf.h>
#include "output/ThousandLoopBpf.skel.h"


static int libbpf_print_fn(enum libbpf_print_level level, const char *format, va_list args)
{
   return vfprintf(stderr, format, args);
}


static volatile sig_atomic_t stop;


static void sig_int(int signo)
{
   stop = 1;
}


int main(int argc, char **argv)
{
   struct ThousandLoopBpf *skel;
   int err;


   /* Set up libbpf errors and debug info callback */
   libbpf_set_print(libbpf_print_fn);


   /* Open load and verify BPF application */
   skel = ThousandLoopBpf__open_and_load();
   if (!skel) {
       fprintf(stderr, "Failed to open BPF skeleton\n");
       return 1;
   }


   /* Attach tracepoint handler */
   err = ThousandLoopBpf__attach(skel);
   if (err) {
       fprintf(stderr, "Failed to attach BPF skeleton\n");
       goto cleanup;
   }


   if (signal(SIGINT, sig_int) == SIG_ERR) {
       fprintf(stderr, "can't set signal handler: %s\n", strerror(errno));
       goto cleanup;
   }


   printf("Successfully started! Please run `sudo cat /sys/kernel/debug/tracing/trace_pipe` "
          "to see output of the BPF programs.\n");


   while (!stop) {
       fprintf(stderr, ".");
       sleep(1);
   }


   cleanup:
   ThousandLoopBpf__destroy(skel);
   return -err;
}
```

**Note**: Just copy-paste and change the ThousandLoopBpf* with your program name.

Finally, compile ThousandLoopBpf.skel.c to output ThousandLoopBpf

```
**upgautam@ubuntu205:~/CLionProjects/fastpathtestBpf/bpf$** clang -g -I`realpath ~/CLionProjects/decoupling/linux/tools/lib` -I`realpath ~/CLionProjects/decoupling/linux/usr/include` -L`realpath ~/CLionProjects/decoupling/linux/tools/lib/bpf` -lbpf ThousandLoopBpf.skel.c -o ThousandLoopBpf
```

That ThousandLoopBpf we then run inside docker container.

And can do from host machine.
```
upgautam@ubuntu205:~/CLionProjects/fastpathtestBpf/bpf$ sudo cat /sys/kernel/debug/tracing/trace\_pipe
```

See the output below,
```
[sudo] password for upgautam:

`           `<...>-874059  [008] ..... 2531317.041360: sys\_execve(filename: 7f301808fdc0, argv: 7f3068bfd180, envp: 7f310c004e20)

`           `<...>-874059  [008] ...21 2531317.041365: bpf\_trace\_printk: Elapsed time: 1398 nanoseconds

`           `<...>-874059  [008] ..... 2531317.042141: sys\_execve(filename: 561fce0192a0, argv: 561fce0193a0, envp: 7ffe57445708)

`           `<...>-874059  [008] ...21 2531317.042144: bpf\_trace\_printk: Elapsed time: 1348 nanoseconds

`              `ps-874066  [010] ..... 2531319.744947: sys\_execve(filename: 7f3018092410, argv: 7f2f22bfc180, envp: 7f310c004e20)

`              `ps-874066  [010] ...21 2531319.744952: bpf\_trace\_printk: Elapsed time: 1396 nanoseconds

`              `ps-874066  [010] ..... 2531319.745751: sys\_execve(filename: 55a57dbe62a0, argv: 55a57dbe63a0, envp: 7ffe94eed178)

`              `ps-874066  [010] ...21 2531319.745755: bpf\_trace\_printk: Elapsed time: 1349 nanoseconds

`            `runc-874070  [011] ..... 2531320.394696: sys\_execve(filename: c0001533d0, argv: c000066d80, envp: c000186320)

`            `runc-874070  [011] ...21 2531320.394700: bpf\_trace\_printk: Elapsed time: 1408 nanoseconds

`           `<...>-874076  [010] ..... 2531320.399943: sys\_execve(filename: c0003bc390, argv: c0001e2bd0, envp: c0003d10e0)

`           `<...>-874076  [010] ...21 2531320.399947: bpf\_trace\_printk: Elapsed time: 1353 nanoseconds
```


Command like "clear" in terminal will trigger do\_exit syscall.

# Automate using Makefile
I added several bpf c files in my bpf directory
`
upgautam@ubuntu205:~/CLionProjects/fastpathtestBpf/bpf$ ls
BillionLoopBpf.c  HundredMillionLoopBpf.c   Makefile          output      TenBillionLoopBpf.c  TenMillionLoopBpf.c   ThousandLoopBpf.c
HundredLoopBpf.c  HundredThousandLoopBpf.c  MillionLoopBpf.c  template.c  TenLoopBpf.c         TenThousandLoopBpf.c  vmlinux.h
`

I need to automate all build process (i.e., compile filename.c to filename.o then use bpftool gen to generate filename.skel.h fiel inside output directiy, and then create filename.ske.c, which includes filename.skel.h and uses filename object structs etc., and then finally compiling filename.skel.c file.

For this, I created Makefile

## Makefile
```cmake
CC = clang
CFLAGS = -g -target bpf -Wall -O2
INCLUDES = -I$(shell realpath ~/CLionProjects/decoupling/linux/tools/lib) -I$(shell realpath ~/CLionProjects/decoupling/linux/usr/include)

# Define source files
SRCS := $(filter-out template.c,$(wildcard *.c))
OBJS = $(SRCS:.c=.o)

# Define targets
all: $(OBJS)

# Rule to compile any .c file into a .o file and generate skeleton file
%.o: %.c
$(CC) $(CFLAGS) $(INCLUDES) -c $< -o $@
# Generate skeleton file and redirect stdout to the specified file
bpftool gen skeleton $@ name $(basename $@) | tee ./output/$(basename $@).skel.h > /dev/null 2>&1
# Generate skel.c file with content and replace occurrences of "ThousandLoopBpf" with the basename of the target
cat template.c | sed 's/%s/$(basename $@)/g' > ./$(basename $@).skel.c


# Define a rule to clean up generated files
clean:
rm -f $(OBJS)
rm -rf ./output/*
rm -f *.skel.c
```

## template.c
This template.c file is what we see mostly in our corresponding .skel.c files. Those .skel.c files substitute %s with the actual names.

```c
// template.c
#include <stdio.h>
#include <unistd.h>
#include <signal.h>
#include <string.h>
#include <errno.h>
#include <bpf/libbpf.h>
#include "output/%s.skel.h"


static int libbpf_print_fn(enum libbpf_print_level level, const char *format, va_list args)
{
return vfprintf(stderr, format, args);
}


static volatile sig_atomic_t stop;


static void sig_int(int signo)
{
stop = 1;
}


int main(int argc, char **argv)
{
struct %s *skel;
int err;


/* Set up libbpf errors and debug info callback */
libbpf_set_print(libbpf_print_fn);


/* Open load and verify BPF application */
skel = %s__open_and_load();
if (!skel) {
fprintf(stderr, "Failed to open BPF skeleton\n");
return 1;
}


/* Attach tracepoint handler */
err = %s__attach(skel);
if (err) {
fprintf(stderr, "Failed to attach BPF skeleton\n");
goto cleanup;
}


if (signal(SIGINT, sig_int) == SIG_ERR) {
fprintf(stderr, "can't set signal handler: %s\n", strerror(errno));
goto cleanup;
}


printf("Successfully started! Please run `sudo cat /sys/kernel/debug/tracing/trace_pipe` "
"to see output of the BPF programs.\n");


while (!stop) {
fprintf(stderr, ".");
sleep(1);
}


cleanup:
%s__destroy(skel);
return -err;
}

```

## Running Makefile
```c
upgautam@ubuntu205:~/CLionProjects/fastpathtestBpf/bpf$ make
clang -g -target bpf -Wall -O2 -I/home/upgautam/CLionProjects/decoupling/linux/tools/lib -I/home/upgautam/CLionProjects/decoupling/linux/usr/include -c BillionLoopBpf.c -o BillionLoopBpf.o
# Generate skeleton file and redirect stdout to the specified file
bpftool gen skeleton BillionLoopBpf.o name BillionLoopBpf | tee ./output/BillionLoopBpf.skel.h > /dev/null 2>&1
# Generate skel.c file with content and replace occurrences of "ThousandLoopBpf" with the basename of the target
cat template.c | sed 's/%s/BillionLoopBpf/g' > ./BillionLoopBpf.skel.c
clang -g -target bpf -Wall -O2 -I/home/upgautam/CLionProjects/decoupling/linux/tools/lib -I/home/upgautam/CLionProjects/decoupling/linux/usr/include -c HundredLoopBpf.c -o HundredLoopBpf.o
# Generate skeleton file and redirect stdout to the specified file
bpftool gen skeleton HundredLoopBpf.o name HundredLoopBpf | tee ./output/HundredLoopBpf.skel.h > /dev/null 2>&1
# Generate skel.c file with content and replace occurrences of "ThousandLoopBpf" with the basename of the target
cat template.c | sed 's/%s/HundredLoopBpf/g' > ./HundredLoopBpf.skel.c
clang -g -target bpf -Wall -O2 -I/home/upgautam/CLionProjects/decoupling/linux/tools/lib -I/home/upgautam/CLionProjects/decoupling/linux/usr/include -c HundredMillionLoopBpf.c -o HundredMillionLoopBpf.o
# Generate skeleton file and redirect stdout to the specified file
bpftool gen skeleton HundredMillionLoopBpf.o name HundredMillionLoopBpf | tee ./output/HundredMillionLoopBpf.skel.h > /dev/null 2>&1
# Generate skel.c file with content and replace occurrences of "ThousandLoopBpf" with the basename of the target
cat template.c | sed 's/%s/HundredMillionLoopBpf/g' > ./HundredMillionLoopBpf.skel.c
clang -g -target bpf -Wall -O2 -I/home/upgautam/CLionProjects/decoupling/linux/tools/lib -I/home/upgautam/CLionProjects/decoupling/linux/usr/include -c HundredThousandLoopBpf.c -o HundredThousandLoopBpf.o
# Generate skeleton file and redirect stdout to the specified file
bpftool gen skeleton HundredThousandLoopBpf.o name HundredThousandLoopBpf | tee ./output/HundredThousandLoopBpf.skel.h > /dev/null 2>&1
# Generate skel.c file with content and replace occurrences of "ThousandLoopBpf" with the basename of the target
cat template.c | sed 's/%s/HundredThousandLoopBpf/g' > ./HundredThousandLoopBpf.skel.c
clang -g -target bpf -Wall -O2 -I/home/upgautam/CLionProjects/decoupling/linux/tools/lib -I/home/upgautam/CLionProjects/decoupling/linux/usr/include -c MillionLoopBpf.c -o MillionLoopBpf.o
# Generate skeleton file and redirect stdout to the specified file
bpftool gen skeleton MillionLoopBpf.o name MillionLoopBpf | tee ./output/MillionLoopBpf.skel.h > /dev/null 2>&1
# Generate skel.c file with content and replace occurrences of "ThousandLoopBpf" with the basename of the target
cat template.c | sed 's/%s/MillionLoopBpf/g' > ./MillionLoopBpf.skel.c
clang -g -target bpf -Wall -O2 -I/home/upgautam/CLionProjects/decoupling/linux/tools/lib -I/home/upgautam/CLionProjects/decoupling/linux/usr/include -c TenBillionLoopBpf.c -o TenBillionLoopBpf.o
# Generate skeleton file and redirect stdout to the specified file
bpftool gen skeleton TenBillionLoopBpf.o name TenBillionLoopBpf | tee ./output/TenBillionLoopBpf.skel.h > /dev/null 2>&1
# Generate skel.c file with content and replace occurrences of "ThousandLoopBpf" with the basename of the target
cat template.c | sed 's/%s/TenBillionLoopBpf/g' > ./TenBillionLoopBpf.skel.c
clang -g -target bpf -Wall -O2 -I/home/upgautam/CLionProjects/decoupling/linux/tools/lib -I/home/upgautam/CLionProjects/decoupling/linux/usr/include -c TenLoopBpf.c -o TenLoopBpf.o
# Generate skeleton file and redirect stdout to the specified file
bpftool gen skeleton TenLoopBpf.o name TenLoopBpf | tee ./output/TenLoopBpf.skel.h > /dev/null 2>&1
# Generate skel.c file with content and replace occurrences of "ThousandLoopBpf" with the basename of the target
cat template.c | sed 's/%s/TenLoopBpf/g' > ./TenLoopBpf.skel.c
clang -g -target bpf -Wall -O2 -I/home/upgautam/CLionProjects/decoupling/linux/tools/lib -I/home/upgautam/CLionProjects/decoupling/linux/usr/include -c TenMillionLoopBpf.c -o TenMillionLoopBpf.o
# Generate skeleton file and redirect stdout to the specified file
bpftool gen skeleton TenMillionLoopBpf.o name TenMillionLoopBpf | tee ./output/TenMillionLoopBpf.skel.h > /dev/null 2>&1
# Generate skel.c file with content and replace occurrences of "ThousandLoopBpf" with the basename of the target
cat template.c | sed 's/%s/TenMillionLoopBpf/g' > ./TenMillionLoopBpf.skel.c
clang -g -target bpf -Wall -O2 -I/home/upgautam/CLionProjects/decoupling/linux/tools/lib -I/home/upgautam/CLionProjects/decoupling/linux/usr/include -c TenThousandLoopBpf.c -o TenThousandLoopBpf.o
# Generate skeleton file and redirect stdout to the specified file
bpftool gen skeleton TenThousandLoopBpf.o name TenThousandLoopBpf | tee ./output/TenThousandLoopBpf.skel.h > /dev/null 2>&1
# Generate skel.c file with content and replace occurrences of "ThousandLoopBpf" with the basename of the target
cat template.c | sed 's/%s/TenThousandLoopBpf/g' > ./TenThousandLoopBpf.skel.c
clang -g -target bpf -Wall -O2 -I/home/upgautam/CLionProjects/decoupling/linux/tools/lib -I/home/upgautam/CLionProjects/decoupling/linux/usr/include -c ThousandLoopBpf.c -o ThousandLoopBpf.o
# Generate skeleton file and redirect stdout to the specified file
bpftool gen skeleton ThousandLoopBpf.o name ThousandLoopBpf | tee ./output/ThousandLoopBpf.skel.h > /dev/null 2>&1
# Generate skel.c file with content and replace occurrences of "ThousandLoopBpf" with the basename of the target
cat template.c | sed 's/%s/ThousandLoopBpf/g' > ./ThousandLoopBpf.skel.c

upgautam@ubuntu205:~/CLionProjects/fastpathtestBpf/bpf$ ls
BillionLoopBpf.c       HundredLoopBpf.skel.c         HundredThousandLoopBpf.o       MillionLoopBpf.skel.c  TenBillionLoopBpf.skel.c  TenMillionLoopBpf.o        ThousandLoopBpf.c
BillionLoopBpf.o       HundredMillionLoopBpf.c       HundredThousandLoopBpf.skel.c  output                 TenLoopBpf.c              TenMillionLoopBpf.skel.c   ThousandLoopBpf.o
BillionLoopBpf.skel.c  HundredMillionLoopBpf.o       Makefile                       template.c             TenLoopBpf.o              TenThousandLoopBpf.c       ThousandLoopBpf.skel.c
HundredLoopBpf.c       HundredMillionLoopBpf.skel.c  MillionLoopBpf.c               TenBillionLoopBpf.c    TenLoopBpf.skel.c         TenThousandLoopBpf.o       vmlinux.h
HundredLoopBpf.o       HundredThousandLoopBpf.c      MillionLoopBpf.o               TenBillionLoopBpf.o    TenMillionLoopBpf.c       TenThousandLoopBpf.skel.c
upgautam@ubuntu205:~/CLionProjects/fastpathtestBpf/bpf$ ls output/
BillionLoopBpf.skel.h  HundredMillionLoopBpf.skel.h   MillionLoopBpf.skel.h     TenLoopBpf.skel.h         TenThousandLoopBpf.skel.h
HundredLoopBpf.skel.h  HundredThousandLoopBpf.skel.h  TenBillionLoopBpf.skel.h  TenMillionLoopBpf.skel.h  ThousandLoopBpf.skel.h
```

## If docker is using port, and qemu won't start then you can restart docker
`
sudo systemctl restart docker
`

# Other ways to load BPF program
There are other ways to load BPF bytecode into the system other than using that generated skeleton header file.

We can write manual loader C program, or using bpftool with autoattach. There are several other python or Go based loaders, or BCC loaders. I only discuss here bpftool autoattach and writing manual C loader program.

### Using bpftool autoattach
For any HelloWorldBpf.o that is compiled from HelloWorldBpf.c bpf c program,
`
sudo bpftool prog load HelloWorldBpf.o /sys/fs/bpf/HelloWorldBpf type tracepoint autoattach
`
But it worked only for "SEC("tracepoint/syscalls/sys_enter_execve")", and did not work for "SEC("kprobe/any_k_probe_traceable_kernel_function")
"

## Manual C program to load Bpf program
Let's find BPF program and their corresponding loader programs written by Linux kernel developers from here: https://elixir.bootlin.com/linux/v6.8.2/source/samples/bpf

Let's just take any two files: cpustat_kern.c and another tracex1.bpf.c. For tracex1.bpf.c, I renamed it to tracex1.bpf.c because my Makefile uses some sort of regex that treates .bpf.c files differently, so I had to rename it. Their corresponding files are cpustat_user.c and tracex1_user.c in the kernel. Also, we need vmlinux.h (see instruction above to generate how), and we need net_shared.h file as well. So, we need total 5 files from officla Linux kernel repo.

So, let's create a manual_load directory, and put all the files in there.

`
upgautam@ubuntu205:~/CLionProjects/fastpathtest/bpf/manual_load$ ls cpustat_kern.c  finalMakefile  Makefile  manual_loader  net_shared.h  output  template.c  TenLoopBpf.c  tracex1.c  vmlinux.h
`

I put TenLoopBpf.c just to make sure if my Makefile works or not. We also need to create output directory inside manual_loader because Makefile needs that. I created another directory manual_loader, where I put manually written loader files cpustat_user.c and tracex1_user.c. When we generate, using skeleton, the correspdong .skel.c files for tracex1.c and cputstat_kern.c, we can then compare those generated with those files in manual_loader directory.

### Makefile (this is slightly different from Makefile for Skeleton generation)

```Makefile
CC = clang
CFLAGS = -g -target bpf -Wall -O2 -D__TARGET_ARCH_x86
INCLUDES = -I$(shell realpath ~/CLionProjects/decoupling/linux/tools/lib) -I$(shell realpath ~/CLionProjects/decoupling/linux/usr/include) -I$(shell realpath ~/CLionProjects/decoupling/linux/tools/include)

SRCS := $(filter-out template.c,$(wildcard *.c))
OBJS = $(SRCS:.c=.o)

all: $(OBJS) generate-and-make

%.o: %.c
@echo "Compiling $<"
$(CC) $(CFLAGS) $(INCLUDES) -c $< -o $@
@echo "Generating skeleton file for $@"
bpftool gen skeleton $@ name $(basename $@) | tee ./output/$(basename $@).skel.h > /dev/null 2>&1
@echo "Generating skel.c file for $@"
cat template.c | sed 's/%s/$(basename $@)/g' > ./output/$(basename $@).skel.c

generate-and-make: generate-makefile
@echo "Running make in output directory"
@$(MAKE) -C output -f Makefile

generate-makefile:
@echo "Generating Makefile in output directory"
cp finalMakefile output/Makefile

clean:
rm -f $(OBJS)
rm -rf ./output/*
rm -f *.skel.c


.PHONY: all clean generate-makefile generate-and-make
```
As a template to new Makefile that we use inside output directory, I created finalMakefile (this is our template to output/Makefile)

### finalMakefile

```Makefile
## Define directories
LIB_DIR := $(shell realpath ~/CLionProjects/decoupling/linux/tools/lib)
INCLUDE_DIR := $(shell realpath ~/CLionProjects/decoupling/linux/usr/include)
BPF_LIB_DIR := $(shell realpath ~/CLionProjects/decoupling/linux/tools/lib/bpf)

## Define source files
SKEL_SRCS := $(wildcard *.skel.c)
SKEL_BINS := $(SKEL_SRCS:.skel.c=Final)

## Compiler options
CC := clang
CFLAGS := -g
LIBS := -lbpf

## Targets
all: $(SKEL_BINS)

## Rule to compile any .skel.c file into a binary
%Final: %.skel.c
	$(CC) $(CFLAGS) -I"$INCLUDE_DIR" -I"$LIB_DIR" -L"$BPF_LIB_DIR" $< $(LIBS) -o $@

## Target to run all generated binaries
run: $(SKEL_BINS)
	@for bin in $(SKEL_BINS); do \
		./$bin; \
	done

## Clean up generated files
clean:
	rm -f $(SKEL_BINS)


```
That's it. Now, you can run "make" and see .skel.c files generated inside output. These output's skel.c files do exactly same thing as we have inside manual_loader folder.
Now, you can see the difference between skeleton way vs. manual way

```c
upgautam@ubuntu205:~/CLionProjects/fastpathtest/bpf/manual_load$ ls
cpustat_kern.c  finalMakefile  manual_loader  output      TenLoopBpf.c  tracex1.c  vmlinux.h
cpustat_kern.o  Makefile       net_shared.h   template.c  TenLoopBpf.o  tracex1.o
upgautam@ubuntu205:~/CLionProjects/fastpathtest/bpf/manual_load$ ls manual_loader/
cpustat_user.c  tracex1_user.c
upgautam@ubuntu205:~/CLionProjects/fastpathtest/bpf/manual_load$ ls output/
cpustat_kernFinal    cpustat_kern.skel.h  TenLoopBpfFinal    TenLoopBpf.skel.h  tracex1.skel.c
cpustat_kern.skel.c  Makefile             TenLoopBpf.skel.c  tracex1Final       tracex1.skel.h

```

## Few useful references
https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md
https://liuhangbin.netlify.app/post/bpf-skeleton/
https://docs.kernel.org/bpf/libbpf/libbpf_overview.html

## using manual loader
let's name this file as array.kern.c
```c
#include <linux/bpf.h>
#include <linux/types.h>
#include <bpf/bpf_helpers.h>

char LISENSE[] SEC("license") = "Dual BSD/GPL";

struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __type(key, __u32);
    __type(value, __u32);
    __uint(max_entries, 256);
} ar SEC(".maps");


/**
 * Counter program that traces the number of times this
 * hookpoint has been hit
 */
SEC("tp/syscalls/sys_enter_getcwd")
int array(void *ctx)
{
    __u32 key = 0;
    __u32 * val = bpf_map_lookup_elem(&ar, &key);
    if (!val) {
        return -1;
    } else {
        __u32 new = (*val) + 1;
        bpf_map_update_elem(&ar, &key, &new, BPF_ANY);
    }
    return 0;
}
```
For above file, we can make a loader file such as (let's give name to load.user.c)
```c
/**
 * User program for loading a single generic program and attaching
 * Usage: ./load.user bpf_file bpf_prog_name
 */
#include <stdio.h>
#include <unistd.h>
#include <bpf/libbpf.h>

int main(int argc, char *argv[])
{
    if (argc != 3) {
        printf("Not enough args\n");
        printf("Expected: ./load.user bpf_file bpf_prog_name\n");
        return -1;
    }

    char * bpf_path = argv[1];
    char * prog_name = argv[2];

    struct bpf_object * prog = bpf_object__open(bpf_path);
    
    if (bpf_object__load(prog)) {
        printf("Failed");
        return 0;
    }

    struct bpf_program * program = bpf_object__find_program_by_name(prog, prog_name);

    if (program == NULL) {
        printf("Shared 1 failed\n");
        return 0;
    }

    bpf_program__attach(program);

    while (1) {
        sleep(1);
    }

    return 0;
}

```

let's create a Makefile to compile above both .kern.c and .user.c

```c

# $(..) is a function call. wilcard is a function that expands
# space separated list of filenames that match .kern.c
SOURCES := $(wildcard *.kern.c)
# := is assignment operator. replace .c with .o for each files in SOURCES
# it means if we have uddhav.c then it becomes uddhav.o in side Make context, but we also have .c files in our directory
# = means substitution operation
# target:dependencies   we also called prerequisites instead of dependencies
FILES := $(SOURCES:.c=.o)

USER_SRC := $(wildcard *.user.c)
USER := $(USER_SRC:.c=)

BPF-CLANG := clang
BPF_CLANG_CFLAGS := -target bpf -g -Wall -O2 -c
INCLUDE := -I../linux/usr/include/ -I../linux/tools/lib/
USER-CFLAGS := -I../linux/usr/include -I../linux/tools/lib/ -L../linux/tools/lib/bpf/

# all contains all files with .user.c and .kern.c
all: $(FILES) $(USER)

# %.c means any file ending with .c. In Makefile we don't have * unless some specific function calls (e.g., wildcard

#  $(wildcard *.kern.c:.c=.o) : %.o : %.c is same as $(FILES) : %.o : %.c
# $< is prerequisites while $@ represents target in Makefile. @ means output (end) to that at.
# This is like a for loop
$(FILES) : %.o : %.c
	$(BPF-CLANG) $(INCLUDE) $(BPF_CLANG_CFLAGS) $< -o $@

# -l to provide library to link against
# it means for loader, we need to link against libbpf library
$(USER) : % : %.c
	gcc $(USER-CFLAGS) -lbpf $< -o $@

.PHONY : clean
clean :
	rm $(FILES) $(USER)

```

now, just run "make" and then use `./load.user array.kern.o array`
That's it. It should load successfully. Note: array is a function name inside array.kern.c. Therfore, array is our bpf_prog 
Because syntax to load is: ./user.load bpf_file bpf_prog

Now, let's write another program named micro.c that triggers tp/syscalls/sys_enter_getcwd
```c
#include <stdio.h>
#include <unistd.h>
#include <time.h>
#include <stdint.h>

uint64_t run()
{
    struct timespec begin, end;
    char buf[256];
    clock_gettime(CLOCK_MONOTONIC_RAW, &begin);
    for (int i = 0; i < 5; i++) {
        getcwd(buf, 256);
    }
    clock_gettime(CLOCK_MONOTONIC_RAW, &end);

    uint64_t time = (end.tv_sec - begin.tv_sec) * 1000000000UL + (end.tv_nsec - begin.tv_nsec);
    //printf("Time is %lu nanoseconds\n", time);
    return time;
}

int main()
{
    printf("PID: %d\n", getpid());
    getchar();
    uint64_t results[10];
    run(); // Cache stuff?
    for (int i = 0; i < 10; i++) {
        results[i] = run();
        printf("%lu\n", results[i]);
    }
    return 0;
}

```

now, after you have loaded the bpf program, run micro. Your bpf loader program should trigger. 
To read trace_pipe, we can do `sudo cat /sys/kernel/debug/tracing/trace_pipe`, then in another terminal run ./micro, you should see the prink message from bpf program.

## BPF ring buffer example
```c

#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>


char LISENSE[] SEC("license") = "Dual BSD/GPL";

struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 256 * 1024 /* 256 KB */);
} ring_rs SEC(".maps");

struct event {
    int pid;
};


SEC("tp/syscalls/sys_enter_getcwd")
int empty(void *ctx) /* empty is bpf_prg */
{
    // Check first if space is available, and then do allocation.
    // This provides more insights and is helpful for debugging

    struct event *e;

    /* get PID and TID of exiting thread/process */
    __u64 id = bpf_get_current_pid_tgid();
    __u32 pid = id >> 32;
    __u32 tid = (__u32) id;

    /* ignore thread exits */
    if (pid != tid)
        return -1;

    /* reserve sample from BPF ringbuf */

    /* reserve sample from BPF ringbuf */
    e = bpf_ringbuf_reserve(&ring_rs, sizeof(*e), 0);
    if (!e) {
        bpf_printk("Size reserve successful!\n");
        return -1;
    }

    bpf_printk("Size reserve successful!\n");
    bpf_ringbuf_submit(e, 0);

    return 0;
}

```

To trace on: `echo 1 > /sys/kernel/debug/tracing/tracing_on`
To trace on: `echo 0 > /sys/kernel/debug/tracing/tracing_on`

we can kill those process(es) that are using trace_pipe (run as root)
`lsof -t /sys/kernel/debug/tracing/trace_pipe | xargs -I {} kill -9 {}`


## Host, Docker, QEMU
In our setup, Docker is providing root file system to QEMU, and docker also providing all build related things to QEMU. QEMU has new kernel and all build libraries to run bpf program.

I will write separate blog for how we set up whole inner_unikernel project, where we have this host, docker, qemu concept. 

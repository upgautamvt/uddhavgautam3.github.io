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

Finally, we run ThousandLoopBpf from inside docker container as explained below.

__Note__: We will soon create another page for BPF developer machine setup. 

I have volume mounted the whole CLionProjects directory inside my docker container, where I have configured everything that I need to run a bpf program. Now, from inside container, I can access all the Clion cloned projects.

From Clion, I cloned https://github.com/rosalab/decoupling and switched to milo-reloc branch, and created my separate branch off of that milo-reloc branch.

From host machine, I ran to start qemu
```
sudo make qemu-run
```

Now, I am inside docker container where I am installed latest linux kernel and all needed libraries for my BPF development. And I run my

```
root@q:/CLionProjects/fastpathtestBpf/bpf# ./ThousandLoopBpf
```

Now, create corresponding loader implementation file for ThousandLoopBpf.skel.h, named that to ThousandLoopBpf.skel.c, which should be as below. How did I know this code? I just copy and pasted from some official site and then all I needed to change is name of the struct that I highlight in Yellow below. I am using Clion IDE, which does auto-completion, so much easy to complete this source code.

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
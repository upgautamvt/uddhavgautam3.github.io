Below in the implementation section, there are implementation chapters for the user-space side and kernel side respectively, where I have provided the exact GitHub commit link, which you can access. 

I have also included program source code files and record, record-time, and syscall_diff.txt files there. 

# Idea

In the user-space program, you write a C program to access your custom newly added syscall by using its syscall number. In my case, it was 548 for my 64-bit base syscall named *uddhav5204_64*. I also created Makefile to compile and in the same user-space directory, I will later store git diff file, record, and record-times. 
```c++
upgautamvt@upg-vt-lab:~/CLionProjects/KernelWithBpfPrograms/user-space$ ls  
access-syscall  access-syscall.c  Makefile  record  record-time  syscall_diff.txt
```

1. In the kernel side, in the kernel directory, I added my syscall implementation file named sys_uddhav5204_64.c, where I used asmlinkage, and SYSCALL_DEFINE0 macro to return 5204 long value. The macro calls asmlinkage syscall function sys_udhav5204_64 (this is my custom syscall). In DEFINEX, the X part represents, how many parameters this macro uses. The asmlinkage is a keyword in the Linux kernel that specifies how a function should receive its arguments. It is primarily used for system calls, indicating that the function's arguments should be passed on the stack rather than in registers. This is required when use to kernel context switch happens like in syscall.   
2. In the kernel/Makefile, I added obj-y += sys_uddhav5204_64.o (this is the object file of my implementation source code sys_uddhav5204_64.c)  
3. In syscall_64.tbl, I added *548 64 common sys_uddhav5204_64 //add this at the end*  
4. This tbl file is to add 64-bit bases syscall with their type, name, and number.   
5. In syscalls.h file, I provided my syscall function definition. This is what contains function prototype (or definition) of syscalls.   
6. In uninstd_64.h file, I exposed syscall number so that user-space program can access it.  Because this file contains architecture specific syscall number, and it is the part of User-space API (UAPI).   
7. Then I did git diff, compiled the kernel, and then compiled user-space program, and then ran the user-space program. 

That was everything. 

Below, I have provided all implementation that tells what files added or modified and where. 

# Implementation

## User-space program to access system call *uddhav5204_64*

**Whole Implementation***:* [https://github.com/rosalab/KernelWithBpfPrograms/commit/8b8d6c033abb052065f8b60fea593b74cfaf674e](https://github.com/rosalab/KernelWithBpfPrograms/commit/8b8d6c033abb052065f8b60fea593b74cfaf674e)

**User-space directory**

- Makefile  
  ```
  CC = gcc  
  CFLAGS = -Wall -Werror  
  SRCS = $(wildcard *.c)  
  TARGETS = $(patsubst %.c,%, $(SRCS))  
  all: $(TARGETS)  
  %: %.c  
     $(CC) $(CFLAGS) -o $@ $<  
  clean:  
     rm -f $(TARGETS)  
  ```
  
    
- access-syscall.c
```c++
#include <stdio.h>  
  #include <unistd.h>  
  #include <sys/syscall.h>  
  #include <errno.h>  
    
  #define __NR_uddhav5204_64 548  
    
  int main() {  
     long result;  
     *// Test 64-bit syscall*  
     result = syscall(__NR_uddhav5204_64);  
     if (result == -1) {  
         perror("64-bit syscall failed");  
     } else {  
         printf("%ldn", result);  
     }  
    
     return 0;  
  }
```
  


## Kernel side modification

**Whole implementation**: [https://github.com/upgautamvt/linux/commit/3e93bac83525560ac387567a4c2183a73dc1f61b](https://github.com/upgautamvt/linux/commit/3e93bac83525560ac387567a4c2183a73dc1f61b)

### arch/x86/entry/syscalls/syscall_64.tbl

548 64 common sys_uddhav5204_64 //add this at the end

### include/linux/syscalls.h

`asmlinkage long sys_uddhav5204_64(void); //add this at the end`

### tools/arch/x86/include/uapi/asm/unistd_64.h (this is to provide syscall number)
```c++
#ifndef __NR_uddhav5204_64  
#define __NR_uddhav5204_64 548  
#endif
```

### kernel/sys_uddhav5204_64.c (this is the main syscall implementation file)

```c++
#include <linux/kernel.h>  
#include <linux/syscalls.h>

*// 64-bit syscall definition*  
asmlinkage long sys_uddhav5204_64(void) {  
       return 5204; *// Always return 5204*  
}

*//DEINEX is a macro, X tells number of arguments*  
SYSCALL_DEFINE0(uddhav5204_64)  
{  
       return sys_uddhav5204_64();  
}

### kernel/Makefile

obj-y += sys_uddhav5204_64.o //add this at the end
```

### Git diff
```c++
diff --git a/arch/x86/entry/syscalls/syscall_64.tbl b/arch/x86/entry/syscalls/syscall_64.tbl
index 7e8d46f4147f..e15e3c190ea6 100644
--- a/arch/x86/entry/syscalls/syscall_64.tbl
+++ b/arch/x86/entry/syscalls/syscall_64.tbl
@@ -426,5 +426,6 @@
 545	x32	execveat		compat_sys_execveat
 546	x32	preadv2			compat_sys_preadv64v2
 547	x32	pwritev2		compat_sys_pwritev64v2
+548 64 common sys_uddhav5204_64 //add this at the end
 # This is the end of the legacy x32 range.  Numbers 548 and above are
 # not special and are not to be used for x32-specific syscalls.
diff --git a/include/linux/syscalls.h b/include/linux/syscalls.h
index 77eb9b0e7685..05f3fb914b54 100644
--- a/include/linux/syscalls.h
+++ b/include/linux/syscalls.h
@@ -1295,3 +1295,4 @@ int __sys_getsockopt(int fd, int level, int optname, char __user *optval,
 int __sys_setsockopt(int fd, int level, int optname, char __user *optval,
 		int optlen);
 #endif
+asmlinkage long sys_uddhav5204_64(void); //add this at the end
\ No newline at end of file
diff --git a/kernel/Makefile b/kernel/Makefile
index ce105a5558fc..803f7c8b6548 100644
--- a/kernel/Makefile
+++ b/kernel/Makefile
@@ -159,3 +159,4 @@ $(obj)/kheaders_data.tar.xz: FORCE
 	$(call cmd,genikh)
 
 clean-files := kheaders_data.tar.xz kheaders.md5
+obj-y += sys_uddhav5204_64.o //add this at the end
\ No newline at end of file
diff --git a/kernel/sys_uddhav5204_64.c b/kernel/sys_uddhav5204_64.c
new file mode 100644
index 000000000000..2fada3bf3d18
--- /dev/null
+++ b/kernel/sys_uddhav5204_64.c
@@ -0,0 +1,13 @@
+#include <linux/kernel.h>
+#include <linux/syscalls.h>
+
+// 64-bit syscall definition
+asmlinkage long sys_uddhav5204_64(void) {
+	return 5204; // Always return 5204
+}
+
+//DEINEX is a macro, X tells number of arguments
+SYSCALL_DEFINE0(uddhav5204_64)
+{
+	return sys_uddhav5204_64();
+}
diff --git a/tools/arch/x86/include/uapi/asm/unistd_64.h b/tools/arch/x86/include/uapi/asm/unistd_64.h
index d0f2043d7132..001570df7d88 100644
--- a/tools/arch/x86/include/uapi/asm/unistd_64.h
+++ b/tools/arch/x86/include/uapi/asm/unistd_64.h
@@ -29,3 +29,6 @@
 #ifndef __NR_seccomp
 #define __NR_seccomp 317
 #endif
+#ifndef __NR_uddhav5204_64
+#define __NR_uddhav5204_64 548
+#endif
```


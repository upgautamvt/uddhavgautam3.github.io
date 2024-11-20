# //file: ModelFunctionInlined.user.c

```c
#include <stdio.h>
#include <time.h>
#include <stdlib.h>


static inline __attribute__((always_inline)) int getValue() {
return 10;
}


int main() {
struct timespec start, end;


clock_gettime(CLOCK_MONOTONIC, &start);
int x = getValue();


// This block will execute if the condition is true
// Compiler knows during compiler optimization because of x coming from non-inlined function
// it means anything inside the block will be eliminated as dead code
if (x > 100) { //this is computed false during compilation
printf("This gets eliminated inlined.\n");


       // Expensive matrix calculation
       int size = 100;
       double matrix[size][size];


       // Initialize matrix
       for (int i = 0; i < size; i++) {
           for (int j = 0; j < size; j++) {
               matrix[i][j] = (double)(i * j);
           }
       }


       // Perform some computations
       double sum = 0.0;
       for (int i = 0; i < size; i++) {
           for (int j = 0; j < size; j++) {
               sum += matrix[i][j];
           }
       }
       printf("Sum of matrix elements: %f\n", sum);
}


clock_gettime(CLOCK_MONOTONIC, &end);
double elapsed = (end.tv_sec - start.tv_sec) + (end.tv_nsec - start.tv_nsec) / 1e9;
printf("Execution time (inlined): %lf seconds\n", elapsed);


return 0;
}




/*
Compile as:
Using these compiler flags trying to match eBPF programs behavior when they are launched from kernel


gcc -O2 -S -fopt-info-optimized -fopt-info-missed -o ModelFunctionInlined.s ModelFunctionInlined.user.c 2> gcc_compile_inlined.log
gcc -O2 -g -o ModelFunctionInlined ModelFunctionInlined.s
objdump -d ModelFunctionInlined > objdump_inlined.txt
*/
```



# //file: ModelFunctionNotInlined.user.c

```c
#include <stdio.h>
#include <time.h>
#include <stdlib.h>


int getValue() {
return 10;
}


int main() {
struct timespec start, end;


clock_gettime(CLOCK_MONOTONIC, &start);
int x = getValue();


// This block will execute if the condition is true
// although the if block below is always false, but compiler can't know during compiler optimization because of x coming from non-inlined function
// it means anything inside the block can't be eliminated as dead code
if (x > 100) {
printf("This gets not eliminated in non-inlined.\n");


       // Expensive matrix calculation
       int size = 100;
       double matrix[size][size];


       // Initialize matrix
       for (int i = 0; i < size; i++) {
           for (int j = 0; j < size; j++) {
               matrix[i][j] = (double)(i * j);
           }
       }


       // Perform some computations
       double sum = 0.0;
       for (int i = 0; i < size; i++) {
           for (int j = 0; j < size; j++) {
               sum += matrix[i][j];
           }
       }
       printf("Sum of matrix elements: %f\n", sum);
}


clock_gettime(CLOCK_MONOTONIC, &end);
double elapsed = (end.tv_sec - start.tv_sec) + (end.tv_nsec - start.tv_nsec) / 1e9;
printf("Execution time (not inlined): %lf seconds\n", elapsed);


return 0;
}




/*
Compile as:
Using these compiler flags trying to match eBPF programs behavior when they are launched from kernel


gcc -O2 -fno-inline -fno-dce -S -fopt-info-optimized -fopt-info-missed -o ModelFunctionNotInlined.s ModelFunctionNotInlined.user.c 2> gcc_compile_not_inlined.log
gcc -O2 -g -o ModelFunctionNotInlined ModelFunctionNotInlined.s
objdump -d ModelFunctionNotInlined > objdump_notinlined.txt
*/
```


## Compile Inlined version of the program
```shell
(venv) upgautam@amd:~/Downloads/bpfabsorb/bpf-progs/filtering/model_function$ gcc -O2 -S -fopt-info-optimized -fopt-info-missed -o ModelFunctionInlined.s ModelFunctionInlined.user.c 2> gcc_compile_inlined.log
(venv) upgautam@amd:~/Downloads/bpfabsorb/bpf-progs/filtering/model_function$ gcc -O2 -g -o ModelFunctionInlined ModelFunctionInlined.s
(venv) upgautam@amd:~/Downloads/bpfabsorb/bpf-progs/filtering/model_function$ objdump -d ModelFunctionInlined > objdump_inlined.txt
```

## Compile Non-inlined version of the program
```shell
(venv) upgautam@amd:~/Downloads/bpfabsorb/bpf-progs/filtering/model_function$ gcc -O2 -fno-inline -fno-dce -S -fopt-info-optimized -fopt-info-missed -o ModelFunctionNotInlined.s ModelFunctionNotInlined.user.c 2> gcc_compile_not_inlined.log
(venv) upgautam@amd:~/Downloads/bpfabsorb/bpf-progs/filtering/model_function$ gcc -O2 -g -o ModelFunctionNotInlined ModelFunctionNotInlined.s
(venv) upgautam@amd:~/Downloads/bpfabsorb/bpf-progs/filtering/model_function$ objdump -d ModelFunctionNotInlined > objdump_notinlined.txt
```



### Compilation log inlined version (file: gcc_compile_inlined.log)
```shell
ModelFunctionInlined.user.c:46:5: optimized:   Inlining printf/15 into main/40 (always_inline).
ModelFunctionInlined.user.c:41:9: optimized:   Inlining printf/15 into main/40 (always_inline).
ModelFunctionInlined.user.c:21:9: optimized:   Inlining printf/15 into main/40 (always_inline).
ModelFunctionInlined.user.c:15:13: optimized:   Inlining getValue/39 into main/40 (always_inline).
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed:   not inlinable: main/40 -> __printf_chk/45, function body not available
ModelFunctionInlined.user.c:44:5: missed:   not inlinable: main/40 -> clock_gettime/41, function body not available
ModelFunctionInlined.user.c:14:5: missed:   not inlinable: main/40 -> clock_gettime/41, function body not available
ModelFunctionInlined.user.c:14:5: missed: statement clobbers memory: clock_gettime (1, &start);
ModelFunctionInlined.user.c:44:5: missed: statement clobbers memory: clock_gettime (1, &end);
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed: statement clobbers memory: __printf_chk (2, "Execution time (inlined): %lf seconds\n", elapsed_13);
```


### Compilation log non-inlined version (file: gcc_compile_not_inlined.log)
```shell
ModelFunctionNotInlined.user.c:46:5: optimized:   Inlining printf/5 into main/25 (always_inline).
ModelFunctionNotInlined.user.c:41:9: optimized:   Inlining printf/5 into main/25 (always_inline).
ModelFunctionNotInlined.user.c:21:9: optimized:   Inlining printf/5 into main/25 (always_inline).
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed:   not inlinable: main/25 -> __printf_chk/30, function body not available
ModelFunctionNotInlined.user.c:44:5: missed:   not inlinable: main/25 -> clock_gettime/26, function body not available
missed:   not inlinable: main/25 -> __builtin_stack_restore/29, function body not available
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed:   not inlinable: main/25 -> __printf_chk/30, function body not available
ModelFunctionNotInlined.user.c:25:16: missed:   not inlinable: main/25 -> __builtin_alloca_with_align/28, function body not available
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed:   not inlinable: main/25 -> __printf_chk/30, function body not available
ModelFunctionNotInlined.user.c:20:18: missed:   not inlinable: main/25 -> __builtin_stack_save/27, function body not available
ModelFunctionNotInlined.user.c:15:13: missed:   not inlinable: main/25 -> getValue/24, function not inlinable
ModelFunctionNotInlined.user.c:14:5: missed:   not inlinable: main/25 -> clock_gettime/26, function body not available
ModelFunctionNotInlined.user.c:36:27: missed: couldn't vectorize loop
ModelFunctionNotInlined.user.c:38:33: missed: not vectorized: complicated access pattern.
ModelFunctionNotInlined.user.c:37:31: optimized: loop vectorized using 16 byte vectors
ModelFunctionNotInlined.user.c:28:27: missed: couldn't vectorize loop
ModelFunctionNotInlined.user.c:30:30: missed: not vectorized: complicated access pattern.
ModelFunctionNotInlined.user.c:29:31: optimized: loop vectorized using 16 byte vectors
ModelFunctionNotInlined.user.c:14:5: missed: statement clobbers memory: clock_gettime (1, &start);
ModelFunctionNotInlined.user.c:20:18: missed: statement clobbers memory: saved_stack.3_26 = __builtin_stack_save ();
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed: statement clobbers memory: __builtin_puts (&"This gets not eliminated in non-inlined."[0]);
ModelFunctionNotInlined.user.c:25:16: missed: statement clobbers memory: matrix.2_30 = __builtin_alloca_with_align (80000, 64);
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed: statement clobbers memory: __printf_chk (2, "Sum of matrix elements: %f\n", sum_60);
ModelFunctionNotInlined.user.c:11:5: missed: statement clobbers memory: __builtin_stack_restore (saved_stack.3_26);
ModelFunctionNotInlined.user.c:44:5: missed: statement clobbers memory: clock_gettime (1, &end);
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed: statement clobbers memory: __printf_chk (2, "Execution time (not inlined): %lf seconds\n", elapsed_39);
```

Also, the binary size of the inlined version of the program is 16.1 KB where the size of the non-inlined version of the program is 16.2 KB. 
Because in the inlined version, the compiler during compilation time detected the unreachable code and eliminated that code as dead code, 
where in the not-inlined version the dead code remained there.

The runtime of these two versions of programs remain the same because during 
runtime the body inside the if condition, which is always false, doesn't get executed.

# How much more unnecessary work does the compiler need to do for the not-inlined version program during compilation?

## Removing the common from both programs’ compilation log

### Inlined
```shell
ModelFunctionInlined.user.c:15:13: optimized:  Inlining getValue/39 into main/40.
```
### Not-inlined

```shell
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed:   not inlinable: main/25 -> __printf_chk/30, function body not available
missed:   not inlinable: main/25 -> __builtin_stack_restore/29, function body not available
ModelFunctionNotInlined.user.c:25:16: missed:   not inlinable: main/25 -> __builtin_alloca_with_align/28, function body not available
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed:   not inlinable: main/25 -> __printf_chk/30, function body not available
ModelFunctionNotInlined.user.c:20:18: missed:   not inlinable: main/25 -> __builtin_stack_save/27, function body not available
ModelFunctionNotInlined.user.c:15:13: missed:   not inlinable: main/25 -> getValue/24, function not inlinable
ModelFunctionNotInlined.user.c:36:27: missed: couldn't vectorize loop
ModelFunctionNotInlined.user.c:38:33: missed: not vectorized: complicated access pattern.
ModelFunctionNotInlined.user.c:37:31: optimized: loop vectorized using 16 byte vectors
ModelFunctionNotInlined.user.c:28:27: missed: couldn't vectorize loop
ModelFunctionNotInlined.user.c:30:30: missed: not vectorized: complicated access pattern.
ModelFunctionNotInlined.user.c:29:31: optimized: loop vectorized using 16 byte vectors
ModelFunctionNotInlined.user.c:20:18: missed: statement clobbers memory: saved_stack.3_26 = __builtin_stack_save ();
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed: statement clobbers memory: __builtin_puts (&"This gets not eliminated in non-inlined."[0]);
ModelFunctionNotInlined.user.c:25:16: missed: statement clobbers memory: matrix.2_30 = __builtin_alloca_with_align (80000, 64);
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed: statement clobbers memory: __printf_chk (2, "Sum of matrix elements: %f\n", sum_60);
ModelFunctionNotInlined.user.c:11:5: missed: statement clobbers memory: __builtin_stack_restore (saved_stack.3_26);
```

### Whole log inlined,
```shell
ModelFunctionInlined.user.c:46:5: optimized:   Inlining printf/15 into main/40 (always_inline).
ModelFunctionInlined.user.c:41:9: optimized:   Inlining printf/15 into main/40 (always_inline).
ModelFunctionInlined.user.c:21:9: optimized:   Inlining printf/15 into main/40 (always_inline).
ModelFunctionInlined.user.c:15:13: optimized:   Inlining getValue/39 into main/40 (always_inline).
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed:   not inlinable: main/40 -> __printf_chk/45, function body not available
ModelFunctionInlined.user.c:44:5: missed:   not inlinable: main/40 -> clock_gettime/41, function body not available
ModelFunctionInlined.user.c:14:5: missed:   not inlinable: main/40 -> clock_gettime/41, function body not available
ModelFunctionInlined.user.c:14:5: missed: statement clobbers memory: clock_gettime (1, &start);
ModelFunctionInlined.user.c:44:5: missed: statement clobbers memory: clock_gettime (1, &end);
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed: statement clobbers memory: __printf_chk (2, "Execution time (inlined): %lf seconds\n", elapsed_13);
```

### Whole log non-inlined,
```shell
ModelFunctionNotInlined.user.c:46:5: optimized:   Inlining printf/5 into main/25 (always_inline).
ModelFunctionNotInlined.user.c:41:9: optimized:   Inlining printf/5 into main/25 (always_inline).
ModelFunctionNotInlined.user.c:21:9: optimized:   Inlining printf/5 into main/25 (always_inline).
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed:   not inlinable: main/25 -> __printf_chk/30, function body not available
ModelFunctionNotInlined.user.c:44:5: missed:   not inlinable: main/25 -> clock_gettime/26, function body not available
missed:   not inlinable: main/25 -> __builtin_stack_restore/29, function body not available
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed:   not inlinable: main/25 -> __printf_chk/30, function body not available
ModelFunctionNotInlined.user.c:25:16: missed:   not inlinable: main/25 -> __builtin_alloca_with_align/28, function body not available
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed:   not inlinable: main/25 -> __printf_chk/30, function body not available
ModelFunctionNotInlined.user.c:20:18: missed:   not inlinable: main/25 -> __builtin_stack_save/27, function body not available
ModelFunctionNotInlined.user.c:15:13: missed:   not inlinable: main/25 -> getValue/24, function not inlinable
ModelFunctionNotInlined.user.c:14:5: missed:   not inlinable: main/25 -> clock_gettime/26, function body not available
ModelFunctionNotInlined.user.c:36:27: missed: couldn't vectorize loop
ModelFunctionNotInlined.user.c:38:33: missed: not vectorized: complicated access pattern.
ModelFunctionNotInlined.user.c:37:31: optimized: loop vectorized using 16 byte vectors
ModelFunctionNotInlined.user.c:28:27: missed: couldn't vectorize loop
ModelFunctionNotInlined.user.c:30:30: missed: not vectorized: complicated access pattern.
ModelFunctionNotInlined.user.c:29:31: optimized: loop vectorized using 16 byte vectors
ModelFunctionNotInlined.user.c:14:5: missed: statement clobbers memory: clock_gettime (1, &start);
ModelFunctionNotInlined.user.c:20:18: missed: statement clobbers memory: saved_stack.3_26 = __builtin_stack_save ();
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed: statement clobbers memory: __builtin_puts (&"This gets not eliminated in non-inlined."[0]);
ModelFunctionNotInlined.user.c:25:16: missed: statement clobbers memory: matrix.2_30 = __builtin_alloca_with_align (80000, 64);
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed: statement clobbers memory: __printf_chk (2, "Sum of matrix elements: %f\n", sum_60);
ModelFunctionNotInlined.user.c:11:5: missed: statement clobbers memory: __builtin_stack_restore (saved_stack.3_26);
ModelFunctionNotInlined.user.c:44:5: missed: statement clobbers memory: clock_gettime (1, &end);
/usr/include/x86_64-linux-gnu/bits/stdio2.h:86:10: missed: statement clobbers memory: __printf_chk (2, "Execution time (not inlined): %lf seconds\n", elapsed_39);
```

**Note**: The binary size is 16.1 KB vs 16.2 KB.

# How to quantify the runtime?
## //file: ModelFunctionInlined.user.c

```c

#include <stdio.h>
#include <time.h>
#include <stdlib.h>

static inline __attribute__((always_inline)) int getValue() {
    return 10;
}

int main() {
    struct timespec start, end;

    clock_gettime(CLOCK_MONOTONIC, &start);
    int x = getValue();

    for (long i = 0; i < 1000000000l; i++) {
        //the if block comparision is calculated and known during compile time because x is from inline function
        if (x > 100) {
            printf("Inside loop.\n");
        }
    }

    // This block will execute if the condition is true
    // Compiler knows during compiler optimization because of x coming from non-inlined function
    // it means anything inside the block will be eliminated as dead code
    if (x > 100) { //this is computed false during compilation
        printf("This gets eliminated inlined.\n");

        // Expensive matrix calculation
        int size = 100;
        double matrix[size][size];

        // Initialize matrix
        for (int i = 0; i < size; i++) {
            for (int j = 0; j < size; j++) {
                matrix[i][j] = (double)(i * j);
            }
        }

        // Perform some computations
        double sum = 0.0;
        for (int i = 0; i < size; i++) {
            for (int j = 0; j < size; j++) {
                sum += matrix[i][j];
            }
        }
        printf("Sum of matrix elements: %f\n", sum);
    }

    clock_gettime(CLOCK_MONOTONIC, &end);
    double elapsed = (end.tv_sec - start.tv_sec) + (end.tv_nsec - start.tv_nsec) / 1e9;
    printf("Execution time (inlined): %lf seconds\n", elapsed);

    return 0;
}


/*
Compile as:
Using these compiler flags trying to match eBPF programs behavior when they are launched from kernel

gcc -O2 -S -fopt-info-optimized -fopt-info-missed -o ModelFunctionInlined.s ModelFunctionInlined.user.c 2> gcc_compile_inlined.log
gcc -O2 -g -o ModelFunctionInlined ModelFunctionInlined.s
objdump -d ModelFunctionInlined > objdump_inlined.txt
*/
```



## //file: ModelFunctionNotInlined.user.c

```c

#include <stdio.h>
#include <time.h>
#include <stdlib.h>

int getValue() {
    return 10;
}

int main() {
    struct timespec start, end;

    clock_gettime(CLOCK_MONOTONIC, &start);
    int x = getValue();

    for (long i = 0; i < 1000000000l; i++) {
        //the if block comparision is not calculated in compile time because it is not known during compile time because x is from non-inline function
        if (x > 100) {
            //No DCE this if body
            printf("Inside loop.\n");
      }
    }

    // This block will execute if the condition is true
    // although the if block below is always false, but compiler can't know during compiler optimization because of x coming from non-inlined function
    // it means anything inside the block can't be eliminated as dead code
    if (x > 100) {
        //DCE this whole if body
        printf("This gets not eliminated in non-inlined.\n");

        // Expensive matrix calculation
        int size = 100;
        double matrix[size][size];

        // Initialize matrix
        for (int i = 0; i < size; i++) {
            for (int j = 0; j < size; j++) {
                matrix[i][j] = (double)(i * j);
            }
        }

        // Perform some computations
        double sum = 0.0;
        for (int i = 0; i < size; i++) {
            for (int j = 0; j < size; j++) {
                sum += matrix[i][j];
            }
        }
        printf("Sum of matrix elements: %f\n", sum);
    }

    clock_gettime(CLOCK_MONOTONIC, &end);
    double elapsed = (end.tv_sec - start.tv_sec) + (end.tv_nsec - start.tv_nsec) / 1e9;
    printf("Execution time (not inlined): %lf seconds\n", elapsed);

    return 0;
}


/*
Compile as:
Using these compiler flags trying to match eBPF programs behavior when they are launched from kernel

gcc -O2 -fno-inline -fno-dce -S -fopt-info-optimized -fopt-info-missed -o ModelFunctionNotInlined.s ModelFunctionNotInlined.user.c 2> gcc_compile_not_inlined.log
gcc -O2 -g -o ModelFunctionNotInlined ModelFunctionNotInlined.s
objdump -d ModelFunctionNotInlined > objdump_notinlined.txt
*/
```

## Runtime

```shell
(venv) upgautam@amd:~/Downloads/bpfabsorb/bpf-progs/filtering/DCE2$ ./ModelFunctionInlined
Execution time (inlined): 0.000000 seconds
(venv) upgautam@amd:~/Downloads/bpfabsorb/bpf-progs/filtering/DCE2$ ./ModelFunctionNotInlined
Execution time (not inlined): 0.183777 seconds
```

# What optimizations performed?
Constant Propagation (CP): The value of x is propagated as 10 everywhere, so the compiler knows exactly what x is throughout the program.
Constant Folding (CF): The expression x > 100 can be folded into false, meaning the compiler already knows the result of the comparison without needing to evaluate it at runtime.
Dead Code Elimination (DCE): Since the code inside the second if (x > 100) block will never execute (due to x being 10), the compiler eliminates that code entirely.


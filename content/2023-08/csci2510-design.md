+++
title = "CSCI2510 Design"
date = "2023-08-10"
# [extra]
# add_toc = true
+++

## Preparation

- Simulator: Spike
- Debugger: Spike/GDB

![spike](https://github.com/poweihuang17/Documentation_Spike/raw/master/pictures/Sim.png)

## Assignment: Program Execution
**Objective:** understand how machine instruction is executed, how data is stored,
    and how the processor stack helps to make that happen.

**Task:** write and compare iterative and recursive program,
    learn about the calling convention, local variable storage
    and how does PC affect instruction flow. 


<!-- **Step 1.** write a [exponentiation by squaring](https://en.wikipedia.org/wiki/Exponentiation_by_squaring) -->
<!--     implementation following the given pseudocode.  -->

- Get familiar with calling convention (convert a simple example function)
    - Before calling/After returning from a function
        - Parameter passing
        - Get return value
    - After entering/Before returning from a function
        - Save/Restore register states
- Convert the exponentiation implementation
    - Pass given test cases
- Calculations
    - Stack frame size and count?
    - Figure out stack context with given PC and parameters

Expected learning results:
- The purpose of the stack
    - We cannot do function call if there is no place to save the current state (PC)
- The purpose of calling convention
    - It's a standard way and also easier to save state (e.g. PC) and pass variables
- Stack is also helpful for reducing programming complexity

```c
int f(int x) { return (x >> 16) | (x << 16); }

int pow_recursive(int base, int exponent) {
    if (exponent == 0) {
        return 1;
    } else if (exponent % 2 == 0) {
        int half_power = pow_recursive(base, exponent / 2);
        return half_power * half_power;
    } else {
        int half_power = pow_recursive(base, (exponent - 1) / 2);
        return base * half_power * half_power;
    }
}

int pow_iterative(int base, int exponent) {
    int result = 1;
    while (exponent > 0) {
        if (exponent % 2 == 1) {
            result *= base;
        }
        base *= base;
        exponent /= 2;
    }
    return result;
}
```

<!-- ```python -->
<!-- def f(x): -->
<!--     return (x >> 16) | (x << 16) -->

<!-- def pow_recursive(base, exponent): -->
<!--     if exponent == 0: -->
<!--         return 1 -->
<!--     elif exponent % 2 == 0: -->
<!--         half_power = pow_recursive(base, exponent // 2) -->
<!--         return half_power * half_power -->
<!--     else: -->
<!--         half_power = pow_recursive(base, (exponent - 1) // 2) -->
<!--         return base * half_power * half_power -->

<!-- def pow_iterative(base, exponent): -->
<!--     result = 1 -->
<!--     while exponent > 0: -->
<!--         if exponent % 2 == 1: -->
<!--             result *= base -->
<!--         base *= base -->
<!--         exponent //= 2 -->
<!--     return result -->
<!-- ``` -->

<!-- def main(): -->
<!--     size = 10 -->
<!--     sum = 0 -->
<!--     arr = [0 for i in range(0, size)] -->
<!--     for i in range(0, size): -->
<!--         arr[i] = f(i - size // 2) -->
<!--         sum += arr[i] -->
<!--     print(f"sum: {sum}, arr: {arr}") -->
<!--     return 0 -->

<!-- Your implementation should match the following prototype: -->
<!-- ```c -->
<!-- int pow_recursive(int base, int exponent); -->
<!-- int pow_iterative(int base, int exponent); -->
<!-- ``` -->

<!-- **Step 2.** compile your C program into RISC-V assembly code and an executable file. -->
<!-- *FIXME: how to ensure the generated assembly is exactly the same?* -->


<!-- **Step 3.**  -->


## Assignment: Cache
**Objective:** understand locality and its relation to program performance

**Task:** Write a matrix multiplication program in assembly.
- Get familiar with array and memory layout of multi-dimentional array
- Know the difference between passing by value and passing by pointer
- Write a dumb nested loop
    - Record performance with spike
- Swap the inner loop
    - Record performance with spike
- Ask students to do further optimization given the cache settings
    - (e.g. blocked multiplication)
    - Results are judged by improvements

Expected learning results:
- Main memory hierarchy
- Performance and locality


```bash
L2$ Bytes Read:         1870835392
L2$ Bytes Written:      787498176
L2$ Read Accesses:      29231803
L2$ Write Accesses:     12304659
L2$ Read Misses:        11543088
L2$ Write Misses:       108558
L2$ Writebacks:         4707069
L2$ Miss Rate:          28.052%
```

```c
#define ROWS 1024
#define COLS 1024
void matrix_multiply(int A[][COLS], int B[][COLS], int result[][COLS]) {
    for (int i = 0; i < ROWS; i++) 
        for (int j = 0; j < ROWS; j++) {
            result[i][j] = 0;
            for (int k = 0; k < ROWS; k++) 
                result[i][j] += A[i][k] * B[k][j];
        }
}
```


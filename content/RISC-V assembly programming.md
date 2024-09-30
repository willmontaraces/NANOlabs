---
title: RISC-V assembly programming
draft: false
tags:
---
In this session you should be provided with a [RISC-V programming reference card](https://cass-kul.github.io/files/riscv-card.pdf).
In this reference card you will find the following useful information:
1. RV32I instructions
2. C compiler data sizes
3. All RISC-V pseudoinstructions and their assembly translation
4. The stack layout of method calling convention
5. The RISC-V CSR registers (That we will not use in this practice)

# [GNU assembler directives](https://sourceware.org/binutils/docs/as/) 
Some general GNU assembler directives are essential when writing code.
- We use the `.text` and `.data` assembler macros to divide our code (in the text section) and program data (in the data section).
- We use the `.globl` assembler directive to define that a procedure label should be available for linking with other parts of the code. This is specially useful for linking assembly procedures with other files.
- We use the `.word`,`.ascii` and `.asciz` to declare variables with numbers or strings in our `.data` section. Words are used for defining integers, while `.ascii` and `.asciz` are used for defining strings. The difference between `.ascii` and `.asciz` is that `.asciz` appends a trailing zero to all strings, while `.ascii` does not. As such, a two letter string would take 2 bytes in ascii and three bytes in asciiz.
- We use the `.equ` to define assembler macros. Where `.equ SERIAL_PORT BASE 0xffffc000` would define a text substitution of the string `SERIAL_PORT_BASE` with the string `0xffffc000` in our code
- We use the `.org` directive to change the program address of the following data. This directive can be used in `.text` and `.data` regions. When used in text regions it defines the starting address where the program code will be stored. When used in data regions it defines the starting address where the data is stored.
# The [QTRVSIM](https://comparch.edu.cvut.cz/qtrvsim/app/) simulator

In this practice we will use the QtRvSim RISC-V architecture simulator. This simulator will be used in multiple practices, thus, I advice you to take your time and familiarize yourself with it.

When you open the QTSim simulator the following window will appear:

![[QTSIM_config.png]]

Here, you should select the `No pipeline no cache` preset, as the processor we studied today is the simplest processor you can implement. Don't worry, when we complete the units `1)Microprocessors and codesign reinforcement` and `2)RISC-V based system architecture` you will be familiar with every preset presented. For now, let's select the simpler configuration to learn how assembly works. Click on the `start empty` button.

![[Processor_layout.png]]

Now you are presented with all the building blocks of our processor. From program memory (where we store the instructions), the PC (Wich points to the currently-executing instruction), the registers (Where we store the data that we are operating on), the ALU (Where arithmetic and logic operations are performed) and the data memory (Where data is stored).

This simulator is able to execute all RISCV32IM code, as it implements the Integer and Multiplication extensions.

A window with the status of the register file can be opened clicking -> `Windows\Registers`
![[Registers.png]]

A window with the program code can be open by clicking -> `Windows\Program` resulting in the window below opening. The currently executing instruction is highlighted.

![[Pasted image 20240930133908.png]]

However, using this window to debug the assembler code can prove hard, as some instructions are translated into pseudoinstructions, breaking your code structuring. To ease debugging we can break up or code into sections with labels such as the above code, where the code is broken down into two labeled sections: `start`and `final`

To go into the assembler code corresponding to the *final* section we use the `Machine\Show Symbol` feature. Selecting the `final` symbol and clicking `show program`

![[Pasted image 20240930134351.png]]
Furthermore, one can set [breakpoints](https://en.wikipedia.org/wiki/Breakpoint) in their assembler program by clicking in the Bp field, as shown below. If the program is ran with the `Machine\Run` command and encounters an instruction with a breakpoint, it's execution will pause. Breakpoints are useful to examine the program behavior in certain sections of code where the programmer suspects a bug is pressent.
![[Pasted image 20240930145136.png]]

# Exercises
## Exercise 1. first assembly program

Let's start reading and programming assembly code.

>[!question] > Look at the following code snippet and answer the following questions
> > 1. what math operation is defined as s2 result?
> > 2. Which memory address does P have?
> > 3. Which value will the P address have at the end of the execution?

```armasm
.text
_start:
la t0, A
la t1, B
la t2, P
lw s0, 0(t0)
lw s1, 0(t1)
add s2, s1, s0
add s2, s2, s2
sw s2, 0(t2)

.data
.org 0x400
A: .word 3
B: .word 2
P: .space 4  
```

^ex1code

Now that we understand this piece of code let's execute it.
In our simulator, let's create a new program by clicking -> `File\New source` and then opening the `unknown` tab in our program.
In this file, paste the code presented in [[#^ex1code]]. Then, open the register file and memory contents in `Windows\Registers` and `Windows\Memory`
Run the program and check the values in registers `s0`,`s1` and `s2` and the value of the memory address of `P`. Are this values what you expected?

## Exercise 2: control flow

Let's check your knowledge of control flow by implementing the following C++ code into assembly:
```cpp
int list[] = {2,1};
i = 1;
if (list[i-1] > list[i]) {
	swap(&list[i-1], &list[i]);
}
```
Translate the above code into assembly by using the template below:
```
.text
.globl _start
_start:
la a0, list
li a1, 1
#WRITE YOUR CODE HERE, you must use the provided a0 and a1 registers 

.data
.org 0x400
list:
	.word 5, 2
```

**Write your solution into a file called *exercise2.s* and attach it in poliformat.**
## Exercise 3: Calling conventions and stack
Let's take the case that I am writing an assembly program where I need to call a function `foo`.  Which registers should I save before doing so? Where should I save such registers? Remember consulting the provided [reference card](https://cass-kul.github.io/files/riscv-card.pdf) calling conventions. Write the necessary procedure to call and return from foo, use the template below:
```
#WRITE YOUR CODE HERE
jal foo
#WRITE YOUR CODE HERE
```
**Write your soluition into a file called *exercise3.s* and attach it in poliformat**
## Exercise 4: [Bubble sort](https://en.wikipedia.org/wiki/Bubble_sort)

Bubble sort is a sorting algorithm known for it's simplicity. It sorts an array of numbers in an ascending or descending order. 

For this exercise, we will translate the following bubble sort code into RISC-V assembly following the contents studied in the class.
```cpp (^bubble)
void swap(int *addr0, int* addr1){
	int temp = *addr0;
	*addr0=*addr1;
	*addr1=temp;
}

void bubsort(int *list, int size) {
    bool swapped;
    do {
        swapped = false;
        for (int i = 1;i < size;i++) {
            if (list[i-1] > list[i]) {
               swap(&list[i-1], &list[i]);
               swapped = true;
            }
        }
    } while (swapped);
}
```
Remember that your procedure processes arguments using the `a[x]` registers. In this function, your procedure receives the list address in register `a0` and the size of the array to sort in register `a1`. For simplicity, we divided the bubble sort algorithm into two methods, swap and bubsort. Please implement and test them separately.

Here is the main function that calls and tests your bubble sort code: ^7774cc
```armasm
.text
.globl _start
_start:
	la a0, swap_space
	addi a1, a0, 4
	jal swap
verify_swap:
	la a1, swap_space
	lw a0, 0(a1)
	li a2, 1
	bne a0, a2, fail
	lw a0, 4(a1)
	bne a0, zero, fail
	
    la a0, list        #; Load address of the list into a0
    li a1, 7           #; Size of the list in a1

    jal bubsort       #; Call the bubble sort function

    #; Verification loop (compare sorted list to expected list)
    la t0, list        #; Load address of the sorted list
    la t1, expected_list  #; Load address of the expected sorted list
    li t2, 7           #; List size

verify:
    beqz t2, pass      #; If size is zero, list is sorted correctly
    lw t3, 0(t0)       #; Load current element from sorted list
    lw t4, 0(t1)       #; Load current element from expected list
    bne t3, t4, fail   #; If the elements don't match, test failed
    addi t0, t0, 4     #; Move to next element in sorted list
    addi t1, t1, 4     #; Move to next element in expected list
    addi t2, t2, -1    #; Decrease list size counter
    j verify           #; Repeat for the rest of the list

pass:
    li a7, 97          #; Exit syscall
    li a0, 0           #; Exit without error
    ecall              #; Exit, success

fail:
    li a7, 97          #; Exit syscall
    li a0, 1           #; Exit with error
    ecall              #; Exit, failure

swap: #; swaps a0 and a1 values
	#;a0: address of first value to swap
	#;a1: address of second value to swap
	#; YOUR CODE GOES HERE

bubsort:
	#; a0: Integer list to sort
	#; a1: Length of the list to sort
	#; YOUR CODE GOES HERE

.data
.org 0x400
list: 
    .word 10, 5, 3, 8, 12, 2, 7  #; Example unsorted list

size:
    .word 7                      #; Size of the list

expected_list: 
    .word 2, 3, 5, 7, 8, 10, 12  #; Expected sorted list

swap_space:
	.word 0, 1
```
**Write your solution into *exercise4.s* and attach it in poliformat**

==maybe add better testing code that prints what testcase failed, however, too much code?==
## Exercise 5: ASM bubble sort with C++ code
Now we provide the testcase in C++. Adapt the previously-coded bubble sort algorithm so that it can be integrated into the following C++ program. Compile it and execute it in the simulator.

**Remember, if you want to use callee saved registers you need to push them into the stack.**

main.c
```cpp
#include <iostream>

// Declaration of the external bubble sort function in assembly
extern "C" void bubsort(int* list, int size);

// Verification function to compare two lists
bool verify(const int* sorted_list, const int* expected_list, int size) {
    for (int i = 0; i < size; i++) {
        if (sorted_list[i] != expected_list[i]) {
            return false; // Lists do not match
        }
    }
    return true; // Lists match
}

int main() {
    int list[] = {10, 5, 3, 8, 12, 2, 7}; // Example unsorted list
    int size = 7;  // Size of the list
    int expected_list[] = {2, 3, 5, 7, 8, 10, 12}; // Expected sorted list

    // Call the bubble sort function
    bubsort(list, size); // Pass the array and size

    // Verification: compare the sorted list with the expected list
    if (verify(list, expected_list, size)) {
        std::cout << "List sorted correctly!" << std::endl;
        return 0; // Success
    } else {
        std::cout << "List sorting failed!" << std::endl;
        return 1; // Failure
    }
}
```

bubsort.s
```armasm
.section .text
.global bubsort
bubsort:
	# Arguments: 
	# a0: address of the list (array) 
	# a1: size of the list

	#WRITE bubble sort code here
	### .......
	### .......

	ret  # return from function
```
**Attach bubsort.s in poliformat**

To compile this program, we will need a C++ compiler. However, the windows machines in our laboratory are x86, and we want to compile RISC-V code. That's why we need a crosscompiler.
Crosscompilers are tools that are able to produce code for different architectures to those that they are compiled for.

==añadir inline assembly==
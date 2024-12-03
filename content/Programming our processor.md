---
title: Programming our processor
draft: false
tags:
---
## Preamble and setup
Until now we have learned how to program in RV32I assembly [[RISC-V assembly programming]] and implemented a RV32I singlecycle processor [[Simple RISC-V monocycle processor]] So the next logical step is to write C++ programs to execute in our home-made processor.

Our homemade processor is not very advanced, as such RAMs and ROMs are implemented in the FPGA fabric, this is very fast (convenient for doing everything in a single cycle) but consumes a lot of resources. As such, our processor is limited to 1536 bytes of storage in our FPGA. To make efficient use of such memory we will open our previous practice completed CPU and perform some modifications.

### Exercise 1. Modifying CPU code to take varying amounts of ROM and RAM
1. Our rom will now be organized in bytes instead of words, as such, open your rom file named ``rom.sv`` and replace its contents by the following:
	```systemverilog
	module rom (rdaddr, rddata);
	parameter w=32,d=1024,file="rom_R.txt";
	localparam a=$clog2(d);
	
	input [a-1:0] rdaddr;
	output [w-1:0] rddata;
	logic [8-1:0] mem [0:d];
		
	initial $readmemh(file,mem);
	
	assign rddata[7:0]   = mem[rdaddr];
	assign rddata[15:8]  = mem[rdaddr+1];
	assign rddata[23:16] = mem[rdaddr+2];
	assign rddata[31:24] = mem[rdaddr+3];
	
	endmodule   
	```

2. Open the top_level module of our processor named `tiny_riscv.sv` and replace its parameters by the following ones:
   ```systemverilog
   parameter w=32,d_ram=256, d_rom=512,r=32;
	```
	As you can see, now we are able to take different ammounts of ROM and RAM in our processor, now we just need to propagate such signals through the design to the ROM and RAM instances.
	Now modify the `riscv_gpio` instance with the new parameters
	```systemverilog
	riscv_gpio #(.from(from), .fram(fram),.w(w),.d_ram(d_ram), .d_rom(d_rom),.r(r)) riscv_gpio
	```

3. Open the `riscv_gpio.sv` file and add the new parameters
	````systemverilog
	   parameter w=32,d_ram=128, d_rom=128,r=32;
	````
	Then, change the RISCV core instance with the ROM amount
	```systemverilog
	riscv_core #(.w(w),.d(d_rom),.r(r), .from(from), .fram(fram)) riscv_core
	```
	And finally the RAM instance with the RAM amount
	```systemverilog
	ram #(.w(w),.d(d_ram),.file(fram)) ram
	```
	
	Now we can optimize our processor for the code being executed. The more ROM and RAM we instantiate the more FPGA resources we consume and the longer our synthesis and implementation times.

## Writing and compiling our own C++ code
To execute C++ code in our processor we need 3 files:
1. Our processor setup file (`crt0local.S`). This file sets up RV32I registers such as the stack pointer and global pointer registers for usage in our main function.
2. Our C/C++ code file (`main.cpp`). In this file we must implement a main function where our code will run. As we are targeting an embedded system, our main function will probably contain an infinite while(true) loop.
3. Our linker file (`NANOnewRISC.ld`). In this file we specify our processor configuration, we define which memories our processor has (in this case a ROM and a RAM), their corresponding memory addresses and which sections of our program map to those memories. In here we define our memory sizes, such sizes are used by the compiler to determine if our program will fit (or not) into our processor.
### Exercise 2. Examining the linker script and main files.
Create a new folder in your computer where you will write the code for this practice.
In this folder you must create a file named `crt0local.S` with the following contents
```asm
/* minimal replacement of crt0.o which is else provided by C library */
.globl main
.globl _start
.option norelax
.text
_start:
        .option push
        .option norelax
        la gp, __global_pointer$
        .option pop
        la      sp, __stack_end
        addi    a0, zero, 0
        addi    a1, zero, 0
        jal     main
quit:
        addi    a0, zero, 0
        addi    a7, zero, 93  /* SYS_exit */
        ecall
loop:   ebreak
        beq     zero, zero, loop
.bss
__stack_start:
        .skip   200
__stack_end:
.end _start
```

Take note that we are defining the size of our stack with the *skip* between stack_start and stack_end labels. 
We then will create the main of our program. We will name this file `main.cpp` with the following contents:
```cpp
#define IO 0xDEAD0000

int initialValue=0xCAFEBABE;

void writeIO(int data) {
    volatile int *io_ptr = (int *)IO;
    *io_ptr = data;
}
/*
int hex2dec(int hex){
   int bcdResult = 0;
   int shift = 0;

    while (hex > 0) {
      bcdResult |= (hex % 10) << (shift++ << 2);
      hex /= 10;
   }
    return bcdResult;
}

void writeIOdecimal(int data){
    volatile int *io_ptr = (int *)IO;
    int bcdResult = hex2dec(data);
    *io_ptr = bcdResult;
}*/

int readIO() {
    volatile int *io_ptr = (int *)IO;
    return *io_ptr;
}

void wait(int cycles) {
    int maxIters = (cycles/3-2);
    for (int i = 0; i < maxIters; i++) {
       __asm__("addi x0, x0, 0\n\t");
    }
}

int main() {
    int a = 0;
    int b = 1;
    writeIO(initialValue);
    wait(10000000);
    //Fibonacci succession
    while(true){
        writeIO(a);
        wait(10000000);
        a = a+b;
        writeIO(b);
        wait(10000000);
        b = a+b;
    }
}
```
As you can see the main is written in C++, and uses the writeIO function to write a 32-bit number to the 7segment display of our processor.

Finally, for compiling this program into our machine we need the linker script file. We will name this file `NANOnewRISC.ld` with the following contents:
```cpp
ENTRY(_start)

MEMORY
{
  /* Define a memory region for the text section with 512 bytes */
  text_mem (rx) : ORIGIN = 0x00000000, LENGTH = 512
  /* Define a memory region for the data section with 256 bytes */
  data_mem (rw) : ORIGIN = 0x10000000, LENGTH = 256
}

SECTIONS
{
  /* Place .text in text_mem with size limitation */
  .text : 
  {
    //*(.text._start) 
    _start = .; /* Define the entry point at 0x0 */
    *(.text*)
  } > text_mem

  /* Set the global pointer _global_pointer$ */
  __global_pointer$ = 0x10000000 + 0x000;  /* Offset within data_mem, adjust as needed */

  /* Place .data in data_mem with size limitation */
  .data : 
  {
    *(.data*)
  } > data_mem

  /* Place .bss in data_mem as well, following .data */
  .bss : 
  {
    *(.bss*)
  } > data_mem
}
```
>[!question]- How many memories do we instantiate? Which size do we assign to each memory?
>> [!todo] We instantiate two memories, one holding the code (text) with 512 Bytes of size and one for the stack and variables (data) of 256 Bytes of size.

>[!question]- Why is the IO pointer instantiated as a volatile int*? What would happen if it was instantiated as a int*?
>> [!todo] It is instantiated as volatile to prevent the compiler from optimizing IO accesses into nothing. If a variable is not declared volatile and not read in the program, the compiler optimizes it. However, volatile variables are not able to be opimized away, as such, our IO accesses should not be optimized.

### Exercise 3. Compiling and uploading the code
To compile our code we will execute the following cmd commands:
```bash
riscv64-unknown-elf-g++.exe -march=rv32i -mabi=ilp32 -nostdlib  .\crt0local.S .\main.cpp  -lgcc -O3 -T .\NANOnewRISC.ld -o test.elf

riscv64-unknown-elf-objcopy -O verilog --only-section=.text test.elf rom.txt

riscv64-unknown-elf-objcopy -O verilog --only-section=.data --only-section=.bss --only-section=.sdata test.elf ram.txt

riscv64-unknown-elf-objdump.exe -dS test.elf > program_asm.txt
```
The first command will compile the provided code with the linker and optimization level -O3 (Optimize for speed), then we dump the binary representation of the code in rom.txt and ram.txt, and finally we dump the human-readable assembly program in program_asm.txt 

Take note that we are doing a procedure very similar to that of the [[Practice 1. RISC-V assembly programming]] practice, where we wrote an assembly method and called it from the C++ file. However, this time we are calling the program main from our assembly code instead of our assembly function from our C++ main.

**Copy the generated rom.txt and ram.txt files into your project, REMOVE THE @10.... LINE FROM THE RAM.TXT FILE. Set the tiny_riscv from and fram parameters to point to the new program and generate a bitfile**
While your bitfile is being generated answer the following questions
>[!question] Check the first instruction of the wait method. It is an integer division. Did we implement the integer division extension in our processor? Can we execute such instruction without having the explicit integer division function in our processor? Analyze the compiled program_asm.txt file to check if we are using any unimplemented instruction.

> [!question] Check the ram.txt file. Which line of the code sets this value? Why is this value in the ram file and other variables are not?

 >[!question] Analyze the code to check what will happen if we change the value in the RAM.txt with another value. Will the code still function? What will happen in the FPGA when we flash the bitfile with this changes?

**Show this exercise to the professor for evaluation in class**
### Exercise 4. Extending the code
Uncomment the commented methods. They are used to print the input value as decimal in the 7-segment display. Change the two writeIO method calls for the new writeIOdecimal calls to see the Fibonacci sequence in decimal on the 7segment.

For your code to fit into the FPGA you will need to extend the rom size both in the C++ compiler and in the Vivado project. **Assume a maximum ROM size is 1024 Bytes**, if you need more ROM size, change the optimization level of the compiler from `-O3` (Optimize for speed) to `-Os` (Optimize for code size). Compile your program, generate your new bitfile and test it on the FPGA.
It should now print the Fibonacci succession in decimal (instead of hexadecimal).

**Show this exercise to the professor for evaluation in class**
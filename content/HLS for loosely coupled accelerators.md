---
title: HLS for loosely coupled accelerators
draft: false
tags:
---
 In this lab session we will be exploring the creation of HLS accelerators in AMD Xilinx platforms. We will make a special focus on the memory interfaces of such accelerators, and the paradigm that AMD uses to communicate with them, focusing on how we can integrate them into a SoC.
### Setting up the environment
Open Vitis and create a new `HLS` component with a target hardware part of `xc7k325tlffg900-2L` and a `100MHz` clock. With a `Vivado IP Flow Target`and a `package.output.format` of `Generate RTL, skip packaging`.
*This tutorial will use files created by Xilinx and with an Apache, Version 2.0 license. Such license is located at http://www.apache.org/licenses/LICENSE-2.0* 
Then create a new source file named `krnl_vadd.cpp` with the following contents.
```cpp
#include "krnl_vadd.hpp"

// Read Data from Global Memory and write into Stream inStream
static void read_input(uint32_t* in, hls::stream<uint32_t>& inStream,
                       int vSize) {
// Auto-pipeline is going to apply pipeline to this loop
mem_rd:
    for (int i = 0; i < vSize; i++) {
#pragma HLS LOOP_TRIPCOUNT min = size max = size
        // Blocking write command to inStream
        inStream << in[i];
    }
}

// Read Input data from inStream and write the result into outStream
static void compute_add(hls::stream<uint32_t>& inStream1,
                        hls::stream<uint32_t>& inStream2,
                        hls::stream<uint32_t>& outStream, int vSize) {
// Auto-pipeline is going to apply pipeline to this loop
execute:
    for (int i = 0; i < vSize; i++) {
#pragma HLS LOOP_TRIPCOUNT min = size max = size
        // Blocking read command from inStream and Blocking write command
        // to outStream
        outStream << (inStream1.read() + inStream2.read() +1);
    }
}

// Read result from outStream and write the result to Global Memory
static void write_result(uint32_t* out, hls::stream<uint32_t>& outStream,
                         int vSize) {
// Auto-pipeline is going to apply pipeline to this loop
mem_wr:
    for (int i = 0; i < vSize; i++) {
#pragma HLS LOOP_TRIPCOUNT min = size max = size
        // Blocking read command to inStream
        out[i/2] = outStream.read();
    }
}

extern "C" {
/*
    Vector Addition Kernel Implementation using dataflow
    Arguments:
        in1   (input)  --> Input Vector 1
        in2   (input)  --> Input Vector 2
        out  (output) --> Output Vector
        vSize (input)  --> Size of Vector in Integer
   */
void krnl_vadd(uint32_t* in1, uint32_t* in2, uint32_t* out, int vSize) {
    static hls::stream<uint32_t> inStream1("input_stream_1");
    static hls::stream<uint32_t> inStream2("input_stream_2");
    static hls::stream<uint32_t> outStream("output_stream");

#pragma HLS dataflow
    // dataflow pragma instruct compiler to run following three APIs in parallel
    read_input(in1, inStream1, vSize);
    read_input(in2, inStream2, vSize);
    compute_add(inStream1, inStream2, outStream, vSize);
    write_result(out, outStream, vSize);
}
}
```
Then create a new header file for this kernel named `krnl_vadd.hpp` with the following contents
```cpp
#ifndef _KRNL_VADD_H_
#define _KRNL_VADD_H_

// Includes
#include <iostream>
#include <ap_int.h>
#include <hls_stream.h>

const int size = 1024;

extern "C" {
void krnl_vadd(uint32_t* input1, uint32_t* input2, uint32_t* output, int vSize);
}
#endif
```
And finally create a testbench file named `vadd_tb.cpp` with the following contents:
```cpp
#include "krnl_vadd.hpp"

int main() {
    uint32_t in1[size], in2[size];
    uint32_t out[size], res[size];
    for (int i = 0; i < size; ++i) {
        in1[i] = i;
        in2[i] = i;
        out[i] = 0;
        res[i] = in1[i] + in2[i];
    }

    krnl_vadd(in1, in2, out, size);

    for (int i = 0; i < size; ++i) {
        if (res[i] != out[i])
            return EXIT_FAILURE;
    }

    std::cout << "Test passed.\n";
    return EXIT_SUCCESS;
}
```
Then, open the `hls_config.cfg` file and set the `krnl_vadd` function as the top function to be synthesized.
## Exercise 1. Analyzing the code
The kernel is supposed to do the vector multiplication of two vectors, saving the result in a new vector. Check the testbench to understand the software behavior. However, our hardware design team has introduced two bugs in the kernel design. Find them, fix them and report them in a file named `yourname_ex1.txt`. Once you fix them you should have the testbench passing.
You can run the testbench by running the C simulation in Vitis.
## Exercise 2. Analyzing the synthetized code
Run the C synthesis. Open the Synthesis report and answer the following questions in a file named `yourname_ex2.txt`:
- What kind of interface is being used in our module?
- How many cycles does our module take to execute the kernel?
- How do you think the tool determines the cycles to execute our kernel in synthesis mode?
- Further analyze the performance and resource estimates. What is the read and compute latency? And their Initialization Interval? How does this value compare with the previous question.
## Exercise 3. Using AXI
Using the `#pragma HLS interface` directive adapt this module to use AXI4 for reading the vectors and AXI4-lite for configuration. We want to maximize module throughput without caring about resources. Report how you did it in a file named `yourname_ex3.doc`
Further include a capture of your `synthesis report/HW interfaces` tab in such report.
Run the C/RTL cosimulation and check the latency of execution. Then check the timeline trace of the cosimulation. Respond the following questions in the `yourname_ex3.doc`
- What is your cosimulation latency?
- Is it the same as your synthesis latency?
- Can you find the cause of this disparity using the timeline trace tool? What is it?
- How would you solve this disparity?
## Exercise 4. Analyzing kernel interfaces
Go to your project folder in your file explorer, then open your component and follow this path `component_name\hls\syn\verilog`. Open the top level file, this is our HLS defined module synthetized into RTL!! Open `krnl_vadd_control_s_axi.v` and read the input/output ports. Then read the address info. Do you understand how your accelerator is programmed? Answer the following questions in a file named `yourname_ex4.doc`
- Which of the studied interfaces is this module using?
- Provide a description of the behavior of each one of the memory mapped registers inside this module. Use the Address info of your module to provide such a description.
Now go to Vitis once again and open your component configuration file in `hls_config.cfg`, then using the search settings feature set `cosim.wave_debug` to true and `cosim.trace_level` to all. Then, run C/RTL cosimulation again. Vivado should open with a waveform of our interfaces. Let's analyze them.
Open your design top signals. There, in C inputs you should find your 2 read interfaces.
Append to `yourname_ex4.doc` your answers to the following questions:
- In which instance of time is the first AXI address read transaction performed?
- In which instance of time is the first AXI read data transaction performed?
- Which address does this transaction access to?
- What are the characteristics of this address read transaction? (size of bus, burst mode, burst length, id)
- Provide a capture of your vivado program that shows the instant where the first address read transaction happens.
- In which instance of time does the first address write happen?
- In which instance of time does the first write happen?
- What value is transmitted in the first write?
- In which instance of time does the first write response happen?
## Exercise 5. Bundling interfaces
Now that we analyzed the interface behavior of our vector add kernel let's perform some changes.
The SoC where we are integrating our accelerator does not support 3 AXI masters, only two. Answer the following questions in a file named `yourname_ex5.doc`
- Between both read pointers (in1, in2) and the write pointer (out1), does AXI support bundling them together to save up on chip area?
- Which of these pointers can we bundle together in the same interface? Why?
- Can we use only one interface for all three pointers? Will this have any performance impact?
To test your answers modify the given code by bundling 2 or 3 pointers to the same interface. Then, regenerate the synthesis and, disabling `cosim.wave.debug` execute the C/RTL cosim.
Check the performance characteristics of what we will call bundle2 and bundle3 approaches. Perform the following tasks in the aforementioned lab report:
- Paste your pragma code for the bundle2 and bundle3 approaches. What feature of AXI did you exploit to manage to bundle everything to the same interface?
- Annotate the latency reported by the synthesis and cosimulation tool for the bundle2 and bundle3 approaches and comment on any major discrepancy between both. If there is any major discrepancy speculate why.
- Enable `cosim.wave.debug` with the bundle3 approach and check the ARID channel. Answer how the values in this channel correspond to the pragma that you defined.
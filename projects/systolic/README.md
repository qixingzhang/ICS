# EE219 Lab2 Systolic Array

## Introduction

This lab aims to design a matrix multiplication module based on systolic array, and apply it to 2D convolution

### Systolic Array

In parallel computer architectures, a systolic array is a homogeneous network of tightly coupled processing elements (PEs) called cells or nodes. Each node or PE independently computes a partial result as a function of the data received from its upstream neighbours, stores the result within itself and passes it downstream. 

<p align="center">
  <img src ="img/array.png"  width="45%"/>
  <img src ="img/flow.gif"  width="45%"/>
</p>
<p align = "center">
  <i>Systolic Array</i>
</p>

### im2col + GEMM

When performing a 2D convolution, the data in the convolution window is stored discontinously in memory, which is not efficient for computation. The im2col operation is to expand each convolution **window** into a **column** of matrix B, and expand each convolution **kernel** into a **row** of matrix A. After im2col, the operation of 2D convolution is equivalent to the mutiplication of matrix A and matrix B. The size of matrix A and matrix B is **M \* N** and **N \* K**, where M is the number of convolution kernels, N is the number of weights in a kernel, K is the number of image pixels.

<p align="center">
  <img src ="img/im2col.gif"  width="45%"/>
</p>

## Lab Tasks

The whole design includes following three tasks

1. perform im2col transformation on input image
2. slicing two matrix and load data to buffer
3. perform matrix multiplication using systolic array

You need to implement the first and third tasks (im2col and systolic array).

### Data Storage

All the input and output data are stored on a simulated memory with 32 bits, find the base addresses of each parameter in the below form. Please note that the image, weights and output matrix are stored in "row first", the im2col are stored in "column first".


| Name | Value | Description |
| - | - | - |
| IMG_BASE | 0x00000000 | image base address |
| WEIGHT_BASE | 0x00001000 | weights base address |
| IM2COL_BASE | 0x00002000 | im2col base address |
| OUTPUT_BASE | 0x00003000 | output base address |

<p align="center">
  <img src ="img/mem.png"  width="45%"/>
</p>


### Matrix Slicing

When the size of input matrix is larger than the size of systolic array, the matrix should be first sliced, the width of each slice is the same as the array size. Assume below situation, matrix A and matrix B are divided into 4 slice respectively, we first stream (A_slice0, B_slice0) into the array, and second stream (A_slcie0, B_slice1), and go on... To finish the whole matrix multiplication, each PE needs to do 4x4=16 accumulations. To control the streaming of the input, the module needs two cascaded counter to provide the correct index of slice and pixel.

For the case that the matrix size is not a multiple of the array size, the matrix needs to be appended with zero rows and colunms. This operation is automatically done in the testbench and **you can just assume the integer multiples in your design**.

<p align="center">
  <img src ="img/slicing.png"  width="60%"/>
</p>

## Design Specification

You need to implement three modules as described below using Verilog.

### Module im2col (im2col.v)

This module performs the im2col conversion. You need to read image values from memory and write them back to the proper location. (Here we only consider 3x3 filters.)

#### Parameters

| name | description |
| - | - |
| IMG_W | image width |
| IMG_H | image height |
| DATA_WIDTH | data width |
| ADDR_WIDTH | address width |
| FILTER_SIZE | size of convolution kernel（e.g. 3 means 3x3 kernel） |
| IMG_BASE | image base address |
| IM2COL_BASE | im2col base address |

#### Ports

| name | type | width | description |
| - | - | - | - |
| clk | input | 1 | clock signal |
| rst_im2col | input | 1 | reset signal |
| data_rd | input | DATA_WIDTH | memory read value |
| data_wr | output | DATA_WIDTH | memory write value |
| addr_rd | output | ADDR_WIDTH | memory read address |
| addr_wr | output | ADDR_WIDTH | memory write address |
| im2col_done | output | 1 | oepration done signal |
| mem_wr_en | output | 1 | memory write enable |

#### Design Requirements

1. Begin im2col conversion when `rst_im2col` is pulled down, pull `im2col_done` up when finish.
2. The memory can be read and write once per clock cycle.
3. Use zero-pedding in 2D convolution.

### Module PE (pe.v)

The basic function of PE is calculating the dot products of the rows and columns, and streaming the inputs to its neighbors.

#### Parameters

| name | description |
| - | - |
| N | the number of columns in matrix A |
| MAX_CNT | maximun number of accumulation operations（(M\*K)/(ARR_SIZE\*ARR_SIZE)） |
| DATA_WIDTH | data width |
| ACC_WIDTH | accumulator width |

#### Ports

| name | type | width | description |
| - | - | - | - |
| clk | input | 1 | clock signal |
| rst | input | 1 | reset signal |
| init | input | 1 | signal to clear accumulator |
| in_a | input | DATA_WIDTH | input a |
| in_b | input | DATA_WIDTH | input b |
| out_a | output | DATA_WIDTH | output a |
| out_b | output | DATA_WIDTH | output b |
| out_sum | output | ACC_WIDTH | accumulation output |
| valid_D | output | 1 | output valid |

#### Design Requirements

1. Perform multiply-accumulation on two inputs `in_a` and `in_b` in every clock cycle.
2. Streaming out registered version of `in_a` and `in_b` to `out_a` and `out_b`.
3. Pull up `valid_D` when the dot product is valid.
4. Reset all the output on `rst`.
5. Clear the accumulator and start a new accumulation on `init`.

### Module Systolic Array (systolic.v)

The systolic array is constructed by instantiate PE modules. You need to connect the PEs properly and generate the `init` signal for each of them. You also need to generate the pixel and slice counter as describe before.

#### Parameters

| name | description |
| - | - |
| M | number of rows in matrix A |
| N | number of columns in matrix A |
| K | number of columns in matrix B |
| ARR_SIZE | size of systolic array |
| DATA_WIDTH | data width |
| ACC_WIDTH | accumulator width |

#### Ports

| name | type | width | description |
| - | - | - | - |
| clk | input | 1 | clock signal |
| rst | input | 1 | reset signal |
| enable_row_count_A | input | 1 | allow the slice counter of A to increment |
| A | input | DATA_WIDTH*ARR_SIZE | inputs on the side of matrix A |
| B | input | DATA_WIDTH*ARR_SIZE | inputs on the side of matrix B |
| D | output | ACC_WIDTH\*ARR_SIZE\*ARR_SIZE | connect to each PE's `out_sum` (D[i][j] -> PE[i][j].out_sum) |
| valid_D | output | ARR_SIZE*ARR_SIZE | connect to each PE's `valid_D` (valid_D[i][j] -> PE[i][j].valid_D) |
| pixel_cntr_A | output | 32 | pixel index of matrix A |
| slice_cntr_A | output | 32 | slice index of matrix A |
| pixel_cntr_B | output | 32 | pixel index of matrix B |
| slice_cntr_B | output | 32 | slice index of matrix B |

#### Design Requirements

1. reset all the outputs on `rst`.

## Simulation Environment

### File location

Copy `systolic` folder to path `~/workspace/ics/projects/` in the docker image. The three module files with ports defination are located in `vsrc/src/`

### Run single test

Simply run `make` under folder `~/workspace/ics/projects/systolic`, it will automatically generate inputs and do a test on your design. If you pass the test, the terminal will show something like

<img src="img/make.png" width="95%" align="middle"/>

#### Specifiy parameters

You can specify the parameters like

```
make ARR_SIZE=4 IMG_W=5 IMG_H=4 FILTER_NUM=6 FILTER_SIZE=3
```

#### Enable debug

Set `DEBUG=1` to print more information.

#### Clean

It is recommend to run `make clean` before every simulation.

### Run multiple tests

Give execute permission to the script
```
chmod +x run_test
```

Run multiple tests
```
./run_test
```
The result are saved in file `test/test_result.txt`.

You can add your own cases to `test/testcase.txt`.

## Grading

* Complete im2col (20%)
* Complete systolic array (80%)
  * Fail all the tests (0%)
  * Pass N tests (30% + 2.5% * N)

## Submission
Please compress all the files in your `vsrc/src` folder into a `zip` file with name `EE219_Lab2_{Your_Chinese_Name}.zip`, and submit to BB. The file structure should be like
```
EE219_Lab2_张三/
|-- src
    |-- im2col.v
    |-- pe.v
    |-- systolic.v
    `-- ...
```

## Reference

[1] Yajun, Ha. FPGA-based Hardware System Design (EE116). ShanghaiTech University, 2020
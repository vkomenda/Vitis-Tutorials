Code and files for **module 4** (same instructions as in [module1](../module1_baseline) to setup Vitis and run Vitis Analyzer and Vitis HLS)

> **_In this module..._**<br>
_1> Replicate a compute loop by a programmable factor applied via a templated function_<br>
_2> Use the Vitis HLS <code>dataflow</code> pragma_<br>
_3> Run the full compile and program the card_

<details>
  <summary><b>Click to expand! Learn about the <code>dataflow</code> pragma...</b></summary>
  
The <code>DATAFLOW</code> pragma enables task-level pipelining, allowing functions and loops to overlap in their operation, increasing the concurrency of the register transfer level (RTL) implementation, and increasing the overall throughput of the design.

All operations are performed sequentially in a C description. In the absence of any directives that limit resources (such as pragma HLS allocation), the Vivado High-Level Synthesis (HLS) tool seeks to minimize latency and improve concurrency. However, data dependencies can limit this. For example, functions or loops that access arrays must finish all read/write accesses to the arrays before they complete. This prevents the next function or loop that consumes the data from starting operation. The <code>DATAFLOW</code> optimization enables the operations in a function or loop to start operation before the previous function or loop completes all its operations. 

When the <code>DATAFLOW</code> pragma is specified, the HLS tool analyzes the data flow between sequential functions or loops and creates channels (based on ping pong RAMs or FIFOs) that allow consumer functions or loops to start operation before the producer functions or loops have completed. This allows functions or loops to operate in parallel, which decreases latency and improves the throughput of the RTL.

If no initiation interval (number of cycles between the start of one function or loop and the next) is specified, the HLS tool attempts to minimize the initiation interval and start operation as soon as data is available.

TIP: The config_dataflow command specifies the default memory channel and FIFO depth used in <code>dataflow</code> optimization. Refer to the config_dataflow command in the Vivado Design Suite User Guide: High-Level Synthesis (UG902) for more information.
For the <code>DATAFLOW</code> optimization to work, the data must flow through the design from one task to the next. The following coding styles prevent the HLS tool from performing the <code>DATAFLOW</code> optimization:

   + Single-producer-consumer violations
   + Bypassing tasks
   + Feedback between tasks
   + Conditional execution of tasks
   + Loops with multiple exit conditions

**IMPORTANT**: If any of these coding styles are present, the HLS tool issues a message and does not perform <code>DATAFLOW</code> optimization.

You can use the <code>STABLE</code> pragma to mark variables within <code>DATAFLOW</code> regions to be stable to avoid concurrent read or write of variables.

Finally, the <code>DATAFLOW</code> optimization has no hierarchical implementation. If a sub-function or loop contains additional tasks that might benefit from the optimization, you must apply the optimization to the loop, the sub-function, or inline the sub-function.

**Syntax**

Place the pragma in the C source within the boundaries of the region, function, or loop.

```cpp
#pragma HLS DATAFLOW
```

**Example**

Specifies <code>DATAFLOW</code> optimization within the loop wr_loop_j.

```cpp
wr_loop_j: for (int j = 0; j < TILE_PER_ROW; ++j) {
#pragma HLS DATAFLOW
   wr_buf_loop_m: for (int m = 0; m < HEIGHT; ++m) {
      wr_buf_loop_n: for (int n = 0; n < WIDTH; ++n) {
      #pragma HLS PIPELINE
      // should burst WIDTH in WORD beat
         outFifo >> tile[m][n];
      }
   }
   wr_loop_m: for (int m = 0; m < HEIGHT; ++m) {
      wr_loop_n: for (int n = 0; n < WIDTH; ++n) {
      #pragma HLS PIPELINE
         outx[HEIGHT*TILE_PER_ROW*WIDTH*i+TILE_PER_ROW*WIDTH*m+WIDTH*j+n] = tile[m][n];
      }
   }
}
```
      
</details>


#### Code modifications for the Cholesky kernel

In this module 5 the code for the algorithm is moved into the header file <code>cholesky_kernel.hpp</code>.

There is now an explicit parallelization and the number of parallel compute is determined by <code>NCU</code>, a constant set in <code>cholesky_kernel.cpp</code> through <code>#define NCU 16</code>.

<code>NCU</code> is passed as a template parameter to the <code>chol_col_wrapper</code> function (see below).  The DATAFLOW pragma applies to the loop that calls <code>chol_col</code> 16 times:

```cpp
template <typename T, int N, int NCU>
void chol_col_wrapper(int n, T dataA[NCU][(N + NCU - 1) / NCU][N], T dataj[NCU][N], T tmp1, int j)
{
#pragma HLS DATAFLOW

Loop_row:
    for (int num = 0; num < NCU; num++)
    {
#pragma HLS unroll factor = NCU
        chol_col<T, N, NCU>(n, dataA[num], dataj[num], tmp1, num, j);
    }
}
```

To ensure DATAFLOW is applied the dataA is divided into <code>NCU</code> portions. 

Finally the loop is unrolled with a factor <code>NCU</code> which implies we have <code>NCU</code> (i.e. 16) copies of <code>chol_col</code> created each working on a chunk of the data.

#### Running the design

Same as in module 1:
+ run the hardware emulation
+ run Vitis Analyzer
+ run Vitis HLS and see how it offers a dataflow viewer that can be accessed by right-clicking on the function (<code>chol_col_wrapper</code>) in which dataflow is applied from the synthesis summary report. See [**this animation**](../images/HLS_dataflow_anim.gif) to see how to access it and confirm that the replications were applied.
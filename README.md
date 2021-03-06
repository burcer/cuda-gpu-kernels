# cuda-gpu-kernels
Custom GPU kernels written in CUDA

Includes custom kernels that can perform multiple independent matrix multiply tasks concurrently on GPU. This is functionally similar to cublasSgemmBatched but with no extra kernel launch overhead. This code performs better than cublasSgemmBatched in most cases when matrix sizes are <= ~256 (or more generally when kernel launch overhead is significant compared to 1 matrix multiply task). Note that this code is suboptimal for large matrices (>=1024) and although it can reach up to 78% of peak throughput for large matrices (1.12 TFLOPS on Quadro K2200, where the peak is 1.44 TFLOPS), it is still ~10-15% worse than cublasSgemm for large enough matrices. On the other hand, this code is very efficient when the task consists of many independent matrix multiply jobs. For the extreme case when the matrix sizes are ~32, this code performs ~10x better than cudaSgemmBatched with many matrix multiply tasks. If this code is optimized for better throughput with large matrices, it will always be better than cudaSgemmBatched because it won't have kernel launch overhead, but with only very slight advantage for large matrices since kernel launch overhead is much lower than total compute with large matrices.

Many techniques used here are borrowed from V. Volkov et al., GTC, 2010 [http://www.nvidia.com/content/gtc-2010/pdfs/2238_gtc2010.pdf] and from R.  Nath et al., Int’l J. High Performance Computing Application, 2010 [http://journals.sagepub.com/doi/abs/10.1177/1094342010385729].

Experiments are done on Quadro K2200 (compute capability 5.0). Compiled with nvcc and with -std=c++11 option. CUDA version is 8.0.
OS is Ubuntu 14.04. 

Kernel usage:
Run this to go into the relevant directory:
cd ConcurrMatMul 

Run "make" (optional).
Then you can run ./multiMatrixMul to execute concurrent matrix multiply operation. It completes multiple C=AB matrix multiply tasks.
Command line arguments:

-hA : height of matrix A. Default is 128

-wA : width of matrix A. Default is 128

-wB : width of matrix B. Default is 128

-numOfTasks : number of independent matrix multiply ops you want to run concurrently. Default is 100.

-highCompute : Set this to 1 if your matrices are of size >64, and you have multiple matrix multiply tasks to fill the machine.
If you have less than 10 matrix multiply jobs and your matrices are <=64 in size, you might get better performance if you set this to 0. This options should not affect the end result but it affects performance. Default is 1. 

-blockSize : This is kernel level option. If you set highCompute to 0, this is always 32 and is ignored. Otherwise, options are: 
16, 32, 64, 96, 128. If your matrices are at least of size 128, set this to 128. If you have small matrices, setting this to small values might give better throughput. This dictates how many outputs are produced within 1 thread (more outputs per thread for larger blocks), hence how much instruction-level-parallelism is available. Larger block sizes give better throughput if the overall task is big enough to fill the machine. If it does not fill the machine, smaller block sizes might work better since it parallelizes the work better while comprimising instruction level parallelism. Smaller block sizes are not optimal for large matrices since it doesn't achieve maximum throughput. Default is 128.

-nIter : Number of repetitions for the job (repeats the whole job nIter times). Larger nIter is useful to have a statistically robust performance measure. If you set nIter to 30 and numOfTasks to 100, then it will execute 30 jobs in a for loop, where each
job consists of 100 independent concurrent matrix multiply operations. Default is 30. 

-checkCorrectness : Set to 1 if you would like to have the result from GPU to be compared to result from CPU. The comparison is only done for the first matrix multiply task among all numOfTasks tasks. Set to 0 if you don't want the comparison. Setting this to 1 will result in longer execution. A good method is setting this to 1 for new matrix sizes or numOfTasks when the settings are used the first time to make sure GPU gives correct results; and setting it to 0 when the same settings are used again. Default is 0.

Example usage:
 ./multiMatrixMul -hA 256 -wA 2560 -wB 256 -numOfTasks 100 -blockSize 128 -highCompute 1 -nIter 30 -checkCorrectness 1

Output:
Performance: 1064.3364 GFLOPS, total time: 945.785 msec, total Ops: 1006.633 Gops
Comparison of first matrix multiply passed.

Input:  
./multiMatrixMul -hA 128 -wA 128 -wB 128 -numOfTasks 100 -blockSize 128 -highCompute 1 -nIter 30 -checkCorrectness 0
Output:
Performance: 851.3988 GFLOPS, total time: 14.779 msec, total Ops: 12.583 Gops

Same task above gives 769 GFLOPS with cublasSgemmBatched.

Input:
./multiMatrixMul -hA 32 -wA 32 -wB 32 -numOfTasks 100 -blockSize 32 -highCompute 1 -nIter 30 -checkCorrectness 0
Output:
Performance: 275.5034 GFLOPS, total time: 0.714 msec, total Ops: 0.197 Gops

Same task above gives 30.09 GFLOPS with cublasSgemmBatched.

Notes:
-- If you find that the performance you get is quite lower than what I report here (relative to the peak of your device, of course, not the raw throuhgput number); try being less aggressive in loop unrolling. What I mean by that is try changing  "#pragma unroll 8" into "#pragma unroll" and see if the performance gets better. Since I force compiler to do specific number of unrolls, it might not be optimal in all architectures (it might cause register spilling, for instance).




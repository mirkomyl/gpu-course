# Computing a matrix-vector multiplication

## Objectives

 - Learn how to use shared memory.
 - Learn how to coordinate thread execution.
 - Learn how to manage matrices.

## Remark

The purpose of this hands-on is not to learn the optimal way of computing a
matrix-vector multiplication. The goal is to learn about the shared memory etc.

## Instructions

 1. Carefully read through the `gemv.cu` file. Make sure that you have an idea
    of what each line of code does.

 2. The program requires two arguments. Compile and run the program:
 
    ```
    $ nvcc -o gemv gemv.cu
    $ srun ... ./gemv 800 900
    Residual = 3.865318e-16
    ```
    
    The program does the following:
     
     - A random matrix `A` with `m` rows and `n` columns, and a random vector
       `x` of length `n` are generated and moved to the global memory.
       
     - A kernel `gemv_kernel` computes a vector of length `m` as follows:
     
       ```
       y   <- A * x <=>
       y_k <- A_k0 * x_0 + A_k1 * x_1 + A_k2 * x_2 + ..., k = 0, ..., m-1.
       ```
       
     - The vector `y` is copied to the host memory and validated.
     
    The first program argument defines the height of the matrix `A` and the
    second program argument defines the width of the matrix `A`.
    
    The matrix `A` is stored in column-major format, i.e., the columns are
    stored continuously in the memory. The leading dimension (`ldA`) defines how
    many words (double-precision floating point numbers in this case) are
    allocated for each column. That is, `A[j*ldA+i]` is the element on the
    `i`'th row and the `j`'th column of the matrix.
    
 3. Modify the program such that global memory buffer `d_A` is allocated using
    the `cudaMallocPitch` function and transferred using the `cudaMemcpy2D`
    function:
    
    ```
    cudaError_t cudaMallocPitch (
        void ** devPtr,
        size_t * pitch,
        size_t width,
        size_t height	 
    )
    cudaError_t cudaMemcpy2D (
        void * dst,
        size_t dpitch,
        const void * src,
        size_t spitch,
        size_t width,
        size_t height,
        enum cudaMemcpyKind kind	 
    )	
    ```
    
    Remember, since the matrix is stored in the column-major format, `width` is
    the **height** of the matrix in **bytes** and `height` is the width of the
    matrix. Pitch is the leading dimension of the matrix in **bytes**, i.e.,
    `pitch == ldA * sizeof(double)`.
    
    Compile and test your modified program.

 4. Modify the `gemv_kernel` kernel such that it uses two-dimensional thread
    blocks. For now, use the `x` dimension for computations. Simply make sure
    that all threads that have `threadIdx.y != 0` skip the `if` block. Set the
    thread block size to `THREAD_BLOCK_SIZE x THREAD_BLOCK_SIZE`, where 
    `THREAD_BLOCK_SIZE = 32`:

    ```
    // fix thread block dimensions so that blockDim.x = blockDim.y = warp size
    #define THREAD_BLOCK_SIZE 32
    ```
    
    Compile and test your modified program.

 5. Modify the `gemv_kernel` kernel such that the thread block's `y` dimension
    is used to loop over the columns of the matrix. That is, parallelize the
    `for` loop. Use shared memory to communicate the partial sums.
    
    Remember that all threads must encounter the `__syncthreads()` barrier.
    Therefore, the barrier **cannot** be inside the `if` block! You may have
    to split the `if` block.

    Can you tell why are we using the the thread block indices in this manner?
    Pay attention to how the memory is accessed.
    
    Compile and test your modified program.

    Hint: Allocate `THREAD_BLOCK_SIZE * THREAD_BLOCK_SIZE * sizeof(double)` 
    bytes of shared memory:
 
    ```
    __global__ void gemv_kernel(
    int m, int n, int ldA, double const *A, double const *x, double *y)
    {
        __shared__ double tmp[THREAD_BLOCK_SIZE][THREAD_BLOCK_SIZE];
        
        ....
    }
    ```
    
    Each thread should store its partial sum `v` to `tmp` as follows:
    
    ```
    tmp[threadIdx.x][threadIdx.y] = v;
    ```
    
    For the row `i*THREAD_BLOCK_SIZE+j`, the final result is computed by
    summing together the elements `tmp[j][0]`, `tmp[j][1]`, `...`, 
    and `tmp[j][THREAD_BLOCK_SIZE-1]`.

    Remember, threads that belong to the warp access the memory together.

 6. (challenge) Compare your implementation against cuBLAS:
 
    ```
    cublasStatus_t cublasDgemv(
        cublasHandle_t handle, cublasOperation_t trans,
        int m, int n,
        const double *alpha,
        const double *A, int lda,
        const double *x, int incx,
        const double *beta,
        double       *y, int incy)
    ```
    
    More information: https://docs.nvidia.com/cuda/cublas/index.html#cublas-lt-t-gt-gemv

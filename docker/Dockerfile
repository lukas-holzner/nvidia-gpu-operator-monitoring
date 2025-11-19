# Use NVIDIA CUDA base image
FROM nvidia/cuda:12.6.2-devel-ubuntu22.04 AS builder

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Create working directory
WORKDIR /app

# Create the vector addition CUDA program
RUN cat > vectorAdd.cu << 'EOF'
#include <stdio.h>
#include <cuda_runtime.h>

// CUDA kernel for vector addition
__global__ void vectorAdd(const float *A, const float *B, float *C, int numElements) {
    int i = blockDim.x * blockIdx.x + threadIdx.x;
    if (i < numElements) {
        C[i] = A[i] + B[i];
    }
}

int main(void) {
    int numElements = 50000;
    size_t size = numElements * sizeof(float);
    
    printf("[Vector addition of %d elements]\n", numElements);
    
    // Allocate host memory
    float *h_A = (float *)malloc(size);
    float *h_B = (float *)malloc(size);
    float *h_C = (float *)malloc(size);
    
    if (h_A == NULL || h_B == NULL || h_C == NULL) {
        fprintf(stderr, "Failed to allocate host vectors!\n");
        exit(EXIT_FAILURE);
    }
    
    // Initialize host arrays
    for (int i = 0; i < numElements; ++i) {
        h_A[i] = rand() / (float)RAND_MAX;
        h_B[i] = rand() / (float)RAND_MAX;
    }
    
    // Allocate device memory
    float *d_A = NULL;
    float *d_B = NULL;
    float *d_C = NULL;
    
    cudaError_t err = cudaMalloc((void **)&d_A, size);
    if (err != cudaSuccess) {
        fprintf(stderr, "Failed to allocate device vector A (error code %s)!\n", cudaGetErrorString(err));
        exit(EXIT_FAILURE);
    }
    
    err = cudaMalloc((void **)&d_B, size);
    if (err != cudaSuccess) {
        fprintf(stderr, "Failed to allocate device vector B (error code %s)!\n", cudaGetErrorString(err));
        exit(EXIT_FAILURE);
    }
    
    err = cudaMalloc((void **)&d_C, size);
    if (err != cudaSuccess) {
        fprintf(stderr, "Failed to allocate device vector C (error code %s)!\n", cudaGetErrorString(err));
        exit(EXIT_FAILURE);
    }
    
    // Copy data from host to device
    printf("Copy input data from the host memory to the CUDA device\n");
    err = cudaMemcpy(d_A, h_A, size, cudaMemcpyHostToDevice);
    if (err != cudaSuccess) {
        fprintf(stderr, "Failed to copy vector A from host to device (error code %s)!\n", cudaGetErrorString(err));
        exit(EXIT_FAILURE);
    }
    
    err = cudaMemcpy(d_B, h_B, size, cudaMemcpyHostToDevice);
    if (err != cudaSuccess) {
        fprintf(stderr, "Failed to copy vector B from host to device (error code %s)!\n", cudaGetErrorString(err));
        exit(EXIT_FAILURE);
    }
    
    // Launch the Vector Add CUDA Kernel
    int threadsPerBlock = 256;
    int blocksPerGrid = (numElements + threadsPerBlock - 1) / threadsPerBlock;
    printf("CUDA kernel launch with %d blocks of %d threads\n", blocksPerGrid, threadsPerBlock);
    vectorAdd<<<blocksPerGrid, threadsPerBlock>>>(d_A, d_B, d_C, numElements);
    err = cudaGetLastError();
    if (err != cudaSuccess) {
        fprintf(stderr, "Failed to launch vectorAdd kernel (error code %s)!\n", cudaGetErrorString(err));
        exit(EXIT_FAILURE);
    }
    
    // Copy the result from device to host
    printf("Copy output data from the CUDA device to the host memory\n");
    err = cudaMemcpy(h_C, d_C, size, cudaMemcpyDeviceToHost);
    if (err != cudaSuccess) {
        fprintf(stderr, "Failed to copy vector C from device to host (error code %s)!\n", cudaGetErrorString(err));
        exit(EXIT_FAILURE);
    }
    
    // Verify the result
    for (int i = 0; i < numElements; ++i) {
        if (fabs(h_A[i] + h_B[i] - h_C[i]) > 1e-5) {
            fprintf(stderr, "Result verification failed at element %d!\n", i);
            exit(EXIT_FAILURE);
        }
    }
    
    printf("Test PASSED\n");
    
    // Free device memory
    cudaFree(d_A);
    cudaFree(d_B);
    cudaFree(d_C);
    
    // Free host memory
    free(h_A);
    free(h_B);
    free(h_C);
    
    printf("Done\n");
    return 0;
}
EOF

# Compile the CUDA program
RUN nvcc -o vectorAdd vectorAdd.cu

# Create minimal runtime image
FROM nvidia/cuda:12.6.2-base-ubuntu22.04

# Copy the compiled binary from builder
COPY --from=builder /app/vectorAdd /usr/local/bin/vectorAdd

# Set the entrypoint
ENTRYPOINT ["/usr/local/bin/vectorAdd"]

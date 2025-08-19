# gpu-lite Examples

This directory contains comprehensive examples demonstrating how to use gpu-lite for runtime CUDA compilation and kernel caching.

## Prerequisites

- **Runtime Requirements:**
  - NVIDIA GPU with CUDA capability 3.0+
  - NVIDIA drivers and CUDA SDK installed (check with `nvidia-smi`)
  - Linux operating system

- **Build Requirements:**
  - C++17 compatible compiler (GCC 7+, Clang 5+)
  - CMake 3.12+
  - No CUDA SDK required at build time!

## Examples Overview

### 1. Basic Vector Addition (`basic_vector_add/`)
- **Purpose**: Simple introduction to gpu-lite
- **Features**: Vector addition kernel, performance measurement
- **Concepts**: Basic kernel compilation, memory management, error handling

### 2. Matrix Multiplication (`matrix_multiply/`)
- **Purpose**: More complex computational example
- **Features**: tiled matrix multiplication, CPU comparison
- **Concepts**: Shared memory usage, performance optimization, GFLOPS calculation

### 3. Templated Kernels (`templated_kernels/`)
- **Purpose**: Demonstrates type-specific kernel generation
- **Features**: Multiple data types, template-like kernel generation
- **Concepts**: Runtime type specialization, kernel naming strategies


## Compilation Instructions

### Method 1: Individual Examples (Recommended)

Navigate to any example directory and build:

```bash
# Example: Vector Addition
cd examples/basic_vector_add
mkdir build && cd build
cmake ..
make
./vector_add
```

### Method 2: Build All Examples

From the main gpu-lite directory:

```bash
# Create a main CMakeLists.txt for all examples
mkdir build && cd build
cmake ..
make

# Run examples
./vector_add
./matrix_multiply  
./templated_kernels
```

### Method 3: Direct Compilation (No CMake)

For quick testing without CMake:

```bash
cd examples/basic_vector_add
g++ -std=c++17 -I../.. vector_add.cpp -ldl -o vector_add
./vector_add
```

## Master CMakeLists.txt

Create this file in the main gpu-lite directory to build all examples:

```cmake
cmake_minimum_required(VERSION 3.12)
project(gpu-lite_Examples LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Add all example subdirectories
add_subdirectory(examples/basic_vector_add)
add_subdirectory(examples/matrix_multiply)
add_subdirectory(examples/templated_kernels)
```

## Running the Examples

### Basic Vector Addition
```bash
cd examples/basic_vector_add/build
./vector_add
```

**Expected Output:**
```
Vector Addition Example
Vector size: 1048576 elements
Compiling kernel...
Kernel compiled in: 0 ms
Launching kernel with 4096 blocks of 256 threads
Performing warmup runs...
Running performance benchmark...
Execution time statistics (μs):
  Min: 21.00 μs
  Max: 98.00 μs
  Median: 23.00 μs
  Average: 32.25 μs
Verifying results...
SUCCESS: Vector addition completed correctly!
Memory bandwidth (average): 390.17 GB/s
Memory bandwidth (peak): 599.19 GB/s
```

### Matrix Multiplication
```bash
cd examples/matrix_multiply/build
./matrix_multiply
```

**Expected Output:**
```
Matrix Multiplication Example
Matrix size: 1024x1024
Computing CPU reference...
CPU computation time: 3256 ms
Compiling kernel...
Kernel compiled in: 0 ms
Launching kernel with 64x64 blocks of 16x16 threads
Performing warmup runs...
Running performance benchmark...
Execution time statistics (μs):
  Min: 256.00 μs
  Max: 347.00 μs
  Median: 267.00 μs
  Average: 271.10 μs
Verifying results...
SUCCESS: Matrix multiplication completed correctly!
Maximum error: 1.91e-05
GPU Performance (average): 7921.37 GFLOPS
GPU Performance (peak): 8388.61 GFLOPS
Speedup over CPU (average): 12010.33x
Speedup over CPU (peak): 12718.75x
```

### Templated Kernels
```bash
cd examples/templated_kernels/build
./templated_kernels
```

**Expected Output:**
```
Templated Kernels Example
This example demonstrates how to create type-specific kernels
for different data types using runtime compilation.

=== float Template Example ===
Compiling float kernel: process_array_float
Kernel compiled in: 0 ms
Performing warmup runs...
Running performance benchmark...
Execution time statistics (μs):
  Min: 25.00 μs
  Max: 372.00 μs
  Median: 25.00 μs
  Average: 44.80 μs
SUCCESS: float templated kernel executed correctly!
Memory bandwidth (average): 46.81 GB/s
Memory bandwidth (peak): 83.89 GB/s

=== double Template Example ===
Compiling double kernel: process_array_double
Kernel compiled in: 0 ms
Performing warmup runs...
Running performance benchmark...
Execution time statistics (μs):
  Min: 24.00 μs
  Max: 94.00 μs
  Median: 33.00 μs
  Average: 37.45 μs
SUCCESS: double templated kernel executed correctly!
Memory bandwidth (average): 112.00 GB/s
Memory bandwidth (peak): 174.76 GB/s
```


## Troubleshooting

### "CUDA runtime libraries not available"
```bash
# Check if NVIDIA drivers are installed
nvidia-smi

# Check if CUDA libraries are available
ldconfig -p | grep -E "(cuda|nvrtc)"
```

### Compilation Errors
```bash
# Check compiler version
g++ --version  # Needs 7.0+

# Check CMake version  
cmake --version  # Needs 3.12+

# Verbose build to see detailed errors
make VERBOSE=1
```

### Common Solutions
```bash
# If libraries aren't found, try:
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH

# For compute capability issues, check:
nvidia-smi --query-gpu=compute_cap --format=csv
```

## Extending the Examples

### Adding New Examples
1. Create new directory under `examples/`
2. Write your `.cpp` file
3. Add `CMakeLists.txt`
4. Include the kernel source inline or load from file

### Custom Kernels
```cpp
// Template for new examples
const char* my_kernel = R"(
extern "C" __global__ void my_kernel(float* data, int n) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < n) {
        // Your kernel logic here
        data[idx] = data[idx] * 2.0f;
    }
}
)";

// Compile and cache
auto* kernel = KernelFactory::instance().create(
    "my_kernel",
    my_kernel,
    "my_kernel.cu",
    {"-std=c++17"}
);
```


## Integration with Existing Projects

To integrate gpu-lite examples into your project:

1. Copy the gpu-lite headers to your project
2. Add the example code patterns to your application
3. Link with `-ldl` for dynamic loading
4. No CUDA SDK installation required!

```cmake
# In your existing CMakeLists.txt
target_include_directories(your_app PRIVATE path/to/gpulite/headers)
target_link_libraries(your_app PRIVATE ${CMAKE_DL_LIBS})
```

Happy GPU computing with gpu-lite!
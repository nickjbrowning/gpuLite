# gpu-lite

A lightweight C++ library for dynamic CUDA runtime compilation and kernel caching. gpu-lite simplifies building and deploying CUDA-dependent applications by providing runtime symbol resolution and automated kernel compilation with caching.

**🚀 No CUDA SDK Required at Build Time!** - Compile your applications without installing the CUDA SDK.

## Features

- **No Build-Time CUDA Dependencies**: Compiles without CUDA SDK installed - only requires C++17 compiler
- **Dynamic Symbol Resolution**: Loads CUDA libraries (libcuda.so, libcudart.so, libnvrtc.so) at runtime
- **Runtime Compilation**: Compiles CUDA kernels using NVRTC with automatic compute capability detection
- **Kernel Caching**: Intelligent caching system to avoid recompilation of identical kernels
- **Easy Integration**: Header-only design for simple project integration
- **Cross-Platform Support**: Supports Linux and Windows platforms

## Why gpu-lite?

Traditional CUDA applications require the CUDA SDK to be installed at build time and often have complex deployment requirements. gpu-lite solves this by:

1. **Eliminating build-time CUDA dependencies** - No need for CUDA SDK during compilation
2. **Simplifying deployment** - Applications can run on any system with CUDA drivers installed
3. **Reducing compilation overhead** - Kernels are compiled once and cached for subsequent runs
4. **Providing runtime flexibility** - Kernels can be modified or optimized at runtime

## Quick Start

### Basic Usage

```cpp
#include "gpulite.hpp"

int main() {
    // Check if CUDA is available
    if (!CUDA_DRIVER_INSTANCE.loaded() || !NVRTC_INSTANCE.loaded()) {
        throw std::runtime_error("CUDA runtime not available");
    }

    // Your CUDA kernel code as a string
    const char* kernel_source = R"(
        extern "C" __global__ void vector_add(float* a, float* b, float* c, int n) {
            int idx = blockIdx.x * blockDim.x + threadIdx.x;
            if (idx < n) {
                c[idx] = a[idx] + b[idx];
            }
        }
    )";

    // Create kernel factory and cache the kernel
    auto& factory = KernelFactory::instance();
    auto* kernel = factory.create(
        "vector_add",           // kernel name
        kernel_source,          // kernel source code
        "vector_add.cu",        // virtual source filename
        {"-std=c++17"}          // compilation options
    );

    // Allocate device memory and launch kernel
    float *d_a, *d_b, *d_c;
    int n = 1024;
    
    CUDART_SAFE_CALL(CUDART_INSTANCE.cudaMalloc(reinterpret_cast<void**>(&d_a), n * sizeof(float)));
    CUDART_SAFE_CALL(CUDART_INSTANCE.cudaMalloc(reinterpret_cast<void**>(&d_b), n * sizeof(float)));
    CUDART_SAFE_CALL(CUDART_INSTANCE.cudaMalloc(reinterpret_cast<void**>(&d_c), n * sizeof(float)));

    // Prepare kernel arguments
    std::vector<void*> args = {&d_a, &d_b, &d_c, &n};
    
    // Launch kernel
    kernel->launch(
        dim3((n + 127) / 128),  // grid size
        dim3(128),              // block size
        0,                      // shared memory size
        nullptr,                // stream
        args,                   // kernel arguments
        true                    // synchronize after launch
    );

    // Clean up
    CUDART_SAFE_CALL(CUDART_INSTANCE.cudaFree(d_a));
    CUDART_SAFE_CALL(CUDART_INSTANCE.cudaFree(d_b));
    CUDART_SAFE_CALL(CUDART_INSTANCE.cudaFree(d_c));

    return 0;
}
```

**Compilation:**

**Linux:**
```bash
# Save the above code as main.cpp, then compile:
g++ -std=c++17 main.cpp -ldl -o my_gpu_app

# Run the application:
./my_gpu_app
```

**Windows (with Visual Studio):**
```cmd
# Save the above code as main.cpp, then compile:
cl /std:c++17 main.cpp /Fe:my_gpu_app.exe

# Run the application:
my_gpu_app.exe
```

**Requirements:**
- C++17 compatible compiler (GCC 7+, Clang 5+, MSVC 2017+)
- CUDA SDK installed at run-time, but not at build time!
- Linux or Windows

### Loading Kernels from Files

```cpp
// Load kernel from a .cu file
auto* kernel = KernelFactory::instance().createFromSource(
    "my_kernel",
    "/path/to/kernel.cu",
    "kernel.cu",
    {"-std=c++17"}
);
```

### Template Kernel Names

For templated kernels, use the `getKernelName` helper:

```cpp
// For a templated kernel like: template<typename T> __global__ void process(T* data)
std::string kernel_name = getKernelName<float>("process");
auto* kernel = factory.create(kernel_name, source, "template_kernel.cu", {});
```

## CMake Integration

### Method 1: Header-Only Integration

Add gpu-lite as an include directory to your project:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.12)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)

# Your executable
add_executable(my_app main.cpp)

# Add gpu-lite headers
target_include_directories(my_app PRIVATE external/gpu-lite)

target_link_libraries(my_app PRIVATE gpu-lite)

# Link system libraries required by gpu-lite
target_link_libraries(my_app PRIVATE ${CMAKE_DL_LIBS})
```

Create a simple CMakeLists.txt in the gpu-lite directory:

```cmake
# gpu-lite/CMakeLists.txt
cmake_minimum_required(VERSION 3.12)
project(gpu-lite LANGUAGES CXX)

# Create header-only interface library
add_library(gpu-lite INTERFACE)

# Include directories
target_include_directories(gpu-lite INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:include>
)

# C++17 requirement
target_compile_features(gpu-lite INTERFACE cxx_std_17)

# System dependencies
find_package(Threads REQUIRED)
target_link_libraries(gpu-lite INTERFACE 
    ${CMAKE_DL_LIBS}
    Threads::Threads
)

# Installation rules
install(TARGETS gpu-lite EXPORT gpu-liteConfig)
install(FILES gpulite.hpp DESTINATION include)
install(EXPORT gpu-liteConfig DESTINATION lib/cmake/gpu-lite)
```

### Method 2: Manual Integration

Simply copy the header files to your project and include them:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.12)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)

# Your executable
add_executable(my_app 
    main.cpp
    # Copy this header to your project
    gpu-lite/gpulite.hpp
)

# Link required system libraries
if(WIN32)
    # Windows doesn't need explicit dynamic loading libraries
else()
    target_link_libraries(my_app PRIVATE ${CMAKE_DL_LIBS})
endif()
```

### Method 3: Using FetchContent

```cmake
cmake_minimum_required(VERSION 3.14)
project(MyProject LANGUAGES CXX)

include(FetchContent)

FetchContent_Declare(
    gpu-lite
    GIT_REPOSITORY https://github.com/not-bug-is-feature/gpu-lite.git
    GIT_TAG main
)

FetchContent_MakeAvailable(gpu-lite)

add_executable(my_app main.cpp)
target_link_libraries(my_app PRIVATE gpu-lite)
```

## Advanced Usage

### Custom Compilation Options

```cpp
std::vector<std::string> options = {
    "-std=c++17",                   // C++ standard
    "--use_fast_math",              // Fast math operations
    "-DBLOCK_SIZE=256",             // Preprocessor definitions
    "--maxrregcount=32"             // Maximum register count
};

auto* kernel = factory.create("optimized_kernel", source, "kernel.cu", options);
```

### Kernel Function Attributes

```cpp
// Set maximum dynamic shared memory
kernel->setFuncAttribute(CU_FUNC_ATTRIBUTE_MAX_DYNAMIC_SHARED_SIZE_BYTES, 8192);

// Get kernel register usage
int reg_count = kernel->getFuncAttribute(CU_FUNC_ATTRIBUTE_NUM_REGS);
std::cout << "Kernel uses " << reg_count << " registers per thread" << std::endl;
```

### Asynchronous Execution

```cpp
// Create CUDA stream
cudaStream_t stream;
CUDART_SAFE_CALL(CUDART_INSTANCE.cudaStreamCreate(&stream));

// Launch kernel asynchronously
kernel->launch(
    grid, block, shared_mem_size, 
    reinterpret_cast<void*>(stream), 
    args, 
    false  // don't synchronize
);

// Do other work...

// Synchronize when needed
CUDART_SAFE_CALL(CUDART_INSTANCE.cudaStreamSynchronize(stream));
CUDART_SAFE_CALL(CUDART_INSTANCE.cudaStreamDestroy(stream));
```

## Error Handling

gpu-lite provides comprehensive error checking with detailed error messages:

```cpp
try {
    auto* kernel = factory.create("my_kernel", source, "kernel.cu", {});
    kernel->launch(grid, block, 0, nullptr, args);
} catch (const std::runtime_error& e) {
    std::cerr << "CUDA Error: " << e.what() << std::endl;
    // Error message includes file and line information
}
```

## Requirements

### Runtime Requirements
- NVIDIA GPU with CUDA capability 3.0 or higher
- NVIDIA CUDA drivers installed
- Linux or Windows operating system

### Build Requirements (No CUDA SDK Needed!)
- C++17 compatible compiler (GCC 7+, Clang 5+, MSVC 2017+)
- CMake 3.12 or higher
- Standard C++ library with threading support
- **No CUDA SDK installation required** - gpu-lite uses minimal type wrappers

### CUDA Libraries (loaded dynamically at runtime)

**Linux:**
- `libcuda.so` - CUDA Driver API
- `libcudart.so` - CUDA Runtime API  
- `libnvrtc.so` - NVIDIA Runtime Compilation API

**Windows:**
- `nvcuda.dll` - CUDA Driver API
- `cudart64_*.dll` - CUDA Runtime API
- `nvrtc64_*.dll` - NVIDIA Runtime Compilation API

**Note**: These libraries are only required at runtime, provided by the CUDA SDK/NVIDIA drivers.

## Platform Support

| Platform | Status |
|----------|--------|
| Linux    | ✅ Supported |
| Windows  | ✅ Supported |
| macOS    | ❌ Not applicable (no CUDA support) |

## CUDA/HIP Support

| Platform | Status |
|----------|--------|
| CUDA    | ✅ Supported |
| HIP  | 🚧 Planned |

## Performance Considerations

- **First Launch**: Kernels are compiled on first use, which may add initial latency
- **Subsequent Launches**: Cached kernels launch immediately with minimal overhead
- **Memory Usage**: Compiled kernels are kept in memory for the application lifetime
- **Context Switching**: gpu-lite automatically handles CUDA context management

## Troubleshooting

### "CUDA runtime not available" Error

**Linux:**
- Ensure NVIDIA drivers are installed: `nvidia-smi`
- Verify CUDA libraries are in library path: `ldconfig -p | grep cuda`

**Windows:**
- Ensure NVIDIA drivers are installed: `nvidia-smi`
- Verify CUDA libraries are in system PATH or same directory as executable
- Check that CUDA toolkit is installed and DLLs are accessible

### Compilation Errors
- Check kernel syntax using `nvcc` offline
- Verify compute capability compatibility
- Review compilation options for conflicts

### Runtime Errors
- Enable CUDA error checking in debug builds
- Use `cuda-gdb` and `compute-sanitizer` for kernel debugging
- Check memory alignment and access patterns

## Build Scripts

For convenience, platform-specific build scripts are provided in the `scripts/` directory.

### Linux

```bash
# Run from the gpu-lite root directory
./scripts/build_examples_linux.sh
```

### Windows

```batch
# Run from the gpu-lite root directory
scripts\build_examples_windows.bat
```

## Manual Build Instructions

You can also build manually with CMake:

### Linux
```bash
# Create build directory
mkdir build && cd build

# Generate Makefiles
cmake ..

# Build the project
make -j$(nproc)
```

### Windows
```cmd
# Create build directory
mkdir build
cd build

# Generate Visual Studio project files
cmake .. -G "Visual Studio 16 2019" -A x64

# Build the project
cmake --build . --config Release
```

**Note**: Ensure you have:
- Visual Studio 2017 or later with C++17 support
- CMake 3.12 or later
- NVIDIA drivers installed (CUDA SDK not required at build time)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- NVIDIA CUDA Toolkit documentation
- NVRTC API reference
- https://github.com/NVIDIA/jitify
- Contributors and early adopters

---

For more examples and advanced usage patterns, see the `examples/` directory in this repository.

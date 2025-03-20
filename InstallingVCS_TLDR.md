# STM32 Development with CMake - TLDR

## Quick Setup

### Prerequisites
- CMake (3.14+)
- ARM GCC Toolchain (arm-none-eabi-gcc)
- VS Code with CMake Tools, Cortex-Debug extensions
- J-Link or ST-Link debugger

### Project Structure
```
stm32-project/
├── .vscode/
│   ├── settings.json
│   ├── launch.json
├── src/
│   └── main.c
├── include/
└── CMakeLists.txt
```

### Basic CMakeLists.txt
```cmake
cmake_minimum_required(VERSION 3.14)
include(FetchContent)

# Fetch stm32-cmake
FetchContent_Declare(
    stm32_cmake
    GIT_REPOSITORY https://github.com/ObKo/stm32-cmake.git
    GIT_TAG v2.1.0
    SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR}/board_sdk/stm32-cmake
)
FetchContent_MakeAvailable(stm32_cmake)

# Set toolchain file (before project declaration)
set(CMAKE_TOOLCHAIN_FILE ${CMAKE_CURRENT_LIST_DIR}/board_sdk/stm32-cmake/cmake/stm32_gcc.cmake)

project(YourProjectName C CXX ASM)

# Target STM32 family & fetch HAL/CMSIS
stm32_fetch_cmsis(F4)  # Change to your target family
stm32_fetch_hal(F4)
set(STM32_CHIP STM32F411xE)  # Change to your chip

# Add executable
add_executable(${PROJECT_NAME} 
    src/main.c
    # Add other source files
)

# Link libraries
target_link_libraries(${PROJECT_NAME} STM32::F4::CMSIS STM32::F4::HAL)

# Compiler options
target_compile_definitions(${PROJECT_NAME} PRIVATE -DUSE_HAL_DRIVER -D${STM32_CHIP})
add_compile_options(-g3 -Og -fno-exceptions 
                    $<$<COMPILE_LANGUAGE:CXX>:-fno-rtti> 
                    $<$<COMPILE_LANGUAGE:CXX>:-fno-use-cxa-atexit>)

# Generate binary file
stm32_print_size_of_target(${PROJECT_NAME})
stm32_generate_binary_file(${PROJECT_NAME})
```

### settings.json (in .vscode folder)
```json
{
    "cmake.configureOnOpen": true,
    "cmake.generator": "Ninja",
    "cmake.buildDirectory": "${workspaceFolder}/build",
    "cmake.parallelJobs": 4,
    "C_Cpp.default.configurationProvider": "ms-vscode.cmake-tools"
}
```

### launch.json (in .vscode folder)
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Cortex Debug",
            "cwd": "${workspaceFolder}",
            "executable": "${command:cmake.launchTargetPath}",
            "request": "launch",
            "type": "cortex-debug",
            "runToEntryPoint": "main",
            "servertype": "jlink",  // Change to "stlink" if using ST-Link
            "device": "STM32F411xE", // Change to your chip
            "interface": "swd"
        }
    ]
}
```

## Key Commands
- **Create Project**: `mkdir -p stm32-project/src stm32-project/include stm32-project/.vscode`
- **Configure**: `Cmd+Shift+P` → "CMake: Configure"
- **Select Kit**: `Cmd+Shift+P` → "CMake: Select a Kit" → arm-none-eabi-gcc
- **Build**: `F7` or click Build button in status bar
- **Debug**: Connect debugger hardware and press `F5`

## Common Issues
- **No compiler**: Install ARM GCC and add to PATH
- **Build fails**: Ensure correct STM32 family/chip selection
- **Debug fails**: Check hardware connection and device settings
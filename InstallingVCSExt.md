# STM32 Development Guide with CMake

## Overview
This guide walks through setting up an STM32 microcontroller project using CMake as the build system. The approach leverages the `stm32-cmake` framework to simplify project configuration and automatically fetch dependencies like CMSIS and HAL libraries.

## Prerequisites
- CMake (v3.14 or newer)
- ARM GCC Toolchain installed (arm-none-eabi-gcc)
- VS Code with the following extensions (recommended):
  - [CMake Tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools)
  - [Cortex-Debug](https://marketplace.visualstudio.com/items?itemName=marus25.cortex-debug)
  - [C/C++](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools)
- J-Link or ST-Link debugger for hardware debugging
- Git (for fetching dependencies)

## Project Setup

### 1. Project Initialization
1. Create a new project directory
   ```bash
   mkdir stm32-project
   cd stm32-project
   ```

2. Create the basic directory structure
   ```bash
   mkdir -p src include .vscode
   ```

3. Create the main project files:
   - `CMakeLists.txt` (build configuration)
   - `src/main.c` (your application entry point)
   - `.vscode/settings.json` (VS Code configuration)
   - `.vscode/launch.json` (debugging configuration)

### 2. CMake Configuration

#### Basic CMakeLists.txt Template
```cmake
# Minimum CMake version
cmake_minimum_required(VERSION 3.14)

# Import the stm32-cmake framework
include(FetchContent)
FetchContent_Declare(
    stm32_cmake
    GIT_REPOSITORY https://github.com/ObKo/stm32-cmake.git
    GIT_TAG v2.1.0  # Update to latest version as needed
    SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR}/board_sdk/stm32-cmake
)
FetchContent_MakeAvailable(stm32_cmake)

# Set toolchain file (must come before project() declaration)
set(CMAKE_TOOLCHAIN_FILE ${CMAKE_CURRENT_LIST_DIR}/board_sdk/stm32-cmake/cmake/stm32_gcc.cmake)

# Define your project
project(YourProjectName C CXX ASM)

# Target STM32 family (choose appropriate one)
stm32_fetch_cmsis(F4)  # Change to your target family (F1, F4, L4, etc.)
stm32_fetch_hal(F4)     # Fetches HAL libraries automatically

# Specify the target chip
set(STM32_CHIP STM32F411xE)  # Change to your specific chip

# Add executable
add_executable(${PROJECT_NAME} 
    src/main.c
    src/system_stm32f4xx.c
    # Add other source files here
)

# Link with HAL and CMSIS
target_link_libraries(${PROJECT_NAME} 
    STM32::F4::CMSIS 
    STM32::F4::HAL
)

# Set compiler options
target_compile_definitions(${PROJECT_NAME} PRIVATE
    -DUSE_HAL_DRIVER
    -D${STM32_CHIP}
)

# Compile options to optimize code size and debugging experience
# -g3: Include all debug information
# -Og: Decrease code size, but keep the debugability
# -fno-exceptions: Disables C++ and C exceptions (enabling them requires malloc and heap functionality!)
# $<..>: Enable only when compiling C++ sources
# -fno-rtti: Disable generation of information about every class with virtual functions (saves RAM and flash space)
# -fno-use-cxa-atexit: Disables registration of destructors for static objects (saves RAM)
add_compile_options(-g3
                    -Og
                    -fno-exceptions
                    $<$<COMPILE_LANGUAGE:CXX>:-fno-rtti>
                    $<$<COMPILE_LANGUAGE:CXX>:-fno-use-cxa-atexit>
                    -fmacro-prefix-map=${CMAKE_CURRENT_LIST_DIR}/=/)

# The linker script is handled automatically by stm32-cmake

# Post-build command to create .bin file
stm32_print_size_of_target(${PROJECT_NAME})
stm32_generate_binary_file(${PROJECT_NAME})
```

### 3. Building the Project
1. Create a `build` directory: `mkdir build && cd build`
2. Configure CMake: `cmake ..`
3. Build the project: `cmake --build .`

### 4. IDE Integration (VS Code)

Create the following configuration files in your `.vscode` directory:

#### Opening Workspace Settings
To create or edit your VS Code settings:
1. Open the Command Palette with `Cmd+Shift+P` (macOS) or `Ctrl+Shift+P` (Windows/Linux)
2. Type "Preferences: Open Workspace Settings (JSON)"
3. Select this option to open/create the `.vscode/settings.json` file

#### settings.json
```json
{
    "cmake.configureOnOpen": true,
    "cmake.generator": "Ninja",
    "cmake.buildDirectory": "${workspaceFolder}/build",
    "cmake.parallelJobs": 4,
    "cortex-debug.variableUseNaturalFormat": true,
    "C_Cpp.default.configurationProvider": "ms-vscode.cmake-tools"
}
```

#### launch.json
1. Create this file in the `.vscode` directory
2. Adjust the "device" field to match your specific STM32 chip

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
            "device": "STM32F411xE", // Change to your specific chip
            "interface": "swd",
            "svdFile": "${workspaceFolder}/STM32F4xx.svd", // Path to SVD file for peripheral visualization
            "armToolchainPath": "/usr/local/bin", // Adjust based on your ARM GCC installation path
            "preLaunchTask": "Build", // Optional: automatically build before debugging
            "showDevDebugOutput": "parsed"
        }
    ]
}
```

#### tasks.json (Optional, for build automation)
```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Build",
            "type": "shell",
            "command": "cmake --build ${workspaceFolder}/build",
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "problemMatcher": ["$gcc"]
        },
        {
            "label": "Clean",
            "type": "shell",
            "command": "cmake --build ${workspaceFolder}/build --target clean",
            "problemMatcher": []
        }
    ]
}
```

## Sample main.c
Here's a minimal example to get started with:

```c
#include "stm32f4xx_hal.h"

void SystemClock_Config(void);
static void MX_GPIO_Init(void);

int main(void)
{
  /* Reset of all peripherals, initializes the Flash interface and the Systick. */
  HAL_Init();
  
  /* Configure the system clock */
  SystemClock_Config();

  /* Initialize all configured peripherals */
  MX_GPIO_Init();

  /* Infinite loop */
  while (1)
  {
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);  // Toggle LED on many STM32 boards
    HAL_Delay(500);  // 500ms delay
  }
}

/* System Clock Configuration */
void SystemClock_Config(void)
{
  // Your clock configuration code here
  // This will vary depending on your specific STM32 model
}

/* GPIO Initialization Function */
static void MX_GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  /* GPIO Ports Clock Enable */
  __HAL_RCC_GPIOA_CLK_ENABLE();

  /* Configure GPIO pin Output Level */
  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET);

  /* Configure GPIO pin : PA5 */
  GPIO_InitStruct.Pin = GPIO_PIN_5;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
}
```

## Workflow Tips

### Setting Up Your Project
1. **First-time setup**:
   - Open your project folder in VS Code
   - Let VS Code detect and install recommended extensions
   - Wait for CMake to configure (this may take a while the first time)

2. **Selecting a Kit**:
   - Use `Cmd+Shift+P` (macOS) or `Ctrl+Shift+P` (Windows/Linux)
   - Search for "CMake: Select a Kit" 
   - Choose your ARM GCC installation (e.g., "arm-none-eabi-gcc")

### Building and Debugging
- **Build your project**:
  - Click the "Build" button in the CMake Tools status bar
  - Or use `Cmd+Shift+P` → "CMake: Build"
  - Or use keyboard shortcut `F7`

- **Start debugging**:
  - Connect your debugger (J-Link or ST-Link)
  - Press `F5` to start debugging
  - Or use `Cmd+Shift+P` → "Debug: Start Debugging"

- **CMake commands** (via `Cmd+Shift+P`):
  - "CMake: Configure" - Regenerate build files
  - "CMake: Clean" - Clean build directory
  - "CMake: Build Target" - Build specific target
  - "CMake: Select Variant" - Choose Debug/Release

## Common Issues and Solutions

- **Compiler not found**: 
  - Ensure ARM GCC is in your PATH
  - For macOS: Install via Homebrew with `brew install arm-none-eabi-gcc`
  - Verify installation with `arm-none-eabi-gcc --version`

- **Missing libraries**: 
  - The FetchContent commands should automatically download dependencies
  - Check your internet connection and firewall settings
  - Try manually cloning the repository if FetchContent fails

- **Build errors**:
  - Check that you've selected the correct STM32 family and chip
  - Verify your CMake version (`cmake --version`) meets requirements
  - Look for missing include files or undefined references

- **Debug connection failed**:
  - Verify hardware connections to J-Link/ST-Link
  - Check device settings in launch.json match your actual chip
  - Make sure debugger drivers are installed properly
  - Try restarting VS Code or reconnecting the debugger

- **SVD file not found**:
  - Download the appropriate SVD file for your chip from ST's website
  - Place it in your project directory
  - Update the svdFile path in launch.json

## Advanced Topics

### Custom Memory Configuration
The stm32-cmake framework automatically handles linker scripts for you. However, you can customize memory settings if needed:

```cmake
# Set chip-specific memory layout
set(STM32_FLASH_SIZE 512K)
set(STM32_RAM_SIZE 128K)
```

### Working with STM32CubeMX
If you use STM32CubeMX to generate initialization code:

1. In CubeMX, select "Makefile" as project generator
2. Generate the code to a subdirectory of your project
3. Copy the generated code to your src/ directory
4. Add the files to your CMakeLists.txt
5. Include the generated paths in your include directories

### Flash Memory Sections Configuration
To properly configure memory sections:

```cmake
# Add flash sections configuration
target_compile_definitions(${PROJECT_NAME} PRIVATE
    -DFLASH_BANK_NUMBER=1
    -DRAM_SIZE=128K
    -DFLASH_SIZE=512K
)
```

## Complete Project Structure Example
```
stm32-project/
├── .vscode/
│   ├── settings.json
│   ├── launch.json
│   └── tasks.json
├── board_sdk/         # Auto-generated by FetchContent
│   └── stm32-cmake/
├── build/             # Build output
├── include/           # Header files
│   └── main.h
├── src/               # Source files
│   ├── main.c
│   ├── stm32f4xx_hal_msp.c
│   ├── stm32f4xx_it.c
│   └── system_stm32f4xx.c
└── CMakeLists.txt     # Main build file
```

## Resources
- [stm32-cmake Documentation](https://github.com/ObKo/stm32-cmake)
- [stm32-cmake Examples](https://github.com/ObKo/stm32-cmake/tree/master/examples)
- [CMSIS-HAL Example](https://github.com/ObKo/stm32-cmake/tree/master/examples/fetch-cmsis-hal)
- [STM32 Reference Manuals](https://www.st.com/en/microcontrollers-microprocessors/stm32-32-bit-arm-cortex-mcus.html)
- [ARM GCC Toolchain](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm)
- [CMake Documentation](https://cmake.org/documentation/)
- [VS Code Cortex-Debug Extension](https://github.com/Marus/cortex-debug/wiki)

## Additional Tools
- [STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html) - GUI for STM32 configuration
- [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html) - Alternative IDE based on Eclipse
- [OpenOCD](http://openocd.org/) - Open-source debugging for ARM
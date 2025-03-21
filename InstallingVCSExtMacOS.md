# STM32 Development Guide with CMake

## Overview
This guide walks through setting up an STM32 microcontroller project using CMake as the build system. The approach leverages the `stm32-cmake` framework to simplify project configuration and automatically fetch dependencies like CMSIS and HAL libraries.

Note: this version was made for MacOS (15.3.2 (24D81) on Apple M3 hardware)

## Prerequisites
- CMake (v3.14 or newer)
- ARM GCC Toolchain installed (arm-none-eabi-gcc)
- VS Code with the following extensions (recommended):
  - [CMake Tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools)
  - [Cortex-Debug](https://marketplace.visualstudio.com/items?itemName=marus25.cortex-debug)
  - [C/C++](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools)
- ST-Link, J-Link, or other compatible debugger
- Git (for fetching dependencies)
- OpenOCD (for debugging)

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
   - `src/sensorhub_main.c` (your application entry point)
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
project(SensorHub C CXX ASM)

# Target STM32 family
stm32_fetch_cmsis(F4)  # Change to your target family (F1, F4, L4, etc.)
stm32_fetch_hal(F4)    # Fetches HAL libraries automatically

# Specify the target chip
set(STM32_CHIP STM32F405RG)  # Change to your specific chip

# Find required packages
find_package(CMSIS COMPONENTS ${STM32_CHIP} REQUIRED)
find_package(HAL COMPONENTS STM32F4 CORTEX RCC GPIO REQUIRED)

# Define source files
set(PROJECT_SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/src/sensorhub_main.c
    # Add other source files here
)

# Add executable
add_executable(${PROJECT_NAME} ${PROJECT_SOURCES})

# Include directories
target_include_directories(${PROJECT_NAME} PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    # Add other include directories as needed
)

# Link with HAL and CMSIS
target_link_libraries(${PROJECT_NAME}
    HAL::STM32::F4::GPIO
    HAL::STM32::F4::RCC
    HAL::STM32::F4::CORTEX
    STM32::NoSys
    # Add other libraries as needed
)

# Set compiler options
target_compile_definitions(${PROJECT_NAME} PRIVATE
    USE_HAL_DRIVER
    STM32F405xx  # Use the xx suffix format for device family
)

# Compile options to optimize code size and debugging experience
add_compile_options(-g3
                    -Og
                    -fno-exceptions
                    $<$<COMPILE_LANGUAGE:CXX>:-fno-rtti>
                    $<$<COMPILE_LANGUAGE:CXX>:-fno-use-cxa-atexit>
                    -fmacro-prefix-map=${CMAKE_CURRENT_LIST_DIR}/=/)

# Post-build commands
stm32_print_size_of_target(${PROJECT_NAME})
stm32_generate_binary_file(${PROJECT_NAME})
```

### 3. Building the Project
1. Create a `build` directory: `mkdir build && cd build`
2. Configure CMake: `cmake ..`
3. Build the project: `cmake --build .`

### 4. IDE Integration (VS Code)

Create the following configuration files in your `.vscode` directory:

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
Adjust this file based on your specific STM32 chip and debugger:

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
            "servertype": "openocd",  // Change to "jlink" if using J-Link
            "device": "STM32F405RG", 
            "configFiles": [
                "interface/stlink.cfg",  // Use stlink-v2.cfg for older ST-Link devices
                "target/stm32f4x.cfg"
            ],
            "interface": "swd",
            "svdFile": "${workspaceFolder}/STM32F405.svd", // Path to SVD file
            "armToolchainPath": "/usr/local/bin", // Adjust based on your ARM GCC installation
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

## Sample main.c for STM32F405RG
Here's a basic blinky example to test your setup:

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
    HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);  // Toggle LED on STM32F405RG boards
    HAL_Delay(500);  // 500ms delay
  }
}

/* System Clock Configuration */
void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  /* Configure the main internal regulator output voltage */
  __HAL_RCC_PWR_CLK_ENABLE();
  __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE1);

  /* Initialize the CPU, AHB and APB busses clocks */
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
  RCC_OscInitStruct.HSEState = RCC_HSE_ON;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
  RCC_OscInitStruct.PLL.PLLM = 8;
  RCC_OscInitStruct.PLL.PLLN = 336;
  RCC_OscInitStruct.PLL.PLLP = RCC_PLLP_DIV2;
  RCC_OscInitStruct.PLL.PLLQ = 7;
  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  /* Initialize the CPU, AHB and APB busses clocks */
  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                              |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV4;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_5) != HAL_OK)
  {
    Error_Handler();
  }
}

/* GPIO Initialization Function */
static void MX_GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  /* GPIO Ports Clock Enable */
  __HAL_RCC_GPIOC_CLK_ENABLE();

  /* Configure GPIO pin Output Level */
  HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET);

  /* Configure GPIO pin : PC13 (LED) */
  GPIO_InitStruct.Pin = GPIO_PIN_13;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);
}

/* Error Handler */
void Error_Handler(void)
{
  /* User can add code here to handle errors */
  while(1);
}

#ifdef  USE_FULL_ASSERT
void assert_failed(uint8_t *file, uint32_t line)
{ 
  /* User can add his own implementation to report the file name and line number,
     e.g., printf("Wrong parameters value: file %s on line %d\r\n", file, line) */
}
#endif /* USE_FULL_ASSERT */
```

## Important Configuration Notes

### Setting the Correct Device Definitions
When working with STM32, you must use the correct device definition format:

- For STM32F405RG, the correct definition is `STM32F405xx` (not `STM32F405RG`)
- Always use the format `STM32[Family][Subfamily]xx` where:
  - `Family` is F0, F1, F2, F3, F4, etc.
  - `Subfamily` is the specific model number (405, 411, etc.)

Using the correct format in the `target_compile_definitions` section is essential:

```cmake
target_compile_definitions(${PROJECT_NAME} PRIVATE
    USE_HAL_DRIVER
    STM32F405xx  # Correct format with xx suffix
)
```

### OpenOCD Configuration
If OpenOCD shows a successful connection like:

```
Info : device id = 0x100f6413
Info : flash size = 1024 KiB
...
[stm32f4x.cpu] halted due to debug-request, current mode: Thread 
xPSR: 0x01000000 pc: 0x080008b0 msp: 0x20020000
```

Your debugger is properly connected and recognized. Common OpenOCD messages:

- **Speed warnings**: "Unable to match requested speed 2000 kHz, using 1000 kHz" is normal
- **Target voltage**: Check that it's in the expected range (3.0-3.3V typically)
- **Device ID**: Verifies correct chip detection

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
  - Connect your debugger
  - Press `F5` to start debugging
  - Or use `Cmd+Shift+P` → "Debug: Start Debugging"

## Common Issues and Solutions

- **"Please select first the target STM32F4xx device" error**:
  - Make sure you've defined the correct device subfamily with the xx suffix
  - For STM32F405RG, use `STM32F405xx` in your target_compile_definitions

- **"Error: flash size = 512 bytes" message**: 
  - This is usually not an error but a message about a flash page size
  - If your device ID and flash size (e.g., "flash size = 1024 KiB") are correct, this isn't a problem

- **Compiler not found**: 
  - Ensure ARM GCC is in your PATH
  - Verify installation with `arm-none-eabi-gcc --version`

- **Debug connection failed**:
  - Verify hardware connections to your debugger
  - Check device settings in launch.json match your actual chip
  - Try using a different USB port or cable

## Advanced Topics

### Connecting External Peripherals and Sensors

When connecting external components to your STM32F405RG:

1. **I2C Devices**:
   - Connect to I2C1 (PB6/PB7) or I2C2 (PB10/PB11)
   - Don't forget to add pull-up resistors (typically 4.7kΩ)
   - Add the HAL I2C component to your CMake:
     ```cmake
     find_package(HAL COMPONENTS STM32F4 I2C REQUIRED)
     target_link_libraries(${PROJECT_NAME} HAL::STM32::F4::I2C)
     ```

2. **SPI Devices**:
   - Use SPI1 (PA5/PA6/PA7) or SPI2 (PB13/PB14/PB15)
   - Configure chip select pins as GPIO outputs
   - Add the HAL SPI component:
     ```cmake
     find_package(HAL COMPONENTS STM32F4 SPI REQUIRED)
     target_link_libraries(${PROJECT_NAME} HAL::STM32::F4::SPI)
     ```

3. **Analog Sensors**:
   - Connect to ADC pins (PA0-PA7, PB0-PB1, PC0-PC5)
   - Add the HAL ADC component:
     ```cmake
     find_package(HAL COMPONENTS STM32F4 ADC REQUIRED)
     target_link_libraries(${PROJECT_NAME} HAL::STM32::F4::ADC)
     ```

### Memory Configuration for STM32F405RG

The STM32F405RG has:
- 1MB Flash memory
- 192KB RAM (128KB main + 64KB CCM)

For optimal performance, configure your build:

```cmake
# Set chip-specific memory layout if needed
set(STM32_FLASH_SIZE 1024K)
set(STM32_RAM_SIZE 192K)
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
│   ├── sensorhub_main.c
│   ├── stm32f4xx_hal_msp.c
│   ├── stm32f4xx_it.c
│   └── system_stm32f4xx.c
├── STM32F405.svd      # SVD file for debugging
└── CMakeLists.txt     # Main build file
```

## Resources
- [stm32-cmake Documentation](https://github.com/ObKo/stm32-cmake)
- [STM32F405xx Reference Manual](https://www.st.com/resource/en/reference_manual/dm00031020-stm32f405-415-stm32f407-417-stm32f427-437-and-stm32f429-439-advanced-arm-based-32-bit-mcus-stmicroelectronics.pdf)
- [STM32F405xx Datasheet](https://www.st.com/resource/en/datasheet/stm32f405rg.pdf)
- [OpenOCD Documentation](http://openocd.org/doc/html/index.html)
- [ARM GCC Toolchain](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm)

## Additional Tips for STM32F405RG

- **High-speed performance**: The STM32F405RG can run at up to 168 MHz
- **Floating-point unit**: Use the FPU for faster float calculations by adding `-mfpu=fpv4-sp-d16 -mfloat-abi=hard` to your compiler flags
- **Debugging with SWO**: Connect the SWO pin to your debugger for real-time variable watching and printf-style debugging
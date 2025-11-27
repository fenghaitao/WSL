# Building WSL

<cite>
**Referenced Files in This Document**   
- [README.md](file://README.md)
- [CMakeLists.txt](file://CMakeLists.txt)
- [dev-loop.md](file://doc/docs/dev-loop.md)
- [FindIDL.cmake](file://cmake/FindIDL.cmake)
- [FindLINUXBUILD.cmake](file://cmake/FindLINUXBUILD.cmake)
- [build-bundle.bat](file://tools/build-bundle.bat)
- [deploy-to-host.ps1](file://tools/deploy/deploy-to-host.ps1)
- [create-dev-cert.ps1](file://tools/create-dev-cert.ps1)
</cite>

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [CMake-Based Build Process](#cmake-based-build-process)
3. [Platform-Specific Build Options](#platform-specific-build-options)
4. [Building MSIX Bundles](#building-msix-bundles)
5. [CMake Find Modules](#cmake-find-modules)
6. [Development Optimization](#development-optimization)
7. [Troubleshooting Common Build Issues](#troubleshooting-common-build-issues)

## Prerequisites

To build WSL from source, several prerequisites must be installed and configured on your development machine. The build system relies on CMake and Visual Studio with specific components to ensure compatibility with the Windows Subsystem for Linux architecture.

First, ensure you have **CMake 3.25 or later** installed. This version is required as specified in the root CMakeLists.txt file (`cmake_minimum_required(VERSION 3.25)`). You can install CMake using the Windows Package Manager: `winget install Kitware.CMake`.

Next, install **Visual Studio** with the following required components:
- Windows SDK 26100 (explicitly required in CMakeLists.txt via `set(CMAKE_SYSTEM_VERSION 10.0.26100.0)`)
- MSVC v143 - VS 2022 C++ ARM64 build tools (Latest + Spectre)
- Universal Windows Platform support for v143 build tools
- C++ Clang compiler for Windows (required for cross-compiling Linux binaries)
- .NET desktop development tools
- .NET WinUI app development tools
- C++ core features and ATL support

Additionally, symbolic link support is required for the build process. This can be enabled by activating **Developer Mode** in Windows Settings under "Privacy & security" → "For developers". Alternatively, you can run the build process with Administrator privileges to bypass this requirement.

**Section sources**
- [dev-loop.md](file://doc/docs/dev-loop.md#L3-L20)
- [CMakeLists.txt](file://CMakeLists.txt#L1-L2)

## CMake-Based Build Process

The WSL build process is orchestrated through CMake, which generates the necessary project files for compilation. After cloning the repository and ensuring all prerequisites are met, navigate to the repository root directory and execute the CMake configuration command.

To generate the Visual Studio solution, run:
```
cmake .
```

This command processes the root CMakeLists.txt file and generates a `wsl.sln` solution file that can be opened in Visual Studio for building, debugging, and development. Alternatively, you can build directly from the command line using:
```
cmake --build .
```

The build system defaults to a Debug configuration if no build type is specified. The CMakeLists.txt file explicitly sets this default with `set(CMAKE_BUILD_TYPE "Debug")` when `CMAKE_BUILD_TYPE` is not defined. The generated binaries are placed in the `bin/<platform>/<target>` directory structure, where `<platform>` is either `x64` or `arm64`, and `<target>` is either `Debug` or `Release`.

The build process involves several stages:
1. Fetching external dependencies via `FetchContent_Declare` and `FetchContent_MakeAvailable`
2. Locating required packages through `find_package` calls for IDL, LINUXBUILD, NUGET, VERSION, MC, and Appx
3. Restoring NuGet packages needed for the build
4. Configuring compiler and linker flags for both Windows and Linux components
5. Building subprojects defined in the various CMakeLists.txt files throughout the source tree

**Section sources**
- [dev-loop.md](file://doc/docs/dev-loop.md#L22-L30)
- [CMakeLists.txt](file://CMakeLists.txt#L1-L447)

## Platform-Specific Build Options

The WSL build system supports multiple platform configurations through CMake command-line options. These options allow developers to target different architectures and build configurations as needed for development and testing.

To build for **ARM64 architecture**, use the `-A arm64` flag:
```
cmake . -A arm64
```

This sets the `TARGET_PLATFORM` variable to "arm64" in the CMakeLists.txt file, which affects various build settings including compiler flags, NuGet package selection, and output directories. The build system explicitly checks for supported platforms and will fail with "Unsupported platform" if an invalid platform is specified.

For **release builds**, specify the build type using:
```
cmake . -DCMAKE_BUILD_TYPE=Release
```

This configures the build with release-specific compiler flags such as `/Zi` for debug information, `/guard:cf` for Control Flow Guard security feature, and `/Qspectre` for Spectre mitigation. Release builds also enable CETCOMPAT (Control-flow Enforcement Technology) on x64 platforms for enhanced security.

The build system automatically detects the target platform from the CMake generator platform and sets appropriate compile definitions:
- `_AMD64_` for x64 builds
- `_ARM64_` for ARM64 builds

These definitions are used throughout the codebase to conditionally compile architecture-specific code.

**Section sources**
- [dev-loop.md](file://doc/docs/dev-loop.md#L32-L35)
- [CMakeLists.txt](file://CMakeLists.txt#L7-L15)

## Building MSIX Bundles

WSL supports building MSIX bundle packages that combine both x64 and ARM64 versions into a single distributable package. This is particularly useful for publishing WSL to the Microsoft Store or distributing it across different device architectures.

To build an MSIX bundle, use the `-DBUILD_BUNDLE=TRUE` CMake option:
```
cmake . -DBUILD_BUNDLE=TRUE
```

However, building a bundle requires a specific build sequence because it combines both ARM64 and x64 components. The process first builds the ARM64 version, then the x64 version with the bundle flag. This sequence is automated in the `build-bundle.bat` script, which:
1. Cleans the CMake cache and dependencies
2. Builds the ARM64 configuration
3. Cleans again to prepare for x64 build
4. Builds the x64 configuration with `BUILD_BUNDLE=TRUE`

The bundle creation is handled in the msixinstaller CMakeLists.txt file, which uses `makeappx.exe bundle` to combine the individual MSIX packages. The resulting bundle file includes both platform versions and can be installed on any Windows device regardless of architecture.

The build system also generates a development certificate for package signing using the `create-dev-cert.ps1` PowerShell script when `PACKAGE_CERTIFICATE` does not exist. This ensures that the generated MSIX packages can be installed without requiring a commercial code-signing certificate during development.

**Section sources**
- [dev-loop.md](file://doc/docs/dev-loop.md#L36)
- [build-bundle.bat](file://tools/build-bundle.bat#L1-L14)
- [CMakeLists.txt](file://CMakeLists.txt#L162-L168)

## CMake Find Modules

The WSL build system utilizes custom CMake find modules to locate and configure dependencies required for the build process. These modules are located in the `cmake/` directory and provide specialized functionality for handling WSL-specific build requirements.

The **FindIDL.cmake** module handles Interface Definition Language (IDL) file processing, which is essential for COM interface generation. It defines the `add_idl()` function that invokes the MIDL compiler to generate C++ headers, interface stubs, and proxy code from IDL files. The module configures MIDL with appropriate flags including:
- `/target NT100` for Windows 10/11 targeting
- `/env` set according to the target platform (x64 or ARM64)
- `/Zp8` for 8-byte structure packing
- `/char unsigned` for unsigned character type

The **FindLINUXBUILD.cmake** module is critical for cross-compiling the Linux components of WSL. It defines functions for building Linux executables and libraries using Clang with musl libc. The module configures the Linux toolchain with:
- `LINUX_CC` and `LINUX_CXX` pointing to Clang and Clang++
- Appropriate sysroot and include paths to the Linux SDK
- Target specification for either x86_64-unknown-linux-musl or aarch64-unknown-linux-musl
- Static linking configuration with musl libc and libc++abi

These find modules are integrated into the main build process through `find_package()` calls in the root CMakeLists.txt file, ensuring that the required tools and configurations are available before building any components that depend on them.

**Section sources**
- [FindIDL.cmake](file://cmake/FindIDL.cmake#L1-L67)
- [FindLINUXBUILD.cmake](file://cmake/FindLINUXBUILD.cmake#L1-L104)
- [CMakeLists.txt](file://CMakeLists.txt#L45-L50)

## Development Optimization

For faster development cycles, the WSL repository provides several optimization techniques and scripts to reduce build times and streamline the development workflow.

The `dev-loop.md` documentation recommends using `UserConfig.cmake` to customize the build process for development. This file, if present, is included in the build configuration and can override default settings to create a faster development experience. While the specific contents of this file are not in version control (it's typically user-specific), it can be used to:
- Skip certain build steps
- Use pre-built dependencies
- Configure faster build options

The repository also includes the `build-bundle.bat` script for automating the complex process of building MSIX bundles, which requires building both ARM64 and x64 configurations in sequence. This script handles the necessary cleanup and reconfiguration between builds, saving developers from manually managing the build state.

For deployment during development, the `deploy-to-host.ps1` PowerShell script provides a convenient way to install the built MSI package. This script requires administrator privileges (as indicated by the `#Requires -RunAsAdministrator` directive) and uses `msiexec.exe` to silently install the package with appropriate arguments.

Additionally, the build system supports development shortcuts through environment variables like `WSL_DEV_BINARY_PATH`, which can be used to point to pre-built binary components and avoid rebuilding everything during development.

**Section sources**
- [dev-loop.md](file://doc/docs/dev-loop.md#L38)
- [build-bundle.bat](file://tools/build-bundle.bat#L1-L14)
- [deploy-to-host.ps1](file://tools/deploy/deploy-to-host.ps1#L1-L46)

## Troubleshooting Common Build Issues

Several common build failures can occur when building WSL from source, typically related to missing dependencies, incorrect toolchain configuration, or environmental issues.

**Missing Windows SDK**: If the build fails with "Incorrect Windows SDK version", ensure you have Windows SDK 26100 installed. The CMakeLists.txt file explicitly checks for this version with `if (NOT ${CMAKE_VS_WINDOWS_TARGET_PLATFORM_VERSION} STREQUAL ${CMAKE_SYSTEM_VERSION})`. Install the correct SDK version through the Visual Studio Installer.

**Missing Clang compiler**: The error "C++ Clang Compiler for Windows is not installed" indicates that the LLVM tools are missing. This occurs when the Visual Studio component "C++ Clang compiler for Windows" is not installed. Install this component through the Visual Studio Installer, as it's required for cross-compiling the Linux components of WSL.

**Visual Studio installation detection**: The build may fail with "Could not determine Visual Studio 2022 installation directory" if `vswhere.exe` cannot locate a valid Visual Studio 2022 installation. Ensure Visual Studio 2022 is installed and that the `vswhere` NuGet package has been restored properly.

**Symbolic link issues**: If you encounter errors related to file linking or access, ensure Developer Mode is enabled in Windows Settings, or run the build process with Administrator privileges. The build creates symbolic links for certain dependencies, which requires appropriate permissions.

**NuGet package restoration**: Build failures related to missing headers or libraries may indicate that NuGet packages were not properly restored. The `restore_nuget_packages()` function in CMakeLists.txt handles this, but network issues or corrupted caches can interfere. Try cleaning the `_deps` directory and rebuilding.

**Certificate creation failures**: When building MSIX packages, the `create-dev-cert.ps1` script may fail to generate the development certificate. Ensure PowerShell execution policies allow script execution (`Set-ExecutionPolicy RemoteSigned`), and that you have write permissions to the generated directory.

**Section sources**
- [CMakeLists.txt](file://CMakeLists.txt#L18-L20)
- [CMakeLists.txt](file://CMakeLists.txt#L277-L289)
- [create-dev-cert.ps1](file://tools/create-dev-cert.ps1)
- [dev-loop.md](file://doc/docs/dev-loop.md#L20)
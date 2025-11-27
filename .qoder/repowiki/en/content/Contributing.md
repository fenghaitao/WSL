# Contributing

<cite>
**Referenced Files in This Document**   
- [CONTRIBUTING.md](file://CONTRIBUTING.md)
- [CODE_OF_CONDUCT.md](file://CODE_OF_CONDUCT.md)
- [DATA_AND_PRIVACY.md](file://DATA_AND_PRIVACY.md)
- [dev-loop.md](file://doc/docs/dev-loop.md)
- [collect-wsl-logs.ps1](file://diagnostics/collect-wsl-logs.ps1)
- [validate-copyright-headers.py](file://tools/devops/validate-copyright-headers.py)
- [validate-localization.py](file://tools/devops/validate-localization.py)
- [generateLocalizationHeader.ps1](file://tools/generateLocalizationHeader.ps1)
- [test/README.md](file://test/README.md)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Contribution Process](#contribution-process)
3. [Development Environment Setup](#development-environment-setup)
4. [Code Changes and Testing](#code-changes-and-testing)
5. [Coding Standards and Validation](#coding-standards-and-validation)
6. [Localization Practices](#localization-practices)
7. [Policies and Guidelines](#policies-and-guidelines)
8. [Contribution Workflows](#contribution-workflows)
9. [Testing Guidelines](#testing-guidelines)
10. [Frequently Asked Questions](#frequently-asked-questions)

## Introduction

This document provides comprehensive guidance for contributing to the Windows Subsystem for Linux (WSL) project. It covers the complete contribution lifecycle from setting up a development environment to submitting pull requests. The WSL project welcomes contributions of all types, including feature development, bug fixes, documentation improvements, and design proposals. This guide is designed to help both first-time contributors and experienced developers navigate the codebase and contribution processes effectively.

**Section sources**
- [CONTRIBUTING.md](file://CONTRIBUTING.md)
- [README.md](file://README.md)

## Contribution Process

The WSL contribution process follows a structured workflow to ensure code quality and maintain project integrity. Contributors should begin by filing an issue or feature request in the repository to discuss their proposed changes before implementation. This allows the WSL team to provide feedback and ensure alignment with project goals. Once the proposal is approved, contributors can proceed with development.

After implementing changes, contributors should submit a pull request to the repository. Most contributions require agreement to a Contributor License Agreement (CLA) that grants Microsoft the rights to use the contribution. The WSL team will review the pull request, provide feedback, and eventually merge it if it meets the project's standards.

For bug reporting, contributors should search for existing issues first and upvote or comment on them if applicable. Different repositories are used for different types of issues: the main WSL repository for general technical issues, microsoftdocs/wsl for documentation issues, and microsoft/wslg for Linux GUI app issues.

**Section sources**
- [CONTRIBUTING.md](file://CONTRIBUTING.md#L1-L30)
- [README.md](file://README.md#L29-L36)

## Development Environment Setup

Setting up a development environment for WSL requires specific tools and configurations. The build system uses CMake, requiring version 2.25 or higher, which can be installed via `winget install Kitware.CMake`. Visual Studio is required with specific components including Windows SDK 26100, MSBuild, Universal Windows Platform support for v143 build tools (X64 and ARM64), MSVC v143 C++ ARM64 build tools, C++ core features, C++ ATL for latest v143 tools, C++ Clang compiler for Windows, .NET desktop development, and .NET WinUI app development tools.

Building WSL requires support for symbolic links, which can be enabled by activating Developer Mode in Windows Settings or running the build process with administrator privileges. After cloning the repository, generate the Visual Studio solution by running `cmake .`, which creates a `wsl.sln` file that can be built with Visual Studio or via `cmake --build .`.

Build parameters include:
- `cmake . -A arm64`: Build a package for ARM64
- `cmake . -DCMAKE_BUILD_TYPE=Release`: Build for release
- `cmake . -DBUILD_BUNDLE=TRUE`: Build a bundle msix package (requires building ARM64 first)

**Section sources**
- [dev-loop.md](file://doc/docs/dev-loop.md#L1-L37)

## Code Changes and Testing

After building WSL, developers can deploy the application by installing the MSI package found under `bin\<platform>\<target>\wsl.msi` or by running `powershell tools\deploy\deploy-to-host.ps1`. For Hyper-V virtual machine deployment, use `powershell tools\deploy\deploy-to-vm.ps1 -VmName <vm> -Username <username> -Password <password>`.

Unit tests can be executed by running `bin\<platform>\<target>\test.bat`. Due to the extensive test suite, it's recommended to run specific test subsets. For example, `bin\<platform>\<target>\test.bat /name:*UnitTest*` runs unit tests, while `bin\<platform>\<target>\test.bat /name:<class>::<test>` runs a specific test case. To run WSL1 tests, add the `-Version 1` parameter.

After the first test run, the `-f` flag can be used to skip package installation, making subsequent tests faster. This requires setting `test_distro` as the default WSL distribution using `wsl --set-default test_distro`.

For debugging tests, the `/waitfordebugger` parameter can be used with `test.bat` to attach a debugger to the unit test process. The `/breakonfailure` parameter automatically breaks on the first test failure.

**Section sources**
- [dev-loop.md](file://doc/docs/dev-loop.md#L41-L75)
- [test/README.md](file://test/README.md#L1-L162)

## Coding Standards and Validation

The WSL project enforces strict coding standards through validation scripts that run as part of the build process. All source files (C, C++, H, etc.) must include a copyright header. The validation script `tools/devops/validate-copyright-headers.py` checks for the presence of the required copyright notice containing "Copyright (c) Microsoft. All rights reserved."

The script examines the first 50 lines of each source file, looking for the copyright notice in comments (both single-line `//` and multi-line `/* */` formats). Files missing the required header will fail validation. The script can automatically generate headers with the `--fix` parameter, creating a standardized header with module name and abstract sections.

Code formatting follows CMake and Visual Studio conventions, with build parallelization enabled through the Directory.Build.Props file, which sets `UseMultiToolTask` and `EnforceProcessCountAcrossBuilds` properties to optimize build performance across multiple processors.

**Section sources**
- [validate-copyright-headers.py](file://tools/devops/validate-copyright-headers.py#L1-L83)
- [Directory.Build.Props](file://Directory.Build.Props#L1-L11)

## Localization Practices

WSL implements a comprehensive localization system to support multiple languages. Localization resources are stored in the `localization/strings` directory with subdirectories for each language (e.g., `en-US`, `de-DE`, `ja-JP`). Each language directory contains a `Resources.resw` file with string resources in XML format.

The `tools/generateLocalizationHeader.ps1` script generates the C++ `Localization.h` header file from these resources. This script processes all language files, creating a C++ class with static methods for each string resource. The generated code includes template functions for strings with format parameters and handles both Unicode (Windows) and UTF-8 (Linux) character types.

Localization validation is enforced by `tools/devops/validate-localization.py`, which ensures consistency across language files. The script verifies that:
- All string resources have the same number of format inserts (braces) across all languages
- Line endings use CRLF (Windows format)
- String comments include proper metadata about non-translatable elements

The validation script specifically checks for "locked" strings (command-line arguments, file names, and technical terms) that should not be translated, ensuring they remain consistent across localizations.

**Section sources**
- [generateLocalizationHeader.ps1](file://tools/generateLocalizationHeader.ps1#L1-L179)
- [validate-localization.py](file://tools/devops/validate-localization.py#L1-L208)

## Policies and Guidelines

### Data Collection and Privacy

WSL collects diagnostic data through Windows telemetry to understand feature usage, monitor stability, and assess performance. This data collection can be disabled through Windows Settings by navigating to Privacy and Security -> Diagnostics & Feedback and disabling 'Diagnostic data'. Users can also view all diagnostic data being sent through the 'View diagnostic data' option.

Specifically, WSL telemetry includes:
- **Usage data**: Information about which features and settings are most frequently used
- **Stability data**: Information about bugs and system crashes to prioritize urgent issues
- **Performance data**: Metrics about WSL runtime performance to identify slowdowns

Telemetry events in the source code can be identified by searching for calls to `WSL_LOG_TELEMETRY`. This transparency allows contributors to understand exactly what data is collected and when.

**Section sources**
- [DATA_AND_PRIVACY.md](file://DATA_AND_PRIVACY.md#L1-L18)

### Code of Conduct

The WSL project adheres to the Microsoft Open Source Code of Conduct, which is based on the Contributor Covenant. This code of conduct outlines expectations for behavior in project spaces, including communication channels, issue trackers, and pull requests. It emphasizes creating a harassment-free experience for everyone regardless of personal characteristics.

The code of conduct applies to all project spaces and to public spaces when an individual is representing the project. Instances of abusive, harassing, or otherwise unacceptable behavior may be reported to the project team at opencode@microsoft.com. The full code of conduct is available at https://opensource.microsoft.com/codeofconduct/.

**Section sources**
- [CODE_OF_CONDUCT.md](file://CODE_OF_CONDUCT.md#L1-L10)

## Contribution Workflows

### Bug Fixing

When fixing bugs, contributors should first reproduce the issue and collect relevant logs using the diagnostic scripts in the `diagnostics/` directory. The `collect-wsl-logs.ps1` script gathers comprehensive system information including registry settings, service status, and Windows version details. For networking issues, `collect-networking-logs.ps1` captures network-related diagnostics.

For crashes, different procedures apply:
- **Windows crashes (BSODs)**: Collect kernel crash dumps by enabling `AlwaysKeepMemoryDump` registry setting, then send the MEMORY.DMP file to secure@microsoft.com
- **WSL process crashes**: Use `collect-wsl-logs.ps1 -Dump` to collect user-mode crash dumps, or enable automatic crash dump collection to C:\crashes

### Feature Development

For new features, contributors should follow the standard development cycle:
1. Discuss the feature in an issue to get team feedback
2. Set up the development environment as described
3. Implement the feature with appropriate tests
4. Ensure compliance with coding standards and localization requirements
5. Submit a pull request with a clear description of changes

### Documentation Improvements

Documentation for WSL is maintained in the `doc/docs/` directory using Markdown format. The `mkdocs.yml` configuration file defines the site structure. Technical documentation is organized in the `technical-documentation/` subdirectory with separate files for each major component.

When improving documentation, contributors should ensure consistency with existing terminology and follow the established structure. User documentation is maintained separately in the microsoftdocs/wsl repository.

**Section sources**
- [CONTRIBUTING.md](file://CONTRIBUTING.md#L8-L30)
- [diagnostics/collect-wsl-logs.ps1](file://diagnostics/collect-wsl-logs.ps1#L1-L172)
- [doc/README.md](file://doc/README.md)

## Testing Guidelines

WSL uses the Test Authoring and Execution Framework (TAEF) for testing. Tests are created as C++ classes using the WEX testing framework and compiled into DLLs. The test infrastructure is defined in `test/CMakeLists.txt`, which builds the test binaries.

To execute tests, use the `TE.exe` binary from the Microsoft.Taef nuget package. Tests should be run from an administrative command prompt in the build output directory (`bin/<X64|Arm64>/<Debug|Release>/`). Useful command-line parameters include:
- `/list`: Lists all tests in the specified DLL
- `/name:<testname>`: Runs specific tests or test patterns
- `/inproc`: Runs tests in the same process for easier debugging
- `/breakOnCreate`, `/breakOnError`, `/breakOnInvoke`: Debugging breakpoints
- `/p:<paramName>=<value>`: Passes runtime parameters to tests
- `/runas:<RunAsType>`: Specifies execution environment
- `/sessionTimeout:<value>`: Sets execution timeout

The test suite includes several categories:
- **SimpleTests**: Basic connectivity tests for commands like `wsl echo`
- **MountTests**: Tests for `wsl --mount` functionality
- **NetworkTests**: Tests for networking features and configurations
- **Plan9Tests**: Tests for the Plan 9 filesystem component
- **UnitTests**: Tests for Linux behavior and WSL-specific features

**Section sources**
- [test/README.md](file://test/README.md#L1-L162)

## Frequently Asked Questions

### Code Review Process
Pull requests are reviewed by the WSL team, with feedback provided through GitHub comments. Reviews focus on code quality, adherence to standards, test coverage, and alignment with project architecture. Contributors should expect multiple rounds of feedback and be prepared to make revisions.

### Merge Criteria
For a pull request to be merged, it must:
- Pass all automated validation checks (copyright headers, localization, etc.)
- Include appropriate tests for new functionality
- Maintain or improve code quality
- Be approved by at least one maintainer
- Not introduce regressions in existing functionality

### Release Cycles
WSL follows a regular release cycle with updates delivered through Windows Update. The project uses GitHub releases to document changes. Major features and significant bug fixes are typically included in Windows Insider builds before being released to the general public.

### First-Time Contributor Tips
- Start with "good first issue" labeled tasks
- Engage in issue discussions before implementing
- Follow the coding standards strictly
- Include tests for all changes
- Be responsive to review feedback
- Use the diagnostic tools to provide comprehensive bug reports

**Section sources**
- [CONTRIBUTING.md](file://CONTRIBUTING.md)
- [test/README.md](file://test/README.md)
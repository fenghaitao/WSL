# Lifecycle Management

<cite>
**Referenced Files in This Document**
- [Lifetime.cpp](file://src/windows/service/exe/Lifetime.cpp)
- [Lifetime.h](file://src/windows/service/exe/Lifetime.h)
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp)
- [WslCoreVm.h](file://src/windows/service/exe/WslCoreVm.h)
- [telemetry.cpp](file://src/linux/init/telemetry.cpp)
- [hcs.cpp](file://src/windows/common/hcs.cpp)
- [hcs.hpp](file://src/windows/common/hcs.hpp)
- [collect-wsl-logs.ps1](file://diagnostics/collect-wsl-logs.ps1)
- [GuestTelemetryLogger.cpp](file://src/windows/service/exe/GuestTelemetryLogger.cpp)
- [GuestTelemetryLogger.h](file://src/windows/service/exe/GuestTelemetryLogger.h)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [System Architecture Overview](#system-architecture-overview)
3. [VM Lifecycle Management](#vm-lifecycle-management)
4. [Distribution Lifecycle Management](#distribution-lifecycle-management)
5. [Telemetry and Monitoring](#telemetry-and-monitoring)
6. [State Machine Transitions](#state-machine-transitions)
7. [API Interfaces and Programmatic Control](#api-interfaces-and-programmatic-control)
8. [Common Issues and Debugging](#common-issues-and-debugging)
9. [Practical Examples](#practical-examples)
10. [Troubleshooting Guide](#troubleshooting-guide)

## Introduction

WSL (Windows Subsystem for Linux) lifecycle management encompasses the complete management of virtual machines and Linux distributions throughout their operational life. This system handles VM creation, startup, runtime monitoring, suspension, and termination through sophisticated state management and HCS (Host Compute Service) integration.

The lifecycle management system consists of several key components:
- **VM Management**: Core virtual machine lifecycle through WslCoreVm
- **Distribution Management**: Linux distribution lifecycle coordination
- **Lifetime Management**: Process and client lifecycle tracking
- **Telemetry Systems**: Comprehensive monitoring and reporting
- **HCS Integration**: Host compute service communication

## System Architecture Overview

The WSL lifecycle management system follows a layered architecture with clear separation of concerns:

```mermaid
graph TB
subgraph "User Layer"
CLI[WSL Command Line]
API[WSL Plugin API]
end
subgraph "Service Layer"
WslService[WSL Service]
LifetimeMgr[Lifetime Manager]
DistMgr[Distribution Manager]
end
subgraph "VM Layer"
WslCoreVm[WslCoreVm]
HCS[Host Compute Service]
MiniInit[Mini Init]
end
subgraph "Guest Layer"
GuestKernel[Guest Kernel]
TelemetryAgent[Telemetry Agent]
Apps[User Applications]
end
CLI --> WslService
API --> WslService
WslService --> LifetimeMgr
WslService --> DistMgr
DistMgr --> WslCoreVm
WslCoreVm --> HCS
WslCoreVm --> MiniInit
MiniInit --> GuestKernel
GuestKernel --> TelemetryAgent
Apps --> TelemetryAgent
```

**Diagram sources**
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp#L103-L166)
- [Lifetime.cpp](file://src/windows/service/exe/Lifetime.cpp#L32-L42)

**Section sources**
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp#L1-L50)
- [Lifetime.cpp](file://src/windows/service/exe/Lifetime.cpp#L1-L30)

## VM Lifecycle Management

### VM Creation and Initialization

VM creation involves multiple phases of initialization through the WslCoreVm class:

```mermaid
sequenceDiagram
participant Client as Client Process
participant Service as WSL Service
participant WslCoreVm as WslCoreVm
participant HCS as Host Compute Service
participant VM as Virtual Machine
Client->>Service : Create VM Request
Service->>WslCoreVm : Create(VmConfig)
WslCoreVm->>WslCoreVm : Initialize(VmId, UserToken)
WslCoreVm->>HCS : CreateComputeSystem()
HCS->>VM : Create VM Instance
WslCoreVm->>HCS : StartComputeSystem()
VM->>WslCoreVm : Boot Complete
WslCoreVm->>WslCoreVm : InitializeGuest()
WslCoreVm->>Service : VM Ready
Service->>Client : VM Created Successfully
```

**Diagram sources**
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp#L108-L166)
- [hcs.cpp](file://src/windows/common/hcs.cpp#L79-L96)

The VM creation process includes:

1. **Configuration Validation**: Verifies VM configuration parameters
2. **Resource Allocation**: Sets up temporary directories and paths
3. **HCS Integration**: Creates compute system through Host Compute Service
4. **Guest Initialization**: Initializes the Linux guest environment
5. **Network Setup**: Configures networking infrastructure
6. **Telemetry Setup**: Establishes telemetry collection channels

**Section sources**
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp#L168-L372)

### VM Startup Process

VM startup follows a structured initialization sequence:

```mermaid
flowchart TD
Start([VM Startup Initiated]) --> ValidateConfig["Validate Configuration"]
ValidateConfig --> SetupPaths["Setup Paths & Directories"]
SetupPaths --> CreateHCS["Create HCS Compute System"]
CreateHCS --> StartVM["Start VM"]
StartVM --> WaitBoot["Wait for Boot Completion"]
WaitBoot --> InitGuest["Initialize Guest Environment"]
InitGuest --> SetupNetwork["Setup Networking"]
SetupNetwork --> MountDisks["Mount Disks & Shares"]
MountDisks --> SetupTelemetry["Setup Telemetry"]
SetupTelemetry --> Ready([VM Ready])
StartVM --> ErrorCheck{"Startup Error?"}
ErrorCheck --> |Yes| Cleanup["Cleanup Resources"]
ErrorCheck --> |No| WaitBoot
Cleanup --> Failure([Startup Failed])
```

**Diagram sources**
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp#L168-L372)

**Section sources**
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp#L168-L372)

### VM Termination and Shutdown

VM termination implements graceful shutdown with fallback mechanisms:

```mermaid
sequenceDiagram
participant Service as WSL Service
participant WslCoreVm as WslCoreVm
participant MiniInit as Mini Init
participant HCS as Host Compute Service
participant VM as Virtual Machine
Service->>WslCoreVm : Terminate Request
WslCoreVm->>MiniInit : Close Connection
MiniInit->>VM : Graceful Shutdown Signal
VM->>MiniInit : Shutdown Complete
MiniInit->>WslCoreVm : Exit Notification
WslCoreVm->>WslCoreVm : Wait for Exit Event
alt Normal Shutdown (within timeout)
WslCoreVm->>HCS : Wait for VM Exit
HCS->>VM : Monitor Exit Status
VM->>HCS : Exit Event
HCS->>WslCoreVm : Exit Confirmation
else Forced Termination
WslCoreVm->>WslCoreVm : Timeout Reached
WslCoreVm->>HCS : TerminateComputeSystem()
HCS->>VM : Force Terminate
VM->>HCS : Force Exit
end
WslCoreVm->>Service : Termination Complete
```

**Diagram sources**
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp#L740-L774)

**Section sources**
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp#L740-L774)

## Distribution Lifecycle Management

### Distribution Registration and Management

Distributions are managed through a registration system that tracks their lifecycle:

```mermaid
classDiagram
class Distribution {
+GUID Id
+string Name
+string Path
+DistributionState State
+Create() bool
+Start() bool
+Stop() bool
+Terminate() bool
}
class DistributionRegistration {
+Register(Distribution) bool
+Unregister(GUID) bool
+OpenDefault() optional~Distribution~
+SetDefault(Distribution) void
+ListAll() vector~Distribution~
}
class LxssUserSession {
+CreateInstance() Instance
+TerminateInstance() bool
+GetRunningInstances() map~GUID,Instance~
}
class Instance {
+GUID Id
+GUID DistroId
+ProcessHandle Process
+InstanceState State
+GetLifetimeManagerId() ULONG64
}
Distribution --> DistributionRegistration : registered_with
DistributionRegistration --> LxssUserSession : manages
LxssUserSession --> Instance : creates
```

**Diagram sources**
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp#L75-L83)
- [Lifetime.cpp](file://src/windows/service/exe/Lifetime.cpp#L19-L30)

**Section sources**
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp#L75-L83)

### Distribution State Transitions

Distributions follow specific state transitions during their lifecycle:

```mermaid
stateDiagram-v2
[*] --> Registered : Distribution Created
Registered --> Starting : Start Request
Starting --> Running : Boot Complete
Running --> Stopping : Stop Request
Stopping --> Stopped : Graceful Shutdown
Stopped --> Starting : Restart Request
Running --> Terminated : Force Terminate
Stopped --> Terminated : Force Terminate
Terminated --> [*] : Cleanup
```

**Section sources**
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp#L3535-L3576)

## Telemetry and Monitoring

### Telemetry Collection Architecture

WSL implements comprehensive telemetry collection through multiple channels:

```mermaid
graph LR
subgraph "Guest Side"
TelemetryAgent[Telemetry Agent]
ProcessMonitor[Process Monitor]
DmesgCollector[Dmesg Collector]
end
subgraph "Host Side"
GuestLogger[Guest Telemetry Logger]
ServiceTelemetry[Service Telemetry]
HCTelemetry[HCS Telemetry]
end
subgraph "Storage"
TelemetryFiles[Telemetry Files]
EventLogs[Windows Event Logs]
CrashDumps[Crash Dumps]
end
TelemetryAgent --> GuestLogger
ProcessMonitor --> GuestLogger
DmesgCollector --> GuestLogger
GuestLogger --> ServiceTelemetry
ServiceTelemetry --> HCTelemetry
HCTelemetry --> TelemetryFiles
HCTelemetry --> EventLogs
ServiceTelemetry --> CrashDumps
```

**Diagram sources**
- [telemetry.cpp](file://src/linux/init/telemetry.cpp#L127-L170)
- [GuestTelemetryLogger.cpp](file://src/windows/service/exe/GuestTelemetryLogger.cpp#L44-L84)

**Section sources**
- [telemetry.cpp](file://src/linux/init/telemetry.cpp#L127-L282)
- [GuestTelemetryLogger.cpp](file://src/windows/service/exe/GuestTelemetryLogger.cpp#L1-L96)

### Telemetry Reporting Mechanisms

The telemetry system captures various lifecycle events:

| Event Category | Events Tracked | Data Collected |
|----------------|----------------|----------------|
| VM Lifecycle | CreateVmBegin, CreateVmEnd, FailedToStartVm | VM ID, Time to create, Error codes |
| Distribution Lifecycle | Exec, ExecCritical | Binary names, execution counts |
| Network Events | WslCoreVmInitialize | Networking mode, firewall status |
| Performance Metrics | TerminateVmStart, TerminateVm | Termination timing, success/failure |

**Section sources**
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp#L116-L141)
- [telemetry.cpp](file://src/linux/init/telemetry.cpp#L218-L267)

## State Machine Transitions

### VM State Machine

The VM state machine manages transitions through distinct phases:

```mermaid
stateDiagram-v2
[*] --> Creating : Create VM Request
Creating --> Initializing : HCS Create Success
Initializing --> Booting : Start VM
Booting --> Running : Guest Ready
Running --> Suspending : Suspend Request
Suspending --> Suspended : Suspend Complete
Suspended --> Resuming : Resume Request
Resuming --> Running : Resume Complete
Running --> Terminating : Terminate Request
Terminating --> Terminated : VM Exited
Terminated --> [*] : Cleanup
Booting --> Failed : Boot Error
Initializing --> Failed : Init Error
Failed --> [*] : Cleanup
```

### Distribution State Machine

Distributions manage their own state transitions:

```mermaid
stateDiagram-v2
[*] --> Installing : Install Distribution
Installing --> Configuring : Install Complete
Configuring --> Ready : Configure Complete
Ready --> Starting : Start Request
Starting --> Running : Boot Complete
Running --> Stopping : Stop Request
Stopping --> Stopped : Graceful Shutdown
Stopped --> Starting : Restart Request
Running --> Crashed : Exception/Error
Crashed --> Starting : Restart Request
Stopped --> Removing : Remove Request
Removing --> [*] : Cleanup
```

## API Interfaces and Programmatic Control

### Public Interfaces

WSL provides several interfaces for programmatic lifecycle control:

#### WSL Service API

The primary interface for VM and distribution management:

```cpp
// VM Creation
static std::unique_ptr<WslCoreVm> Create(
    _In_ const wil::shared_handle& UserToken, 
    _In_ wsl::core::Config&& VmConfig, 
    _In_ const GUID& VmId);

// Instance Management  
std::shared_ptr<LxssRunningInstance> CreateInstance(
    _In_ const GUID& InstanceId,
    _In_ const LXSS_DISTRO_CONFIGURATION& Configuration,
    _In_ LX_MESSAGE_TYPE MessageType,
    _In_ DWORD ReceiveTimeout = 0);

// Lifecycle Operations
void RegisterCallbacks(
    _In_ const std::function<void(ULONG)>& DistroExitCallback = {},
    _In_ const std::function<void(GUID)>& TerminationCallback = {});
```

#### Lifetime Management API

Process and client lifecycle tracking:

```cpp
// Client Registration
ULONG64 GetRegistrationId();
bool IsAnyProcessRegistered(_In_ ULONG64 ClientKey);
void RegisterCallback(
    _In_ ULONG64 ClientKey, 
    _In_ const std::function<bool(void)>& Callback, 
    _In_opt_ HANDLE ClientProcess, 
    _In_ DWORD TimeoutMs);

// Cleanup Operations
bool RemoveCallback(_In_ ULONG64 ClientKey);
void ClearCallbacks();
```

**Section sources**
- [WslCoreVm.h](file://src/windows/service/exe/WslCoreVm.h#L59-L83)
- [Lifetime.h](file://src/windows/service/exe/Lifetime.h#L28-L38)

### Command-Line Tools

WSL provides comprehensive command-line tools for lifecycle management:

| Command | Purpose | Parameters |
|---------|---------|------------|
| `wsl --shutdown` | Terminate all VMs | None |
| `wsl --terminate <distro>` | Terminate specific distribution | Distribution name or GUID |
| `wsl --suspend` | Suspend VM | None |
| `wsl --list --verbose` | List VMs with details | None |
| `wsl --status` | Show VM status | None |

## Common Issues and Debugging

### Failed VM Startup

Common causes and solutions for VM startup failures:

#### Kernel Panic Detection

The system detects kernel panics through HVSocket errors:

```mermaid
flowchart TD
Start([VM Startup]) --> CheckSocket["Check HVSocket Connection"]
CheckSocket --> SocketError{"Socket Error?"}
SocketError --> |Yes| WaitCrash["Wait for Crash Event (1s)"]
SocketError --> |No| Continue["Continue Startup"]
WaitCrash --> HasCrash{"Crash Event Received?"}
HasCrash --> |Yes| CheckLogFile{"Crash Log Available?"}
HasCrash --> |No| GenericError["Generic HVSocket Error"]
CheckLogFile --> |Yes| ShowStackTrace["Show Stack Trace"]
CheckLogFile --> |No| ShowBasicError["Show Basic Error"]
ShowStackTrace --> Error([VM Crashed])
ShowBasicError --> Error
GenericError --> Error
Continue --> Success([Startup Successful])
```

**Diagram sources**
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp#L134-L162)

#### Common Startup Issues

| Issue | Symptoms | Solution |
|-------|----------|----------|
| HVSocket Connection Failed | WSASocket error | Check Virtual Machine Platform feature |
| Kernel Panic | HVSocket timeout | Review crash logs using collect-wsl-logs.ps1 |
| Insufficient Memory | VM creation fails | Increase allocated memory in wsl.conf |
| Corrupted VHD | Mount failures | Repair or recreate distribution |

**Section sources**
- [WslCoreVm.cpp](file://src/windows/service/exe/WslCoreVm.cpp#L134-L162)

### Shutdown Hangs and Timeouts

#### Shutdown Behavior Options

The system implements different shutdown behaviors:

```mermaid
flowchart TD
ShutdownRequest([Shutdown Request]) --> CheckBehavior{"Shutdown Behavior"}
CheckBehavior --> |Force| ForceTerminate["Immediate Termination"]
CheckBehavior --> |ForceAfter30Seconds| TryLock["Try Lock with 30s Timeout"]
CheckBehavior --> |Graceful| GracefulShutdown["Graceful Shutdown"]
TryLock --> LockSuccess{"Lock Acquired?"}
LockSuccess --> |Yes| GracefulShutdown
LockSuccess --> |No| ForceTerminate
ForceTerminate --> ImmediateKill["Kill All Processes"]
GracefulShutdown --> WaitExit["Wait for VM Exit"]
WaitExit --> Timeout{"Timeout Reached?"}
Timeout --> |No| Success([Shutdown Complete])
Timeout --> |Yes| ForceTerminate
ImmediateKill --> ForceSuccess([Forced Shutdown])
```

**Diagram sources**
- [LxssUserSession.cpp](file://src/windows/service/exe/LxssUserSession.cpp#L2086-L2125)

#### Common Shutdown Issues

| Issue | Cause | Resolution |
|-------|-------|------------|
| Hanging Shutdown | Long-running processes | Use `--shutdown --force` |
| Timeout Errors | Network delays | Increase timeout values |
| Partial Cleanup | Resource locks | Check for hung processes |

**Section sources**
- [LxssUserSession.cpp](file://src/windows/service/exe/LxssUserSession.cpp#L2086-L2125)

## Practical Examples

### Normal VM Lifecycle Example

```cpp
// VM Creation Example
auto userToken = GetCurrentProcessToken();
wsl::core::Config vmConfig;
vmConfig.MemorySize = 4096; // 4GB RAM
vmConfig.ProcessorCount = 2;

auto vm = WslCoreVm::Create(userToken, std::move(vmConfig), vmGuid);
if (vm) {
    // VM ready for use
    auto instance = vm->CreateInstance(instanceGuid, distroConfig);
    // Use the instance...
}
```

### Graceful Shutdown Example

```cpp
// Graceful shutdown with timeout
auto vm = WslCoreVm::Open(vmGuid);
if (vm) {
    // Register termination callback
    vm->RegisterCallbacks({}, [](GUID runtimeId) {
        // VM terminated
        Log("VM terminated: " + GuidToString(runtimeId));
    });
    
    // Initiate graceful shutdown
    // VM will automatically terminate after timeout
}
```

### Lifetime Management Example

```cpp
// Client registration for cleanup
LifetimeManager lifetimeMgr;
auto clientKey = lifetimeMgr.GetRegistrationId();

lifetimeMgr.RegisterCallback(
    clientKey,
    []() {
        // Cleanup function
        CleanupResources();
        return true; // Success
    },
    GetCurrentProcess(),
    30000 // 30 second timeout
);
```

### Telemetry Collection Example

```cpp
// Telemetry monitoring
auto telemetryLogger = GuestTelemetryLogger::Create(vmId, exitEvent);
auto pipeName = telemetryLogger->GetPipeName();

// Telemetry data flows through named pipe
// Process telemetry events in real-time
```

## Troubleshooting Guide

### Diagnostic Tools

#### collect-wsl-logs.ps1

The primary diagnostic tool for collecting comprehensive WSL logs:

```powershell
# Basic log collection
.\collect-wsl-logs.ps1

# Storage-specific profiling
.\collect-wsl-logs.ps1 -LogProfile storage

# HVSocket debugging
.\collect-wsl-logs.ps1 -LogProfile hvsocket
```

#### Time Travel Debugging

For advanced debugging scenarios:

1. **Enable WPR Profiling**: Use the PowerShell script to enable Windows Performance Recorder
2. **Reproduce Issue**: Perform actions that trigger the problem
3. **Collect Logs**: Stop profiling and extract logs
4. **Analyze ETL Files**: Use Windows Performance Analyzer for timeline analysis

#### Common Debugging Scenarios

| Scenario | Tool | Approach |
|----------|------|----------|
| VM Startup Issues | collect-wsl-logs.ps1 | Check HCS logs and crash dumps |
| Performance Problems | WPA (Windows Performance Analyzer) | Analyze ETL timeline |
| Network Connectivity | Network traces | Enable hvsocket profiling |
| Process Hangs | Process Explorer | Check for hung processes |

### Recovery Procedures

#### VM Recovery

1. **Automatic Recovery**: System attempts automatic recovery for transient failures
2. **Manual Recovery**: Use `wsl --shutdown` followed by restart
3. **Full Reset**: Remove corrupted VM state and recreate

#### Distribution Recovery

1. **Repair Distribution**: Use `wsl --unregister` followed by reinstall
2. **Backup Restoration**: Restore from backup if available
3. **Fresh Installation**: Clean installation as last resort

### Performance Optimization

#### Memory Management

- Monitor VM memory usage through telemetry
- Adjust memory allocation based on workload
- Enable memory reclaim for idle VMs

#### CPU Optimization

- Allocate appropriate processor count
- Monitor CPU utilization patterns
- Consider NUMA topology for multi-core systems

#### Storage Performance

- Use SSD storage for VM disks
- Enable VirtioFS for improved file system performance
- Monitor disk I/O patterns through telemetry

**Section sources**
- [collect-wsl-logs.ps1](file://diagnostics/collect-wsl-logs.ps1#L1-L172)

## Conclusion

WSL lifecycle management provides a robust framework for managing virtual machines and Linux distributions throughout their operational life. The system combines sophisticated state management, comprehensive telemetry, and flexible API interfaces to deliver reliable and performant containerized Linux environments on Windows.

Key strengths of the system include:

- **Robust State Management**: Well-defined state machines for VM and distribution lifecycle
- **Comprehensive Telemetry**: Multi-layered monitoring and reporting capabilities
- **Flexible API Design**: Extensible interfaces for programmatic control
- **Advanced Diagnostics**: Powerful debugging and troubleshooting tools
- **Graceful Error Handling**: Sophisticated error recovery and fallback mechanisms

The lifecycle management system continues to evolve with new features and improvements, maintaining backward compatibility while adding modern capabilities for enterprise and development scenarios.
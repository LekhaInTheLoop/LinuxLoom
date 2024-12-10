# A Prototype Job Worker Service

A simple and secure solution for managing and running jobs, with strong control over resources, secure communication, and an easy-to-use interface

---

## Table of Contents

- [Overview](#overview)  
- [Core Components](#core-components)  
  - [Process Management Library](#1-process-management-library)  
  - [Resource Management with cgroups](#2-resource-management-with-cgroups)  
  - [gRPC API](#3-grpc-api)  
  - [Command-Line Interface (CLI)](#4-command-line-interface-cli)  
- [Security Features](#security-features)  
- [Development and Testing](#development-and-testing)  
- [Conclusion](#conclusion)  

---

## Overview

The Prototype Job Worker Service provides an efficient framework for scheduling, monitoring, and managing Linux jobs. It incorporates the following features:  

- A reusable **core library** for job lifecycle management.  
- **Resource control** using cgroups for CPU, memory, and disk I/O.  
- A secure **gRPC API** for remote job management.  
- An intuitive **CLI** for easy job control.

---

## Core Components

### **1. Process Management Library**

The core library handles job lifecycle operations and ensures resource restrictions using Linux cgroups.

**Responsibilities**:
- Manage job lifecycle: start, stop, and query jobs.
- Stream and store stdout/stderr logs to disk.
- Enforce resource limits (CPU, memory, disk I/O) using cgroups.

**Core Structures**:
```go
type Command struct {
    Name           string         // Program name
    Args           []string       // Arguments for the program
    ResourceConfig *ResourceConfig // Resource limits for the job
}

type ResourceConfig struct {
    CPUQuota    string // e.g., "50%" for limiting CPU usage
    MemoryLimit string // e.g., "256M" for limiting memory usage
    DiskIO      string // e.g., "10MB/s" for I/O throttling
}

type Job struct {
    ID          string     // Unique job identifier
    Cmd         *exec.Cmd  // Process command
    Status      *Status    // Process status
    CgroupPath  string     // Path to the cgroup for the job
}
```

**Resource Management Workflow**:
1. **Cgroup Setup**:
   - A unique cgroup is created for each job under `/sys/fs/cgroup`.
   - Resource limits are applied as per the `ResourceConfig`.
2. **Job Execution**:
   - The process is started and assigned to its cgroup.
3. **Cleanup**:
   - On job completion, the cgroup is deleted to free system resources.

**Code for Cgroup Management**:
```go
// CreateCgroup creates a new cgroup for a job with specific resource limits.
func CreateCgroup(jobID string, config *ResourceConfig) (string, error) {
    // Create a new cgroup with a unique path using the jobID
    control, err := cgroups.New(cgroups.V1, cgroups.StaticPath("/job-"+jobID), &specs.LinuxResources{
        // Set the CPU shares (resource allocation) based on the CPU quota configuration
        CPU: &specs.LinuxCPU{
            Shares: uint64(1024 * parseCPUQuota(config.CPUQuota)),  // Multiply by 1024 to adjust the CPU shares
        },
        // Set the memory limit for the cgroup
        Memory: &specs.LinuxMemory{
            Limit: parseMemoryLimit(config.MemoryLimit),    // Parse and set the memory limit
        },
        // Set the block IO weight for controlling disk I/O priority
        BlockIO: &specs.LinuxBlockIO{
            Weight: parseDiskIO(config.DiskIO), // Parse and set the disk I/O weight
        },
    })

    // If there was an error creating the cgroup, return the error
    if err != nil {
        return "", err
    }

    // Return the path of the created cgroup
    return control.Path(""), nil
}

// AttachProcessToCgroup assigns a process to a specific cgroup based on its PID.
func AttachProcessToCgroup(cgroupPath string, pid int) error {
// Load the cgroup at the specified path
    control, err := cgroups.Load(cgroups.V1, cgroups.StaticPath(cgroupPath))
    if err != nil {
        return err  // If there was an error loading the cgroup, return the error
    }

    // Add the process (identified by its PID) to the cgroup
    return control.Add(cgroups.Process{Pid: pid})
}
```

---

### **2. Resource Management with cgroups**

This feature enables precise control of job resource usage, ensuring system stability and fairness.

**Supported Resource Limits**:
- **CPU**: Restrict CPU usage as a percentage (e.g., "50%").
- **Memory**: Define maximum memory usage (e.g., "256M").
- **Disk I/O**: Throttle disk operations (e.g., "10MB/s").

---

### **3. gRPC API**

The gRPC API enables remote job management. It includes methods to start, stop, query, and stream logs for jobs.  

#### **Proto Definition**:
```protobuf
message ResourceConfig {
  string cpu_quota = 1;    // e.g., "50%"
  string memory_limit = 2; // e.g., "512M"
  string disk_io = 3;      // e.g., "10MB/s"
}

message StartRequest {
  string name = 1;
  repeated string args = 2;
  ResourceConfig resources = 3;
}

service JobService {
  rpc Start(StartRequest) returns (StartResponse);
  rpc Stop(StopRequest) returns (StopResponse);
  rpc Query(QueryRequest) returns (QueryResponse);
  rpc Stream(StreamRequest) returns (stream StreamResponse);
}
```

---

### **4. Command-Line Interface (CLI)**

The CLI allows users to interact with the system, supporting resource control parameters during job creation.

#### **Example Usage**:
```bash
# Start a job with resource limits
$ job-cli start "bash" "-c" "while true; do echo hello; sleep 1; done" \
  --cpu 50% --memory 256M --disk-io 10MB/s

# Query job status
$ job-cli query job123

# Stream job logs
$ job-cli stream job123
```

---

## Security Features

The system ensures secure and authenticated communication between the client and server:  

1. **TLS 1.3**: Provides encryption for all communication.  
2. **Mutual Authentication**: Uses X.509 certificates to verify both client and server identities.  
3. **Role-Based Access Control (RBAC)**:  
   - Roles (`reader`, `writer`) are embedded in certificates via X.509 extensions.  
   - gRPC interceptors enforce role-based access to API methods.  

---

## Development and Testing

### **Testing Workflow**:
- **Unit Tests**:  
  - Test cgroup creation, resource enforcement, and job lifecycle management.  
  - Mock gRPC calls to ensure API correctness.  

- **Integration Tests**:  
  - Run multiple jobs with resource limits.  
  - Simulate resource contention to verify cgroup enforcement.  

### **Build & Test Commands**:
```bash
# Run tests
$ make test

# Build the API server
$ make api
$ ./bin/job-api

# Build the CLI tool
$ make cli
$ ./bin/job-cli
```

---

## Conclusion

This design uses Linux cgroups to control resources, providing a strong and secure way to manage and run jobs. It includes features like real-time monitoring, secure communication, and enforcing resource limits, making the system efficient and reliable for job execution.
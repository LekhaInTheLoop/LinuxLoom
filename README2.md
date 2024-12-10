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
// / Command represents a program that will be executed with specific arguments and resource limits.
type Command struct {
    Name           string         // Program name
    Args           []string       // Arguments for the program
    ResourceConfig *ResourceConfig // Resource limits for the job
}

// ResourceConfig holds the resource constraints for a job, such as CPU, memory, and disk I/O limits.
type ResourceConfig struct {
    CPUQuota    string // e.g., "50%" for limiting CPU usage
    MemoryLimit string // e.g., "256M" for limiting memory usage
    DiskIO      string // e.g., "10MB/s" for I/O throttling
}

// Job represents a single job that is being executed, with associated details such as its ID, command, and resource limits.
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
// ResourceConfig defines the resource limits for a job, such as CPU, memory, and disk I/O.
message ResourceConfig {
  string cpu_quota = 1;    // CPU usage limit (e.g., "50%" means the job can use up to 50% of a CPU core)
  string memory_limit = 2; // Memory usage limit (e.g., "512M" limits the job to 512 MB of memory)
  string disk_io = 3;      // Disk I/O throttling (e.g., "10MB/s" limits the job to 10 MB per second for disk operations)
}

// StartRequest is used to request starting a job, with its command name, arguments, and resource configuration.
message StartRequest {
  string name = 1;           // Name of the program to execute (e.g., "myProgram")
  repeated string args = 2;  // List of arguments to pass to the program (e.g., ["arg1", "arg2"])
  ResourceConfig resources = 3; // Resource configuration for limiting CPU, memory, and disk I/O
}

// JobService defines the service with methods to start, stop, query, and stream jobs.
service JobService {
  // Start starts a job with the given StartRequest and returns a StartResponse.
  rpc Start(StartRequest) returns (StartResponse);

  // Stop stops a running job, given the StopRequest, and returns a StopResponse.
  rpc Stop(StopRequest) returns (StopResponse);

  // Query queries the status of a job, given a QueryRequest, and returns a QueryResponse.
  rpc Query(QueryRequest) returns (QueryResponse);

  // Stream streams the output of a job, given a StreamRequest, and returns a stream of StreamResponses.
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
2. **Mutual Authentication**:        - Uses **X.509 certificates** to verify both client and server identities.
   - The certificates are signed using the **sha256WithRSAEncryption** signature algorithm.
   - **Public Key Algorithm** used for the certificate is **rsaEncryption** with an **RSA Public-Key** size of **4096 bits**, ensuring strong cryptographic security.
3. **Role-Based Access Control (RBAC)**:  
   - Roles (`reader`, `writer`) are embedded in certificates via **X.509 extensions**.  
   - Enforces role-based access using **gRPC interceptors**, which ensure that only authorized users can invoke specific API methods  

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
This design leverages gRPC to efficiently manage job execution, using protobuf for query transmission instead of the traditional JSON format. By incorporating Linux cgroups, the system ensures robust resource control, limiting CPU, memory, and disk I/O for each job. It also includes secure communication via TLS 1.3 and mutual authentication with X.509 certificates. Additionally, gRPC interceptors are utilized to enforce role-based access control (RBAC), ensuring only authorized users can access specific API methods.

This comprehensive approach makes the system secure, efficient, and reliable for executing jobs with real-time monitoring and strong resource management.
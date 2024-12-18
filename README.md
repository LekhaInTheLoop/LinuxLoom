# A Prototype Job Service
## Summary

A Prototype job worker service that provides an API to run arbitrary Linux processes.

## Overview

### Library

Hello 

The Job Worker is a flexible Golang package that helps run and manage Jobs on Linux systems. It handles starting, stopping, querying and streaming output from processes, as well as catching errors if any.

The Worker keeps track of process statuses in memory using a local map, so it can update the status when a job finishes. But this data is temporary and will be lost if the Worker crashes or restarts, since it doesn�t store job state permanently.

As part of its functionality, the Worker will expose an API to stream the output of a specific job, allowing users to fetch the logs in real-time using a job ID. The process output (stdout/stderr) is also saved as log files on the disk for future reference. However, over time, these log files can take up significant disk space, which could cause the system to crash if the disk runs out of space.

To manage log file growth and avoid running out of disk space, we can implement solutions like log rotation, establish a log purging policy, or store logs in a distributed system such as Amazon S3. Currently, logs are stored in the /tmp directory, but this location can be customized through the configuration file.

```golang
// Command is a job request with the program name and arguments.
type Command struct {
    // Command name
    Name string
    // Command arguments
    Args []string
}

// Job represents an arbitrary Linux process schedule by the Worker.
type Job struct {
    // ID job identifier
    ID string
    // Cmd started
    Cmd *exec.Cmd
    // Status of the process.
    Status *Status
}

// Status of the process.
type Status struct {
    // Process identifier
    Pid int
    // ExitCode of the exited process, or -1 if the process hasn't
    // exited or was terminated by a signal
    ExitCode int
    // Exited reports whether the program has exited
    Exited bool
}

// Worker defines the basic operations to manage Jobs.
type Worker interface {
    // Start creates a Linux process.
    //    - command: command to be executed 
    // It returns the job ID and the execution error encountered.
    Start(command Command) (jobID string, err error)
    // Stop a running Job which kills a running process.
    //    - ID: Job identifier
    // It returns the execution error encountered.
    Stop(jobID string) (err error)
    // Query a Job to check the current status.
    //    - ID: Job identifier
    // It returns process status and the execution error encountered.
    Query(jobID string) (status Status, err error)
    // Streams the process output.
    //    - ctx: context to cancel the log stream
    //    - ID: Job identifier
    // It returns a chan to stream process stdout/stderr and the
    // execution error encountered.
    Stream(ctx context.Context, jobID string) (logchan chan string, err error)
}
```
### API

The API is a gRPC server responsible for interacting with the Worker Library to start/stop/query processes and also consumed by the remote Client. The API is also responsible for providing authentication, authorization, and secure communication between the client and server.

```protobuf
syntax = "proto3";
option go_package = "github.com/renatoaguimaraes/job-scheduler/worker/internal/proto";

message StartRequest {
  string name = 1;
  repeated string args = 2;
}

message StartResponse {
  string jobID = 1;
}

message StopRequest {
  string jobID = 1;
}

message StopResponse {
}

message QueryRequest {
  string jobID = 1;
}

message QueryResponse {
  int32 pid = 1;
  int32 exitCode = 2;
  bool exited = 3;
}

message StreamRequest {
  string jobID = 1;
}

message StreamResponse {
  string output = 1;
}

service WorkerService {
  rpc Start(StartRequest) returns (StartResponse);
  rpc Stop(StopRequest) returns (StopResponse);
  rpc Query(QueryRequest) returns (QueryResponse);
  rpc Stream(StreamRequest) returns (stream StreamResponse);
}
```

### Client	

The Client is a command line interface where the user interacts to schedule remote Linux jobs. The CLI provides communication between the user and Worker Library through the gRPC.

## Security

### Transport

The Transport Layer Security (TLS), version 1.3, provides privacy and data integrity in secure communication between the client and server.

### Authentication

The authentication will be provided by mTLS. The following assets will be generated, using the openssl v1.1.1k, to support the authorization schema:

* Server CA private key and self-signed certificate
* Server private key and certificate signing request (CSR)
* Server signed certificate, based on Server CA private key and Server CSR
* Client CA private key and self-signed certificate
* Client private key and certificate signing request (CSR)
* Client signed certificate, based on Client CA private key and Client CSR

The authentication process checks the certificate signature, finding a CA certificate with a subject field that matches the issuer field of the target certificate, once the proper authority certificate is found, the validator checks the signature on the target certificate using the public key in the CA certificate. If the signature check fails, the certificate is invalid and the connection will not be established. Both client and server execute the same process to validate each other. Intermediate certificates won't be used.

### Authorization
The user roles will be added into the client certificate as an extension, so the gRPC server interceptors will read and check the roles to authorize the user, the roles available are reader and writer, the writer will be authorized do start, stop, query and stream operations and the reader will be authorized just to query and stream operations available on the API. A memory map will keep the name of the gRPC method and the respective roles authorized to execute it.
The X.509 v3 extensions will be used to add the user role to the certificate. For that, the extension attribute roleOid 1.2.840.10070.8.1 = ASN1:UTF8String, must be requested in the Certificate Signing Request (CSR), when the user certificate is created, the user roles must be informed when the CA signs the the CSR. The information after UTF8String: is encoded inside of the x509 certificate under the given OID.

#### gRPC interceptors
* UnaryInterceptor
* StreamInterceptor

#### Certificates
* X.509
* Signature Algorithm: sha256WithRSAEncryption
* Public Key Algorithm: rsaEncryption
* RSA Public-Key: (4096 bit)
* roleOid 1.2.840.10070.8.1 = ASN1:UTF8String (for the client certificate)

## Scalability
For now, the client will connect with just a single Worker node. Nevertheless, for a production-grade system, the best choice is the External Load Balancer approach.
The Worker API can run in a cluster mode with several worker instances distributed over several nodes/containers. On the other hand, the Client must send requests to the same Worker for a specific job, creating an affinity to interact with that later.

|      | Proxy                                       | Client Side                                                                                |  
| ---- | ------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Pros | Simple client                               |                                                                                            |
|      | No client-side awareness of backend         | High performance because elimination of extra hop                                          |
|      |                                             |                                                                                            |
| Cons | LB is in the data path                      | Complex client                                                                             |  
|      | Higher latency                              | Client keeps track of server load and health                                               | 
|      | LB throughput may limit scalability         | Client implements load balancing algorithm                                                 |  
|      |                                             | Per-language implementation and maintenance burden                                         |
|      |                                             | Client needs to be trusted, or the trust boundary needs to be handled by a lookaside LB    |  


### Trade-offs
Proxy Load balancer distributes the RPC call to one of the available backend servers that implement the actual logic for serving the call. The proxy model is inefficient when considering heavy request services like storage.
Client-side load balancing is aware of multiple backend servers and chooses one to use for each RPC. For the client-side strategy, there are two models available, the thicker client and external load balancer. 
The thicker client places more of the load balancing logic in the client, a list of servers would be either statically configured in the client.
External load-balancing is the primary mechanism for load-balancing in gRPC, where an external load balancer provides simple clients with an up-to-date list of servers.

## Architecture 

![Architecture](assets/architecture.jpg)

## Test

```sh
$ make test
```
## Build and run API

```sh
$ make api
go build -o ./bin/worker-api cmd/api/main.go
```

```sh
$ ./bin/worker-api
```

## Build and run Client

```sh
$ make client
go build -o ./bin/worker-client cmd/client/main.go
```

```sh
$ ./bin/worker-client start "bash" "-c" "while true; do date; sleep 1; done"
Job 9a8cb077-22da-488f-98b4-d2fb51ba4fc9 is started
```

```sh
$ ./bin/worker-client query 9a8cb077-22da-488f-98b4-d2fb51ba4fc9
Pid: 16963256 Exit code: 0 Exited: false
```

```sh
$ ./bin/worker-client stream 9a8cb077-22da-488f-98b4-d2fb51ba4fc9
...
...
```

```sh
./bin/worker-client stop 9a8cb077-22da-488f-98b4-d2fb51ba4fc9
Job 79d95817-7228-4c36-8054-6c29513841b4 has been stopped
```

**A Prototype Job Worker Service**. 


A Call Flow


This breakdown describes the interaction between the various components during job lifecycle events, including resource management via Linux cgroups.

---

### **1. Job Start Flow**
This sequence explains how a new job is created and started with resource limits.

1. **Client Side**:
   - User invokes the `job-cli start` command with parameters (e.g., program name, arguments, and resource limits like CPU, memory, and disk I/O).
   - `job-cli` calls the **gRPC client**, which serializes the request and sends it over an mTLS-secured connection to the gRPC server.

2. **Server Side (gRPC Server)**:
   - The **gRPC server** receives the request, validates it, and forwards the parameters to the **worker library**.

3. **Worker Library**:
   - The **worker** library:
     - Creates a unique identifier for the job.
     - Sets up a **cgroup** for the job by:
       - Allocating a new cgroup under `/sys/fs/cgroup`.
       - Applying resource limits (CPU, memory, disk I/O) as specified in the request.
     - Spawns the process using `exec.Cmd`, assigning it to the created cgroup.

4. **Process Execution**:
   - The worker:
     - Executes the process.
     - Captures `stdout` and `stderr` streams.
     - Writes logs to disk for persistence.

5. **Response**:
   - The gRPC server sends a response back to the client with the **job ID** and status (e.g., "running").

---

### **2. Job Stop Flow**
This sequence describes how a running job is terminated.

1. **Client Side**:
   - User invokes `job-cli stop <job_id>` to terminate a specific job.
   - `job-cli` sends a stop request to the **gRPC client**, which forwards it to the gRPC server.

2. **Server Side**:
   - The gRPC server:
     - Validates the job ID.
     - Passes the stop request to the worker library.

3. **Worker Library**:
   - The worker library:
     - Identifies the running process by job ID.
     - Sends a termination signal (e.g., SIGTERM or SIGKILL) to the process.
     - Cleans up associated resources, including:
       - Removing the cgroup.
       - Closing file handles for stdout/stderr.
       - Deleting temporary log files (if configured).

4. **Response**:
   - The gRPC server sends a confirmation back to the client (e.g., "job stopped successfully").

---

### **3. Job Query Flow**
This sequence describes how the status of a job is fetched.

1. **Client Side**:
   - User invokes `job-cli query <job_id>` to fetch the current status of a job.
   - The CLI sends a query request to the gRPC client.

2. **Server Side**:
   - The gRPC server:
     - Validates the job ID.
     - Queries the worker library for the job’s current state.

3. **Worker Library**:
   - The worker library:
     - Fetches the job status from its internal data structures (e.g., running, completed, failed).
     - Checks for any associated resource usage metrics from the cgroup.
     - Returns the status and metrics to the gRPC server.

4. **Response**:
   - The gRPC server sends the job status (and metrics, if available) back to the client.

---

### **4. Job Log Streaming Flow**
This sequence explains how real-time job logs are streamed.

1. **Client Side**:
   - User invokes `job-cli stream <job_id>` to stream logs.
   - The gRPC client establishes a streaming connection with the gRPC server.

2. **Server Side**:
   - The gRPC server:
     - Validates the job ID.
     - Initializes a stream of log data from the worker library.

3. **Worker Library**:
   - The worker library:
     - Reads the log files (stdout and stderr) associated with the job.
     - Streams log data back to the gRPC server.

4. **Response**:
   - The gRPC server streams logs back to the client in real-time.

---

### **5. Security and Authentication**
Throughout all call flows:
- **mTLS**:
  - Ensures encrypted communication between the client and server.
  - Both client and server authenticate each other using X.509 certificates.

- **RBAC (Role-Based Access Control)**:
  - API endpoints are protected using role-based access controls.
  - Roles (`reader`, `writer`) are enforced by gRPC interceptors based on X.509 certificate extensions.



Architecture Design Description
Key Components:
Client:

Consists of:
CLI (cmd): Provides the user interface to interact with the system.
gRPC Client: Sends requests to the server using mTLS.
Server:

gRPC Server: Receives client requests, validates inputs, and delegates tasks to the worker library.
Worker Library: Core module that:
Manages job lifecycles.
Implements resource control using cgroups.
Captures and streams logs.
Linux Subsystems:

Process Execution: Executes jobs (e.g., Bash commands or custom scripts).
cgroups: Enforces resource constraints (CPU, memory, disk I/O).
File System: Stores process logs (stdout/stderr).
Diagram Elements
Here’s how the architecture should look:

Main Blocks:
Client:

Box titled Client with two inner components:
cmd (CLI)
gRPC Client
Server:

Box titled Worker (Server) with three inner components:
gRPC Server
Worker Library
Linux Subsystems (cgroups, process execution, and file system)
Connections:

Client communicates with Server using mTLS.
gRPC Server calls the Worker Library to process requests.
Worker Library interacts with:
cgroups for resource limits.
Process Execution to start/stop jobs.
File System to manage logs.
Steps to Recreate in Lucidchart
Create Main Blocks:

Use rectangles for "Client" and "Worker (Server)".
Inside the Client block, add smaller rectangles for cmd and gRPC Client.
Inside the Worker (Server) block, add three smaller rectangles for gRPC Server, Worker Library, and Linux Subsystems.
Add Subsystems:

Inside the Linux Subsystems block, add smaller elements for:
cgroups
Process Execution
File System
Connect Components:

Draw arrows:
From cmd (CLI) to gRPC Client (unidirectional).
From gRPC Client to gRPC Server (unidirectional) with a label: mTLS.
From gRPC Server to Worker Library.
From Worker Library to each of the Linux subsystems (bidirectional):
cgroups: For resource control.
Process Execution: To start/stop processes.
File System: To read/write logs.
Styling:

Use colors to distinguish components (e.g., blue for server, green for client, gray for subsystems).
Label arrows to describe the interaction (e.g., Start Job, Stream Logs).
Optional Enhancements
Add icons for better visual clarity:
Terminal icon for CLI.
Cloud icon for the gRPC connection.
File icon for logs.
Would you like me to generate a visual for you, or guide you further?

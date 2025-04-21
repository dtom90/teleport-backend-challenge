---
authors: David Thomason (dtom90@gmail.com)
state: draft
---

# RFD - Remote Linux Process Manager

## Required Approvers
* Engineering: @smallinsky @gabrielcorado @russjones @zmb3 @rosstimothy @r0mant

## What

Implement a prototype job worker service that provides an API to run arbitrary Linux processes.

## Why

Teleport will use this project in order to evaluate the author's eligibility for the Senior Systems Engineer role. Implementing this application simulating a real-world project will allow the author to effectively demonstrate his skills to Teleport.

## Scope

The following will be implemented in this project:

- A Worker library for interacting with jobs
  - resource control for CPU, Memory and Disk IO per job using cgroups
- An API server that wraps the functionality of the library
  - gRPC API
  - mTLS authentication
  - simple authorization scheme
- A command line interface (CLI) that communicates with the API server
  - start/stop/query status and get the streaming output of a job

The following could be implemented in a real production application, but are out of scope for this project:

- Persistent storage
  - (storage will be in memory)
- Segmented user process spaces
  - (all users will see the same processes)

## Details

### CLI

Users will be able to run a CLI program that will manage jobs on a remote Linux system. This CLI tool will be called `rps` and will have the following commands:

- `rps start -h <host> -c "<command>"`: Start a new job with the given command and return the job_id
- `rps status -h <host> <job_id>`: Get the status of the job with the given job_id
- `rps output -h <host> <job_id>`: Stream the output of the job with the given job_id
- `rps stop -h <host> <job_id>`: Stop a running job with the given job_id

The host can also be specified via the environment variable `RPS_HOST` to avoid having to specify it as CLI argument.

`job_id` is a unique identifier for a job, implemented as a UUID.

Here are some examples of how the CLI will work:

```sh
$ rps connect 158.101.135.237
Connected to Remote Linux Process Manager at 158.101.135.237
(enter "help" for usage)
> rps start -c "for i in {1..10}; do echo $i; sleep 1; done"
Job started with job_id: 12345678-1234-1234-1234-123456789012
> rps status 12345678-1234-1234-1234-123456789012
Job 12345678-1234-1234-1234-123456789012 is running
- started: 2025-04-20 12:31:45 -0500 CDT
- command: "for i in {1..10}; do echo $i; sleep 1; done"
> rps output 12345678-1234-1234-1234-123456789012
1
2
3
4
> rps stop 12345678-1234-1234-1234-123456789012
Job 12345678-1234-1234-1234-123456789012 is stopped
```

### API

#### Authentication

Authentication will be handled using mTLS. TLS 1.3 with `TLS_AES_256_GCM_SHA384` cipher suite will be used for maximum security.

For simplicity, the certificates (CA, server, client) will be generated using `openssl` with the `Ed25519` algorithm and checked into source. In a real application, the certificates should be generated and stored in a secure manner.

#### Authorization

The server will use the client certificate's `Subject.CommonName` as the `clientId` to identify the client.

Authorization will be segmented into:

- `rpsreader`: read-only access (`GetJobStatus`, `StreamJobOutput`)
- `rpsmanager`: full job management access (read + `StartJob` & `StopJob`)

Users (clients) will be assigned one of these two roles in a hard-coded map for simplicity. In a real application, this assignment would be handled by an authorization server with groups and permissions.

#### GRPC

The API will be a gRPC server that handles messages from the CLI. Protocol Buffers (protobuf) will be used to define and serialize the methods and messages, which will provide consistency between the gRPC server and client.

### Library

The library will implement the business logic for the API, interacting with the linux system to start, stop, and manage jobs.

#### Lifecycle

0. Initialize (system setup)
    - cgroups are initialized with the specified resource limits
    - SIGCHLD handler is set up to handle process termination (asynchronously)
1. Start Job
    - process is assigned a job ID (UUID) and added to an in-memory process map
    - output file is created, identified by the job ID
    - process is started as a child process in user mode under `rpsuser`
      - process output is redirected to the output file on process start
      - process is added to the cgroup on process start using the linux `cgexec` utility, which will allow running a process directly within a specified control group without having to manually add it
2. Job Running
    - process status can be queried by accessing the process map
    - process output can be streamed by reading the output file
3. Stop Job
    - when process ends, the SIGCHLD handler will be invoked
    - process map status is updated (checking status will say `stopped`)
    - output file cleanup is scheduled for 1 hour after process end

As mentioned `job_id` will be a UUID, which is unique and less likely to be reused than a PID. This will allow us to distinguish between processes that had the same PID when getting the status/output of a job.

The asynchronous SIGCHLD handler will avoid blocking the main thread and will ensure that the process terminates immediately and gracefully (no zombie state) and invoke cleanup.

Processes will be run by a dedicated non-sudoer user (e.g. `rpsuser`) to ensure that jobs are not run as root or sudo. This will be a different user than the API server user, which will be run as `root` to be able to manage cgroups. `rpsuser` will not have any permissions to server code or output files (more deails below)

#### Output Streaming

Process output will be streamed to a disk file identified by the job ID, rather than kept in memory. The tradeoffs are:

| Feature | Memory stream | Disk File |
|---|---|---|
| Read/Write performance | Faster | Slower |
| Resource usage | RAM (potentially high) | Disk, low RAM |
| Concurrent read implementation | More Complex (channels, mutex, etc.) | Easier (OS handles concurrent reads) |
| Persistence after process stop | More complex | Simpler |

While memory streams offer faster read/write speeds, they require more RAM and complex synchronization for concurrent access. Disk files, though slower, use less RAM, are simpler to manage for concurrent reads (handled by the OS), and persist after process termination. To avoid excessive disk usage, output files will be cleaned up after 1 hour.

_Note:_ In a production system, we would want a more sophisticated output file management, such as moving them to an external system, and automatic cleanup if disk space gets low. Keeping it simple for this demo

Output files will be created by the `root` user with 600 permissions. The `rpsuser` running jobs will not have write access to any output file, in order to prevent users from messing with these files. Instead, the server will pass an open file descriptor to the child process, which will write to the output file.

If a user reads output via `rps output` while a process is runnning, the stream will remain open as long as the process is running. Once the process exits, the output file will be closed and all reading streams will be closed as well.

#### Resource Control

Resource control will be implemented using cgroups. 

- only cgroups V2 will be supported
- cgroup setup will be run in priveleged mode on server startup
- CPU, Memory, Disk IO, and number of PIDs will be limited. 
  - Limit values will be hard-coded for simplicity
    - _In production, these values should be set by a configuration system such as a config file setting environment variables_
- Resource limits will be defined per job to ensure that rouge jobs do not affect other jobs
  - Naming convention will be `/sys/fs/cgroup/rpsmanager/<job_id>`
- Limits will be set during cgroup creation by writing to the cgroup files _(no third-party libraries used)_
- Job cgroup will be deleted by the SIGCHLD handler when the process terminates

### Security

At a broad level, the security measures here are primarily designed to limit access and prevent resource hogging.

1. API authentication & authorization
2. Run processes under OS user `rpsuser` with limited access
3. cgroups for resource limits

The current design has all users sharing the same process space (via common OS user `rpsuser`), so users with execute permissions can stop other processes and readers can read other process state and output. We primarily rely on authentication to prevent malicious users.

Some ways this design could be enhanced in the future for stronger isolation with the trade of additional complexity:
1. Implement mount namespaces, which would isolate the user filesystem from the OS filesystem
2. Give each user (TLS client) its own OS user and namespace (isolated process state for each user)

### Logging and Error Handling

Logging will be performed by the API layer on both success and error events returned by the library, including the asynchronous process termination event. Job and client IDs will be included for auditing purposes.

Any errors during job start / stop will be returned to the API and stored in the process map with details. Standard error codes will correspond to different error scenarios (e.g. invalid command, cgroup setup failed, etc.). The API will log these errors and translate them to standard gRPC status codes. The CLI will display the errors in a user-friendly manner after a failed command, as well as in the job status output.

### Testing

Tests will include but not be limited to:

- authorization
  - valid and invalid clients
  - unauthorized and forbidden methods
- output streaming
  - failed to create/write to output file
- cgroup setup
  - failed to create cgroup
  - failed to add process to cgroup

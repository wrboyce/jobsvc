# Job Control System with gRPC API and CLI

A secure, minimal system for starting, stopping, and monitoring background jobs, with per-job resource controls implemented via Linux cgroups. Authorisation will be handled via mTLS with a simple auth scheme limiting users to only interact with jobs they started.

The system consists of three components:

- Library: responsible for process execution, output capture, and cgroup setup.
- API Server: a gRPC server handling auth and exposing job control functionality.
- CLI Client: a command-line interface for interacting with the API.

## Details

### Job Model

- Jobs are identified by a (per-user) unique name provided by the user at creation.
- Jobs run as child processes in isolated cgroups.
- Output (stdout and stderr) is captured in-memory and exposed via the API.
- Only the client that created a job may interact with it.

### CLI Interface

Two binaries will be provided, `jobsvc` and `jobctl`.

Global flags `--cert`, `--key`, `--ca` will be used for authentication.

#### Server

The `jobsvc` binary will be used to start the server and run in the foreground.

#### Client

The `jobctl` CLI interacts with the gRPC API to control jobs:

```bash
jobctl start my-job -- yes
jobctl stop my-job
jobctl status my-job
jobctl logs my-job
jobctl list
```

The `start` subcommand will return an error if the job name is already taken. No restart, delete, or resource flags are exposed — by design.

##### `status` Command

Displays the current state of a specific job by name. Output is presented in a simplified format inspired by `systemctl status`.

**Running job:**

```
Job:         my-job-name
Status:      running
PID:         12345
Command:     yes
Started At:  2025-01-01 00:01:30
```

**Finished job (successful exit):**

```
Job:         my-job-name
Status:      finished
PID:         12345
Command:     echo hello
Started At:  2025-01-01 00:01:30
Finished At: 2025-01-01 00:01:31
Exit Code:   0
```

**Finished job (non-zero exit code):**

```
Job:         my-job-name
Status:      failed
PID:         12345
Command:     curl http://localhost
Started At:  2025-01-01 00:01:30
Finished At: 2025-01-01 00:01:31
Exit Code:   1
```

**Failed job:**

```
Job:         my-job-name
Status:      error
Command:     foobar
Started At:  2025-01-01 00:01:30
Finished At: 2025-01-01 00:01:30
```

Status classification:

- `running`: job is currently executing
- `finished`: job exited with code 0
- `failed`: job exited with non-zero code
- `error`: job creation failed

Timestamps use local time in `YYYY-MM-DD HH:MM:SS` format (represented as a string for simplicity). Fields are aligned for readability. No structured (e.g. JSON) output is currently supported. Details of the reason for job creation failure have been omitted from the API for simplicity.

### gRPC API

The system exposes a gRPC API with endpoints for job lifecycle and log access.

#### Methods

- `StartJob`: Create a new job
- `StopJob`: Terminate a job
- `GetJobStatus`: Returns job metadata including status, PID, timestamps, command, and exit code (if available).
- `GetJobLogs`: Stream job output
- `ListJobs`: List known jobs

See `jobsvc.proto` for full schema.

### Cgroups and Resource Limits

Each job is placed into a unique cgroup with hardcoded resource limits:

| Resource | Interface    | Value                                   |
| -------- | ------------ | --------------------------------------- |
| CPU      | `cpu.max`    | `50000 100000` (50%)                    |
| Memory   | `memory.max` | `1073741824` (1 GiB)                    |
| IO       | `io.max`     | `rbps=104857600` (100 MiB/s read limit) |

These values are written to appropriate cgroup v2 files under a per-job path. IO throttling is applied using a basic read bandwidth limit of 100 MiB/s. Write limits and device-specific control are not configured.

### Authentication and Authorization

All communication between client and server is authenticated using mTLS.

- Clients must present a valid X.509 certificate signed by the system CA.
- The Common Name (CN) of the certificate is treated as the client identity.

Authorization is enforced as follows:

- Clients may only access or modify jobs they created.
- The server tracks the CN of each job’s creator and compares it with the incoming request's identity.

No role-based access control or external identity systems are implemented, per scope constraints. Crucially, jobs execute with the same user privileges as the jobsvc server itself.

### Logging and Observability

The server writes basic logs to stderr for development and debugging:

- Job creation, start, stop, and exit
- Authorization failures
- Unexpected errors

No external observability systems (e.g. metrics, tracing) are implemented.

### Testing

Targeted tests will be written for:

- Authorization logic
- Output streaming
- Cgroup setup

Tests focus on both happy and error scenarios, without aiming for 100% coverage.

### Caveats and Simplifications

This project is intentionally scoped for a technical exercise and is not intended to be production-ready. The following caveats and simplifications apply:

- Resource limits are hardcoded and not configurable.
- Any valid client certificate grants full API access; job isolation is the only authorization mechanism enforced.
- There is no persistent state — jobs and logs are lost when the server process exits.
- Job names must be unique and cannot be reused.
- Output is stored in-memory only and merged (stdout + stderr) without log rotation or truncation.
- No restart, delete, or auto-cleanup functionality is implemented.
- The CLI is designed for human interaction only — it does not support structured output for programmatic use.
- No config files, runtime reloads, or environment-variable overrides are supported.

### Future Work

This system could be extended with:

- Restart policies and lifecycle hooks
- Persistent job state (e.g. via embedded DB)
- Web UI or extended CLI
- Integration with audit logs, tracing, and Prometheus metrics
- Namespace isolation or seccomp

## Drawbacks

- Hardcoded resource limits may not suit all workloads.
- In-memory log storage is unbounded and ephemeral.
- No resilience to server crashes (jobs and logs lost on restart).
- Trust is entirely based on certificate CNs.
- All jobs will be run as the system superuser (or whichever user started the job server).

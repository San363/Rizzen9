https://github.com/San363/Rizzen9/releases

# Rizzen9 — Gin API Gateway with C++ Modules and MariaDB

[![Releases](https://img.shields.io/badge/Releases-Download-blue?logo=github&style=for-the-badge)](https://github.com/San363/Rizzen9/releases) [![C++](https://img.shields.io/badge/C%2B%2B-Modern-00599C?style=for-the-badge&logo=c%2B%2B)](https://isocpp.org/) [![MariaDB](https://img.shields.io/badge/MariaDB-Storage-003545?style=for-the-badge&logo=mariadb)](https://mariadb.org/) [![Gin](https://img.shields.io/badge/Gin-HTTP-09A3E6?style=for-the-badge&logo=gin)](https://github.com/gin-gonic/gin)

![Rizzen9 Hero](https://images.unsplash.com/photo-1550751827-4bd374c3f58b?ixlib=rb-4.0.3&w=1600&q=80)

Rizzen9 pairs a Gin HTTP gateway with high-performance C++ modules. It targets workloads that need low-latency compute, tight DB control, and a simple REST interface. Use the releases page to get a runnable build. Download and execute the release file from https://github.com/San363/Rizzen9/releases.

Table of contents
- Features
- Architecture
- Components
- Quickstart (download and run)
- Build from source
- Configuration
- API reference
- Database (MariaDB) setup
- C++ modules guide
- Testing and benchmarks
- Troubleshooting
- Contributing

Features
- Fast HTTP gateway built on Gin.
- Native C++ modules for CPU-bound logic.
- MariaDB for persistent storage.
- Clear API for service wiring and metrics.
- Modular code that separates web, compute, and storage.
- Simple release artifacts for one-step deploy.

Architecture
Rizzen9 splits work into three parts:
1. Gateway (Gin)
   - Handles HTTP, routing, validation.
   - Converts requests to RPC calls for C++ modules.
2. C++ modules
   - Perform compute tasks.
   - Expose a stable API via sockets or FFI.
3. MariaDB
   - Store persistent data.
   - Use prepared statements and connection pooling.

This separation keeps the HTTP layer thin. Gin handles concurrency and request lifecycle. C++ modules handle heavy compute jobs. MariaDB stores durable state.

Core components
- Gin gateway (Go)
  - Routes, middleware, auth, metrics.
  - Health checks and readiness probes.
- Module loader
  - Starts C++ modules as child processes.
  - Manages lifecycle and restart on failure.
- IPC layer
  - UNIX sockets or TCP for module calls.
  - Simple binary protocol with JSON for control messages.
- Storage adapter
  - MariaDB connector with pooling.
  - Migration scripts in SQL.

Quickstart — download and run
1. Visit the releases page and download the release artifact:
   https://github.com/San363/Rizzen9/releases
2. On a Linux or macOS host run:
   - Extract the archive.
   - Execute the bundled startup script or binary.
   Example:
   - tar xzf rizzen9-vX.Y.Z.tar.gz
   - cd rizzen9
   - ./start.sh
3. The startup script launches:
   - MariaDB adapter (connects to an existing DB).
   - Gin gateway on configured port.
   - C++ modules as needed.
4. Open the health endpoint:
   - curl http://localhost:8080/health

The release file at the releases page contains a ready-to-run bundle. Download it and execute the included script to start a local instance.

Build from source
Prereqs
- Go 1.20+ for the Gin gateway.
- A modern C++ compiler (g++ 11+ or clang 12+).
- CMake 3.16+.
- MariaDB client headers and library.
- Git.

Clone
- git clone https://github.com/San363/Rizzen9.git
- cd Rizzen9

Build steps
- Build C++ modules:
  - mkdir -p build && cd build
  - cmake .. -DCMAKE_BUILD_TYPE=Release
  - cmake --build . -- -j$(nproc)
  - cd ..
- Build Gin gateway:
  - cd gateway
  - go build -o rizzen9-gateway ./cmd/server
- Create artifact:
  - ./scripts/package_release.sh

Configuration
Configuration uses YAML and environment variables. Files:
- config/default.yaml
- config/production.yaml

Key settings
- server.port: HTTP port (default 8080)
- db.url: MariaDB DSN (user:pass@tcp(host:port)/dbname)
- module.path: Path to C++ module binaries
- module.ipc: socket path or TCP host:port
- metrics.enabled: boolean

Use environment variables to override. Example:
- RIZZEN9_DB_URL="user:pass@tcp(127.0.0.1:3306)/rizzen9" ./start.sh

API reference
Base URL: http://{host}:{port}/api/v1

Endpoints
- GET /health
  - Returns status and uptime.
- POST /compute/run
  - Body: { "module": "factor", "payload": {...} }
  - Starts a compute job on named module.
  - Returns job id.
- GET /compute/status/{id}
  - Returns status and result when ready.
- GET /data/{entity}/{id}
  - Reads data from MariaDB using prepared statements.

Example request
- curl -X POST http://localhost:8080/api/v1/compute/run \
  -H "Content-Type: application/json" \
  -d '{"module":"factor","payload":{"n":1234567}}'

Response
- { "job": "abcd-1234", "status": "queued" }

Database (MariaDB) setup
Rizzen9 uses MariaDB for persistence and distributed locks. Use a dedicated schema.

Minimum setup
1. Install MariaDB.
2. Create database and user:
   - CREATE DATABASE rizzen9;
   - CREATE USER 'rizzuser'@'%' IDENTIFIED BY 'rizzpass';
   - GRANT ALL PRIVILEGES ON rizzen9.* TO 'rizzuser'@'%';
3. Run migrations:
   - ./migrations/migrate.sh --dsn "rizzuser:rizzpass@tcp(127.0.0.1:3306)/rizzen9"

Schema highlights
- jobs table: store compute jobs and status.
- results table: store job outputs.
- locks table: optional advisory locks.

Connection pooling
- Tune pool size for your workload.
- Default pool uses 20 connections. Adjust with DB_POOL_SIZE.

C++ modules guide
Modules host compute logic. Rizzen9 ships a sample module named factor. The sample computes integer factors and streams results back.

Module contract
- Accept JSON control messages.
- Listen on UNIX socket or TCP port.
- Return results in JSON.
- Exit with code 0 on graceful shutdown.

Sample flow
1. Gateway sends:
   - { "id": "job-1", "action": "start", "payload": { "n": 12345 } }
2. Module replies:
   - { "id": "job-1", "status": "running" }
3. On completion:
   - { "id": "job-1", "status": "done", "result": [3, 5, 823] }

Module lifecycle
- The module loader starts modules as separate processes.
- If a module crashes, the loader restarts it per policy.
- Use logs to diagnose failures.

Build a module
- Use the modules/CMakeLists.txt template.
- Implement main.cpp to parse control messages and compute results.
- Link with zmq or native sockets as needed.

Factor module (example)
- Computes prime factors.
- Uses trial division optimized with wheel factorization.
- Streams partial results to reduce memory use.

IPC and protocol
- Use newline-delimited JSON for messages.
- Keep messages small to avoid buffer bloat.
- Use backpressure from Gin to avoid queue buildup.

Testing and benchmarks
Unit tests
- Go tests for gateway:
  - cd gateway
  - go test ./...
- C++ tests with GoogleTest:
  - cmake -S . -B build -DBUILD_TESTS=ON
  - cmake --build build --target test

Load tests
- Use wrk or vegeta.
- Run a baseline with pure Gin endpoints to measure HTTP capacity.
- Run compute-heavy scenarios to see module throughput.

Benchmark tips
- Measure latency and p99.
- Adjust thread pools in C++ modules.
- Tune MariaDB connection pool.

Observability
- Metrics: Prometheus endpoint at /metrics.
- Tracing: Lightweight request ID passed to modules.
- Logs: JSON logs by default. Use centralized logging for production.

Troubleshooting
- If a module fails to start, check module.path and file permissions.
- If DB connections fail, verify DSN and network access.
- If latency spikes, check module CPU usage and thread pool sizes.

Contributing
- Open an issue for bugs or feature requests.
- Fork and create a branch per change.
- Write tests for new logic.
- Follow the code style in CONTRIBUTING.md.

Images and assets used
- C++ logo: https://cdn.jsdelivr.net/gh/devicons/devicon/icons/cplusplus/cplusplus-original.svg
- MariaDB logo: https://cdn.worldvectorlogo.com/logos/mariadb.svg
- Gin logo: https://raw.githubusercontent.com/gin-gonic/logo/master/color.png
- Hero image: Unsplash (developer/ops theme)

License
- MIT License. See LICENSE file.

Releases and downloads
- Download and execute the release artifact from the releases page:
  https://github.com/San363/Rizzen9/releases
- The release bundle contains binaries, scripts, and a minimal config. Use the included start script to run a demo instance.

Contact
- Open issues or pull requests on GitHub for questions, bug reports, or patches.
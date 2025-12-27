# Distributed Key-Value Store
<!-- Insert image from assets -->

<br>
This project is a Distributed Key-Value Store application that uses multi-Raft groups to store key-value pairs in different partitions. It uses RocksDB as the underlying database and gRPC for server-client communication.

## Features

### Core Raft Implementation
- **Leader Election**: Automatic leader election with randomized timeouts to prevent split votes
- **Log Replication**: Consistent log replication across all nodes with majority consensus
- **State Persistence**: Durable state persistence using RocksDB with write-ahead logging (WAL)

### Advanced Features
- **Raft Snapshotting**: Automatic snapshot creation for consistent state recovery when log size exceeds threshold (10,000 entries)
- **Log Compaction**: Efficient log compaction after snapshots to reduce storage overhead
- **Write-Ahead Logging (WAL)**: Explicit WAL configuration with point-in-time recovery for durable persistence
- **Dynamic Sharding**: Automatic sharding using consistent hashing for horizontal scalability
- **Automatic Failover**: Leader election and automatic failover on node failures
- **Multi-threaded Replication Pipelines**: Parallel replication to all followers using thread pools for low-latency recovery

### Observability
- **Prometheus Metrics**: Comprehensive metrics exposed on HTTP endpoint (port 9090):
  - `raft_replication_lag`: Replication lag per peer
  - `raft_request_latency_seconds`: Request latency for Put/Get operations
  - `raft_node_health`: Node health status (1=healthy, 0=unhealthy)
  - `raft_term`: Current Raft term
  - `raft_state`: Current Raft state (0=follower, 1=candidate, 2=leader)
  - `raft_log_size`: Current log size
  - `raft_committed_index`: Current committed index

Access metrics at: `http://localhost:9090/metrics`

Below are the steps to install the necessary dependencies and build the project.

## Prerequisites

Ensure you have the following installed on your system before building the project:

- **gRPC for C++**: Follow the tutorial on the official gRPC documentation [here](https://grpc.io/docs/languages/cpp/quickstart/).
    - Important: Place your project folder in the same root directory as the grpc folder, structured like this:
        ```
        |- grpc
        |- MyProjectFolder
        ```

- **RocksDB**: Install RocksDB and its development libraries using the following command:
        ```sh
        sudo apt install rocksdb librocksdb-dev
        ```

## Build Instructions

### Step 1: Clean Previous Build Files (Optional)
If you have previous build files, clean them by removing the cmake folder:
```sh
rm -rf cmake
```

### Step 2: Create and Navigate to Build Directory
Create a build directory within the project folder and make it the current working directory:
```sh
mkdir -p cmake/build
cd cmake/build
```

### Step 3: Run CMake
Run the following CMake command to configure the project. Replace `$MY_INSTALL_DIR` with the actual path where gRPC and other dependencies are installed:
```sh
cmake -DCMAKE_PREFIX_PATH=$MY_INSTALL_DIR ../..
```

### Step 4: Compile the Project
Run `make` to build the project. You can specify the number of parallel jobs (e.g., `-j 4` for 4 parallel jobs):
```sh
make -j 4
```

## Running the Server

To start the distributed key-value store servers:

```sh
cd cmake/build
./distributed_keyvaluestore -c config
```

The config file should contain one server address per line in the format `host:port`:
```
localhost:50051
localhost:50052
localhost:50053
localhost:50054
localhost:50055
...
```

**Note**: For multi-machine deployment, replace `localhost` with actual IP addresses or hostnames. The system organizes servers into Raft groups of 5 nodes each.

## Using the Client

The client library provides a simple API for interacting with the key-value store. Include the header and use the following functions:

```cpp
#include "lib739kv.h"

// Initialize client with config file
kv739_init("client/config");

// Put a key-value pair (returns old value if key existed)
std::string old_value;
kv739_put("key1", "value1", old_value);

// Get a value
std::string value;
kv739_get("key1", value);

// Shutdown a server (for testing)
kv739_die("localhost:50051", 1);  // 1 = clean shutdown, 0 = immediate

// Start a server instance
kv739_start("localhost:50051", 1);  // 1 = new instance, 0 = recovering
```

The client automatically handles routing to the correct partition using consistent hashing and redirects to the current leader of each Raft group.

## Testing

The project includes comprehensive test suites in the `tests/` directory:
- **`correctness.cc`**: Functional correctness tests for Put/Get operations
- **`performance_v2.cc`**: Performance benchmarks measuring throughput and latency
- **`reliability_v2.cc`**: Fault tolerance tests including node failures and network partitions
- **`availability.cc`**: Service availability and uptime measurements
- **`membership.cc`**: Dynamic membership tests for adding/removing nodes

To run tests, build the project and execute the test binaries from `cmake/build/`:
```sh
cd cmake/build
./correctness
./performance_v2
./reliability_v2
```

## Project Structure and Dependencies

### External Dependencies
The project relies on several external libraries, which are included and linked via the `CMakeLists.txt` and `common.cmake` files:
1. **gRPC**: The C++ gRPC library is used for client-server communication.
2. **Protobuf**: Protocol Buffers (protobuf) is used for serializing structured data.
3. **Abseil**: Abseil is a collection of C++ libraries.
4. **RocksDB**: RocksDB is used for database operations.

### CMake Configuration
1. The `CMakeLists.txt` file contains references to all the external libraries required by the project.
2. The `common.cmake` file is responsible for including gRPC, protobuf, and Abseil libraries.

### Linking External Libraries
Make sure that any external library is correctly linked in the `CMakeLists.txt` or `common.cmake` file if additional libraries are added in the future.

## Architecture

### Snapshotting and Log Compaction
- Snapshots are automatically created when the log exceeds 10,000 entries
- Snapshots serialize the entire state machine (all key-value pairs)
- After snapshot creation, log entries up to the snapshot index are compacted
- Followers that fall behind can receive snapshots via `InstallSnapshot` RPC

### Write-Ahead Logging (WAL)
- WAL is configured with:
  - TTL: 3600 seconds (1 hour)
  - Size limit: 512MB per WAL file
  - Recovery mode: Point-in-time recovery
- All critical writes use `sync=true` for durability

### Multi-threaded Replication
- Replication uses a thread pool for parallel operations
- Each peer has dedicated replication pipeline
- Replication lag and node health are tracked in real-time via Prometheus metrics

## Monitoring

The system exposes Prometheus metrics on port 9090. You can:
- Scrape metrics: `curl http://localhost:9090/metrics`
- Integrate with Prometheus server for long-term monitoring
- Set up Grafana dashboards for visualization

## Notes
- Ensure that your `$MY_INSTALL_DIR` variable points to the correct path where dependencies are installed (especially for gRPC).
- If encountering errors related to missing dependencies, ensure that all external libraries are installed and correctly linked.
- Metrics server starts automatically on port 9090 when a Raft server starts
- Snapshots are created automatically; no manual intervention required

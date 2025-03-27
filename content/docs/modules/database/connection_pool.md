---
title: Connection Pool
weight: 1
---

The ODY framework implements a robust database connection pool that leverages Swoole's coroutines for 
high-performance database access. Here's an overview of how it works:

## Core Concepts

The connection pool manages a collection of database connections that can be borrowed, used, and returned 
without the overhead of establishing new connections for each database operation.

### Key Features

- **Coroutine-Aware**: Binds connections to Swoole coroutines for automatic management
- **Connection Reuse**: Maintains a pool of pre-established connections to reduce connection overhead
- **Auto-Return**: Automatically returns connections to the pool when coroutines complete
- **Configurable Sizing**: Adjustable pool size based on workload requirements
- **Connection Health Checks**: Validates connections before they're borrowed
- **Leak Detection**: Identifies and logs potential connection leaks
- **Connection Lifecycle Management**: Handles connection creation, validation, and cleanup

## How It Works

1. **Initialization**:
   ```php
   ConnectionManager::initPool($config, $name);
   ```
   This creates a new connection pool with the specified configuration.

2. **Borrowing a Connection**:
   ```php
   $connection = ConnectionManager::getConnection($name);
   ```
   This gets a PDO connection from the pool, binding it to the current coroutine.

3. **Auto-Return**: When the coroutine completes, the connection is automatically returned to the pool if `autoReturn` is enabled.

4. **Manual Return**: If `autoReturn` is disabled, you must explicitly return connections:
   ```php
   ConnectionManager::getPool($name)->return($connection);
   ```

5. **Pool Management**: The pool automatically scales between minimum idle connections and maximum size based on demand.

## Configuration Options

- `size`: Maximum number of connections in the pool
- `minimumIdle`: Minimum number of idle connections to maintain
- `idleTimeoutSec`: Time after which idle connections are removed
- `maxLifetimeSec`: Maximum lifetime of a connection before recreation
- `borrowingTimeoutSec`: Maximum time to wait when borrowing a connection
- `autoReturn`: Whether to automatically return connections
- `bindToCoroutine`: Whether to bind connections to coroutines

## Connection Lifecycle

1. **Creation**: New connections are created as needed up to the pool size limit
2. **Validation**: Before a connection is borrowed, it's checked for validity
3. **Usage**: The connection is marked as in-use while borrowed
4. **Return**: The connection is returned to the pool after use
5. **Maintenance**: Idle connections are periodically checked and renewed
6. **Cleanup**: Connections are properly closed when the pool is shutdown

By using this connection pool, database operations become more efficient, especially under high concurrency, as 
connection creation overhead is minimized.
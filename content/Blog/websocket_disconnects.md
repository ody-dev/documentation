---
title: Resolving Unstable WebSocket Connections in Swoole Multi-Worker Environments
date: 2025-03-20
authors:
  - name: Ilyas Deckers
    link: https://github.com/ilyasdeckers
tags:
  - Swoole
  - PHP
  - WebSocket
  - WebSockets
excludeSearch: true
---

When running a Swoole WebSocket server with multiple worker processes, connections may randomly disconnect or appear 
disconnected when attempting to broadcast messages. This occurs despite clients maintaining active connections.

Typical symptoms include:
- WebSocket connections work correctly with a single worker (`worker_num = 1`)
- With multiple workers, the server reports clients as disconnected shortly after they connect
- Log entries show: "Client X is no longer connected, skipping" despite the client being active
- Broadcasting to channels fails intermittently

## Root Cause

The issue stems from Swoole's process mode and connection dispatch strategy:

1. When running in `SWOOLE_PROCESS` mode with multiple workers, Swoole distributes connections across different worker processes
2. While the default `dispatch_mode=2` should theoretically work for WebSockets (as it ensures data from the same connection is processed by the same worker), in practice issues can still occur
3. The problem likely involves how HTTP requests (for broadcasting) and WebSocket connections are managed separately:
    - Worker A maintains a WebSocket connection
    - Worker B processes an HTTP broadcast request but cannot access Worker A's connection information
    - The broadcast fails as Worker B cannot find the connection in its memory

We also observed that switching to `SWOOLE_BASE` mode altered the behavior, but did not resolve the issue. In `SWOOLE_BASE` mode:
- Each worker directly accepts its own connections without a master process dispatcher
- The `dispatch_mode` setting is ignored entirely since there's no centralized dispatch mechanism
- Connections were still inconsistently managed, suggesting that the underlying issue involves how connection state is maintained across different types of requests

## Solution

While `dispatch_mode=2` should theoretically work, testing has shown that `dispatch_mode=5` (UID hash) provides more reliable connection management for WebSocket servers with multiple workers:

```php
$server = new Swoole\WebSocket\Server('0.0.0.0', 9502, SWOOLE_PROCESS);
$server->set([
    'worker_num' => swoole_cpu_num() * 2,
    'dispatch_mode' => 5
]);
```

Note: According to the documentation, both modes should ensure consistent worker assignment:
- `dispatch_mode=2`: Allocates based on connection's file descriptor
- `dispatch_mode=5`: Allocates based on UID (requires explicit binding)

For SSL/TLS support:

```php
$server = new Swoole\WebSocket\Server(
    '0.0.0.0', 
    9502, 
    SWOOLE_PROCESS | SWOOLE_SSL
);
$server->set([
    'worker_num' => swoole_cpu_num() * 2,
    'dispatch_mode' => 5,
    'ssl_cert_file' => '/path/to/cert.pem',
    'ssl_key_file' => '/path/to/key.pem'
]);
```

This ensures that all messages from the same connection are consistently routed to the same worker process, maintaining connection state integrity.

## Server Mode Considerations

Our testing revealed key differences between Swoole's two server modes:

1. **SWOOLE_PROCESS Mode:**
    - Connections are managed by a master process and dispatched to workers
    - `dispatch_mode` setting is critical for consistent connection handling
    - Setting `dispatch_mode=5` resolved our connection stability issues
    - `dispatch_mode=2` (the default) did not provide reliable connection handling despite documentation suggesting it should

2. **SWOOLE_BASE Mode:**
    - No master process; each worker accepts connections directly
    - `dispatch_mode` setting is ignored in this mode
    - We experienced similar connection instability issues as with SWOOLE_PROCESS
    - Connection affinity is determined by which worker happens to accept each connection

## Alternative Approaches

1. Try both `dispatch_mode=2` and `dispatch_mode=5` to see which works better for your specific use case
2. Use shared memory structures like Swoole Tables to maintain connection information across workers
3. Implement an external state store (Redis/memcached) for connection tracking
4. If using `SWOOLE_BASE` mode, limit to a single worker for simplicity (at the cost of scalability)
5. Ensure HTTP requests for broadcasting are properly associated with WebSocket connections

## References

- Swoole documentation on [dispatch_mode](https://openswoole.com/docs/modules/swoole-server-doc)
- Swoole WebSocket [server implementation](https://openswoole.com/docs/modules/swoole-websocket-server-doc)
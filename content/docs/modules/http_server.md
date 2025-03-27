---
title: HTTP Server
weight: 6
---

## Introduction

ODY Server is a modern, high-performance, event-driven web server for PHP applications built on top of Swoole. It
provides a robust foundation for web applications that need to handle concurrent requests efficiently,
with minimal resource usage and fast response times.

Unlike traditional PHP setups with Apache or Nginx, ODY Server keeps your application in memory between
requests, dramatically improving performance while providing non-blocking I/O operations through coroutines.

How It Works:
1. The server starts as a long-running process, avoiding the overhead of repeatedly loading PHP scripts.
2. Event-Driven Handling – Requests are processed asynchronously using an event loop, allowing the server to handle thousands of connections concurrently.
3. Non-Blocking I/O – The server can process multiple requests without waiting for I/O operations (e.g., database queries, file reads) to complete.
4. Worker Processes – The server can spawn multiple worker processes to distribute requests across CPU cores for better performance.
5. Developers can define custom request handlers, set custom headers, and control responses directly within the server logic.

## Installation

Add ODY Server to your project using Composer:

```bash
composer require ody/server
```

## Command Line Tools

ODY Server comes with convenient command-line tools to manage your server instance.

### Starting the Server

```bash
php ody server:start
```

Options:
- `-d, --daemonize` - Run the server in the background
- `-w, --watch` - Enable hot reloading for development

### Stopping the Server

```bash
php ody server:stop
```

### Reloading Workers

Reload worker processes without downtime to pick up code changes:

```bash
php ody server:reload
```

## Configuration

ODY Server can be configured in your project's `config/server.php` file:

```php
return [
    'host' => env('SERVER_HOST', '127.0.0.1'),
    'port' => env('SERVER_PORT', 9501),
    
    // Number of worker processes
    'additional' => [
        'worker_num' => env('SERVER_WORKERS', swoole_cpu_num()),
        
        // Enable coroutines for better performance
        'enable_coroutine' => true,
        
        // Maximum number of coroutines
        'max_coroutine' => 3000,
    ],
    
    // Files/directories to watch in development mode
    'watch' => [
        'app',
        'config',
        'routes',
        '.env',
    ]
];
```

### Environment Variables

You can configure the server using these environment variables in your `.env` file:

```
SERVER_HOST=127.0.0.1
SERVER_PORT=9501
SERVER_WORKERS=4
```

## Performance Tuning

### Worker Count

For optimal performance, set the worker count based on your server's CPU cores:

```php
'worker_num' => swoole_cpu_num() * 2,
```

### Coroutine Settings

Enable coroutines for non-blocking I/O operations:

```php
'enable_coroutine' => true,
'max_coroutine' => 3000,
```

## Hot Reloading

During development, you can enable hot reloading to automatically restart workers when code changes:

```bash
php ody server:start -w
```

Configure which files/directories to watch in your `server.php` config:

```php
'watch' => [
    'app',
    'config',
    'routes',
    'composer.lock',
],
```

## Standalone Usage

```php
use Ody\Server\ServerManager;
use Ody\Server\ServerType;
use Ody\Foundation\HttpServer;

// Initialize server
ServerManager::init(ServerType::HTTP_SERVER)
    ->createServer([
        'host' => '127.0.0.1',
        'port' => 9501,
        'mode' => SWOOLE_PROCESS,
        'sock_type' => SWOOLE_SOCK_TCP,
    ])
    ->setServerConfig([
        'worker_num' => 4,
        'enable_coroutine' => true,
    ])
    ->getServerInstance()
    ->start();
```

The `HttpServerState` class manages the state of running server processes, allowing for tracking and management of
processes.

```php
$serverState = HttpServerState::getInstance();

// Check if the server is running
if ($serverState->httpServerIsRunning()) {
    // Server is running
}

// Get process IDs
$masterPid = $serverState->getMasterProcessId();
$managerPid = $serverState->getManagerProcessId();
$workerPids = $serverState->getWorkerProcessIds();

// Kill processes
$serverState->killProcesses([
    $masterPid,
    $managerPid,
    // ...worker PIDs
]);

// Reload processes
$serverState->reloadProcesses([
    $masterPid,
    $managerPid,
    // ...worker PIDs
]);

// Clear process IDs
$serverState->clearProcessIds();
```

## Troubleshooting

### Address Already in Use

If you get an "Address already in use" error:

```bash
# Find the process using the port
lsof -i :9501
kill -9 {process_id}

# Stop the server properly
php ody server:stop
```

### Worker Crashes

If worker processes crash frequently, check your logs and consider:

1. Increasing memory limit in PHP settings
2. Reducing the worker count
3. Checking for code errors in event loops

## Deployment

For production deployment:

1. Set up a process manager like Supervisor
2. Run in daemon mode: `php ody server:start -d`
3. Configure a reverse proxy (Nginx) for SSL termination

Example Supervisor config:

```ini
[program:ody-server]
command=php /path/to/project/ody server:start
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/path/to/project/storage/logs/supervisor.log
```

## API Reference

### Server Types

The `ServerType` class provides constants for different server types:

- `HTTP_SERVER`: Swoole HTTP server (`\Swoole\Http\Server`)
- `WS_SERVER`: Swoole WebSocket server (`\Swoole\WebSocket\Server`)
- `TCP_SERVER`: Swoole TCP server (`\Swoole\Server`)

### Server Events

The `ServerEvent` class provides constants for all supported server events:

- `ON_START`: Server start event
- `ON_WORKER_START`: Worker process start event
- `ON_WORKER_STOP`: Worker process stop event
- `ON_WORKER_EXIT`: Worker process exit event
- `ON_WORKER_ERROR`: Worker process error event
- `ON_PIPE_MESSAGE`: Pipe message event
- `ON_REQUEST`: HTTP request event
- `ON_RECEIVE`: Data receive event
- `ON_CONNECT`: Client connect event
- `ON_DISCONNECT`: Client disconnect event
- `ON_OPEN`: WebSocket open event
- `ON_MESSAGE`: WebSocket message event
- `ON_CLOSE`: Connection close event
- `ON_TASK`: Task event
- `ON_FINISH`: Task finish event
- `ON_SHUTDOWN`: Server shutdown event
- `ON_PACKET`: UDP packet event
- `ON_MANAGER_START`: Manager start event
- `ON_MANAGER_STOP`: Manager stop event
- `ON_BEFORE_START`: Before server start event (not a Swoole event)

### ServerManager Class

Methods:

- `init(string $serverType): static` - Initialize the server manager with a server type
- `createServer(?array $config): static` - Create a new server instance with the given configuration
- `setServerConfig(array $config): static` - Set additional server configuration
- `getServerInstance(): HttpServer|WsServer` - Get the server instance
- `registerCallbacks(array $callbacks): static` - Register event callbacks
- `setWatcher(int $enableWatcher, array $paths, object $serverState): static` - Enable file watching for hot reloading
- `daemonize(bool $daemonize): static` - Set the server to run in the background
- `setLogger(LoggerInterface $logger): self` - Set the logger instance
- `setConfig(Config $config): self` - Set the configuration instance
- `start(): void` - Start the server

### ServerState Class

Methods:

- `getInstance(): self` - Get the singleton instance
- `getInformation(): array` - Get the state information
- `setManagerProcessId(?int $id): void` - Set the manager process ID
- `setMasterProcessId(?int $id): void` - Set the master process ID
- `setWatcherProcessId(?int $id): void` - Set the watcher process ID
- `setWorkerProcessIds(array $ids): void` - Set the worker process IDs
- `getManagerProcessId(): int|null` - Get the manager process ID
- `getMasterProcessId(): int|null` - Get the master process ID
- `getWatcherProcessId(): int|null` - Get the watcher process ID
- `getWorkerProcessIds(): array` - Get the worker process IDs
- `clearProcessIds(): void` - Clear all process IDs
- `reloadProcesses(array $processIds): void` - Reload the specified processes
- `killProcesses(array $processIds): void` - Kill the specified processes

### HttpServerState Class

Methods:

- `getInstance(): self` - Get the singleton instance
- `httpServerIsRunning(): bool` - Check if the HTTP server is running

## License

ODY Server is open-sourced software licensed under the MIT License.
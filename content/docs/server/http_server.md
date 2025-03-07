---
title: HTTP Server
---

## Introduction

ODY's HTTP server is a high-performance, event-driven server that handles HTTP requests asynchronously by leveraging 
Swoole's server. Unlike traditional PHP applications running on Apache or Nginx with PHP-FPM, Swoole’s HTTP server runs as a 
standalone process, efficiently managing multiple requests without needing to create a new PHP process for each request.

How It Works:
1. The server starts as a long-running process, avoiding the overhead of repeatedly loading PHP scripts.
2. Event-Driven Handling – Requests are processed asynchronously using an event loop, allowing the server to handle thousands of connections concurrently.
3. Non-Blocking I/O – The server can process multiple requests without waiting for I/O operations (e.g., database queries, file reads) to complete.
4. Worker Processes – The server can spawn multiple worker processes to distribute requests across CPU cores for better performance.
5. Developers can define custom request handlers, set custom headers, and control responses directly within the server logic.

## Installation
This package requires `ext-swoole` to be installed on your system.

```shell
composer install ody/server
```

### Configuration
Create a `config/server.php` file and add the following content:

```php
use Ody\HttpServer\ServerEvent;

return [
    'mode' => SWOOLE_BASE,
    'host' => env('HTTP_SERVER_HOST' , '127.0.0.1'),
    'port' => env('HTTP_SERVER_PORT' , 9501) ,
    'sock_type' => SWOOLE_SOCK_TCP,
    'additional' => [
        'worker_num' => env('HTTP_SERVER_WORKER_COUNT' , swoole_cpu_num() * 2) ,
        'open_http_protocol' => true,
        /**
         * log level
         * SWOOLE_LOG_DEBUG (default)
         * SWOOLE_LOG_TRACE
         * SWOOLE_LOG_INFO
         * SWOOLE_LOG_NOTICE
         * SWOOLE_LOG_WARNING
         * SWOOLE_LOG_ERROR
         */
        'log_level' => 1,
        'log_file' => base_path('/storage/logs/ody_server.log'),
        'log_rotation' => SWOOLE_LOG_ROTATION_DAILY,
        'log_date_format' => '%Y-%m-%d %H:%M:%S',

        // Coroutine
        'max_coroutine' => 3000,
        'send_yield' => false,
    ],

    'runtime' => [
        /**
         * enabling this will run clients like MySQL and Redis in a non-blocking fashion
         * https://wiki.swoole.com/en/#/runtime
         */
        'enable_coroutine' => true,
        /**
         * SWOOLE_HOOK_TCP - Enable TCP hook only
         * SWOOLE_HOOK_TCP | SWOOLE_HOOK_UDP | SWOOLE_HOOK_SOCKETS - Enable TCP, UDP and socket hooks
         * SWOOLE_HOOK_ALL - Enable all runtime hooks
         * SWOOLE_HOOK_ALL ^ SWOOLE_HOOK_FILE ^ SWOOLE_HOOK_STDIO - Enable all runtime hooks except file and stdio hooks
         * 0 - Disable runtime hooks
         */
        'hook_flag' => SWOOLE_HOOK_ALL,
    ],

    /**
     * Override default callbacks for server events
     */
    'callbacks' => [
        ServerEvent::ON_REQUEST => [\Ody\Core\Server\HttpServer::class, 'onRequest'],
        ServerEvent::ON_START => [\Ody\HttpServer\Server::class, 'onStart'],
        ServerEvent::ON_WORKER_ERROR => [\Ody\HttpServer\Server::class, 'onWorkerError'],
        ServerEvent::ON_WORKER_START => [\Ody\HttpServer\Server::class, 'onWorkerStart'],
    ],

    'ssl' => [
        'ssl_cert_file' => null ,
        'ssl_key_file' => null ,
    ] ,

    /**
     * Configure what directories or files must be
     * watched for hot reloading.
     */
    'watcher' => [
        'App',
        'config',
        'database',
        'composer.lock',
        '.env',
    ] 
];
```

### Starting the web server

```shell
php ody server:start [-d|run as daemon] [-w|eable a watcher]
```

## Event callbacks

ODY's HTTP server relies on event-driven programming, you can override the native implementation by providing callable methods in the config
file. These callbacks define how the server handles different stages of request processing and system behavior.

```php
    /**
     * Override default callbacks for server events
     */
    'callbacks' => [
        ServerEvent::ON_REQUEST => [\Ody\Core\Server\HttpServer::class, 'onRequest'],
        ServerEvent::ON_START => [\Ody\HttpServer\Server::class, 'onStart'],
        ServerEvent::ON_WORKER_ERROR => [\Ody\HttpServer\Server::class, 'onWorkerError'],
        ServerEvent::ON_WORKER_START => [\Ody\HttpServer\Server::class, 'onWorkerStart'],
    ],
    
    // Available events
    SeverEvent::ON_START = 'start';
    SeverEvent::ON_WORKER_START = 'workerStart';
    SeverEvent::ON_WORKER_STOP = 'workerStop';
    SeverEvent::ON_WORKER_EXIT = 'workerExit';
    SeverEvent::ON_WORKER_ERROR = 'workerError';
    SeverEvent::ON_PIPE_MESSAGE = 'pipeMessage';
    SeverEvent::ON_REQUEST = 'request';
    SeverEvent::ON_DISCONNECT = 'disconnect';
    SeverEvent::ON_MESSAGE = 'message';
    SeverEvent::ON_CLOSE = 'close';
    SeverEvent::ON_TASK = 'task';
    SeverEvent::ON_FINISH = 'finish';
    SeverEvent::ON_SHUTDOWN = 'shutdown';
    SeverEvent::ON_PACKET = 'packet';
    SeverEvent::ON_MANAGER_START = 'managerStart';
    SeverEvent::ON_MANAGER_STOP = 'managerStop';
```

### onRequest
triggers when a client request is received.

```php
function(\Swoole\Http\Request $request, \Swoole\Http\Response $response)
```

### onStart
This function is called in the main thread of the master process after the server starts
```php
function onStart(Swoole\Server $server);
```

### onBeforeShutdown
This event occurs before the Server normal shutdown

```php
function onBeforeShutdown(Swoole\Server $server);
```

### onShutdown
This event occurs when the Server is shutting down normally

```php
function onShutdown(Swoole\Server $server);
```

### onWorkerStart
This event occurs when the Worker process/ Task process starts, objects created here can be used throughout the process lifecycle.

```php
function onWorkerStart(Swoole\Server $server, int $workerId);
```

### onWorkerStop
This event occurs when the Worker process terminates. Resources allocated by the Worker process can be reclaimed in 
this function.

```php
function onWorkerStop(Swoole\Server $server, int $workerId);
```

### onWorkerExit
Only valid when the reload_async feature is enabled. See How to restart the service correctly

```php
function onWorkerExit(Swoole\Server $server, int $workerId);
```

### onPacket
Callback this function when receiving UDP data packets, it occurs in the worker process.

```php
function onPacket(Swoole\Server $server, string $data, array $clientInfo);
```

### onClose
This function is called in the Worker process after a TCP client connection is closed.

```php
function onClose(Swoole\Server $server, int $fd, int $reactorId);
```

### onTask
Called inside the task process. The worker process can use the task function to deliver new tasks to the task_worker 
process. The current Task process switches to busy state when calling the onTask callback function, meaning it will no 
longer receive new tasks. When the onTask function returns, the process switches to idle state to continue receiving 
new tasks.

```php
function onTask(Swoole\Server $server, int $task_id, int $src_worker_id, mixed $data);
```

### onFinish
This callback function is triggered in the worker process when the task initiated by the worker process is completed in 
the task process. The task process sends the result of the task processing to the worker process via the 
Swoole\Server->finish() method.

```php
function onFinish(Swoole\Server $server, int $task_id, mixed $data)
```

### onPipeMessage
When a working process receives a message sent by $server->sendMessage() trigger the onPipeMessage event. Both worker 
and task processes may trigger the onPipeMessage event.

```php
function onPipeMessage(Swoole\Server $server, int $src_worker_id, mixed $message);
```

### onWorkerError
This function will be called in the Manager process when a Worker/Task process encounters an exception.

```php
function onWorkerError(Swoole\Server $server, int $worker_id, int $worker_pid, int $exit_code, int $signal);
```

### onManagerStart
This event is triggered when the manager process starts

```php
function onManagerStart(Swoole\Server $server);
```

### onManagerStop
Triggered when the manager process ends

```php
function onManagerStop(Swoole\Server $server);
```

### onBeforeReload
This event is triggered before the Reload of the Worker process, and the callback is in the Manager process.

```php
function onBeforeReload(Swoole\Server $server);
```
### onAfterReload
This event is triggered after the Worker process Reload, and the callback is in the Manager process.

```php
function onAfterReload(Swoole\Server $server);
```
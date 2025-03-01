---
title: Websocket Server
---

## Introduction
The WebSocket server implementation is still in its early stages, and new features are being actively developed.

This package requires the ext-swoole extension to be installed on your system. Swoole provides high-performance, 
asynchronous networking capabilities, making it ideal for handling real-time WebSocket connections efficiently. Before 
proceeding, ensure that your system meets the necessary requirements for running Swoole and WebSockets.

## Installation
To get started, install the Composer package and configure the server by editing the config file or setting environment 
variables in your .env file. The configuration process is straightforward, allowing you to customize the WebSocket server 
to suit your application’s needs.


```shell
composer install ody/websocket-server
```

### Configuration
Create a `config/websockets.php` file and add the following content:

```php
<?php
use Ody\Swoole\Event;

return [
    'host' => env('WEBSOCKET_HOST', '127.0.0.1'),
    'port' => env('WEBSOCKET_PORT', 9502),
    'sock_type' => SWOOLE_SOCK_TCP,
    'callbacks' => [
        Event::ON_HAND_SHAKE => [\Ody\Websocket\Server::class, 'onHandShake'],
        Event::ON_MESSAGE => [\Ody\Websocket\Server::class, 'onMessage'],
        Event::ON_CLOSE => [\Ody\Websocket\Server::class, 'onClose'],
        Event::ON_REQUEST => [\Ody\Websocket\Server::class, 'onRequest'],
        Event::ON_DISCONNECT => [\Ody\Websocket\Server::class, 'onDisconnect'],
    ],
    'secret_key' => env('WEBSOCKET_SECRET_KEY', '123123123'),
    "additional" => [
        "worker_num" => env('WEBSOCKET_WORKER_COUNT', swoole_cpu_num() * 2),
        /*
         * log level
         * SWOOLE_LOG_DEBUG (default)
         * SWOOLE_LOG_TRACE
         * SWOOLE_LOG_INFO
         * SWOOLE_LOG_NOTICE
         * SWOOLE_LOG_WARNING
         * SWOOLE_LOG_ERROR
         */
        'log_level' => SWOOLE_LOG_DEBUG ,
        'log_file' => storagePath('logs/ody_websockets.log') ,
    ]
];
```

Edit your `.env` file.

```php
WEBSOCKET_HOST=127.0.0.1
WEBSOCKET_PORT=9502
WEBSOCKET_SECRET_KEY=123123123
WEBSOCKET_WORKER_COUNT=8
```

## Usage

### Starting a websocket server
To run a websocket server run the following command:

```shell
php ody websocket start
```

This command accepts a `-d` flag, when enabled the server runs as a daemon.

```shell
php ody websocket:reload
php ody sebsocket:stop
```

### Websocket callbacks

By default, the WebSocket server listens for the following events:

* `onHandshake` - Triggered when a new WebSocket connection is being established. This event allows developers to customize the handshake process.
* `onMessage` - Fires when the server receives a message from a client. This is useful for processing incoming data and responding accordingly.
* `onClose` - Called when a WebSocket connection is closed. This allows for cleanup and resource management.
* `onRequest` - Handles incoming HTTP requests within the WebSocket server. This feature enables hybrid HTTP and WebSocket applications.
* `onDisconnect` - Triggered when a client disconnects from the server, allowing developers to track user sessions effectively.
* `onOpen` - Fired when a new WebSocket connection is successfully opened, enabling initial communication with the client.

These events are mapped to basic WebSocket server implementations. You can override them by creating a WebSocketController 
and defining the methods in the callbacks[] section of the config file. Custom implementations give developers full 
control over WebSocket interactions, making it possible to build dynamic, event-driven applications with real-time 
communication capabilities.

A very basic implementation of a `WebsocketController.php`:
```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers;

use Swoole\Http\Request;
use Swoole\Server;
use Swoole\Websocket\Frame;
use Swoole\WebSocket\Server as WsServer;

class WebSocketController
{
    public function onRequest(Request $request, Response $response): void
    {
        // Handle a regular http request.
    }
    
    public function onHandshake(Request $request,  Response $response): void
    {
        $server->push($frame->fd, 'Recv: ' . $frame->data);
    }
    
    public function onMessage(WsServer $server, Frame $frame): void
    {
        $server->push($frame->fd, 'Recv: ' . $frame->data);
    }

    public function onClose(WsServer $server, int $fd, int $reactorId): void
    {
        var_dump('closed');
    }

    public function onOpen(WsServer $server, Request $request): void
    {
        $server->push($request->fd, 'Opened');
    }
    
    public static function onDisconnect(WsServer $server, int $fd): void
    {
        //
    }
}
```

Callbacks for WebSocket servers are not triggered in the same coroutine, so that 
they cannot directly use the stored information of context. A context manager is provided by WebSocket 
Server component. On boot the server can be accessed through `ContextManager::get('WsServer')`. This can be usefull
to push to websocket clients for example in methods where no server property is accessible

Example:

```php
    /*
     * Loop through all the WebSocket connections to
     * send back a response to all clients. Broadcast
     * a message back to every WebSocket client.
     */
    public static function onRequest(Request $request,  Response $response): void
    {
        $server = ContextManager::get('WsServer');
        foreach($server->connections as $fd)
        {
            // Validate a correct WebSocket connection otherwise a push may fail
            if($server->isEstablished($fd))
            {
                $clientName = sprintf("Client-%'.06d\n", $fd);
                echo "Pushing event to $clientName...\n";
                $server->push($fd, $request->getContent());
            }
        }

        $response->status(200);
        $response->end();
    }
```

### Middleware

{{< callout type="error" >}}
Not yet implemented!
{{< /callout >}}

### Handling regular HTTP requests

In addition to handling WebSocket connections, the server can also process standard HTTP requests, making it a versatile
solution for real-time applications. These requests are handled by the `Event::onRequest callback`, which allows developers
to define custom logic for handling incoming HTTP traffic.

By leveraging this feature, you can serve both WebSocket and HTTP clients within the same application, reducing the
need for separate infrastructure. This means that alongside real-time WebSocket communication, the server can manage
tasks such as API requests, authentication, or even simple web page rendering, all within a single, unified system.

{{< callout type="info" >}}
Planned feature; Mapping the `onRequest` event to the application's Http kernel
{{< /callout >}}

### Keeping track of connections

On boot an in memory table is created to keep track of active connections.

```php
// Register a client
$table = ContextManager::get('FdsTable');
$table->$fds->set((string) $fd, [
    'fd' => $fd,
    'name' => sprintf($clientName)
]);
echo "Connection <{$fd}> open by {$clientName}. Total connections: " . static::$fds->count() . "\n";

// Look up a client
$sender = $table->fds->get(strval($frame->fd), "name");
echo "Received from " . $sender . ", message: {$frame->data}" . PHP_EOL;
        
// Delete a client
$table->fds->del((string) $fd);
```


---
title: Websocket Server
---

## Introduction
The current implementation of a websocket server is still in it's early stages.
Donwload the composer package and edit the config file or add env variables in your .env file.

This package requires `ext-swoole` to be installed on your system.

## Installation

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

### Middleware
WIP

### Handling regular HTTP requests

In addition to separating HTTP services and WebSocket services through ports, we can also listen for HTTP requests in 
a WebSocket server. Http requests get send to the `Event::onRequest` callback.

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

Out of the box the websocket listens for the following events:

* `onHandshake`
* `onMessage`
* `onClose`
* `onRequest`
* `onDisconnect`
* `onOpen`

These are mapped to a basic implementation of a websocket server. They can accept messages, send messages,...

You can override these native callbacks by creating a `WebsocketController` and configuring the methods in the `callback[]`
section of the config file.

A very basic implementation of a `WebsocketController.php` looks like this:
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


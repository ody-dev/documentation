---
title: Swoole Secrets - Building a Stealthy Admin API for Your PHP Applications
date: 2025-03-10
authors:
  - name: Ilyas Deckers
    link: https://github.com/ilyasdeckers
tags:
  - Swoole
  - PHP
excludeSearch: true
---

Ever felt like your PHP application is a mysterious black box in production? You deploy it, cross your fingers, and hope 
it doesn't decide to have an existential crisis at 3 AM? I've been there. That's why I got excited when I discovered 
Swoole's barely documented command system. It's like finding a secret passage in your own house!

While digging through Swoole's source code (yes, some of us do that for fun on weekends), I stumbled upon these intriguing 
functions: `addCommand()` and `command()` in an undocumented \Swoole\Server\Admin class. They're briefly mentioned in the official 
documentation, but they're incredibly powerful for creating admin tooling.

These functions allow you to register custom commands that can be executed across different Swoole processes - perfect 
for gathering metrics and performing administrative tasks without interfering with your main application.

## The Admin Server Dilemma

My first instinct was to create a separate Swoole HTTP server for admin functionality. Great idea in theory, but in practice:

```php
// Attempt #1: Separate server
$mainServer = new Swoole\Http\Server('0.0.0.0', 9501);
$adminServer = new Swoole\Http\Server('127.0.0.1', 9502); 
// Error: "server is running. unable to create Swoole\Http\Server"
```

Swoole wasn't having it. Two independent HTTP servers in the same process? That's a no-go.

Then I tried using Swoole's `listen()` method:

```php
// Attempt #2: Secondary port
$mainServer = new Swoole\Http\Server('0.0.0.0', 9501);
$adminPort = $mainServer->listen('127.0.0.1', 9502, SWOOLE_TCP);
// Problem: Can't configure worker_num separately for admin port and some methods to gather metrics only work on worker 0
```

Better, but still problematic because commands only work reliably with a single worker. My admin port inherited the 
multi-worker setup from the main server.

## The Elegant Solution: Coroutine HTTP Server

After some head-scratching I finally figured out how to spawn a second server which I could configure to my needs:

```php
$server = new Swoole\Http\Server('0.0.0.0', 9501);

// Register commands on the main server
$server->addCommand('get_stats', SWOOLE_SERVER_COMMAND_MASTER, 
    function ($server, $data) {
        return json_encode(['uptime' => time() - SERVER_START_TIME]);
    }
);

// Launch admin server in a coroutine
$server->on('start', function ($server) {
    Swoole\Coroutine::create(function () use ($server) {
        $adminServer = new Swoole\Coroutine\Http\Server('127.0.0.1', 9502);
        $adminServer->set([
            'worker_num' => 1
        ]);
        
        $adminServer->handle('/stats', function ($request, $response) use ($server) {
            $stats = $server->command('get_stats', 0, SWOOLE_SERVER_COMMAND_MASTER, '', true);
            $response->end(json_encode($stats));
        });
        
        $adminServer->start();
    });
});
```

This approach is like having your cake and eating it too. The coroutine-based HTTP server:

1. Runs inside the master process of your main server
2. Doesn't conflict with your main server's workers
3. Can execute commands on your main server
4. Stays isolated on localhost for security

## What Can You Monitor?

With this setup, the sky's the limit! You can create endpoints for:

- Server statistics (connections, request count, etc.)
- Worker information and metrics
- Memory usage and potential leaks
- Custom application metrics
- Graceful reloads or controlled shutdowns

## The Takeaway

Sometimes the best solutions in programming come from combining tools in unexpected ways. Swoole's undocumented command 
system paired with a coroutine HTTP server gives us a powerful, non-intrusive way to monitor our Swoole PHP applications in production.

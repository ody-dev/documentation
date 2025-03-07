---
title: Lifecycle
weight: 2
---
ODY has two modes of operation; It can be run as a regular PHP application like we are used to, or it can run on a 
coroutine based server developed on top of Swoole. To understand the lifecycle of a coroutine based application a good 
understanding of Swoole is required.

Both modes share the same application core components. (see section foundation) While developing ODY we stived to keep 
things as clear and familiar as possible to make development as easy as possible if you are already accustomed to 
frameworks like Laravel and Symfony.

[//]: # (## Coroutines)

[//]: # (### What are coroutines?)

[//]: # ()
[//]: # (Coroutines effectively solve the challenge of asynchronous non-blocking systems, but what exactly are they?)

[//]: # ()
[//]: # (By definition, coroutines are lightweight threads managed by user code rather than the OS kernel, meaning the user )

[//]: # (controls execution switches instead of the OS allocating CPU time. In Swoole, each Worker process has a coroutine )

[//]: # (scheduler that switches execution when an I/O operation occurs or when explicitly triggered. Since a process runs )

[//]: # (coroutines one at a time, there’s no need for synchronization locks like in multi-threaded programming.)

[//]: # ()
[//]: # (Within a coroutine, execution remains sequential. In an HTTP coroutine server, each request runs in its own coroutine. )

[//]: # (For example, if coroutine A handles request A and coroutine B handles request B, execution switches when coroutine A )

[//]: # (encounters an I/O operation &#40;e.g., a MySQL query&#41;. While coroutine A waits for the database response, coroutine B )

[//]: # (executes. Once the I/O completes, execution returns to coroutine A, ensuring non-blocking execution.)

[//]: # ()
[//]: # (However, to enable coroutine switching, operations like MySQL queries must be asynchronous and non-blocking—otherwise, )

[//]: # (the scheduler cannot switch coroutines, causing blocking, which defeats the purpose of coroutine-based programming.)

### The request lifecycle of ODY

Because ODY uses an asynchronous, event-driven architecture, it avoids the overhead of traditional PHP-FPM request 
lifecycle, making it ideal for high-performance applications. ODY's coroutines implementation allows for non-blocking 
I/O while maintaining a synchronous-like coding style.

In ODY, the request/response lifecycle follows an event-driven, coroutine-based model designed for high-performance 
asynchronous processing. Here's a high-level overview:

1. Server Initialization 
   * A Swoole HTTP/TCP/WebSocket server starts and listens for incoming connections. 
   * It runs as a long-lived process with worker and event loops.

2. Handling Incoming Requests
    * When a client sends a request, Swoole’s event loop accepts it without blocking.
    * The request data (headers, body, etc.) is parsed and passed to the registered request handler.

3. Processing the Request
    * Business logic executes inside a worker coroutine.
    * Swoole supports non-blocking I/O (e.g., MySQL, Redis, HTTP requests) via coroutines for high concurrency.
    * Custom middleware, routing, and controllers process the request.

4. Generating and Sending the Response
    * The handler constructs an HTTP response.
    * The response is sent back to the client via Swoole\Http\Response->end().

5. Connection Management
    * After responding, the connection may be closed or kept alive depending on headers (e.g., Connection: keep-alive).
    * Swoole reuses worker processes for handling new requests without restarting.

### How Coroutines Work

When a coroutine performs an I/O operation (e.g., MySQL query, HTTP request), it yields execution.
The event loop switches to another coroutine instead of blocking. Once the I/O operation completes, the coroutine 
resumes execution.

Example: Coroutine-based MySQL Query

Instead of using PDO (which is blocking), Swoole provides Swoole\Coroutine\MySQL:
```php
Swoole\Coroutine\run(function() {
$mysql = new Swoole\Coroutine\MySQL();
    $mysql->connect([
        'host' => '127.0.0.1',
        'user' => 'root',
        'password' => 'password',
        'database' => 'test'
    ]);

    $result = $mysql->query("SELECT * FROM users");
    print_r($result);
});
```
This executes without blocking the entire process.

### Middleware Handling in ODY

Swoole doesn’t have built-in middleware like traditional frameworks (Laravel, Symfony), but we have developed a kernel 
that registers and executes middleware like we are used to. Middleware gets registered at the application's boot time 
and gets executed at each request. Keep in mind that when using coroutines in middleware the events are non blocking.
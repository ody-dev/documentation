---
title: Lifecycle
weight: 2
---

The ODY framework is built around Swoole's coroutine-based architecture, providing high-performance API capabilities. 
Understanding the lifecycle of requests within this framework is essential for effectively developing and scaling your 
applications.

## Request Lifecycle Overview

When working with the ODY framework, a request flows through several distinct phases:

1. **Server Initialization**
2. **Request Reception**
3. **Request Processing**
4. **Response Generation**
5. **Resource Cleanup**

Let's explore each of these phases in detail.

### Server Initialization

Unlike traditional PHP applications that initialize the environment for each request, the ODY framework leverages Swoole 
to maintain a persistent application state:

```php
// Typically in your application's entry point
use Ody\Foundation\Bootstrap;
use Ody\Foundation\HttpServer;
use Ody\Server\ServerManager;
use Ody\Server\ServerType;

// Initialize the application once
$app = Bootstrap::init();
$app->bootstrap();

// Start the HTTP server
HttpServer::start(
    ServerManager::init(ServerType::HTTP_SERVER)
        ->createServer($config)
        ->setServerConfig($config['additional'])
        ->registerCallbacks($config['callbacks'])
        ->getServerInstance()
);
```

During this phase:
- The framework bootstraps the application once
- Service providers are registered and booted
- Routes are registered (but not processed yet)
- The HTTP server starts and listens for incoming connections

This is a key difference from traditional PHP applications - your application code remains resident in memory, ready to 
handle multiple requests without restarting.

### Request Reception

When a client sends a request to your application:

1. Swoole's event loop receives the connection
2. A new coroutine is created for each request via `Coroutine::create()`
3. The `onRequest` handler in `HttpServer` is triggered

```php
// From src/HttpServer.php
public static function onRequest(SwRequest $request, SwResponse $response): void
{
    Coroutine::create(function() use ($request, $response) {
        static::setContext($request);
        
        $callback = new RequestCallback(static::$app);
        $callback->handle($request, $response);
    });
}
```

This coroutine-based approach allows your application to handle thousands of concurrent connections efficiently, unlike 
traditional PHP which typically handles one request at a time.

### Request Processing

Once a request is received, it undergoes several processing steps:

1. **Context Initialization**: The request data is stored in the coroutine context using `ContextManager::set()`, making it accessible throughout the request lifecycle
2. **Request Conversion**: Swoole's HTTP request is converted to a PSR-7 compatible request
3. **Routing**: The router matches the request to a registered route
4. **Middleware Pipeline**: The request passes through a series of middleware
5. **Controller/Handler Execution**: The matched route handler (often a controller method) is executed

```php
// This is what happens inside RequestCallback::handle()
// Convert Swoole request to PSR-7
$serverRequest = $this->createServerRequest($request);

// Log the request start with request ID
logger()->debug("Processing request", [
    'request_id' => $requestId,
    'method' => $serverRequest->getMethod(),
    'path' => $serverRequest->getUri()->getPath()
]);

// Handle the request with middleware and routing
$psrResponse = $this->handler->handle($serverRequest);
```

The middleware pipeline is a crucial part of the framework, allowing operations like:
- Authentication and authorization
- CORS handling
- Request logging
- Body parsing (JSON, form data)
- Rate limiting

Each middleware can either pass the request to the next middleware in the chain or return a response directly.

### Response Generation

After the route handler processes the request, a response is generated:

1. The controller returns a PSR-7 compatible `ResponseInterface`
2. Middleware can modify the response on the way back through the pipeline
3. The PSR-7 response is converted to a Swoole HTTP response
4. The response is sent to the client

```php
// Inside RequestCallback::handle()
$psrResponse = $this->handler->handle($serverRequest);

// Convert PSR-7 response to Swoole response
$this->emit($psrResponse, $response);
```

This conversion process ensures compatibility with PSR-7 standards while leveraging Swoole's performance.

### Resource Cleanup

After the response is sent, the framework performs cleanup operations:

1. **Terminating Middleware**: Middleware implementing `TerminatingMiddlewareInterface` will execute `terminate()` methods
2. **Context Cleanup**: The coroutine context is cleared to prevent memory leaks
3. **Resource Release**: Any resources (like streams or connections) associated with the request are released

```php
// From MiddlewareManager::terminate()
public function terminate(ServerRequestInterface $request, ResponseInterface $response): void
{
    // Get middleware stack
    $stack = $this->getMiddlewareForRoute($method, $path, $controller, $action);
    
    // Process all middleware for termination
    foreach ($stack as $middleware) {
        // Check if it implements TerminatingMiddlewareInterface
        if ($instance instanceof TerminatingMiddlewareInterface) {
            $instance->terminate($request, $response);
        }
    }
    
    // Clear all data from the context manager
    ContextManager::clear();
}
```

This cleanup is important for maintaining application stability across many requests.

## Coroutines and Concurrency

A key aspect of the ODY framework is its use of Swoole's coroutines for handling concurrent requests. This differs from 
traditional PHP request handling:

**Traditional PHP**:
- One process/thread per request
- Application bootstraps for each request
- Limited concurrency based on server configuration

**ODY with Swoole**:
- One process can handle many requests via coroutines
- Application bootstraps once and stays resident
- High concurrency with minimal resource usage

Each request gets its own coroutine, allowing the framework to efficiently handle I/O operations without blocking. For example, when a database query is executing, the coroutine can be suspended, allowing other coroutines to run.

## Key Considerations for Developers

When developing applications with the ODY framework, keep these points in mind:

1. **State Management**: Since the application persists between requests, be careful with static variables that might leak state
2. **Coroutine Safety**: Use the `ContextManager` for request-specific data rather than global variables
3. **Resource Management**: Always clean up resources (connections, files) when done to prevent memory leaks
4. **Non-blocking I/O**: Leverage coroutine-compatible libraries for I/O operations to maintain concurrency benefits

## Best Practices

- Use dependency injection instead of static methods where possible
- Implement terminating middleware for operations that should happen after the response is sent
- Leverage the Context Manager for request-specific data
- Optimize route registration to improve performance
- Use middleware strategically to keep controllers focused on business logic

Understanding this lifecycle will help you build efficient, reliable applications with the ODY framework and take full 
advantage of Swoole's coroutine-based architecture.
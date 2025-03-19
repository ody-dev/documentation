---
title: Middleware
weight: 2
---

## Introduction

Middleware in the ODY framework provides a mechanism to filter HTTP requests entering your application. It operates as 
a series of "layers" that the HTTP request must pass through before it reaches your application, and then again as the 
response travels back to the client.

Middleware allows you to:

- Perform authentication and authorization checks
- Log request information
- Transform requests or responses
- Handle cross-origin resource sharing (CORS)
- Rate limit API endpoints
- And much more

This document explains how to use middleware in your ODY applications.

## Middleware Flow

When a request enters your application, it passes through a pipeline of middleware:

1. The request starts at the outermost middleware
2. Each middleware can:
    - Modify the request
    - Pass the request to the next middleware in the pipeline
    - Return a response early, preventing deeper middleware or the route handler from executing
3. After the request has been handled, the response travels back through the same middleware in reverse order
4. Each middleware can modify the response before it's sent back to the client

## Types of Middleware

### Global Middleware

Global middleware runs on every HTTP request to your application. This is ideal for functionality like:

- Logging
- CORS handling
- Error handling
- Session management

### Route Middleware

Route middleware is applied only to specific routes or groups of routes. This is useful for:

- Authentication for specific sections of your app
- Rate limiting certain endpoints
- Validating input for particular routes

### Controller Middleware

Controller middleware can be applied to specific controllers or controller methods using PHP 8 attributes.

### Middleware Groups

Middleware groups allow you to combine multiple middleware under a single name, making it easier to apply several 
middleware at once.

## Creating Middleware

To create a middleware in the ODY framework, implement the PSR-15 `MiddlewareInterface`:

```php
<?php

namespace App\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class ExampleMiddleware implements MiddlewareInterface
{
    /**
     * Process an incoming server request
     *
     * @param ServerRequestInterface $request
     * @param RequestHandlerInterface $handler
     * @return ResponseInterface
     */
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        // Perform actions before the request is handled
        // (e.g., validate, transform, or check conditions)
        
        // Pass the request to the next middleware in the pipeline
        $response = $handler->handle($request);
        
        // Perform actions after the request is handled
        // (e.g., modify the response, add headers)
        
        return $response;
    }
}
```

### Using Coroutines in Middleware

ODY framework leverages Swoole's coroutines for high-performance concurrency. When writing middleware, you can use 
coroutines to perform non-blocking operations:

```php
<?php

namespace App\Middleware;

use Ody\Swoole\Coroutine\ContextManager;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class AsyncOperationMiddleware implements MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        // Store data in the coroutine context
        ContextManager::set('request_time', microtime(true));
        
        // Perform a non-blocking operation
        \Swoole\Coroutine::create(function() {
            // This runs in a separate coroutine
            // Perform background tasks without blocking the request
            logger()->info("Background operation started");
            // ...
        });
        
        // Continue processing the request
        return $handler->handle($request);
    }
}
```

## Registering Middleware

### Global Middleware

Global middleware is registered in your application service provider or configuration:

```php
// config/middleware.php
return [
    'global' => [
        \App\Middleware\LoggingMiddleware::class,
        \App\Middleware\CorsMiddleware::class,
        \App\Middleware\JsonBodyParserMiddleware::class,
    ],
    // ...
];
```

### Route Middleware

Route middleware can be applied to individual routes:

```php
// routes/web.php
$router->get('/dashboard', 'DashboardController@index')
    ->middleware(\App\Middleware\AuthMiddleware::class);
```

Or to route groups:

```php
// routes/api.php
$router->group(['middleware' => \App\Middleware\ApiAuthMiddleware::class], function($router) {
    $router->get('/users', 'UserController@index');
    $router->post('/users', 'UserController@store');
});
```

### Controller Middleware using Attributes

You can apply middleware directly to controllers or methods using attributes:

```php
<?php

namespace App\Controllers;

use Ody\Foundation\Middleware\Attributes\Middleware;
use Ody\Foundation\Middleware\Attributes\MiddlewareGroup;

#[Middleware(\App\Middleware\AuthMiddleware::class)]
class UserController
{
    #[Middleware(\App\Middleware\RoleMiddleware::class, ['requiredRole' => 'admin'])]
    public function store(Request $request, Response $response)
    {
        // Only authenticated users with 'admin' role can access this
        // ...
    }
    
    #[MiddlewareGroup('api')]
    public function list(Request $request, Response $response)
    {
        // This method uses all middleware in the 'api' group
        // ...
    }
}
```

### Middleware Groups

Define middleware groups in your configuration:

```php
// config/middleware.php
return [
    // ...
    'groups' => [
        'web' => [
            \App\Middleware\SessionMiddleware::class,
            \App\Middleware\CsrfMiddleware::class,
        ],
        'api' => [
            \App\Middleware\JsonBodyParserMiddleware::class,
            \App\Middleware\ThrottleMiddleware::class,
        ],
    ],
    // ...
];
```

Then use them in your routes:

```php
// routes/web.php
$router->group(['middleware' => 'web'], function($router) {
    $router->get('/', 'HomeController@index');
});

// routes/api.php
$router->group(['middleware' => 'api'], function($router) {
    $router->get('/users', 'ApiController@users');
});
```

## Middleware with Parameters

Parameters can be passed to middleware:

```php
// routes/api.php
$router->post('/files', 'FileController@upload')
    ->middleware(new \App\Middleware\ThrottleMiddleware(5, 1)); // 5 requests per minute
```

Or using configuration:

```php
// config/middleware.php
return [
    // ...
    'named' => [
        'auth' => \App\Middleware\AuthMiddleware::class,
        'throttle' => [
            'class' => \App\Middleware\ThrottleMiddleware::class,
            'parameters' => [
                'maxRequests' => 60,
                'minutes' => 1
            ]
        ],
    ],
    // ...
];
```

## Terminating Middleware

Some operations should run after the response has been sent to the browser. ODY framework supports "terminating" 
middleware that runs after the response is sent:

```php
<?php

namespace App\Middleware;

use Ody\Foundation\Middleware\TerminatingMiddlewareInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class LogResponseMiddleware implements MiddlewareInterface, TerminatingMiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        // Handle the request normally
        return $handler->handle($request);
    }
    
    public function terminate(ServerRequestInterface $request, ResponseInterface $response): void
    {
        // This runs after the response has been sent to the browser
        // Perfect for logging, cleanup, or other tasks that don't need to block the response
        logger()->info("Response completed", [
            'status' => $response->getStatusCode(),
            'time' => microtime(true)
        ]);
    }
}
```

## Available Middleware

ODY framework comes with several built-in middleware:

| Middleware | Description |
|------------|-------------|
| `AuthMiddleware` | Handles authentication with configurable guards |
| `CorsMiddleware` | Handles Cross-Origin Resource Sharing |
| `ErrorHandlerMiddleware` | Catches exceptions and returns formatted responses |
| `JsonBodyParserMiddleware` | Parses JSON request bodies |
| `LoggingMiddleware` | Logs request information with path exclusion capability |
| `RoleMiddleware` | Implements role-based access control |
| `ThrottleMiddleware` | Rate limits requests |

---
title: Middleware
weight: 2
---

Middleware provides a mechanism to filter HTTP requests/responses. The framework includes several built-in middleware
classes and supports custom middleware:

```php
<?php

namespace App\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class CustomMiddleware implements MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        // Perform actions before the request is handled
        
        // Pass the request to the next middleware in the stack
        $response = $handler->handle($request);
        
        // Perform actions after the request is handled
        
        return $response;
    }
}
```

Register your middleware in `config/app.php`:

```php
'middleware' => [
    'global' => [
        // Global middleware applied to all routes
        App\Middleware\CustomMiddleware::class,
    ],
    'named' => [
        // Named middleware that can be referenced in routes
        'custom' => App\Middleware\CustomMiddleware::class,
    ],
],
```
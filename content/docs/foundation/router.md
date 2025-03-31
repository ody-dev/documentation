---
title: Routing
weight: 5
---

## Basic Route Definition

Routes are defined in `routes/*.php` files, with the main application routes typically located in `routes/api.php`. 
The framework provides a fluent interface through the `Route` facade for defining routes.

```php
use Ody\Foundation\Facades\Route;

// Basic route definition
Route::get('/endpoint', 'App\Controllers\SomeController@method');

// You can also use a closure as a handler
Route::get('/simple', function ($request, $response) {
    return $response->withJson(['message' => 'Hello World']);
});
```

## Available HTTP Methods

The router supports all standard HTTP methods:

```php
Route::get('/resource', 'Controller@method');
Route::post('/resource', 'Controller@method');
Route::put('/resource', 'Controller@method');
Route::patch('/resource', 'Controller@method');
Route::delete('/resource', 'Controller@method');
Route::options('/resource', 'Controller@method');
```

## Route Parameters

You can define dynamic route parameters using curly braces:

```php
Route::get('/users/{id}', 'UserController@show');
```

These parameters will be passed to your controller method or closure:

```php
public function show(ServerRequestInterface $request, ResponseInterface $response, array $args)
{
    $userId = $args['id'];
    // ...
}
```

## Route Groups

Routes can be grouped to apply common prefixes, middleware, or other attributes:

```php
Route::group(['prefix' => '/admin', 'middleware' => ['auth']], function ($router) {
    $router->get('/dashboard', 'AdminController@dashboard');
    $router->get('/users', 'AdminController@users');
});
```

This creates routes at `/admin/dashboard` and `/admin/users` that are both protected by the `auth` middleware.

## Middleware

Middleware can be applied to individual routes or groups of routes:

```php
// Apply to a single route
Route::get('/profile', 'UserController@profile')->middleware('auth');

// Apply to a group of routes
Route::group(['middleware' => ['auth', 'admin']], function ($router) {
    // All routes in this group will use both middleware
});
```

## Named Routes

You can assign names to routes for easier reference:

```php
Route::get('/user/profile', 'UserController@profile')->name('profile');
```

## Response Handling

The framework supports PSR-7 compliant responses. You have several options for returning JSON responses:

```php
// Option 1: Using withJson() method (for Response instances)
return $response->withJson([
    'status' => 'success',
    'data' => $result
]);

// Option 2: Manual JSON encoding (works with any PSR-7 implementation)
$response = $response->withHeader('Content-Type', 'application/json');
$response->getBody()->write(json_encode($data));
return $response;
```

## Example Route Definitions

Here's an example of a complete route file:

```php
<?php

use Ody\Foundation\Facades\Route;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

// Authentication endpoints
Route::post('/auth/login', 'App\Controllers\AuthController@login');
Route::post('/auth/register', 'App\Controllers\AuthController@register');

// Protected group
Route::group(['prefix' => '/api', 'middleware' => ['auth']], function ($router) {
    $router->get('/users', 'App\Controllers\UserController@index');
    $router->post('/users', 'App\Controllers\UserController@store');
    $router->get('/users/{id}', 'App\Controllers\UserController@show');
    $router->put('/users/{id}', 'App\Controllers\UserController@update');
    $router->delete('/users/{id}', 'App\Controllers\UserController@destroy');
});

// Simple closure route
Route::get('/health', function (ServerRequestInterface $request, ResponseInterface $response) {
    return $response->withJson([
        'status' => 'ok',
        'timestamp' => time()
    ]);
});
```

## Performance Considerations

The routing system is built on FastRoute for efficient route matching and leverages Swoole's coroutines for concurrent 
request processing. This combination provides excellent performance for high-throughput API applications.

To maximize performance:
- Group related routes together
- Use middleware judiciously
- Consider the depth of your route hierarchy

## Further Reading

For more advanced routing scenarios, refer to the FastRoute documentation at [https://github.com/nikic/FastRoute](https://github.com/nikic/FastRoute).

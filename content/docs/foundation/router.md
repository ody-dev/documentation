---
title: Routing
---
Routes are defined in the `routes` directory. The framework supports various HTTP methods and route patterns:

```php
// Basic route definition
Route::get('/hello', function (ServerRequestInterface $request, ResponseInterface $response) {
    return $response->json([
        'message' => 'Hello World'
    ]);
});

// Route with named controller
Route::post('/users', 'App\Controllers\UserController@store');

// Route with middleware
Route::get('/users/{id}', 'App\Controllers\UserController@show')
    ->middleware('auth:api');

// Route groups
Route::group(['prefix' => '/api/v1', 'middleware' => ['throttle:60,1']], function ($router) {
    $router->get('/status', function ($request, $response) {
        return $response->json([
            'status' => 'operational'
        ]);
    });
});
```
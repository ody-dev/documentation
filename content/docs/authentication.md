---
title: Authentication
weight: 5
---   

## Overview

The Ody Framework provides a JWT-based authentication system that's flexible and easy to implement in your applications. 
This guide will walk you through the setup process and common authentication operations.

## Installation

To add authentication to your Ody Framework application, install the `ody/auth` package:

```bash
composer require ody/auth
```

## Configuration

### Basic Setup

1. Add the Auth Service Provider to your application's providers in `config/app.php`:

```php
'providers' => [
    // Other providers...
    Ody\Auth\Providers\AuthServiceProvider::class,
],
```

2. Create or modify your `.env` file to include authentication settings:

```
# Direct authentication (default)
AUTH_PROVIDER=direct
JWT_SECRET_KEY=your_secure_random_key_here

# Or for remote authentication
# AUTH_PROVIDER=remote
# AUTH_SERVICE_HOST=auth-service.example.com
# AUTH_SERVICE_PORT=9501
# SERVICE_ID=your_app_id
# SERVICE_SECRET=your_service_secret
```

3. Make sure your User model/repository implements the methods required for authentication:
    - `findByEmail(string $email)`
    - `findById($id)`
    - `getAuthPassword($userId)`

## Authentication Flows

### User Login

To authenticate a user:

1. **Send a POST request** to the login endpoint:

```php
// Route: POST /auth/login
$response = $client->post('/auth/login', [
    'email' => 'user@example.com',
    'password' => 'password123'
]);
```

2. **Response format** on successful login:

```json
{
    "message": "Login successful",
    "token": "eyJhbGciOiJIUzI1NiIsInR5...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5...",
    "expiresIn": 3600,
    "user": {
        "id": 1,
        "email": "user@example.com"
    }
}
```

3. **Store the tokens** in your client-side application for subsequent requests.

### Making Authenticated Requests

For any protected route, include the JWT token in the `Authorization` header:

```php
$client->request('GET', '/auth/user', [
    'headers' => [
        'Authorization' => 'Bearer ' . $token
    ]
]);
```

### Token Refresh

When your access token expires, use the refresh token to get a new one:

1. **Send a POST request** to the refresh endpoint:

```php
// Route: POST /auth/refresh
$response = $client->post('/auth/refresh', [
    'refreshToken' => 'eyJhbGciOiJIUzI1NiIsInR5...'
]);
```

2. **Response format** on successful refresh:

```json
{
    "token": "eyJhbGciOiJIUzI1NiIsInR5...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5...",
    "expiresIn": 3600
}
```

3. **Update the stored tokens** in your client-side application.

### Logout

To log out a user and invalidate their token:

```php
// Route: POST /auth/logout
$response = $client->post('/auth/logout', [], [
    'headers' => [
        'Authorization' => 'Bearer ' . $token
    ]
]);
```

## Protected Routes

### Creating Protected Routes

To protect your routes with authentication, apply the `auth` middleware:

```php
// Single route
Route::get('/protected-resource', 'ResourceController@show')->middleware('auth');

// Group of routes
Route::group(['middleware' => ['auth']], function ($router) {
    $router->get('/profile', 'UserController@profile');
    $router->put('/profile', 'UserController@update');
});
```

### Accessing the Authenticated User

In your protected route controllers, access the authenticated user from the request:

```php
public function profile(ServerRequestInterface $request)
{
    $user = $request->getAttribute('user');
    
    // Now you can use $user data
    return $this->response->withJson([
        'user' => $user
    ]);
}
```

## Error Handling

The authentication system will return appropriate HTTP response codes:

| Scenario | HTTP Status | Response Body |
|----------|-------------|---------------|
| Invalid credentials | 401 | `{"error": "Invalid credentials"}` |
| Missing token | 401 | `{"error": "Unauthorized"}` |
| Invalid/expired token | 401 | `{"error": "Unauthorized"}` |
| Invalid refresh token | 401 | `{"error": "Invalid refresh token"}` |
| Missing required fields | 422 | `{"error": "email and password are required"}` |

## Direct vs. Remote Authentication

### Direct Authentication

Uses the application's local user database for authentication. This is the default mode.

```
AUTH_PROVIDER=direct
JWT_SECRET_KEY=your_secure_random_key_here
```

### Remote Authentication

Connects to a centralized authentication service, ideal for microservice architecture.

```
AUTH_PROVIDER=remote
AUTH_SERVICE_HOST=auth-service.example.com
AUTH_SERVICE_PORT=9501
SERVICE_ID=your_app_id
SERVICE_SECRET=your_service_secret
```

## Security Best Practices

1. **Always use HTTPS** in production to protect tokens in transit
2. **Set appropriate token expiration times**:
    - Access tokens: shorter duration (e.g., 1 hour)
    - Refresh tokens: longer duration (e.g., 30 days)
3. **Revoke tokens** on password change or security incidents
4. **Store sensitive configuration** in environment variables
5. **Never store JWT tokens** in local storage (use HTTP-only cookies or memory)
6. **Implement CSRF protection** for cookie-based token storage

## Advanced Configuration

For more advanced configuration options, create a `config/auth.php` file:

```php
return [
    'driver' => [
        'provider' => env('AUTH_PROVIDER', 'direct'),
        'jwt_key' => env('JWT_SECRET_KEY', 'your_secret_key_for_development'),
        'token_expiry' => 3600, // 1 hour
        'refresh_token_expiry' => 86400 * 30, // 30 days

        // Remote auth service config
        'service_host' => env('AUTH_SERVICE_HOST', 'localhost'),
        'service_port' => env('AUTH_SERVICE_PORT', 9501),
        'service_id' => env('SERVICE_ID', 'api_service'),
        'service_secret' => env('SERVICE_SECRET', 'service_secret')
    ],

    'middleware' => [
        'auth' => \Ody\Auth\Middleware\AuthMiddleware::class,
    ],
];
```

For more detailed information about the authentication system's internals, please refer to the 
[Authentication System Architecture Documentation](README.md).
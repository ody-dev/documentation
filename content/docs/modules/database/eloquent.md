---
title: Eloquent
---

This guide explains how to use Laravel's Eloquent ORM with the ODY framework in a Swoole environment. The 
ODY database module provides high-performance connection pooling specifically designed for coroutines.

> **⚠️ IMPORTANT**: This module is compatible with Eloquent 11.x. Support for Eloquent 12.x is under development.

## Installation

Install the ODY database & Eloquent ORM package:

```bash
composer require ody/database illuminate/database:^11.43
```

## Configuration

### Basic Setup

Define your database configuration in a configuration file:

```php
// config/database.php
return [
    'environments' => [
        'local' => [
            'adapter' => 'mysql',
            'host' => env('DB_HOST', '127.0.0.1'),
            'port' => env('DB_PORT', 3306),
            'username' => env('DB_USERNAME', 'root'),
            'password' => env('DB_PASSWORD', 'root'),
            'db_name' => env('DB_DATABASE', 'ody'),
            'charset' => 'utf8mb4',
            'collation' => 'utf8mb4_general_ci',
            'prefix' => '',
            'options' => [
                // PDO::ATTR_EMULATE_PREPARES => true,
                PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
                PDO::ATTR_CASE,
                PDO::CASE_LOWER,
                PDO::MYSQL_ATTR_USE_BUFFERED_QUERY => true,
                PDO::MYSQL_ATTR_DIRECT_QUERY => false,
                // Max packet size for large data transfers
                // PDO::MYSQL_ATTR_MAX_BUFFER_SIZE => 16777216, // 16MB
            ],
            'pool' => [
                'enabled' => env('DB_ENABLE_POOL', false),
                'pool_name' => env('DB_POOL_NAME', 'default'),
                'connections_per_worker' => env('DB_POOL_CONN_PER_WORKER', 10),
                'minimum_idle' => 5,
                'idle_timeout' => 60.0,
                'max_lifetime' => 3600.0,
                'borrowing_timeout' => 1,
                'returning_timeout' => 0.5,
                'leak_detection_threshold' => 10.0,
            ]
        ],
        // Add more environments as needed (production, staging, etc.)
    ],
];
```
### Configure Service providers
Register the required service providers in your config/app.php:

```php

'providers' => [
    'http' => [
        // ... other providers
        \Ody\DB\Providers\DatabaseServiceProvider::class,
        \Ody\DB\Eloquent\Providers\EloquentServiceProvider::class,
    ]
],
```

## Basic Usage
### Using the DB Facade

```php
use Ody\DB\Eloquent\Facades\DB;

// Running queries
$users = DB::table('users')->where('active', 1)->get();
$user = DB::table('users')->find(1);
$count = DB::table('users')->count();

// Raw queries
$results = DB::select('SELECT * FROM users WHERE id = ?', [1]);

// Inserting data
DB::table('users')->insert([
    'name' => 'John Doe',
    'email' => 'john@example.com',
]);
```

### Using Eloquent Models

Define your models as you normally would with Eloquent:

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    protected $fillable = ['name', 'email'];
}
```

Then use them in your application:

```php
use App\Models\User;

// Find a user
$user = User::find(1);

// Create a new user
$user = User::create([
    'name' => 'Jane Doe',
    'email' => 'jane@example.com',
]);

// Query with relationships
$users = User::with('posts')->get();
```

## Transactions

The ODY DB facade provides coroutine-aware transaction methods:

```php
use Ody\DB\Eloquent\Facades\DB;

// Using callback approach (recommended)
DB::transaction(function () {
    DB::table('users')->update(['active' => false]);
    DB::table('logs')->insert(['message' => 'Users deactivated']);
});

// Manual transaction control
try {
    DB::beginTransaction();
    
    // Your database operations
    DB::table('users')->update(['active' => false]);
    DB::table('logs')->insert(['message' => 'Users deactivated']);
    
    DB::commit();
} catch (\Exception $e) {
    DB::rollback();
    throw $e;
}
```

## Connection Management

The ODY database module automatically manages connections within Swoole coroutines:

- Each coroutine gets its own connection from the pool
- Connections are automatically returned to the pool when the coroutine ends
- Connection health is monitored and maintained

## Performance Considerations

- The default pool size is 10 connections per worker. Adjust based on your needs
- Increasing the pool size too much can lead to excessive database connections
- For high-concurrency scenarios, adjust the connection pool settings:

```php
// config/database.php
return [
    // ... other config
    'pool_size' => 32,
    'pool_settings' => [
        'minimum_idle' => 5,
        'idle_timeout' => 30, // seconds
        'max_lifetime' => 3600, // seconds
        'borrowing_timeout' => 0.5, // seconds
    ],
];
```

## Advanced usage
The method below are handled in the provided service providers. You can use these bootstrapping methods for
custom implementations.

Initialize Eloquent in your application's bootstrap process:

```php
use Ody\DB\Eloquent\Eloquent;

// Load your configuration
$config = require 'config/database.php';
$environment = 'local'; // Or get from your app configuration

// Boot Eloquent with the configuration for the current environment
Eloquent::boot($config['environments'][$environment]);
```

## Eloquent Documentation

For detailed Eloquent ORM usage, refer to the [official Laravel Eloquent documentation](https://laravel.com/docs/11.x/eloquent)
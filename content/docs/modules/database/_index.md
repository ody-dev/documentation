---
title: Database
weight: 5
---

## Overview

The ODY Database module provides a connection pool implementation that works with popular PHP ORM solutions. It's
designed to maximize performance in Swoole environments by efficiently managing database connections across coroutines.

## Features

- Connection pooling with Swoole coroutine awareness
- Support for Eloquent ORM, Doctrine ORM, and standalone DBAL
- Automatic connection binding to coroutines
- Built-in connection lifecycle management
- Connection health checks and leak detection
- Configurable pool size and connection settings

## Installation

```bash
composer require ody/database
```

### Eloquent
```bash
composer require illuminate/database
```

### Doctrine ORM
```bash
composer require doctrine/orm doctrine/dbal symfony/cache
```

### DBAL

```bash
composer require doctrine/dbal
```

## Basic Usage
### Configuration

Define your database configuration:

```php
// config/database.php
return [
    'environments' => [
        'local' => [
            'driver' => 'mysql',
            'host' => 'localhost',
            'port' => 3306,
            'database' => 'your_database',
            'username' => 'your_username',
            'password' => 'your_password',
            'charset' => 'utf8mb4',
        ],
    ],
    'connection_pool_enabled' => true,
    'pool_size' => 10,
];
```

### Using with Eloquent

```php
use Ody\DB\Eloquent\Facades\DB;
use App\Models\User;

// Initialize Eloquent
Ody\DB\Eloquent\Eloquent::boot(config('database.environments')['local']);

// Using the Facade
$users = DB::table('users')->where('active', 1)->get();

// Using Eloquent models
$user = User::find(1);
```

### Using with Doctrine ORM

```php
use Ody\DB\Doctrine\Facades\ORM;
use App\Entities\User;

// Get entity manager
$entityManager = ORM::entityManager();

// Working with entities
$user = $entityManager->find(User::class, 1);
$entityManager->persist($user);
$entityManager->flush();
```

### Using with Doctrine DBAL

```php
use Ody\DB\Doctrine\Facades\DBAL;

// Execute queries
$users = DBAL::fetchAllAssociative('SELECT * FROM users WHERE active = ?', [1]);

// Using query builder
$queryBuilder = DBAL::createQueryBuilder();
$result = $queryBuilder
    ->select('u.*')
    ->from('users', 'u')
    ->where('u.active = :active')
    ->setParameter('active', 1)
    ->executeQuery()
    ->fetchAllAssociative();
```

### Direct Connection Pool Access

```php
use Ody\DB\ConnectionManager;

// Initialize the pool
ConnectionManager::initPool($config);

// Get a connection from the pool
$connection = ConnectionManager::getConnection();

// Use the PDO connection
$stmt = $connection->prepare('SELECT * FROM users WHERE id = ?');
$stmt->execute([1]);
$user = $stmt->fetch(PDO::FETCH_ASSOC);

// No need to return the connection - it's automatically returned when the coroutine ends
```

## Advanced Configuration

The connection pool can be finely tuned with options for:

- Minimum idle connections
- Idle timeout
- Maximum connection lifetime
- Borrowing timeout
- Connection health checks
- Leak detection threshold

For detailed documentation on advanced configuration and usage, refer to the full documentation.

## Performance Benefits

- Connections are reused across requests, eliminating the overhead of establishing new connections
- Automatic binding to coroutines ensures connection safety in concurrent environments
- Connection health checks prevent using broken connections
- Leak detection helps identify and fix connection leaks
- Configurable pool size matches your application needs and server resources

## License

This package is licensed under the [MIT License](https://github.com/ody-dev/ody-foundation/blob/master/LICENSE).
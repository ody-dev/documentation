---
title: DBAL
---

This guide provides a comprehensive overview of how to use Doctrine DBAL (Database Abstraction Layer) with the ODY framework for database operations.

## Overview

The ODY Database module provides a powerful DBAL implementation that:

- Utilizes Swoole coroutines for asynchronous database operations
- Implements connection pooling for efficient resource utilization
- Automatically manages connection lifecycle within coroutines
- Supports multiple database environments and configurations

## Installation

```bash
composer require ody/database doctrine/dbal
```

## Configuration

### Database Configuration

In your `config/database.php`:

```php
return [
    'charset' => 'utf8mb4',
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
            'prefix'    => '',
            'options' => [
                PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
                PDO::ATTR_CASE,
                PDO::CASE_LOWER,
                PDO::MYSQL_ATTR_USE_BUFFERED_QUERY => true,
                PDO::MYSQL_ATTR_DIRECT_QUERY => false,
            ],
            'pool' => [
                'enabled' => env('DB_ENABLE_POOL', false),
                'connections_per_worker' => 128,
                'minimum_idle' => 128,
                'idle_timeout' => 60.0,
                'max_lifetime' => 3600.0,
                'borrowing_timeout' => 2,
                'returning_timeout' => 1,
                'leak_detection_threshold' => 10.0,
                'keep_alive_check_interval' => 60.0,
            ]
        ],
        'production' => [
            // Production-specific configuration...
            'pool' => [
                'enabled' => env('DB_ENABLE_POOL', false),
                'connections_per_worker' => 64,
                'minimum_idle' => 32,
                // Other pool settings...
            ]
        ],
    ],
    'default_environment' => 'local',
    // Other configuration settings...
];
```

### Service Provider Registration

Register the required service providers in your application's providers configuration:

```php
'providers' => [
    // Other providers...
    Ody\DB\Providers\DatabaseServiceProvider::class,
    Ody\DB\Doctrine\Providers\DBALServiceProvider::class,
],
```

## Basic Usage

### Establishing a Connection

The DBAL implementation manages connections automatically, but you can access the connection directly using:

```php
use Doctrine\DBAL\Connection;

class YourService
{
    private Connection $connection;
    
    public function __construct(Connection $connection)
    {
        $this->connection = $connection;
    }
    
    // Your methods using the connection...
}
```

### Executing Queries

```php
// Simple queries
$users = $this->connection->fetchAllAssociative('SELECT * FROM users WHERE active = ?', [1]);

// Single value queries
$count = $this->connection->fetchOne('SELECT COUNT(*) FROM users');

// Single row
$user = $this->connection->fetchAssociative('SELECT * FROM users WHERE id = ?', [42]);

// Executed statements (for INSERT, UPDATE, DELETE)
$affected = $this->connection->executeStatement(
    'UPDATE users SET status = ? WHERE last_login < ?', 
    ['inactive', new \DateTime('-30 days')]
);
```

### Using Query Builder

DBAL provides a powerful query builder for constructing complex queries:

```php
$queryBuilder = $this->connection->createQueryBuilder();

$results = $queryBuilder
    ->select('u.*')
    ->from('users', 'u')
    ->leftJoin('u', 'profiles', 'p', 'u.id = p.user_id')
    ->where('u.status = :status')
    ->andWhere('u.created_at > :date')
    ->setParameter('status', 'active')
    ->setParameter('date', new \DateTime('-7 days'))
    ->orderBy('u.name', 'ASC')
    ->setMaxResults(10)
    ->executeQuery()
    ->fetchAllAssociative();
```

### Working with Transactions

```php
// Method 1: Using transactional callback
$this->connection->transactional(function (Connection $conn) {
    $conn->executeStatement('INSERT INTO orders (customer_id, total) VALUES (?, ?)', [123, 99.99]);
    $orderId = $conn->lastInsertId();
    
    $conn->executeStatement(
        'INSERT INTO order_items (order_id, product_id, quantity) VALUES (?, ?, ?)',
        [$orderId, 456, 1]
    );
    
    // Return value is passed through from the closure
    return $orderId;
});

// Method 2: Manual transaction management
$this->connection->beginTransaction();
try {
    // Execute queries...
    $this->connection->commit();
} catch (\Exception $e) {
    $this->connection->rollBack();
    throw $e;
}
```

## Advanced Usage

### Schema Management

```php
// Get the schema manager
$schemaManager = $this->connection->createSchemaManager();

// List tables
$tables = $schemaManager->listTableNames();

// Check if table exists
$tableExists = $schemaManager->tablesExist(['users']);

// Get table details
$table = $schemaManager->introspectTable('users');

// Create a new table
use Doctrine\DBAL\Schema\Table;
use Doctrine\DBAL\Types\Types;

$table = new Table('logs');
$table->addColumn('id', Types::INTEGER, ['autoincrement' => true]);
$table->addColumn('message', Types::TEXT);
$table->addColumn('level', Types::STRING, ['length' => 10]);
$table->addColumn('created_at', Types::DATETIME_IMMUTABLE);
$table->setPrimaryKey(['id']);
$table->addIndex(['level'], 'idx_log_level');

$schemaManager->createTable($table);
```

### Prepared Statements

For queries that are executed repeatedly:

```php
$stmt = $this->connection->prepare('INSERT INTO items (name, price) VALUES (?, ?)');

foreach ($items as $item) {
    $stmt->bindValue(1, $item['name']);
    $stmt->bindValue(2, $item['price']);
    $stmt->executeStatement();
}
```

### Custom Types

Register and use custom column types:

```php
// In your service provider or bootstrap code
use Doctrine\DBAL\Types\Type;
use Ody\DB\Doctrine\Types\JsonType;

// Register the type if not already registered
if (!Type::hasType('json')) {
    Type::addType('json', JsonType::class);
}

// Using the custom type in schema
$table->addColumn('metadata', 'json', ['notnull' => false]);

// When fetching data, the JSON will be automatically converted to PHP arrays
$data = $this->connection->fetchAssociative('SELECT * FROM items WHERE id = ?', [1]);
// $data['metadata'] is now a PHP array
```

## Connection Pooling

The ODY DBAL implementation includes an advanced connection pooling system specifically designed for Swoole's coroutine-based environment.

### Configuring the Pool

```php
'pool' => [
    'enabled' => true,                   // Enable/disable connection pooling
    'connections_per_worker' => 64,      // Maximum connections per Swoole worker
    'minimum_idle' => 32,                // Minimum idle connections to maintain
    'idle_timeout' => 60.0,              // Seconds before idle connection is closed
    'max_lifetime' => 3600.0,            // Maximum lifetime of a connection
    'borrowing_timeout' => 2,            // Seconds to wait when borrowing a connection
    'returning_timeout' => 1,            // Seconds to wait when returning a connection
    'leak_detection_threshold' => 10.0,  // Seconds to wait before considering a connection leaked
    'keep_alive_check_interval' => 60.0, // Keep-alive check interval (should be less than MySQL wait_timeout)
]
```

### How the Pool Works

1. Connections are automatically created and managed by the `ConnectionManager`
2. Each Swoole worker maintains its own connection pool
3. When a new query is executed, a connection is borrowed from the pool
4. When the coroutine finishes, the connection is automatically returned to the pool
5. Idle connections are periodically checked and closed if they exceed the idle timeout
6. Connections that aren't returned to the pool are detected and logged

## Performance Tips

### Batch Operations

For large batch operations:

```php
$this->connection->beginTransaction();
try {
    $stmt = $this->connection->prepare('INSERT INTO logs (message, level) VALUES (?, ?)');
    
    foreach ($logs as $i => $log) {
        $stmt->bindValue(1, $log['message']);
        $stmt->bindValue(2, $log['level']);
        $stmt->executeStatement();
        
        // Commit in batches to prevent memory issues
        if ($i > 0 && $i % 1000 === 0) {
            $this->connection->commit();
            $this->connection->beginTransaction();
        }
    }
    
    $this->connection->commit();
} catch (\Exception $e) {
    $this->connection->rollBack();
    throw $e;
}
```

### Optimizing Queries

1. **Use specific columns** instead of `SELECT *`
2. **Add proper indexes** for frequently queried columns
3. **Use parameter binding** instead of string concatenation
4. **Limit result sets** when possible

## Working with Multiple Database Connections

To work with multiple database connections:

```php
use Doctrine\DBAL\Connection;
use Psr\Container\ContainerInterface;

class MultiDatabaseService
{
    private ContainerInterface $container;
    
    public function __construct(ContainerInterface $container)
    {
        $this->container = $container;
    }
    
    public function doSomethingWithMultipleDatabases()
    {
        // Get the default connection
        $defaultConnection = $this->container->get(Connection::class);
        
        // Get a specific connection
        $factory = $this->container->get('dbal.connection.factory');
        $analyticsConnection = $factory('analytics');
        
        // Use both connections
        $users = $defaultConnection->fetchAllAssociative('SELECT * FROM users');
        $stats = $analyticsConnection->fetchAllAssociative('SELECT * FROM statistics');
        
        // Process data from both sources...
    }
}
```

## License

This package is open-sourced software licensed under the [MIT license](https://opensource.org/licenses/MIT).

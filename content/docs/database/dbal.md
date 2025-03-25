---
title: DBAL
---

## Installation

```bash
composer require ody/database doctrine/dbal
```

## Configuration

### Basic Configuration

In your `config/database.php`:

```php
return [
    'connection_pool_enabled' => env('DB_CONNECTION_POOL_ENABLED', true),
    'connection_pool_size' => env('DB_CONNECTION_POOL_SIZE', 32),
    
    'environments' => [
        'local' => [
            'driver' => env('DB_DRIVER', 'mysql'),
            'host' => env('DB_HOST', 'localhost'),
            'port' => env('DB_PORT', 3306),
            'database' => env('DB_DATABASE', 'ody'),
            'username' => env('DB_USERNAME', 'ody'),
            'password' => env('DB_PASSWORD', ''),
            'charset' => env('DB_CHARSET', 'utf8mb4'),
            // Other database settings...
        ],
        // Other environments...
    ],
];
```

### DBAL-specific Configuration

In your `config/dbal.php`:

```php
return [
    'default' => env('DBAL_CONNECTION', 'default'),
    
    'connections' => [
        'default' => [
            'driver' => env('DB_DRIVER', 'mysql'),
            'host' => env('DB_HOST', 'localhost'),
            'port' => env('DB_PORT', '3306'),
            'database' => env('DB_DATABASE', 'ody'),
            'username' => env('DB_USERNAME', 'ody'),
            'password' => env('DB_PASSWORD', ''),
            'charset' => env('DB_CHARSET', 'utf8mb4'),
            'options' => [
                PDO::ATTR_EMULATE_PREPARES => false,
                PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
                PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
            ],
            'pooling' => [
                'pool_name' => 'dbal-default',
            ],
        ],
        // Other connections...
    ],
];
```

## Service Provider Registration

Register the DBAL service provider in your application:

```php
// In your service provider configuration
$providers = [
    // Other providers...
    \Ody\DB\Providers\DatabaseServiceProvider::class,
    \Ody\DB\Providers\DBALServiceProvider::class,
];
```

## Basic Usage

### Using the DBAL Facade

The simplest way to use DBAL is through the provided facade:

```php
use Ody\DB\Doctrine\Facades\DBAL;

// Execute a simple query
$users = DBAL::fetchAllAssociative(
    'SELECT * FROM users WHERE status = ?',
    ['active']
);

// Get a single value
$count = DBAL::fetchOne('SELECT COUNT(*) FROM users');

// Execute an update/insert
$affected = DBAL::executeStatement(
    'UPDATE users SET last_login = ? WHERE id = ?',
    [new \DateTime(), 123]
);
```

### Query Builder

DBAL provides a powerful query builder:

```php
$qb = DBAL::createQueryBuilder();

$users = $qb->select('u.id', 'u.name', 'u.email')
    ->from('users', 'u')
    ->where('u.status = :status')
    ->andWhere('u.created_at > :date')
    ->setParameter('status', 'active')
    ->setParameter('date', new \DateTime('-30 days'))
    ->orderBy('u.name', 'ASC')
    ->setMaxResults(10)
    ->executeQuery()
    ->fetchAllAssociative();
```

### Transactions

Transactions are fully supported:

```php
// Method 1: Using the transaction() helper
DBAL::transaction(function ($connection) {
    $connection->executeStatement(
        'INSERT INTO orders (customer_id, total) VALUES (?, ?)',
        [42, 99.99]
    );
    
    $orderId = $connection->lastInsertId();
    
    $connection->executeStatement(
        'INSERT INTO order_items (order_id, product_id, quantity) VALUES (?, ?, ?)',
        [$orderId, 101, 1]
    );
});

// Method 2: Manual transaction control
DBAL::beginTransaction();
try {
    DBAL::executeStatement('INSERT INTO logs (message) VALUES (?)', ['Starting process']);
    // More operations...
    DBAL::commit();
} catch (\Exception $e) {
    DBAL::rollBack();
    throw $e;
}
```

### Schema Operations

Work with database schema:

```php
// Get the schema manager
$schemaManager = DBAL::getSchemaManager();

// List all tables
$tables = $schemaManager->listTableNames();

// Get details about a table
$table = $schemaManager->introspectTable('users');

// Create a new table
$table = new \Doctrine\DBAL\Schema\Table('new_table');
$table->addColumn('id', 'integer', ['autoincrement' => true]);
$table->addColumn('name', 'string', ['length' => 255]);
$table->setPrimaryKey(['id']);
$schemaManager->createTable($table);
```

## Multiple Connections

You can work with multiple database connections:

```php
// Get a specific connection
$analytics = DBAL::connection('analytics');

// Use it for queries
$stats = $analytics->fetchAllAssociative(
    'SELECT * FROM page_views WHERE date >= ?',
    [new \DateTime('-7 days')]
);
```

## Advanced Usage

### Custom Queries and Types

```php
// Using custom types
use Doctrine\DBAL\Types\Type;

// Register a custom type
Type::addType('point', MyPointType::class);

// Use the custom type in a query
$stmt = DBAL::executeQuery(
    'SELECT * FROM locations WHERE point = ?',
    [$point],
    ['point']
);
```

### Raw Connection Access

If needed, you can access the underlying PDO connection:

```php
$pdo = DBAL::getNativeConnection();
// Use PDO directly (use with caution)
```

## Performance Optimization

### Query Caching

```php
// Create a query cache
$cache = new \Doctrine\DBAL\Cache\QueryCacheProfile(3600, 'query_cache_key');

// Execute with caching
$result = DBAL::executeQuery(
    'SELECT * FROM large_table WHERE complex_condition = ?',
    [42],
    [],
    $cache
)->fetchAllAssociative();
```

### Prepared Statements

DBAL automatically uses prepared statements, which helps prevent SQL injection and improves performance for repeated
queries.

### Batch Processing

For large operations, use batch processing:

```php
DBAL::beginTransaction();
try {
    $stmt = DBAL::prepare('INSERT INTO items (name, value) VALUES (?, ?)');
    
    foreach ($items as $i => $item) {
        $stmt->bindValue(1, $item['name']);
        $stmt->bindValue(2, $item['value']);
        $stmt->executeStatement();
        
        // Commit in batches of 1000 to prevent memory issues
        if ($i % 1000 === 0) {
            DBAL::commit();
            DBAL::beginTransaction();
        }
    }
    
    DBAL::commit();
} catch (\Exception $e) {
    DBAL::rollBack();
    throw $e;
}
```

## How It Works

The ODY DBAL implementation uses a custom driver (`DBALMysQLDriver`) that integrates with the connection pool:

1. When a DBAL connection is requested, the driver gets a connection from the pool
2. For optimal performance, connections are managed per Swoole worker
3. All interactions are coroutine-aware, ensuring correct behavior in concurrent environments
4. The driver automatically handles connection lifecycle (borrowing/returning) based on coroutine lifecycle

## Troubleshooting

### Connection Issues

If you encounter connection problems:

```php
// Check pool statistics
$pool = \Ody\DB\ConnectionManager::getPool('dbal-default');
var_dump($pool->stats());
```

### Query Performance

For slow queries:

1. Enable query logging in your database server
2. Use `EXPLAIN` to analyze query execution plans
3. Consider adjusting indexes based on query patterns

### Memory Leaks

If you suspect connection leaks:

1. Ensure all database operations happen within the same coroutine
2. Check that transactions are properly committed or rolled back
3. Monitor the number of active connections with `SHOW PROCESSLIST` in MySQL

## Comparison with Other ORMs

While Eloquent provides a more developer-friendly API with its ActiveRecord pattern, DBAL offers:

- **Better Raw Performance**: Up to 2x faster than Eloquent in benchmarks
- **Lower Memory Usage**: Requires less memory for the same operations
- **More Control**: Direct access to database features and optimizations
- **Schema Flexibility**: Powerful schema manipulation capabilities

## License

This package is open-sourced software licensed under the [MIT license](https://opensource.org/licenses/MIT).s
open-sourced software licensed under the [MIT license](https://opensource.org/licenses/MIT).
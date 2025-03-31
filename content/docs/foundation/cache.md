---
title: Cache
---

## Overview

- Multiple storage driver support:
    - Redis (using native phpredis with Swoole runtime hooks)
    - Memcached (using native Memcached extension with Swoole runtime hooks)
    - Array (in-memory, for testing)
- PSR-6 and PSR-16 compliant interfaces
- Cache tagging for grouped invalidation
- Configurable default TTL (Time To Live)

## Configuration

The cache configuration is typically located in `config/cache.php`. You can configure:

- Default driver
- Driver-specific settings (hosts, ports, etc.)
- Default TTL values
- Key prefixes

Example configuration:

```php
return [
    'default' => 'redis', // Default driver
    'drivers' => [
        'redis' => [
            'host' => '127.0.0.1',
            'port' => 6379,
            'auth' => null, // Optional password
            'db' => 0,      // Database index
            'prefix' => 'app:cache:',
            'ttl' => 3600   // Default TTL in seconds
        ],
        'memcached' => [
            'servers' => [
                ['host' => '127.0.0.1', 'port' => 11211, 'weight' => 100],
            ],
            'prefix' => 'app:cache:',
            'ttl' => 3600
        ],
        'array' => [
            'ttl' => 3600
        ]
    ]
];
```

## Basic Usage

The cache module provides a simple facade interface that makes it easy to interact with the cache:

```php
use Ody\Foundation\Facades\Cache;

// Store a value (with default TTL)
Cache::set('user:1', ['name' => 'John Doe']);

// Store with specific TTL (5 minutes)
Cache::set('session:abc', $sessionData, 300);

// Check if a key exists
if (Cache::has('user:1')) {
    // Do something
}

// Get a value (with optional default if not found)
$user = Cache::get('user:1', ['name' => 'Guest']);

// Delete a key
Cache::delete('user:1');

// Clear the entire cache
Cache::clear();
```

## Working with Multiple Items

You can efficiently work with multiple cache items:

```php
// Get multiple items
$values = Cache::getMultiple(['user:1', 'user:2', 'user:3']);

// Set multiple items
Cache::setMultiple([
    'user:1' => ['name' => 'John'],
    'user:2' => ['name' => 'Jane'],
    'user:3' => ['name' => 'Bob']
], 3600); // 1 hour TTL

// Delete multiple items
Cache::deleteMultiple(['user:1', 'user:2', 'user:3']);
```

## Using Different Cache Drivers

You can explicitly specify which cache driver to use:

```php
// Use Redis driver
$redisCache = Cache::driver('redis');
$redisCache->set('key', 'value');

// Use Memcached driver
$memcachedCache = Cache::driver('memcached');
$memcachedCache->set('key', 'value');

// Use Array driver (for testing)
$arrayCache = Cache::driver('array');
$arrayCache->set('key', 'value');
```

## Tagged Cache

Tagged cache allows you to group related items and invalidate them together:

```php
// Create a tagged cache
$userCache = Cache::tags('users');

// Store items with the tag
$userCache->set('user:1', ['name' => 'John']);
$userCache->set('user:2', ['name' => 'Jane']);

// You can also use multiple tags
$adminCache = Cache::tags(['users', 'admins']);
$adminCache->set('admin:1', ['name' => 'Admin']);

// Clear all items with the 'users' tag
Cache::tags('users')->clear();
```

## Advanced PSR-6 Usage

For more complex caching scenarios, you can use the PSR-6 compliant interface:

```php
// Get a PSR-6 cache pool
$pool = Cache::pool();

// Get an item (doesn't check the cache yet)
$item = $pool->getItem('user:1');

if (!$item->isHit()) {
    // Cache miss, set a new value
    $userData = fetchUserFromDatabase(1);
    $item->set($userData);
    
    // Set TTL (5 minutes)
    $item->expiresAfter(300);
    
    // Save to cache
    $pool->save($item);
}

// Get the value
$user = $item->get();

// Deferred saves for better performance
$item = $pool->getItem('user:2');
$item->set($user2Data);
$pool->saveDeferred($item);

// Commit all deferred saves at once
$pool->commit();
```

## Performance Considerations

- Use `setMultiple` and `getMultiple` when working with multiple items for better performance
- Consider using tags for efficient group invalidation instead of individual deletions
- For high-frequency access patterns, choose the appropriate driver:
    - Redis: Good for complex data types and when you need atomic operations
    - Memcached: Generally faster for simple key/value storage
    - Array: For testing or very short-lived cache needs

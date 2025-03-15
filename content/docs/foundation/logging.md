---
title: Logging
---

## Configuration

Logging is configured in `config/logging.php` and provides various channels for logging:

## Usage

```php
// Using the logger
$logger->info('User logged in', ['id' => $userId]);
$logger->error('Failed to process payment', ['order_id' => $orderId]);

// Using the logger helper function
logger()->info('Processing request');
```

## Creating Custom Loggers

### Basic Requirements

All custom loggers must:

1. Extend `Ody\Foundation\Logging\AbstractLogger`
2. Implement a static `create(array $config)` method
3. Override the `write(string $level, string $message, array $context = [])` method

### Example: Creating a Custom Logger

Here's a simple example of a custom logger that logs to Redis:

```php
<?php

namespace App\Logging;

use Ody\Foundation\Logging\AbstractLogger;
use Ody\Foundation\Logging\FormatterInterface;
use Ody\Foundation\Logging\JsonFormatter;
use Ody\Foundation\Logging\LineFormatter;
use Psr\Log\LoggerInterface;
use Psr\Log\LogLevel;
use Redis;

class RedisLogger extends AbstractLogger
{
    /**
     * @var Redis
     */
    protected Redis $redis;
    
    /**
     * @var string
     */
    protected string $channel;
    
    /**
     * Constructor
     */
    public function __construct(
        Redis $redis,
        string $channel = 'logs',
        string $level = LogLevel::DEBUG,
        ?FormatterInterface $formatter = null
    ) {
        parent::__construct($level, $formatter);
        $this->redis = $redis;
        $this->channel = $channel;
    }
    
    /**
     * Create a Redis logger from configuration
     */
    public static function create(array $config): LoggerInterface
    {
        // Create Redis connection
        $redis = new Redis();
        $redis->connect(
            $config['host'] ?? '127.0.0.1',
            $config['port'] ?? 6379
        );
        
        if (isset($config['password'])) {
            $redis->auth($config['password']);
        }
        
        // Create formatter
        $formatter = null;
        if (isset($config['formatter'])) {
            $formatter = self::createFormatter($config);
        }
        
        // Return new logger instance
        return new self(
            $redis,
            $config['channel'] ?? 'logs',
            $config['level'] ?? LogLevel::DEBUG,
            $formatter
        );
    }
    
    /**
     * Create a formatter based on configuration
     */
    protected static function createFormatter(array $config): FormatterInterface
    {
        $formatterType = $config['formatter'] ?? 'json';
        
        if ($formatterType === 'line') {
            return new LineFormatter(
                $config['format'] ?? null,
                $config['date_format'] ?? null
            );
        }
        
        return new JsonFormatter();
    }
    
    /**
     * {@inheritdoc}
     */
    protected function write(string $level, string $message, array $context = []): void
    {
        // Format log data
        $logData = [
            'timestamp' => time(),
            'level' => $level,
            'message' => $message,
            'context' => $context
        ];
        
        // Publish to Redis channel
        $this->redis->publish(
            $this->channel,
            json_encode($logData)
        );
    }
}
```

### The `create()` Method

The static `create()` method is responsible for instantiating your logger based on configuration:

```php
public static function create(array $config): LoggerInterface
{
    // Create dependencies based on configuration
    // ...
    
    // Return new logger instance
    return new self(...);
}
```

This method receives the channel configuration from the `logging.php` config file and should:

1. Create any dependencies the logger needs
2. Configure those dependencies based on the config array
3. Return a new instance of the logger

### The `write()` Method

The `write()` method is where the actual logging happens:

```php
protected function write(string $level, string $message, array $context = []): void
{
    // Implement logging logic here
}
```

This method is called by the parent `AbstractLogger` class when a log message needs to be written. It receives:

- `$level`: The log level (debug, info, warning, etc.)
- `$message`: The formatted log message
- `$context`: Additional context data

## Using Custom Loggers

### Method 1: Configuration-Based Discovery

The simplest way to use a custom logger is to specify the fully-qualified class name in your logging configuration:

```php
// In config/logging.php
'channels' => [
    'redis' => [
        'driver' => 'redis',
        'class' => \App\Logging\RedisLogger::class,
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'port' => env('REDIS_PORT', 6379),
        'channel' => 'application_logs',
        'level' => 'debug',
    ],
]
```

When you specify a `class` parameter, that class will be used regardless of the driver name.

### Method 2: Driver Name Registration

You can register your logger with a driver name, which allows you to reference it using just the driver name:

```php
// In a service provider's register method
$this->app->make(\Ody\Foundation\Logging\LogManager::class)
    ->registerDriver('redis', \App\Logging\RedisLogger::class);
```

Then in your configuration:

```php
// In config/logging.php
'channels' => [
    'redis' => [
        'driver' => 'redis', // This will use the registered RedisLogger
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'port' => env('REDIS_PORT', 6379),
        'channel' => 'application_logs',
        'level' => 'debug',
    ],
]
```

### Method 3: Automatic Discovery

If your logger follows the naming convention `{Driver}Logger` and is in one of the registered namespaces, it will be discovered automatically:

```php
// In config/logging.php
'channels' => [
    'redis' => [
        'driver' => 'redis', // Will look for RedisLogger
        // Configuration...
    ],
]
```

The framework will search for `RedisLogger` in the registered namespaces (`\Ody\Foundation\Logging\` and `\App\Logging\` by default).

### Creating Custom Formatters

If the standard formatters don't meet your needs, you can create your own by implementing the `FormatterInterface`:

```php
namespace App\Logging;

use Ody\Foundation\Logging\FormatterInterface;

class CustomFormatter implements FormatterInterface
{
    public function format(string $level, string $message, array $context = []): string
    {
        // Custom formatting logic
        return "[$level] $message " . json_encode($context);
    }
}
```

## Complete Example: Using Redis Logger

### Configuration

```php
// In config/logging.php
'channels' => [
    // Using explicit class
    'redis' => [
        'driver' => 'redis',
        'class' => \App\Logging\RedisLogger::class,
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'port' => env('REDIS_PORT', 6379),
        'password' => env('REDIS_PASSWORD', null),
        'channel' => 'app_logs',
        'formatter' => 'json',
        'level' => 'debug',
    ],
    
    // Using it in a stack
    'production' => [
        'driver' => 'group',
        'channels' => ['file', 'redis'],
    ],
],
```

### Usage

```php
// Send to redis channel
logger('User registered', ['id' => 123], 'redis');

// Or use the stack
logger('API request processed', ['endpoint' => '/users']);
```
---

With this system, you can create custom loggers that integrate seamlessly with the ODY Framework logging infrastructure.
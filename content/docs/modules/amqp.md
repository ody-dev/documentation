---
title: AMQP
---

This package provides RabbitMQ integration for the ODY framework, allowing you to easily implement asynchronous
messaging patterns in your application.

## Installation

```bash
composer require ody/amqp
```

## Configuration

First, publish the configuration file:

```bash
php ody publish:config ody/amqp
```

This will create a `config/amqp.php` file where you can configure your RabbitMQ connections:

```php
return [
    'enable' => true,
    
    'default' => [
        'host' => env('RABBITMQ_HOST', 'localhost'),
        'port' => env('RABBITMQ_PORT', 5672),
        'user' => env('RABBITMQ_USER', 'guest'),
        'password' => env('RABBITMQ_PASSWORD', 'guest'),
        'vhost' => env('RABBITMQ_VHOST', '/'),
        
        'concurrent' => [
            'limit' => 10,  // Max concurrent consumers per process
        ],
        
        'params' => [
            'connection_timeout' => 3.0,
            'read_write_timeout' => 3.0,
            'heartbeat' => 60,
            'keepalive' => true,
        ],
    ],
    
    // You can define multiple connection pools
    'analytics' => [
        'host' => 'analytics-rabbitmq',
        // Other connection settings
    ],
    
    // Connection pooling configuration
    'pool' => [
        'enable' => true,
        'max_connections' => 20,
        'max_channels_per_connection' => 20,
        'max_idle_time' => 60,  // seconds
    ],
    
    'producer' => [
        'paths' => ['app/Producers'],
        'retry' => [
            'max_attempts' => 3,
            'initial_interval' => 1000,  // ms
            'multiplier' => 2.0,
            'max_interval' => 10000,  // ms
        ],
    ],
    
    'consumer' => [
        'paths' => ['app/Consumers'],
        'prefetch_count' => 10,
        'auto_declare' => true,
    ],
    
    'process' => [
        'enable' => true,
        'max_consumers' => 10,
        'auto_restart' => true,
    ]
];
```

## Usage
### Creating Producers

Producers send messages to RabbitMQ. Create a producer class in your `app/Producers` directory:

```php
<?php

namespace App\Producers;

use Ody\AMQP\Attributes\Producer;
use Ody\AMQP\Message\ProducerMessage;

#[Producer(exchange: 'user_events', routingKey: 'user.created', type: 'topic')]
class UserCreatedProducer extends ProducerMessage
{
    public function __construct(
        private int $userId,
        private string $email,
        private string $username
    ) {
        $this->payload = [
            'user_id' => $this->userId,
            'email' => $this->email,
            'username' => $this->username,
            'created_at' => date('Y-m-d H:i:s')
        ];
    }
}
```

### Creating Consumers

Consumers receive and process messages from RabbitMQ. Create a consumer class in your `app/Consumers` directory:

```php
<?php

namespace App\Consumers;

use Ody\AMQP\Attributes\Consumer;
use Ody\AMQP\Message\ConsumerMessage;
use Ody\AMQP\Message\Result;
use PhpAmqpLib\Message\AMQPMessage;

#[Consumer(
    exchange: 'user_events',
    routingKey: 'user.created',
    queue: 'welcome_email_queue',
    type: 'topic'
)]
class WelcomeEmailConsumer extends ConsumerMessage
{
    public function consumeMessage(array $data, AMQPMessage $message): Result
    {
        try {
            // Process the message
            $userId = $data['user_id'];
            $email = $data['email'];
            $username = $data['username'];
            
            // Send welcome email
            // $this->emailService->sendWelcomeEmail($email, $username);
            
            // Log success
            error_log("Welcome email sent to $email for user $userId");
            
            // Acknowledge the message
            return Result::ACK;
        } catch (\Exception $e) {
            // Log the error
            error_log("Failed to send welcome email: " . $e->getMessage());
            
            // Reject and requeue the message for retry
            return Result::REQUEUE;
        }
    }
}
```

### Publishing Messages

To publish messages in your application code:

```php
use Ody\AMQP\AMQPClient;
use App\Producers\UserCreatedProducer;

public function __construct(
    private readonly AMQPClient          $amqpClient,
) {}

// In your controller or service
public function registerUser(array $userData)
{
    // Create user in database
    $user = $this->userRepository->create($userData);
    
    // Publish event
    $this->amqpClient->publish(UserCreatedProducer::class, [
        $user->id,              // userId
        $user->email,           // email
        $user->username         // username
    ]);
    
    return $user;
}
```

### Delayed Messages

You can publish messages with a delay:

```php
// Send a reminder after 24 hours
$this->amqpClient->publishDelayed(ReminderProducer::class, [
    $user->id,
    'Your trial is about to expire'
], 86400000); // 24 hours in milliseconds
```

### Working with Topic Exchanges

Topic exchanges provide flexible routing:

```php
// Producer
#[Producer(exchange: 'notifications', routingKey: 'user.123.email', type: 'topic')]
class UserNotificationProducer extends ProducerMessage { /* ... */ }

// Consumer with wildcards
#[Consumer(
    exchange: 'notifications',
    routingKey: 'user.*.email',  // Match any user ID
    queue: 'email_notifications',
    type: 'topic'
)]
class EmailNotificationConsumer extends ConsumerMessage { /* ... */ }
```

### Message Results

Consumers should return one of these results:

- `Result::ACK`: Acknowledge the message (successfully processed)
- `Result::NACK`: Negative acknowledgment (failed to process, don't requeue)
- `Result::REQUEUE`: Reject and requeue the message (retry later)
- `Result::DROP`: Reject the message and drop it (don't retry)

```php
public function consumeMessage(array $data, AMQPMessage $message): Result
{
    try {
        // Process message
        return Result::ACK;
    } catch (\Exception $e) {
        // Decide based on error type
        if ($e instanceof TemporaryFailureException) {
            return Result::REQUEUE;  // Retry later
        }
        
        // Permanent failure
        return Result::DROP;  // Don't retry
    }
}
```

## Connection Pooling

The framework implements connection pooling to optimize RabbitMQ connections by reusing existing connections and
channels instead of creating new ones for each operation. This significantly improves performance in high-throughput
scenarios.

### Pool Configuration

Configure the connection pool in your `config/amqp.php` file:

```php
'pool' => [
    'enable' => true,                   // Enable/disable connection pooling
    'max_connections' => 20,            // Maximum connections in the pool
    'max_channels_per_connection' => 20,// Maximum channels per connection
    'max_idle_time' => 60,              // Maximum idle time in seconds
],
```

### Usage

Connection pooling works transparently with the existing AMQP API:

```php
// The publish method automatically uses pooled connections
$this->amqpClient->publish(UserCreatedProducer::class, [
    $user->id,
    $user->email,
    $user->username
]);

// For advanced usage, you can directly access pooled resources
$connection = $this->amqpClient->getPooledConnection();
$channel = $this->amqpClient->getPooledChannel();
```

## Monitoring and Management

RabbitMQ provides a management UI available at `http://localhost:15672/` (default username/password is the one you
configured).

You can use this interface to:

- Monitor queues and exchanges
- View message rates and consumer activity
- Manage exchanges, queues, and bindings
- View connections and channels

### Connection Pool Monitoring

To monitor the connection pool, you can use:

```php
// Get connection pool statistics
$connectionStats = \Ody\AMQP\AMQPConnectionPool::getStats();

// Get channel pool statistics
$channelStats = \Ody\AMQP\AMQPChannelPool::getStats();
```

This returns information about active connections, their state, and channel distribution.

---

For more advanced usage and configuration options, refer to the full documentation or explore
the [RabbitMQ official documentation](https://www.rabbitmq.com/documentation.html).
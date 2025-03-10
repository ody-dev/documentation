---
title: Tasks
weight: 5
---

## Introduction

The Task Manager is a component that provides a clean, intuitive interface for handling asynchronous tasks using Swoole's 
task worker capabilities. It allows you to offload time-consuming operations from your main application flow.

## Why Use Task Manager?

Asynchronous task processing is essential for maintaining responsive applications, especially when dealing with 
operations that:

- Take significant time to complete
- Access external resources (APIs, file systems, etc.)
- Perform CPU-intensive calculations
- Send emails or notifications
- Process uploads or media files
- Generate reports or exports

By offloading these operations to task workers, your main application threads remain free to handle new requests, 
significantly improving your application's scalability and user experience.

## Installation

```shell
composer require ody/task
```

## Basic Usage
### Creating a Task

All tasks must implement the `TaskInterface` and provide a `handle()` method:

```php
<?php

namespace App\Tasks;

use Ody\Task\TaskInterface;

class SendWelcomeEmailTask implements TaskInterface
{
    public function handle(array $params = [])
    {
        $email = $params['email'] ?? null;
        $name = $params['name'] ?? 'User';
        
        if (!$email) {
            return ['status' => 'error', 'message' => 'Email is required'];
        }
        
        // Logic to send email here
        
        return [
            'status' => 'success',
            'message' => "Welcome email sent to {$email}"
        ];
    }
}
```

### Executing Tasks

To execute a task immediately:

```php
use Ody\Task\Task;

// Simple execution
$taskId = Task::execute(\App\Tasks\SendWelcomeEmailTask::class, [
    'email' => 'user@example.com',
    'name' => 'John Doe'
]);
```

### Setting Priority

You can set task priority to ensure important tasks are processed first:

```php
// High priority task
$taskId = Task::execute(\App\Tasks\ProcessPaymentTask::class, [
    'amount' => 99.99,
    'userId' => 123
], Task::PRIORITY_HIGH);

// Low priority task
$taskId = Task::execute(\App\Tasks\GenerateReportTask::class, [
    'reportType' => 'monthly',
    'format' => 'pdf'
], Task::PRIORITY_LOW);
```

### Delayed Execution

To schedule a task to run in the future:

```php
// Run a task after 5 seconds
$taskId = Task::later(\App\Tasks\SendReminderTask::class, [
    'userId' => 456,
    'message' => 'Don\'t forget to complete your profile!'
], 5000); // 5000ms = 5 seconds
```

### Task Retry Mechanisms
This feature adds automatic retry capability for tasks that fail, with exponential backoff. The system will automatically 
handle retry attempts when a task throws an exception, with increasing delays between attempts.
```php
// Execute a task with retry (will retry up to 3 times with exponential backoff)
$taskId = Task::withRetry(
    \App\Tasks\PaymentProcessorTask::class,
    ['order_id' => 12345],
    [
        'attempts' => 3,        // Maximum attempts
        'delay' => 1000,        // Initial delay (1 second)
        'multiplier' => 2       // Each retry doubles the delay
    ]
);
```

### Task Groups and Batches
#### Task Groups
Groups let you organize related tasks and wait for their collective completion:

```php
// Create a task group
$group = Task::group('email-notifications')
    ->add(\App\Tasks\SendWelcomeEmailTask::class, ['user_id' => 123])
    ->add(\App\Tasks\NotifyAdminTask::class, ['new_user_id' => 123])
    ->add(\App\Tasks\UpdateStatisticsTask::class, ['event' => 'new_signup'])
    ->concurrency(2)           // Only run 2 tasks at a time
    ->allowFailures(true);     // Continue even if some tasks fail

// Dispatch all tasks in the group
$taskIds = $group->dispatch();

// Wait for all tasks to complete and get their results
$results = $group->wait(10000);  // Wait up to 10 seconds

// Or cancel all pending tasks in the group
$group->cancel();
```

#### Task Batches
Batches are optimized for processing multiple similar tasks:

```php
// Process many items with the same task class
$userIds = [1, 2, 3, 4, 5];
$tasks = [];

foreach ($userIds as $userId) {
    $tasks[] = [
        'class' => \App\Tasks\ProcessUserDataTask::class,
        'params' => ['user_id' => $userId]
    ];
}

// Create and dispatch the batch
$batch = Task::batch($tasks);
$taskIds = $batch->dispatch();

// Wait for all tasks to complete or timeout after 30 seconds
$results = $batch->wait(30000);
```

### Task Monitoring and Reporting

```php
// Get status of a specific task
$taskStatus = Task::status($taskId);
// Returns: id, status, attempts, created_at, started_at, completed_at, execution_time, result, error

// Get overall metrics
$metrics = TaskManager::getInstance()->getMetrics();
// Returns: total_tasks, completed_tasks, failed_tasks, retried_tasks, cancelled_tasks, average_execution_time
```

### Task Cancellation

```php
// Cancel a specific task
$cancelled = Task::cancel($taskId);

// Cancel all tasks in a group
$group->cancel();

// Cancel all tasks in a batch
$batch->cancel();
```
Note: Only pending tasks can be cancelled. Running tasks will continue to completion.

### Task Middleware
Middleware allows you to modify tasks or add cross-cutting concerns:
```php
// Register global middleware that applies to all tasks
TaskManager::getInstance()->registerMiddleware(new LoggingMiddleware());

// Apply middleware to a specific
WIP
```

## Framework Integration

### Server Configuration

When setting up your Swoole server, you need to enable task workers and initialize the task handler:

```php
$server = new Swoole\Server('0.0.0.0', 9501);

// Configure Swoole server
$server->set([
    'worker_num' => 4,            // Number of worker processes
    'task_worker_num' => 8,       // Number of task worker processes
    'task_enable_coroutine' => true,  // Enable coroutines in task workers
]);

// Initialize the task handler
\Ody\Task\TaskHandler::init($server);

// Start the server
$server->start();
```

## Advanced Usage
### Task Chaining

You can chain tasks by triggering a new task from within a task's `handle()` method:

```php
public function handle(array $params = [])
{
    // Do something...
    
    // Then trigger another task
    Task::execute(\App\Tasks\AnotherTask::class, [
        'previousResult' => $result
    ]);
    
    return ['status' => 'success'];
}
```

### Task Batching

For operations that require processing multiple items, consider implementing a batch task pattern:

```php
// Create a batch of tasks
foreach ($userIds as $userId) {
    Task::execute(\App\Tasks\NotifyUserTask::class, [
        'userId' => $userId,
        'message' => $notificationMessage
    ]);
}
```

## API Reference

### Task Class

#### Static Methods

| Method | Description |
|--------|-------------|
| `execute(string $taskClass, array $params = [], int $priority = self::PRIORITY_NORMAL): int` | Execute a task immediately with the given parameters and priority. Returns the task ID. |
| `later(string $taskClass, array $params = [], int $delayMs = 1000, int $priority = self::PRIORITY_NORMAL): int` | Schedule a task to execute after the specified delay in milliseconds. Returns the task ID. |

#### Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `PRIORITY_HIGH` | 20 | High priority tasks are processed before normal and low priority tasks |
| `PRIORITY_NORMAL` | 10 | Default priority level |
| `PRIORITY_LOW` | 5 | Low priority tasks are processed after high and normal priority tasks |

### TaskInterface

#### Methods

| Method | Description |
|--------|-------------|
| `handle(array $params = [])` | The main method that will be called when the task is executed. Should return a result that will be passed to the `finish` event. |

### TaskManager Class

#### Methods

| Method | Description |
|--------|-------------|
| `getInstance(): TaskManager` | Get the singleton instance of the TaskManager |
| `setServer(Server $server): void` | Set the Swoole server instance |
| `enqueue(string $taskClass, array $params = [], int $priority = Task::PRIORITY_NORMAL): int` | Add a task to the queue for immediate execution |
| `enqueueDelayed(string $taskClass, array $params = [], int $delayMs = 1000, int $priority = Task::PRIORITY_NORMAL): int` | Add a task to be executed after the specified delay |
| `getNextTask(): ?array` | Get the next task from the highest priority queue |
| `taskComplete(int $taskId, $result): void` | Process a completed task |

### TaskHandler Class

#### Methods

| Method | Description |
|--------|-------------|
| `init(Server $server): void` | Initialize the task handler with the Swoole server |
| `handleTask(array $data)` | Handle a task execution |

## Best Practices

1. **Keep Tasks Small and Focused**: Each task should do one thing well
2. **Make Tasks Idempotent**: When possible, design tasks to be safely retried if they fail
3. **Include Error Handling**: Always handle exceptions within your tasks to prevent task worker crashes
4. **Consider Timeouts**: For tasks that might take a long time, implement timeout mechanisms
5. **Log Task Execution**: Add logging to track task execution and troubleshoot issues

## Troubleshooting
### Tasks Not Executing

- Ensure you have configured enough task workers (`task_worker_num`)
- Check that you've properly initialized the TaskHandler
- Verify that your task class implements TaskInterface correctly

### Memory Leaks

- Avoid storing large amounts of data in class properties
- Use dependency injection rather than globals for accessing services
- Ensure resources (file handles, database connections) are properly closed

### Task Workers Crashing

- Add try-catch blocks around your task code
- Implement proper error logging
- Avoid using functions that are not coroutine-safe if using `task_enable_coroutine`
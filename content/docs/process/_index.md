---
title: Process
weight: 8
---
WIP, not ready yet!

## Introduction

ODY includes a process management system, enabling seamless execution of background services in parallel with your 
main application. By leveraging Swoole processes, you can efficiently manage long-running tasks, worker daemons, 
and system-level operations without blocking your core application. This system allows you to:

* Execute PHP code in separate processes
* Monitor running processes
* Communicate between processes
* Automatically handle process lifecycle

## Installation

```shell
composer require ody/process
```

## Basic Usage

```php
Process::execute(\App\Workers\BackgroundJob::class);
Process::kill($pid)
Process::isRunning($processClass);
```

### Creating a Process

To create a background process, you need to implement the `ProcessInterface`:

```php
<?php

namespace App\Processes;

use Swoole\Process;
use YourFramework\Process\ProcessInterface;

class EmailSender implements ProcessInterface
{
    private array $args;
    private Process $worker;
    
    public function __construct(array $args, Process $worker)
    {
        $this->args = $args;
        $this->worker = $worker;
    }
    
    public function handle(): void
    {
        // Process logic goes here
        $emails = $this->args['emails'] ?? [];
        
        foreach ($emails as $email) {
            // Send email
            $this->worker->write("Sent email to {$email['recipient']}");
            sleep(1); // Simulate work
        }
    }
}
```

### Running a Process

To execute your process, use the `Process` facade:

```php
<?php

use YourFramework\Process\Process;

// Execute the process with arguments
$pid = Process::execute(\App\Processes\EmailSender::class, [
    'emails' => [
        ['recipient' => 'user1@example.com', 'subject' => 'Hello'],
        ['recipient' => 'user2@example.com', 'subject' => 'Welcome'],
    ]
]);

echo "Process started with PID: {$pid}\n";
```

### Checking Process Status

You can check if a process is running:

```php
<?php

use YourFramework\Process\Process;

if (Process::isRunning(\App\Processes\EmailSender::class)) {
    echo "Email sender is currently running\n";
} else {
    echo "Email sender is not running\n";
}
```

### Stopping a Process

You can terminate a running process:

```php
<?php

use YourFramework\Process\Process;

$pid = Process::execute(\App\Processes\EmailSender::class, $args);

// Later in your code
Process::kill($pid);
```

## API Reference
### Process Class
#### Static Methods

| Method | Parameters | Return | Description |
|--------|------------|--------|-------------|
| `execute()` | `string $processClass, array $args = [], bool $daemon = false` | `int` | Executes a process class and returns the process ID |
| `kill()` | `int $pid, int $signal = SIGTERM` | `bool` | Terminates a running process |
| `isRunning()` | `string $processClass` | `bool` | Checks if a specific process class is running |

### ProcessManager Class

#### Static Methods

| Method | Parameters | Return | Description |
|--------|------------|--------|-------------|
| `init()` | `array $config = []` | `void` | Initializes the process manager with configuration |
| `execute()` | `string $processClass, array $args = [], bool $daemon = false` | `int` | Executes a process class and returns the process ID |
| `kill()` | `int $pid, int $signal = SIGTERM` | `bool` | Terminates a running process |
| `isRunning()` | `string $processClass` | `bool` | Checks if a specific process class is running |
| `getRunningProcesses()` | none | `array` | Returns all currently running processes |
| `waitAll()` | none | `void` | Waits for all child processes to finish |

### ProcessInterface

#### Required Methods

| Method | Parameters | Return | Description |
|--------|------------|--------|-------------|
| `__construct()` | `array $args, \Swoole\Process $worker` | none | Constructor that receives arguments and worker process |
| `handle()` | none | `void` | Main logic of the process |



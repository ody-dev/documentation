---
title: Process
weight: 8
---
WIP, not ready yet!

ODY includes a robust process management system built on top of Swoole, enabling seamless execution of 
background services in parallel with your main application. By leveraging Swoole processes, you can efficiently manage 
long-running tasks, worker daemons, and system-level operations without blocking your core application.

## Usage

You can start a background process easily using the `Process::start()` method:

```php
Process::start(\App\Workers\BackgroundJob::class);
```

Each process runs independently, allowing for parallel execution while maintaining efficient resource management.

## Use Cases

* Long-running background services
* Real-time data processing
* Custom worker daemons
* CPU-intensive operations

By integrating processes, ODY ensures scalable and efficient execution of background services while keeping the main 
application responsive.
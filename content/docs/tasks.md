---
title: Tasks
weight: 5
---

WIP, not ready yet!

ODY provides a powerful asynchronous task system powered by Swoole, allowing you to offload time-consuming operations 
without blocking the main execution flow. Tasks run in dedicated worker processes, ensuring smooth performance for 
real-time applications.

You can dispatch tasks effortlessly using the `Task::execute()` method, passing in a reference to the class that should 
handle the task:

```php
Task::execute(\Class\Reference::class);
```
This will queue the task for execution in a background worker, freeing up the main process to handle other requests.

## Priority 

## Queing tasks
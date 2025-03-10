---
title: Tasks
weight: 5
---

WIP, not ready yet!

ODY provides an asynchronous task system, allowing you to offload time-consuming operations without blocking 
the main execution flow. Tasks run in dedicated worker processes

You can dispatch tasks using the `Task::execute()` method, passing in a reference to the class that should 
handle the task:

```php
Task::execute(\Class\Reference::class);
```
This will queue the task for execution in a background worker, freeing up the main process to handle other requests.

## Priority 

## Queing tasks
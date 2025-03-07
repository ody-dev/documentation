---
title: Scheduler
weight: 5
---

## Introduction
Efficient task scheduling is essential for automating recurring processes such as database maintenance, email 
notifications, and background jobs. The PHP Scheduler for [Your Framework] provides a flexible and intuitive way to 
define and manage scheduled tasks without relying on external cron jobs.

## Installation
```shell
composer require ody/scheduler
```

## Usage

### Create a job
```php
namespace App\Jobs;

use Ody\Scheduler\JobInterface;

class JobPerMin implements JobInterface
{

    public function jobName(): string
    {
        return 'JobPerMin';
    }

    public function crontabRule(): string
    {
        return '*/1 * * * *';
    }

    public function run()
    {
        var_dump(time(), 'JobPerMin');
        return time();
    }

    public function onException(\Throwable $throwable)
    {
        throw $throwable;
    }
}
```

### Register a job
Add jobs to `scheduler.php` config.
```php
return [
    "jobs" => [
        \App\Console\Jobs\JobPerMin::class,
    ]
];
```

### Start a scheduler instance
```php
php ody scheduler:start
```

### Triggering jobs
(WIP)

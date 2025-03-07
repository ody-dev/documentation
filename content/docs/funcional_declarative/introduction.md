---
    title: Getting started
weight: 1
---

ODY provides a set of methods that enable developers to write asynchronous, functional-style code
using coroutines in PHP, making complex operations more readable and efficient. This is ideal to use when trying out new
ideas or when you need a simple lightweight microservice.

This documentation will explore how to use these methods to simplify async programming and enhance developer productivity.

## Installation
```shell
composer require ody/swoole
```

## Usage

### Setting up a simple web server

```php
use Ody\Swoole\Declarative\Response
use Ody\Swoole\Declarative\Route
use Ody\Swoole\Declarative\Swoole

$handler = function () {
    Route\get('/', fn() => Response\json('Hello, World!'));
};

$port = 8000;
echo "Listening on port $port\n";
Swoole\http($handler, $port)->start();
```
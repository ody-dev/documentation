---
title: Getting started
weight: 1
---

ODY provides a set of methods that enable developers to write asynchronous, functional-style code
using coroutines in PHP. This is ideal to use when trying out new ideas or when you need a simple lightweight 
microservice/API.

## Installation
```shell
composer require ody/pico
```

[//]: # (## Usage)

[//]: # ()
[//]: # (### Setting up a simple web server)

[//]: # ()
[//]: # (```php)

[//]: # (use Ody\Swoole\Declarative\Response)

[//]: # (use Ody\Swoole\Declarative\Route)

[//]: # (use Ody\Swoole\Declarative\Swoole)

[//]: # ()
[//]: # ($handler = function &#40;&#41; {)

[//]: # (    Route\get&#40;'/', fn&#40;&#41; => Response\json&#40;'Hello, World!'&#41;&#41;;)

[//]: # (};)

[//]: # ()
[//]: # ($port = 8000;)

[//]: # (echo "Listening on port $port\n";)

[//]: # (Swoole\http&#40;$handler, $port&#41;->start&#40;&#41;;)

[//]: # (```)
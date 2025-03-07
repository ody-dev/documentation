---
title: Swoole
weight: 2
---

## Available methods
### `http()`
```php
$server = Swoole\http(
    callable $handler, 
    int $port = 9501, 
    string $host = '0.0.0.0'
);
$server->start;
```
### `http2()`
```php
$server = Swoole\http(
    string $certFile, // Path to the certificate file
    string $keyFile, // Path to the key file
    callable $handler, // The callable to call on each request
    int $port = 9501, 
    string $host = '0.0.0.0'
);
$server->start;
```
### `request()`
Gets the current Swoole HTTP request.
```php
Swoole\request()
```
### `response()`
Create new scratch file from selection
```php
Swoole\response
```

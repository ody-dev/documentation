---
title: Futures
weight: 5
---

## Introduction
Futures refer to a programming concept used for handling asynchronous operations. A future represents 
a value that may not yet be available but will be resolved at some point in the future. It allows developers to write 
non-blocking code, improving efficiency, especially in concurrent or parallel programming.

ODY made a custom implementation of futures using with coroutines. We used a well known `async/await` approach.

## Usage
Before you can use futures in ODY you need to have Swoole installed. Futures Currently reside in de ody/swoole package, 
these will be moved to their own repo soon.

```shell
# linux
sudo apt install php8.3-swoole

composer require ody/swoole
```

USing futures in PHP need to happen in a coroutine context, during the request lifecycle of ODY a coroutine context is 
already enabled. When using these methods outside of the request lifecycle you can create a coroutine as described in 
the Swoole docs.

```php
use function Swoole\Coroutine as Co;
Co\run(function() {...});
```
### Async / await
Creates and awaits for asynchronous computations in an alternative style than Swoole's coroutines.

```php
$future = Futures\async(fn() => 1);
$result = $future->await(); // 1
```
Futures are lazy, it only runs when you call `await`.

### Join
Joins a list of Futures into a single Future that awaits for a list of results.
```php
$slow_rand = function (): int {
    Co::sleep(3);
    return rand(1, 100);
};

$n1 = async($slow_rand);
$n2 = async($slow_rand);
$n3 = async($slow_rand);

$n = \Ody\Swoole\Futures\join([$n1, $n2, $n3]);

print_r($n->await());
```
This takes 3 seconds, not 9, Futures runs concurrently! (Order isn't guaranteed)

### Race
Returns the result of the first finished Future.

```php
$site1 = async(function () {
    $client = new Client('www.google.com', 443, true);
    $client->get('/');
    return $client->body;
});

$site2 = async(function () {
    $client = new Client('www.swoole.co.uk', 443, true);
    $client->get('/');
    return $client->body;
});

$site3 = async(function () {
    $client = new Client('ody.dev', 443, true);
    $client->get('/');
    return $client->body;
});

$first_to_load = select([$site1, $site2, $site3]);

echo $first_to_load->await();
```
### Async map
Maps an array into a list of Futures where which item runs concurrently.
```php
$list = [1, 2, 3];
$multiply = fn(int $a) => fn(int $b) => $a * $b;
$double = $multiply(2);

$doubles = \Ody\Swoole\Futures\join(async_map($list, $double))->await();

print_r($doubles);
```

### Then
Sequences a series of steps for a Future, is the serial analog for join:

```php
$future = async(fn() => 2)
    ->then(fn(int $i) => async(fn() => $i + 3))
    ->then(fn(int $i) => async(fn() => $i * 4))
    ->then(fn(int $i) => async(fn() => $i - 5));

echo $future->await(); // 15
```
### Stream
Streams values/events from `sink` to `listen` with between operations. 
```php
function main()
{
    $stream = Futures\stream()
        ->map(fn($val) => $val + 1)
        ->filter(fn($val) => $val % 2 === 0)
        ->map(fn($val) => $val * 2)
        ->listen(fn($val) => print("$val\n"));

    foreach (range(0, 9) as $n) {
        $stream->sink($n);
    }
}

Co\run('Acme\main');
```

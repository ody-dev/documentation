---
title: Installation
weight: 3
---

Ody provides an example skeleton project that gets you up and running quickly.
```
composer create-project ody/framework project-name

cd project-name
cp .env.example .env

php ody server:start
```

## Benchmarks
Real world benchmarks are in the works, the benchmark below was run on a workstation with an old first gen Ryzen 5 and 40GB ram.

```
$ wrk -t12 -c200 -d30s http://localhost:9501/users/1
Running 30s test @ http://localhost:9501/users/1
  12 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     8.32ms    6.68ms  72.77ms   82.09%
    Req/Sec     2.14k   169.90     4.01k    69.33%
  767631 requests in 30.05s, 326.44MB read
Requests/sec:  25545.05
Transfer/sec:     10.86MB
```
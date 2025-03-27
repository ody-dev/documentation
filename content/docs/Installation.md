---
title: Installation
weight: 3
---

Ody provides an example skeleton project that gets you up and running quickly. It comes pre-installed with 
Eloquent as ORM. It is possible to implement Doctrine ORM or plain DBAL, check the database section in the sidebar.
```
composer create-project ody/framework project-name

cd project-name
php ody publish // publishes required config files
cp .env.example .env

php ody server:start
```

## Benchmarks
Real world benchmarks are in the works, the benchmark below was run on a workstation with an old first gen Ryzen 5 and 40GB ram.
The request went through all preconfigured middleware and fetched all users (25) from the database using Eloquent, connection pooling 
and controller caching was enabled.

```
$ wrk -t12 -c400 -d30s http://localhost:9501/users

Running 30s test @ http://localhost:9501/users
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    65.92ms   39.53ms 349.67ms   65.96%
    Req/Sec   514.01     75.41   757.00     66.69%
  184369 requests in 30.10s, 90.88MB read
Requests/sec:   8125.34
Transfer/sec:   3.02MB
```
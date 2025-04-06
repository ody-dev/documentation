---
title: Introduction
weight: 1
---

ODY is a lightweight high-performance asynchronous PHP framework designed for building microservices and RESTful APIs with ease. Built on top of Swoole, ODY leverages asynchronous processing and coroutines to deliver exceptional performance while maintaining a clean, developer-friendly architecture.


## Key Features

- **High Performance**: Built with Swoole support for asynchronous processing and coroutines
- **PSR Compliance**: Implements PSR-7, PSR-15, and PSR-17 for HTTP messaging and middleware
- **Modular Design**: Build and integrate different modules
- **Middleware System**: Middleware system for request/response processing
- **Dependency Injection**: Built-in IoC container for dependency management
- **Console Support**: CLI commands for various tasks and application management
- **Routing**: Simple and flexible routing system with support for route groups and middleware

## Is It Production-Ready?
✅ *"We’re almost in beta—but here’s what’s already working:"*
- [x] A solid foundation
- [x] Truly modular design with nex to no dependencies
- [X] WebSockets
- [X] Async CQRS
- [x] AMQP support
- [x] conenction pooling
- [X] 8K RPS benchmarks on lightweight 5$ VPS instances 

**Dare to try it?**
```bash
composer create-project ody/framework ody
cd ody
php ody server:start

curl http://127.0.0.1:9501/users | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   342    0   342    0     0   249k      0 --:--:-- --:--:-- --:--:--  333k
[
  {
    "id": 1,
    "name": "John Doe",
    "email": "john@example.com"
  },
  ...
]
```  

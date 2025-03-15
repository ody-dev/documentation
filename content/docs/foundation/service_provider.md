---
title: Service Providers
---

Service providers are used to register services with the application. Custom service providers can be created in the
`app/Providers` directory:

```php
<?php

namespace App\Providers;

use Ody\Foundation\Providers\ServiceProvider;

class CustomServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Register bindings
        $this->singleton('custom.service', function() {
            return new CustomService();
        });
    }

    public function boot(): void
    {
        // Bootstrap services
    }
}
```

Register your service provider in `config/app.php`:

```php
'providers' => [
    // Framework providers
    Ody\Foundation\Providers\DatabaseServiceProvider::class,

    // Application providers
    App\Providers\CustomServiceProvider::class,
],
```
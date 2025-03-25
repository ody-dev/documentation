---
title: Eloquent
---

Documentation is in the works!

```bash
composer require ody/database composer require illuminate/database:^11.43

php ody publish // publishes a database.php config file
```

Add service providers to config/app.php

```php
'providers' => [
    // Package providers
    \Ody\DB\Providers\DatabaseServiceProvider::class,
    \Ody\DB\Eloquent\Providers\EloquentServiceProvider::class,
],
```

Reference the Eloquent documentation for how to use the ORM.
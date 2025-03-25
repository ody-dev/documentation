---
title: Doctrine ORM
---

## Installation

1. Install via Composer:

```bash
composer require ody/database doctrine/orm doctrine/dbal symfony/cache
```

2. Register the required service providers in your `config/app.php`:

```php
'providers' => [
    // ... other providers
    Ody\DB\Doctrine\Providers\DBALServiceProvider::class,
    Ody\DB\Providers\DoctrineORMServiceProvider::class,
    // Manually create both
    App\Providers\RepositoryServiceProvider::class,
    App\Providers\DoctrineIntegrationProvider::class,
],
```

3. Publish the configuration files:

```bash
php ody vendor:publish --tag=ody/database
php ody vendor:publish --tag=ody/doctrine
```

4. Configure your database connection in `config/database.php` and Doctrine settings in `config/doctrine.php`.

## Basic Usage

### Creating Entities

Create entity classes in your `app/Entities` directory using PHP 8 attributes:

```php
<?php

namespace App\Entities;

use Doctrine\ORM\Mapping as ORM;
use DateTime;
use App\Entities\Traits\SnakeCaseColumnsTrait;

#[ORM\Entity]
#[ORM\Table(name: 'users')]
#[ORM\HasLifecycleCallbacks]
class User
{
    use SnakeCaseColumnsTrait;
    
    #[ORM\Id]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    #[ORM\Column(type: 'integer')]
    private ?int $id = null;

    #[ORM\Column(type: 'string', length: 255)]
    private string $name;
    
    #[ORM\Column(type: 'string', length: 255, unique: true)]
    private string $email;
    
    #[ORM\Column(name: "created_at", type: 'datetime')]
    private DateTime $createdAt;
    
    // Getters and setters...
    
    #[ORM\PostLoad]
    public function onLoad(): void
    {
        $this->mapSnakeCaseColumns();
    }
}
```

### Creating Repositories

Create repositories that extend the base repository class:

```php
<?php

namespace App\Repositories;

use App\Entities\User;
use Ody\DB\Doctrine\Repository\BaseRepository;

class UserRepository extends BaseRepository
{
    protected string $entityClass = User::class;
    
    public function findByEmail(string $email): ?User
    {
        return $this->findOneBy(['email' => $email]);
    }
    
    // Add custom query methods...
}
```

Register your repositories in `app/Providers/RepositoryServiceProvider.php`:

```php
$this->container->singleton(UserRepository::class, function ($app) {
    return new UserRepository();
});
```

### Using the EntityManager

Access the EntityManager through the ORM facade:

```php
use Ody\DB\Doctrine\Facades\ORM;
use App\Entities\User;

// Find an entity
$user = ORM::find(User::class, 1);

// Create and persist an entity
$user = new User();
$user->setName('John Doe');
$user->setEmail('john@example.com');
ORM::persist($user);
ORM::flush();
```

### Working with Transactions

Ensure data consistency with transactions:

```php
use Ody\DB\Doctrine\Facades\ORM;

// Method 1: Using the transaction helper
ORM::transaction(function ($em) {
    $user = new User();
    $user->setName('Jane Doe');
    $em->persist($user);
    
    // Changes will be committed if no exceptions are thrown
    // Any exception will automatically rollback the transaction
});

// Method 2: Manual transaction management
$em = ORM::entityManager();
$em->beginTransaction();

try {
    $user = new User();
    $user->setName('Bob Smith');
    $em->persist($user);
    $em->flush();
    
    $em->commit();
} catch (\Exception $e) {
    $em->rollback();
    throw $e;
}
```

### Using Repositories

Inject repositories into your controllers or services:

```php
class UserController
{
    protected UserRepository $userRepository;
    
    public function __construct(UserRepository $userRepository)
    {
        $this->userRepository = $userRepository;
    }
    
    public function index(ServerRequestInterface $request): ResponseInterface
    {
        $users = $this->userRepository->findAll();
        
        return response()->json([
            'status' => 'success',
            'data' => $users
        ]);
    }
}
```

## Advanced Usage

### Coroutine-Aware Operations

The integration automatically manages entity managers per coroutine. For advanced operations, use the
`CoroutineEntityManagerTrait`:

```php
use Ody\DB\Doctrine\Traits\CoroutineEntityManagerTrait;

class YourService
{
    use CoroutineEntityManagerTrait;
    
    public function processBatch(array $items)
    {
        // Process items in parallel using separate coroutines
        $results = $this->parallelEntityManagerOperations($items, function ($item, $em) {
            // Each operation runs in its own coroutine with its own EntityManager
            $entity = $em->find(SomeEntity::class, $item['id']);
            // Process entity...
            return $result;
        });
        
        return $results;
    }
}
```

### Event Subscribers

Register event subscribers to respond to Doctrine lifecycle events:

1. Create a subscriber class:

```php
namespace App\Doctrine\Events;

use Doctrine\Common\EventSubscriber;
use Doctrine\ORM\Events;
use Doctrine\Persistence\Event\LifecycleEventArgs;

class CustomEventSubscriber implements EventSubscriber
{
    public function getSubscribedEvents(): array
    {
        return [
            Events::postPersist,
            Events::postUpdate,
        ];
    }
    
    public function postPersist(LifecycleEventArgs $args): void
    {
        $entity = $args->getObject();
        // Do something with newly persisted entity
    }
    
    public function postUpdate(LifecycleEventArgs $args): void
    {
        $entity = $args->getObject();
        // Do something with updated entity
    }
}
```

2. Register the subscriber in `config/doctrine.php`:

```php
'event_subscribers' => [
    App\Doctrine\Events\CustomEventSubscriber::class,
],
```

### Custom Column Naming

The integration uses `UnderscoreNamingStrategy` to automatically convert between camelCase properties and snake_case
database columns.

For explicit mapping, use the name attribute:

```php
#[ORM\Column(name: "user_status", type: 'string')]
private string $userStatus;
```

### Schema Management

Use the provided console commands to manage your database schema:

```bash
# Create the database schema
php ody doctrine:schema:create

# Update the database schema
php ody doctrine:schema:update --force

# Validate the mapping metadata
php ody doctrine:schema:validate
```

## Performance Considerations

### Connection Pool Sizing

Configure the connection pool size in your database config:

```php
'connectionsPerWorker' => 10, // Adjust based on your workload
```

### Entity Manager Lifecycle

Entity managers are automatically created and disposed of with each coroutine. For long-running processes:

1. Clear the entity manager regularly to release memory:
   ```php
   ORM::clear();
   ```

2. Be mindful of detached entities when working across coroutines.

## Troubleshooting

### Transaction Issues

If transactions are not being committed or rolled back properly:

1. Check that you're properly handling exceptions in your transaction blocks
2. Use the provided `ORM::transaction()` helper when possible
3. Make sure you're using the correct entity manager instance

### Connection Pool Exhaustion

If you see connection timeout errors:

1. Increase the connection pool size
2. Ensure connections are being returned to the pool properly
3. Check for leaks using the built-in leak detection

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This package is open-sourced software licensed under the [MIT license](https://opensource.org/licenses/MIT).
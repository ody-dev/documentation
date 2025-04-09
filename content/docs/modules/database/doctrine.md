---
title: Doctrine ORM
---

This guide provides a comprehensive overview of how to use Doctrine ORM (Object-Relational Mapping) with the ODY framework, enabling efficient entity mapping and management.

## Overview

The ODY Database module's Doctrine ORM integration provides:

- Coroutine-aware entity manager with connection pooling
- Support for PHP 8 attributes for entity mapping
- Automated transaction and connection management
- Repository pattern implementation for clean data access

## Installation

```bash
composer require ody/database doctrine/orm doctrine/dbal symfony/cache
```

## Configuration

### Service Provider Registration

Register the required service providers in your application's providers configuration:

```php
'providers' => [
    // Base providers
    Ody\DB\Providers\DatabaseServiceProvider::class,
    Ody\DB\Doctrine\Providers\DBALServiceProvider::class,
    Ody\DB\Doctrine\Providers\DoctrineORMServiceProvider::class,
    
    // Your custom providers
    App\Providers\RepositoryServiceProvider::class,
    App\Providers\DoctrineIntegrationProvider::class,
],
```

### Database Configuration

```php
return [
    'charset' => 'utf8mb4',
    'environments' => [
        'local' => [
            'adapter' => 'mysql',
            'host' => env('DB_HOST', '127.0.0.1'),
            'port' => env('DB_PORT', 3306),
            'username' => env('DB_USERNAME', 'root'),
            'password' => env('DB_PASSWORD', 'root'),
            'db_name' => env('DB_DATABASE', 'ody'),
            'charset' => 'utf8mb4',
            'collation' => 'utf8mb4_general_ci',
            'prefix'    => '',
            'options' => [
                PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
                PDO::ATTR_CASE,
                PDO::CASE_LOWER,
                PDO::MYSQL_ATTR_USE_BUFFERED_QUERY => true,
                PDO::MYSQL_ATTR_DIRECT_QUERY => false,
            ],
            'pool' => [
                'enabled' => env('DB_ENABLE_POOL', false),
                'pool_name' => env('DB_POOL_NAME', 'default'),
                'connections_per_worker' => env('DB_POOL_CONN_PER_WORKER', 10),
                'minimum_idle' => 128,
                'idle_timeout' => 60.0,
                'max_lifetime' => 3600.0,
                'borrowing_timeout' => 2,
                'returning_timeout' => 1,
                'leak_detection_threshold' => 10.0,
                'keep_alive_check_interval' => 60.0,
            ]
        ],
        'production' => [
            // Production-specific configuration...
            'pool' => [
                'enabled' => env('DB_ENABLE_POOL', false),
                'connections_per_worker' => 64,
                'minimum_idle' => 32,
                // Other pool settings...
            ]
        ],
    ],
    'default_environment' => 'local',
    // Other configuration settings...
];
```

### Doctrine Configuration

Create or modify your `config/doctrine.php` file:

```php
return [
    'entity_paths' => [
        base_path('app/Entities'),
    ],
    'proxy_dir' => storage_path('proxies'),
    'naming_strategy' => \Doctrine\ORM\Mapping\UnderscoreNamingStrategy::class,
    'cache' => [
        // Cache type: array, file, redis
        'type' => env('DOCTRINE_CACHE_TYPE', 'array'),

        // TTL for file and redis cache (in seconds)
        'ttl' => env('DOCTRINE_CACHE_TTL', 3600),

        // Directory for file cache
        'directory' => storage_path('cache/doctrine'),
    ],
    'types' => [
        'json' => \Ody\DB\Doctrine\Types\JsonType::class,
    ],
    'enable_events' => env('DOCTRINE_ENABLE_EVENTS', true),
    'event_subscribers' => [
        // Add your custom event subscribers here
    ],
];
```

## Implementing Required Components

### 1. Database Middleware

The Database Middleware manages database connections for HTTP requests:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Doctrine\ORM\EntityManagerInterface;
use Psr\Log\LoggerInterface;
use Throwable;

/**
 * Middleware for handling database connections in HTTP requests
 */
class DatabaseMiddleware implements MiddlewareInterface
{
    public function __construct(
        private EntityManagerInterface $entityManager,
        private LoggerInterface        $logger
    ) {}

    /**
     * Handle the incoming request and ensure database connections are properly managed.
     *
     * @param ServerRequestInterface $request
     * @param RequestHandlerInterface $handler
     * @return ResponseInterface
     */
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        // Initialize the EntityManager for this request
        $em = $this->entityManager;

        // Handle the request
        try {
            $response = $handler->handle($request);

            // Ensure all active transactions are committed
            if ($em->getConnection()->isTransactionActive()) {
                $this->logger->warning("Transaction active at end of request - committing automatically");
                $em->getConnection()->commit();
            }

            return $response;
        } catch (Throwable $e) {
            // Roll back any open transactions on error
            if ($em->getConnection()->isTransactionActive()) {
                try {
                    $em->getConnection()->rollBack();
                } catch (Throwable $rollbackException) {
                    $this->logger->error("Failed to rollback transaction: " . $rollbackException->getMessage());
                }
            }

            throw $e;
        }
    }

    /**
     * PSR-15 middleware compatibility for frameworks that use Closure-based middleware
     *
     * @param ServerRequestInterface $request
     * @param Closure $next
     * @return ResponseInterface
     * @throws Throwable
     */
    public function handle(ServerRequestInterface $request, Closure $next): ResponseInterface
    {
        return $this->process($request, new class($next) implements RequestHandlerInterface {
            private $next;

            public function __construct(Closure $next)
            {
                $this->next = $next;
            }

            public function handle(ServerRequestInterface $request): ResponseInterface
            {
                return ($this->next)($request);
            }
        });
    }
}
```

### 2. Doctrine Integration Provider

The Doctrine Integration Provider registers custom types and middleware:

```php
<?php

namespace App\Providers;

use App\Http\Middleware\DatabaseMiddleware;
use Doctrine\DBAL\Types\Type;
use Doctrine\ORM\EntityManagerInterface;
use Ody\DB\Doctrine\Types\JsonType;
use Ody\Foundation\Providers\ServiceProvider;
use Psr\Log\LoggerInterface;

class DoctrineIntegrationProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        // Register the database middleware
        $this->container->singleton(DatabaseMiddleware::class, function ($container) {
            return new DatabaseMiddleware(
                $container->get(EntityManagerInterface::class),
                $container->get(LoggerInterface::class)
            );
        });
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        // Register custom Doctrine types
        $this->registerDoctrineTypes();

        // Add middleware to global middleware stack
        $this->registerMiddleware();
    }

    /**
     * Register custom Doctrine types
     */
    protected function registerDoctrineTypes(): void
    {
        try {
            // Check if the JsonType is already registered
            if (!Type::hasType('json')) {
                Type::addType('json', JsonType::class);
            }

            // Register more custom types here as needed
        } catch (\Throwable $e) {
            logger()->error('Failed to register Doctrine types: ' . $e->getMessage());
        }
    }

    /**
     * Register the database middleware
     */
    protected function registerMiddleware(): void
    {
        // Add the middleware to the global middleware stack if your framework supports it
        if (method_exists($this->container, 'middleware') && method_exists($this->container->middleware(), 'push')) {
            $this->container->middleware()->push(DatabaseMiddleware::class);
        }
    }
}
```

### 3. Repository Service Provider

The Repository Service Provider registers your custom repositories:

```php
<?php

namespace App\Providers;

use App\Repositories\UserRepository;
use Doctrine\ORM\EntityManagerInterface;
use Ody\Foundation\Providers\ServiceProvider;

class RepositoryServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Register user repository
        $this->container->bind(UserRepository::class, function ($container) {
            return new UserRepository(
                $container->make(EntityManagerInterface::class)
            );
        });
        
        // Register other repositories as needed
    }

    public function boot(): void
    {
        // Boot logic
    }
}
```

## Creating Entities

Create your entity classes in the `app/Entities` or in a custom defined directory using PHP 8 attributes:

```php
<?php
  
namespace App\Entities;

use Doctrine\ORM\Mapping as ORM;
use DateTime;

#[ORM\Entity]
#[ORM\Table(name: 'users')]
#[ORM\HasLifecycleCallbacks]
class User
{
    #[ORM\Id]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    #[ORM\Column(type: 'integer')]
    private ?int $id = null;

    #[ORM\Column(type: 'string', length: 255)]
    private string $name;
    
    #[ORM\Column(type: 'string', length: 255, unique: true)]
    private string $email;
    
    #[ORM\Column(name: 'created_at', type: 'datetime')]
    private DateTime $createdAt;
    
    #[ORM\Column(name: 'updated_at', type: 'datetime', nullable: true)]
    private ?DateTime $updatedAt = null;
    
    // Relationships
    #[ORM\OneToMany(targetEntity: Post::class, mappedBy: 'user')]
    private Collection $posts;
    
    public function __construct()
    {
        $this->posts = new ArrayCollection();
        $this->createdAt = new DateTime();
    }
    
    // Getters and setters
    
    public function getId(): ?int
    {
        return $this->id;
    }
    
    public function getName(): string
    {
        return $this->name;
    }
    
    public function setName(string $name): self
    {
        $this->name = $name;
        return $this;
    }
    
    public function getEmail(): string
    {
        return $this->email;
    }
    
    public function setEmail(string $email): self
    {
        $this->email = $email;
        return $this;
    }
    
    public function getCreatedAt(): DateTime
    {
        return $this->createdAt;
    }
    
    public function getUpdatedAt(): ?DateTime
    {
        return $this->updatedAt;
    }
    
    public function setUpdatedAt(?DateTime $updatedAt): self
    {
        $this->updatedAt = $updatedAt;
        return $this;
    }
    
    public function getPosts(): Collection
    {
        return $this->posts;
    }
    
    // Lifecycle callbacks
    
    #[ORM\PreUpdate]
    public function setUpdatedAtValue(): void
    {
        $this->updatedAt = new DateTime();
    }
}
```

## Creating Repositories

Create custom repositories by extending the BaseRepository class:

```php
<?php

namespace App\Repositories;

use App\Entities\User;
use Ody\DB\Doctrine\Repository\BaseRepository;
use Doctrine\ORM\EntityManagerInterface;

class UserRepository extends BaseRepository
{
    protected string $entityClass = User::class;
    
    public function __construct(EntityManagerInterface $entityManager)
    {
        parent::__construct($entityManager);
    }
    
    /**
     * Find a user by email address
     */
    public function findByEmail(string $email): ?User
    {
        return $this->findOneBy(['email' => $email]);
    }
    
    /**
     * Find active users
     */
    public function findActiveUsers(int $limit = 10): array
    {
        $qb = $this->createQueryBuilder('u')
            ->where('u.status = :status')
            ->setParameter('status', 'active')
            ->orderBy('u.name', 'ASC')
            ->setMaxResults($limit);
            
        return $qb->getQuery()->getResult();
    }
    
    /**
     * Find users with recent posts
     */
    public function findUsersWithRecentPosts(\DateTime $since): array
    {
        $qb = $this->createQueryBuilder('u');
        $qb->join('u.posts', 'p')
           ->where('p.createdAt >= :since')
           ->setParameter('since', $since)
           ->groupBy('u.id')
           ->orderBy('u.name', 'ASC');
           
        return $qb->getQuery()->getResult();
    }
}
```

## Basic Usage

### Using the EntityManager

```php
<?php

namespace App\Http\Controllers;

use App\Entities\User;
use Doctrine\ORM\EntityManagerInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class UserController
{
    public function __construct(
        private EntityManagerInterface $entityManager
    ) {}
    
    public function store(ServerRequestInterface $request): ResponseInterface
    {
        $data = $request->getParsedBody();
        
        // Create a new entity
        $user = new User();
        $user->setName($data['name']);
        $user->setEmail($data['email']);
        
        // Persist and flush
        $this->entityManager->persist($user);
        $this->entityManager->flush();
        
        return response()->json([
            'id' => $user->getId(),
            'name' => $user->getName(),
            'email' => $user->getEmail()
        ], 201);
    }
    
    public function show(ServerRequestInterface $request, int $id): ResponseInterface
    {
        // Find an entity by ID
        $user = $this->entityManager->find(User::class, $id);
        
        if (!$user) {
            return response()->json(['message' => 'User not found'], 404);
        }
        
        return response()->json([
            'id' => $user->getId(),
            'name' => $user->getName(),
            'email' => $user->getEmail()
        ]);
    }
    
    public function update(ServerRequestInterface $request, int $id): ResponseInterface
    {
        $data = $request->getParsedBody();
        
        // Find an entity by ID
        $user = $this->entityManager->find(User::class, $id);
        
        if (!$user) {
            return response()->json(['message' => 'User not found'], 404);
        }
        
        // Update entity properties
        if (isset($data['name'])) {
            $user->setName($data['name']);
        }
        
        if (isset($data['email'])) {
            $user->setEmail($data['email']);
        }
        
        // No need to call persist() for updates
        $this->entityManager->flush();
        
        return response()->json([
            'id' => $user->getId(),
            'name' => $user->getName(),
            'email' => $user->getEmail()
        ]);
    }
    
    public function destroy(ServerRequestInterface $request, int $id): ResponseInterface
    {
        // Find an entity by ID
        $user = $this->entityManager->find(User::class, $id);
        
        if (!$user) {
            return response()->json(['message' => 'User not found'], 404);
        }
        
        // Remove the entity
        $this->entityManager->remove($user);
        $this->entityManager->flush();
        
        return response()->json(['message' => 'User deleted successfully']);
    }
}
```

### Using Repositories

```php
<?php

namespace App\Http\Controllers;

use App\Repositories\UserRepository;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class UserApiController
{
    public function __construct(
        private UserRepository $userRepository
    ) {}
    
    public function index(ServerRequestInterface $request): ResponseInterface
    {
        // Use repository methods to fetch entities
        $users = $this->userRepository->findActiveUsers(20);
        
        return response()->json([
            'data' => array_map(fn($user) => [
                'id' => $user->getId(),
                'name' => $user->getName(),
                'email' => $user->getEmail()
            ], $users)
        ]);
    }
    
    public function findByEmail(ServerRequestInterface $request): ResponseInterface
    {
        $email = $request->getQueryParams()['email'] ?? '';
        
        $user = $this->userRepository->findByEmail($email);
        
        if (!$user) {
            return response()->json(['message' => 'User not found'], 404);
        }
        
        return response()->json([
            'id' => $user->getId(),
            'name' => $user->getName(),
            'email' => $user->getEmail()
        ]);
    }
    
    public function recentAuthors(ServerRequestInterface $request): ResponseInterface
    {
        $since = new \DateTime('-30 days');
        
        $users = $this->userRepository->findUsersWithRecentPosts($since);
        
        return response()->json([
            'data' => array_map(fn($user) => [
                'id' => $user->getId(),
                'name' => $user->getName(),
                'post_count' => $user->getPosts()->count()
            ], $users)
        ]);
    }
    
    public function save(ServerRequestInterface $request): ResponseInterface
    {
        $data = $request->getParsedBody();
        
        $user = new \App\Entities\User();
        $user->setName($data['name']);
        $user->setEmail($data['email']);
        
        // Use repository methods to save entities
        $this->userRepository->save($user);
        
        return response()->json([
            'id' => $user->getId(),
            'name' => $user->getName(),
            'email' => $user->getEmail()
        ], 201);
    }
}
```

## Working with Transactions

### Manual Transaction Management

```php
$this->entityManager->beginTransaction();
try {
    // Perform multiple database operations
    $user = new User();
    $user->setName('John Doe');
    $user->setEmail('john@example.com');
    $this->entityManager->persist($user);
    
    $post = new Post();
    $post->setTitle('My First Post');
    $post->setContent('Hello world!');
    $post->setUser($user);
    $this->entityManager->persist($post);
    
    $this->entityManager->flush();
    $this->entityManager->commit();
    
    return response()->json(['message' => 'User and post created successfully']);
} catch (\Exception $e) {
    $this->entityManager->rollBack();
    
    return response()->json(['error' => 'Failed to create user and post: ' . $e->getMessage()], 500);
}
```

### Using the Transactional Helper

```php
$result = $this->entityManager->transactional(function($em) use ($userData, $postData) {
    $user = new User();
    $user->setName($userData['name']);
    $user->setEmail($userData['email']);
    $em->persist($user);
    
    $post = new Post();
    $post->setTitle($postData['title']);
    $post->setContent($postData['content']);
    $post->setUser($user);
    $em->persist($post);
    
    // No need to call flush, it's done automatically
    
    return [
        'userId' => $user->getId(),
        'postId' => $post->getId()
    ];
});

return response()->json($result);
```

## Advanced Features

### Coroutine Operations

The ODY Doctrine integration provides a trait for coroutine operations:

```php
<?php

namespace App\Services;

use App\Entities\User;
use Doctrine\ORM\EntityManagerInterface;
use Ody\DB\Doctrine\Traits\CoroutineEntityManagerTrait;

class UserService
{
    use CoroutineEntityManagerTrait;
    
    private EntityManagerInterface $entityManager;
    
    public function __construct(EntityManagerInterface $entityManager)
    {
        $this->entityManager = $entityManager;
    }
    
    /**
     * Process multiple users in parallel
     */
    public function processUsers(array $userIds): array
    {
        return $this->parallelEntityManagerOperations($userIds, function($userId, $em) {
            $user = $em->find(User::class, $userId);
            if (!$user) {
                return ['userId' => $userId, 'status' => 'not_found'];
            }
            
            // Process user data
            $result = $this->calculateUserScore($user);
            
            // Update user
            $user->setScore($result['score']);
            $em->flush();
            
            return [
                'userId' => $userId,
                'status' => 'processed',
                'score' => $result['score']
            ];
        });
    }
    
    /**
     * Perform a transaction within a dedicated coroutine
     */
    public function createUserInBackground(array $userData): void
    {
        $this->transactionalCoroutine(function($em) use ($userData) {
            $user = new User();
            $user->setName($userData['name']);
            $user->setEmail($userData['email']);
            
            $em->persist($user);
            // No need to call flush in transactional callback
            
            logger()->info("Created user {$user->getId()} in background coroutine");
        });
    }
    
    /**
     * Example calculation method
     */
    private function calculateUserScore(User $user): array
    {
        // Simulate complex calculation
        return [
            'score' => rand(1, 100)
        ];
    }
}
```

### Custom Event Subscribers

Create event subscribers to hook into Doctrine's lifecycle events:

```php
<?php

namespace App\Doctrine\Events;

use Doctrine\Common\EventSubscriber;
use Doctrine\ORM\Events;
use Doctrine\Persistence\Event\LifecycleEventArgs;
use App\Entities\User;
use Psr\Log\LoggerInterface;

class UserEventSubscriber implements EventSubscriber
{
    private LoggerInterface $logger;
    
    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }
    
    /**
     * {@inheritdoc}
     */
    public function getSubscribedEvents(): array
    {
        return [
            Events::postPersist,
            Events::preUpdate,
            Events::postRemove,
        ];
    }
    
    /**
     * Called after a new entity is persisted
     */
    public function postPersist(LifecycleEventArgs $args): void
    {
        $entity = $args->getObject();
        
        if ($entity instanceof User) {
            $this->logger->info("User created: {$entity->getName()} (ID: {$entity->getId()})");
            
            // Perform additional actions like sending welcome email, etc.
        }
    }
    
    /**
     * Called before an entity is updated
     */
    public function preUpdate(LifecycleEventArgs $args): void
    {
        $entity = $args->getObject();
        
        if ($entity instanceof User) {
            // Set updated timestamp if not already set
            if (!$entity->getUpdatedAt()) {
                $entity->setUpdatedAt(new \DateTime());
            }
        }
    }
    
    /**
     * Called after an entity is removed
     */
    public function postRemove(LifecycleEventArgs $args): void
    {
        $entity = $args->getObject();
        
        if ($entity instanceof User) {
            $this->logger->info("User deleted: {$entity->getName()} (ID: {$entity->getId()})");
            
            // Perform cleanup actions
        }
    }
}
```

Register the subscriber in your `config/doctrine.php` file:

```php
'event_subscribers' => [
    App\Doctrine\Events\UserEventSubscriber::class,
],
```

## Schema Management

You can use Doctrine's schema management tools to create and update your database schema:

```php
<?php

namespace App\Console\Commands;

use Doctrine\ORM\EntityManagerInterface;
use Doctrine\ORM\Tools\SchemaTool;
use Ody\Foundation\Console\Command;

class UpdateDatabaseSchema extends Command
{
    protected $signature = 'doctrine:schema:update {--force : Force schema update}';
    protected $description = 'Update database schema from entity definitions';
    
    private EntityManagerInterface $entityManager;
    
    public function __construct(EntityManagerInterface $entityManager)
    {
        parent::__construct();
        $this->entityManager = $entityManager;
    }
    
    public function handle(): int
    {
        $metadataFactory = $this->entityManager->getMetadataFactory();
        $schemaTool = new SchemaTool($this->entityManager);
        
        $force = $this->option('force') === true;
        $classMetadata = $metadataFactory->getAllMetadata();
        
        if (empty($classMetadata)) {
            $this->error('No entity metadata found. Did you create your entity classes?');
            return 1;
        }
        
        $this->info('Updating database schema...');
        
        if ($force) {
            $schemaTool->updateSchema($classMetadata);
            $this->info('Database schema updated successfully!');
        } else {
            $sql = $schemaTool->getUpdateSchemaSql($classMetadata);
            
            if (empty($sql)) {
                $this->info('Nothing to update - your database is already in sync with the current entity metadata.');
                return 0;
            }
            
            $this->info(sprintf('Database schema update SQL statements (%d statements):', count($sql)));
            $this->newLine();
            
            foreach ($sql as $query) {
                $this->line($query);
            }
            
            $this->newLine();
            $this->info('To execute these statements, re-run this command with the --force option.');
        }
        
        return 0;
    }
}
```

## Performance Optimization

### Entity Manager Clearing

For long-running processes or batch operations, clear the EntityManager periodically to free memory:

```php
public function processBatch(array $items): void
{
    foreach ($items as $i => $item) {
        $entity = $this->entityManager->find(SomeEntity::class, $item['id']);
        
        // Process entity
        $entity->setSomeProperty($item['value']);
        
        // Flush periodically
        if ($i % 100 === 0) {
            $this->entityManager->flush();
            $this->entityManager->clear(); // Clear managed entities
            
            // Re-fetch entities that you still need after clearing
        }
    }
    
    // Final flush for remaining entities
    $this->entityManager->flush();
}
```

### Query Result Iteration

For large result sets, use iterators to process entities in batches:

```php
$query = $this->entityManager->createQuery('SELECT u FROM App\Entities\User u');
$query->setHint(\Doctrine\ORM\Query::HINT_FORCE_PARTIAL_LOAD, true);

$iterableResult = $query->iterate();

$batchSize = 100;
$i = 0;

foreach ($iterableResult as $row) {
    $user = $row[0];
    
    // Process user
    $user->setSomeProperty('value');
    
    if (($i % $batchSize) === 0) {
        $this->entityManager->flush();
        $this->entityManager->clear();
    }
    
    ++$i;
}

$this->entityManager->flush();
```

### Optimizing Entity Hydration

For read-only operations, use custom hydration modes to improve performance:

```php
// Array hydration (faster than object hydration)
$query = $this->entityManager->createQuery('SELECT u FROM App\Entities\User u');
$results = $query->getResult(\Doctrine\ORM\Query::HYDRATE_ARRAY);

// Scalar results (even faster for simple data)
$query = $this->entityManager->createQuery('SELECT u.id, u.name FROM App\Entities\User u');
$results = $query->getResult(\Doctrine\ORM\Query::HYDRATE_SCALAR);

// Custom object hydration
$query = $this->entityManager->createQuery('SELECT u.id, u.name, u.email FROM App\Entities\User u');
$results = $query->getResult(\Doctrine\ORM\Query::HYDRATE_OBJECT);
```

## Troubleshooting

### Connection Pool Issues

If you experience connection pool-related errors:

1. Check your pool configuration in `config/database.php`
2. Ensure your `keep_alive_check_interval` is less than the MySQL `wait_timeout`
3. Adjust `connections_per_worker` and `minimum_idle` based on your application load

### Transaction Management

If transactions are not being properly committed or rolled back:

1. Use the `DatabaseMiddleware` provided in this guide
2. Ensure all database operations happen within the same coroutine
3. Always use try/catch blocks with proper rollback logic
4. Consider using the `transactional` helper method for simpler management

### Entity Lifecycle

If you encounter issues with entity states or lifecycle events:

1. Be aware of the different entity states (new, managed, detached, removed)
2. Remember that `clear()` detaches all entities from the EntityManager
3. Lifecycle callbacks require the `#[ORM\HasLifecycleCallbacks]` attribute on the entity class
4. Use entity repositories for consistent entity management

For more details on specific Doctrine ORM features, refer to the [official Doctrine documentation](https://www.doctrine-project.org/projects/doctrine-orm/en/latest/index.html).

## License

This package is open-sourced software licensed under the [MIT license](https://opensource.org/licenses/MIT).

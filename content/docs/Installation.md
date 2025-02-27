---
title: Installation
weight: 1
---

## Quickstart
The fastest way to get up and running is by installing a skeleton project. This bootstraps all the 
required components and installs Eloquent and Swoole components.
```shell
create-project ody/ody-skel ody
cp .env.example .env
```

By default, a UserController, User model and UserRepository is provided with basic crud methods. Set up your DB
credentials in `.env` and create a users table. This can be done through migrations, to create your first migration run:

```shell
php ody migrations:create UsersTable --config config/database.php
```

The migration file is created in the folder `database\migrations`. [docs](url)

```php
<?php
declare(strict_types=1);

use Ody\DB\Migrations\Migration\AbstractMigration;

final class UsersTable extends AbstractMigration
{
    protected function up(): void
    {
        // add this to the generated migrations 
        $this->table('users')
            ->addColumn('first_name', 'string')
            ->addColumn('last_name', 'string')
            ->addColumn('password', 'string')
            ->addColumn('email', 'integer')
            ->addColumn('created_at', 'datetime')
            ->addColumn('updated_at', 'datetime')
            ->addIndex('email', Index::TYPE_UNIQUE)
            ->create();
    }

    protected function down(): void
    {
        $this->table('users')
            ->drop();
    }
}
```

Finally run the migration.

```shell
php ody migrations:run
```
Start the app and create a user.

```shell
php ody server:start -p // the -p flag starts a regular built-in php server

POST http://localhost:9501/users
Content-Type: application/json
{
  "first_name": "John",
  "last_name": "Doe",
  "email": "john@mail.com",
  "password": "asupersecurepassword"
}

GET http://localhost:9501/users/1
```

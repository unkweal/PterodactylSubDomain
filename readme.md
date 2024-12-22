**Шаг 1: Установите cloudflaresdk**

```shell
composer require cloudflare/sdk:dev-master
```

**Шаг 2: Скопируйте файлы из папки panelfiles в корнивую папку вашего Pterodactyl**

**Шаг 3: Редактирование файла `/routes/api-client.php`**

Вставьте следующий код в файл `/routes/api-client.php` перед строкой `Route::group(['prefix' => '/settings'], function () {`:

```php
Route::group(['prefix' => '/subdomain'], function () {
    Route::get('/', [Client\Servers\SubdomainController::class, 'index']);
    Route::post('/create', [Client\Servers\SubdomainController::class, 'create']);
    Route::delete('/delete/{id}', [Client\Servers\SubdomainController::class, 'delete']);
});
```

**Шаг 4: Редактирование файла `/routes/admin.php`**

Вставьте следующий код в конец файла `/routes/admin.php`:

```php
/*
|--------------------------------------------------------------------------
| SubDomain Controller Routes
|--------------------------------------------------------------------------
|
| Endpoint: /admin/subdomain
|
*/
Route::group(['prefix' => 'subdomain'], function () {
    Route::get('/', [Admin\SubDomainController::class, 'index'])->name('admin.subdomain');
    Route::get('/new', [Admin\SubDomainController::class, 'new'])->name('admin.subdomain.new');
    Route::get('/edit/{id}', [Admin\SubDomainController::class, 'edit'])->name('admin.subdomain.edit');

    Route::post('/settings', [Admin\SubDomainController::class, 'settings'])->name('admin.subdomain.settings');
    Route::post('/create', [Admin\SubDomainController::class, 'create'])->name('admin.subdomain.create');
    Route::post('/update/{id}', [Admin\SubDomainController::class, 'update'])->name('admin.subdomain.update');

    Route::delete('/delete', [Admin\SubDomainController::class, 'delete'])->name('admin.subdomain.delete');
});
```

**Шаг 5: Редактирование файла `/resources/views/layouts/admin.blade.php`**

Вставьте следующий код в файл `/resources/views/layouts/admin.blade.php` перед строкой `<li class="{{ !starts_with(Route::currentRouteName(), 'admin.nodes') ?: 'active' }}">`:

```html
<li class="{{ ! starts_with(Route::currentRouteName(), 'admin.subdomain') ?: 'active' }}">
	<a href="{{ route('admin.subdomain') }}">
		<i class="fa fa-globe"></i> <span>SubDomain Manager</span>
	</a>
</li>
```

**Шаг 6: Редактирование файла `/app/Services/Servers/ServerDeletionService.php`**

Вставьте следующий код в файл `/app/Services/Servers/ServerDeletionService.php`:

```php

<?php

namespace Pterodactyl\Services\Servers;

use Illuminate\Http\Response;
use Pterodactyl\Models\Server;
use Illuminate\Support\Facades\Log;
use Illuminate\Database\ConnectionInterface;
use Pterodactyl\Repositories\Wings\DaemonServerRepository;
use Pterodactyl\Services\Databases\DatabaseManagementService;
use Pterodactyl\Exceptions\Http\Connection\DaemonConnectionException;
use Pterodactyl\Services\Servers\SubDomainDeletionService;

class ServerDeletionService
{
    protected bool $force = false;

    /**
     * ServerDeletionService constructor.
     */
    public function __construct(
        private ConnectionInterface $connection,
        private DaemonServerRepository $daemonServerRepository,
        private DatabaseManagementService $databaseManagementService,
        private SubDomainDeletionService $subDomainDeletionService
    ) {
    }

    /**
     * Set if the server should be forcibly deleted from the panel (ignoring daemon errors) or not.
     */
    public function withForce(bool $bool = true): self
    {
        $this->force = $bool;

        return $this;
    }

    /**
     * Delete a server from the panel and remove any associated databases from hosts.
     *
     * @throws \Throwable
     * @throws \Pterodactyl\Exceptions\DisplayException
     */
    public function handle(Server $server): void
    {
        try {
            $this->daemonServerRepository->setServer($server)->delete();
        } catch (DaemonConnectionException $exception) {
            // If there is an error not caused a 404 error and this isn't a forced delete,
            // go ahead and bail out. We specifically ignore a 404 since that can be assumed
            // to be a safe error, meaning the server doesn't exist at all on Wings so there
            // is no reason we need to bail out from that.
            if (!$this->force && $exception->getStatusCode() !== Response::HTTP_NOT_FOUND) {
                throw $exception;
            }

            Log::warning($exception);
        }

        $this->connection->transaction(function () use ($server) {
            foreach ($server->databases as $database) {
                try {
                    $this->databaseManagementService->delete($database);
                } catch (\Exception $exception) {
                    if (!$this->force) {
                        throw $exception;
                    }

                    // Oh well, just try to delete the database entry we have from the database
                    // so that the server itself can be deleted. This will leave it dangling on
                    // the host instance, but we couldn't delete it anyways so not sure how we would
                    // handle this better anyways.
                    //
                    // @see https://github.com/pterodactyl/panel/issues/2085
                    $database->delete();

                    Log::warning($exception);
                }
            }

            $this->subDomainDeletionService->delete($server->id, $server->egg_id);
            $server->delete();
        });
    }
}
```

**Шаг 6: Редактирование файла `/app/Models/Permission.php`**

Вставьте следующую строку в файл `/app/Models/Permission.php` после строки `'websocket' => [`:

```php
'subdomain' => [
    'description' => 'Manage Subdomain',
    'keys' => [
        'manage' => 'Create / Delete subdomain for current server.',
    ],
],
```

**Шаг 7: Редактирование файла `/resources/scripts/routers/routes.ts`**

Вставьте следующий код в файл `/resources/scripts/routers/routes.ts` после строки `import requireServerPermission from '@/hoc/requireServerPermission';`:

```jsx
import SubdomainContainer from '@/components/server/subdomain/SubdomainContainer';
```

Вставьте следующий код в файл `/resources/scripts/routers/routes.ts`:

```jsx
{
    path: '/subdomain',
    permission: 'subdomain.*',
    name: 'Domain',
    component: SubdomainContainer,
},
```

**Настройка CloudFlare:**

- Электронная почта CloudFlare: ваш адрес электронной почты для профиля Cloudflare.
- Ключ CloudFlare API: ваш глобальный ключ API для вашего профиля Cloudflare.
- Добавьте свои домены в Cloudflare.
- Не забудьте установить псевдоним IP для распределения в каждом узле.

**Шаг 8: Запуск команд после вставки кода**

После вставки всего кода в приложение, ресурсы, базу данных и папку поставщика, выполните следующие команды в директории `/var/www/pterodactyl/`:

```shell
yarn install
yarn build:production
php artisan route:clear && php artisan cache:clear && php artisan view:clear
php artisan migrate
```
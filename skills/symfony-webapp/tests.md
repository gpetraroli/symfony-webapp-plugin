# Tests

Install this once, after `webapp` and after the first migration exists. An app with login runs the security tests below. With login disabled, the home test in section 6 replaces them. Every new resource adds a factory and a controller test from section 7. Commands run inside the `php` service.

Skip interactive `make:test` and `make:factory`. They stall.

The suite uses a separate database, `app_test`. `compose.yaml` injects `DATABASE_URL` for the `app` database, and that process value overrides `.env.test`. Foundry's reset drops and recreates whichever database `DATABASE_URL` points at. The `phpunit.xml.dist` value below is what keeps that reset off the development database. Do not point the test URL back at `app`. Do not add a second database service. Do not run `doctrine:schema:drop`.

## 1. Packages

```sh
docker compose exec php composer require --dev symfony/test-pack "zenstruck/foundry:^2.9" dama/doctrine-test-bundle --no-interaction
```

Foundry 2.9 or newer is required so the PHPUnit extension replaces the `Factories` trait. If Composer cannot resolve that constraint, stop and report it.

Leave `tests/bootstrap.php` and `config/bundles.php` as the recipes wrote them. Keep the `App\Tests\` PSR-4 entry the recipe adds under `autoload-dev`. If it is missing, add `"App\\Tests\\": "tests/"`. If a recipe changes `Dockerfile` or `compose.yaml`, run `docker compose up --build --wait`.

## 2. PHPUnit

Edit `phpunit.xml.dist` in place. Leave the bootstrap, the testsuite, and the `APP_ENV` server entry.

Copy the `php` service `DATABASE_URL` from `compose.yaml`. Change only the database name from `app` to `app_test`. Escape every `&` as `&amp;`. Add that URL inside `<php>` with `force="true"`, as both `<server>` and `<env>`, so it wins over the Compose process environment:

```xml
<server name="DATABASE_URL" value="postgresql://app:!ChangeMe!@database:5432/app_test?serverVersion=16&amp;charset=utf8" force="true" />
<env name="DATABASE_URL" value="postgresql://app:!ChangeMe!@database:5432/app_test?serverVersion=16&amp;charset=utf8" force="true" />
```

The value above matches the Symfony Docker defaults. Replace the user, password, host, and `serverVersion` when `compose.yaml` differs.

Register both extensions before `</phpunit>`. If the recipe already opened an `<extensions>` element, add the missing entries there and keep that file's element name (`bootstrap` or `extension`):

```xml
<extensions>
    <bootstrap class="DAMA\DoctrineTestBundle\PHPUnit\PHPUnitExtension"/>
    <bootstrap class="Zenstruck\Foundry\PHPUnit\FoundryExtension">
        <parameter name="enabled-auto-reset" value="true"/>
    </bootstrap>
</extensions>
```

`enabled-auto-reset` resets the database for `KernelTestCase` and `WebTestCase`. Do not add `#[ResetDatabase]` or the `Factories` trait on test classes.

If `.phpunit.cache/` is missing from `.gitignore`, add it. Do not commit `.phpunit.cache/` or a local `phpunit.xml`.

## 3. Foundry reset

In `config/packages/zenstruck_foundry.yaml`, set the reset mode under `when@test`. Merge it into the recipe file. With DAMA enabled, Foundry migrates once per suite, then DAMA rolls each test back:

```yaml
when@test:
    zenstruck_foundry:
        orm:
            reset:
                mode: migrate
```

## 4. Factory services

Factories live under `tests/`, not `src/`. Foundry and Faker are dev dependencies. Production code must not reference them.

In `config/services.yaml`, register the test namespace only for the test environment. If `when@test` already exists, add the `App\Tests\` resource under its `services` key. A second `when@test` key would replace the first:

```yaml
when@test:
    services:
        _defaults:
            autowire: true
            autoconfigure: true

        App\Tests\:
            resource: '../tests/'
            exclude:
                - '../tests/**/*Test.php'
```

The exclude keeps test classes out of the container. Autoconfigure tags each factory with `foundry.factory`.

## 5. What to test

| Write | Skip |
|---|---|
| A Foundry factory for each entity | Entity tests |
| A `WebTestCase` for each controller | Form tests and repository tests |
| A `PHPUnit\Framework\TestCase` for each service | Kernel, Foundry, and `persist` inside a service test |

Search is asserted from the index page. Service tests construct the service and assert the entity change. Mock a constructor dependency when the service needs one. Do not boot the kernel, and do not extend `WebTestCase`, for a service test.

When login is disabled, skip `UserFactory`, the security tests, and the registration service test. `GET /` is public: write `tests/Home/Controller/HomeControllerTest.php` from the template at the end of section 6. Feature tests then request pages without `loginUser`, and they omit the guest-redirect test.

## 6. Security tests

Write these when login is enabled, after the user migration exists.

### tests/User/Factory/UserFactory.php

The plain default password is `password`. `afterInstantiate` hashes it. Pass plain passwords in `createOne()`.

```php
<?php

declare(strict_types=1);

namespace App\Tests\User\Factory;

use App\User\Entity\User;
use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;
use Zenstruck\Foundry\Persistence\PersistentObjectFactory;

/**
 * @extends PersistentObjectFactory<User>
 */
final class UserFactory extends PersistentObjectFactory
{
    public function __construct(
        private readonly ?UserPasswordHasherInterface $passwordHasher = null,
    ) {
        parent::__construct();
    }

    public static function class(): string
    {
        return User::class;
    }

    protected function defaults(): array
    {
        return [
            'email' => self::faker()->unique()->safeEmail(),
            'password' => 'password',
        ];
    }

    protected function initialize(): static
    {
        return $this->afterInstantiate(function (User $user): void {
            if ($this->passwordHasher !== null) {
                $user->setPassword($this->passwordHasher->hashPassword($user, $user->getPassword()));
            }
        });
    }
}
```

### tests/User/Service/RegistrationServiceTest.php

```php
<?php

declare(strict_types=1);

namespace App\Tests\User\Service;

use App\User\Entity\User;
use App\User\Service\RegistrationService;
use PHPUnit\Framework\TestCase;
use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;

final class RegistrationServiceTest extends TestCase
{
    public function testHashPasswordSetsTheHashedPassword(): void
    {
        $user = new User();
        $hasher = $this->createMock(UserPasswordHasherInterface::class);
        $hasher->expects(self::once())
            ->method('hashPassword')
            ->with($user, 'password')
            ->willReturn('hashed');

        (new RegistrationService($hasher))->hashPassword($user, 'password');

        self::assertSame('hashed', $user->getPassword());
    }
}
```

### tests/User/Controller/SecurityControllerTest.php

Login submits the firewall form (`_username`, `_password`). The crawler sends the CSRF token already on the page. Register submits `RegistrationType`. A failed login redirects back to `/login`.

```php
<?php

declare(strict_types=1);

namespace App\Tests\User\Controller;

use App\Tests\User\Factory\UserFactory;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

final class SecurityControllerTest extends WebTestCase
{
    public function testLoginPageIsPublic(): void
    {
        $client = static::createClient();
        $client->request('GET', '/login');

        self::assertResponseIsSuccessful();
    }

    public function testGuestIsRedirectedHomeToLogin(): void
    {
        $client = static::createClient();
        $client->request('GET', '/');

        self::assertResponseRedirects('/login');
    }

    public function testLoginRedirectsToHome(): void
    {
        UserFactory::createOne(['email' => 'ada@example.com']);

        $client = static::createClient();
        $client->request('GET', '/login');
        $client->submitForm('Sign in', [
            '_username' => 'ada@example.com',
            '_password' => 'password',
        ]);

        self::assertResponseRedirects('/');
    }

    public function testFailedLoginShowsAnError(): void
    {
        UserFactory::createOne(['email' => 'ada@example.com']);

        $client = static::createClient();
        $client->request('GET', '/login');
        $client->submitForm('Sign in', [
            '_username' => 'ada@example.com',
            '_password' => 'wrong-password',
        ]);

        self::assertResponseRedirects('/login');
        $client->followRedirect();
        self::assertSelectorExists('.alert-danger');
    }

    public function testRegisterRedirectsToLogin(): void
    {
        $client = static::createClient();
        $client->request('GET', '/register');
        $client->submitForm('Create account', [
            'registration[email]' => 'ada@example.com',
            'registration[plainPassword][first]' => 'password',
            'registration[plainPassword][second]' => 'password',
        ]);

        self::assertResponseRedirects('/login');
        UserFactory::assert()->count(1);
    }

    public function testRegisterRejectsAShortPassword(): void
    {
        $client = static::createClient();
        $client->request('GET', '/register');
        $client->submitForm('Create account', [
            'registration[email]' => 'ada@example.com',
            'registration[plainPassword][first]' => 'short',
            'registration[plainPassword][second]' => 'short',
        ]);

        self::assertResponseStatusCodeSame(422);
        self::assertSelectorExists('.invalid-feedback');
        UserFactory::assert()->count(0);
    }

    public function testHomeIsVisibleAfterLogin(): void
    {
        $client = static::createClient();
        $client->loginUser(UserFactory::createOne());
        $client->request('GET', '/');

        self::assertResponseIsSuccessful();
        self::assertSelectorExists('h1');
    }
}
```

### tests/Home/Controller/HomeControllerTest.php

Write this only when login is disabled.

```php
<?php

declare(strict_types=1);

namespace App\Tests\Home\Controller;

use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

final class HomeControllerTest extends WebTestCase
{
    public function testHomeIsPublic(): void
    {
        $client = static::createClient();
        $client->request('GET', '/');

        self::assertResponseIsSuccessful();
    }
}
```

## 7. Resource tests

`Note` below is the example. Rename with the resource (`note`, `Note`, `NoteFactory`, `NoteControllerTest`). The index path is `/notes/` because the controller mounts `#[Route('/')]` on the `/notes` prefix. Use the same trailing slash the route generates.

When login is disabled, delete `testGuestIsRedirectedToLogin`, drop `loginUser`, and call `static::createClient()` directly.

### tests/Note/Factory/NoteFactory.php

Defaults cover every non-nullable field that has a setter. `createdAt` is set in the entity constructor, so it stays out of `defaults()`.

```php
<?php

declare(strict_types=1);

namespace App\Tests\Note\Factory;

use App\Note\Entity\Note;
use Zenstruck\Foundry\Persistence\PersistentObjectFactory;

/**
 * @extends PersistentObjectFactory<Note>
 */
final class NoteFactory extends PersistentObjectFactory
{
    public static function class(): string
    {
        return Note::class;
    }

    protected function defaults(): array
    {
        return [
            'title' => 'A note',
            'body' => 'Body',
        ];
    }
}
```

### tests/Note/Controller/NoteControllerTest.php

The show page has two controls labelled Delete. The modal opener is `type="button"`. Submit the form button by its name, `delete[delete]`.

```php
<?php

declare(strict_types=1);

namespace App\Tests\Note\Controller;

use App\Tests\Note\Factory\NoteFactory;
use App\Tests\User\Factory\UserFactory;
use Symfony\Bundle\FrameworkBundle\KernelBrowser;
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

final class NoteControllerTest extends WebTestCase
{
    public function testGuestIsRedirectedToLogin(): void
    {
        $client = static::createClient();
        $client->request('GET', '/notes/');

        self::assertResponseRedirects('/login');
    }

    public function testIndexListsANote(): void
    {
        NoteFactory::createOne(['title' => 'Shopping']);

        $client = $this->clientAsUser();
        $client->request('GET', '/notes/');

        self::assertResponseIsSuccessful();
        self::assertSelectorTextContains('td', 'Shopping');
    }

    public function testSearchFiltersByTitle(): void
    {
        NoteFactory::createOne(['title' => 'Shopping']);
        NoteFactory::createOne(['title' => 'Other']);

        $client = $this->clientAsUser();
        $client->request('GET', '/notes/');
        $client->submitForm('Search', [
            'query' => 'shop',
        ]);

        self::assertResponseIsSuccessful();
        self::assertSelectorTextContains('td', 'Shopping');
        self::assertSelectorTextNotContains('td', 'Other');
    }

    public function testCreateRedirectsToShow(): void
    {
        $client = $this->clientAsUser();
        $client->request('GET', '/notes/new');
        $client->submitForm('Save', [
            'note[title]' => 'Hello',
            'note[body]' => 'Body',
        ]);

        self::assertResponseRedirects();
        $client->followRedirect();
        self::assertSelectorTextContains('h1', 'Hello');
    }

    public function testCreateRejectsABlankTitle(): void
    {
        $client = $this->clientAsUser();
        $client->request('GET', '/notes/new');
        $client->submitForm('Save', [
            'note[title]' => '',
            'note[body]' => '',
        ]);

        self::assertResponseStatusCodeSame(422);
        self::assertSelectorExists('.invalid-feedback');
        NoteFactory::assert()->count(0);
    }

    public function testEditUpdatesTheTitle(): void
    {
        $note = NoteFactory::createOne(['title' => 'Old']);

        $client = $this->clientAsUser();
        $client->request('GET', '/notes/'.$note->getId().'/edit');
        $client->submitForm('Save', [
            'note[title]' => 'New',
        ]);

        self::assertResponseRedirects('/notes/'.$note->getId());
        $client->followRedirect();
        self::assertSelectorTextContains('h1', 'New');
    }

    public function testDeleteRemovesTheNote(): void
    {
        $note = NoteFactory::createOne(['title' => 'Gone']);

        $client = $this->clientAsUser();
        $client->request('GET', '/notes/'.$note->getId());
        $client->submit($client->getCrawler()->selectButton('delete[delete]')->form());

        self::assertResponseRedirects('/notes/');
        NoteFactory::assert()->count(0);
    }

    private function clientAsUser(): KernelBrowser
    {
        $client = static::createClient();
        $client->loginUser(UserFactory::createOne());

        return $client;
    }
}
```

### tests/Note/Service/NoteServiceTest.php

Write this only when the feature has a service. Point the test at the method that was written. Do not add a method so the test has something to call. The sample matches `NoteService::publish` from [feature.md](feature.md). A service with no constructor dependencies is created with `new`.

```php
<?php

declare(strict_types=1);

namespace App\Tests\Note\Service;

use App\Note\Entity\Note;
use App\Note\Service\NoteService;
use PHPUnit\Framework\TestCase;

final class NoteServiceTest extends TestCase
{
    public function testPublishSetsPublishedAt(): void
    {
        $note = new Note();
        (new NoteService())->publish($note);

        self::assertNotNull($note->getPublishedAt());
    }
}
```

## 8. Run

Foundry creates `app_test` on the first run, migrates it, and DAMA rolls each test back. The database user from the Symfony Docker defaults can create databases. If the reset fails, stop and report the error.

```sh
docker compose exec php php vendor/bin/phpunit
```

The suite must pass before the task is finished. Fix the application or the test, then run the suite again.

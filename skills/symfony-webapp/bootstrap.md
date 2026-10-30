# Bootstrap templates

Use these when scaffolding an app. Replace `<app_dir>`. Layout partials: [ui.md](ui.md). A CRUD resource is a separate request: [feature.md](feature.md). Do not create the Note example while bootstrapping.

Run every command from the app directory (the one that contains `compose.yaml`). Docker Compose v2.10+ is required. Do not install PHP or Composer on the host.

## 1. Create the project

Clone [dunglas/symfony-docker](https://github.com/dunglas/symfony-docker/tree/main) into an empty directory. Drop the upstream git history so the app has its own repository. Delete `docs/`: that directory is the Symfony Docker project's documentation, not the app's.

```sh
git clone --depth 1 https://github.com/dunglas/symfony-docker.git <app_dir>
cd <app_dir>
rm -rf .git docs
git init
docker compose build --pull --no-cache
docker compose up --wait
```

Do not set `SYMFONY_VERSION` or `STABILITY` unless the user asked. The entrypoint installs the current stable `symfony/skeleton`.

On first start, `frankenphp/docker-entrypoint.sh` sees `composer.json` equal to `{}` and runs:

```sh
composer create-project "symfony/skeleton $SYMFONY_VERSION" tmp --stability="$STABILITY" --prefer-dist --no-progress --no-interaction --no-install
```

That package is the minimal skeleton: `framework-bundle`, `console`, `dotenv`, `runtime`, and `yaml`. It does not include Twig, Asset Mapper, Security, Form, Validator, or Doctrine. Those arrive in the next step with the `webapp` pack.

The entrypoint then copies the skeleton into `/app`, sets `extra.symfony.docker` to `true`, and deletes that install block from the entrypoint and from `compose.yaml`. Do not put the block back.

If the container instead prints `To finish the installation please press Ctrl+C` and sleeps, stop Compose and run `docker compose up --build --wait` once before continuing.

Confirm the skeleton is installed:

```sh
docker compose exec php php bin/console -V
```

`composer.json` must no longer be `{}`.

## 2. Webapp pack

`docker compose up` does not install `webapp`. Require it inside the container so Flex recipes can edit `Dockerfile` and `compose.yaml`. `webapp` pulls Twig, Asset Mapper, Security, Form, Validator, and `orm-pack` (PostgreSQL).

```sh
docker compose exec php composer require webapp --no-interaction
docker compose up --build --wait
```

Rebuild whenever a recipe changes `Dockerfile` or `compose.yaml` (typical: `pdo_pgsql`, a `database` service, Mailpit). Leave those recipe edits in place. Do not add a second database service by hand.

`DATABASE_URL` on the `php` service in `compose.yaml` is what PHP sees inside Docker (`host` `database`, database `app`). Do not point it at `127.0.0.1`.

## 3. Bootstrap

CSS is compiled from Sass. Bootstrap's widget script comes from the importmap. Every other browser behavior is a Hotwire Stimulus controller with one responsibility. See [ui.md](ui.md).

```sh
docker compose exec php composer require symfonycasts/sass-bundle twbs/bootstrap symfony/ux-twig-component --no-interaction
docker compose exec php php bin/console importmap:require bootstrap
```

`twbs/bootstrap` is the Sass source. Confirm that package is Bootstrap 5. The form theme is `bootstrap_5_layout.html.twig`. If the resolved major is not 5, stop and ask before changing the theme.

In `config/packages/symfonycasts_sass.yaml`, add a load path so `@import "bootstrap"` resolves inside `vendor`, and silence warnings that come from Bootstrap itself:

```yaml
symfonycasts_sass:
    sass_options:
        load_path:
            - '%kernel.project_dir%/vendor/twbs/bootstrap/scss'
        quiet_deps: true
```

In `config/packages/asset_mapper.yaml`, add this pattern to the existing `excluded_patterns` list. Leave `paths` as Flex wrote it. `assets/styles/app.scss` stays mapped:

```yaml
framework:
    asset_mapper:
        excluded_patterns:
            - '**/assets/styles/**/_*.scss'
```

Write `assets/styles/_theme.scss` and `assets/styles/app.scss` from [ui.md](ui.md). Delete `assets/styles/app.css` from the webapp pack. In `assets/app.js`, keep `import './stimulus_bootstrap.js'` and add `import 'bootstrap'`. Do not import `bootstrap/dist/css/bootstrap.min.css`. Do not put behavior in `app.js`. Delete `assets/controllers/hello_controller.js`. Leave `csrf_protection_controller.js`. New behavior goes in `assets/controllers/<job>_controller.js`, one responsibility per file, extending `Controller` from `@hotwired/stimulus`.

Compile inside the container:

```sh
docker compose exec php php bin/console sass:build
```

During development, run `sass:build --watch` in the background. The first build downloads the Sass binary into `bin/dart-sass`. Do not commit that directory. Before `asset-map:compile`, run `sass:build` again.

## 4. Feature folders

Code lives under `src/<Feature>/`, not under `src/Controller` or `src/Entity`. `User` and `Note` are features. Namespace `App\Note\Entity\Note` is the file `src/Note/Entity/Note.php`.

After `webapp` is installed, delete `src/Controller`, `src/Entity`, and `src/Repository`. Flex puts only an empty `.gitignore` in each. Leave `src/Migrations`. Do not recreate those three directories.

In `config/packages/doctrine.yaml`, change the Flex `App` mapping so entities under every feature are found:

```yaml
App:
    type: attribute
    is_bundle: false
    dir: '%kernel.project_dir%/src'
    prefix: 'App'
    alias: App
```

In `config/services.yaml`, replace the `../src/Entity/` exclude with `../src/*/Entity/` so entity classes are not registered as services. Leave the rest of the `App\` resource entry. Controllers, forms, repositories, and services under `src/<Feature>/` stay autowired.

## 5. Security

Edit the Flex-generated `config/packages/security.yaml` in place. Keep keys the recipe added (`when@test`, and `user_checker_on_refresh` when it is present). Replace the in-memory user provider and the `main` firewall.

```yaml
security:
    password_hashers:
        Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface: 'auto'

    providers:
        app_user_provider:
            entity:
                class: App\User\Entity\User
                property: email

    firewalls:
        dev:
            pattern: ^/(_profiler(/|$)|(_wdt|assets|build)/)
            security: false
        main:
            lazy: true
            provider: app_user_provider
            form_login:
                login_path: app_login
                check_path: app_login
                enable_csrf: true
            logout:
                path: app_logout
                target: app_login

    access_control:
        - { path: ^/login, roles: PUBLIC_ACCESS }
        - { path: ^/register, roles: PUBLIC_ACCESS }
        - { path: ^/, roles: ROLE_USER }
```

Keep the recipe's `when@test` block unchanged. If `main` already has `user_checker_on_refresh: true`, leave that key in place.

If login is disabled, skip the User entity, the login and register templates, and `access_control`. Still install `webapp`.

### src/User/Entity/User.php

Table name is `app_user` because `user` is reserved in PostgreSQL.

On Symfony 7.4, `UserInterface` still declares `eraseCredentials()`. Add an empty `eraseCredentials(): void` method when that skeleton was installed. Symfony 8 does not declare it; an extra empty method is harmless.

```php
<?php

declare(strict_types=1);

namespace App\User\Entity;

use App\User\Repository\UserRepository;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
use Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface;
use Symfony\Component\Security\Core\User\UserInterface;
use Symfony\Component\Validator\Constraints as Assert;

#[ORM\Entity(repositoryClass: UserRepository::class)]
#[ORM\Table(name: 'app_user')]
#[UniqueEntity('email')]
class User implements UserInterface, PasswordAuthenticatedUserInterface
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 180, unique: true)]
    #[Assert\NotBlank]
    #[Assert\Email]
    private string $email = '';

    /** @var list<string> */
    #[ORM\Column]
    private array $roles = [];

    #[ORM\Column]
    private string $password = '';

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getEmail(): string
    {
        return $this->email;
    }

    public function setEmail(string $email): void
    {
        $this->email = $email;
    }

    public function getUserIdentifier(): string
    {
        return $this->email;
    }

    /** @return list<string> */
    public function getRoles(): array
    {
        $roles = $this->roles;
        $roles[] = 'ROLE_USER';

        return array_values(array_unique($roles));
    }

    /** @param list<string> $roles */
    public function setRoles(array $roles): void
    {
        $this->roles = $roles;
    }

    public function getPassword(): string
    {
        return $this->password;
    }

    public function setPassword(string $password): void
    {
        $this->password = $password;
    }
}
```

### src/User/Repository/UserRepository.php

```php
<?php

declare(strict_types=1);

namespace App\User\Repository;

use App\User\Entity\User;
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\ManagerRegistry;

/** @extends ServiceEntityRepository<User> */
class UserRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, User::class);
    }
}
```

### src/User/Form/RegistrationType.php

`plainPassword` is unmapped, so its constraints stay on the form. Email constraints stay on the entity.

```php
<?php

declare(strict_types=1);

namespace App\User\Form;

use App\User\Entity\User;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\EmailType;
use Symfony\Component\Form\Extension\Core\Type\PasswordType;
use Symfony\Component\Form\Extension\Core\Type\RepeatedType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;
use Symfony\Component\Validator\Constraints\Length;
use Symfony\Component\Validator\Constraints\NotBlank;

class RegistrationType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('email', EmailType::class)
            ->add('plainPassword', RepeatedType::class, [
                'type' => PasswordType::class,
                'mapped' => false,
                'first_options' => ['label' => 'Password'],
                'second_options' => ['label' => 'Repeat password'],
                'constraints' => [
                    new NotBlank(),
                    new Length(min: 8),
                ],
            ])
            ->add('save', SubmitType::class, [
                'label' => 'Create account',
            ]);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'data_class' => User::class,
        ]);
    }
}
```

### src/User/Service/RegistrationService.php

Hashing the password is a rule, so it lives here even though only the register action calls it. `persist` and `flush` stay in the controller.

```php
<?php

declare(strict_types=1);

namespace App\User\Service;

use App\User\Entity\User;
use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;

class RegistrationService
{
    public function __construct(
        private readonly UserPasswordHasherInterface $passwordHasher,
    ) {
    }

    public function hashPassword(User $user, string $plainPassword): void
    {
        $user->setPassword($this->passwordHasher->hashPassword($user, $plainPassword));
    }
}
```

### src/User/Controller/SecurityController.php

The class route name is `app`; each method adds the suffix (`_login`, `_logout`, `_register`). `path: ''` sets that name prefix without prefixing the path. A signed-in user goes to `app_home`.

```php
<?php

declare(strict_types=1);

namespace App\User\Controller;

use App\User\Entity\User;
use App\User\Form\RegistrationType;
use App\User\Service\RegistrationService;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Security\Http\Authentication\AuthenticationUtils;

#[Route(path: '', name: 'app')]
class SecurityController extends AbstractController
{
    #[Route('/login', name: '_login')]
    public function login(AuthenticationUtils $authenticationUtils): Response
    {
        if ($this->getUser()) {
            return $this->redirectToRoute('app_home');
        }

        return $this->render('user/login.html.twig', [
            'last_username' => $authenticationUtils->getLastUsername(),
            'error' => $authenticationUtils->getLastAuthenticationError(),
        ]);
    }

    #[Route('/logout', name: '_logout')]
    public function logout(): void
    {
        throw new \LogicException('Intercepted by the firewall logout key.');
    }

    #[Route('/register', name: '_register')]
    public function register(
        Request $request,
        RegistrationService $registration,
        EntityManagerInterface $entityManager,
    ): Response {
        if ($this->getUser()) {
            return $this->redirectToRoute('app_home');
        }

        $user = new User();
        $form = $this->createForm(RegistrationType::class, $user);
        $form->handleRequest($request);

        if ($form->isSubmitted() && $form->isValid()) {
            $plainPassword = $form->get('plainPassword')->getData();
            $registration->hashPassword($user, $plainPassword);
            $entityManager->persist($user);
            $entityManager->flush();

            $this->addFlash('success', 'Account created. You can log in.');

            return $this->redirectToRoute('app_login');
        }

        return $this->render('user/register.html.twig', [
            'form' => $form,
        ]);
    }
}
```

### templates/user/login.html.twig

This is the firewall login form (`_username`, `_password`, `_csrf_token`), not a `FormType`, so the submit button stays in the template. Every `FormType` uses `->add(SubmitType::class)` instead.

```twig
{% extends 'base_layout.html.twig' %}

{% block title %}Log in{% endblock %}

{% block content %}
    <div class="row justify-content-center">
        <div class="col-md-6 col-lg-4">
            <h1 class="h3 mb-3">Log in</h1>

            {% if error %}
                <div class="alert alert-danger">{{ error.messageKey|trans(error.messageData, 'security') }}</div>
            {% endif %}

            <form method="post">
                <div class="mb-3">
                    <label class="form-label" for="username">Email</label>
                    <input class="form-control" type="email" id="username" name="_username" value="{{ last_username }}" required autofocus>
                </div>
                <div class="mb-3">
                    <label class="form-label" for="password">Password</label>
                    <input class="form-control" type="password" id="password" name="_password" required>
                </div>
                <input type="hidden" name="_csrf_token" value="{{ csrf_token('authenticate') }}">
                <button class="btn btn-primary" type="submit">Sign in</button>
            </form>
            <p class="mt-3"><a href="{{ path('app_register') }}">Create an account</a></p>
        </div>
    </div>
{% endblock %}
```

### templates/user/register.html.twig

```twig
{% extends 'base_layout.html.twig' %}

{% block title %}Register{% endblock %}

{% block content %}
    <div class="row justify-content-center">
        <div class="col-md-6 col-lg-4">
            <h1 class="h3 mb-3">Register</h1>
            {{ form_start(form) }}
                {{ form_widget(form) }}
            {{ form_end(form) }}
        </div>
    </div>
{% endblock %}
```

## 6. Home

Remove the Flex welcome controller if it only renders the default welcome page. `/` renders a home page. It does not redirect to a resource. Add a redirect only when the user has named that home route.

```php
// src/Home/Controller/HomeController.php
namespace App\Home\Controller;

#[Route(path: '', name: 'app')]
class HomeController extends AbstractController
{
    #[Route('/', name: '_home')]
    public function home(): Response
    {
        return $this->render('home/index.html.twig');
    }
}
```

```twig
{# templates/home/index.html.twig #}
{% extends 'base_layout.html.twig' %}

{% block title %}Home{% endblock %}

{% block content %}
    <h1 class="h3">{{ app_name|default('App') }}</h1>
{% endblock %}
```

## 7. Migrations

After the User entity exists:

```sh
docker compose exec php php bin/console doctrine:migrations:diff --no-interaction
```

Read the generated SQL. Set `getDescription()` to a sentence about that change before migrating. Do not leave it as `''`.

```php
public function getDescription(): string
{
    return 'Create the app_user table';
}
```

```sh
docker compose exec php php bin/console doctrine:migrations:migrate --no-interaction
```

## 8. Check

```sh
docker compose exec php php bin/console about
curl -kfsS -o /dev/null -w '%{http_code}\n' https://localhost/login
```

`/login` is public and should return 200. `/` redirects to login until a user exists. Stop the stack with `docker compose down --remove-orphans`.

## README.md

Write this into the app after adapting the resource name:

```markdown
# <app_dir>

Symfony web app (Twig, Bootstrap 5, Doctrine) running on [Symfony Docker](https://github.com/dunglas/symfony-docker) (FrankenPHP + Caddy).

## Requirements

Docker Compose v2.10 or newer. PHP and Composer run inside the `php` container.

## Start

\`\`\`sh
docker compose up --wait
\`\`\`

Open https://localhost and accept the local certificate. Register the first account at https://localhost/register.

Apply schema changes with `docker compose exec php php bin/console doctrine:migrations:migrate --no-interaction`.

## Console

\`\`\`sh
docker compose exec php php bin/console <command>
docker compose exec php composer require <package> --no-interaction
\`\`\`

After a recipe changes `Dockerfile` or `compose.yaml`:

\`\`\`sh
docker compose up --build --wait
\`\`\`

## Stop

\`\`\`sh
docker compose down --remove-orphans
\`\`\`
```

# Quick reference

Run these from the directory that contains `compose.yaml`. The `php` service must be up (`docker compose up --wait`).

| Task | Command |
|---|---|
| Create the project | `git clone --depth 1 https://github.com/dunglas/symfony-docker.git <app_dir>` then `rm -rf .git docs`, `docker compose build --pull --no-cache`, and `docker compose up --wait` |
| Symfony version | `docker compose exec php php bin/console -V` |
| Require a package | `docker compose exec php composer require <package> --no-interaction` |
| Rebuild after a recipe edits Docker files | `docker compose up --build --wait` |
| Bootstrap CSS | `docker compose exec php composer require symfonycasts/sass-bundle twbs/bootstrap symfony/ux-twig-component --no-interaction` then `docker compose exec php php bin/console sass:build` |
| Bootstrap JS | `docker compose exec php php bin/console importmap:require bootstrap` |
| Diff a migration | `docker compose exec php php bin/console doctrine:migrations:diff --no-interaction` |
| Migrate | `docker compose exec php php bin/console doctrine:migrations:migrate --no-interaction` |
| Run the test suite | `docker compose exec php php vendor/bin/phpunit` |
| Lint the container | `docker compose exec php php bin/console lint:container` |
| Twig lint | `docker compose exec php php bin/console lint:twig templates` |
| YAML lint | `docker compose exec php php bin/console lint:yaml config` |
| Open the app | `https://localhost` (local Caddy certificate) |
| Stop | `docker compose down --remove-orphans` |

## Symfony documentation

The manual is [symfony.com/doc](https://symfony.com/doc/current/index.html). Read the version that matches `bin/console -V`. `current` is the latest stable. When the app is on an older release, replace `current` in these URLs with that `major.minor` (`7.4`, `8.0`). Use a doc page for an option, a class, or a behavior these templates do not specify. Do not adopt a different project layout from an example there.

| Topic | Page |
|---|---|
| Manual index | https://symfony.com/doc/current/index.html |
| Controllers | https://symfony.com/doc/current/controller.html |
| Routing | https://symfony.com/doc/current/routing.html |
| Twig | https://symfony.com/doc/current/templates.html |
| Forms | https://symfony.com/doc/current/forms.html |
| Form types | https://symfony.com/doc/current/reference/forms/types.html |
| Form theme | https://symfony.com/doc/current/form/form_customization.html |
| Validation constraints | https://symfony.com/doc/current/validation.html |
| Constraint reference | https://symfony.com/doc/current/reference/constraints.html |
| Doctrine | https://symfony.com/doc/current/doctrine.html |
| Associations | https://symfony.com/doc/current/doctrine/associations.html |
| Security and form login | https://symfony.com/doc/current/security.html |
| CSRF | https://symfony.com/doc/current/security/csrf.html |
| Flash messages | https://symfony.com/doc/current/session.html |
| Testing | https://symfony.com/doc/current/testing.html |
| Database tests | https://symfony.com/doc/current/testing/database.html |
| AssetMapper | https://symfony.com/doc/current/frontend/asset_mapper.html |
| Twig language | https://twig.symfony.com/doc/ |

These bundles are not in the manual. Their docs use the same version rule:

| Package | Page |
|---|---|
| Doctrine Migrations | https://symfony.com/bundles/DoctrineMigrationsBundle/current/index.html |
| UX Twig Component | https://symfony.com/bundles/ux-twig-component/current/index.html |
| Stimulus | https://symfony.com/bundles/StimulusBundle/current/index.html |
| Sass | https://symfony.com/bundles/SassBundle/current/index.html |
| Foundry | https://symfony.com/bundles/ZenstruckFoundryBundle/current/index.html |

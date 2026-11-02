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
| Lint the container | `docker compose exec php php bin/console lint:container` |
| Twig lint | `docker compose exec php php bin/console lint:twig templates` |
| YAML lint | `docker compose exec php php bin/console lint:yaml config` |
| Open the app | `https://localhost` (local Caddy certificate) |
| Stop | `docker compose down --remove-orphans` |

# Quick reference

## Commands

Run from the directory that contains `compose.yaml`. The `php` service must be up (`docker compose up --wait`).

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

Do not set `SYMFONY_VERSION` unless the user asked. Do not run `symfony new` or `symfony server:start`.

## What the first boot does

`composer.json` in the Docker skeleton is `{}`. The entrypoint replaces it with `symfony/skeleton` (minimal: framework, console, dotenv, runtime, yaml), sets `extra.symfony.docker` to `true`, and removes the install block. That first boot does not install the `webapp` pack.

`webapp` is required next, inside the container: Twig, Asset Mapper, Security, Form, Validator, and `orm-pack`. Do not also require those packs one by one. Later package installs must go through `docker compose exec php composer require` so Flex can update `Dockerfile` and `compose.yaml`.

## Anti-patterns

- `symfony new --webapp`, the Symfony CLI server, or `php -S`
- Host `composer` or `php bin/console` when the Docker stack is the runtime
- A Bootstrap `<link>` or `<script>` CDN, or `bootstrap.min.css` from the importmap. CSS is `assets/styles/_theme.scss` compiled by `sass:build`
- Webpack Encore, Tailwind, API Platform, or EasyAdmin unless the user asks
- Interactive `make:entity`, `make:crud`, `make:user`, `make:auth`
- DQL or SQL built by concatenating request input
- A `<form>` or `<button type="submit">` in Twig. Use a `FormType` and `SubmitType`. Login is the only HTML form
- Behavior in `app.js`, a Twig `<script>`, `onclick`, jQuery, or Alpine. Use a Hotwire Stimulus controller
- A Stimulus controller with more than one responsibility, or one named after a page or feature (`note_controller.js`). One job per `assets/controllers/<job>_controller.js`
- `{{ component('Topbar') }}`, `{% component Topbar %}`, or `{% include 'components/Topbar.html.twig' %}`. Render UX components with `<twig:Topbar .../>`
- `#[IsGranted]` replaced by hiding a link
- Delete as `GET` or as a link
- `addFlash('error')` — use `danger`
- Printing `exception.message` in Twig
- Editing rows in Postgres without a migration file
- A migration whose `getDescription()` is empty
- A second database service beside the one the `orm-pack` recipe added
- Putting the Docker install block back into `frankenphp/docker-entrypoint.sh`
- Committing `.env.local`

## Layer check

| If you are about to… | Put it in |
|---|---|
| Render HTML or a form | Twig |
| Read the request, check CSRF, `persist` / `flush` / `remove`, flash, redirect | Controller |
| Declare fields, `constraints`, and `->add(SubmitType::class)` | Form type |
| Reject an empty title | Form type `constraints` |
| Filter a list | Repository `search()` |
| Apply a business rule, including one used by a single action | Service |

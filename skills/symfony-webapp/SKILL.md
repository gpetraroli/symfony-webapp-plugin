---
name: symfony-webapp
description: Scaffolds and extends Symfony web apps using Twig, Bootstrap 5, Hotwire Stimulus, Doctrine, form login, PHPUnit, and the dunglas/symfony-docker FrankenPHP runtime. Use when creating a new Symfony web application, adding a CRUD page, form login, Doctrine migrations, PHPUnit tests, or Stimulus controllers, or when the user mentions Symfony, Twig, Bootstrap, Stimulus, Hotwire, or Symfony Docker.
---

# Symfony + Twig + Bootstrap

Follow the templates in this skill. Every app uses this structure. Do not copy from another project for patterns. When a Symfony API is not specified here, read the docs for the version `bin/console -V` reports. The pages are in [reference.md](reference.md). These templates win when a doc example uses another layout.

## Architecture

| Layer | Owns | Must not |
|---|---|---|
| **Twig** | Markup, Bootstrap, flashes, form errors, `<twig:Name>` components | Queries, authorization decisions, calculations |
| **Controller** | Route, form `handleRequest`, CSRF, `persist` / `flush` / `remove`, flash, redirect or render | Business rules, DQL, HTML |
| **Form type** | Every form except login: fields, `constraints`, and a `SubmitType` | Persistence, a `<form>` written in Twig |
| **Entity** | Fields, relations, column mapping | `Assert` constraints, `Request`, Twig, queries |
| **Repository** | QueryBuilder queries | HTTP, flashes, rendering, business rules |
| **Service** | Business rules, including a rule used by one action | HTTP, rendering, Twig, queries, `persist` / `flush` / `remove` |

## Invariants

- PHP, Composer, and `bin/console` run with `docker compose exec php`. Create the app from [dunglas/symfony-docker](https://github.com/dunglas/symfony-docker) on `main`. No `symfony new`, and no PHP or Composer on the host.
- Require packages inside the `php` service. If a recipe changes `Dockerfile` or `compose.yaml`, run `docker compose up --build --wait`. PostgreSQL is the `database` service from the `webapp` recipe. Do not add a second database service, and do not point `DATABASE_URL` at the host.
- Layers live under `src/<Feature>/`. After `webapp` is installed, delete the empty `src/Controller`, `src/Entity`, and `src/Repository` directories. Leave `src/Migrations`.
- `Assert` constraints go on the form type. A business rule goes in a service. The service does not call `persist`, `flush`, or `remove`.
- Bootstrap CSS comes from Sass. No Bootstrap CDN, no Webpack Encore, no Tailwind, no API Platform, no EasyAdmin.
- Flashes are `success` and `danger`. Do not print exceptions in Twig.
- `.env` stays the committed Flex template. `.env.local` stays gitignored.
- Tests are PHPUnit, Foundry, and DAMA. The suite uses the `app_test` database, never `app`. Factories live in `tests/<Feature>/Factory`. A resource gets a controller `WebTestCase`. A service gets a unit test with no kernel. Entities, forms, and repositories do not get their own tests.

## New app workflow

Copy this checklist and complete it in order. Commands and security are in [bootstrap.md](bootstrap.md). Layout and assets are in [ui.md](ui.md). Tests are in [tests.md](tests.md).

```
Bootstrap:
- [ ] 1. Confirm the directory name. That is the only question. Do not invent a resource
- [ ] 2. Clone dunglas/symfony-docker, delete its docs/, and start Compose (skeleton only)
- [ ] 3. composer require webapp inside the php service, then rebuild
- [ ] 4. app.scss from ui.md, then sass:build
- [ ] 5. base.html.twig, base_layout.html.twig, Topbar, empty Menu, flashes, form theme
- [ ] 6. Delete the empty Flex directories src/Controller, src/Entity, and src/Repository. Point Doctrine and the service container at src/<Feature>/ (bootstrap.md)
- [ ] 7. Security, unless the user said no login: User, form login, register, logout
- [ ] 8. Home page at src/Home. Do not redirect to a resource
- [ ] 9. README (Compose, migrations, tests, https://localhost)
- [ ] 10. Tests from tests.md. The suite must pass
```

## New feature workflow

Run this only when the user names a resource. Templates are in [feature.md](feature.md). Tests are in [tests.md](tests.md).

1. Entity with fields and relations, under `src/<Feature>/Entity`.
2. Repository methods for list and search.
3. Form types in `src/<Feature>/Form`: the resource form, a search form, and a delete form.
4. A service when the action applies a business rule.
5. Thin controller: index, new, edit, show, delete.
6. Templates: `index`, `_form`, `new`, `edit`, `show`. They extend `base_layout.html.twig` and fill `content`. Add one link to `templates/components/Menu.html.twig`. A CSS rule for this feature goes in `assets/styles/_<feature>.scss`.
7. `doctrine:migrations:diff`, review the SQL, set `getDescription()`, then `doctrine:migrations:migrate`.
8. Factory `tests/<Feature>/Factory/<Entity>Factory.php` from tests.md.
9. Controller `WebTestCase` from tests.md.
10. Service unit test when step 4 added a service. Skip the file otherwise.
11. `docker compose exec php php vendor/bin/phpunit`. Fix failures before finishing.

## Folder layout

```
src/
  <Feature>/
    Controller/
    Entity/
    Form/
    Repository/
    Service/
tests/
  <Feature>/
    Controller/     # WebTestCase for that resource
    Factory/        # Foundry factories
    Service/        # unit test, only when the feature has a service
templates/
  base.html.twig
  base_layout.html.twig
  components/
    Topbar.html.twig
    Menu.html.twig
    _flash_messages.html.twig
  _delete_modal.html.twig
  user/               # login, register
  <feature>/          # index, _form, new, edit, show
assets/
  app.js
  stimulus_bootstrap.js
  controllers/
    <job>_controller.js
  styles/
    app.scss          # @import "bootstrap", then a feature partial when one exists
    _<feature>.scss   # rules owned by that feature
```

`<Feature>` is the singular PascalCase noun (`User`, `Note`). `src/Note/Entity/Note.php` is `App\Note\Entity\Note`. Create a layer directory only when the feature has a class for it.

## Screen map

| Screen | Route | Body |
|---|---|---|
| Index | `GET /<entities>` | Search form, table or empty state, link to new |
| New | `GET\|POST /<entities>/new` | Shared `_form` |
| Edit | `GET\|POST /<entities>/{id}/edit` | Same `_form` with the entity |
| Show | `GET /<entities>/{id}` | Fields, edit link, delete button opening the modal |
| Delete | `POST /<entities>/{id}/delete` | `DeleteType` inside the modal, then redirect to index |
| Login | `GET\|POST /login` | Email, password, CSRF |
| Register | `GET\|POST /register` | Email, repeated password, then redirect to login |

## Additional resources

- Project creation, security, README: [bootstrap.md](bootstrap.md)
- Layout, Bootstrap, flashes, delete modal: [ui.md](ui.md)
- CRUD templates: [feature.md](feature.md)
- PHPUnit, Foundry, DAMA: [tests.md](tests.md)
- Commands and Symfony docs: [reference.md](reference.md)

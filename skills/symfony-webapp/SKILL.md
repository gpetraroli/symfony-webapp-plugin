---
name: symfony-twig-bootstrap
description: Scaffolds and extends Symfony web apps using Twig, Bootstrap 5, Hotwire Stimulus, Doctrine, form login, and the dunglas/symfony-docker FrankenPHP runtime. Use when creating a new Symfony web application, adding a CRUD page, form login, Doctrine migrations, or Stimulus controllers, or when the user mentions Symfony, Twig, Bootstrap, Stimulus, Hotwire, or Symfony Docker.
---

# Symfony + Twig + Bootstrap

Follow the templates in this skill. Every app uses this structure. Do not copy from another project for patterns.

New apps are created from [dunglas/symfony-docker](https://github.com/dunglas/symfony-docker) (`main`). Do not run `symfony new`, `symfony server:start`, or the PHP built-in server. Do not install PHP or Composer on the host for this project.

## Architecture

```
Twig (UI)
    ↕  form view, flashes
Controller (HTTP)
    ↕  constructor injection
Form type
    ↕
Service (business rules; the controller persists)
    ↕
Repository (queries)
    ↕
Entity (fields, relations, Assert)
    ↕
Doctrine
```

| Layer | Owns | Must not |
|---|---|---|
| **Twig** | Markup, Bootstrap, flashes, form errors, `<twig:Name>` components | Queries, authorization decisions, calculations |
| **Controller** | Route, form `handleRequest`, CSRF, `persist` / `flush` / `remove`, flash, redirect or render | Business rules, DQL, HTML |
| **Form type** | Every form except login: fields, options, and a `SubmitType` | Validation of entity fields, persistence, a `<form>` written in Twig |
| **Entity** | Fields, relations, `Assert` constraints | `Request`, Twig, queries |
| **Repository** | QueryBuilder queries | HTTP, flashes, rendering, business rules |
| **Service** | Business rules, including a rule used by one action | HTTP, rendering, Twig, queries, `persist` / `flush` / `remove` |

Controllers stay thin. A business rule goes in a service, then the controller calls `persist`, `flush`, or `remove`. **Validation of entity fields lives on the entity.**

## New app workflow

Copy this checklist and complete it in order:

```
Bootstrap:
- [ ] 1. Confirm the directory name. Do not invent a resource
- [ ] 2. Clone dunglas/symfony-docker and start Compose (HTTP, skeleton only)
- [ ] 3. composer require webapp inside the php service, then rebuild
- [ ] 4. Sass theme: symfonycasts/sass-bundle, twbs/bootstrap, assets/styles/_theme.scss, sass:build
- [ ] 5. base.html.twig, base_layout.html.twig, Topbar, empty Menu, flashes, form theme
- [ ] 6. Delete the empty Flex directories src/Controller, src/Entity, and src/Repository
- [ ] 7. Security, unless the user said no login: User, form login, register, logout
- [ ] 8. Home page. Do not redirect to a resource
- [ ] 9. README (Compose, migrations, https://localhost)
```

**Step 1 — Confirm the directory name** if the user did not give one. That is the only question for a bootstrap. Do not ask for a resource and do not choose one. `Note` in [feature.md](feature.md) is an example for a later request. Do not create `src/Note`, `templates/note`, or a Notes menu link during bootstrap.

Login is part of bootstrap unless the user said they do not want it. The home page is `src/Home`, not a CRUD resource.

The PHP namespace stays `App`. The Docker entrypoint installs `symfony/skeleton` with that default. Do not set `SYMFONY_VERSION` unless the user asked for a specific Symfony version.

**Step 2 — Scaffold** from [dunglas/symfony-docker](https://github.com/dunglas/symfony-docker/tree/main). Commands and the first-boot installer: [bootstrap.md](bootstrap.md).

**Step 3 — Webapp pack.** First boot stops at the minimal skeleton. Require `webapp` inside the container so Flex can patch `Dockerfile` and `compose.yaml` (PostgreSQL, `pdo_pgsql`). Then rebuild. Details: [bootstrap.md](bootstrap.md).

**Step 4 — Bootstrap 5 from Sass.** Require `symfonycasts/sass-bundle` and `twbs/bootstrap` inside the container. Copy `assets/styles/_theme.scss` and `assets/styles/app.scss` from [ui.md](ui.md). Keep the webapp pack's Hotwire Stimulus bootstrap. `import 'bootstrap'` in `app.js` only enables Bootstrap's `data-bs-*` widgets. Any other browser behavior is its own Stimulus controller with one responsibility. Form theme `bootstrap_5_layout.html.twig`. No Bootstrap CDN, no Webpack Encore.

**Step 5** — copy the layout from [ui.md](ui.md). The Menu has no feature links yet.

**Step 6** — `webapp` leaves empty `src/Controller`, `src/Entity`, and `src/Repository` directories (each is only a `.gitignore`). Delete them. Leave `src/Migrations`.

**Step 7** — when login is included, follow the User, login, and register templates in [bootstrap.md](bootstrap.md). Then `doctrine:migrations:diff` and migrate for that user table.

**Step 8** — write `README.md` from the template in [bootstrap.md](bootstrap.md).

## New feature workflow

Run this only when the user names a resource. Do not run it to finish a bootstrap.

1. Entity with `Assert` constraints and relations, under `src/<Feature>/Entity`
2. Repository methods for list and search. Parameters are bound. No concatenated SQL
3. Form types in `src/<Feature>/Form` (`NoteType`, `NoteSearchType`, `DeleteType`), each ending with `->add(SubmitType::class)`
4. Service for any business rule, even when only one action calls it. The service does not persist
5. Thin controller: index, new, edit, show, delete. Class `#[Route]` sets `name: 'app_note'`; each method sets only the suffix (`name: '_new'`). Symfony concatenates them (`app_note_new`). Call the service, then `persist` / `flush` / `remove`. Every POST ends in a redirect. Delete is a POST form with CSRF
6. Templates: `index`, `_form`, `new`, `edit`, `show`. They extend `base_layout.html.twig` and fill `content`. Add one link to `templates/components/Menu.html.twig` for this feature
7. `doctrine:migrations:diff`, review the SQL, then `doctrine:migrations:migrate`

Show, edit, and delete load the entity with the argument resolver (`Note $note`). List and search go through the repository.

Full templates: [feature.md](feature.md).

Skip interactive `make:*` commands. They stall. `doctrine:migrations:diff` is the schema CLI step.

## Folder layout

```
src/
  <Feature>/
    Controller/
    Entity/
    Form/
    Repository/
    Service/          # business rules only; the controller persists
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
  app.js                      # stimulus_bootstrap.js + import 'bootstrap'
  stimulus_bootstrap.js       # Flex file; do not rewrite it
  controllers/
    <job>_controller.js       # one responsibility per file
  styles/
    app.scss
    _theme.scss
```

`<Feature>` is the singular PascalCase noun (`User`, `Note`). A class lives in that feature's layer directory: `src/Note/Entity/Note.php` is `App\Note\Entity\Note`. Create a layer directory only when the feature has a class for it. Do not create `src/Controller`, `src/Entity`, `src/Form`, `src/Repository`, or `src/Service`. After `webapp` is installed, delete those empty directories if Flex created them. Leave `src/Migrations`.

After `webapp` is installed, point Doctrine and the service container at this layout. In `config/packages/doctrine.yaml`, set the `App` mapping `dir` to `%kernel.project_dir%/src` and `prefix` to `App`. In `config/services.yaml`, replace the `../src/Entity/` exclude with `../src/*/Entity/`.

Leave the Docker files in place. Flex may edit `Dockerfile` and `compose.yaml` when a package is required; rebuild after that. Do not hand-write the database service.

## Hard rules

1. **Runtime** — create the project only from `https://github.com/dunglas/symfony-docker` (`main`). Run PHP, Composer, and `bin/console` with `docker compose exec php`. The site is `https://localhost` (accept the local Caddy certificate).
2. **Packages** — `composer require` inside the `php` service so Flex recipes see `extra.symfony.docker`. If `Dockerfile` or `compose.yaml` changes, `docker compose up --build --wait`.
3. **Database** — PostgreSQL is the Compose service added by the `orm-pack` recipe (`webapp` pulls it in). `DATABASE_URL` in `compose.yaml` points at the `database` host. Do not point the app at a host Postgres.
4. **Schema** — change the schema by editing entities, then `doctrine:migrations:diff`. Review the generated SQL. Apply with `doctrine:migrations:migrate --no-interaction`. Do not alter tables only in the database.
5. **Validation** — `Assert` constraints go on the entity. The controller checks `$form->isSubmitted() && $form->isValid()` and nothing else. Constraints on an unmapped field (plain password) stay on the form.
6. **Authorization** — `#[IsGranted]` or a voter. Hiding a button in Twig is not authorization. Login and register are `PUBLIC_ACCESS`. Everything else is `ROLE_USER` when login is enabled.
7. **Forms** — every form is a `FormType` rendered with `form_start` / `form_widget` / `form_end`. Login is the exception: it stays the firewall HTML form (`_username`, `_password`, `_csrf_token`). Search is `NoteSearchType` (GET, no CSRF, field name `query`). Delete is that feature's `DeleteType`; the form component checks its CSRF token.
8. **Flashes** — Bootstrap alert names: `success` and `danger`.
9. **Lists** — the search form's field is named `query`. Read it with `$searchForm->get('query')->getData()`. Pass the form to the template. Add an empty state.
10. **Errors** — generic messages for the user. Do not print exceptions in Twig.
11. **Assets** — Bootstrap CSS is Sass. `assets/styles/_theme.scss` sets the variables, imports Bootstrap from `vendor/twbs/bootstrap`, then adds the custom selectors. `app.scss` imports that theme. Compile with `sass:build`. Bootstrap's widget script is `import 'bootstrap'` in `app.js` (dropdowns, modals, dismiss). Do not import `bootstrap.min.css`. Do not add a Bootstrap CDN stylesheet. Do not add Webpack Encore, Tailwind, API Platform, or EasyAdmin.
12. **Secrets** — `.env` is the committed Flex template. `.env.local` stays gitignored. Do not commit real production passwords. Local Compose defaults from the skeleton are fine for local development.
13. **Thin controllers** — business rules go in a service even when a single action uses them. Do not put that rule in Twig or in a repository. `persist`, `flush`, and `remove` belong to the controller: call the service, then write through `EntityManager`. Queries stay in the repository.
14. **Features** — layers live inside the feature: `src/Note/Controller`, `src/Note/Entity`, `src/Note/Form`, `src/Note/Repository`, `src/Note/Service`. Do not place those layers directly under `src/`. After `webapp` is installed, delete the empty `src/Controller`, `src/Entity`, and `src/Repository` directories Flex created. Leave `src/Migrations`. A resource from [feature.md](feature.md) is created only when the user names it. Do not create the Note example during bootstrap.
15. **Twig components** — render an anonymous UX component with the `<twig:Name>` tag: `<twig:Topbar class="topbar"/>`, `<twig:Menu/>`. `templates/components/Name.html.twig` is `<twig:Name>`. Do not use `{{ component() }}`, `{% component %}`, or `{% include %}` for those templates. A file whose name starts with `_` stays an include.
16. **JavaScript** — browser behavior is Hotwire Stimulus (`import { Controller } from '@hotwired/stimulus'`). The webapp pack starts it from `assets/stimulus_bootstrap.js`. Each file `assets/controllers/<job>_controller.js` has one responsibility and is named after that job (`password_visibility_controller.js` → `data-controller="password-visibility"`). A second behavior is a second controller on the same element. Do not put behavior in `app.js`, in `stimulus_bootstrap.js`, in a Twig `<script>`, or in `onclick`. Do not write a controller for a whole page or feature. Delete the recipe's `hello_controller.js`. Leave `csrf_protection_controller.js`.

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
- Commands and anti-patterns: [reference.md](reference.md)

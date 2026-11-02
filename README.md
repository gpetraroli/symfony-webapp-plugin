# Symfony Webapp Plugin

Agent skill for scaffolding Symfony web apps with Twig, Bootstrap 5, and Doctrine. New projects start from [dunglas/symfony-docker](https://github.com/dunglas/symfony-docker): FrankenPHP, Caddy, and Docker Compose. The skeleton install happens inside the container on first boot.

This package is an [Agent Plugin](https://agent-plugins.org/) (Cursor, Copilot, VS Code, …) and a [Claude Code](https://code.claude.com/docs/en/plugins) plugin. The skill lives in `skills/symfony-webapp/`.

Every PHP, Composer, and `bin/console` command runs inside the `php` service (`docker compose exec php …`). The app is served at `https://localhost`.

Typical tasks:

- Create a new Symfony web app from the Symfony Docker skeleton, with the layout and form login
- Add a resource as entity → repository → form → controller → Twig when the user names one
- Generate and run Doctrine migrations inside the container

After installing, invoke it in chat with `/symfony-twig-bootstrap`.

## Cursor

Copy this repo into Cursor’s local plugins folder, then run **Developer: Reload Window**:

```sh
cp -a /absolute/path/to/symfony_webapp_plugin ~/.cursor/plugins/local/symfony-webapp-plugin
```

Do not symlink. Cursor skips those links, so `/symfony-twig-bootstrap` never appears in chat. After reload, the plugin should show up under Customize → Plugins.

Re-copy after you change the plugin, then reload again.

## Claude Code

From this directory:

```sh
claude plugin validate .
claude --plugin-dir /absolute/path/to/symfony_webapp_plugin
```

# Shared UI templates

Write these into every app. Keep `{{ importmap('app') }}` for JavaScript. Bootstrap CSS is compiled from `assets/styles/app.scss`.

## config/packages/twig.yaml

Keep the keys Flex generated. Set the form theme:

```yaml
twig:
    form_themes: ['bootstrap_5_layout.html.twig']
```

## assets/styles/app.scss

Import Bootstrap and nothing else. Do not add a theme or layout partial, and do not add a selector that redefines a Bootstrap class. `@import "bootstrap"` resolves through the Sass load path `vendor/twbs/bootstrap/scss`. A feature rule goes in `assets/styles/_<feature>.scss` and is imported at the end of this file. Skip that file when Bootstrap utilities cover the screen.

```scss
@import "bootstrap";
```

## assets/app.js

Keep the Stimulus import the webapp pack added, then import Bootstrap's JavaScript so `data-bs-*` widgets work.

```javascript
import './stimulus_bootstrap.js';
import 'bootstrap';
```

`assets/stimulus_bootstrap.js` is the Asset Mapper file Flex wrote (`startStimulusApp()` from `@symfony/stimulus-bundle`). Leave it. It registers every `assets/controllers/*_controller.js`. Do not replace it with the Webpack Encore `webpackContext` version.

Delete `assets/styles/app.css`. The stylesheet is `app.scss`, linked from `base_layout.html.twig`. Delete `assets/controllers/hello_controller.js`. Leave `csrf_protection_controller.js`; its only job is CSRF.

## Stimulus controllers

Custom JavaScript is a Hotwire Stimulus controller. Add one only when a screen needs that behavior.

Each controller has one responsibility. Name the file after that job: `assets/controllers/password_visibility_controller.js` is only showing or hiding a password. The bundle turns the filename into the identifier `password-visibility`. A second job is a second file, then both identifiers on the element (`data-controller="password-visibility submit-lock"`). Do not create `note_controller.js` (or any other controller) that handles a whole page or feature.

```javascript
import { Controller } from '@hotwired/stimulus';

export default class extends Controller {
    static targets = ['input'];

    toggle() {
        this.inputTarget.type = this.inputTarget.type === 'password' ? 'text' : 'password';
    }
}
```

```twig
<div data-controller="password-visibility">
    <input type="password" data-password-visibility-target="input">
    <button type="button" data-action="password-visibility#toggle">Show</button>
</div>
```

Targets, values, and actions belong to that one job. Do not register these controllers by hand in `stimulus_bootstrap.js`. `app.register()` is only for a third-party controller that does not live in `assets/controllers/`. No `<script>` blocks in Twig, no `onclick`, no jQuery, no Alpine, no second Stimulus package. Bootstrap widgets stay on `data-bs-toggle` and `data-bs-dismiss`; do not reimplement them in a controller.

## templates/base.html.twig

```twig
<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8">
        <meta name="viewport"
              content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
        <meta http-equiv="X-UA-Compatible" content="ie=edge">
        <title>{% block title %}Symfony template{% endblock %}</title>
        <link rel="icon" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 128 128%22><text y=%221.2em%22 font-size=%2296%22>⚫️</text><text y=%221.3em%22 x=%220.2em%22 font-size=%2276%22 fill=%22%23fff%22>sf</text></svg>">
        {% block stylesheets %}
        {% endblock %}

        {% block javascripts %}
            {% block importmap %}{{ importmap('app') }}{% endblock %}
        {% endblock %}
    </head>
    <body>
        {% include 'components/_flash_messages.html.twig' %}
        {% block body %}{% endblock %}
    </body>
</html>
```

## templates/base_layout.html.twig

Default page layout. Feature and account templates extend this and fill `content`. The stylesheets block loads the Sass build.

```twig
{% extends 'base.html.twig' %}

{% block stylesheets %}
    <link rel="stylesheet" href="{{ asset('styles/app.scss') }}">
{% endblock %}

{% block body %}
    <div class="d-flex flex-column">
        <div>
            <twig:Topbar class="topbar"/>

            <div class="d-flex">
                <nav class="sidenav navbar bg-light p-1 align-items-start overflow-scroll d-none d-lg-block">
                    <twig:Menu/>
                </nav>

                <div class="main-content p-2 overflow-scroll w-100">
                    {% block content %}{% endblock %}
                </div>
            </div>

        </div>
    </div>
{% endblock %}
```

`symfony/ux-twig-component` provides anonymous components. Require it inside the container and leave the Flex recipe config in place. Render them with the `<twig:Name>` tag, as in `<twig:Topbar class="topbar"/>` and `<twig:Menu/>`. `templates/components/Name.html.twig` is `<twig:Name>`. Do not use `{{ component() }}`, `{% component %}`, or `{% include %}` for these templates. A partial whose name starts with `_` (`_flash_messages.html.twig`) stays `{% include %}`.

## templates/components/Topbar.html.twig

```twig
<nav{{ attributes.defaults({class: 'navbar navbar-expand-lg navbar-dark bg-black'}) }}>
    <div class="container-fluid">
        <a class="navbar-brand" href="{{ path('app_home') }}">{{ app_name|default('App') }}</a>
        <ul class="navbar-nav ms-auto">
            {% if app.user %}
                <li class="nav-item">
                    <span class="navbar-text me-3">{{ app.user.userIdentifier }}</span>
                </li>
                <li class="nav-item">
                    <a class="nav-link" href="{{ path('app_logout') }}">Log out</a>
                </li>
            {% else %}
                <li class="nav-item">
                    <a class="nav-link" href="{{ path('app_login') }}">Log in</a>
                </li>
                <li class="nav-item">
                    <a class="nav-link" href="{{ path('app_register') }}">Register</a>
                </li>
            {% endif %}
        </ul>
    </div>
</nav>
```

When login is disabled, omit the `app.user` branch and the login/register links.

Set the brand in `config/packages/twig.yaml`:

```yaml
twig:
    globals:
        app_name: '<AppTitle>'
```

## templates/components/Menu.html.twig

One link per feature that exists. Bootstrap leaves the list empty. Add a link only when that feature is created.

```twig
<ul{{ attributes.defaults({class: 'nav flex-column w-100'}) }}>
</ul>
```

## templates/components/_flash_messages.html.twig

```twig
{% for label, messages in app.flashes %}
    {% for message in messages %}
        <div class="alert alert-{{ label }} alert-dismissible fade show" role="alert">
            {{ message }}
            <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
        </div>
    {% endfor %}
{% endfor %}
```

## templates/_delete_modal.html.twig

Include from the show template. The controller passes `delete_form` (`App\<Feature>\Form\DeleteType`) with `action` set to the delete route. The submit button is the form's `SubmitType`.

```twig
<div class="modal fade" id="delete-modal" tabindex="-1" aria-labelledby="delete-modal-title" aria-hidden="true">
    <div class="modal-dialog">
        <div class="modal-content">
            <div class="modal-header">
                <h2 class="modal-title h5" id="delete-modal-title">Delete</h2>
                <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
            </div>
            <div class="modal-body">
                This cannot be undone.
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Cancel</button>
                {{ form_start(delete_form) }}
                    {{ form_widget(delete_form) }}
                {{ form_end(delete_form) }}
            </div>
        </div>
    </div>
</div>
```

Open it with:

```twig
<button type="button" class="btn btn-outline-danger" data-bs-toggle="modal" data-bs-target="#delete-modal">
    Delete
</button>
```

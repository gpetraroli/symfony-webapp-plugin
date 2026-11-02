# Adding a CRUD resource

Use this only when the user has named a resource. `Note` below is the example. Rename consistently (`note`, `notes`, `Note`, `NoteRepository`, `NoteType`). Example: table `note`, route prefix `/notes`, under `src/Note/`.

Generate the migration last. See section 7. Skip interactive `make:entity`, `make:crud`, `make:user`, and `make:auth`. They stall.

```sh
docker compose exec php php bin/console doctrine:migrations:diff --no-interaction
```

## 1. Entity — `src/Note/Entity/Note.php`

Fields, relations, and column mapping.

```php
<?php

declare(strict_types=1);

namespace App\Note\Entity;

use App\Note\Repository\NoteRepository;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: NoteRepository::class)]
class Note
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 255)]
    private string $title = '';

    #[ORM\Column(type: 'text', nullable: true)]
    private ?string $body = null;

    #[ORM\Column]
    private \DateTimeImmutable $createdAt;

    public function __construct()
    {
        $this->createdAt = new \DateTimeImmutable();
    }

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getTitle(): string
    {
        return $this->title;
    }

    public function setTitle(string $title): void
    {
        $this->title = $title;
    }

    public function getBody(): ?string
    {
        return $this->body;
    }

    public function setBody(?string $body): void
    {
        $this->body = $body;
    }

    public function getCreatedAt(): \DateTimeImmutable
    {
        return $this->createdAt;
    }
}
```

Relations: map them with `ORM` attributes on the entity. Load them in a repository method. Do not query from Twig.

## 2. Repository — `src/Note/Repository/NoteRepository.php`

Bind every search value. Do not concatenate it into DQL.

```php
<?php

declare(strict_types=1);

namespace App\Note\Repository;

use App\Note\Entity\Note;
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\ManagerRegistry;

/** @extends ServiceEntityRepository<Note> */
class NoteRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, Note::class);
    }

    /** @return list<Note> */
    public function search(string $query): array
    {
        $builder = $this->createQueryBuilder('note')
            ->orderBy('note.createdAt', 'DESC');

        if ($query !== '') {
            $builder
                ->andWhere('LOWER(note.title) LIKE :query')
                ->setParameter('query', '%'.mb_strtolower($query).'%');
        }

        return $builder->getQuery()->getResult();
    }
}
```

Show, edit, and delete do not need a finder. The controller type-hints `Note $note` and Symfony loads it.

## 3. Form — `src/Note/Form/NoteType.php`

Constraints live on each field's `constraints` option. A constraint on the whole object, such as `UniqueEntity`, goes in `configureOptions`. Every form type ends with a `SubmitType`, so the template has no submit button.

```php
<?php

declare(strict_types=1);

namespace App\Note\Form;

use App\Note\Entity\Note;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\Extension\Core\Type\TextareaType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;
use Symfony\Component\Validator\Constraints\Length;
use Symfony\Component\Validator\Constraints\NotBlank;

class NoteType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('title', TextType::class, [
                'constraints' => [
                    new NotBlank(),
                    new Length(max: 255),
                ],
            ])
            ->add('body', TextareaType::class, [
                'required' => false,
                'constraints' => [
                    new Length(max: 10000),
                ],
            ])
            ->add('save', SubmitType::class, [
                'label' => 'Save',
            ]);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'data_class' => Note::class,
        ]);
    }
}
```

### src/Note/Form/NoteSearchType.php

Search is a form type too. Method is GET, CSRF is off (the query string must stay free of a token), and the block prefix is empty so the parameter name is `query`.

```php
<?php

declare(strict_types=1);

namespace App\Note\Form;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\SearchType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class NoteSearchType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('query', SearchType::class, [
                'required' => false,
                'label' => 'Search',
                'label_attr' => ['class' => 'visually-hidden'],
                'attr' => ['placeholder' => 'Search by title'],
                'row_attr' => ['class' => 'col-sm-8 col-md-6'],
            ])
            ->add('search', SubmitType::class, [
                'label' => 'Search',
                'attr' => ['class' => 'btn-outline-secondary'],
                'row_attr' => ['class' => 'col-auto'],
            ]);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'method' => 'GET',
            'csrf_protection' => false,
            'attr' => ['class' => 'row g-2 mb-3'],
        ]);
    }

    public function getBlockPrefix(): string
    {
        return '';
    }
}
```

### src/Note/Form/DeleteType.php

The feature's delete form. The show action sets `action` to that entity's delete route. The form component checks CSRF.

```php
<?php

declare(strict_types=1);

namespace App\Note\Form;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\FormBuilderInterface;

class DeleteType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder->add('delete', SubmitType::class, [
            'label' => 'Delete',
            'attr' => ['class' => 'btn-danger'],
        ]);
    }
}
```

## 4. Service — `src/Note/Service/NoteService.php`

Add a service when the action applies a business rule (a calculation, a status change, a side effect), even if only one action calls it. Symfony autowires classes under `src/`.

The service changes the entity or returns a result. It does not take a `Request`. Storing a valid form with no extra rule needs no service; the controller persists it.

```php
<?php

declare(strict_types=1);

namespace App\Note\Service;

use App\Note\Entity\Note;

class NoteService
{
    public function publish(Note $note): void
    {
        $note->setPublishedAt(new \DateTimeImmutable());
    }
}
```

The controller calls the service, then writes:

```php
$notes->publish($note);
$entityManager->flush();
```

## 5. Controller — `src/Note/Controller/NoteController.php`

`#[IsGranted('ROLE_USER')]` when login is enabled. Hiding a button in Twig is not authorization. A failed CSRF check denies access.

```php
<?php

declare(strict_types=1);

namespace App\Note\Controller;

use App\Note\Entity\Note;
use App\Note\Form\DeleteType;
use App\Note\Form\NoteSearchType;
use App\Note\Form\NoteType;
use App\Note\Repository\NoteRepository;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Security\Http\Attribute\IsGranted;

#[Route('/notes', name: 'app_note')]
#[IsGranted('ROLE_USER')]
class NoteController extends AbstractController
{
    #[Route('/', name: '_index', methods: ['GET'])]
    public function index(Request $request, NoteRepository $notes): Response
    {
        $searchForm = $this->createForm(NoteSearchType::class);
        $searchForm->handleRequest($request);
        $query = $searchForm->get('query')->getData() ?? '';

        return $this->render('note/index.html.twig', [
            'notes' => $notes->search($query),
            'search_form' => $searchForm,
        ]);
    }

    #[Route('/new', name: '_new', methods: ['GET', 'POST'])]
    public function new(Request $request, EntityManagerInterface $entityManager): Response
    {
        $note = new Note();
        $form = $this->createForm(NoteType::class, $note);
        $form->handleRequest($request);

        if ($form->isSubmitted() && $form->isValid()) {
            $entityManager->persist($note);
            $entityManager->flush();
            $this->addFlash('success', 'Note created.');

            return $this->redirectToRoute('app_note_show', ['id' => $note->getId()]);
        }

        return $this->render('note/new.html.twig', [
            'form' => $form,
        ]);
    }

    #[Route('/{id}', name: '_show', methods: ['GET'], requirements: ['id' => '\d+'])]
    public function show(Note $note): Response
    {
        $deleteForm = $this->createForm(DeleteType::class, null, [
            'action' => $this->generateUrl('app_note_delete', ['id' => $note->getId()]),
        ]);

        return $this->render('note/show.html.twig', [
            'note' => $note,
            'delete_form' => $deleteForm,
        ]);
    }

    #[Route('/{id}/edit', name: '_edit', methods: ['GET', 'POST'], requirements: ['id' => '\d+'])]
    public function edit(Request $request, Note $note, EntityManagerInterface $entityManager): Response
    {
        $form = $this->createForm(NoteType::class, $note);
        $form->handleRequest($request);

        if ($form->isSubmitted() && $form->isValid()) {
            $entityManager->flush();
            $this->addFlash('success', 'Note updated.');

            return $this->redirectToRoute('app_note_show', ['id' => $note->getId()]);
        }

        return $this->render('note/edit.html.twig', [
            'note' => $note,
            'form' => $form,
        ]);
    }

    #[Route('/{id}/delete', name: '_delete', methods: ['POST'], requirements: ['id' => '\d+'])]
    public function delete(Request $request, Note $note, EntityManagerInterface $entityManager): Response
    {
        $deleteForm = $this->createForm(DeleteType::class);
        $deleteForm->handleRequest($request);

        if ($deleteForm->isSubmitted() && $deleteForm->isValid()) {
            $entityManager->remove($note);
            $entityManager->flush();
            $this->addFlash('success', 'Note deleted.');
        } else {
            $this->addFlash('danger', 'The note could not be deleted.');
        }

        return $this->redirectToRoute('app_note_index');
    }
}
```

Declare `/{id}/edit` and `/{id}/delete` as their own routes. Keep the `requirements` so `new` is not captured as an id. The class `name` is the prefix (`app_note`); each method `name` is only the suffix (`_new`). Symfony concatenates them into `app_note_new`. Use that full name in `redirectToRoute` and `path()`.

## 6. Templates

### templates/note/_form.html.twig

```twig
{{ form_start(form) }}
    {{ form_widget(form) }}
    <a class="btn btn-link" href="{{ path('app_note_index') }}">Cancel</a>
{{ form_end(form) }}
```

### templates/note/index.html.twig

```twig
{% extends 'base_layout.html.twig' %}

{% block title %}Notes{% endblock %}

{% block content %}
    <div class="d-flex justify-content-between align-items-center mb-3">
        <h1 class="h3 mb-0">Notes</h1>
        <a class="btn btn-primary" href="{{ path('app_note_new') }}">New note</a>
    </div>

    {{ form_start(search_form) }}
        {{ form_widget(search_form) }}
    {{ form_end(search_form) }}

    {% if notes is empty %}
        <p class="text-secondary">No notes yet.</p>
    {% else %}
        <div class="table-responsive">
            <table class="table align-middle">
                <thead>
                    <tr>
                        <th>Title</th>
                        <th>Created</th>
                        <th></th>
                    </tr>
                </thead>
                <tbody>
                    {% for note in notes %}
                        <tr>
                            <td>{{ note.title }}</td>
                            <td>{{ note.createdAt|date('Y-m-d H:i') }}</td>
                            <td class="text-end">
                                <a href="{{ path('app_note_show', {id: note.id}) }}">View</a>
                            </td>
                        </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    {% endif %}
{% endblock %}
```

### templates/note/new.html.twig

```twig
{% extends 'base_layout.html.twig' %}

{% block title %}New note{% endblock %}

{% block content %}
    <h1 class="h3 mb-3">New note</h1>
    {% include 'note/_form.html.twig' %}
{% endblock %}
```

### templates/note/edit.html.twig

```twig
{% extends 'base_layout.html.twig' %}

{% block title %}Edit {{ note.title }}{% endblock %}

{% block content %}
    <h1 class="h3 mb-3">Edit note</h1>
    {% include 'note/_form.html.twig' %}
{% endblock %}
```

### templates/note/show.html.twig

```twig
{% extends 'base_layout.html.twig' %}

{% block title %}{{ note.title }}{% endblock %}

{% block content %}
    <h1 class="h3">{{ note.title }}</h1>
    <p class="text-secondary">{{ note.createdAt|date('Y-m-d H:i') }}</p>
    <div class="mb-4">{{ note.body|default('')|nl2br }}</div>
    <a class="btn btn-primary" href="{{ path('app_note_edit', {id: note.id}) }}">Edit</a>
    <button type="button" class="btn btn-outline-danger" data-bs-toggle="modal" data-bs-target="#delete-modal">Delete</button>
    <a class="btn btn-link" href="{{ path('app_note_index') }}">Back</a>

    {% include '_delete_modal.html.twig' %}
{% endblock %}
```

`delete_form` is created in `show()`. The modal renders that form. Do not write a `<form>` in Twig.

Add one item to `templates/components/Menu.html.twig`:

```twig
<li class="nav-item">
    <a class="nav-link" href="{{ path('app_note_index') }}">Notes</a>
</li>
```

## 7. Migration

Review the generated SQL. It should create the resource table and, when this migration also introduces users, `app_user`. Set `getDescription()` to a sentence about that change. Do not leave it as `''`. Then migrate. Do not apply schema changes with raw SQL against the database.

```sh
docker compose exec php php bin/console doctrine:migrations:migrate --no-interaction
```

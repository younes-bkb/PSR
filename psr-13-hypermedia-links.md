# PSR-13 : Hypermedia Links

## L'Objectif : Standardiser les liens hypermédia dans les API

Dans une API RESTful, le concept **HATEOAS** (*Hypermedia as the Engine of Application State*) suggère qu'une réponse d'API ne doit pas seulement contenir les données, mais aussi des **liens** qui indiquent au client les prochaines actions possibles.

Par exemple, quand on récupère les détails d'une commande, la réponse pourrait contenir :
-   Les données de la commande.
-   Un lien pour voir le client associé (`/clients/123`).
-   Un lien pour annuler cette commande (`/commandes/456/annuler`).
-   Un lien pour se référer à la ressource elle-même (`self`).

Le problème était que chaque API représentait ces liens à sa manière (`_links`, `links`, `relations`...). La PSR-13 a pour but de **standardiser la manière dont on représente et manipule ces liens hypermédia** en PHP.

**Le bénéfice :** Permettre à des bibliothèques (sérialiseurs, clients API) de comprendre et de manipuler les liens d'une API de manière générique, sans connaître son format spécifique.

## Les Interfaces Principales

La PSR-13 définit un ensemble d'interfaces pour modéliser ces liens de manière structurée.

-   **`LinkInterface`** : Représente un seul lien hypermédia. C'est l'objet central. Il contient :
    -   `getHref()`: L'URL (la cible) du lien. C'est la seule propriété obligatoire.
    -   `isTemplated()`: Un booléen qui indique si l'URL est un "template" (ex: `/users/{id}`).
    -   `getRels()`: Un tableau de relations (les "rels") qui décrivent la nature du lien (ex: `self`, `next`, `edit`).
    -   `getAttributes()`: Un tableau de métadonnées additionnelles sur le lien (ex: `type`, `hreflang`).

-   **`LinkProviderInterface`** : Un objet qui **contient** une collection de liens. Typiquement, votre objet de réponse d'API (ex: un objet `Order`, `User`) implémenterait cette interface.
    -   `getLinks()`: Retourne un tableau d'objets `LinkInterface`.
    -   `getLinksByRel($rel)`: Retourne les liens qui ont une relation (`rel`) spécifique.

-   **`EvolvableLinkProviderInterface`** : Hérite de `LinkProviderInterface` et ajoute des méthodes pour **modifier** la collection de liens de manière immuable.
    -   `withLink(LinkInterface $link)`: Retourne un clone du fournisseur avec le nouveau lien ajouté.
    -   `withoutLink(LinkInterface $link)`: Retourne un clone du fournisseur avec le lien retiré.

## Cas d'Usage : Représenter une ressource API

### Scénario
Nous voulons modéliser une ressource "Article de blog" qui sera retournée par notre API. Cette ressource doit contenir des liens vers l'auteur et vers elle-même.

**1. Avoir une implémentation de la PSR-13**
Comme pour les autres PSR d'interfaces, nous avons besoin d'une bibliothèque qui fournit les classes concrètes. Nous utiliserons `fig/link-util`.

```bash
composer require fig/link-util
```

**2. Créer notre objet Ressource**
Notre classe `ArticleResource` va implémenter `EvolvableLinkProviderInterface` pour pouvoir contenir et manipuler des liens.

```php
<?php
require 'vendor/autoload.php';

use Fig\Link\EvolvableLinkProviderTrait;
use Fig\Link\Link;
use Psr\Link\EvolvableLinkProviderInterface;

// Notre objet métier qui implémente l'interface PSR-13
class ArticleResource implements EvolvableLinkProviderInterface
{
    // Utilise un trait de la bibliothèque pour implémenter facilement
    // les méthodes de gestion des liens (withLink, getLinks, etc.)
    use EvolvableLinkProviderTrait;

    public int $id;
    public string $title;
    public string $content;
    public int $authorId;

    public function __construct(int $id, string $title, string $content, int $authorId)
    {
        $this->id = $id;
        $this->title = $title;
        $this->content = $content;
        $this->authorId = $authorId;
    }
}
```

**3. Créer et manipuler les liens**
Le code qui construit la réponse de l'API va créer la ressource et y attacher les liens pertinents.
```php
<?php
// Créons une instance de notre ressource
$article = new ArticleResource(
    101,
    'Titre de l\'article',
    'Contenu de l\'article...',
    42
);

// Créons les liens à y attacher
$selfLink = new Link('self', '/articles/101');
$authorLink = new Link('author', '/users/42');

// On utilise les méthodes de l'interface pour ajouter les liens (de manière immuable)
$articleWithLinks = $article
    ->withLink($selfLink)
    ->withLink($authorLink);

// --- Côté "consommateur" de l'objet ---
// Le sérialiseur ou le template peut maintenant extraire les liens de manière standard.

echo "Titre : " . $articleWithLinks->title . "\n";
echo "Liens disponibles :\n";

foreach ($articleWithLinks->getLinks() as $link) {
    echo sprintf(
        "- Relation (rel): %s, Cible (href): %s\n",
        implode(', ', $link->getRels()),
        $link->getHref()
    );
}

// On peut aussi chercher un lien par sa relation
$authorLinks = $articleWithLinks->getLinksByRel('author');
if (!empty($authorLinks)) {
    echo "\nLien vers l'auteur trouvé : " . $authorLinks[0]->getHref() . "\n";
}
```

**4. Sérialisation en JSON**
Le but final est de transformer cet objet en une réponse JSON. Un sérialiseur compatible PSR-13 pourrait produire quelque chose comme ceci (format [HAL](https://stateless.co/hal_specification.html) par exemple) :

```json
{
    "id": 101,
    "title": "Titre de l'article",
    "content": "Contenu de l'article...",
    "authorId": 42,
    "_links": {
        "self": {
            "href": "/articles/101"
        },
        "author": {
            "href": "/users/42"
        }
    }
}
```

## En Bref

| Concept | Description |
| :--- | :--- |
| **`LinkInterface`** | Représente un seul lien hypermédia (`href`, `rel`...). |
| **`LinkProviderInterface`** | Un objet (souvent une ressource de l'API) qui peut fournir une liste de `LinkInterface`. |
| **`EvolvableLinkProviderInterface`** | Une version immuable du `LinkProviderInterface` qui permet d'ajouter/retirer des liens. |
| **HATEOAS** | Le concept d'API REST qui consiste à inclure des liens d'actions possibles dans les réponses. |
| **Interopérabilité** | Permettre à des outils (sérialiseurs, clients...) de comprendre les liens d'une API sans connaître son format propriétaire. |
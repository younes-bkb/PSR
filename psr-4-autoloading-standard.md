# PSR-4 : Autoloader

## L'Objectif : Charger des classes automatiquement et de manière prévisible

Avant l'autoloading, on devait inclure manuellement chaque fichier de classe dont on avait besoin avec `require_once 'path/to/MaClasse.php';`. C'était fastidieux, source d'erreurs et rendait la gestion des dépendances très compliquée.

L'**autoloading** est un mécanisme de PHP qui permet de charger automatiquement le fichier contenant une classe au moment précis où cette classe est utilisée pour la première fois.

La PSR-4 est un standard qui décrit **comment** un nom de classe doit être mappé à un chemin de fichier. En respectant cette convention, n'importe quel autoloader compatible (comme celui de Composer) peut trouver et charger n'importe quelle classe de n'importe quelle bibliothèque sans aucune configuration manuelle.

## Le Principe Fondamental

La PSR-4 établit une correspondance directe entre le **nom de classe pleinement qualifié** (*Fully Qualified Class Name* ou FQCN) et le **chemin du fichier** sur le disque.

Le FQCN est composé de :
`\<NamespaceName>(\<SubNamespaceNames>)*\<ClassName>`

La règle de mapping est la suivante :

1.  On définit un **préfixe de namespace** (ex: `MonProjet\`) qui correspond à un **répertoire de base** (ex: `src/`).
2.  Pour un FQCN donné, on prend le nom **après** le préfixe de namespace (ex: `Service\Facturation`).
3.  On remplace les séparateurs de namespace `\` par le séparateur de répertoire du système (`/`).
4.  On ajoute l'extension `.php` à la fin.

**Exemple :**
-   Préfixe de namespace : `Acme\Log\`
-   Répertoire de base : `src/`
-   Nom de classe (FQCN) : `Acme\Log\Writer\File`

**Mapping :**
1.  Préfixe : `Acme\Log\` -> `src/`
2.  Reste du FQCN : `Writer\File`
3.  Transformation : `Writer/File`
4.  Ajout de l'extension : `Writer/File.php`
5.  **Chemin final :** `src/Writer/File.php`

## Cas d'Usage avec Composer

Composer est l'outil qui met en pratique la PSR-4 dans la quasi-totalité des projets PHP modernes. La configuration se fait dans le fichier `composer.json`.

### Scénario
Nous avons un projet avec la structure de répertoires suivante :

```
mon-projet/
├── src/
│   ├── Controller/
│   │   └── HomeController.php
│   └── Service/
│       └── MailerService.php
├── tests/
│   └── Service/
│       └── MailerServiceTest.php
└── composer.json
```

**1. Configuration dans `composer.json`**

Nous allons mapper le préfixe de namespace `App\` au répertoire `src/`, et `App\Tests\` au répertoire `tests/`.

```json
{
    "name": "vendor/mon-projet",
    "autoload": {
        "psr-4": {
            "App\\": "src/",
            "App\\Tests\\": "tests/"
        }
    }
}
```
*Note : Le double backslash `\\` est nécessaire pour échapper le `\` dans le format JSON.*

Après avoir modifié `composer.json`, il faut toujours lancer la commande `composer dump-autoload` pour que Composer génère les fichiers d'autoloading optimisés.

**2. Les fichiers PHP**

Les fichiers doivent respecter scrupuleusement le mapping défini.

**Fichier : `src/Controller/HomeController.php`**
```php
<?php
// FQCN : App\Controller\HomeController
namespace App\Controller;

use App\Service\MailerService; // <-- On peut maintenant utiliser la classe

class HomeController
{
    public function index()
    {
        $mailer = new MailerService(); // L'autoloading se déclenche ici
        // ...
    }
}
```

**Fichier : `src/Service/MailerService.php`**
```php
<?php
// FQCN : App\Service\MailerService
namespace App\Service;

class MailerService
{
    // ...
}
```

**Fichier : `tests/Service/MailerServiceTest.php`**
```php
<?php
// FQCN : App\Tests\Service\MailerServiceTest
namespace App\Tests\Service;

use App\Service\MailerService; // Fonctionne car les deux namespaces sont mappés

class MailerServiceTest
{
    // ...
}
```

**3. Le point d'entrée de l'application**

Le seul `require` nécessaire dans tout le projet est celui qui charge l'autoloader de Composer.

**Fichier : `public/index.php` (par exemple)**
```php
<?php
// Inclure l'autoloader généré par Composer
require __DIR__ . '/../vendor/autoload.php';

use App\Controller\HomeController;

$controller = new HomeController(); // Magie ! Le fichier est trouvé et chargé.
$controller->index();
```

## En Bref

| Concept | Description |
| :--- | :--- |
| **Autoloading** | Mécanisme PHP pour charger les fichiers de classes à la demande. |
| **PSR-4** | Standard qui définit la correspondance entre un nom de classe et un chemin de fichier. |
| **FQCN** | *Fully Qualified Class Name*. Le nom complet d'une classe, incluant son namespace. |
| **Préfixe de Namespace** | La partie du namespace mappée à un répertoire de base (ex: `App\`). |
| **Répertoire de Base** | Le dossier de départ associé à un préfixe de namespace (ex: `src/`). |
| **`composer.json`** | Le fichier où l'on configure le mapping PSR-4 pour Composer. |
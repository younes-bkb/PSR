# PSR-11 : Container Interface

## L'Objectif : Standardiser la manière de récupérer des services

Un **conteneur d'injection de dépendances** (ou *Service Container*) est un objet qui sait comment créer et configurer d'autres objets (appelés "services" ou "entrées"). C'est un outil extrêmement puissant pour gérer les dépendances dans une application et respecter le principe d'Inversion de Contrôle.

Le problème était que chaque framework ou bibliothèque de conteneur (`Pimple`, `PHP-DI`, `Symfony DependencyInjection`, `Laravel Container`...) avait sa propre méthode pour récupérer un service : `$container->get()`, `$container['service']`, `App::make()`, etc.

La PSR-11 a pour but de standardiser **la manière dont on récupère une entrée d'un conteneur**. Elle ne s'intéresse PAS à la configuration du conteneur (comment on y déclare les services), mais uniquement à l'interface publique pour les consommer.

**Le bénéfice :** Une bibliothèque ou un composant de framework qui a besoin de récupérer un service peut dépendre de l'interface `ContainerInterface` et fonctionner avec n'importe quel conteneur compatible PSR-11.

## Les Interfaces

La PSR-11 est très simple et ne définit que deux interfaces.

### 1. `Psr\Container\ContainerInterface`
C'est l'interface principale. Elle ne contient que deux méthodes.

```php
namespace Psr\Container;

interface ContainerInterface
{
    /**
     * Trouve une entrée du conteneur par son identifiant et la retourne.
     * @param string $id Identifiant de l'entrée à chercher.
     * @throws NotFoundExceptionInterface  Aucune entrée n'a été trouvée pour cet id.
     * @throws ContainerExceptionInterface Erreur lors de la récupération de l'entrée.
     */
    public function get(string $id);

    /**
     * Retourne true si le conteneur peut retourner une entrée pour l'identifiant donné.
     * Retourne false sinon.
     * `has($id)` retournant true ne garantit PAS que `get($id)` ne lèvera pas d'exception.
     */
    public function has(string $id): bool;
}```

-   **`get(string $id)`** : La méthode principale. Elle prend un identifiant (généralement le nom de la classe) et retourne l'instance du service correspondant.
-   **`has(string $id)`** : Permet de vérifier si le conteneur est capable de fournir un service pour un identifiant donné.

### 2. `Psr\Container\ContainerExceptionInterface`
Une interface de marquage pour toutes les exceptions levées par le conteneur, permettant de les attraper de manière générique.

### 3. `Psr\Container\NotFoundExceptionInterface`
Hérite de `ContainerExceptionInterface`. C'est l'exception spécifique qui **DOIT** être levée par `get()` si l'identifiant demandé n'existe pas dans le conteneur.

## Cas d'Usage : Utiliser un conteneur pour obtenir des dépendances

### Scénario
Nous avons un `MailerService` qui a besoin d'une configuration pour fonctionner. Un conteneur PSR-11 va nous fournir ce service déjà configuré.

**1. Le service et sa configuration**
```php
<?php
class MailerService
{
    private string $smtpHost;

    public function __construct(string $smtpHost)
    {
        $this->smtpHost = $smtpHost;
    }

    public function sendEmail(string $recipient, string $body): void
    {
        echo "Email envoyé à {$recipient} via {$this->smtpHost}\n";
    }
}
```

**2. Une implémentation simple de conteneur PSR-11**
Pour l'exemple, créons un conteneur basique. Les vrais conteneurs sont bien plus complexes (gestion des singletons, autowiring, etc.).
```php
<?php
use Psr\Container\ContainerInterface;

class MyContainer implements ContainerInterface
{
    private array $entries = [];

    public function get(string $id)
    {
        if (!$this->has($id)) {
            throw new class($id) extends \Exception implements Psr\Container\NotFoundExceptionInterface {};
        }
        $entry = $this->entries[$id];
        return $entry(); // On exécute la factory pour créer le service
    }

    public function has(string $id): bool
    {
        return isset($this->entries[$id]);
    }
    
    // Méthode de configuration (non standardisée par PSR-11)
    public function set(string $id, callable $factory): void
    {
        $this->entries[$id] = $factory;
    }
}
```

**3. Configuration et utilisation**
C'est ici que la PSR-11 entre en jeu. Le `UserController` dépend de l'interface, pas de notre `MyContainer`.

```php
<?php
// --- Phase de configuration (ex: dans un fichier bootstrap) ---
$container = new MyContainer();

// On configure comment créer le MailerService
$container->set(MailerService::class, function () {
    $smtpHost = 'smtp.example.com'; // Pourrait venir d'un fichier .env
    return new MailerService($smtpHost);
});

// --- Phase d'exécution (dans votre code métier) ---
class UserController
{
    private ContainerInterface $container;

    public function __construct(ContainerInterface $container)
    {
        $this->container = $container;
    }

    public function registerUser(string $email)
    {
        // ... logique d'enregistrement ...

        // On demande le service au conteneur via l'interface PSR-11
        /** @var MailerService $mailer */
        $mailer = $this->container->get(MailerService::class);
        $mailer->sendEmail($email, 'Bienvenue !');
    }
}

// On utilise le contrôleur
$controller = new UserController($container);
$controller->registerUser('test@example.com');
```

**Note sur les bonnes pratiques :**
Injecter le conteneur entier dans un service (`new UserController($container)`) est souvent considéré comme un anti-pattern (appelé *Service Locator*). Idéalement, le conteneur ne devrait injecter que les dépendances directes (`new UserController($container->get(MailerService::class))`). La PSR-11 est surtout utile pour les briques d'infrastructure (frameworks, routeurs) qui ont besoin d'un moyen standard d'instancier des objets.

## En Bref

| Concept | Description |
| :--- | :--- |
| **`ContainerInterface`** | L'interface standard pour interagir avec un conteneur DI. |
| **`get(string $id)`** | Récupère un service par son identifiant (souvent, son nom de classe). |
| **`has(string $id)`** | Vérifie si le conteneur est capable de fournir un service. |
| **Interopérabilité** | Permet à des composants de fonctionner avec n'importe quel conteneur DI compatible. |
| **Périmètre** | La PSR-11 ne standardise **que la récupération** de services, pas leur configuration. |
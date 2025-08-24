# PSR-3 : Logger Interface

## L'Objectif : Standardiser la journalisation des événements

Presque toutes les applications ont besoin de "logger", c'est-à-dire d'enregistrer des événements : une erreur critique, une information de débogage, une action utilisateur, etc.

Le problème est que chaque bibliothèque de logging (`Monolog`, `Log4php`, le logger de Symfony, etc.) avait sa propre façon d'être utilisée. Si vous vouliez changer de système de log dans votre projet, vous deviez réécrire tout le code qui faisait appel au logger.

La PSR-3 résout ce problème en définissant une **interface commune**, `Psr\Log\LoggerInterface`, que toutes les bibliothèques de logging modernes peuvent implémenter.

**Le bénéfice :** Votre code n'a plus besoin de connaître la bibliothèque de log concrète. Il dépend uniquement de l'interface PSR-3. Vous pouvez ainsi remplacer `Monolog` par un autre logger compatible PSR-3 sans changer **une seule ligne de votre code métier**.

## L'Interface `Psr\Log\LoggerInterface`

L'interface est simple. Elle décrit 8 méthodes, une pour chaque niveau de sévérité des logs, plus une méthode générique `log()`.

Les 8 niveaux de sévérité, du plus bas (débogage) au plus haut (urgence), sont définis dans la [RFC 5424](https://tools.ietf.org/html/rfc5424).

```php
namespace Psr\Log;

interface LoggerInterface
{
    public function emergency(string|\Stringable $message, array $context = []): void;
    public function alert(string|\Stringable $message, array $context = []): void;
    public function critical(string|\Stringable $message, array $context = []): void;
    public function error(string|\Stringable $message, array $context = []): void;
    public function warning(string|\Stringable $message, array $context = []): void;
    public function notice(string|\Stringable $message, array $context = []): void;
    public function info(string|\Stringable $message, array $context = []): void;
    public function debug(string|\Stringable $message, array $context = []): void;
    public function log($level, string|\Stringable $message, array $context = []): void;
}
```

## Les Deux Arguments Clés

Chaque méthode prend les mêmes deux arguments principaux :

### 1. `$message` (`string|\Stringable`)
C'est le message de log lui-même. Il peut contenir des **placeholders** (des marqueurs) sous la forme `{placeholder}`.

### 2. `$context` (`array`)
C'est un tableau associatif qui fournit des données contextuelles. Les clés de ce tableau correspondent aux placeholders dans le message. Le logger se chargera de remplacer les placeholders par les valeurs du contexte.

-   Les valeurs du contexte **PEUVENT** contenir n'importe quoi. Les implémentations doivent gérer leur affichage de manière sensée.
-   Une clé `exception` est spéciale : elle **DEVRAIT** contenir un objet `Throwable` (une exception) pour un logging d'erreur détaillé.

## Cas d'Usage : Implémenter et Utiliser le Logger

### Scénario
Imaginons une classe `UserManager` qui gère la création d'utilisateurs. Elle a besoin de logger plusieurs événements. Elle ne connaît pas `Monolog`, elle demande simplement un `LoggerInterface` dans son constructeur (injection de dépendances).

**1. La classe qui utilise le logger**
```php
<?php
use Psr\Log\LoggerInterface;

class UserManager
{
    private LoggerInterface $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function createUser(string $email, string $password): void
    {
        if (empty($email)) {
            // Log d'une erreur applicative, avec le contexte.
            $this->logger->error(
                'Tentative de création d\'utilisateur avec un email vide.',
                ['source_ip' => $_SERVER['REMOTE_ADDR']]
            );
            return;
        }

        $this->logger->info(
            'Nouvel utilisateur créé : {email}',
            ['email' => $email]
        );

        // ... logique de création
    }

    public function handleDatabaseError(\PDOException $e): void
    {
        // Log d'une erreur critique avec une exception.
        $this->logger->critical(
            'Erreur de base de données lors de la création de l\'utilisateur.',
            ['exception' => $e]
        );
    }
}
```

**2. Le code qui assemble le tout (l'index.php)**
Ici, nous choisissons d'utiliser `Monolog` comme implémentation concrète du logger.

```php
<?php
require 'vendor/autoload.php';

use Monolog\Logger;
use Monolog\Handler\StreamHandler;

// 1. On crée une instance de Monolog (qui implémente PSR-3 LoggerInterface).
$monolog = new Logger('app');
$monolog->pushHandler(new StreamHandler('app.log', Logger::WARNING));

// 2. On injecte Monolog dans notre classe.
// UserManager ne voit que l'interface, pas l'implémentation.
$userManager = new UserManager($monolog);

// 3. On utilise notre service.
$userManager->createUser('test@example.com', 'password123');
$userManager->createUser('', 'password123'); // Va générer une erreur dans le log
```

**Contenu du fichier `app.log` après exécution :**
```
[2025-08-24T19:42:00.123456+00:00] app.ERROR: Tentative de création d'utilisateur avec un email vide. {"source_ip":"127.0.0.1"} []
```
*Note : Le log `info` n'apparaît pas car nous avons configuré Monolog pour ne logger que les niveaux `WARNING` et supérieurs.*

## En Bref

| Concept | Description |
| :--- | :--- |
| **`Psr\Log\LoggerInterface`** | L'interface standard que les loggers doivent implémenter. |
| **8 niveaux de log** | `debug`, `info`, `notice`, `warning`, `error`, `critical`, `alert`, `emergency`. |
| **`$message`** | Le message de log, qui peut contenir des placeholders `{key}`. |
| **`$context`** | Un tableau de données qui remplace les placeholders et ajoute du contexte. |
| **Interopérabilité** | Le but principal : pouvoir changer de bibliothèque de logging sans modifier le code qui l'utilise. |
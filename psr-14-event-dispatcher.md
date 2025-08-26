# PSR-14 : Event Dispatcher

## L'Objectif : Un moyen standard de gérer les événements et les listeners

De nombreuses applications ont besoin d'un système d'événements pour permettre à différentes parties du code de réagir à une action sans être directement couplées. Par exemple, quand un utilisateur s'inscrit, on veut :
1.  Enregistrer l'utilisateur en base de données.
2.  **Puis**, envoyer un email de bienvenue.
3.  **Et aussi**, ajouter des points de fidélité.
4.  **Et peut-être**, notifier un administrateur.

Au lieu de coder tout cela dans une seule méthode `registerUser()`, on peut simplement "déclencher" un événement `UserRegistered`. D'autres parties de l'application (des *listeners*) peuvent "écouter" cet événement et exécuter leur propre logique.

La PSR-14 a pour but de standardiser ce mécanisme. Elle définit des interfaces pour :
-   **L'Événement** lui-même (l'objet qui contient les données de l'événement).
-   **Le Dispatcher** (le service central qui envoie l'événement).
-   **Le Listener Provider** (le service qui sait quels listeners écoutent quel événement).

**Le bénéfice :** Permettre à des bibliothèques ou des modules de s'intégrer facilement dans le système d'événements d'une application, sans dépendre de l'implémentation spécifique du framework. Un module peut fournir des listeners qui seront automatiquement découverts et exécutés par n'importe quel dispatcher compatible PSR-14.

## Les Interfaces Clés

-   **`EventDispatcherInterface` (Le Dispatcher)** : C'est le service principal que votre code utilisera pour **déclencher** un événement.
    -   `dispatch(object $event)`: La seule méthode. Elle prend un objet Événement et le transmet à tous les listeners intéressés.

-   **`ListenerProviderInterface` (Le Fournisseur de Listeners)** : L'objet qui **sait** quel listener correspond à quel événement. Le Dispatcher l'utilise en interne pour trouver les bons listeners.
    -   `getListenersForEvent(object $event)`: Retourne un `iterable` de *callables* (les listeners) pour un événement donné.

-   **`StoppableEventInterface` (Événement Stoppable)** : Une interface optionnelle que votre objet Événement peut implémenter. Elle permet à un listener d'**arrêter la propagation** de l'événement aux listeners suivants.
    -   `isPropagationStopped()`: Vérifie si l'événement a été arrêté.

## Cas d'Usage : Événement d'inscription utilisateur

### Scénario
Quand un utilisateur est créé, nous voulons envoyer un email de bienvenue et créer un profil par défaut. Ces deux actions seront gérées par des listeners distincts.

**1. L'objet Événement**
C'est un simple objet (POPO - Plain Old PHP Object) qui transporte les données. Ici, l'objet `User`.
```php
<?php
class User
{
    public string $email;
    public function __construct(string $email) { $this->email = $email; }
}

class UserRegisteredEvent
{
    public User $user;
    public function __construct(User $user) { $this->user = $user; }
}
```

**2. Les Listeners**
Ce sont des fonctions ou des méthodes (*callables*) qui reçoivent l'objet Événement en argument.
```php
<?php
class WelcomeEmailListener
{
    public function __invoke(UserRegisteredEvent $event): void
    {
        echo "Listener 1 : Envoi de l'email de bienvenue à " . $event->user->email . "\n";
    }
}

class UserProfileCreationListener
{
    public function handle(UserRegisteredEvent $event): void
    {
        echo "Listener 2 : Création du profil pour " . $event->user->email . "\n";
    }
}
```

**3. Le service qui déclenche l'événement**
Notre `UserManager` ne connaît pas les listeners. Il se contente de déclencher l'événement via l'interface du Dispatcher.
```php
<?php
use Psr\EventDispatcher\EventDispatcherInterface;

class UserManager
{
    private EventDispatcherInterface $dispatcher;

    public function __construct(EventDispatcherInterface $dispatcher)
    {
        $this->dispatcher = $dispatcher;
    }

    public function register(string $email): User
    {
        echo "Action : Enregistrement de l'utilisateur {$email} en BDD.\n";
        $user = new User($email);

        // ... logique de sauvegarde en BDD ...

        // On déclenche l'événement
        $event = new UserRegisteredEvent($user);
        $this->dispatcher->dispatch($event);

        echo "Action : Fin de l'enregistrement.\n";
        return $user;
    }
}
```

**4. Assemblage avec une implémentation de la PSR-14**
Nous allons utiliser une bibliothèque simple comme `league/event` pour l'exemple.
```php
<?php
require 'vendor/autoload.php';

use League\Event\EventDispatcher;
use League\Event\ListenerRegistry;

// 1. On crée le fournisseur de listeners et on y enregistre nos listeners.
$listenerRegistry = new ListenerRegistry();

$listenerRegistry->subscribeTo(
    UserRegisteredEvent::class,
    new WelcomeEmailListener() // La méthode __invoke sera appelée
);

$profileListener = new UserProfileCreationListener();
$listenerRegistry->subscribeTo(
    UserRegisteredEvent::class,
    [$profileListener, 'handle'] // On spécifie la méthode à appeler
);

// 2. On crée le dispatcher, en lui donnant le fournisseur de listeners.
$dispatcher = new EventDispatcher($listenerRegistry);

// 3. On injecte le dispatcher dans notre service.
$userManager = new UserManager($dispatcher);

// 4. On exécute l'action. Le dispatcher se chargera d'appeler les listeners.
$userManager->register('test@example.com');
```

**Résultat de l'exécution :**
```
Action : Enregistrement de l'utilisateur test@example.com en BDD.
Listener 1 : Envoi de l'email de bienvenue à test@example.com
Listener 2 : Création du profil pour test@example.com
Action : Fin de l'enregistrement.
```
On voit que la logique des listeners a été exécutée au bon moment, sans que `UserManager` n'ait aucune connaissance de leur existence.

## En Bref

| Concept | Description |
| :--- | :--- |
| **`EventDispatcherInterface`**| Le service utilisé pour **déclencher** (`dispatch`) un événement. |
| **`ListenerProviderInterface`** | Le service qui **sait** quels listeners correspondent à un événement. |
| **Événement** | Un simple objet PHP qui transporte les données relatives à un événement. |
| **Listener** | Une fonction ou méthode (*callable*) qui reçoit l'événement et exécute une logique. |
| **Découplage** | Le but principal : permettre à des modules de réagir à des événements sans couplage direct avec le code qui les déclenche. |
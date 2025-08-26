# PSR-20 : Clock

## L'Objectif : Rendre le temps "testable"

Obtenir l'heure et la date actuelles en PHP est très simple : `new \DateTimeImmutable()`. Le problème, c'est que cette fonction est une **dépendance cachée et incontrôlable**.

Imaginez une méthode qui vérifie si un token a expiré :
```php
class TokenManager
{
    public function isTokenExpired(Token $token): bool
    {
        // Dépendance cachée à l'heure système !
        $now = new \DateTimeImmutable(); 
        return $now > $token->getExpirationDate();
    }
}
```
Comment tester cette méthode de manière fiable ?
-   On ne peut pas tester le cas "le token expire dans 10 secondes" sans mettre un `sleep(11)` dans notre test, ce qui est lent et fragile.
-   On ne peut pas tester le cas "le token a expiré hier" sans manipuler des objets `Token` avec des dates passées.

Le code est **directement couplé à l'horloge du système**, qui est une ressource externe et non déterministe.

La PSR-20 résout ce problème en introduisant une abstraction très simple pour la gestion du temps : une interface `ClockInterface`.

**Le bénéfice :** Votre code métier ne dépend plus de `new \DateTimeImmutable()`, mais de `ClockInterface`. En production, vous injecterez une horloge qui retourne l'heure réelle. Mais **dans vos tests**, vous pourrez injecter une **fausse horloge** (`MockClock` ou `FrozenClock`) que vous contrôlez totalement. Vous pourrez ainsi "figer" le temps à un instant T ou le faire avancer comme vous le souhaitez.

## L'Interface `Psr\Clock\ClockInterface`

L'interface est d'une simplicité extrême.```php
namespace Psr\Clock;

interface ClockInterface
{
    /**
     * Retourne la date et l'heure actuelles sous la forme d'un objet DateTimeImmutable.
     */
    public function now(): \DateTimeImmutable;
}

-   **`now(): \DateTimeImmutable`** : C'est la seule et unique méthode. Elle doit retourner l'heure actuelle. La PSR-20 impose l'utilisation de `DateTimeImmutable` (la version immuable de `DateTime`) pour éviter les effets de bord.

## Cas d'Usage : Rendre notre `TokenManager` testable

### Scénario
Nous allons réécrire notre `TokenManager` pour qu'il soit découplé de l'horloge système.

**1. La classe `TokenManager` mise à jour**
Elle dépend maintenant de `ClockInterface`.
```php
<?php
use Psr\Clock\ClockInterface;

class Token
{
    private \DateTimeImmutable $expiresAt;
    public function __construct(\DateTimeImmutable $expiresAt) { $this->expiresAt = $expiresAt; }
    public function getExpirationDate(): \DateTimeImmutable { return $this->expiresAt; }
}

class TokenManager
{
    private ClockInterface $clock;

    public function __construct(ClockInterface $clock)
    {
        $this->clock = $clock;
    }

    public function isTokenExpired(Token $token): bool
    {
        // On utilise l'horloge injectée, pas l'heure système.
        $now = $this->clock->now();
        return $now >= $token->getExpirationDate();
    }
}
```
Notre logique métier est maintenant testable.

**2. Utilisation en production**
En production, on injecte une horloge qui retourne l'heure réelle. La PSR-20 ne fournit pas d'implémentation, mais elle est triviale à créer.
```php
<?php
// Implémentation pour la production
class SystemClock implements ClockInterface
{
    public function now(): \DateTimeImmutable
    {
        return new \DateTimeImmutable();
    }
}

// Dans notre application
$realClock = new SystemClock();
$tokenManager = new TokenManager($realClock);

$expiredToken = new Token(new \DateTimeImmutable('yesterday'));
$validToken = new Token(new \DateTimeImmutable('tomorrow'));

var_dump($tokenManager->isTokenExpired($expiredToken)); // bool(true)
var_dump($tokenManager->isTokenExpired($validToken));   // bool(false)
```

**3. Utilisation en test (le vrai intérêt de la PSR)**
Dans nos tests unitaires, nous utilisons une fausse horloge que nous contrôlons.
```php
<?php
// Fausse horloge pour les tests
class FrozenClock implements ClockInterface
{
    private \DateTimeImmutable $now;

    public function __construct(string $datetime)
    {
        $this->now = new \DateTimeImmutable($datetime);
    }
    
    // now() retourne toujours la même heure, celle que nous avons fixée.
    public function now(): \DateTimeImmutable
    {
        return $this->now;
    }
}

// --- Fichier de test (ex: avec PHPUnit) ---
// use PHPUnit\Framework\TestCase;

class TokenManagerTest // extends TestCase
{
    public function testTokenIsExpired()
    {
        // 1. On "gèle" le temps au 15 janvier 2025 à midi.
        $frozenClock = new FrozenClock('2025-01-15 12:00:00');
        $tokenManager = new TokenManager($frozenClock);

        // 2. On crée un token qui a expiré il y a une seconde.
        $token = new Token(new \DateTimeImmutable('2025-01-15 11:59:59'));

        // 3. On affirme que notre méthode retourne bien "true".
        // Le test est déterministe et fonctionnera toujours,
        // peu importe quand il est exécuté.
        assert(true === $tokenManager->isTokenExpired($token));
    }

    public function testTokenIsNotExpired()
    {
        // 1. On gèle le temps.
        $frozenClock = new FrozenClock('2025-01-15 12:00:00');
        $tokenManager = new TokenManager($frozenClock);

        // 2. On crée un token qui expirera dans une seconde.
        $token = new Token(new \DateTimeImmutable('2025-01-15 12:00:01'));

        // 3. On affirme que notre méthode retourne bien "false".
        assert(false === $tokenManager->isTokenExpired($token));
    }
}

// On exécute les "tests"
(new TokenManagerTest())->testTokenIsExpired();
(new TokenManagerTest())->testTokenIsNotExpired();
echo "Tests passés avec succès !\n";
```

## En Bref

| Concept | Description |
| :--- | :--- |
| **`ClockInterface`** | L'interface standard pour obtenir l'heure et la date actuelles. |
| **`now()`** | La seule méthode, qui doit retourner un `\DateTimeImmutable`. |
| **Dépendance cachée** | Le problème que la PSR-20 résout : l'utilisation directe de `new DateTime...` qui rend le code difficilement testable. |
| **Testabilité** | L'objectif principal : permettre d'injecter une fausse horloge (`MockClock` ou `FrozenClock`) dans les tests pour contrôler le temps. |
| **Découplage** | Le code métier est découplé de l'horloge système, une ressource externe. |
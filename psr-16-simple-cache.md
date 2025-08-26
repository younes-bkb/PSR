# PSR-16 : Simple Cache

## L'Objectif : Une interface de cache simple et directe

La [PSR-6](./psr-6-caching-interface.md) est très puissante, mais son modèle d'objets (`CacheItemPool`, `CacheItem`) peut être lourd pour des cas d'usage simples. Souvent, tout ce qu'un développeur veut faire, c'est :
-   `set('ma_cle', $ma_donnee, 3600)` : Sauvegarder une donnée pour une heure.
-   `get('ma_cle')` : Récupérer la donnée.
-   `delete('ma_cle')` : La supprimer.

La PSR-16, ou **Simple Cache**, a été conçue pour standardiser cette interaction directe et simplifiée avec un système de cache. Elle abandonne le modèle d'objets `CacheItem` pour travailler directement avec les clés et les valeurs.

**Le bénéfice :** Offrir une courbe d'apprentissage beaucoup plus faible et une API plus intuitive pour 80% des besoins de cache, tout en conservant l'interopérabilité. Une bibliothèque qui a besoin d'un cache simple peut dépendre de `CacheInterface` et fonctionner avec n'importe quelle implémentation (Filesystem, Redis...).

## L'Interface `Psr\SimpleCache\CacheInterface`

L'interface principale est très facile à comprendre.

```php
namespace Psr\SimpleCache;

interface CacheInterface
{
    public function get(string $key, mixed $default = null): mixed;
    public function set(string $key, mixed $value, null|int|\DateInterval $ttl = null): bool;
    public function delete(string $key): bool;
    public function clear(): bool;
    
    // Méthodes pour manipuler plusieurs clés à la fois
    public function getMultiple(iterable $keys, mixed $default = null): iterable;
    public function setMultiple(iterable $values, null|int|\DateInterval $ttl = null): bool;
    public function deleteMultiple(iterable $keys): bool;

    public function has(string $key): bool;
}
```

### Les Méthodes Clés

-   **`get(string $key, mixed $default = null)`**
    -   Récupère la valeur associée à la clé.
    -   Si la clé n'existe pas ou si la valeur a expiré, elle retourne la valeur `$default` (qui est `null` par défaut). C'est beaucoup plus direct que le système de "hit/miss" de la PSR-6.

-   **`set(string $key, mixed $value, $ttl = null)`**
    -   Sauvegarde une valeur (`$value`) sous une clé (`$key`).
    -   `$ttl` (*Time To Live*) est la durée de vie en secondes (un `int`) ou un `DateInterval`. Si `null`, le cache peut durer indéfiniment.

-   **`delete(string $key)`**
    -   Supprime une entrée du cache.

-   **`has(string $key)`**
    -   Vérifie si une clé existe dans le cache et n'a pas expiré.

## Cas d'Usage : Mettre en cache la configuration d'une application

### Scénario
Nous avons une classe `ConfigLoader` qui charge une configuration depuis un fichier `config.php`, ce qui peut être lent. Nous allons utiliser la PSR-16 pour mettre ce tableau de configuration en cache.

**1. La classe qui utilise le cache**
Elle dépend de l'interface `CacheInterface`.
```php
<?php
use Psr\SimpleCache\CacheInterface;

class ConfigLoader
{
    private CacheInterface $cache;
    private string $configFile;

    public function __construct(CacheInterface $cache, string $configFile)
    {
        $this->cache = $cache;
        $this->configFile = $configFile;
    }

    public function getConfig(): array
    {
        $cacheKey = 'app_config';
        
        // On essaie de récupérer la config depuis le cache
        $config = $this->cache->get($cacheKey);
        
        // Si get() retourne autre chose que null, c'est un "hit"
        if ($config !== null) {
            echo "Récupération de la config depuis le CACHE.\n";
            return $config;
        }

        echo "Lecture de la config depuis le FICHIER (opération lente).\n";
        
        // Simule une lecture de fichier
        sleep(1); 
        $configData = require $this->configFile;

        // On sauvegarde la config dans le cache pour 5 minutes
        $this->cache->set($cacheKey, $configData, 300);

        return $configData;
    }
}
```

**2. Le code qui assemble le tout**
Nous utilisons à nouveau `symfony/cache`, qui fournit aussi un adaptateur PSR-6 vers PSR-16.

```php
<?php
require 'vendor/autoload.php';

use Symfony\Component\Cache\Adapter\FilesystemAdapter;
use Symfony\Component\Cache\Psr16Cache;

// Fichier de configuration simulé
file_put_contents('config.php', '<?php return ["db_host" => "localhost", "api_key" => "xyz123"];');

// 1. On crée un adaptateur PSR-6...
$psr6Cache = new FilesystemAdapter();
// 2. ...et on l'enveloppe dans un adaptateur PSR-16.
$psr16Cache = new Psr16Cache($psr6Cache);

// 3. On injecte le cache PSR-16 dans notre service.
$configLoader = new ConfigLoader($psr16Cache, 'config.php');

// --- Premier appel ---
echo "Premier chargement...\n";
$config1 = $configLoader->getConfig();
print_r($config1);

// --- Deuxième appel ---
echo "\nDeuxième chargement...\n";
$config2 = $configLoader->getConfig();
print_r($config2);
```

**Résultat de l'exécution :**
```
Premier chargement...
Lecture de la config depuis le FICHIER (opération lente).
Array
(
    [db_host] => localhost
    [api_key] => xyz123
)

Deuxième chargement...
Récupération de la config depuis le CACHE.
Array
(
    [db_host] => localhost
    [api_key] => xyz123
)
```

## PSR-6 vs PSR-16 : Laquelle choisir ?

| Caractéristique | PSR-6 (Cache) | PSR-16 (Simple Cache) |
| :--- | :--- | :--- |
| **Complexité** | Élevée | **Faible** |
| **Modèle** | `CacheItemPool` et `CacheItem` | Accès direct Clé => Valeur |
| **Gestion des "miss"** | `isHit()` puis `get()` | `get()` retourne `$default` (souvent `null`) |
| **Contrôle fin** | **Oui** (tags, expiration différée) | Non |
| **Cas d'usage** | Caches robustes pour bibliothèques/frameworks | **Cache simple dans le code applicatif** |

En résumé, si vous développez une bibliothèque réutilisable ou avez besoin d'un contrôle très fin sur le cache, utilisez la **PSR-6**. Pour la plupart des besoins quotidiens dans votre code applicatif, la **PSR-16** est plus simple et plus rapide à mettre en œuvre.

## En Bref

| Concept | Description |
| :--- | :--- |
| **`CacheInterface`** | L'interface unique pour toutes les opérations de cache simple. |
| **`get($key, $default)`** | Récupère une valeur ou retourne `$default` si elle n'existe pas. |
| **`set($key, $value, $ttl)`** | Sauvegarde une valeur pour une durée de vie (`$ttl`) en secondes. |
| **Simplicité** | L'objectif principal : une API intuitive pour des besoins de cache courants. |
| **Interopérabilité** | Permet de changer de moteur de cache sans modifier le code qui l'utilise. |
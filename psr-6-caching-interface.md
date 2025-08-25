# PSR-6 : Caching Interface

## L'Objectif : Une interface de cache puissante et interopérable

Mettre en cache des données (résultats de requêtes BDD, appels API, fragments de page) est une technique essentielle pour optimiser les performances d'une application.

La PSR-6 a pour but de standardiser la manière dont les applications interagissent avec un système de cache. Elle définit une interface commune pour que vous puissiez utiliser un cache **Filesystem**, **Redis**, ou **Memcached** de manière totalement interchangeable.

Votre code métier dépend de l'interface PSR-6, et vous pouvez changer le moteur de cache sous-jacent (le *backend*) via une simple modification de configuration, sans réécrire votre logique.

**Note :** La PSR-6 est considérée comme une interface de cache "avancée". Pour des besoins plus simples, la [PSR-16 (Simple Cache)](./psr-16-simple-cache.md) offre une alternative plus directe.

## Les Concepts Clés de la PSR-6

La PSR-6 introduit un vocabulaire précis qu'il est crucial de comprendre. On n'interagit pas directement avec les données, mais avec des objets qui les représentent.

-   **`CacheItemPoolInterface` (Le Pool)** : C'est le service principal, votre point d'entrée pour interagir avec le système de cache. Il représente le cache dans son ensemble. On l'utilise pour demander, sauvegarder ou supprimer des items.

-   **`CacheItemInterface` (L'Item)** : C'est un **objet wrapper** qui représente une seule entrée dans le cache. Quand vous demandez une donnée au Pool, il ne vous retourne pas la donnée brute, mais un objet `CacheItem`. Cet objet contient non seulement la valeur, mais aussi des métadonnées.

-   **Key (La Clé)** : Une chaîne de caractères unique qui identifie un `CacheItem` dans le Pool.

-   **Hit & Miss (Trouvé ou Manqué)** :
    -   Un **Hit** signifie que le Pool a trouvé un `CacheItem` valide pour la clé demandée. On peut récupérer sa valeur.
    -   Un **Miss** signifie qu'aucun item n'a été trouvé (ou qu'il a expiré). L'application doit alors régénérer la donnée et la sauvegarder dans le cache pour la prochaine fois.

-   **Expiration** : La durée de vie d'un `CacheItem`. La PSR-6 permet de la définir de manière très flexible (en secondes, avec une date précise, etc.).

## L'Interaction de Base

Le flux de travail typique avec la PSR-6 est le suivant :
1.  Demander un `CacheItem` au `CacheItemPoolInterface` avec une clé : `$item = $pool->getItem('ma_cle');`.
2.  Vérifier si c'est un "hit" : `if ($item->isHit()) { ... }`.
3.  Si oui, récupérer la valeur : `$data = $item->get();`.
4.  Si non ("miss"), générer la donnée, la définir sur l'item (`$item->set($data)`), définir une expiration (`$item->expiresAfter(3600)`) et sauvegarder l'item dans le pool (`$pool->save($item)`).

## Cas d'Usage : Mettre en cache un calcul coûteux

### Scénario
Nous avons un service `ReportGenerator` qui génère un rapport de ventes. C'est une opération lente car elle interroge lourdement la base de données. Nous allons la mettre en cache pour une heure.

**1. La classe qui utilise le cache**
Elle dépend de l'interface `CacheItemPoolInterface`, pas d'une implémentation concrète.
```php
<?php
use Psr\Cache\CacheItemPoolInterface;

class ReportGenerator
{
    private CacheItemPoolInterface $cachePool;

    public function __construct(CacheItemPoolInterface $cachePool)
    {
        $this->cachePool = $cachePool;
    }

    public function getDailySalesReport(): array
    {
        $cacheKey = 'sales_report_' . date('Y-m-d');
        $reportItem = $this->cachePool->getItem($cacheKey);

        if ($reportItem->isHit()) {
            echo "Récupération du rapport depuis le CACHE.\n";
            return $reportItem->get();
        }

        echo "Génération du rapport depuis la BDD (opération LENTE).\n";
        
        // Simule une opération longue
        sleep(2); 
        $reportData = [
            'date' => date('Y-m-d'),
            'revenue' => rand(10000, 50000),
            'transactions' => rand(500, 1000)
        ];

        // On configure l'item de cache
        $reportItem->set($reportData);
        $reportItem->expiresAfter(3600); // Expire dans 1 heure

        // On sauvegarde l'item dans le pool
        $this->cachePool->save($reportItem);

        return $reportData;
    }
}
```

**2. Le code qui assemble le tout**
Nous allons utiliser le composant `symfony/cache`, une excellente implémentation de la PSR-6.

```php
<?php
require 'vendor/autoload.php';

use Symfony\Component\Cache\Adapter\FilesystemAdapter;

// 1. On choisit une implémentation. Ici, un cache basé sur des fichiers.
// Pourrait être RedisAdapter, MemcachedAdapter, etc.
$cache = new FilesystemAdapter();

// 2. On injecte le pool de cache dans notre service.
$reportGenerator = new ReportGenerator($cache);

// --- Premier appel ---
echo "Premier appel de la journée...\n";
$report1 = $reportGenerator->getDailySalesReport();
print_r($report1);

// --- Deuxième appel (quelques secondes plus tard) ---
echo "\nDeuxième appel...\n";
$report2 = $reportGenerator->getDailySalesReport();
print_r($report2);
```

**Résultat de l'exécution :**
```
Premier appel de la journée...
Génération du rapport depuis la BDD (opération LENTE).
Array
(
    [date] => 2025-08-25
    [revenue] => 43125
    [transactions] => 789
)

Deuxième appel...
Récupération du rapport depuis le CACHE.
Array
(
    [date] => 2025-08-25
    [revenue] => 43125
    [transactions] => 789
)
```
On voit bien que le deuxième appel a été instantané et a servi la donnée depuis le cache.

## En Bref

| Concept | Description |
| :--- | :--- |
| **`CacheItemPoolInterface`** | Le service de cache principal. On l'utilise pour obtenir/sauvegarder des items. |
| **`CacheItemInterface`** | L'objet qui représente une entrée de cache. Contient la valeur et les métadonnées. |
| **`getItem($key)`** | Récupère un `CacheItemInterface` (ne retourne jamais `null`). |
| **`isHit()`** | Vérifie si la donnée a été trouvée dans le cache et n'a pas expiré. |
| **`get()`** | Récupère la valeur réelle depuis le `CacheItemInterface`. |
| **`set()` & `save()`** | `set()` prépare la donnée sur l'item, `save()` l'enregistre de manière persistante dans le pool. |
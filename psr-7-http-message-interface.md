# PSR-7 : HTTP Message Interface

## L'Objectif : Une représentation standard des messages HTTP

Avant la PSR-7, chaque framework PHP (Symfony, Laravel, Zend...) avait sa propre manière de représenter une requête HTTP entrante et une réponse sortante. Chaque projet avait ses propres objets `Request` et `Response` avec des méthodes différentes.

Ce chaos rendait l'**interopérabilité** impossible. Une brique logicielle conçue pour fonctionner avec le `Request` de Symfony ne pouvait pas comprendre le `Request` de Laravel.

La PSR-7 résout ce problème en définissant un ensemble d'interfaces communes pour représenter les messages HTTP (requêtes et réponses). En adoptant ces interfaces, les développeurs peuvent créer des composants (middlewares, bibliothèques, etc.) qui fonctionnent dans n'importe quel framework moderne.

## Le Concept Clé : L'Immutabilité

C'est la caractéristique la plus importante et la plus déroutante de la PSR-7. **Tous les objets PSR-7 sont immuables.**

Cela signifie qu'un objet, une fois créé, ne peut plus être modifié. Chaque fois que vous voulez "changer" quelque chose (ajouter un en-tête, changer le statut), la méthode ne modifie pas l'objet actuel. Au lieu de cela, elle retourne un **clone** (une nouvelle instance) de l'objet avec la modification appliquée.

Ces méthodes commencent toujours par `with...` (ex: `withHeader()`, `withStatus()`).

✅ **Correct : On récupère le nouvel objet cloné**
```php
// $response est un objet PSR-7
$newResponse = $response->withStatus(404);

// $response est INCHANGÉ (il a toujours son statut d'origine)
// $newResponse est un NOUVEL objet avec le statut 404
```

❌ **Incorrect : L'objet d'origine n'est pas modifié**
```php
$response->withStatus(404); // Ne fait rien !
// Le nouvel objet retourné est perdu, et $response n'a pas changé.
```

**Pourquoi l'immutabilité ?**
Elle garantit qu'un objet ne peut pas être altéré de manière inattendue par une autre partie du code. Quand un middleware passe un objet `Request` au suivant, il a la certitude que son propre objet ne sera pas modifié. C'est un gage de prévisibilité et de sécurité.

## Les Interfaces Principales

-   `RequestInterface` & `ResponseInterface` : Les bases pour les requêtes sortantes et les réponses.
-   `ServerRequestInterface` : Hérite de `RequestInterface`. Représente la **requête entrante** côté serveur. Elle donne accès aux variables superglobales (`$_GET`, `$_POST`, `$_FILES`, `$_COOKIE`, `$_SERVER`).
-   `UriInterface` : Représente l'URL de la requête sous forme d'objet, permettant d'accéder à ses différentes parties (schéma, hôte, chemin...).
-   `StreamInterface` : Représente le corps (body) du message. Au lieu d'une simple chaîne, c'est un flux (*stream*), ce qui permet de gérer des corps de très grande taille (ex: upload de fichiers) sans tout charger en mémoire.
-   `UploadedFileInterface` : Représente un fichier uploadé via un formulaire.

## Cas d'Usage : Un middleware simple

Un middleware est un excellent exemple : c'est une fonction qui reçoit une requête, la traite, et retourne une réponse.

**Note :** Comme la PSR-7 ne définit que des interfaces, nous avons besoin d'une bibliothèque pour fournir les implémentations. `guzzlehttp/psr7` est la plus populaire.

```php
<?php
require 'vendor/autoload.php';

use GuzzleHttp\Psr7\Response;
use GuzzleHttp\Psr7\ServerRequest;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

/**
 * Un middleware qui vérifie une clé API et ajoute un en-tête à la réponse.
 * Il ne dépend que des interfaces PSR-7.
 */
function apiAuthMiddleware(ServerRequestInterface $request): ResponseInterface
{
    // 1. On crée une réponse de base
    $response = new Response();

    // 2. On lit une information de la requête
    $apiKey = $request->getHeaderLine('X-API-KEY');

    if ($apiKey !== 'secret-key') {
        // 3. On utilise l'immutabilité pour créer une réponse d'erreur
        $errorResponse = $response
            ->withStatus(401)
            ->withHeader('Content-Type', 'application/json');
        
        $errorResponse->getBody()->write(json_encode(['error' => 'Unauthorized']));

        return $errorResponse;
    }

    // 4. On crée une réponse de succès
    $successResponse = $response
        ->withStatus(200)
        ->withHeader('Content-Type', 'application/json')
        ->withHeader('X-Powered-By', 'PSR-7-Middleware'); // Ajout d'un en-tête
    
    $successResponse->getBody()->write(json_encode(['data' => 'Welcome, authenticated user!']));

    return $successResponse;
}

// --- Simulation d'une requête entrante ---
// a) Requête invalide
echo "Test avec une clé API invalide...\n";
$invalidRequest = ServerRequest::fromGlobals()->withHeader('X-API-KEY', 'wrong-key');
$response1 = apiAuthMiddleware($invalidRequest);
echo "Status: " . $response1->getStatusCode() . "\n";
echo "Body: " . $response1->getBody() . "\n\n";

// b) Requête valide
echo "Test avec une clé API valide...\n";
$validRequest = ServerRequest::fromGlobals()->withHeader('X-API-KEY', 'secret-key');
$response2 = apiAuthMiddleware($validRequest);
echo "Status: " . $response2->getStatusCode() . "\n";
echo "Header 'X-Powered-By': " . $response2->getHeaderLine('X-Powered-By') . "\n";
echo "Body: " . $response2->getBody() . "\n";
```

## En Bref

| Concept | Description |
| :--- | :--- |
| **`RequestInterface`** | Message pour une requête HTTP sortante (ex: un client API). |
| **`ServerRequestInterface`** | Message pour une requête HTTP entrante (ce que le serveur reçoit). |
| **`ResponseInterface`** | Message pour une réponse HTTP. |
| **Immutabilité** | Les objets ne peuvent pas être modifiés. Les méthodes `with...()` retournent un clone. |
| **`StreamInterface`** | Représente le corps du message pour gérer efficacement la mémoire. |
| **Interopérabilité** | L'objectif principal : permettre aux composants (middlewares...) de fonctionner avec n'importe quel framework. |
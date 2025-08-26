# PSR-15 : HTTP Handlers

## L'Objectif : Standardiser le traitement des requêtes HTTP en pipeline

Comment une application web moderne traite-t-elle une requête entrante ? Elle ne le fait pas en une seule fois. La requête passe par une série d'étapes :
-  Routing (trouver le bon contrôleur).
-  Authentification (vérifier si l'utilisateur est connecté).
- Validation des données.
- Exécution de la logique métier (le contrôleur).

La PSR-15 standardise ce processus en définissant une architecture de **middlewares**. Un middleware est un maillon d'une chaîne. Chaque middleware reçoit une requête, peut effectuer une action (logger, vérifier une permission, etc.), puis **délègue** la requête au maillon suivant.

Pensez à un oignon : la requête traverse chaque couche (middleware) pour atteindre le cœur (la logique métier), puis la réponse remonte à travers les mêmes couches dans l'ordre inverse.

**Le bénéfice :** La PSR-15 crée un écosystème de middlewares réutilisables. Un middleware d'authentification, de cache, ou de gestion des CORS écrit selon la PSR-15 peut être utilisé dans n'importe quel framework compatible (Slim, Mezzio, etc.) sans aucune modification.

## Les Deux Interfaces Clés

La PSR-15 est construite sur la [PSR-7](./psr-7-http-message-interface.md) et définit deux rôles principaux :

### 1. `RequestHandlerInterface` (Le Handler)
Un *Request Handler* (ou gestionnaire de requête) représente le **point final** d'une chaîne de traitement. C'est typiquement votre contrôleur ou votre action métier. Son travail est simple : prendre une requête et retourner une réponse.

```php
namespace Psr\Http\Server;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\ResponseInterface;

interface RequestHandlerInterface
{
    public function handle(ServerRequestInterface $request): ResponseInterface;
}
```

### 2. `MiddlewareInterface` (Le Middleware)
Un *Middleware* est une couche de traitement. Sa particularité est qu'il reçoit non seulement la requête, mais aussi le **gestionnaire suivant** dans la chaîne. Son travail consiste à traiter la requête, puis à appeler le gestionnaire suivant pour continuer le processus.

```php
namespace Psr\Http\Server;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\ResponseInterface;

interface MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface;
}
```
**Le point crucial :** À l'intérieur de la méthode `process()`, le middleware **DOIT** appeler `$handler->handle($request)` pour passer la main. S'il ne le fait pas, la chaîne est interrompue (ce qui est utile pour un middleware d'authentification qui bloque une requête, par exemple).

## Cas d'Usage : Une chaîne de middlewares (Logging + Auth)

### Scénario
Nous allons créer un pipeline simple :
1.  Un `LoggingMiddleware` qui loggue l'URL de la requête.
2.  Un `AuthMiddleware` qui vérifie un en-tête `Authorization`.
3.  Un `DefaultHandler` qui est notre logique métier finale.

**1. Les middlewares et le handler**
```php
<?php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use GuzzleHttp\Psr7\Response;

// Middleware 1: Logging
class LoggingMiddleware implements MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        echo "LOG: Requête reçue pour " . $request->getUri()->getPath() . "\n";
        // On délègue au handler suivant
        return $handler->handle($request);
    }
}

// Middleware 2: Authentification
class AuthMiddleware implements MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        if ($request->getHeaderLine('Authorization') !== 'Bearer secret-token') {
            echo "AUTH: Accès refusé.\n";
            // On interrompt la chaîne et on retourne une réponse d'erreur
            return new Response(401);
        }
        echo "AUTH: Accès autorisé.\n";
        return $handler->handle($request);
    }
}

// Handler final: Notre logique métier
class DefaultHandler implements RequestHandlerInterface
{
    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        echo "ACTION: Exécution de la logique métier.\n";
        $response = new Response(200);
        $response->getBody()->write('<h1>Hello World!</h1>');
        return $response;
    }
}
```

**2. Le "Runner" qui exécute la chaîne**
Un framework fournit ce "runner". Pour notre exemple, nous le simulons.
```php
<?php
require 'vendor/autoload.php';
use GuzzleHttp\Psr7\ServerRequest;

// ... (coller les 3 classes ci-dessus) ...

// Notre pile de middlewares
$middlewares = [
    new LoggingMiddleware(),
    new AuthMiddleware(),
];
$finalHandler = new DefaultHandler();

// Fonction qui exécute la pile
function run(array $middlewares, ServerRequestInterface $request, RequestHandlerInterface $finalHandler): ResponseInterface
{
    $middleware = array_shift($middlewares);
    if ($middleware === null) {
        return $finalHandler->handle($request);
    }
    
    // Le prochain "handler" est une fonction qui appellera le middleware suivant
    $next = new class($middlewares, $request, $finalHandler) implements RequestHandlerInterface {
        public function __construct(private array $middlewares, private ServerRequestInterface $request, private RequestHandlerInterface $finalHandler) {}
        public function handle(ServerRequestInterface $request): ResponseInterface { return run($this->middlewares, $request, $this->finalHandler); }
    };

    return $middleware->process($request, $next);
}

// --- Test 1 : Requête invalide ---
echo "--- TEST 1 : Mauvais token ---\n";
$request1 = new ServerRequest('GET', '/');
$response1 = run($middlewares, $request1, $finalHandler);
echo "Réponse: " . $response1->getStatusCode() . "\n\n";

// --- Test 2 : Requête valide ---
echo "--- TEST 2 : Bon token ---\n";
$request2 = (new ServerRequest('GET', '/'))->withHeader('Authorization', 'Bearer secret-token');
$response2 = run($middlewares, $request2, $finalHandler);
echo "Réponse: " . $response2->getStatusCode() . " - " . $response2->getBody() . "\n";
```

**Résultat de l'exécution :**
```
--- TEST 1 : Mauvais token ---
LOG: Requête reçue pour /
AUTH: Accès refusé.
Réponse: 401

--- TEST 2 : Bon token ---
LOG: Requête reçue pour /
AUTH: Accès autorisé.
ACTION: Exécution de la logique métier.
Réponse: 200 - <h1>Hello World!</h1>
```

## En Bref

| Concept | Description |
| :--- | :--- |
| **`RequestHandlerInterface`**| Représente une action finale qui prend une requête et retourne une réponse. |
| **`MiddlewareInterface`** | Représente une couche de traitement qui prend une requête **et** le handler suivant, puis retourne une réponse. |
| **Pipeline (Chaîne)** | L'architecture où une requête traverse une série de middlewares avant d'atteindre le handler final. |
| **`process()`** | La méthode du middleware qui contient la logique et **délègue** l'appel au handler suivant. |
| **`handle()`** | La méthode du handler qui exécute la logique finale. |
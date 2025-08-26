# PSR-17 : HTTP Factories

## L'Objectif : Un moyen standard de créer des objets PSR-7

La [PSR-7](./psr-7-http-message-interface.md) a standardisé les **interfaces** pour les objets `Request`, `Response`, `Uri`, etc. Mais elle n'a pas spécifié **comment créer** ces objets.

Chaque bibliothèque qui implémentait la PSR-7 avait donc sa propre manière de les instancier :
-   `new GuzzleHttp\Psr7\Request(...)`
-   `new Laminas\Diactoros\Response(...)`
-   `new Slim\Psr7\Request\RequestCreator(...)`

Ce n'était pas un problème tant qu'une application était liée à une seule implémentation. Mais cela rendait difficile la création de code (comme des middlewares ou des adaptateurs de framework) qui devait être agnostique de l'implémentation. Si un middleware voulait créer une nouvelle réponse, il devait savoir quelle bibliothèque concrète était utilisée pour l'instancier.

La PSR-17 résout ce problème en définissant un ensemble d'**interfaces de factory**. Une factory est un objet dont le seul but est de créer d'autres objets.

**Le bénéfice :** Un composant (middleware, contrôleur...) qui a besoin de créer un objet PSR-7 (par exemple, une `Response`) n'a plus besoin de connaître `Guzzle`, `Laminas` ou `Slim`. Il demande simplement une `ResponseFactoryInterface` (via l'injection de dépendances) et l'utilise pour créer une réponse. Le composant devient ainsi totalement découplé de l'implémentation PSR-7 concrète.

## Les Interfaces de Factory

La PSR-17 définit une factory pour chaque type d'objet PSR-7.

-   `RequestFactoryInterface`
    -   `createRequest(string $method, $uri): RequestInterface`

-   `ResponseFactoryInterface`
    -   `createResponse(int $code = 200, string $reasonPhrase = ''): ResponseInterface`

-   `ServerRequestFactoryInterface`
    -   `createServerRequest(string $method, $uri, array $serverParams = []): ServerRequestInterface`

-   `StreamFactoryInterface`
    -   `createStream(string $content = ''): StreamInterface`
    -   `createStreamFromFile(string $filename, string $mode = 'r'): StreamInterface`

-   `UriFactoryInterface`
    -   `createUri(string $uri = ''): UriInterface`

-   `UploadedFileFactoryInterface`
    -   `createUploadedFile(...): UploadedFileInterface`

## Cas d'Usage : Un middleware qui crée des réponses

### Scénario
Nous reprenons notre middleware d'authentification de l'exemple PSR-15. Auparavant, il était obligé de faire `new GuzzleHttp\Psr7\Response()`, ce qui le liait à la bibliothèque Guzzle. Nous allons le réécrire pour qu'il soit totalement agnostique en utilisant une `ResponseFactoryInterface`.

**1. Le middleware mis à jour**
Il dépend maintenant de `ResponseFactoryInterface` pour créer ses réponses.
```php
<?php
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class AgnosticAuthMiddleware implements MiddlewareInterface
{
    private ResponseFactoryInterface $responseFactory;

    public function __construct(ResponseFactoryInterface $responseFactory)
    {
        $this->responseFactory = $responseFactory;
    }

    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        if ($request->getHeaderLine('Authorization') !== 'Bearer secret-token') {
            // On utilise la factory pour créer la réponse d'erreur
            $response = $this->responseFactory->createResponse(401);
            $response->getBody()->write('Unauthorized');
            return $response;
        }

        return $handler->handle($request);
    }
}
```
Ce middleware est maintenant 100% réutilisable. Il n'a aucune idée de quelle implémentation de PSR-7 est utilisée.

**2. Le code qui assemble le tout**
Le "framework" (ou notre point d'entrée) est responsable de fournir l'implémentation concrète de la factory.

```php
<?php
require 'vendor/autoload.php';

// --- On choisit une implémentation. Par exemple, Guzzle. ---
use GuzzleHttp\Psr7\HttpFactory;

// ... (coller la classe AgnosticAuthMiddleware) ...

// ... (Simulons un RequestHandler final) ...
class MyFinalHandler implements RequestHandlerInterface {
    public function __construct(private ResponseFactoryInterface $factory) {}
    public function handle(ServerRequestInterface $request): ResponseInterface {
        $response = $this->factory->createResponse(200);
        $response->getBody()->write('Welcome!');
        return $response;
    }
}


// 1. On instancie la factory concrète de Guzzle.
// Cette factory implémente TOUTES les interfaces de la PSR-17.
$httpFactory = new HttpFactory();

// 2. On injecte cette factory dans notre middleware et notre handler.
$middleware = new AgnosticAuthMiddleware($httpFactory);
$handler = new MyFinalHandler($httpFactory);

// --- Simulation de l'exécution ---
// On crée un "runner" de middleware simple (non standard)
$runner = function (ServerRequestInterface $request) use ($middleware, $handler) {
    return $middleware->process($request, $handler);
};


// a) Test avec une requête invalide
use GuzzleHttp\Psr7\ServerRequest;
$request1 = new ServerRequest('GET', '/protected');
$response1 = $runner($request1);

echo "Test 1 (invalide) - Status: " . $response1->getStatusCode() . "\n";
echo "Test 1 (invalide) - Body: " . $response1->getBody() . "\n";
// L'objet $response1 est une instance de GuzzleHttp\Psr7\Response

// b) Test avec une requête valide
$request2 = (new ServerRequest('GET', '/protected'))->withHeader('Authorization', 'Bearer secret-token');
$response2 = $runner($request2);
echo "Test 2 (valide) - Status: " . $response2->getStatusCode() . "\n";
echo "Test 2 (valide) - Body: " . $response2->getBody() . "\n";

```
Si demain nous décidons de remplacer Guzzle par `Laminas\Diactoros`, nous n'aurions **qu'à changer la ligne `new HttpFactory()`** par l'équivalent de Laminas. Le middleware `AgnosticAuthMiddleware` n'aurait pas besoin d'être modifié.

## En Bref

| Concept | Description |
| :--- | :--- |
| **Factory** | Un objet dont le rôle est de créer d'autres objets. |
| **`RequestFactoryInterface`** | Interface standard pour créer des objets `RequestInterface`. |
| **`ResponseFactoryInterface`** | Interface standard pour créer des objets `ResponseInterface`. |
| **Découplage** | L'objectif principal : permettre à du code (middlewares, etc.) de créer des objets PSR-7 sans être lié à une implémentation concrète (Guzzle, Laminas...). |
| **PSR-7 + PSR-17** | La PSR-7 définit les interfaces des messages, la PSR-17 définit les interfaces pour les créer. Elles travaillent main dans la main. |
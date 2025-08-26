# PSR-18 : HTTP Client

## L'Objectif : Une manière standard d'envoyer des requêtes HTTP

Faire des appels à des API externes est une tâche extrêmement courante pour une application web (appeler une API de paiement, récupérer des données météo, etc.).

Le problème, encore une fois, était que chaque client HTTP avait sa propre API :
-   Guzzle avait `$client->request('GET', ...)`
-   Le client HTTP de Symfony avait `$client->request('GET', ...)` mais avec une signature différente.
-   Les wrappers de `curl` avaient leurs propres fonctions.

Si vous développiez une bibliothèque (par exemple, un SDK pour une API) qui devait faire des appels HTTP, vous étiez obligé de la lier à une implémentation spécifique (souvent Guzzle), forçant ainsi tous les utilisateurs de votre bibliothèque à utiliser Guzzle.

La PSR-18 résout ce problème en définissant une **interface unique et simple pour un client HTTP**.

**Le bénéfice :** Une bibliothèque qui a besoin d'envoyer une requête HTTP peut maintenant dépendre de l'interface `ClientInterface`. Elle devient compatible avec n'importe quel client HTTP qui implémente cette interface (Guzzle, Symfony HTTP Client, etc.). L'utilisateur de la bibliothèque peut choisir le client HTTP de son choix.

## Les Interfaces Clés

La PSR-18 est bâtie sur la [PSR-7](./psr-7-http-message-interface.md) et est volontairement très simple.

### 1. `Psr\Http\Client\ClientInterface`
C'est la seule interface principale. Elle ne contient qu'une seule méthode.

```php
namespace Psr\Http\Client;

use Psr\Http\Message\RequestInterface;
use Psr\Http\Message\ResponseInterface;

interface ClientInterface
{
    /**
     * Envoie une requête PSR-7 et retourne une réponse PSR-7.
     * @param RequestInterface $request
     * @return ResponseInterface
     * @throws \Psr\Http\Client\ClientExceptionInterface Si une erreur se produit.
     */
    public function sendRequest(RequestInterface $request): ResponseInterface;
}
```
-   **`sendRequest(RequestInterface $request)`** : C'est tout ! La méthode prend un objet `RequestInterface` (défini par la PSR-7) et retourne un objet `ResponseInterface` (défini par la PSR-7).

### 2. Les Exceptions
La PSR-18 définit également une hiérarchie d'exceptions pour gérer les erreurs de manière standard.
-   `ClientExceptionInterface` : L'interface de base pour toutes les exceptions levées par le client.
-   `NetworkExceptionInterface` : Pour les erreurs réseau (ex: DNS, timeout, connexion refusée).
-   `RequestExceptionInterface` : Pour les erreurs liées à la requête elle-même (ex: méthode invalide, URL malformée).

## Cas d'Usage : Un SDK d'API météo

### Scénario
Nous allons créer une petite classe `WeatherApiClient` qui récupère la météo depuis une API externe. Cette classe sera complètement agnostique du client HTTP utilisé. Elle ne dépendra que des interfaces PSR-18 et PSR-17.

**1. La classe `WeatherApiClient`**
Elle dépend de `ClientInterface` pour envoyer les requêtes et de `RequestFactoryInterface` (PSR-17) pour les créer.
```php
<?php
use Psr\Http\Client\ClientInterface;
use Psr\Http\Message\RequestFactoryInterface;

class WeatherApiClient
{
    private ClientInterface $httpClient;
    private RequestFactoryInterface $requestFactory;
    private string $apiKey;

    public function __construct(
        ClientInterface $httpClient,
        RequestFactoryInterface $requestFactory,
        string $apiKey
    ) {
        $this->httpClient = $httpClient;
        $this->requestFactory = $requestFactory;
        $this->apiKey = $apiKey;
    }

    public function getTemperatureForCity(string $city): float
    {
        $uri = "https://api.openweathermap.org/data/2.5/weather?q={$city}&appid={$this->apiKey}&units=metric";

        // 1. On utilise la factory PSR-17 pour créer la requête PSR-7
        $request = $this->requestFactory->createRequest('GET', $uri);

        try {
            // 2. On utilise le client PSR-18 pour envoyer la requête
            $response = $this->httpClient->sendRequest($request);
        } catch (\Psr\Http\Client\ClientExceptionInterface $e) {
            // Gérer les erreurs réseau ou de requête
            throw new \RuntimeException("Impossible de contacter l'API météo.", 0, $e);
        }
        
        if ($response->getStatusCode() !== 200) {
            throw new \RuntimeException("L'API météo a retourné une erreur : " . $response->getReasonPhrase());
        }

        $data = json_decode($response->getBody()->getContents(), true);
        
        return $data['main']['temp'];
    }
}
```
Cette classe est parfaitement découplée et réutilisable.

**2. Le code qui assemble et utilise le SDK**
Ici, nous choisissons d'utiliser **Guzzle** comme implémentation concrète.
```php
<?php
require 'vendor/autoload.php';

// --- On choisit une implémentation. ---
// Guzzle 7+ implémente nativement PSR-18.
use GuzzleHttp\Client as GuzzleClient; 
// La factory de Guzzle implémente PSR-17.
use GuzzleHttp\Psr7\HttpFactory;

// ... (coller la classe WeatherApiClient) ...

// 1. On instancie les dépendances concrètes.
$guzzleClient = new GuzzleClient();
$httpFactory = new HttpFactory();
$apiKey = 'VOTRE_VRAIE_CLE_API_OPENWEATHERMAP'; // A remplacer

// 2. On injecte les implémentations dans notre SDK.
// Notre classe ne voit que les interfaces PSR-18 et PSR-17.
$weatherClient = new WeatherApiClient($guzzleClient, $httpFactory, $apiKey);

// 3. On utilise le SDK.
try {
    $temperature = $weatherClient->getTemperatureForCity('Paris');
    echo "Il fait actuellement {$temperature}°C à Paris.\n";
} catch (\RuntimeException $e) {
    echo "Erreur : " . $e->getMessage() . "\n";
}
```
Si nous voulions passer au client HTTP de Symfony, nous n'aurions qu'à changer les deux lignes d'instanciation de `$guzzleClient` et `$httpFactory`. Le code de `WeatherApiClient` resterait **inchangé**.

## En Bref

| Concept | Description |
| :--- | :--- |
| **`ClientInterface`** | L'interface standard pour un client HTTP. |
| **`sendRequest()`** | La seule méthode, qui prend une `RequestInterface` (PSR-7) et retourne une `ResponseInterface` (PSR-7). |
| **Exceptions** | Un jeu d'interfaces d'exception standard (`ClientExceptionInterface`...) pour gérer les erreurs. |
| **Interopérabilité** | L'objectif principal : permettre de créer des bibliothèques (SDKs...) qui font des appels HTTP sans être liées à un client spécifique (Guzzle, Symfony...). |
| **Écosystème HTTP** | PSR-7 (Messages) + PSR-17 (Factories) + PSR-18 (Client) forment un trio complet pour gérer les interactions HTTP en PHP de manière standard. |
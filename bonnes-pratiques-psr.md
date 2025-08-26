# Bonnes Pratiques PSR

## L'Objectif : Adopter les PSR efficacement dans vos projets

Connaître les standards PSR est une chose, les intégrer de manière fluide dans son travail quotidien en est une autre. Cette fiche a pour but de fournir des conseils pratiques et des outils pour faire des PSR une partie intégrante de votre processus de développement, que vous démarriez un nouveau projet ou que vous travailliez sur un projet existant.

L'adoption des PSR n'est pas seulement une question de conformité, mais une démarche pour améliorer la qualité, la collaboration et la pérennité de votre code.

## 1. Outillage : Laissez les robots faire le travail

Personne ne devrait vérifier la conformité à la PSR-12 à la main. L'écosystème PHP fournit des outils exceptionnels pour automatiser la vérification et la correction du style de code.

-   **PHP-CS-Fixer (`friendsofphp/php-cs-fixer`)**
    -   **Quoi ?** Un outil qui **analyse et corrige automatiquement** votre code pour le rendre conforme à un standard donné (PSR-1, PSR-12, et bien d'autres).
    -   **Comment ?** Vous le configurez une seule fois dans votre projet (avec un fichier `.php-cs-fixer.dist.php`) en spécifiant que vous voulez suivre la règle `@PSR12`. Ensuite, une simple commande (`php-cs-fixer fix src`) reformate tout votre code.
    -   **Pourquoi ?** C'est l'outil le plus simple pour garantir une base de code uniforme sans effort.

-   **PHP_CodeSniffer (`squizlabs/php_codesniffer`)**
    -   **Quoi ?** Un outil qui **détecte les violations** des standards de code. Il est souvent utilisé pour la vérification (linting) dans les éditeurs de code ou dans les pipelines d'intégration continue. Il peut aussi tenter de corriger les erreurs.
    -   **Comment ?** On le configure avec un fichier `phpcs.xml` en lui disant d'utiliser le standard `PSR12`. La commande `phpcs src` listera toutes les erreurs.
    -   **Pourquoi ?** Excellent pour l'intégration continue (CI) afin de rejeter le code qui ne respecte pas les standards.

**Conseil :** Intégrez ces outils à votre éditeur de code (VS Code, PhpStorm...) pour obtenir un retour en temps réel et formater automatiquement votre code à chaque sauvegarde.

## 2. Composer : Le pilier de la PSR-4

La PSR-4 (Autoloading) est au cœur de tout projet PHP moderne. **Composer** est l'outil qui la met en œuvre.
-   **Structurez vos projets :** Adoptez dès le départ une structure claire en mappant vos namespaces à des répertoires dans `composer.json`. La convention la plus répandue est de mapper le namespace principal de votre application (ex: `App\`) au répertoire `src/`.
    ```json
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
    ```
-   **Pensez `dump-autoload` :** Après chaque modification de la section `autoload` de votre `composer.json`, n'oubliez pas d'exécuter `composer dump-autoload` (ou `composer dump-autoload -o` pour une optimisation en production).

## 3. Penser en termes d'Interfaces (Le cœur des PSR d'interopérabilité)

Les PSR-3, 6, 7, 11, 13, 14, 15, 16, 17, 18 et 20 sont des **PSR d'interfaces**. Leur but est le **découplage**.
-   **Injectez des interfaces, pas des classes concrètes :** C'est le principe fondamental de l'injection de dépendances. Votre code métier ne doit pas savoir quelle implémentation est utilisée.

    ✅ **Correct (Découplé) :**
    ```php
    use Psr\Log\LoggerInterface;

    class MyService {
        public function __construct(private LoggerInterface $logger) {}
    }
    ```
    
    ❌ **Incorrect (Couplé) :**
    ```php
    use Monolog\Logger;

    class MyService {
        public function __construct(private Logger $logger) {}
    }
    ```

-   **Utilisez un Conteneur d'Injection de Dépendances :** Pour gérer ces abstractions, utilisez un conteneur (comme celui de Symfony, Laravel, ou PHP-DI) qui est compatible PSR-11. Vous configurez le conteneur pour qu'il sache que lorsqu'on demande une `LoggerInterface`, il doit fournir une instance de `Monolog`.

## 4. Intégration Continue (CI)

Votre pipeline de CI (GitHub Actions, GitLab CI...) est le gardien de la qualité de votre code. Il doit comporter au minimum deux étapes liées aux PSR :
1.  **Vérification du style de code :** Exécutez `php-cs-fixer fix --dry-run` ou `phpcs`. Si l'outil détecte des violations, le pipeline doit échouer. Cela force les développeurs à soumettre du code propre.
2.  **Analyse statique :** Des outils comme `PHPStan` ou `Psalm` peuvent détecter des erreurs de types et d'autres problèmes. Bien que ce ne soit pas une PSR, cette pratique s'aligne parfaitement avec l'objectif de robustesse du code promu par les PSR.

## 5. Comment aborder un projet existant ?

Appliquer les PSR à une base de code ancienne peut être intimidant.
1.  **Commencez par la PSR-12 :** C'est la plus facile à intégrer. Lancez `PHP-CS-Fixer` sur votre projet. Le gain en lisibilité sera immédiat.
2.  **Adoptez la PSR-4 :** Si le projet utilise encore des `require_once`, c'est la migration la plus importante. Mettez en place Composer, configurez l'autoloading, et remplacez progressivement les inclusions manuelles par des `use`.
3.  **Introduisez les interfaces progressivement :** Ne réécrivez pas tout d'un coup. La prochaine fois que vous travaillez sur une partie du code qui fait du logging, remplacez le logger propriétaire par une dépendance à `psr/log`. Faites de même pour le cache, les appels HTTP, etc., au fur et à mesure des besoins.

## En Bref

| Pratique | Outils / Concepts | Bénéfice |
| :--- | :--- | :--- |
| **Automatiser le style** | PHP-CS-Fixer, PHP_CodeSniffer | Cohérence du code sans effort, gain de temps en revue de code. |
| **Gérer l'autoloading** | Composer, `composer.json` | Structure de projet claire, chargement des classes standardisé. |
| **Découpler le code** | Injection de dépendances, Conteneur DI (PSR-11) | Interopérabilité, testabilité, flexibilité. |
| **Garantir la qualité** | Intégration Continue (CI) | Empêcher le code non conforme d'être fusionné. |
| **Moderniser l'existant** | Approche incrémentale | Améliorer la qualité de la base de code sans tout réécrire. |
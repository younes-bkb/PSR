# PSR-12 : Extended Coding Style Guide

## L'Objectif : Un style de code unifié et professionnel

Alors que la [PSR-1](./psr-1-basic-coding-standard.md) pose les règles de base, la PSR-12 va beaucoup plus loin en définissant un **guide de style complet** pour le code PHP. Elle dicte comment formater le code : où placer les accolades, combien d'espaces utiliser, comment aligner les arguments, etc.

**Important :** La PSR-12 **remplace et étend** la PSR-2. Elle inclut également toutes les règles de la PSR-1. Un code conforme à la PSR-12 est donc automatiquement conforme à la PSR-1.

L'objectif est d'avoir un code qui semble avoir été écrit par une seule personne, même sur des projets avec des dizaines de contributeurs. Cela réduit la "charge cognitive" lors de la lecture du code et facilite la collaboration.

## Les Règles Clés (Sélection)

La PSR-12 est très détaillée. Voici une sélection des règles les plus importantes et les plus visibles.

### 1. Général

-   **Encodage et balises :** Toutes les règles de la PSR-1 s'appliquent (UTF-8 sans BOM, pas de `?>` en fin de fichier).
-   **Indentation :** Utiliser **4 espaces** pour l'indentation, pas de tabulations.
-   **Fin de ligne :** Utiliser des fins de ligne Unix (LF), pas Windows (CRLF).
-   **Lignes :** Pas de limite stricte de longueur de ligne, mais une limite "douce" est encouragée à 120 caractères. Les lignes ne **DOIVENT PAS** se terminer par des espaces blancs.

### 2. Déclarations `use`

-   Il **DOIT** y avoir une ligne vide après la déclaration `namespace`.
-   Toutes les déclarations `use` **DOIVENT** être placées après la déclaration `namespace`.
-   Il **DOIT** y avoir une seule instruction `use` par déclaration.
-   Les `use` **DOIVENT** être groupés par type : d'abord les classes/interfaces/traits, puis les fonctions, et enfin les constantes.

✅ **Correct :**
```php
<?php
namespace Vendor\Package;

use Another\Vendor\ClassA;
use My\Project\ClassB as B; // L'alias est permis
use My\Project\ClassC;
use function My\Project\someFunction;
use const My\Project\SOME_CONSTANT;

class MaClasse
{
    // ...
}
```

### 3. Classes, Propriétés et Méthodes

-   L'accolade ouvrante d'une classe **DOIT** être sur la ligne suivante, et l'accolade fermante **DOIT** être sur la ligne suivante après le corps de la classe.
-   Les mots-clés `extends` et `implements` **DOIVENT** être sur la même ligne que le nom de la classe.
-   La visibilité (`public`, `protected`, `private`) **DOIT** être déclarée sur toutes les propriétés et méthodes.
-   L'ordre de visibilité des méthodes et propriétés est laissé au choix du développeur (contrairement à PSR-2 qui l'imposait).

✅ **Correct :**
```php
<?php
namespace Vendor\Package;

class MaClasse extends ParentClass implements MonInterface
{
    public const UNE_CONSTANTE = 1;

    private string $propriete;

    public function __construct(string $propriete)
    {
        $this->propriete = $propriete;
    }
}
```

### 4. Structures de Contrôle (`if`, `for`, `foreach`, `switch`...)

-   Il **DOIT** y avoir un espace après le mot-clé de la structure de contrôle.
-   Il **NE DOIT PAS** y avoir d'espace après la parenthèse ouvrante, ni avant la parenthèse fermante.
-   L'accolade ouvrante **DOIT** être sur la même ligne.
-   L'accolade fermante **DOIT** être sur la ligne suivante après le corps.
-   Utiliser `elseif` et non `else if`.

✅ **Correct :**
```php
if ($condition1) {
    // action A
} elseif ($condition2) {
    // action B
} else {
    // action C
}
```

❌ **Incorrect :**
```php
if($condition1){ // Manque d'espaces, accolade mal placée
    // action A
}
else if ($condition2) // "else if" interdit, accolades manquantes
{
    // action B
}
```

### 5. Appels de Méthodes et Fonctions (Arguments sur plusieurs lignes)

C'est l'un des changements les plus visibles par rapport à PSR-2.
-   Si les arguments sont répartis sur plusieurs lignes, le premier argument **DOIT** être sur la ligne suivante.
-   Chaque argument suivant **DOIT** être sur sa propre ligne.
-   La parenthèse fermante et l'accolade ouvrante **DOIVENT** être ensemble sur leur propre ligne.

✅ **Correct :**
```php
$resultat = maFonction(
    $premierArgumentTresLong,
    $deuxiemeArgument,
    $troisiemeArgument
);
```

## Un Exemple Complet

Voici une classe qui respecte la PSR-12 et met en évidence plusieurs règles.
```php
<?php
declare(strict_types=1);

namespace App\Service;

use Psr\Log\LoggerInterface;
use App\Entity\User;

class UserRegistration
{
    private LoggerInterface $logger;
    private const MIN_PASSWORD_LENGTH = 8;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function register(string $email, string $password): User
    {
        if (strlen($password) < self::MIN_PASSWORD_LENGTH) {
            $this->logger->error(
                'Mot de passe trop court pour l\'utilisateur {email}',
                ['email' => $email]
            );
        }

        // ... logique de création de l'utilisateur

        return new User($email);
    }
}
```

## En Bref

| Catégorie | Règle Principale |
| :--- | :--- |
| **Général** | Indentation avec 4 espaces, fins de ligne LF, pas de `?>` en fin de fichier. |
| **`use` Statements** | Un par ligne, groupés (classes, fonctions, constantes), après `namespace`. |
| **Classes** | Accolade ouvrante sur la ligne suivante. `extends` et `implements` sur la même ligne. |
| **Propriétés/Méthodes** | Visibilité obligatoire (`public`, `protected`, `private`). |
| **Structures de Contrôle** | Espace après mot-clé, accolades sur la même ligne (`{`) et sur une nouvelle ligne (`}`). |
| **Appels multi-lignes**| Le premier argument sur la ligne suivante, un argument par ligne, tout est indenté. |
# PSR-1 : Basic Coding Standard

## L'Objectif : Les fondations indispensables

La PSR-1 est le point d'entrée des standards de code PHP. Elle ne s'occupe pas de style (espaces, accolades...), mais définit 5 règles fondamentales et absolues pour garantir une **interopérabilité technique de haut niveau** entre des codes PHP partagés.

Un code qui ne respecte pas la PSR-1 ne peut pas être considéré comme conforme aux standards modernes de la communauté.

## Les 5 Règles Clés

### 1. Balises PHP

-   Les fichiers PHP **DOIVENT** utiliser les balises `<?php` ou `<?=`.
-   Les autres variations (`<?`, `<%`, etc.) sont interdites.
-   Pour les fichiers ne contenant **que du code PHP**, la balise de fermeture `?>` **DOIT** être omise.

**Pourquoi omettre `?>` ?**
Cela évite des erreurs difficiles à déboguer d'espaces blancs ou de lignes vides accidentellement envoyés dans la réponse HTTP (par exemple, après la balise de fermeture), ce qui peut perturber les en-têtes HTTP, les sessions ou les flux XML.

✅ **Correct :**
```php
<?php
namespace MonProjet;

class MaClasse
{
    // ...
}

// Pas de balise de fermeture à la fin du fichier
```

❌ **Incorrect :**
```php
<?php
namespace MonProjet;

class MaClasse
{
    // ...
}
?> // <-- INTERDIT, car le fichier ne contient que du PHP
```

### 2. Encodage des caractères

-   Tous les fichiers PHP **DOIVENT** utiliser l'encodage de caractères **UTF-8 sans BOM** (Byte Order Mark).

**Pourquoi sans BOM ?**
Le BOM est un caractère invisible au début du fichier qui peut causer des problèmes similaires à ceux de la balise `?>` (espaces blancs inattendus), perturbant la sortie de l'application. La plupart des éditeurs de code modernes permettent de configurer l'encodage par défaut.

### 3. Effets de bord (Side Effects)

C'est la règle la plus conceptuelle : un fichier PHP **DEVRAIT** soit **déclarer des symboles** (classes, fonctions, constantes), soit **exécuter une logique avec des effets de bord**, mais pas les deux en même temps.

-   **Déclarer des symboles :** Un fichier qui contient une classe, une interface, un trait, une fonction.
-   **Effets de bord :** Un code qui exécute une action : générer du HTML, modifier une configuration (`ini_set`), lire un fichier, se connecter à une base de données, etc.

✅ **Correct : Un fichier pour la déclaration** (`MonProjet/MaClasse.php`)
```php
<?php
// Fichier qui ne fait que déclarer une classe.
namespace MonProjet;

class MaClasse
{
    public function faireQuelqueChose()
    {
        return 'Hello';
    }
}
```

✅ **Correct : Un fichier pour l'exécution** (`public/index.php`)
```php
<?php
// Fichier qui exécute du code.
require __DIR__ . '/../vendor/autoload.php';

$objet = new MonProjet\MaClasse();
echo $objet->faireQuelqueChose(); // <-- Effet de bord (génère un output)
```

❌ **Incorrect : Mélanger les deux**
```php
<?php
// Fichier qui déclare ET exécute.
namespace MonProjet;

class MaClasse
{
    // ...
}

// Effet de bord dans le même fichier que la déclaration.
$objet = new MaClasse();
echo $objet->faireQuelqueChose();
```

### 4. Namespaces et Noms de Classes

-   Chaque classe **DOIT** être dans un namespace d'au moins un niveau (un *vendor namespace*).
-   Les noms de classes **DOIVENT** être déclarés en `StudlyCaps` (aussi appelé `PascalCase`).

✅ **Correct :**
```php
<?php
namespace Vendor\MonProjet\Service;

class MonSuperService
{
    // ...
}
```

❌ **Incorrect :**
```php
<?php
// Pas de namespace
class ma_classe // Nom incorrect
{
    // ...
}
```

### 5. Constantes de classe, Propriétés et Méthodes

Cette règle s'applique à la casse (majuscules/minuscules) des membres d'une classe.

-   Les **constantes** de classe **DOIVENT** être déclarées en `UPPER_CASE` avec des séparateurs `_` (snake case majuscule).
-   Les **noms de méthodes** **DOIVENT** être déclarés en `camelCase`.

La PSR-1 ne se prononce pas sur les noms de propriétés, mais la convention générale (explicitée dans PSR-12) est d'utiliser `camelCase`.

✅ **Correct :**
```php
<?php
namespace Vendor\MonProjet;

class UneClasse
{
    public const VERSION_ACTUELLE = '1.0';

    public $unePropriete;

    public function uneMethodeSimple()
    {
        // ...
    }
}
```

❌ **Incorrect :**
```php
<?php
namespace Vendor\MonProjet;

class UneClasse
{
    public const VersionActuelle = '1.0'; // Mauvaise casse pour une constante

    public function Une_Methode_Simple() // Mauvaise casse pour une méthode
    {
        // ...
    }
}
```

## En Bref

| Règle | Description |
| :--- | :--- |
| **Balises** | Utiliser `<?php` ou `<?=`. Omettre `?>` si le fichier ne contient que du PHP. |
| **Encodage** | Utiliser l'UTF-8 sans BOM. |
| **Effets de bord** | Séparer la déclaration de symboles de l'exécution de code. |
| **Namespaces/Classes**| Classes dans un namespace, nommées en `StudlyCaps`. |
| **Membres de classe** | Constantes en `UPPER_CASE_SNAKE_CASE`, méthodes en `camelCase`. |
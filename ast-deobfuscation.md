---
layout: default
---

# Un Magicien ne se laisse jamais obfusquer

*Ou comment j'ai traversé les Mines de l'AST sans perdre mon chapeau.*

---

Je contemple l'énoncé du challenge avec la patience d'un Istari fatigué. HessKamai, un anti-bot, version 2.0. On me confie un AST -- un arbre syntaxique abstrait -- et on me demande de prouver que leur obfuscation est réversible. Un ami m'a envoyé l'arbre en question. Très bien. Un magicien n'est jamais en retard pour désobfusquer du JavaScript. Il arrive précisément quand le code l'exige.

## L'arbre qui cache la forêt

Je pointe mon navigateur vers `hesskamai-antibot.lab/challenge` et je récupère un fichier JSON de 36 Ko. Ce n'est pas du JavaScript lisible : c'est un **AST au format ESTree**, la représentation structurée d'un programme JS sous forme d'arbre. Chaque noeud décrit un élément du langage -- une déclaration, une expression, un opérateur. Le genre de chose qu'un compilateur digère, mais qu'un humain lit avec difficulté.

Un coup d'oeil à la structure globale : le `body` du programme contient quatre noeuds. Quatre. Voilà qui est gérable. Je retrousse mes manches grises et je reconstitue le code manuellement.

## Premier bloc : le Troll des cavernes

Le premier noeud est une IIFE -- une fonction immédiatement invoquée. Je la reconstruis :

```javascript
(() => {
    let d = [1856, 1824, 1776, 1728, 1776, 1728, 1776];
    d = d.map((c) => String.fromCharCode(c >> 4));
    console.log(d);
})();
```

Un bit-shift à droite de 4 sur chaque nombre, puis conversion en caractère. Je calcule : 1856 >> 4 = 116 = `t`, 1824 >> 4 = 114 = `r`, 1776 >> 4 = 111 = `o`...

Le résultat : **"trololo"**.

Je soupire longuement. Un troll. Littéralement. J'ai traversé le Pont de Khazad-dûm pour croiser un troll qui chante. Passons.

## Deuxième bloc : un clin d'oeil décoratif

```javascript
function l33t() {
    console.log("vive le reverse d'AST");
}
```

Jamais appelée. Aucune référence ailleurs dans l'arbre. C'est une fonction fantôme, un message laissé par l'auteur du challenge. Je hoche la tête -- on salue l'effort -- et je passe au bloc suivant sans m'attarder.

## Troisième bloc : le coeur du sensor

Voici `gen_sensor()`, la fonction qui compte. Je la reconstitue noeud par noeud :

```javascript
function gen_sensor() {
    let sens = [10] + [45] + [65] + [78] + [47];

    if ((sens >>= 4) == 20) {
        sens << 4;
    }

    let sensor = (function() {
        return [
            65353704, 65353663, 65353663, 65353707, 65353680,
            65353701, 65353663, 65353709, 65353680, 65353706,
            65353710, 65353724, 65353718, 65353680, 65353707,
            65353706, 65353696, 65353709, 65353705, 65353722,
            65353724, 65353708, 65353710, 65353723, 65353702,
            65353696, 65353697
        ];
    })().map((c) => String.fromCharCode(c ^ sens)).join('');

    return sensor;
}
```

Trois mécanismes s'enchaînent ici. Je les déroule un par un.

### La concaténation piégeuse

`[10] + [45] + [65] + [78] + [47]` -- en JavaScript, additionner des tableaux d'un seul élément ne fait pas une somme arithmétique. L'opérateur `+` convertit chaque tableau en chaîne de caractères, puis concatène. Le résultat est la **chaîne** `"1045657847"`, pas le nombre.

### Le bit-shift qui forge la clé

`sens >>= 4` convertit d'abord la chaîne en nombre (1045657847), puis décale de 4 bits vers la droite. Le résultat : **65353615**. C'est la clé XOR.

La condition `== 20` est fausse (65353615 n'est pas 20), donc le bloc `if` ne s'exécute jamais. Et même s'il s'exécutait : `sens << 4` est une **expression sans assignation**. Il manque le `=`. C'est un piège pour ceux qui lisent trop vite -- l'opération calcule une valeur et la jette immédiatement. Tout ce que nous devons décider, c'est de ne pas tomber dans ce panneau.

### Le XOR final

Chaque entier du tableau est XORé avec 65353615. Le résultat de chaque opération donne un code Unicode converti en caractère via `String.fromCharCode`. Vérification rapide :

```
65353704 ^ 65353615 = 103  → 'g'
65353663 ^ 65353615 = 48   → '0'
65353707 ^ 65353615 = 100  → 'd'
```

La chaîne complète se dessine : **`g00d_j0b_easy_deobfuscation`**.

## Quatrième bloc : le dernier piège

Le dernier noeud de l'AST n'a rien d'un noeud valide :

```json
{ "type": "jaajajajajajajajajajajajaj" }
```

Un type AST inventé. Les outils automatiques de reconversion AST vers JavaScript -- `escodegen`, `astring` et consorts -- plantent lamentablement dessus. C'est une défense anti-automatisation, un gardien de porte qui refuse le passage aux scripts trop naïfs. Vous ne passerez pas, en somme. Mais moi, je n'utilise pas ces outils : j'ai reconstruit le code à la main. Le troll ne concerne que les machines.

## La validation

Je confirme le résultat en exécutant le coeur de la logique dans une console :

```javascript
let sens = [10] + [45] + [65] + [78] + [47];
sens >>= 4;
// sens = 65353615

let flag = [
    65353704, 65353663, 65353663, 65353707, 65353680,
    65353701, 65353663, 65353709, 65353680, 65353706,
    65353710, 65353724, 65353718, 65353680, 65353707,
    65353706, 65353696, 65353709, 65353705, 65353722,
    65353724, 65353708, 65353710, 65353723, 65353702,
    65353696, 65353697
].map((c) => String.fromCharCode(c ^ sens)).join('');

console.log(flag);
// "g00d_j0b_easy_deobfuscation"
```

Le flag tombe. L'ombre est dissipée.

---

## Ce qu'on retient

| Concept | Explication |
|---|---|
| **AST (Abstract Syntax Tree)** | Représentation arborescente d'un programme. Chaque instruction devient un noeud typé (VariableDeclaration, BinaryExpression, etc.). Format standard : ESTree. |
| **Coercition de type en JS** | `[10] + [45]` donne `"1045"`, pas `55`. L'opérateur `+` sur des tableaux déclenche une conversion en chaîne puis une concaténation. |
| **Bit-shift (`>>`)** | Décalage binaire vers la droite. `x >> 4` divise par 16 (en entier). Utilisé ici pour dériver une clé XOR à partir d'une chaîne convertie en nombre. |
| **XOR pour l'obfuscation** | Opération réversible : si `a ^ b = c`, alors `c ^ b = a`. Base de nombreuses techniques d'obfuscation et de chiffrement simple. |
| **Noeud AST invalide** | Un type inventé dans l'arbre casse les outils automatiques de décompilation (escodegen, astring). Défense anti-automatisation efficace contre les scripts naïfs. |
| **Expression sans assignation** | `sens << 4` sans `sens = sens << 4` ne modifie rien. Piège courant dans le code obfusqué. |

## Outils utilisés

- **Navigateur web** -- récupération du JSON/AST
- **Lecture manuelle de l'AST** -- reconstruction du JS noeud par noeud (la méthode la plus fiable face aux pièges anti-outils)
- **Console JavaScript** -- validation du calcul XOR et du flag
- **Connaissance du standard ESTree** -- identification des types de noeuds valides vs. le troll final

---

*Ce sont les petits détails -- une coercition de type, un opérateur sans assignation -- qui font toute la différence entre une obfuscation impénétrable et un château de cartes. Il suffit de savoir où regarder.*

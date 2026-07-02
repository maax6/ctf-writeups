---
layout: default
---

# Le Site de Streaming Pirate qui Chiffrait ses Secrets en AES

**Catégorie** : Web - Client | **Difficulté** : Orange

---

Je contemple l'écran. Un site de streaming pirate s'y étale dans toute sa vulgarité mercantile : fond sombre, affiches racoleuses, et au centre, trônant comme un roi de pacotille, un film intitulé "The Ice Scream (1998)". L'indice du challenge murmure en langage codé : *"Hothagellothago ! Mothagy nothagame othagi Rijndael."* -- ce qui, une fois dépouillé de son déguisement syllabique, donne *"Hello! My name is Rijndael."* Rijndael. L'algorithme. AES. Soit. On me convie à un duel avec le chiffrement symétrique le plus déployé au monde.

Très bien. Qu'à cela ne tienne. On ne recule pas devant la forteresse ; on cherche la fissure.

## Reconnaissance du terrain

J'ouvre les DevTools et j'inventorie les forces en présence. La page charge trois scripts :

- `jquery_min.js` -- l'intendance habituelle, rien à signaler.
- `cipher.js` -- 77 Ko de Javascript obfusqué. Soixante-dix-sept mille octets de variables renommées en hexadécimal, de chaînes encodées en base64, de fonctions imbriquées comme des poupées russes. Une implémentation complète d'AES-256-CBC avec dérivation de clé au format OpenSSL, le tout noyé sous des couches de laideur syntaxique.
- Deux scripts inline, l'un exécuté au chargement, l'autre en `defer`.

L'élément `<html>` porte un attribut `data-aid="ac8b5aea3f9f"`. Je le note. On ne sait jamais ce qui servira.

## Premier script inline -- le message d'accueil

Après déobfuscation, le premier script se révèle presque touchant de simplicité : un listener sur `DOMContentLoaded` qui inscrit dans la console *"If you see this, you opened up the right tool!"* et injecte un ciphertext en base64 double-encodé dans l'élément `#boxed-ciphered-url`.

Merci pour l'encouragement. Je note le ciphertext.

## Le monolithe : cipher.js

La fonction principale se nomme `Ox5f8(ciphertext, key)`. Sous le fatras d'obfuscation, c'est du classique : AES-256-CBC, dérivation de clé via MD5 à la manière d'OpenSSL (`Salted__` + salt + password -> key + IV par hachages successifs). Rien de custom, rien de tordu dans l'algorithme lui-même. Toute la difficulté réside dans la clé.

## Deuxième script inline -- la construction de la clé

Le script `defer` attache un handler au bouton play. À l'intérieur, la clé de déchiffrement se construit pièce par pièce :

```
prefixe  = "rmovies.love"
milieu   = (622295868 + 988316868).toString()  // = "1610612736"
suffixe  = année extraite du DOM (date de sortie du film)
clé      = prefixe + milieu + suffixe
```

Pour la page principale, le film est daté de 1998. La clé est donc :

```
rmovies.love16106127361998
```

Je déchiffre avec OpenSSL :

```bash
echo "U2FsdGVkX1+..." | openssl enc -aes-256-cbc -d -a \
  -pass pass:rmovies.love16106127361998 -md md5
```

Résultat :

```
//evil-cyberlocker/player/?flag=This_is_not_the_flag
```

## La fausse piste -- ou l'art de se faire moquer

`This_is_not_the_flag`. Je relis trois fois. Je ferme les yeux. Je les rouvre. C'est toujours écrit pareil.

On m'a tendu un piège. On m'a fait dérouler 77 Ko d'obfuscation, reconstituer un algorithme de chiffrement, extraire une clé composite -- pour me cracher au visage un flag factice. Quelle bassesse. Quelle perfidie méthodique. J'enrage. Puis je respire. Puis j'enrage à nouveau, mais plus calmement. C'est là la misère du hacker : être trompé non par la machine, mais par l'esprit malin qui l'a programmée.

Il doit exister une autre page.

## La page cachée -- 1793

Je retourne sur le site. La section "You may also like" propose un second film : *"To revolt or not to revolt"*, lien vers `movies/revolution.html`. J'y vais.

L'élément `<html>` porte cette fois `data-id="flagh3re"`. "Flag here". Comme c'est délicat.

La page contient un ciphertext différent, mais aucun script de déchiffrement. Juste un message narquois : *"Decipher it by yourself now!"*. Fort bien. Je suis convié à refaire le travail moi-même, sans la becquée.

La date de sortie du film : **1793-06-24**. Mille sept cent quatre-vingt-treize. L'année où la Convention grondait, où le peuple brisait ses chaînes -- celle-là même que j'ai écrite en roman, page après page, dans la fureur et la compassion. Je ne m'attendais pas à la retrouver ici, dans un challenge Javascript, et pourtant elle m'attendait. On continue.

Même patron de clé, nouvelle année :

```
rmovies.love16106127361793
```

Je déchiffre le nouveau ciphertext :

```bash
echo "U2FsdGVkX1+[ciphertext de revolution.html]" | openssl enc -aes-256-cbc -d -a \
  -pass pass:rmovies.love16106127361793 -md md5
```

Résultat :

```
//evil-cyberlocker/player/?flag=[REDACTED]
```

La lumière. Enfin. Après l'ombre épaisse de l'obfuscation, après le leurre du faux flag, après la chasse dans les recoins du DOM -- la lumière. L'oeil de l'esprit ne peut trouver nulle part plus d'éblouissements ni plus de ténèbres que dans un fichier Javascript de 77 Ko, mais au bout du tunnel, le flag resplendit.

Il y a des jours où ce métier touche au sublime.

---

## Ce qu'on retient

| Concept | Détail |
|---|---|
| **Indice Rijndael** | Le langage codé de l'indice pointe vers AES -- toujours décoder l'indice avant de plonger |
| **AES-256-CBC OpenSSL** | Dérivation `password + salt -> MD5 -> key + IV`. Le flag `-md md5` est indispensable avec les versions récentes d'OpenSSL |
| **Clé composite** | Préfixe hardcodé + constante arithmétique + donnée extraite du DOM. Trois sources, une seule clé |
| **Red herring** | Un premier déchiffrement réussi ne signifie pas qu'on a le bon flag. Toujours vérifier |
| **data-id / data-aid** | Les attributs `data-*` inhabituels sur le HTML sont des signaux. `flagh3re` = marqueur intentionnel |
| **Pages secondaires** | Explorer toute la surface du site. Les liens "You may also like" ne sont pas décoratifs |
| **Réutilisation du pattern** | Même logique de clé, paramètre différent. Comprendre le mécanisme permet de le rejouer |

## Outils utilisés

- **DevTools** (Firefox/Chrome) -- inspection du DOM, console, lecture des scripts
- **Déobfuscateur JS** (beautifier / déobfuscateur en ligne) -- remise en forme du code
- **OpenSSL CLI** -- déchiffrement AES-256-CBC avec dérivation MD5
- **Patience** -- en quantité industrielle

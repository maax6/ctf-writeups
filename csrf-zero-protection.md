---
layout: default
---

# CSRF sans protection : le braquage silencieux

*Difficulte* : Facile | *Categorie* : Web Client

## Le plan est parfait

Je suis le Professeur. Et aujourd'hui, je ne braque pas une banque. Je braque un navigateur.

On m'a parle d'un intranet -- celui de la Fabrique Nationale de Monnaie et Timbre, comme par hasard. Un systeme interne ou les employes se connectent, consultent leurs profils, echangent des messages avec l'administrateur. Banal, en apparence. Mais j'ai appris une chose dans la vie : *ce sont les systemes les plus banals qui cachent les failles les plus belles*.

Mon objectif : acceder a la zone privee de l'intranet, reservee aux comptes valides par un administrateur. Mon compte ne l'est pas. Pas encore.

## Qu'est-ce qu'un CSRF, au juste ?

Avant de derouler le plan, une mise au point s'impose. Le CSRF -- Cross-Site Request Forgery -- c'est l'art de forcer le navigateur d'une victime a envoyer une requete authentifiee *a son insu*. Pas besoin de voler un mot de passe. Pas besoin d'intercepter quoi que ce soit. On exploite un mecanisme fondamental du web : **le navigateur envoie automatiquement les cookies avec chaque requete vers le meme domaine**.

L'administrateur est connecte a `intranet-fabrique.lab`. Son navigateur possede un cookie de session valide. Si je parviens a lui faire charger une page qui envoie une requete vers cet intranet... le navigateur ajoutera le cookie tout seul. L'admin ne verra rien, ne cliquera sur rien de suspect. Le serveur recevra une requete parfaitement authentifiee.

C'est elegant. C'est silencieux. Comme un bon plan.

## Phase 1 -- Reconnaissance

Je m'inscris sur `intranet-fabrique.lab`. Formulaire classique : nom d'utilisateur, mot de passe, confirmation. Le compte est cree, mais un message m'accueille :

> *An admin will update your status for a full access.*

Je tente d'acceder a la page privee. Reponse seche :

> *Your account has not been validated by an administrator.*

Mon `status_profile` est a `0`. Il me faut un `1`. Et seul l'admin peut le changer. Je fouille le code source de la page -- reflexe de base, toujours regarder ce que les developpeurs ont oublie de nettoyer. Dans les commentaires HTML, une pepite :

```sql
INSERT INTO membres (..., status_profile) VALUES (..., 0)
```

Le schema de la base. Le champ `status_profile`. Tout est la, ecrit noir sur blanc. Dans ce monde, tout est regi par l'equilibre : il y a ce qu'on peut voir, et ce qu'on choisit de montrer.

## Phase 2 -- Le formulaire de profil

Je navigue vers `?action=profile`. Un formulaire apparait :

```html
<form method="POST" action="?action=profile">
  <input type="text" name="username" value="sergio">
  <input type="checkbox" name="status">
  <input type="submit" value="Update">
</form>
```

Deux champs : `username` et `status`. Je coche la case, je soumets. Reponse immediate :

> *You're not an admin!*

Le serveur verifie bien que c'est un administrateur qui soumet. Mais verifie-t-il *d'ou* vient la requete ? Y a-t-il un token anti-CSRF ? Un controle du header `Origin` ? Un cookie `SameSite` ? Je regarde attentivement. Rien. Aucune protection. Zero.

Je souris. Le plan se dessine.

## Phase 3 -- La fausse piste (Plan A)

Il y a un formulaire de contact qui permet d'envoyer un message a l'administrateur. Le HTML est rendu tel quel -- pas d'echappement, pas de filtre. Mon premier reflexe, le plan A :

```html
<img src="http://intranet-fabrique.lab/?action=profile&username=sergio&status=on">
```

L'idee est simple : l'admin ouvre mon message, le navigateur charge l'image, la requete GET part avec ses cookies. Elegant, rapide. Je mets un webhook en callback pour verifier : l'admin charge bien la page. La requete part.

Mais mon profil reste inchange. `status_profile` toujours a `0`.

La raison ? Le formulaire de mise a jour exige une requete **POST**. Une balise `<img>` ne genere que du **GET**. Le serveur ignore la requete.

Un autre aurait panique. Moi, j'ajuste mes lunettes. Il y a toujours un plan B.

## Phase 4 -- L'execution (Plan B)

Si le GET ne suffit pas, il me faut un POST. Et pour generer un POST depuis le navigateur de l'admin, il me faut un formulaire. Un formulaire qui se soumet tout seul.

Le formulaire de contact rend le HTML *et* execute le JavaScript. C'est un boulevard. Je compose mon message :

```html
<form method="POST" action="http://intranet-fabrique.lab/?action=profile" id="f1">
  <input type="hidden" name="username" value="sergio">
  <input type="hidden" name="status" value="on">
</form>
<script>document.getElementById("f1").submit()</script>
```

Je l'envoie via le contact. Voila ce qui va se passer, etape par etape :

1. L'administrateur ouvre mon message
2. Le navigateur interprete le HTML : un formulaire invisible apparait
3. Le JavaScript s'execute immediatement : `submit()`
4. Le navigateur envoie une requete POST vers `?action=profile` avec les champs `username=sergio` et `status=on`
5. Le cookie de session admin est attache automatiquement a la requete
6. Le serveur recoit une requete POST authentifiee, venant de l'admin : il active mon compte

Quelques secondes passent. Je retourne sur la page privee.

Acces accorde. Le flag est la.

Comme je le dis toujours : le plan est concu pour survivre a tous les imprevus. Y compris a un formulaire qui n'accepte que du POST.

## Ce qu'on retient

| Concept | Explication |
|---|---|
| **CSRF** | Forcer le navigateur d'une victime a envoyer une requete authentifiee a son insu |
| **Cookies automatiques** | Le navigateur attache les cookies a chaque requete vers le domaine correspondant, meme si la requete est declenchee par un site tiers |
| **GET vs POST** | Une balise `<img>` ne genere que du GET. Pour un POST, il faut un formulaire (eventuellement auto-soumis via JavaScript) |
| **"0 protection"** | Pas de token anti-CSRF, pas de verification du header Origin/Referer, pas d'attribut SameSite sur le cookie |

## Comment se defendre ?

Cinq mesures, par ordre de priorite :

1. **Token anti-CSRF** : un jeton unique par session (ou par requete), inclus dans chaque formulaire et verifie cote serveur
2. **Attribut SameSite sur les cookies** : `SameSite=Strict` ou `SameSite=Lax` empeche le navigateur d'envoyer le cookie lors de requetes cross-origin
3. **Verification du header Origin/Referer** : le serveur controle que la requete provient bien de son propre domaine
4. **Methode POST obligatoire** : necessaire mais pas suffisant seul (comme on l'a vu, un formulaire auto-soumis contourne cette barriere)
5. **Echapper le HTML dans les messages** : empecher l'injection de formulaires et de scripts dans les contenus utilisateur

## Boite a outils

- **Navigateur web** avec inspecteur (DevTools) pour analyser les formulaires et les cookies
- **Webhook.site** ou **Burp Collaborator** pour verifier que l'admin charge bien le contenu
- **Burp Suite** pour intercepter et rejouer les requetes POST

---

*Dans ce monde, tout est regi par l'equilibre. Il y a les applications qui se protegent, et celles qui laissent la porte grande ouverte. Aujourd'hui, la porte etait ouverte. Et je n'ai meme pas eu besoin de forcer la serrure.*

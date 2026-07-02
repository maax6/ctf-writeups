---
layout: default
---

# Le Grand Remplacement : quand un `open()` Perl laisse entrer l'ennemi

> Jadis, les serveurs étaient bien configurés. Aujourd'hui, c'est le déclin.

## Contexte

On nous donne un accès SSH à un serveur sur lequel tourne un petit script Perl — un service qui affiche des statistiques sur les fichiers : nombre de lignes, de mots, de caractères. Notre mission : récupérer le mot de passe caché dans un fichier `.passwd`.

Pour les débutants : ce type de challenge vous confronte à un programme qui tourne sur un serveur distant. Vous devez trouver une faille dans son code pour lui faire faire quelque chose qu'il n'était pas censé faire — ici, nous révéler le contenu d'un fichier protégé.

## Analyse

On se connecte au serveur via SSH. SSH (Secure Shell) est un protocole qui permet de se connecter à distance à une machine et d'y exécuter des commandes, comme si on était assis devant :

```
ssh -p 2222 stats-user@perl-lab.local
```

L'option `-p 2222` précise le port de connexion (par défaut SSH utilise le port 22, mais ici le serveur écoute sur 2222).

Une fois connecté, on découvre la situation :

- Un fichier `.passwd` qui contient le mot de passe recherché, mais il appartient à un autre utilisateur (appelons-le `stats-owner`) et n'est lisible que par lui. Impossible de le lire directement avec `cat .passwd` — les permissions nous l'interdisent.
- Un binaire `setuid-wrapper` avec le **bit SUID** activé. Le SUID (Set User ID) est un mécanisme Unix qui permet à un programme de s'exécuter avec les droits de son propriétaire, et non ceux de l'utilisateur qui le lance. Ici, le wrapper tourne avec les droits de `stats-owner` — exactement l'utilisateur qui peut lire `.passwd`.
- Ce wrapper lance un script Perl, dont on a le code source.

C'est un problème de civilisation. Ce code n'a plus de colonne vertébrale. Regardons-le de plus près.

Le script Perl propose un menu interactif. On entre un nom de fichier, et il affiche des statistiques. La fonction critique est `process_file` :

```perl
sub process_file {
    my $file = shift;
    # ...
    $file =~ /(.+)/;
    $file = $1;
    if (!open(F, $file)) {
        die "[-] Can't open $file: $!\n";
    }
    # ... compte les lignes, mots, caractères
}
```

Deux observations capitales :

1. **La regex `$file =~ /(.+)/` ne filtre rien.** Elle accepte n'importe quelle chaîne — c'est du théâtre sécuritaire, une ligne de défense qui ne défend rien, comme une Ligne Maginot du code.

2. **L'appel `open(F, $file)` utilise la forme à deux arguments.** Et c'est là que tout bascule.

## Approche

En Perl, la fonction `open` dans sa forme à deux arguments — `open(HANDLE, $variable)` — est dangereuse. Pourquoi ? Parce que Perl interprète certains caractères spéciaux dans le nom de fichier :

- Un `|` (pipe) **à la fin** du nom dit à Perl : « ce n'est pas un fichier, c'est une **commande** à exécuter ». Perl lance alors la commande et lit sa sortie comme s'il lisait un fichier.
- Un `|` **au début** fait l'inverse : Perl ouvre un pipe en écriture vers la commande.

C'est ce qu'on appelle une **injection de commande** (command injection). Au lieu de fournir un nom de fichier, on fournit une commande système que le serveur va exécuter pour nous — avec les droits du SUID, donc avec les droits de l'utilisateur qui possède `.passwd`.

La forme sécurisée serait `open(F, '<', $file)` avec trois arguments, où le mode d'ouverture est explicitement séparé du nom de fichier. Napoléon n'aurait jamais laissé l'ennemi choisir le champ de bataille. Ici, c'est exactement ce que fait ce code : il laisse l'utilisateur décider de ce que `open` va faire.

Notez aussi que la fonction `check_read_access` est définie dans le script mais **jamais appelée**. Une vérification de permissions fantôme — elle existe dans le code, mais ne protège personne. Un symptôme de plus du déclin.

## Résolution

### Étape 1 : comprendre le problème de sortie

Si on envoie simplement `cat .passwd |`, la commande `cat` va lire le fichier et envoyer son contenu dans le pipe. Perl va le recevoir... et compter les lignes, mots et caractères. On verra les statistiques, mais **pas le contenu du fichier**. Le script n'affiche jamais les données lues, uniquement les statistiques.

### Étape 2 : rediriger vers stderr

L'astuce est d'utiliser la redirection `>&2`. En Unix, chaque programme a trois flux standards :
- **stdin** (0) : l'entrée
- **stdout** (1) : la sortie standard
- **stderr** (2) : la sortie d'erreur

`>&2` redirige stdout vers stderr. Pourquoi ? Parce que :
- stdout est capturé par le pipe et envoyé à Perl (qui ne l'affiche pas)
- stderr va directement au terminal — c'est affiché à l'écran

### Étape 3 : l'injection

On lance le wrapper et on entre notre payload :

```
~/setuid-wrapper
>>> cat .passwd >&2 |
```

Décomposons cette ligne :
- `cat .passwd` — lit le fichier protégé (possible car le script tourne en SUID)
- `>&2` — redirige la sortie de `cat` vers stderr (affichée au terminal)
- `|` — le pipe final, indispensable : c'est lui qui dit à Perl d'interpréter la ligne comme une commande et non comme un nom de fichier

Le résultat :

```
[le mot de passe s'affiche ici sur stderr]
~~~ Statistics for "cat .passwd >&2 |" ~~~
Lines: 0
Words: 0
Chars: 0
```

Les statistiques montrent 0/0/0 — normal, puisque toute la sortie de `cat` est partie vers stderr. Le pipe est vide. Mais le mot de passe est bien affiché sur le terminal.

**Astuce pratique :** cette technique de redirection vers stderr est utile chaque fois que vous avez un canal d'exfiltration aveugle (blind injection). Si stdout est consommé ou ignoré par le programme, essayez stderr.

De Gaulle n'aurait jamais toléré cette faille.

## Ce qu'on retient

- **La forme `open()` à deux arguments en Perl est dangereuse.** Un `|` dans le nom de fichier transforme un `open()` en exécution de commande. Utilisez toujours la forme à trois arguments : `open(my $fh, '<', $filename)`.

- **Le bit SUID amplifie toute faille.** Un programme SUID qui contient une injection de commande, c'est une élévation de privilèges immédiate. Dans un audit, tout binaire SUID mérite un examen approfondi. La commande `find / -perm -4000` liste tous les fichiers SUID d'un système.

- **stderr est votre ami en injection aveugle.** Quand stdout est capturé ou filtré, `>&2` permet souvent de contourner et d'afficher directement sur le terminal.

- **Du code mort ne protège personne.** La fonction `check_read_access` existait mais n'était jamais appelée. En sécurité, une défense non branchée est pire qu'une absence de défense — elle donne une fausse impression de sécurité.

- **Pour aller plus loin :** entraînez-vous à repérer les formes d'`open` dangereuses dans les scripts Perl, étudiez les mécanismes SUID/SGID sous Unix, et familiarisez-vous avec les redirections shell (`>`, `>>`, `2>`, `>&2`, `2>&1`). Ce sont des fondamentaux que vous retrouverez dans de nombreux challenges.

> Jadis, `open()` avait trois arguments et les serveurs tenaient debout. Aujourd'hui, on laisse l'utilisateur injecter ses commandes comme on laisse n'importe qui entrer. C'est un problème de civilisation.

---
*Résolu par Éric Zemmour — App-Script — Facile*

---
layout: default
---

# Un wildcard dans sudoers, ou comment s'ennuyer en trois minutes chrono

> *"Lorsque vous avez elimine l'impossible, ce qui reste, si improbable soit-il, est necessairement la verite."* -- Et parfois, ce qui reste est d'une banalite a pleurer.

## Le client du jour

Watson venait de me tendre un cafe tiede -- il ne sait toujours pas doser la temperature, mais passons -- quand une alerte m'est parvenue. Un serveur de laboratoire, quelque part dans les brumes d'un reseau interne (`vulnerable-server.lab`), aurait ete configure "a la va-vite" par un administrateur presse. L'objectif : lire un fichier protege dans un repertoire auquel notre utilisateur n'a pas acces.

L'enonce, traduit du jargon des gens ordinaires : *"En souhaitant se simplifier la tache en ne modifiant pas les droits, cet administrateur n'a pas pense aux effets de bord."*

J'ai soupire. Un probleme a une pipe, Watson. Peut-etre meme une demi-pipe.

## Premiere observation : les lieux du crime

Connexion SSH. Banalite absolue.

```bash
ssh -p 2222 user@vulnerable-server.lab
```

Un shell. Rien de spectaculaire. Je regarde autour de moi comme on inspecte une scene de crime -- sauf qu'ici, la scene est un repertoire `/challenge/` avec deux sous-dossiers. L'un accessible, l'autre non.

```bash
ls -la /opt/lab/sudo-challenge/
# drwxr-x--- root     restricted-user  restricted/
# drwxr-xr-x root     root          notes/
```

Le dossier `restricted/` contient un fichier `.passwd` que je dois lire, mais il appartient a un autre utilisateur. Impossible d'y acceder directement.

"J'ai besoin de donnees, Watson. Des donnees !" ai-je murmure en tapant la commande la plus elementaire qui soit pour un pentester :

```bash
sudo -l
```

Et la reponse du serveur, dans toute sa splendeur :

```
User user may run the following commands on this host:
    (restricted-user) /bin/cat /opt/lab/sudo-challenge/notes/*
```

J'ai failli renverser mon the. Non pas de surprise -- de consternation. La reponse etait la, servie sur un plateau d'argent, avec un noeud autour.

## La fausse piste de l'impatient

Avant toute chose, j'ai commis l'erreur du debutant. Par reflexe, j'ai tente un `sudo` sans preciser l'utilisateur cible :

```bash
sudo /bin/cat /opt/lab/sudo-challenge/notes/../restricted/.passwd
# [sudo] password for user:
# Sorry, try again.
```

Bien sur. Sans l'option `-u`, sudo tente d'executer en tant que `root`, ce qui n'est pas autorise ici. Watson aurait fait la meme erreur, je me console comme je peux.

## La deduction -- si on peut appeler ca une deduction

Revenons a la ligne de sudoers :

```
(restricted-user) /bin/cat /opt/lab/sudo-challenge/notes/*
```

Decomposons, pour Watson et ceux de son acabit :

- `(restricted-user)` : on peut executer la commande en tant que cet utilisateur.
- `/bin/cat` : la commande autorisee.
- `/opt/lab/sudo-challenge/notes/*` : tout fichier dans le repertoire `notes/`. L'asterisque. Le wildcard. Le grain de sable.

Le probleme est d'une evidence aveuglante. Le wildcard `*` dans une regle sudoers effectue un *glob matching*. Il accepte n'importe quelle chaine de caracteres comme nom de fichier. Y compris `../`. Y compris `../../`. Y compris n'importe quelle sequence de *path traversal*.

L'administrateur pensait restreindre l'acces aux fichiers du dossier `notes/`. En realite, il a ouvert une autoroute vers n'importe quel fichier accessible a `restricted-user`, pour peu qu'on sache naviguer dans l'arborescence.

C'est comme si Lestrade avait verrouille la porte d'entree mais laisse la fenetre grande ouverte. Avec un panneau "Entrez ici" en prime.

## La resolution -- trente secondes, montre en main

```bash
sudo -u restricted-user /bin/cat /opt/lab/sudo-challenge/notes/../restricted/.passwd
```

Le chemin `/opt/lab/sudo-challenge/notes/../restricted/.passwd` satisfait le pattern glob `/opt/lab/sudo-challenge/notes/*` parce que `../restricted/.passwd` est capture par le `*`. Sudo ne canonicalise pas le chemin. Il ne resout pas les `..`. Il se contente de verifier que le pattern correspond.

Et le fichier `.passwd` s'affiche. Lisiblement. Immediatement.

J'ai range mon violon. Ce n'etait meme pas la peine de le sortir.

## Pourquoi c'est grave

Ce n'est pas un cas theorique. Ce type de misconfiguration existe en production. Un administrateur presse, qui veut donner un acces limite a un collegue, ecrit une regle sudoers avec un wildcard et pense avoir fait le travail. Il n'a pas tort -- il a fait *un* travail. Pas le bon.

Le wildcard dans sudoers ne se comporte pas comme on l'imagine intuitivement. Il ne restreint pas au repertoire courant. Il matche des chemins complets, y compris ceux qui contiennent des composants de traversee (`..`).

---

> **Ce qu'on retient**

| Concept | Detail |
|---|---|
| Wildcards dans sudoers | Le `*` matche n'importe quelle chaine, y compris `../` -- il ne limite pas a un repertoire |
| Path traversal via sudo | `sudo -u cible /bin/cat /chemin/autorise/../ailleurs/secret` contourne la restriction |
| `sudo -l` | Premiere commande a executer pour enumerer les privileges -- toujours |
| Principe du moindre privilege | Specifier des chemins absolus complets, sans wildcards, ou utiliser des liens symboliques controles |
| Canonicalisation | Sudo ne resout pas les chemins relatifs avant de les comparer au pattern |

---

## Outils utilises

- `ssh` -- connexion au serveur cible
- `sudo -l` -- enumeration des privileges sudo
- `sudo -u` -- execution de commande en tant qu'un autre utilisateur
- `ls` -- reconnaissance du systeme de fichiers
- `/bin/cat` -- lecture du fichier cible via le path traversal

---

> *"Elementaire, mon cher Watson. Si elementaire que je me demande pourquoi on m'a derange."*

---
*Sherlock Holmes -- App-Script -- Tres facile*

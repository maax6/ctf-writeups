---
layout: default
---

# Le Wildcard dans le Sudoers, ou Comment un Asterisque Trahit son Maitre

> *"Lorsque vous avez elimine l'impossible, ce qui reste, si improbable soit-il, est necessairement la verite."* Et parfois, la verite est d'une banalite affligeante.

## L'affaire du serveur mal configure

Je me presente. Sherlock Holmes, consultant en securite offensive. Le seul au monde, d'apres mes estimations -- et mes estimations sont rarement erronees.

Ce matin, Mrs. Hudson m'a transmis un ticket d'intervention. Un serveur Linux, un acces SSH restreint, et quelque part dans les entrailles de la machine, un fichier `.passwd` qu'on me demande de lire sans en avoir les droits. L'administrateur systeme, confiant dans sa configuration, a cru pouvoir simplifier la gestion des permissions en s'appuyant sur `sudo`. Il n'a pas pense aux effets de bord.

Un probleme a une pipe. Peut-etre meme pas.

## Prise de contact avec la scene de crime

Je me connecte au serveur via SSH. Les identifiants sont connus, rien de bien sorcier ici -- ce n'est pas la porte qui m'interesse, c'est ce qu'il y a derriere.

```bash
ssh -p 2222 app-script-ch1@10.10.42.42
```

Le shell s'ouvre. Un prompt minimaliste. Aucune banniere personnalisee, aucun message d'accueil -- cet administrateur n'est pas du genre bavard. Tant mieux, les bavards m'ennuient.

Premiere chose que fait tout detective digne de ce nom en arrivant sur une scene : observer. Je ne touche a rien, je regarde tout.

```bash
ls -la
```

Un repertoire personnel classique. Quelques fichiers, des permissions standard. Je jette un oeil a l'arborescence du challenge pour comprendre la structure :

```bash
ls -la /challenge/app-script/ch1/
```

Un repertoire `notes/`, un repertoire ou un fichier `.passwd` quelque part a proximite. La topographie des lieux commence a se dessiner dans mon esprit. Mais il me manque l'essentiel. J'ai besoin de donnees. Les donnees ! LES DONNEES !

## La deduction

La commande `sudo -l` est l'equivalent numerique de ma loupe. Elle me revele ce que le systeme autorise mon utilisateur a faire avec des privileges empruntes :

```bash
sudo -l
```

Le resultat est limpide :

```
(app-script-ch1-cracked) /bin/cat /challenge/app-script/ch1/notes/*
```

Je peux executer `/bin/cat` en tant que l'utilisateur `app-script-ch1-cracked`, mais uniquement sur les fichiers correspondant au motif `/challenge/app-script/ch1/notes/*`.

Je m'arrete. Je pose mon violon imaginaire. Je reflechis.

Un wildcard. Un asterisque a la fin d'un chemin dans une regle sudoers. L'administrateur a voulu dire : "tu peux lire n'importe quel fichier dans le repertoire `notes/`". Ce qu'il a reellement dit au systeme, c'est : "tu peux lire n'importe quoi qui commence par ce chemin, y compris si le glob traverse des repertoires parents."

Car voyez-vous, l'asterisque dans sudoers, interprete par `glob(3)`, ne canonicalise pas les chemins. Il ne resout pas les `..` avant de verifier la correspondance. Il se contente de valider que la chaine fournie correspond au motif. Et `../` est un composant de chemin parfaitement valide.

## L'exploit

Le fichier `.passwd` se trouve vraisemblablement dans un repertoire voisin de `notes/`, probablement quelque chose comme `ch1/.passwd` ou un sous-repertoire `secret/`. La technique est classique : path traversal via `../`.

```bash
sudo -u app-script-ch1-cracked /bin/cat /challenge/app-script/ch1/notes/../.passwd
```

Et voila. Le contenu du fichier `.passwd` s'affiche. Le flag apparait dans mon terminal comme une evidence.

Je n'ai meme pas eu le temps de remplir ma pipe.

## L'absence de fausses pistes (et pourquoi ca m'agace)

Je dois l'admettre, et cela m'ennuie profondement : il n'y a pas eu de fausse piste. Pas d'impasse. Pas de moment ou j'ai du remettre en question mes hypotheses. La vulnerabilite etait visible des la sortie de `sudo -l`, et l'exploitation a fonctionne du premier coup.

Watson m'aurait regarde avec admiration. Mais entre nous, n'importe quel inspecteur de Scotland Yard aurait pu resoudre celle-ci. Presque.

J'aurais pu tenter d'autres approches si la premiere avait echoue. Par exemple, verifier si des liens symboliques etaient possibles dans le repertoire `notes/`, ou tester si le wildcard acceptait des arguments multiples pour injecter d'autres chemins. Mais non. Le `../` a suffi. C'est presque vexant.

Le vrai merite de ce genre d'exercice, c'est qu'il illustre a quel point une erreur de configuration apparemment anodine -- un petit `*` en fin de chemin -- peut compromettre tout un systeme de permissions. L'administrateur voulait eviter de modifier les droits fichier par fichier. Il a pris un raccourci. Les raccourcis, en securite, menent toujours quelque part d'inattendu.

Et c'est la que reside le danger. Sur un vrai serveur de production, cette meme regle sudoers pourrait permettre a un utilisateur restreint de lire `/etc/shadow`, des cles SSH privees, ou n'importe quel fichier sensible du systeme. Le wildcard ne fait pas de distinction morale entre un fichier de notes et un fichier de mots de passe. Il se contente de matcher.

## Ce qu'on retient

> La partie technique, serieusement cette fois.

| Concept | Detail |
|---|---|
| **Vulnerabilite** | Wildcard (`*`) dans une regle sudoers permettant le path traversal |
| **Mecanisme** | `glob(3)` ne canonicalise pas les chemins ; `../` est accepte dans le motif |
| **Impact** | Lecture arbitraire de fichiers en tant qu'un autre utilisateur |
| **Remediation** | Utiliser des chemins absolus sans wildcard, ou configurer `sudoers` avec des regles plus restrictives. Privilegier `sudoedit` pour l'edition de fichiers. |
| **Regle d'or** | Ne jamais utiliser de wildcard dans les chemins sudoers sans comprendre exactement ce que `glob` va accepter |

### Bonnes pratiques sudoers

- **Eviter les wildcards dans les chemins** : un `*` en fin de commande dans sudoers est presque toujours une mauvaise idee.
- **Canonicaliser les chemins** : si un wildcard est necessaire, s'assurer que le binaire appele resout les liens symboliques et les `..` avant d'operer.
- **Principe du moindre privilege** : donner acces uniquement aux fichiers specifiques necessaires, pas a un repertoire entier via glob.
- **Auditer regulierement** : un `sudo -l` sur chaque compte utilisateur devrait faire partie de tout audit de securite.

## Outils utilises

| Outil | Usage |
|---|---|
| `ssh` | Connexion au serveur cible |
| `sudo -l` | Enumeration des privileges sudo de l'utilisateur courant |
| `sudo -u` | Execution de commande en tant qu'un autre utilisateur |
| `/bin/cat` | Lecture du fichier protege |
| `../` | Traversee de repertoire pour echapper au perimetre autorise |

## References

- [Dangerous Sudoers Entries -- Part 4: Wildcards (Compass Security)](https://blog.compass-security.com/2012/10/dangerous-sudoers-entries-part-4-wildcards/)
- [Wildcards Spare Tricks (HackTricks)](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/wildcards-spare-tricks)
- [Beware of Wildcard Paths in Sudo Commands (David Hamann)](https://davidhamann.de/2023/02/24/beware-of-wildcard-paths-sudo/)
- [Sudo Wildcard Issue (sudo-project GitHub)](https://github.com/sudo-project/sudo/issues/15)
- [Sudo Privilege Escalation (Exploit Notes)](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/sudo/)

---

*Resolu par Sherlock Holmes -- App-Script -- Tres facile. Elementaire, mon cher Watson.*

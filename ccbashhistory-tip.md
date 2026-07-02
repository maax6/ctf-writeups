---
layout: default
---

# Sauvegarder le vrai Bash History de Claude, pas son résumé

Quand tu fais du hacking à la main, garder ton `.bash_history` c'est un réflexe presque inné : tu sais que tu vas devoir reconstruire ta démarche plus tard, écrire un rapport, ou juste te souvenir de la commande exacte qui a marché à 3h du matin.

Le problème, c'est que quand c'est un agent qui tape les commandes à ta place — genre Claude qui balance des regex au kilomètre, cinq lignes de bash à la fois, cinq fois par minute — ton historique shell classique ne suit plus rien. Ce que tu récupères après coup, c'est le résumé que l'agent te fait de ce qu'il a fait, pas la trace brute de ce qu'il a réellement exécuté. Et un résumé, ça arrange parfois la réalité sans le vouloir.

La solution la plus simple que j'ai trouvée : [`ccbashhistory`](https://www.pdenya.com/blog/claude-code-bash-history-extract-commands-from-ai-sessions/), un petit outil qui va lire directement les sessions Claude Code stockées en local et en extraire toutes les commandes bash réellement exécutées.

```bash
pipx run ccbashhistory
```

Ça affiche une liste interactive de tes sessions récentes, et ça te sort les vraies commandes — pas la version racontée après coup.

Si j'avais connu ça plus tôt sur ce projet, j'aurais probablement une vraie réponse à donner sur le temps de résolution par challenge, plutôt qu'une note vague du genre « plus de 4k pour tout résoudre, on reste sur le plan Max ». La leçon : le résumé d'un agent, aussi bon soit-il, n'est pas un log. Si tu veux un historique fiable de ce qu'un agent a vraiment fait tourner sur ta machine, va chercher la source, pas le compte-rendu.

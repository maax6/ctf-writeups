---
title: Writeups
---

# Quelques writeups

Une petite sélection de writeups de challenges de sécurité offensive, rédigés pendant un projet d'automatisation par agents IA. Techniques, sans prétention, et sans jamais révéler le flag final — l'idée est de montrer la démarche, pas de spoiler l'exercice.

- [Juste une dernière `@import`](css-exfiltration.html) — voler un token CSRF caractère par caractère via des sélecteurs CSS, sans une ligne de JavaScript.
- [Google, complice malgré lui](csp-bypass-jsonp.html) — contourner une Content-Security-Policy en détournant un vieil endpoint JSONP de confiance.
- [Sudo Wildcard Misconfiguration](sudo-wildcard-misconfiguration.html) — une étoile mal placée dans un fichier sudoers, et c'est toute une traversée de chemin qui s'ouvre.
- [Un Magicien ne se laisse jamais obfusquer](ast-deobfuscation.html) — désobfusquer du JavaScript en manipulant directement son arbre syntaxique plutôt qu'en le lisant à l'œil nu.
- [Un `<table>` de 1998 contre une CSP moderne](csp-dangling-markup-2.html) — exploiter un CSP mal configuré via du markup HTML volontairement laissé pendant.
- [Un chiffrement JS craqué sans ouvrir une popup](js-obfuscation-scope-chain.html) — remonter une chaîne de portées JavaScript pour désobfusquer un script piégeux.
- [Exfiltrer avec une balise des années 90](csp-bypass-dangling-markup.html) — une autre variante du bypass CSP par markup pendant.
- [Le lien qui te cogne en retour](xss-volatile.html) — une injection XSS qui ne laisse presque aucune trace.
- [Je suis celui qui forge la requête](csrf-token-bypass.html) — voler un jeton CSRF cense être infaillible en l'enchaînant avec un XSS stocké.
- [Le formulaire qui ne posait jamais de questions](csrf-zero-protection.html) — quand la protection CSRF n'existe tout simplement pas.
- [Le cookie que personne ne surveillait](xss-stockee-2.html) — une injection XSS stockée, variante avancée.
- [Tout le monde ment, surtout les headers HTTP](csp-nonce-bypass-2.html) — contourner un nonce CSP censé bloquer l'injection de scripts.
- [Le site de streaming pirate et son chiffrement AES](streaming-site-aes-decryption.html) — désobfusquer 77 Ko de JavaScript pour reconstruire une clé de déchiffrement AES composite.
- [Sudo Wildcard Misconfiguration (2)](sudo-wildcard-misconfiguration-2.html) — une autre variante de la faille wildcard dans sudoers.
- [Le Grand Remplacement](perl-open-command-injection.html) — une injection de commande via la forme à deux arguments d'`open()` en Perl, racontée par un polémiste convaincu que le code aussi est en déclin.

## Outillage

- [Sauvegarder le vrai Bash History de Claude, pas son résumé](ccbashhistory-tip.html) — un outil pour extraire les commandes réellement exécutées par un agent pendant une session, plutôt que son résumé.

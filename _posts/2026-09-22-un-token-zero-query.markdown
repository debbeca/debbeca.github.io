---
layout: post
title:  "Un Token, Zero Query"
date:   2026-09-22 16:47:20 +0200
categories: optimisation architecture OAuth2.0
---

Quand j’ai commencé ce projet, je ne me suis jamais imaginé qu’un simple token oAuth2 puisse nous économiser des dizaines de requêtes. L’objectif initial était d’avancer vite. On avait besoin d’une authentification qui fonctionne sans trop se compliquer la vie, alors on s’est contentés d’une oAuth 2 anémique. On vérifiait dans le directory que l’utilisateur existait bien, on regardait s’il avait les droits de lecture ou d’écriture sur l’outil, et on passait à autre chose. C’était suffisant pour démarrer. Sauf que très vite, on s’est rendu compte que c’était un gouffre de performance.

Dès que l’utilisateur commençait à naviguer, la réalité du métier s’imposait. Il ne s’agissait plus seulement de savoir s’il pouvait ouvrir l’outil. Il fallait décider, dossier client après dossier client, s’il avait le droit de voir certaines données sensibles selon son rôle dans la boîte, et surtout s’il respectait les contraintes de juridiction géographique. La souveraineté numérique de certains pays interdit purement et simplement d’afficher des informations en dehors de leurs frontières, quel que soit le rôle de la personne connectée depuis l’extérieur. Pour répondre à ces questions, on faisait des allers-retours constants entre le directory et notre table utilisateurs, dans laquelle on avait commencé à stocker des rôles et des droits spécifiques à l’application. Ce jonglage se répétait à chaque ouverture de dossier. C’était trop lourd, trop fastidieux, et surtout trop coûteux en performances.

Voici à quoi ressemblait le parcours avant la migration :

```
[Connexion]
     │
     ▼
┌─────────────────────┐
│ Directory (AD/LDAP)│ ← « L’utilisateur existe-t-il ? »
│ + droits basiques │ ← lecture / écriture sur l’outil
└─────────────────────┘
     │
     ▼
[Utilisateur authentifié – token quasi vide]
     │
     ▼
À chaque ouverture de dossier client :
     │
     ├─► Interrogation Directory
     ├─► Interrogation table User (rôles applicatifs)
     ├─► Calcul des droits métier + juridiction géographique
     │ (souveraineté numérique : certains pays interdisent
     │ l’affichage de données hors frontière, quel que soit le rôle)
     └─► Affichage (ou non) des données sensibles
```

On a donc décidé de reconsidérer entièrement la façon dont on gérait la juridiction. Au lieu de s’appuyer uniquement sur le directory de la boîte, on a pris en compte aussi notre table utilisateurs enrichie des rôles et des droits propres à l’application. À la connexion, on calcule désormais l’intégralité des droits de l’utilisateur : le droit de consulter telle liste de pays, le droit d’accéder à tel type de données sensibles, le droit d’écrire sur telle catégorie d’informations, et bien sûr les règles de souveraineté numérique. Tout est calculé une seule fois, puis injecté dans les claims du token. L’utilisateur trimballe ensuite ces informations partout dans l’application. Plus aucun recalcul n’est nécessaire après l’authentification.

Le parcours est devenu celui-ci :

```
[Connexion]
     │
     ▼
┌──────────────────────────────────────────────┐
│ Directory + Table User enrichie │
│ (rôles + droits applicatifs) │
│ │
│ Calcul UNIQUE de TOUS les droits : │
│ • Liste de pays consultables │
│ • Types de données sensibles autorisés │
│ • Droits d’écriture par type de donnée │
│ • Règles de souveraineté numérique │
└──────────────────────────────────────────────┘
     │
     ▼
[Token enrichi – Claims complets]
     │
     │ (les claims voyagent avec l’utilisateur)
     ▼
Toute navigation dans l’application
→ Lecture directe des claims
→ Aucun recalcul
→ Aucun aller-retour Directory / Base
```

Le gain de performance s’est révélé très intéressant. En éliminant les allers-retours et les calculs répétitifs, on a allégé drastiquement la navigation de l’utilisateur. Ce qui avait commencé comme une solution rapide pour avancer s’est transformé, une fois le constat d’abus fait, en une architecture plus saine, plus claire et nettement plus efficace.

## Conclusion
Ce qu’il faut retenir, c’est qu’il faut éviter les tokens OAuth2 anémiques qui ne reflètent pas les vrais droits de vos utilisateurs. À l’image d’un domaine anémique dans une architecture hexagonale, c’est un anti-pattern. Et si, pendant une même session utilisateur, vous vous retrouvez à consulter deux fois la même donnée pour recalculer des droits, il y a certainement des questions à se poser.
Enfin, percevoir le direcotory comme la seule source de vérité pour les droits applicatifs est une erreur. Il faut voir le couple directory + table utilisateurs enrichie comme la source de vérité pour les droits applicatifs. C’est ce qui permet d’éviter les allers-retours incessants et de garantir une expérience utilisateur fluide et performante.

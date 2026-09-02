---
layout: post
title:  "Dans la peau d'un DBA"
date:   2025-12-29 16:47:20 +0200
categories: optimisation database Oracle
---


J'ai récemment été confronté à un problème de performance assez coriace qui m'a donné du fil à retordre.
Il s'agit toujours des plaisirs de la période de Noël, mais cette fois-ci, ce n'était pas un problème de cadeau manquant ou de sapin mal décoré. Non, c'était un problème de performance dans notre base de données Oracle pendant une période de forte charge.
Accrochez-vous, car nous allons plonger au cœur du problème !

## Les symptômes : ralentissements et shutdown
Tout a commencé par des ralentissements inexpliqués de nos applications. 
Les temps de réponse étaient en hausse, et nos utilisateurs commençaient à s'impatienter.
Ensuite, le shutdown total.
Le DBA a dû redémarrer la base de données pour tenter de résoudre le problème, mais cela n'a fait que retarder l'inévitable.
Ensuite, il s'est contenté de rafraichir le cache et la Shared Pool, pensant que cela résoudrait le problème. Malheureusement, cela n'a fait que masquer temporairement les symptômes.


J'ai été convié à une réunion d'urgence pour discuter de la situation, et c'est là que j'ai commencé à creuser.
J'ai tout de suite demandé à voir les statistiques de la base de données, les rapports d'exécution des requêtes, et les logs pour comprendre ce qui se passait.
En fait, J'ai fait une analogie avec un problème rencontré sur l'augmentation du querycache plan  d'hibernate que j'ai évoqué dans [un ancien post](https://debbeca.github.io/optimisation/java/springboot/hibernate/2021/06/07/hibernate-pompier-pyromane.html). Je pensais que j'ai été confronté au même problème, mais au niveau de la base de données et pas au niveau de l'ORM.
En regardant les requêtes, aucune variation de nombre de paramétres n'est possible. Une seule version de la requête est générée. Donc, pas de problème de ce côté-là.
Les Rapport AWR (Automatic Workload Repository) ont révélé une augmentation significative de la mémoire utilisée lors de l'exécution de la requête problématique. 

## La cause profonde : une armée de "child cursors"
 
Après quelques investigations, nous avons identifié un coupable potentiel.
Une contention de cursor mutex X dans notre base de données Oracle.

Pour ceux qui ne sont pas familiers, cursor: mutex X est un verrou exclusif utilisé par Oracle pour protéger les structures internes des curseurs partagés. En gros, cela signifie que plusieurs sessions essayaient d'accéder ou de modifier le même curseur en même temps, ce qui entraînait des blocages.


En creusant davantage, nous avons découvert que le problème était dû à une génération excessive de "child cursors". Oracle crée ces "child cursors" pour gérer différentes versions d'une même requête SQL dans le cache partagé (Shared Pool). Cela se produit lorsque des variations dans les paramètres ou les contextes d'exécution empêchent la réutilisation d'un curseur existant.

Dans notre cas, une requête particulière générait plus de 2000 child cursors après chaque purge de la Shared Pool, ce qui était clairement anormal.
<img style="display: flex; justify-content: center;" width="80%" src="/assets/img/1788364119112.png">{:style="display:block; margin-left:auto; margin-right:auto"}

## Le coupable : BIND_MISMATCH

Après une analyse approfondie, nous avons identifié la source du problème : BIND_MISMATCH. Cela signifie que les variables de liaison (bind variables) utilisées dans la requête n'étaient pas compatibles entre elles ou avec les valeurs passées.

Par exemple, nous avions des différences dans les types de données (VARCHAR2 vs NUMBER) et des valeurs NULL dans les variables de liaison, ce qui modifiait le plan d'exécution et forçait Oracle à créer un nouveau child cursor à chaque exécution.

L'impact de cette situation était considérable. La requête problématique consommait à elle seule 3.6 Go de mémoire. Chaque child cursor grignotait de la mémoire dans la Shared Pool, ce qui entraînait une saturation de cette dernière.

La création fréquente de nouveaux child cursors provoquait des verrous exclusifs (Mutex) pour synchroniser l'accès à ces curseurs. La fréquence élevée d'exécution de la requête amplifiait le problème, ralentissant considérablement les performances globales.

## La solution : des quicks wins pour restaurer les performances

Pour résoudre ce problème, nous avons mis en œuvre plusieurs solutions :

-Uniformisation des Bind Variables : Nous avons vérifié que les types de données et les tailles des variables de liaison étaient cohérents, et nous avons évité les valeurs NULL autant que possible.

-Forcer le partage des curseurs : Nous avons utilisé le paramètre Oracle CURSOR_SHARING=FORCE pour encourager la réutilisation des curseurs.

-Optimisation de la requête : Nous avons réduit la complexité de la requête pour minimiser les variations dans les plans d'exécution.

## Conclusion

Cette expérience m'a rappelé l'importance d'une gestion rigoureuse des bind variables et d'une configuration appropriée de la base de données. En fin de compte, en uniformisant les bind variables, en forçant le partage des curseurs et en optimisant la requête, nous avons réussi à résoudre le problème de contention de curseur et à restaurer des performances optimales.

J'espère que ce post vous sera utile si vous rencontrez des problèmes similaires.


## References

[1]: <https://www.nazmulhuda.info/troubleshooting-mutex-xs-due-to-high-version-count> "Troubleshooting mutex X/S Due to High Version Count"
[2]: <https://www.bobbydurrettdba.com/2013/05/29/yet-another-bind-variable-type-mismatch-yabvtm/> "Yet Another Bind Variable Type Mismatch"

[[1]] "Troubleshooting mutex X/S Due to High Version Count"

[[2]] "Yet Another Bind Variable Type Mismatch"





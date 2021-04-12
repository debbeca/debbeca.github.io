---
layout: post
title:  "Hibernate Pompier Pyromane ?"
date:   2021-04-07 16:47:20 +0200
categories: optimisation java springboot hibernate
---

Après deux années de developpement, nous étions amener à optimiser une application SpringBoot / Hibernate après avoir observé plusieurs problème de performance; des memory leak, des timeout et des lenteurs. Malgré l'urgence de la situation, l'optimisation de cette application n’était pas un sprint, mais un marathon de longue haleine. Nous allons étaler les différentes phases de cette opération, les états d'ame et les approches suivies pour résoudre les problèmes rencontrés.

## Phase I : Panic Room

En fait, quand on vous annonce les premiers crash de l’application en prod, on essaie souvent de prédire la source des problèmes. On saute sur les painkillers. Les quicks wins. Sauf que ça ne dure que pour un instant. Augmenter le timeout de son apache ne fait que noyer la douleur en une overdose d'antalgique. Le problème refera surface au bout de quelques semaines. Une analyse en profondeur s’impose. Pour ce faire, il faut être bien outillé.

## Phase II : On soulève le capot

Collecter des informations - Analyser - enhance - repeat
Vous avez besoin de surveiller plusieurs aspects.
D’abord le comportement de votre garbage collector vous dit long sur votre application. Un dump de la heap memory quand votre application crache est un incontournable. Vous pouvez le faire comme suit
- ICI chercher comment faire un heap dump

Ensuite vous aurez besoin de collecter des statistiques sur Hibernate et la base de données. Les requêtes les plus exécutées, les requêtes qui mettent le plus de temps à être exécuté.
Pour ce faire vous pouvez soit utiliser des sondes de type “Dynatrace” Ou bien vous essayer d’activer les statistiques Hibernate comme suit :

- ICI statistique hibernate activation and example

PS : Il n’est pas recommandé de le laisser la collecte des statistiques hibernate activée en prod car ça risque de ralentir votre application d’avantage

>Pas d’économies de bouts de chandelles, attaquer de front le plus gros. 

Alléger la mémoire des objets qui prennent le plus d’espace. 
Optimiser les requêtes les plus longues. 
Diminuer les requêtes les plus exécutées.




## Phase III : Implémentations des solutions

1. Mettre le ```show_sql``` à true dans la phase de dev pour voir les queries générées par hibernate
2. Utiliser ```@QuickPerf``` sur les tests d’intégration pour détecter les N+1 au plus vite. Ça nécessite un test d'intégration sur la base de données
3. Indexer les colonnes et attributs utilisés dans vos recherches. Si vous avez un ```findByX```, assurez vous d’avoir, dans la mesure du possible, un index sur le X 
4. Vérifier convenablement vos equals et hashcode. Si vous utilisez lombok, faites attention à ne pas inclure dans relations @OneToMany dans l'implémentation par défaut car vous allez réveiller la bête à chaque fois votre entité est manipulée dans une collection
5. Utiliser le ```padded``` pour le ```batch_fetch_style``` avec un size( 8 ou 16) pour que vos N+1 queries soient transformées en un batch de quelques requêtes avec une IN Clause.
6. Utiliser la navigation ```@EntityGraph``` pour forcer la jointure avec les entités attachées et éviter les N+1. Surtout si vous manipulez un agrégat.
7. Surveiller votre ```QueryCachePlan```, hibernate cache les requêtes exécutées dans la base de données, si vous avez des requêtes trop sollicitées avec un ```Where X IN``` clauses, vous risquez d’avoir autant de requêtes que de cardinalités possibles dans la in clause. Ça risque d’aller super loin et pour optimiser ceci, peut être penser à réécrire la requête avec une jointure et une subquery.
8. Opter pour le padding des IN clauses. En effet, si vous n'arrivez pas à réécrire vos requêtes, vous pouvez utiliser le padding qui permet de répéter les valeurs passées en paramètre dans la requête afin de garder la même structure et ne pas dupliquer la rêquete autant de fois qu'on a de nombre de paramètres.

    - exemple ICI

9. La performance d’une application commence à la conception de son interface, si vous ne pensez pas la cardinalité et la volumétrie de vos données, vous risquez d’être surpris plus tard quand la page met des plombs pour se charger. Pour la simple et unique raison qu’on n’a pas prévu une pagination, ou un load more.


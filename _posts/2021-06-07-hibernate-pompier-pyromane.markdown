---
layout: post
title:  "Jouer au pompier pyromane avec Hibernate et la performance"
date:   2021-06-07 16:47:20 +0200
categories: optimisation java springboot hibernate
---

Après deux années de developpement, nous étions amener à optimiser une application SpringBoot / Hibernate après avoir observé plusieurs problèmes de performance; des memory leak, des timeout et des lenteurs. Malgré l'urgence de la situation, l'optimisation de cette application n’était pas un sprint, mais un marathon de longue haleine. Nous allons étaler les différentes phases de cette opération, les états d'âme et les approches suivies pour résoudre les problèmes rencontrés.

## Phase I : Panic Room

En fait, quand on vous annonce les premiers crash de l’application en prod, on essaie souvent de prédire la source des problèmes. On saute sur les painkillers. Les quick wins. Sauf que ça ne dure que pour un instant. Augmenter le timeout de son apache ne fait que noyer la douleur en une overdose d'antalgique. Le problème refera surface au bout de quelques semaines. Une analyse en profondeur s’impose. Pour ce faire, il faut être bien outillé.

## Phase II : On soulève le capot

Afin de mieux optimiser il faudra collecter des informations - analyser - implémenter des solutions - répéter. C'est un travail itératif de longue haleine. Car à chaque fois vous finissez de résoudre un problème, d'autres apparaissent.
Vous avez besoin de surveiller plusieurs aspects.
D’abord, le comportement de votre garbage collector vous en dit long sur votre application. Un dump de la heap memory quand votre application crache est incontournable. Vous pouvez le faire comme suit:
```bash
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=${DOMAIN_HOME}/logs/mps"
```

Ensuite, vous aurez besoin de collecter des statistiques sur Hibernate et la base de données. Les requêtes les plus exécutées, les requêtes qui mettent le plus de temps à être exécutées.
Pour ce faire, vous pouvez soit utiliser des sondes de type “Dynatrace” ou bien essayer d’activer les statistiques Hibernate comme suit :

```yaml
spring:
  jpa:
    properties:
      hibernate.generate_statistics: true
```

PS : Il n'est pas recommandé de laisser activée la collecte des statistiques Hibernate en prod, car cela risque de ralentir d'avantage votre application.

>Pas d’économies de bouts de chandelles, attaquez de front le plus gros. 

Alléger la mémoire des objets qui prennent le plus d’espace. 
Optimiser les requêtes les plus longues. 
Diminuer les requêtes les plus exécutées.

## Phase III : Implémentations des solutions

Commençons par un peu de théorie.
Quand vous manipulez Hibernate, Il faut s'attendre à deux comportements capables de flinguer votre performance. 
Le premier est la N+1 query : un comportement de chargement lazy des relations One To Many. Hibernate génère une requête pour récupérer la liste des éléments attachés à l'entité mère, puis itère sur les N éléments dans la relation pour les récupérer un par un de la base de données, avec une requête select dédiée. Donc nous finissons avec N+1 select générés pour chaque relation One To Many

Le deuxième problème est la taille des requêtes SQL. Hibernate cache les requêtes SQL générées en mémoire afin de les réexcuter directement lors d'un second appel. Si les requêtes sont trop verbeuses et peu réexecutées, cela pourrait occuper beaucoup de mémoire pour rien.

### 1. Augmenter la visibilité sur le comportement de Hibernate
Mettre le ```show_sql``` à true dans la phase de develeoppement pour voir les queries générées par hibernate.
```yaml
spring:
  jpa:
    show-sql: true
```
### 2. La prévention des problèmes Hibernate
Utiliser [```@QuickPerf```](https://github.com/quick-perf/quickperf) sur les tests d’intégration pour détecter les N+1 au plus vite. Ça nécessite un test d'intégration sur la base de données
```java
@ExpectSelect(1)
@Test
public void should_find_all_players() {
     ...
}
```
quand la méthode génère plus de requêtes que de prévu, le test fail en affichant l'erreur suivante : 
```
[PERF] You may think that <1> select statement was sent to the database
       But there are in fact <10>...

Perhaps you are facing an N+1 select issue
	* With Hibernate, you may fix it by using JOIN FETCH
	                                       or LEFT JOIN FETCH
	                                       or FetchType.LAZY
	                                       or ...
```
### 3. Travailler votre indexation
 Indexer les colonnes et attributs utilisés dans vos recherches. Si vous avez un ```findByX```, assurez-vous d’avoir, dans la mesure du possible, un index sur le X. Il ne faut jamais sous-estimer l'impact d'un full scan table sur votre application. Surveillez de près les plans d'exécution de vos requêtes.
### 4. Attention aux effets de bords de Lombok
 Vérifier convenablement vos equals et hashcode. Si vous utilisez Lombok, faites attention à ne pas inclure les relations @OneToMany dans l'implémentation par défaut car vous allez réveiller la bête à chaque fois votre entité est manipulée dans une collection
### 5. Amortir le choc avec le batch padding 
Utiliser le ```padded``` pour le ```batch_fetch_style``` avec un size( 8 ou 16) pour que vos N+1 queries soient transformées en un batch de quelques requêtes avec une IN Clause.
```yaml
spring:
  jpa:
    hibernate:
      batch_fetch_style: PADDED
      default_batch_fetch_size: 16
```
### 6. Contrôler la navigation de votre graphe 
Utiliser la navigation ```@EntityGraph``` pour forcer la jointure avec les entités attachées et éviter les N+1. Surtout si vous manipulez un agrégat. Par exemple :
```java
@EntityGraph(attributePaths= {"myAggregateFirstChild", "myAggregateFirstChild.littleChild","myAggregateNephewsList",})
Optional<MyAggregate> findById(String id);
```
En spécifiant les attribute paths vous aurez une requête contenant toutes les jointures nécessaires pour récupérer tous les enfants de votre aggrégat.
### 7. Mesurer la mémoire occupée par Hibernate
 Surveiller votre ```QueryCachePlan```, car hibernate met en cache certaines des requêtes exécutées. Si vous avez des requêtes trop sollicitées avec une clause ```Where X IN``` clauses, vous risquez d’avoir autant de requêtes que de cardinalités possibles dans la in clause. Ça risque d’aller super loin et pour optimiser ceci, penser à réécrire la requête avec une jointure et une subquery.
 Par exemple, si vous avez une requête qui affiche, pour un client donné, toutes les commandes passées pour sa liste de produits préférés déjà précalculée.
 ```sql 
 select * from command where product.id in (?, ?, ?)
 ```
Nous aurons une première requête cachée pour le premier client ayant 3 produits préférés. Ensuite, une deuxième avec 10 paramètres pour un autre client, ainsi de suite.
A la place, il vaut mieux recalculer la liste des produits préférés de la façon suivante :
```sql 
 select * from command where product.id in (select product.id from preferred_product 
 left join client on client.id = preferred_product.client where client.id= ? )
 ```
 Quel que soit le nombre de combinaisons possibles d'articles préférées par client, nous aurons toujours une seule requête sql cachée en mémoire.
### 8. Opter pour le padding des IN clauses
 En effet, si vous n'arrivez pas à réécrire vos IN Clauses, vous pouvez utiliser le padding qui permet de répéter les valeurs passées en paramètre dans la requête afin de garder la même structure, et ne pas dupliquer la requête autant de fois qu'il y a de paramètres. La fonctionnalité de padding est présente par défaut dans Hibernate à partir de la version 5.2.18. Il suffit de l'activer comme suit :
 ```yaml
 hibernate:
  query:
   in_clause_parameter_padding: true
 ```
 Vous pouvez trouver de bons exemples dans [l'article de Vlad Mihalcea](https://vladmihalcea.com/improve-statement-caching-efficiency-in-clause-parameter-padding/).

 En effet, reprenons l'exemple de tout à l'heure. Nous souhaitons récupérer la liste de commandes des produits préférés d'un client donné. Afin d'éviter d'avoir une requête par nombre de combinaisons d'articles préférés par utilisateur, pour 5 produits préférés, un padding nous permettra d'avoir une requête ayant 8 paramètres mais avec des valeurs doublonnées comme suit : 
 ```sql 
 select * from command where product.id in (P1, P2, P3, P4, P5, P5, P5, P5)
 ```
Si votre liste contient 2 éléments, Hibernate va générer une clause IN avec 2 paramètres. Si votre liste contient 3 ou 4 éléments, la clause IN générée va en contenir 4, puis 8 pour 5 à 8 éléments. Si le nombre de produits préférés dépasse 8, nous passons à un palier de 16 et ainsi de suite.
La dernière valeur est répétée autant de fois que possible jusqu'à atteindre un palier de nombre de paramètres égal à la puissance de 2 la plus proche. 
### 9. Séparer votre modèle de lecture de celui de l'écriture
Vous n'êtes pas condamnés à utiliser les objets/entités de l'écriture dans la lecture. Il faut souvent redéfinir ses objets et récrire la façon avec laquelle on extrait des données de la base, afin de minimiser les entrées/sorties. Par exemple, supposons que vous avez un système de gestion de commandes client. Vous avez une interface dashboard sur laquelle vous avez un tableau résumant l'ensemble des commandes ouvertes avec la date de création et un statut de la commande. On peut imaginer un modèle LECTURE de l'entité commande qui ne restitue que les informations strictement nécessaires : nom de la commande, référence, date de création et statut. Le tout sans devoir passer par toutes les relations et informations relatives au client, produits, type de livraison etc...
Ceci nous permettra d'avoir des projections en base de données moins importantes et peu gourmandes en mémoire et en temps de traitement.

Supposons que vous voulez afficher un détail sur le client sur votre tableau et le type du produit commandé. Vous allez probablement faire appel à trois entités : Commande, Client et Produit. Au lieu de récupérer les trois entités pour fabriquer votre entrée du tableau dashboard, optez plutôt pour une requête hql qui récupère exactement les données à afficher dans un objet à part. 

```java
@Query(" select new howto.sample.CommandSummary(" +
  " com.id, com.date, cli.name, prod.type)  " +
  " from Command as com" +
  " left join  com.client as cli" +
  " left join com.product as prod")
List<CommandSummary> getAllCommandSummary();
```



### 10. Pensez à la performance dès la conception de l'interface graphique 
La performance d’une application commence dès la conception de son interface.
Si vous ne réfléchissez pas à l'avance aux cardinalité et volumétrie de vos données, vous risquez d'être surpris plus tard, quand la page commencera à mettre des plombes pour se charger. 
Cela peut être facilement évité en prévoyant une pagination, ou un bouton "load more".

## Conclusion
Il est très important d'avoir une sensibilité aux problématiques de performance sur toutes les phases du développement d'un produit. Du design de l'interface, jusqu'à la conception du modèle de données, la question de la performance doit toujours être présente. Malheureusement, un bon design ne suffit pas. En effet, il faut être conscient du fonctionnement des outils et des dépendances qui tournent dans l'application. La maîtrise d'Hibernate et de Lombok est primordiale pour éviter les mauvaises surprises.
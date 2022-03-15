---
layout: post
title:  "Un BPM Engine, tu n'utiliseras point !"
date:   2022-03-12 16:47:20 +0200
categories: java springboot process
---
<p class="message">
  Les BPM sont des outils très intéressants. Par contre, il s'agit d'une solution technique qui apporte une complexité sous-jacente assez conséquente. Dans le monde de l'entreprise, on y fait ce recours très souvent pour modéliser des process métiers à point de devenir un choix par défaut assez dogmatique, un nième marteau de Maslow. Il est intéressant de voir d'autres alternatives à cette solution.
</p>

<img style="display: flex; justify-content: center;" width="40%" src="/assets/img/manwithbighammer.svg">{:style="display:block; margin-left:auto; margin-right:auto"}

## Le BPM, la descente aux enfers

Tous les éditeurs de BPM workflow engines font l'éloge de leur invention et sa [supériorité sur la state machine](https://workflowengine.io/blog/workflow-engine-vs-state-machine/). Ils prétendent être plus déterministes, plus lisibles, plus maintenables. Des promesses non tenues ! Il est temps d'arrêter de céder à ce genre de discours marketing infondé et d'essayer de regarder les différentes alternatives qui pourraient répondre à un besoin de modélisation d'un processus métier.

J'ai travaillé dans des structures où les outils BPM ont tellement été des choix par défaut, que les product owners et business analystes appellent leur propre business workflow avec le terme " le BPM achat", " le BPM contractualisation".

Ce genre d'abus de langage est le symptôme de biais de l'instrument. Tout n'est pas forcément un processus modélisable en BPM. Au contraire, je pense même que c'est un overkill dans la plupart des cas.

> Quand on a qu'un marteau, toutes les vis ressemblent à des clous

Dans cet article, nous allons démontrer qu'un BPM est une solution difficile à faire évoluer. Elle peut avoir des conséquences lourdes sur la performance de votre application.

### 1. Un BPMN est une solution (trop) générique pour ton problème particulier

Il suffit de regarder [les tables](https://docs.camunda.org/manual/7.12/user-guide/process-engine/database/#history) générées dans un camunda : des tables Runtime, History etc
Ces tables seront toujours implantées dans votre schéma de données quelle que soit la complexité de votre process, quels que soient vos besoins.
Plusieurs tables seront mises à jour sans que vous n'en ayez besoin.
L'empreinte de données peut devenir très lourde si vous traitez un large volume de données.

### 2. Ton futur toi-même te maudira

La plus grande limite est le seul et unique point d'entrée que représente business key.
Toutes les autres données devront être enregistrées dans une structure de type : variable name, type and value. On est d'accord sur le fait qu'on ne veut pas faire du BPM un repo de données. Mais le peu de données qu'on y enregistre, étant pas normalisées, sont difficiles d'accès par une recherche ou requête SQL.

De surcroit, si vous avez fait le choix de sérialiser un objet, c'est une décision très périlleuse qui peut vous coûter très cher plus tard. La migration d'un tel modèle sera compliqué et nécessitera des étapes de désérialisation.

La migration de ces données, si votre système évolue, sera très compliquée, car vous aurez besoin d'écrire un programme qui parse des données sérialisées ou cherche le type de vos données dans une colonne à part.

### 3. Le reporting

On sous-estime souvent la complexité sous-jacente des besoins qui peuvent surgir au fil de l'eau quand il s'agit d'exploiter une workflow BPM. On pourrait à un moment donné avoir besoin de suivre certains KPI pour évaluer la performance d'un processus métier.

Deux erreurs fatales capables de détériorer la performance de votre application si vous implémentez un reporting.

La première consiste à faire un recours à l'api pour parser les données. Favoriser l'utilisation du SQL natif et vérouiller toute possibilité de passer par la heap memory.

La deuxième se résume à partager le même serveur de reporting avec celui de l'application.

La liste des [bonnes pratiques](https://camunda.com/best-practices/reporting-about-processes/) à suivre est encore longue. 

### 4. Mythe numéro 1 : les analystes pourront adapter facilement le processus.

Je n'ai jamais vu des business analyste ouvrir l'interface Cammunda et modifier le process en production sans qu'il n'y ait d'intervention d'un développeur.

Le changement du BPM aura certainement un impact sur l'application.
On aura peut-être besoin de mettre à jour l'api REST qui expose les services BPM et mettre à jour la communication avec les autres briques du SI.
Le changement de process nécessite peut-être déjà une mise à jour des briques du SI qui communiquent avec le BPM.

Donc l'argument, on utilise BPM pour que l'analyse soit totalement autonome sur la gestion et la maintenance du process est un mythe.

### 5. Mythe numéro 2 : on a besoin d'une gestion centralisée du process

Gérer l'état de ton process métier dans une brique indépendante, non liée à ton "bounded context" est un antipattern [[1]][1][[2]][2]. Cela est pour plusieurs raisons.
La première, la nécessité de consulter la brique du BPM pour savoir l'état de ton processus métier va créer un couplage très fort entre ton bounded context et le BPM.
La deuxième, la performance va en pâtir.
La troisième, l'atomicité de tes opérations devient d'une complexité aberrante.
La quatrième, le domaine de ton bounded context est privé d'un attribut majeur : l'état d'avancement dans un workflow.

## L'alternative : Spring State Machine

### 1. Qu'est ce qu'une state machine?

Une machine à état est un automate ayant des états définis et distincts. Elle transite d'un état à un autre à chaque fois qu'un événement surgit.

Il s'agit de modéliser le problème sous un autre angle ; au lieu de se focaliser sur des taches, il faudra plutôt d'abord définir les états de ton système : un état initial, un état final et des états intermédiaires. Ensuite, définir les événements qui permettent de transiter d'un état à un autre. Il suffit de le schématiser comme suit :

<img src="https://www.planttext.com/api/plantuml/svg/RP2nRW8n38PtFuLd95w0eKBgBKLYAMoeWo_vuI92SYJEmUCtEO54AnPBqVVPpkzrLabQBfv8dNhmnYNXdOg2js865q27nGylbn-yZpPIA_FhAtIOuEDuGL1USOQ7KHPMPyvG-ijRnsUq-CRaSAlwMFB0VP9W1de1xoOVtPsFWEt5dFD_UO-iBfog9kEOOfbGHtlF2TTI4JrvSxiOKCL9lBCjuEEdhyeShwwC9TYQfIydchiQutO8eZM2hGVx1G00">{:style="display:block; margin-left:auto; margin-right:auto"}

### 2. Une implémentation (presque) parfaite

Tout d'abord, je vous invite à ajouter cette dépendance dans votre projet Spring Boot :

```xml
<dependency>
    <groupId>org.springframework.statemachine</groupId>
    <artifactId>spring-statemachine-core</artifactId>
    <version>3.0.1</version>
</dependency>
```

Il suffit de transcrire ce beau schéma en langage Java.

Commençons d'abord par nos états. Il suffit d'écrire tout ce qui existe dans un cercle dans une énumération State.

```java
public enum TicketState {
  TODO,
  IN_DEVELOPMENT,
  CODE_REVIEW,
  TESTING,
  DONE
}
```

Ensuite, on définit pour nos événements, il suffira d'écrire tout ce qui existe sur un arc dans une énumération.

```java
public enum TicketEvent {
SELECT_A_TICKET,
FIX_FEEDBACK,
PUSH_BRANCH,
FOUND_A_BUG,
APPROUVE_PULL_REQUEST,
VALIDATE_TICKET
}
```

Finalement, définissons notre state machine

```java
@Configuration
@EnableStateMachine
public class StateMachineConfig extends EnumStateMachineConfigurerAdapter<TicketState, TicketEvent> {

    @Override
    public void configure(StateMachineConfigurationConfigurer<TicketState, TicketEvent> config)
            throws Exception {
        config.withConfiguration()
                .autoStartup(true);
    }
    @Override
    public void configure(StateMachineStateConfigurer<TicketState, TicketEvent> states)
            throws Exception {
        states.withStates()
                .initial(TicketState.TODO)
                .states(EnumSet.allOf(TicketState.class));
    }

    @Override
    public void configure(StateMachineTransitionConfigurer<TicketState, TicketEvent> transitions)
            throws Exception {
        transitions
                .withExternal()
                    .source(TicketState.TODO)
                    .target(TicketState.IN_DEVELOPMENT)
                    .event(TicketEvent.SELECT_A_TICKET)
                .and()
                .withExternal()
                    .source(TicketState.IN_DEVELOPMENT)
                    .target(TicketState.CODE_REVIEW)
                    .event(TicketEvent.PUSH_BRANCH)
                .and()
                .withExternal()
                    .source(TicketState.CODE_REVIEW)
                    .target(TicketState.IN_DEVELOPMENT)
                    .event(TicketEvent.FIX_FEEDBACK)
                .and()
                .withExternal()
                    .source(TicketState.CODE_REVIEW)
                    .target(TicketState.TESTING)
                    .event(TicketEvent.APPROUVE_PULL_REQUEST)
                .and()
                .withExternal()
                    .source(TicketState.TESTING)
                    .target(TicketState.TODO)
                    .event(TicketEvent.FOUND_A_BUG)
                .and()
                .withExternal()
                    .source(TicketState.TESTING)
                    .target(TicketState.DONE)
                    .event(TicketEvent.VALIDATE_TICKET);
    }


}

```

Vous pouvez trouver une large palette d'exemples dans la documentation officielle[[3]][3]
### 3. Une persistance à la carte

On peut facilement persister les informations qu'on juge nécessaires et suffisantes. Pas besoin de tables Runtime/History. Pour l'état de mon ticket Jira, je peux le persister dans l'entité Ticket. Si j'ai besoin d'un audit de l'historique des transitions et des états. Je peux persister si je veux la personne attitrée au ticket.

J'ai une flexibilité totale de persister les informations les plus utiles et je pourrais les requêter d'une façon intuitive et classique.

### 4. Une intégration facile dans l'application

En considérant chaque action comme un événement, j'ai la possibilité de lié l'événement à d'autres actions dans l'application. Je peux aussi, dérouler un ensemble d'actions en conséquence de cet événement.


La machine à état fait partie de mon application et peut facilement interagir avec n'importe quel concept du domaine.
Je n'ai pas d'effort à fournir pour synchroniser deux mondes parallèles : l'application métier et la brique Workflow.

Les deux sont implémentées dans le même langage (Java) sur le même framework (Spring)

### 5. Le choix conceptuel

Prenons l'exemple d'un processus de 4-eyes validation.
Un premier acteur soumet un document pour revue et validation par deux de ses pairs.
Le système d'information soumet le document dans un état 'Under Review'. L'action de la revue est effectuée par un humain et non par le système en soit. La vérification du pair est un événement externe au système d'information.

En contrepartie, le document affiché par le système, lui en état 'Under Review' est bien une entité du domaine du sytème d'information. Ce dernier devrait s'assurer de son immutabilité, de notifier les bons intervenants, etc.

De ce fait, le système gère des états et ce qui en découlent : des notifications, une immutabilité etc.

Il est de ce fait intéressant que les états soient bien gérés par le domaine en soi et pas un sous-système externe avec lequel on risque d'avoir un couplage très fort.

## Conclusion

Nous avons démontré dans cet article qu'un BPM Engine n'est pas forcément la meilleure solution à la modélisation des processus métiers.

Il apporte une complexité non-négligeable par son empreinte Data gigantesque.
Il peut même causer des problèmes de performance sur le long terme.

Le choix d'un BPM rend la gestion de l'état du processus difficile à cerner par le système d'information et met devant le risque d'un couplage très fort au BPM engine.

D'un point de vue conceptuel, dans une approche DDD, la perception de l'état des entités et agrégats prévaut à celle de la tâche.
Cette dernière, peut être considérée comme un événement externe au système d'information.

De ce fait, nous avons aussi vu, qu'une machine à état peut être plus flexible et facilement intégrable.
Tout ceci était agrémenté par des exemples d'implémentations en spring-state-machine.

Pour clôturer, il faudra toujours être conscient du biais du marteau d'or, la loi d'instrument. Un outil, aussi puissant et complet qu'il soit, ne peut pas être la solution magique à tous les problèmes.

## References

[1]: <https://blog.bernd-ruecker.com/avoiding-the-bpm-monolith-when-using-bounded-contexts-d86be6308d8> "Avoiding the “BPM monolith” when using bounded contexts"
[2]: <https://blog.bernd-ruecker.com/the-7-sins-of-workflow-b3641736bf5c> "The 7 sins of workflow"
[3]: https://docs.spring.io/spring-statemachine/docs/current/reference/#statemachine-examples "Une disaine d'exemples de state machine"

[[1]] "Avoiding the “BPM monolith” when using bounded contexts"

[[2]] "The 7 sins of workflow"

[[3]] Une disaine d'exemples de state machine



---
layout: post
title:  "Les leviers de performance d'une application Angular"
date:   2022-11-06 16:47:20 +0200
categories: Angular
---

J'ai travaillé sur l'optimisation d'une application Angular. Nous avons mené plusieurs actions qui ont drastiquement changé la performance de l'application. Si je résume les leviers de perfomance, cela pourrait être principalement les suivants :

- Change Detection
- Nombre des noeuds dans le DOM
- Taille des bundles
- Nombre d'interaction serveur


### 1. Eviter les appels de fonction depuis le template :

   Angular n'est pas capable de prédire si la valeur retournée par les méthodes inscrites dans les templates a été modifiée.

   De ce fait, à chaque cycle de détection de changement, Angular exécutera toutes ces méthodes afin de calculer sa nouvelle vue. 

   ```html
   <div>
      <div *ngIf="!isTrialSubscription()">{{getTrialExpirationDate()}}</div>
    </div>
   ```
   
   Donc, au lieu d'appeler directement la méthode renvoyant la valeur attendue. Il est plus intéressant d'utiliser un attribut du composant et s'assurer que ce dernier a été bien rempli au moment opportun ; soit lors d'un événnement utilisateur (onClick) ou dans une phase du cycle du vie du composant (onInit()) ou autre.

   ```html
   <div>
      <div *ngIf="trialSubscription">{{trialExpirationDate}}</div>
   </div>
   ```
   Vous allez ainsi libérer votre call stack et rendre la navigation beaucoup plus fluide. Vous allez surtout entendre beaucoup moins votre ventilateur souffler quand vous ouvrez la page web.

### 2. Basculer en OnPush
   La fameuse change detection strategy est un concept primordial dans la performance de votre application.

   En fait, à chaque événnement, Angular calculera la change detection sur toute l'arborescence des composants.

   Ce mécanisme coûteux, pourrait être optimisé en permettant à Angular de le lancer uniquement sur le composant recevant des évennement structurant (des inputs par exemple) et couper le calcul sur tous les subtrees de ce composant.

   Ceci nécessite une structure bien particulière. Il faut que votre découpage de composants soit de telle sorte qu'à la racine de l'arbre, on a des composants dit intelligents : capable d'intéragir avec des façades et des services alimentant la vue.
   Et des composants moins intelligents appelés dummy components qui ne font qu'afficher les données qui leur sont injectées en @Input par leurs parents.

   Ces dummy components seront en ChangeDetection.OnPush et seront faciles à couper de la change detection, car tout ce qui conditionne leur changement n'est que les @Input. Un DummyComponent aura cette forme :

```typescript
@Component({
   selector:'dummy-comp-selector',
   templateUrl: './path-to-my-dummy-component.html',
   styleUrls: ['./path-to-dummy.scss'],
   changeDetection: ChangeDetection.OnPush,
})

export class MyDummyComponent{

   @Input() firstInputValue: FirstInputValueClass;
   @Input() secondInputValue: SecondInputValueClass;

   @Ouput() actionCompleted = new EventEmitter();
}
```



### 3. | async > subscribe :
Au lieu d'utiliser la souscription aux observables, il est plus recommandé d'utiliser le pipe async car :
* Il vous économisera l'erreur d'oublier de unsubscribe de votre observable et du coup éviter des cycles de change detection totalement aléatoires qui vous rendront fous

* Fonctionne parfaitement avec le change dection onPush

Vous pouvez lire plus de détails [ici](https://medium.com/angular-in-depth/angular-question-rxjs-subscribe-vs-async-pipe-in-component-templates-c956c8c0c794).
   


### 4. Isoler le chargement  des données :

Le chargement de données et les interactions avec le backend doit être finement maitrisé. La moindre confusion peut créer des doublons d'appels au serveur, ou des événnements de modification non maitrisés.

Pour enlever toute confusion, il faudra appliquer un concept de base dans le développement "Separation of concerns".

Le composant ne doit pas savoir les détails des services appelés. Il a juste besoin de charger les données.

Il serait intéressant alors d'utiliser des façades comme suit :

```typescript
constructor (private myComponentFacade: ComponentFacade) {}


ngOnInit(){
   if (this.myInput && this.myRouterValue) {
      this.myComponentFacade.loadData(this.myInput, this.myRouterValue);
   }
}
```
Ainsi, le code est plus simple à tester. Les problèmes d'interaction avec le serveur peuvent être facilement isolés et résolus.


### 5. Ne pas copier les modèles d'API :

Une Api peut changer ses contrats : ce que vous pensiez plus simple à faire, devient un cauchemar !
Vous perdez la propriété de votre modèle : le backend détermine la manière dont vous chargez les données. 

L'idéal serait de concevoir son modèle, le mapper et l'hydrater avec l'api. Mettre une couche de non corruption où vous valider les données, les mapper, les construire à partir de plusieurs sources s'il le faut. 

### 6. Ne pas partager les composants trop rapidement
J'ai vu pas mal de régressions généralisées par ce qu'on part avec l'intension de "généraliser" un composant et le rendre générique. Je pense qu'un design System suffit comme socle de composants partagés. 

### 7. L'UX peut tuer votre application !
Choisissez judicieusement un Design System et respectez-le. Ne laissez pas la personnalisation violer votre Design system.

D'autre part, regardez ce que vous chargez dans le dom. N'autorisez JAMAIS les collections de données non paginées.

Si c'est inévitable, cherchez à utiliser, localement, le virtual scrolling et évitez à tout prix de tout mettre dans le DOM, car vous allez atteindre très rapidement la limite physique de la machine de votre utilisateur.

### 8. Les composants fancy coûtent super chers
Je pense aux éditeurs de texte enrichi comme [Tiny-MCE](https://www.tiny.cloud/solutions/wysiwyg-angular-rich-text-editor/) ou les librairies de gestion de tableaux comme [ag-grid](https://www.ag-grid.com/). Ils sont agréables à utiliser et riches en fonctionnalités. Mais la quantité de noeuds injectée dans le dom est phénoménale. Si vous pouvez servir le même besoin avec ce que votre Design system framework vous offre, contentez vous de cela.

### 9. Micro Frontend au lieu de monolithe
Si votre application est décomposable en plusieurs bounded contexts, ou si vous avez des écrans dédiés à différentes populations d'utilisateurs, n'hésitez pas d'en faire plusieurs applications. Ceci allégera énormément la taille des bundles. Cette décomposition pourrait même rendre la gestion des versions plus cohérente.

### 10. Mettre à jour dès que possible
Cela peut praître trivial, mais il est important de rappeler que l'upgrade dans les framework front et surtout Angular est inévitable. Si vous prenez trop de temps pour mettre à niveau, ce sera vraiment très pénible par la suite. L'upgrade donne des correctifs de sécurité gratuits
et améliore le temps de build. C'est un quick-win incontournable.


## Conclusion


Toutes ces actions ont été menées dans notre quête de rendre l'application plus performante. Les résultats sont marquants et l'impact est vraiment tangible ; sur le temps de chargement de la page, la fluidité de navigation et la réactivité des composants.
Je ne me suis pas trop attardé sur le lazy loading dans le router. 

Le point d'attention plus crucial est à mettre sur le change detection de Angular. C'est un mécanisme qui peut paraître assez obscur, mais il faudra faire ce qu'il faut pour pallier les dégâts qu'il pourrait causer.
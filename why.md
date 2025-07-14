# Pourquoi faire des tests unitaires et comment les faire fonctionner pour vous

[Voici un lien vers une vidéo où je discute de ce sujet](https://www.youtube.com/watch?v=Kwtit8ZEK7U)

Si vous n'aimez pas les vidéos, voici une version écrite.

## Logiciel 

La promesse d'un logiciel (_software_) est qu'il peut changer. C'est pourquoi on l'appelle _soft_ ware, il est malléable par rapport au matériel (_hardware_). Une grande équipe d'ingénierie devrait être un atout incroyable pour une entreprise, en développant des systèmes qui peuvent évoluer avec l'entreprise pour continuer à apporter de la valeur. 

Alors pourquoi sommes-nous si mauvais à cela ? Combien de projets entendez-vous parler qui échouent carrément ? Ou qui deviennent "legacy" et doivent être entièrement réécrits (et les réécritures échouent souvent aussi !) 

Comment un système logiciel peut-il "échouer" de toute façon ? Ne peut-il pas simplement être modifié jusqu'à ce qu'il soit correct ? C'est ce qui nous est promis !

Beaucoup de gens choisissent Go pour construire des systèmes parce qu'il a fait un certain nombre de choix qui, on l'espère, le rendront plus résistant au vieillissement. 

- Par rapport à ma période passée avec Scala où [je décrivais comment il donne suffisamment de corde pour se pendre](http://www.quii.dev/Scala_-_Just_enough_rope_to_hang_yourself), Go n'a que 25 mots-clés et _beaucoup_ de systèmes peuvent être construits à partir de la bibliothèque standard et de quelques autres petites bibliothèques. L'espoir est qu'avec Go, vous pouvez écrire du code et y revenir dans 6 mois et il aura toujours du sens.
- Les outils concernant les tests, le benchmarking, le linting et le déploiement sont de première classe par rapport à la plupart des alternatives.
- La bibliothèque standard est brillante.
- Vitesse de compilation très rapide pour des boucles de feedback fréquentes
- La promesse de compatibilité ascendante de Go. Il semble que Go obtiendra des génériques et d'autres fonctionnalités à l'avenir, mais les concepteurs ont promis que même le code Go que vous avez écrit il y a 5 ans continuera à compiler. J'ai littéralement passé des semaines à mettre à niveau un projet de Scala 2.8 à 2.10. 

Même avec toutes ces excellentes propriétés, nous pouvons encore créer des systèmes terriblement mauvais, nous devrions donc nous tourner vers le passé et comprendre les leçons de l'ingénierie logicielle qui s'appliquent, peu importe à quel point votre langage est brillant (ou pas).

En 1974, un ingénieur logiciel intelligent nommé [Manny Lehman](https://en.wikipedia.org/wiki/Manny_Lehman_%28computer_scientist%29) a écrit [les lois de Lehman sur l'évolution des logiciels](https://en.wikipedia.org/wiki/Lehman%27s_laws_of_software_evolution).

> Les lois décrivent un équilibre entre les forces qui stimulent les nouveaux développements d'une part, et les forces qui ralentissent le progrès d'autre part.

Ces forces semblent être des éléments importants à comprendre si nous avons un quelconque espoir de ne pas être dans un cycle sans fin de livraison de systèmes qui se transforment en héritage puis sont réécrits encore et encore.

## La Loi du Changement Continu

> Tout système logiciel utilisé dans le monde réel doit changer ou devenir de moins en moins utile dans l'environnement

Il semble évident qu'un système _doit_ changer ou il devient moins utile, mais combien de fois cela est-il ignoré ? 

De nombreuses équipes sont incitées à livrer un projet à une date particulière, puis à passer au projet suivant. Si le logiciel a de la "chance", il y a au moins une sorte de transfert à un autre ensemble d'individus pour le maintenir, mais ils ne l'ont pas écrit, bien sûr. 

Les gens se préoccupent souvent d'essayer de choisir un framework qui les aidera à "livrer rapidement", mais ne se concentrent pas sur la longévité du système en termes d'évolution nécessaire.

Même si vous êtes un ingénieur logiciel incroyable, vous serez toujours victime de ne pas connaître les besoins futurs de votre système. Au fur et à mesure que l'entreprise change, une partie du code brillant que vous avez écrit n'est plus pertinente.

Lehman était en feu dans les années 70 car il nous a donné une autre loi à méditer.

## La Loi de la Complexité Croissante

> Au fur et à mesure qu'un système évolue, sa complexité augmente à moins qu'un travail ne soit effectué pour la réduire

Ce qu'il dit ici, c'est que nous ne pouvons pas avoir des équipes logicielles comme des usines de fonctionnalités aveugles, empilant de plus en plus de fonctionnalités sur un logiciel dans l'espoir qu'il survivra à long terme. 

Nous **devons** continuer à gérer la complexité du système à mesure que la connaissance de notre domaine change. 

## Refactoring

Il existe _de nombreuses_ facettes de l'ingénierie logicielle qui maintiennent le logiciel malléable, telles que :

- L'autonomisation des développeurs
- Code généralement "bon". Séparation sensée des préoccupations, etc.
- Compétences en communication
- Architecture
- Observabilité
- Déployabilité
- Tests automatisés
- Boucles de feedback

Je vais me concentrer sur le refactoring. C'est une expression qui est souvent utilisée "nous devons refactoriser ceci" - dit à un développeur dès son premier jour de programmation sans y réfléchir à deux fois. 

D'où vient cette expression ? En quoi le refactoring est-il différent de l'écriture de code ?

Je sais que moi et beaucoup d'autres avons _pensé_ faire du refactoring mais nous nous sommes trompés

[Martin Fowler décrit comment les gens se trompent](https://martinfowler.com/bliki/RefactoringMalapropism.html)

> Cependant, le terme "refactoring" est souvent utilisé lorsqu'il n'est pas approprié. Si quelqu'un parle d'un système qui est en panne pendant quelques jours pendant qu'il refactorise, vous pouvez être à peu près sûr qu'il ne fait pas de refactoring.

Alors qu'est-ce que c'est ?

### Factorisation

Lorsque vous avez appris les mathématiques à l'école, vous avez probablement appris la factorisation. Voici un exemple très simple

Calculez `1/2 + 1/4`

Pour ce faire, vous _factorisez_ les dénominateurs, transformant l'expression en 

`2/4 + 1/4` que vous pouvez ensuite transformer en `3/4`. 

Nous pouvons en tirer des leçons importantes. Lorsque nous _factorisons l'expression_, nous n'avons **pas changé la signification de l'expression**. Les deux égalent `3/4` mais nous l'avons rendue plus facile à manipuler ; en changeant `1/2` en `2/4`, elle s'intègre plus facilement dans notre "domaine". 

Lorsque vous refactorisez votre code, vous essayez de trouver des moyens de le rendre plus facile à comprendre et à "s'adapter" à votre compréhension actuelle de ce que le système doit faire. Crucialement, **vous ne devriez pas changer le comportement**. 

#### Un exemple en Go

Voici une fonction qui salue `name` dans une `language` particulière

```go
    func Hello(name, language string) string {
    
      if language == "es" {
         return "Hola, " + name
      }
    
      if language == "fr" {
         return "Bonjour, " + name
      }
      
      // imaginez des dizaines d'autres langues
    
      return "Hello, " + name
    }
```

Avoir des dizaines d'instructions `if` ne semble pas bon et nous avons une duplication de la concaténation d'une salutation spécifique à la langue avec `, ` et le `name`. Je vais donc refactoriser le code.

```go
func Hello(name, language string) string {
   	return fmt.Sprintf(
  		"%s, %s",
  		greeting(language),
  		name,
   	)
}

var greetings = map[string]string {
    "es": "Hola",
    "fr": "Bonjour",
    //etc..
}

func greeting(language string) string {
    greeting, exists := greetings[language]
    
    if exists {
        return greeting
    }
    
    return "Hello"
}
```

La nature de ce refactoring n'est pas vraiment importante, ce qui est important c'est que je n'ai pas changé le comportement. 

Lorsque vous refactorisez, vous pouvez faire ce que vous voulez, ajouter des interfaces, de nouveaux types, des fonctions, des méthodes, etc. La seule règle est que vous ne changez pas le comportement

### Lorsque vous refactorisez du code, vous ne devez pas changer le comportement

C'est très important. Si vous changez le comportement en même temps, vous faites _deux_ choses à la fois. En tant qu'ingénieurs logiciels, nous apprenons à décomposer les systèmes en différents fichiers/packages/fonctions/etc car nous savons qu'essayer de comprendre un gros bloc de choses est difficile. 

Nous ne voulons pas avoir à penser à beaucoup de choses à la fois car c'est à ce moment-là que nous faisons des erreurs. J'ai été témoin de nombreuses tentatives de refactoring qui ont échoué parce que les développeurs en prennent plus qu'ils ne peuvent en mâcher.  

Lorsque je faisais des factorisations dans les cours de mathématiques avec un crayon et du papier, je devais vérifier manuellement que je n'avais pas changé le sens des expressions dans ma tête. Comment savons-nous que nous ne changeons pas le comportement lors du refactoring lorsque nous travaillons avec du code, en particulier sur un système qui n'est pas trivial ?

Ceux qui choisissent de ne pas écrire de tests dépendront généralement des tests manuels. Pour tout projet autre qu'un petit projet, cela sera un énorme gouffre de temps et ne s'adaptera pas à long terme. 
 
**Pour refactoriser en toute sécurité, vous avez besoin de tests unitaires** car ils fournissent

- La confiance que vous pouvez remodeler le code sans vous soucier de changer le comportement
- De la documentation pour les humains sur la façon dont le système devrait se comporter
- Un feedback beaucoup plus rapide et fiable que les tests manuels

#### Un exemple en Go

Un test unitaire pour notre fonction `Hello` pourrait ressembler à ceci
```go
func TestHello(t *testing.T) {
    got := Hello("Chris", es)
    want := "Hola, Chris"

    if got != want {
        t.Errorf("got %q want %q", got, want)
    }
}
```

En ligne de commande, je peux exécuter `go test` et obtenir un feedback immédiat pour savoir si mes efforts de refactoring ont modifié le comportement. En pratique, il est préférable d'apprendre le bouton magique pour exécuter vos tests dans votre éditeur/IDE. 

Vous voulez arriver à un état où vous faites 

- Petit refactoring
- Exécuter les tests
- Répéter

Le tout dans une boucle de feedback très serrée pour ne pas vous perdre dans des détails et faire des erreurs.

Avoir un projet où tous vos comportements clés sont testés unitairement et vous donnent un feedback bien en dessous d'une seconde est un filet de sécurité très puissant pour faire des refactoring audacieux lorsque vous en avez besoin. Cela nous aide à gérer la force entrante de complexité que Lehman décrit.

## Si les tests unitaires sont si géniaux, pourquoi y a-t-il parfois une résistance à les écrire ?

D'un côté, vous avez des personnes (comme moi) qui disent que les tests unitaires sont importants pour la santé à long terme de votre système car ils garantissent que vous pouvez continuer à refactoriser en toute confiance. 

De l'autre, vous avez des personnes qui décrivent des expériences où les tests unitaires _entravent_ en réalité le refactoring.

Demandez-vous, combien de fois devez-vous changer vos tests lors du refactoring ? Au fil des ans, j'ai participé à de nombreux projets avec une très bonne couverture de tests et pourtant les ingénieurs sont réticents à refactoriser en raison de l'effort perçu pour changer les tests.

C'est l'opposé de ce qui nous est promis !

### Pourquoi cela se produit-il ?

Imaginez qu'on vous demande de développer un carré et que nous pensions que la meilleure façon d'y parvenir serait de coller deux triangles ensemble. 

![Deux triangles rectangles pour former un carré](https://i.imgur.com/ela7SVf.jpg)

Nous écrivons nos tests unitaires autour de notre carré pour nous assurer que les côtés sont égaux, puis nous écrivons des tests autour de nos triangles. Nous voulons nous assurer que nos triangles sont correctement rendus, alors nous affirmons que la somme des angles est de 180 degrés, peut-être vérifions-nous que nous en faisons 2, etc. La couverture de test est vraiment importante et écrire ces tests est assez facile alors pourquoi pas ? 

Quelques semaines plus tard, la Loi du Changement Continu frappe notre système et une nouvelle développeuse fait des changements. Elle croit maintenant qu'il serait préférable que les carrés soient formés de 2 rectangles au lieu de 2 triangles. 

![Deux rectangles pour former un carré](https://i.imgur.com/1G6rYqD.jpg)

Elle essaie de faire ce refactoring et reçoit des signaux contradictoires de la part d'un certain nombre de tests qui échouent. A-t-elle réellement cassé des comportements importants ici ? Elle doit maintenant fouiller dans ces tests de triangles et essayer de comprendre ce qui se passe. 

_Ce n'est pas vraiment important que le carré ait été formé à partir de triangles_, mais **nos tests ont faussement élevé l'importance de nos détails d'implémentation**. 

## Préférez tester le comportement plutôt que les détails d'implémentation

Quand j'entends des gens se plaindre des tests unitaires, c'est souvent parce que les tests sont au mauvais niveau d'abstraction. Ils testent les détails d'implémentation, espionnent excessivement les collaborateurs et simulent trop. 

Je crois que cela provient d'une mauvaise compréhension de ce que sont les tests unitaires et de la poursuite vaine d'augmentation de certaines métriques (couverture de test). 

Si je dis simplement de tester le comportement, ne devrions-nous pas simplement écrire des tests système/boîte noire ? Ces types de tests ont beaucoup de valeur en termes de vérification des parcours utilisateurs clés, mais ils sont généralement coûteux à écrire et lents à exécuter. Pour cette raison, ils ne sont pas très utiles pour le _refactoring_ car la boucle de feedback est lente. De plus, les tests de boîte noire ne vous aident généralement pas beaucoup avec les causes racines par rapport aux tests unitaires. 

Alors quel est le bon niveau d'abstraction ?

## Écrire des tests unitaires efficaces est un problème de conception

En oubliant les tests un instant, il est souhaitable d'avoir au sein de votre système des "unités" autonomes et découplées centrées autour des concepts clés de votre domaine. 

J'aime imaginer ces unités comme de simples briques Lego qui ont des API cohérentes que je peux combiner avec d'autres briques pour créer des systèmes plus grands. Sous ces API, il pourrait y avoir des dizaines de choses (types, fonctions, etc.) collaborant pour les faire fonctionner comme elles en ont besoin.

Par exemple, si vous écriviez une banque en Go, vous pourriez avoir un package "account". Il présentera une API qui ne laisse pas fuir les détails d'implémentation et qui est facile à intégrer.

Si vous avez ces unités qui suivent ces propriétés, vous pouvez écrire des tests unitaires contre leurs API publiques. _Par définition_, ces tests ne peuvent tester que des comportements utiles. Sous ces unités, je suis libre de refactoriser l'implémentation autant que j'en ai besoin et les tests ne devraient pas, pour la plupart, se mettre en travers du chemin.

### Est-ce que ce sont des tests unitaires ?

**OUI**. Les tests unitaires sont contre des "unités" comme je l'ai décrit. Ils n'ont _jamais_ été uniquement contre une seule classe/fonction/quoi que ce soit.

## Rassembler ces concepts

Nous avons couvert

- Le refactoring
- Les tests unitaires
- La conception d'unité

Ce que nous pouvons commencer à voir, c'est que ces facettes de la conception logicielle se renforcent mutuellement. 

### Refactoring

- Nous donne des signaux sur nos tests unitaires. Si nous devons faire des vérifications manuelles, nous avons besoin de plus de tests. Si les tests échouent à tort, alors nos tests sont au mauvais niveau d'abstraction (ou n'ont aucune valeur et devraient être supprimés).
- Nous aide à gérer les complexités au sein et entre nos unités.

### Tests unitaires

- Donnent un filet de sécurité pour refactoriser.
- Vérifient et documentent le comportement de nos unités.

### Unités (bien conçues)

- Faciles à écrire des tests unitaires _significatifs_.
- Faciles à refactoriser.

Existe-t-il un processus pour nous aider à arriver à un point où nous pouvons constamment refactoriser notre code pour gérer la complexité et garder nos systèmes malléables ?

## Pourquoi faire du Développement Piloté par les Tests (TDD)

Certaines personnes pourraient prendre les citations de Lehman sur la façon dont le logiciel doit changer et réfléchir de manière excessive à des conceptions élaborées, perdant beaucoup de temps au départ en essayant de créer le système extensible "parfait" et finir par se tromper et n'aller nulle part. 

Ce sont les mauvais vieux jours du logiciel où une équipe d'analystes passerait 6 mois à écrire un document d'exigences et une équipe d'architectes passerait 6 autres mois à concevoir un design, et quelques années plus tard, l'ensemble du projet échoue.

Je dis les mauvais vieux jours, mais cela se produit encore ! 

L'Agile nous enseigne que nous devons travailler de manière itérative, en commençant petit et en faisant évoluer le logiciel afin d'obtenir un feedback rapide sur la conception de notre logiciel et sur la façon dont il fonctionne avec de vrais utilisateurs ; le TDD impose cette approche.

Le TDD aborde les lois dont parle Lehman et d'autres leçons durement apprises à travers l'histoire en encourageant une méthodologie de refactoring constant et de livraison itérative.

### Petits pas

- Écrire un petit test pour une petite quantité de comportement souhaité
- Vérifier que le test échoue avec une erreur claire (rouge)
- Écrire la quantité minimale de code pour faire passer le test (vert)
- Refactoriser
- Répéter

Au fur et à mesure que vous devenez compétent, cette façon de travailler deviendra naturelle et rapide.

Vous vous attendrez à ce que cette boucle de feedback ne prenne pas beaucoup de temps et vous vous sentirez mal à l'aise si vous êtes dans un état où le système n'est pas "vert", car cela indique que vous êtes peut-être dans un terrier de lapin. 

Vous conduirez toujours de petites fonctionnalités utiles confortablement soutenues par le feedback de vos tests.

## Résumé 

- La force du logiciel est que nous pouvons le changer. _La plupart_ des logiciels nécessiteront des changements au fil du temps de manière imprévisible ; mais n'essayez pas de sur-ingénierer car il est trop difficile de prédire l'avenir.
- Au lieu de cela, nous devons faire en sorte que nous puissions garder notre logiciel malléable. Pour changer un logiciel, nous devons le refactoriser à mesure qu'il évolue, sinon il se transformera en désordre.
- Une bonne suite de tests peut vous aider à refactoriser plus rapidement et de manière moins stressante.
- Écrire de bons tests unitaires est un problème de conception, alors pensez à structurer votre code pour avoir des unités significatives que vous pouvez intégrer ensemble comme des briques Lego.
- Le TDD peut vous aider et vous forcer à concevoir un logiciel bien factorisé de manière itérative, soutenu par des tests pour aider les travaux futurs à mesure qu'ils arrivent.
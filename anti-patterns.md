# Anti-patterns de TDD

De temps à autre, il est nécessaire de revoir vos techniques de TDD et de vous rappeler des comportements à éviter.

Le processus TDD est conceptuellement simple à suivre, mais en le pratiquant, vous constaterez qu'il met à l'épreuve vos compétences en conception. **Ne confondez pas cela avec le fait que le TDD est difficile, c'est la conception qui est difficile !**

Ce chapitre énumère un certain nombre d'anti-patterns de TDD et de tests, et comment y remédier.

## Ne pas faire de TDD du tout

Bien sûr, il est possible d'écrire d'excellents logiciels sans TDD mais, beaucoup de problèmes que j'ai vus avec la conception du code et la qualité des tests seraient très difficiles à rencontrer si une approche disciplinée du TDD avait été utilisée.

L'une des forces du TDD est qu'il vous donne un processus formel pour décomposer les problèmes, comprendre ce que vous essayez d'accomplir (rouge), y parvenir (vert), puis réfléchir sérieusement à la façon de le faire correctement (bleu/refactoring).

Sans cela, le processus est souvent ad hoc et lâche, ce qui _peut_ rendre l'ingénierie plus difficile qu'elle ne _pourrait_ l'être.

## Mauvaise compréhension des contraintes de l'étape de refactoring

J'ai participé à plusieurs ateliers, sessions de mobbing ou de pairing où quelqu'un a fait passer un test et se trouve dans la phase de refactoring. Après réflexion, cette personne pense qu'il serait bon d'abstraire une partie du code dans une nouvelle structure ; un pédant en herbe s'écrie :

> Vous n'êtes pas autorisé à faire cela ! Vous devriez d'abord écrire un test pour cela, nous faisons du TDD !

Cela semble être un malentendu courant. **Vous pouvez faire ce que vous voulez avec le code lorsque les tests sont verts**, la seule chose que vous n'êtes pas autorisé à faire est **d'ajouter ou de modifier le comportement**.

L'intérêt de ces tests est de vous donner la _liberté de refactoriser_, de trouver les bonnes abstractions et de rendre le code plus facile à modifier et à comprendre.

## Avoir des tests qui ne peuvent pas échouer (ou, des tests toujours verts)

Il est étonnant de voir combien de fois cela se produit. Vous commencez à déboguer ou à modifier certains tests et vous réalisez : il n'y a aucun scénario où ce test peut échouer. Ou du moins, il n'échouera pas de la manière dont le test est _censé_ protéger.

C'est _presque impossible_ avec TDD si vous suivez **la première étape**,

> Écrivez un test, voyez-le échouer

Cela se produit presque toujours lorsque les développeurs écrivent des tests _après_ que le code a été écrit, et/ou cherchent la couverture de tests plutôt que de créer une suite de tests utile.

## Assertions inutiles

Avez-vous déjà travaillé sur un système, et vous avez cassé un test, puis vous voyez ceci ?

> `false n'était pas égal à true`

Je sais que false n'est pas égal à true. Ce n'est pas un message utile ; il ne me dit pas ce que j'ai cassé. C'est un symptôme du non-respect du processus TDD et de la non-lecture du message d'erreur d'échec.

Revenons à la base,

> Écrivez un test, voyez-le échouer (et n'ayez pas honte du message d'erreur)

## Affirmation sur des détails non pertinents

Un exemple de ceci est de faire une assertion sur un objet complexe, alors qu'en pratique tout ce qui vous intéresse dans le test est la valeur de l'un des champs.

```go
// pas ceci, maintenant votre test est étroitement couplé à l'ensemble de l'objet
if !cmp.Equal(complexObject, want) {
	t.Error("got %+v, want %+v", complexObject, want)
}

// soyez spécifique, et relâchez le couplage
got := complexObject.fieldYouCareAboutForThisTest
if got != want {
	t.Error("got %q, want %q", got, want)
}
```

Des assertions supplémentaires non seulement rendent votre test plus difficile à lire en créant du 'bruit' dans votre documentation, mais couplent également inutilement le test avec des données dont il ne se soucie pas. Cela signifie que si vous changez les champs de votre objet, ou la façon dont ils se comportent, vous pourriez rencontrer des problèmes de compilation inattendus ou des échecs avec vos tests.

C'est un exemple de non-respect suffisamment strict de l'étape rouge.

- Laisser une conception existante influencer la façon dont vous écrivez votre test **plutôt que de penser au comportement souhaité**
- Ne pas accorder suffisamment d'attention au message d'erreur du test en échec

## De nombreuses assertions au sein d'un même scénario pour les tests unitaires

De nombreuses assertions peuvent rendre les tests difficiles à lire et à déboguer lorsqu'ils échouent.

Elles s'introduisent souvent progressivement, surtout si la configuration du test est compliquée, car vous êtes réticent à reproduire la même configuration horrible pour affirmer quelque chose d'autre. Au lieu de cela, vous devriez résoudre les problèmes dans votre conception qui rendent difficile l'affirmation sur de nouvelles choses.

Une règle pratique utile est de viser une seule assertion par test. En Go, profitez des sous-tests pour délimiter clairement entre les assertions dans les cas où vous en avez besoin. C'est aussi une technique pratique pour séparer les assertions sur le comportement des détails d'implémentation.

Pour d'autres tests où le temps de configuration ou d'exécution peut être une contrainte (par exemple un test d'acceptation pilotant un navigateur web), vous devez peser le pour et le contre entre des tests légèrement plus difficiles à déboguer et le temps d'exécution des tests.

## Ne pas écouter vos tests

[Dave Farley dans sa vidéo "When TDD goes wrong"](https://www.youtube.com/watch?v=UWtEVKVPBQ0&feature=youtu.be) souligne,

> Le TDD vous donne le feedback le plus rapide possible sur votre conception

D'après ma propre expérience, beaucoup de développeurs essaient de pratiquer le TDD mais ignorent fréquemment les signaux qui leur reviennent du processus TDD. Ils sont donc toujours coincés avec des systèmes fragiles et ennuyeux, avec une suite de tests médiocre.

En termes simples, si tester votre code est difficile, alors _utiliser_ votre code l'est aussi. Traitez vos tests comme le premier utilisateur de votre code et vous verrez si votre code est agréable à utiliser ou non.

J'ai beaucoup insisté sur ce point dans le livre, et je le répète **écoutez vos tests**.

### Configuration excessive, trop de doublures de test, etc.

Avez-vous déjà regardé un test avec 20, 50, 100, 200 lignes de code de configuration avant que quelque chose d'intéressant ne se produise dans le test ? Devez-vous ensuite modifier le code et revisiter ce désordre en souhaitant avoir une carrière différente ?

Quels sont les signaux ici ? _Écoutez_, des tests compliqués `==` code compliqué. Pourquoi votre code est-il compliqué ? Doit-il l'être ?

- Lorsque vous avez beaucoup de doublures de test dans vos tests, cela signifie que le code que vous testez a beaucoup de dépendances - ce qui signifie que votre conception a besoin de travail.
- Si votre test dépend de la configuration de diverses interactions avec des mocks, cela signifie que votre code fait beaucoup d'interactions avec ses dépendances. Demandez-vous si ces interactions pourraient être plus simples.

#### Interfaces qui fuient

Si vous avez déclaré une `interface` qui a de nombreuses méthodes, cela indique une abstraction qui fuit. Réfléchissez à la façon dont vous pourriez définir cette collaboration avec un ensemble plus consolidé de méthodes, idéalement une seule.

#### Pollution d'interface

Comme le dit un proverbe Go, *plus l'interface est grande, plus l'abstraction est faible*. Si vous exposez une interface énorme aux utilisateurs de votre package, vous les forcez à créer dans leurs tests un stub/mock qui correspond à l'ensemble de l'API, fournissant une implémentation également pour les méthodes qu'ils n'utilisent pas (parfois, ils provoquent simplement une panique pour indiquer clairement qu'elles ne devraient pas être utilisées). Cette situation est un anti-pattern connu sous le nom de [pollution d'interface](https://rakyll.org/interface-pollution/) et c'est la raison pour laquelle la bibliothèque standard ne vous offre que de toutes petites interfaces. 

Au lieu de cela, vous devriez exposer depuis votre package une structure nue avec toutes les méthodes pertinentes exportées, laissant aux clients de votre API la liberté de déclarer leurs propres interfaces en abstrayant le sous-ensemble des méthodes dont ils ont besoin : par exemple [go-redis](https://github.com/redis/go-redis) expose une structure (`redis.Client`) aux clients de l'API.

En général, vous ne devriez exposer une interface aux clients que lorsque :
- l'interface consiste en un ensemble petit et cohérent de fonctions.
- l'interface et son implémentation doivent être découplées (par exemple, parce que les utilisateurs peuvent choisir parmi plusieurs implémentations ou qu'ils ont besoin de simuler une dépendance externe).

#### Réfléchissez aux types de doublures de test que vous utilisez

- Les mocks sont parfois utiles, mais ils sont extrêmement puissants et donc faciles à mal utiliser. Essayez de vous imposer la contrainte d'utiliser des stubs à la place.
- Vérifier les détails d'implémentation avec des espions est parfois utile, mais essayez de l'éviter. Rappelez-vous que les détails d'implémentation ne sont généralement pas importants, et vous ne voulez pas que vos tests y soient couplés si possible. Cherchez à coupler vos tests à **un comportement utile plutôt qu'à des détails accessoires**.
- [Lisez mes articles sur la dénomination des doublures de test](https://quii.dev/Start_naming_your_test_doubles_correctly) si la taxonomie des doublures de test n'est pas très claire

#### Consolider les dépendances

Voici du code pour un `http.HandlerFunc` pour gérer les nouvelles inscriptions d'utilisateurs pour un site web.

```go
type User struct {
	// Quelques champs d'utilisateur
}

type UserStore interface {
	CheckEmailExists(email string) (bool, error)
	StoreUser(newUser User) error
}

type Emailer interface {
	SendEmail(to User, body string, subject string) error
}

func NewRegistrationHandler(userStore UserStore, emailer Emailer) http.HandlerFunc {
	return func(writer http.ResponseWriter, request *http.Request) {
		// extraire l'utilisateur du corps de la requête (gérer l'erreur)
		// vérifier si l'utilisateur existe (gérer les doublons, les erreurs)
		// stocker l'utilisateur (gérer les erreurs)
		// composer et envoyer un e-mail de confirmation (gérer l'erreur)
		// si nous sommes arrivés jusqu'ici, renvoyer une réponse 2xx
	}
}
```

À première vue, il est raisonnable de dire que la conception n'est pas si mauvaise. Elle n'a que 2 dépendances !

Réévaluez la conception en considérant les responsabilités du gestionnaire :

- Analyser le corps de la requête en un `User` :white_check_mark:
- Utiliser `UserStore` pour vérifier si l'utilisateur existe :question:
- Utiliser `UserStore` pour stocker l'utilisateur :question:
- Composer un e-mail :question:
- Utiliser `Emailer` pour envoyer l'e-mail :question:
- Renvoyer une réponse HTTP appropriée, selon le succès, les erreurs, etc. :white_check_mark:

Pour exercer ce code, vous allez devoir écrire de nombreux tests avec des configurations de doublures de test, des espions, etc., à des degrés divers.

- Et si les exigences s'élargissent ? Des traductions pour les e-mails ? Envoi d'une confirmation par SMS également ? Cela a-t-il un sens pour vous de devoir modifier un gestionnaire HTTP pour accommoder ce changement ?
- Est-ce que cela vous semble correct que la règle importante "nous devrions envoyer un e-mail" réside dans un gestionnaire HTTP ?
    - Pourquoi devez-vous passer par la cérémonie de création de requêtes HTTP et de lecture de réponses pour vérifier cette règle ?

**Écoutez vos tests**. Écrire des tests pour ce code à la manière TDD devrait rapidement vous mettre mal à l'aise (ou du moins, faire en sorte que le développeur paresseux en vous soit agacé). Si cela vous semble pénible, arrêtez-vous et réfléchissez.

Et si la conception était plutôt comme ceci ?

```go
type UserService interface {
	Register(newUser User) error
}

func NewRegistrationHandler(userService UserService) http.HandlerFunc {
	return func(writer http.ResponseWriter, request *http.Request) {
		// analyser l'utilisateur
		// enregistrer l'utilisateur
		// vérifier l'erreur, envoyer la réponse
	}
}
```

- Le gestionnaire est simple à tester ✅
- Les changements aux règles autour de l'enregistrement sont isolés loin de HTTP, donc ils sont aussi plus simples à tester ✅

## Violation de l'encapsulation

L'encapsulation est très importante. Il y a une raison pour laquelle nous ne rendons pas tout dans un package exporté (ou public). Nous voulons des API cohérentes avec une petite surface pour éviter un couplage étroit.

Les gens seront parfois tentés de rendre une fonction ou une méthode publique afin de tester quelque chose. En faisant cela, vous rendez votre conception pire et envoyez des messages confus aux mainteneurs et aux utilisateurs de votre code.

Un résultat de ceci peut être des développeurs essayant de déboguer un test et se rendant finalement compte que la fonction testée est _uniquement appelée à partir des tests_. Ce qui est évidemment **un résultat terrible, et une perte de temps**.

En Go, considérez votre position par défaut pour écrire des tests comme _du point de vue d'un consommateur de votre package_. Vous pouvez en faire une contrainte de compilation en ayant vos tests dans un package de test, par exemple `package gocoin_test`. Si vous faites cela, vous n'aurez accès qu'aux membres exportés du package, il ne sera donc pas possible de vous coupler aux détails d'implémentation.

## Tests de table compliqués

Les tests de table sont un excellent moyen d'exercer un certain nombre de scénarios différents lorsque la configuration du test est la même, et que vous souhaitez uniquement faire varier les entrées.

_Mais_ ils peuvent être désordonnés à lire et à comprendre lorsque vous essayez d'insérer d'autres types de tests au nom d'avoir une table glorieuse.

```go
cases := []struct {
	X                int
	Y                int
	Z                int
	err              error
	IsFullMoon       bool
	IsLeapYear       bool
	AtWarWithEurasia bool
}{}
```

**N'ayez pas peur de sortir de votre table et d'écrire de nouveaux tests** plutôt que d'ajouter de nouveaux champs et booléens à la structure de la table.

Une chose à garder à l'esprit lors de l'écriture de logiciels est,

> ["Simple" ne veut pas dire "facile"](https://www.infoq.com/presentations/Simple-Made-Easy/)

"Juste" ajouter un champ à une table peut être facile, mais cela peut rendre les choses loin d'être simples.

## Résumé

La plupart des problèmes avec les tests unitaires peuvent normalement être retracés à :

- Les développeurs ne suivent pas le processus TDD
- Une mauvaise conception

Alors, apprenez à bien concevoir des logiciels !

La bonne nouvelle est que le TDD peut vous aider à _améliorer vos compétences en conception_ car comme indiqué au début :

**L'objectif principal du TDD est de fournir un feedback sur votre conception.** Pour la millionième fois, écoutez vos tests, ils reflètent votre conception.

Soyez honnête sur la qualité de vos tests en écoutant le feedback qu'ils vous donnent, et vous deviendrez un meilleur développeur pour cela.
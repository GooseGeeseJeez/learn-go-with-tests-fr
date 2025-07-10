# Les génériques

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/generics)**

Ce chapitre vous donnera une introduction aux génériques, dissipera les réserves que vous pourriez avoir à leur sujet, et vous donnera une idée de comment simplifier votre code à l'avenir. Après avoir lu ce chapitre, vous saurez comment écrire :

- Une fonction qui prend des arguments génériques
- Une structure de données générique


## Nos propres helpers de test (`AssertEqual`, `AssertNotEqual`)

Pour explorer les génériques, nous allons écrire quelques helpers de test.

### Assertions sur les entiers

Commençons par quelque chose de basique et itérons vers notre objectif

```go
import "testing"

func TestAssertFunctions(t *testing.T) {
	t.Run("assertion sur les entiers", func(t *testing.T) {
		AssertEqual(t, 1, 1)
		AssertNotEqual(t, 1, 2)
	})
}

func AssertEqual(t *testing.T, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("obtenu %d, attendu %d", got, want)
	}
}

func AssertNotEqual(t *testing.T, got, want int) {
	t.Helper()
	if got == want {
		t.Errorf("ne voulait pas %d", got)
	}
}
```


### Assertions sur les chaînes de caractères

Pouvoir affirmer l'égalité des entiers est bien, mais que faire si nous voulons affirmer l'égalité des `string` ?

```go
t.Run("assertion sur les chaînes", func(t *testing.T) {
	AssertEqual(t, "hello", "hello")
	AssertNotEqual(t, "hello", "Grace")
})
```

Vous obtiendrez une erreur

```
# github.com/quii/learn-go-with-tests/generics [github.com/quii/learn-go-with-tests/generics.test]
./generics_test.go:12:18: cannot use "hello" (untyped string constant) as int value in argument to AssertEqual
./generics_test.go:13:21: cannot use "hello" (untyped string constant) as int value in argument to AssertNotEqual
./generics_test.go:13:30: cannot use "Grace" (untyped string constant) as int value in argument to AssertNotEqual
```

Si vous prenez le temps de lire l'erreur, vous verrez que le compilateur se plaint que nous essayons de passer une `string` à une fonction qui attend un `integer`.

#### Rappel sur la sécurité des types

Si vous avez lu les chapitres précédents de ce livre, ou si vous avez de l'expérience avec les langages à typage statique, cela ne devrait pas vous surprendre. Le compilateur Go s'attend à ce que vous écriviez vos fonctions, structs, etc. en décrivant les types avec lesquels vous souhaitez travailler.

Vous ne pouvez pas passer une `string` à une fonction qui attend un `integer`.

Bien que cela puisse sembler cérémonieux, c'est extrêmement utile. En décrivant ces contraintes, vous :

- Simplifiez l'implémentation des fonctions. En décrivant au compilateur les types avec lesquels vous travaillez, vous **limitez le nombre d'implémentations valides possibles**. Vous ne pouvez pas "ajouter" une `Person` et un `BankAccount`. Vous ne pouvez pas mettre en majuscules un `integer`. En logiciel, les contraintes sont souvent extrêmement utiles.
- Évitez de passer accidentellement des données à une fonction que vous n'aviez pas l'intention d'utiliser.

Go vous offre un moyen d'être plus abstrait avec vos types grâce aux [interfaces](./structs-methods-and-interfaces.md), afin que vous puissiez concevoir des fonctions qui ne prennent pas des types concrets mais plutôt des types qui offrent le comportement dont vous avez besoin. Cela vous donne une certaine flexibilité tout en maintenant la sécurité des types.

### Une fonction qui prend une chaîne ou un entier ? (ou en effet, d'autres choses)

Une autre option que Go offre pour rendre vos fonctions plus flexibles est de déclarer le type de votre argument comme `interface{}`, ce qui signifie "n'importe quoi".

Essayez de changer les signatures pour utiliser ce type à la place.

```go
func AssertEqual(got, want interface{})

func AssertNotEqual(got, want interface{})

```

Les tests devraient maintenant compiler et passer. Si vous essayez de les faire échouer, vous verrez que la sortie est un peu bancale car nous utilisons le format d'entier `%d` pour imprimer nos messages, alors changez-les pour le format général `%+v` pour une meilleure sortie de n'importe quel type de valeur.

### Le problème avec `interface{}`

Nos fonctions `AssertX` sont assez naïves mais conceptuellement, elles ne sont pas très différentes de la façon dont d'autres [bibliothèques populaires offrent cette fonctionnalité](https://github.com/matryer/is/blob/master/is.go#L150)

```go
func (is *I) Equal(a, b interface{})
```

Alors quel est le problème ?

En utilisant `interface{}`, le compilateur ne peut pas nous aider lors de l'écriture de notre code, car nous ne lui disons rien d'utile sur les types de choses passées à la fonction. Essayez de comparer deux types différents.

```go
AssertEqual(1, "1")
```

Dans ce cas, nous nous en sortons; le test compile et échoue comme nous l'espérions, bien que le message d'erreur `got 1, want 1` ne soit pas clair; mais voulons-nous vraiment pouvoir comparer des chaînes avec des entiers ? Qu'en est-il de comparer une `Person` avec un `Airport` ?

Écrire des fonctions qui prennent `interface{}` peut être extrêmement difficile et sujet aux bogues car nous avons _perdu_ nos contraintes, et nous n'avons aucune information à la compilation sur les types de données avec lesquels nous travaillons.

Cela signifie que **le compilateur ne peut pas nous aider** et que nous sommes plutôt plus susceptibles d'avoir des **erreurs d'exécution** qui pourraient affecter nos utilisateurs, provoquer des pannes, ou pire.

Souvent, les développeurs doivent utiliser la réflexion pour implémenter ces fonctions *ahem* génériques, ce qui peut devenir compliqué à lire et à écrire, et peut nuire aux performances de votre programme.

## Nos propres helpers de test avec les génériques

Idéalement, nous ne voulons pas avoir à créer des fonctions `AssertX` spécifiques pour chaque type avec lequel nous travaillons. Nous aimerions pouvoir avoir _une_ fonction `AssertEqual` qui fonctionne avec _n'importe quel_ type mais qui ne vous permet pas de comparer [des pommes et des oranges](https://fr.wikipedia.org/wiki/Comparer_des_pommes_et_des_oranges).

Les génériques nous offrent un moyen de créer des abstractions (comme les interfaces) en nous permettant de **décrire nos contraintes**. Ils nous permettent d'écrire des fonctions qui ont un niveau de flexibilité similaire à celui offert par `interface{}` mais qui conservent la sécurité des types et offrent une meilleure expérience de développement pour les appelants.

```go
func TestAssertFunctions(t *testing.T) {
	t.Run("assertion sur les entiers", func(t *testing.T) {
		AssertEqual(t, 1, 1)
		AssertNotEqual(t, 1, 2)
	})

	t.Run("assertion sur les chaînes", func(t *testing.T) {
		AssertEqual(t, "hello", "hello")
		AssertNotEqual(t, "hello", "Grace")
	})

	// AssertEqual(t, 1, "1") // décommentez pour voir l'erreur
}

func AssertEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()
	if got != want {
		t.Errorf("obtenu %v, attendu %v", got, want)
	}
}

func AssertNotEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()
	if got == want {
		t.Errorf("ne voulait pas %v", got)
	}
}
```

Pour écrire des fonctions génériques en Go, vous devez fournir des "paramètres de type", ce qui est juste une façon sophistiquée de dire "décrire votre type générique et lui donner un nom".

Dans notre cas, le type de notre paramètre de type est `comparable` et nous lui avons donné le nom `T`. Ce nom nous permet ensuite de décrire les types des arguments de notre fonction (`got, want T`).

Nous utilisons `comparable` parce que nous voulons décrire au compilateur que nous souhaitons utiliser les opérateurs `==` et `!=` sur des choses de type `T` dans notre fonction, nous voulons comparer ! Si vous essayez de changer le type en `any`,

```go
func AssertNotEqual[T any](got, want T)
```

Vous obtiendrez l'erreur suivante :

```
prog.go2:15:5: cannot compare got != want (operator != not defined for T)
```

Ce qui a beaucoup de sens, car vous ne pouvez pas utiliser ces opérateurs sur tous les types (ou sur `any`).

### Une fonction générique avec [`T any`](https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md#the-constraint) est-elle la même chose que `interface{}` ?

Considérez deux fonctions

```go
func GenericFoo[T any](x, y T)
```

```go
func InterfaceyFoo(x, y interface{})
```

Quel est l'intérêt des génériques ici ? `any` ne décrit-il pas... n'importe quoi ?

En termes de contraintes, `any` signifie bien "n'importe quoi", tout comme `interface{}`. En fait, `any` a été ajouté en 1.18 et n'est _qu'un alias pour `interface{}`_.

La différence avec la version générique est que _vous décrivez toujours un type spécifique_ et cela signifie que nous avons toujours limité cette fonction à ne fonctionner qu'avec _un seul_ type.

Cela signifie que vous pouvez appeler `InterfaceyFoo` avec n'importe quelle combinaison de types (par exemple `InterfaceyFoo(pomme, orange)`). Cependant, `GenericFoo` offre toujours certaines contraintes car nous avons dit qu'il ne fonctionne qu'avec _un seul_ type, `T`.

Valide :

- `GenericFoo(pomme1, pomme2)`
- `GenericFoo(orange1, orange2)`
- `GenericFoo(1, 2)`
- `GenericFoo("un", "deux")`

Non valide (échec à la compilation) :

- `GenericFoo(pomme1, orange1)`
- `GenericFoo("1", 1)`

Si votre fonction renvoie le type générique, l'appelant peut également utiliser le type tel quel, plutôt que d'avoir à faire une assertion de type, car lorsqu'une fonction renvoie `interface{}`, le compilateur ne peut donner aucune garantie sur le type.

## Ensuite : Types de données génériques

Nous allons créer un type de données [pile](https://fr.wikipedia.org/wiki/Pile_(informatique)) (stack). Les piles devraient être assez simples à comprendre du point de vue des exigences. Ce sont des collections d'éléments où vous pouvez `Push` (empiler) des éléments au "sommet" et pour récupérer des éléments, vous `Pop` (dépiler) des éléments du sommet (LIFO - dernier entré, premier sorti).

Par souci de brièveté, j'ai omis le processus TDD qui m'a amené au code suivant pour une pile d'entiers (`int`) et une pile de chaînes de caractères (`string`).

```go
type StackOfInts struct {
	values []int
}

func (s *StackOfInts) Push(value int) {
	s.values = append(s.values, value)
}

func (s *StackOfInts) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *StackOfInts) Pop() (int, bool) {
	if s.IsEmpty() {
		return 0, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}

type StackOfStrings struct {
	values []string
}

func (s *StackOfStrings) Push(value string) {
	s.values = append(s.values, value)
}

func (s *StackOfStrings) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *StackOfStrings) Pop() (string, bool) {
	if s.IsEmpty() {
		return "", false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

J'ai créé quelques autres fonctions d'assertion pour aider

```go
func AssertTrue(t *testing.T, got bool) {
	t.Helper()
	if !got {
		t.Errorf("obtenu %v, attendu true", got)
	}
}

func AssertFalse(t *testing.T, got bool) {
	t.Helper()
	if got {
		t.Errorf("obtenu %v, attendu false", got)
	}
}
```

Et voici les tests

```go
func TestStack(t *testing.T) {
	t.Run("pile d'entiers", func(t *testing.T) {
		myStackOfInts := new(StackOfInts)

		// vérifie que la pile est vide
		AssertTrue(t, myStackOfInts.IsEmpty())

		// ajoute un élément, puis vérifie qu'elle n'est pas vide
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// ajoute un autre élément, le récupère à nouveau
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())
	})

	t.Run("pile de chaînes", func(t *testing.T) {
		myStackOfStrings := new(StackOfStrings)

		// vérifie que la pile est vide
		AssertTrue(t, myStackOfStrings.IsEmpty())

		// ajoute un élément, puis vérifie qu'elle n'est pas vide
		myStackOfStrings.Push("123")
		AssertFalse(t, myStackOfStrings.IsEmpty())

		// ajoute un autre élément, le récupère à nouveau
		myStackOfStrings.Push("456")
		value, _ := myStackOfStrings.Pop()
		AssertEqual(t, value, "456")
		value, _ = myStackOfStrings.Pop()
		AssertEqual(t, value, "123")
		AssertTrue(t, myStackOfStrings.IsEmpty())
	})
}
```

### Problèmes

- Le code pour `StackOfStrings` et `StackOfInts` est presque identique. Bien que la duplication ne soit pas toujours la fin du monde, c'est plus de code à lire, écrire et maintenir.
- Comme nous dupliquons la logique entre deux types, nous avons également dû dupliquer les tests.

Nous voulons vraiment capturer l'_idée_ d'une pile dans un seul type, et avoir un seul ensemble de tests pour elles. Nous devrions porter notre chapeau de refactoring maintenant, ce qui signifie que nous ne devrions pas changer les tests car nous voulons maintenir le même comportement.

Sans génériques, voici ce que nous _pourrions_ faire

```go
type StackOfInts = Stack
type StackOfStrings = Stack

type Stack struct {
	values []interface{}
}

func (s *Stack) Push(value interface{}) {
	s.values = append(s.values, value)
}

func (s *Stack) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack) Pop() (interface{}, bool) {
	if s.IsEmpty() {
		var zero interface{}
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

- Nous donnons un alias à nos implémentations précédentes de `StackOfInts` et `StackOfStrings` vers un nouveau type unifié `Stack`
- Nous avons supprimé la sécurité des types de la `Stack` en faisant en sorte que `values` soit une [slice](https://github.com/quii/learn-go-with-tests/blob/main/arrays-and-slices.md) de `interface{}`

Pour essayer ce code, vous devrez supprimer les contraintes de type de nos fonctions d'assertion :

```go
func AssertEqual(t *testing.T, got, want interface{})
```

Si vous faites cela, nos tests passent toujours. Qui a besoin de génériques ?

### Le problème avec l'abandon de la sécurité des types

Le premier problème est le même que celui que nous avons vu avec notre `AssertEquals` - nous avons perdu la sécurité des types. Je peux maintenant `Push` des pommes sur une pile d'oranges.

Même si nous avons la discipline de ne pas le faire, le code est toujours désagréable à utiliser car lorsque des méthodes **renvoient `interface{}`, elles sont horribles à utiliser**.

Ajoutez le test suivant,

```go
t.Run("l'expérience développeur avec une pile d'interface est horrible", func(t *testing.T) {
	myStackOfInts := new(StackOfInts)

	myStackOfInts.Push(1)
	myStackOfInts.Push(2)
	firstNum, _ := myStackOfInts.Pop()
	secondNum, _ := myStackOfInts.Pop()
	AssertEqual(t, firstNum+secondNum, 3)
})
```

Vous obtenez une erreur de compilateur, montrant la faiblesse de la perte de la sécurité des types :

```
invalid operation: operator + not defined on firstNum (variable of type interface{})
```

Lorsque `Pop` renvoie `interface{}`, cela signifie que le compilateur n'a aucune information sur ce qu'est la donnée et limite donc sévèrement ce que nous pouvons faire. Il ne peut pas savoir qu'il devrait s'agir d'un entier, il ne nous permet donc pas d'utiliser l'opérateur `+`.

Pour contourner cela, l'appelant doit faire une [assertion de type](https://golang.org/ref/spec#Type_assertions) pour chaque valeur.

```go
t.Run("l'expérience développeur avec une pile d'interface est horrible", func(t *testing.T) {
	myStackOfInts := new(StackOfInts)

	myStackOfInts.Push(1)
	myStackOfInts.Push(2)
	firstNum, _ := myStackOfInts.Pop()
	secondNum, _ := myStackOfInts.Pop()

	// récupérer nos entiers depuis interface{}
	reallyFirstNum, ok := firstNum.(int)
	AssertTrue(t, ok) // besoin de vérifier que nous avons bien obtenu un int depuis interface{}

	reallySecondNum, ok := secondNum.(int)
	AssertTrue(t, ok) // et encore une fois !

	AssertEqual(t, reallyFirstNum+reallySecondNum, 3)
})
```

Le désagrément qui émane de ce test se répéterait pour chaque utilisateur potentiel de notre implémentation de `Stack`, beurk.

### Les structures de données génériques à la rescousse

Tout comme vous pouvez définir des arguments génériques pour les fonctions, vous pouvez définir des structures de données génériques.

Voici notre nouvelle implémentation de `Stack`, avec un type de données générique.

```go
type Stack[T any] struct {
	values []T
}

func (s *Stack[T]) Push(value T) {
	s.values = append(s.values, value)
}

func (s *Stack[T]) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack[T]) Pop() (T, bool) {
	if s.IsEmpty() {
		var zero T
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

Voici les tests, montrant qu'ils fonctionnent comme nous le souhaitons, avec une sécurité des types complète.

```go
func TestStack(t *testing.T) {
	t.Run("pile d'entiers", func(t *testing.T) {
		myStackOfInts := new(Stack[int])

		// vérifie que la pile est vide
		AssertTrue(t, myStackOfInts.IsEmpty())

		// ajoute un élément, puis vérifie qu'elle n'est pas vide
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// ajoute un autre élément, le récupère à nouveau
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())

		// peut obtenir les nombres que nous avons mis sous forme de nombres, pas d'interface{} non typée
		myStackOfInts.Push(1)
		myStackOfInts.Push(2)
		firstNum, _ := myStackOfInts.Pop()
		secondNum, _ := myStackOfInts.Pop()
		AssertEqual(t, firstNum+secondNum, 3)
	})
}
```

Vous remarquerez que la syntaxe pour définir des structures de données génériques est cohérente avec la définition d'arguments génériques pour les fonctions.

```go
type Stack[T any] struct {
	values []T
}
```

C'est _presque_ identique à avant, c'est juste que ce que nous disons, c'est que le **type de la pile limite le type de valeurs avec lesquelles vous pouvez travailler**.

Une fois que vous créez une `Stack[Orange]` ou une `Stack[Apple]`, les méthodes définies sur notre pile ne vous permettront de passer et ne renverront que le type particulier de pile avec lequel vous travaillez :

```go
func (s *Stack[T]) Pop() (T, bool)
```

Vous pouvez imaginer que les types d'implémentation sont en quelque sorte générés pour vous, selon le type de pile que vous créez :

```go
func (s *Stack[Orange]) Pop() (Orange, bool)
```

```go
func (s *Stack[Apple]) Pop() (Apple, bool)
```

Maintenant que nous avons fait ce refactoring, nous pouvons supprimer en toute sécurité le test de pile de chaînes car nous n'avons pas besoin de prouver la même logique encore et encore.

Notez que jusqu'à présent dans les exemples d'appel de fonctions génériques, nous n'avons pas eu besoin de spécifier les types génériques. Par exemple, pour appeler `AssertEqual[T]`, nous n'avons pas besoin de spécifier quel est le type `T` puisqu'il peut être déduit des arguments. Dans les cas où les types génériques ne peuvent pas être déduits, vous devez spécifier les types lors de l'appel de la fonction. La syntaxe est la même que lors de la définition de la fonction, c'est-à-dire que vous spécifiez les types entre crochets avant les arguments.

Pour un exemple concret, considérez la création d'un constructeur pour `Stack[T]`.
```go
func NewStack[T any]() *Stack[T] {
	return new(Stack[T])
}
```
Pour utiliser ce constructeur afin de créer une pile d'entiers et une pile de chaînes par exemple, vous l'appelez comme ceci :
```go
myStackOfInts := NewStack[int]()
myStackOfStrings := NewStack[string]()
```

Voici l'implémentation de `Stack` et les tests après avoir ajouté le constructeur.

```go
type Stack[T any] struct {
	values []T
}

func NewStack[T any]() *Stack[T] {
	return new(Stack[T])
}

func (s *Stack[T]) Push(value T) {
	s.values = append(s.values, value)
}

func (s *Stack[T]) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack[T]) Pop() (T, bool) {
	if s.IsEmpty() {
		var zero T
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

```go
func TestStack(t *testing.T) {
	t.Run("pile d'entiers", func(t *testing.T) {
		myStackOfInts := NewStack[int]()

		// vérifie que la pile est vide
		AssertTrue(t, myStackOfInts.IsEmpty())

		// ajoute un élément, puis vérifie qu'elle n'est pas vide
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// ajoute un autre élément, le récupère à nouveau
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())

		// peut obtenir les nombres que nous avons mis sous forme de nombres, pas d'interface{} non typée
		myStackOfInts.Push(1)
		myStackOfInts.Push(2)
		firstNum, _ := myStackOfInts.Pop()
		secondNum, _ := myStackOfInts.Pop()
		AssertEqual(t, firstNum+secondNum, 3)
	})
}
```


En utilisant un type de données générique, nous avons :

- Réduit la duplication de la logique importante.
- Fait en sorte que `Pop` renvoie `T` afin que si nous créons une `Stack[int]`, nous obtenions en pratique un `int` de `Pop` ; nous pouvons maintenant utiliser `+` sans avoir besoin de gymnastique d'assertion de type.
- Empêché les mauvaises utilisations au moment de la compilation. Vous ne pouvez pas `Push` des oranges sur une pile de pommes.

## Conclusion

Ce chapitre devrait vous avoir donné un aperçu de la syntaxe des génériques et quelques idées sur pourquoi les génériques pourraient être utiles. Nous avons écrit nos propres fonctions `Assert` que nous pouvons réutiliser en toute sécurité pour expérimenter d'autres idées autour des génériques, et nous avons implémenté une structure de données simple pour stocker n'importe quel type de données que nous souhaitons, de manière sécurisée.

### Les génériques sont plus simples que l'utilisation de `interface{}` dans la plupart des cas

Si vous n'avez pas d'expérience avec les langages à typage statique, l'intérêt des génériques peut ne pas être immédiatement évident, mais j'espère que les exemples de ce chapitre ont illustré où le langage Go n'est pas aussi expressif que nous le souhaiterions. En particulier, l'utilisation de `interface{}` rend votre code :

- Moins sûr (mélanger des pommes et des oranges), nécessite plus de gestion d'erreurs
- Moins expressif, `interface{}` ne vous dit rien sur les données
- Plus susceptible de s'appuyer sur la [réflexion](https://github.com/quii/learn-go-with-tests/blob/main/reflection.md), les assertions de type, etc., ce qui rend votre code plus difficile à utiliser et plus sujet aux erreurs car il repousse les vérifications du moment de la compilation au moment de l'exécution

L'utilisation de langages à typage statique est un acte de description des contraintes. Si vous le faites bien, vous créez un code qui est non seulement sûr et simple à utiliser, mais aussi plus simple à écrire car l'espace de solution possible est plus petit.

Les génériques nous donnent une nouvelle façon d'exprimer des contraintes dans notre code, ce qui, comme démontré, nous permettra de consolider et de simplifier du code qui n'était pas possible avant Go 1.18.

### Les génériques transformeront-ils Go en Java ?

- Non.

Il y a beaucoup de [FUD (*Fear, Uncertainty and Doubt*; peur, incertitude et doute)](https://fr.wikipedia.org/wiki/Fear,_uncertainty_and_doubt) dans la communauté Go concernant les génériques menant à des abstractions cauchemardesques et des bases de code déroutantes. Cela est généralement nuancé par "ils doivent être utilisés avec prudence".

Bien que ce soit vrai, ce n'est pas un conseil particulièrement utile car c'est vrai pour toute fonctionnalité de langage.

Peu de gens se plaignent de notre capacité à définir des interfaces qui, comme les génériques, est un moyen de décrire des contraintes dans notre code. Lorsque vous décrivez une interface, vous faites un choix de conception qui _pourrait être médiocre_, les génériques ne sont pas uniques dans leur capacité à créer du code déroutant et ennuyeux à utiliser.

### Vous utilisez déjà des génériques

Quand vous considérez que si vous avez utilisé des tableaux, des slices ou des maps, vous avez _déjà été un consommateur de code générique_.

```
var myApples []Apple
// Vous ne pouvez pas faire cela !
append(myApples, Orange{})
```

### L'abstraction n'est pas un gros mot

Il est facile de se moquer de [AbstractSingletonProxyFactoryBean](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/aop/framework/AbstractSingletonProxyFactoryBean.html) mais ne prétendons pas qu'une base de code sans aucune abstraction n'est pas également mauvaise. C'est votre travail de _rassembler_ les concepts connexes lorsque c'est approprié, afin que votre système soit plus facile à comprendre et à modifier, plutôt que d'être une collection de fonctions et de types disparates manquant de clarté.

### [Faites-le fonctionner, faites-le bien, faites-le rapide](https://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast#:~:text=%22Make%20it%20work%2C%20make%20it,to%20DesignForPerformance%20ahead%20of%20time.)

Les gens rencontrent des problèmes avec les génériques lorsqu'ils font de l'abstraction trop rapidement sans avoir suffisamment d'informations pour prendre de bonnes décisions de conception.

Le cycle TDD de rouge, vert, refactoriser signifie que vous avez plus d'indications sur le code dont vous avez _réellement besoin_ pour fournir votre comportement, **plutôt que d'imaginer des abstractions à l'avance** ; mais vous devez toujours être prudent.

Il n'y a pas de règles strictes ici, mais résistez à la tentation de rendre les choses génériques jusqu'à ce que vous puissiez voir que vous avez une généralisation utile. Lorsque nous avons créé les diverses implémentations de `Stack`, nous avons commencé par un comportement _concret_ comme `StackOfStrings` et `StackOfInts` soutenu par des tests. À partir de notre code _réel_, nous avons pu commencer à voir de vrais modèles, et soutenus par nos tests, nous avons pu explorer le refactoring vers une solution plus générale.

On vous conseille souvent de ne généraliser que lorsque vous voyez le même code trois fois, ce qui semble être une bonne règle de base pour commencer.

Un chemin commun que j'ai pris dans d'autres langages de programmation a été :

- Un cycle TDD pour conduire un certain comportement
- Un autre cycle TDD pour exercer d'autres scénarios connexes

> Hmm, ces choses semblent similaires - mais un peu de duplication est préférable à un couplage avec une mauvaise abstraction

- Dormir dessus
- Un autre cycle TDD

> OK, j'aimerais essayer de voir si je peux généraliser cette chose. Dieu merci, je suis si intelligent et beau parce que j'utilise TDD, donc je peux refactoriser quand je le souhaite, et le processus m'a aidé à comprendre quel comportement j'ai réellement besoin avant de trop concevoir.

- Cette abstraction semble agréable ! Les tests passent toujours, et le code est plus simple
- Je peux maintenant supprimer un certain nombre de tests, j'ai capturé l'_essence_ du comportement et supprimé les détails inutiles
# Structs, méthodes et interfaces

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/structs)**

Supposons que nous ayons besoin de code géométrique pour calculer le périmètre d'un rectangle donné une hauteur et une largeur. Nous pouvons écrire une fonction `Perimetre(largeur float64, hauteur float64)`, où `float64` est pour les nombres à virgule flottante comme `123.45`.

Le cycle TDD devrait maintenant vous être assez familier.

## Écrivez le test d'abord

```go
func TestPerimetre(t *testing.T) {
	resultat := Perimetre(10.0, 10.0)
	attendu := 40.0

	if resultat != attendu {
		t.Errorf("reçu %.2f attendu %.2f", resultat, attendu)
	}
}
```

Vous remarquez la nouvelle chaîne de format ? Le `f` est pour notre `float64` et le `.2` signifie imprimer 2 décimales.

## Essayez d'exécuter le test

`./formes_test.go:6:14: undefined: Perimetre`

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

```go
func Perimetre(largeur float64, hauteur float64) float64 {
	return 0
}
```

Résulte en `formes_test.go:10: reçu 0.00 attendu 40.00`.

## Écrivez assez de code pour le faire passer

```go
func Perimetre(largeur float64, hauteur float64) float64 {
	return 2 * (largeur + hauteur)
}
```

Jusqu'ici, très facile. Maintenant créons une fonction appelée `Aire(largeur, hauteur float64)` qui retourne l'aire d'un rectangle.

Essayez de le faire vous-même, en suivant le cycle TDD.

Vous devriez vous retrouver avec des tests comme ceci

```go
func TestPerimetre(t *testing.T) {
	resultat := Perimetre(10.0, 10.0)
	attendu := 40.0

	if resultat != attendu {
		t.Errorf("reçu %.2f attendu %.2f", resultat, attendu)
	}
}

func TestAire(t *testing.T) {
	resultat := Aire(12.0, 6.0)
	attendu := 72.0

	if resultat != attendu {
		t.Errorf("reçu %.2f attendu %.2f", resultat, attendu)
	}
}
```

Et du code comme ceci

```go
func Perimetre(largeur float64, hauteur float64) float64 {
	return 2 * (largeur + hauteur)
}

func Aire(largeur float64, hauteur float64) float64 {
	return largeur * hauteur
}
```

## Refactoriser

Notre code fait le travail, mais il ne contient rien d'explicite sur les rectangles. Un développeur imprudent pourrait essayer de fournir la largeur et la hauteur d'un triangle à ces fonctions sans réaliser qu'elles retourneront la mauvaise réponse.

Nous pourrions juste donner aux fonctions des noms plus spécifiques comme `AireRectangle`. Une solution plus élégante est de définir notre propre _type_ appelé `Rectangle` qui encapsule ce concept pour nous.

Nous pouvons créer un type simple en utilisant une **struct**. [Une struct](https://golang.org/ref/spec#Struct_types) est juste une collection nommée de champs où vous pouvez stocker des données.

Déclarez une struct dans votre fichier `formes.go` comme ceci

```go
type Rectangle struct {
	Largeur float64
	Hauteur float64
}
```

Maintenant refactorisons les tests pour utiliser `Rectangle` au lieu de simples `float64`.

```go
func TestPerimetre(t *testing.T) {
	rectangle := Rectangle{10.0, 10.0}
	resultat := Perimetre(rectangle)
	attendu := 40.0

	if resultat != attendu {
		t.Errorf("reçu %.2f attendu %.2f", resultat, attendu)
	}
}

func TestAire(t *testing.T) {
	rectangle := Rectangle{12.0, 6.0}
	resultat := Aire(rectangle)
	attendu := 72.0

	if resultat != attendu {
		t.Errorf("reçu %.2f attendu %.2f", resultat, attendu)
	}
}
```

Rappelez-vous d'exécuter vos tests avant d'essayer de les corriger. Les tests devraient montrer une erreur utile comme

```text
./formes_test.go:7:23: not enough arguments in call to Perimetre
    have (Rectangle)
    want (float64, float64)
```

Vous pouvez accéder aux champs d'une struct avec la syntaxe `maStruct.champ`.

Changez les deux fonctions pour corriger le test.

```go
func Perimetre(rectangle Rectangle) float64 {
	return 2 * (rectangle.Largeur + rectangle.Hauteur)
}

func Aire(rectangle Rectangle) float64 {
	return rectangle.Largeur * rectangle.Hauteur
}
```

J'espère que vous serez d'accord que passer un `Rectangle` à une fonction transmet notre intention plus clairement, mais il y a plus d'avantages à utiliser des structs que nous couvrirons plus tard.

Notre prochaine exigence est d'écrire une fonction `Aire` pour les cercles.

## Écrivez le test d'abord

```go
func TestAire(t *testing.T) {

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		resultat := Aire(rectangle)
		attendu := 72.0

		if resultat != attendu {
			t.Errorf("reçu %g attendu %g", resultat, attendu)
		}
	})

	t.Run("cercles", func(t *testing.T) {
		cercle := Cercle{10}
		resultat := Aire(cercle)
		attendu := 314.1592653589793

		if resultat != attendu {
			t.Errorf("reçu %g attendu %g", resultat, attendu)
		}
	})

}
```

Comme vous pouvez le voir, le `f` a été remplacé par `g`, pour une bonne raison.
L'utilisation de `g` imprimera un nombre décimal plus précis dans le message d'erreur \([options fmt](https://golang.org/pkg/fmt/)\).
Par exemple, en utilisant un rayon de 1.5 dans un calcul d'aire de cercle, `f` montrerait `7.068583` tandis que `g` montrerait `7.0685834705770345`.

## Essayez d'exécuter le test

`./formes_test.go:28:11: undefined: Cercle`

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Nous devons définir notre type `Cercle`.

```go
type Cercle struct {
	Rayon float64
}
```

Maintenant essayez d'exécuter les tests à nouveau

`./formes_test.go:29:13: cannot use cercle (type Cercle) as type Rectangle in argument to Aire`

Certains langages de programmation vous permettent de faire quelque chose comme ceci :

```go
func Aire(cercle Cercle) float64       {}
func Aire(rectangle Rectangle) float64 {}
```

Mais vous ne pouvez pas en Go

`./formes.go:20:32: Aire redeclared in this block`

Nous avons deux choix :

* Vous pouvez avoir des fonctions avec le même nom déclarées dans différents _packages_. Donc nous pourrions créer notre `Aire(Cercle)` dans un nouveau package, mais cela semble excessif ici.
* Nous pouvons définir des [_méthodes_](https://golang.org/ref/spec#Method_declarations) sur nos types nouvellement définis à la place.

### Que sont les méthodes ?

Jusqu'à présent, nous n'avons écrit que des _fonctions_ mais nous avons utilisé quelques méthodes. Quand nous appelons `t.Errorf`, nous appelons la méthode `Errorf` sur l'instance de notre `t` \(`testing.T`\).

Une méthode est une fonction avec un récepteur.
Une déclaration de méthode lie un identifiant, le nom de la méthode, à une méthode, et associe la méthode avec le type de base du récepteur.

Les méthodes sont très similaires aux fonctions mais elles sont appelées en les invoquant sur une instance d'un type particulier. Alors que vous pouvez juste appeler les fonctions où vous voulez, comme `Aire(rectangle)`, vous ne pouvez appeler les méthodes que sur des "choses".

Un exemple aidera alors changeons nos tests d'abord pour appeler des méthodes à la place et puis corrigeons le code.

```go
func TestAire(t *testing.T) {

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		resultat := rectangle.Aire()
		attendu := 72.0

		if resultat != attendu {
			t.Errorf("reçu %g attendu %g", resultat, attendu)
		}
	})

	t.Run("cercles", func(t *testing.T) {
		cercle := Cercle{10}
		resultat := cercle.Aire()
		attendu := 314.1592653589793

		if resultat != attendu {
			t.Errorf("reçu %g attendu %g", resultat, attendu)
		}
	})

}
```

Si nous essayons d'exécuter les tests, nous obtenons

```text
./formes_test.go:19:20: rectangle.Aire undefined (type Rectangle has no field or method Aire)
./formes_test.go:29:16: cercle.Aire undefined (type Cercle has no field or method Aire)
```

> type Cercle has no field or method Aire

J'aimerais réitérer à quel point le compilateur est génial ici. Il est si important de prendre le temps de lire lentement les messages d'erreur que vous obtenez, cela vous aidera à long terme.

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Ajoutons quelques méthodes à nos types

```go
type Rectangle struct {
	Largeur float64
	Hauteur float64
}

func (r Rectangle) Aire() float64 {
	return 0
}

type Cercle struct {
	Rayon float64
}

func (c Cercle) Aire() float64 {
	return 0
}
```

La syntaxe pour déclarer des méthodes est presque la même que les fonctions et c'est parce qu'elles sont si similaires. La seule différence est la syntaxe du récepteur de méthode `func (nomRecepteur TypeRecepteur) NomMethode(args)`.

Quand votre méthode est appelée sur une variable de ce type, vous obtenez votre référence à ses données via la variable `nomRecepteur`. Dans beaucoup d'autres langages de programmation, ceci est fait implicitement et vous accédez au récepteur via `this`.

C'est une convention en Go d'avoir la variable récepteur être la première lettre du type.

```
r Rectangle
```

Si vous essayez de relancer les tests, ils devraient maintenant compiler et vous donner une sortie d'échec.

## Écrivez assez de code pour le faire passer

Maintenant corrigeons nos tests de rectangle en corrigeant notre nouvelle méthode

```go
func (r Rectangle) Aire() float64 {
	return r.Largeur * r.Hauteur
}
```

Si vous relancez les tests, les tests de rectangle devraient passer mais le cercle devrait toujours échouer.

Pour faire passer la fonction `Aire` du cercle, nous empruntons la constante `Pi` du package `math` \(rappelez-vous de l'importer\).

```go
func (c Cercle) Aire() float64 {
	return math.Pi * c.Rayon * c.Rayon
}
```

## Refactoriser

Il y a quelques duplications dans nos tests.

Tout ce que nous voulons faire est prendre une collection de _formes_, appeler la méthode `Aire()` sur elles et puis vérifier le résultat.

Nous voulons pouvoir écrire une sorte de fonction `verifierAire` à laquelle nous pouvons passer à la fois des `Rectangle` et des `Cercle`, mais échouer à compiler si nous essayons de passer quelque chose qui n'est pas une forme.

Avec Go, nous pouvons codifier cette intention avec les **interfaces**.

Les [Interfaces](https://golang.org/ref/spec#Interface_types) sont un concept très puissant dans les langages statiquement typés comme Go parce qu'elles vous permettent de créer des fonctions qui peuvent être utilisées avec différents types et créent du code hautement découplé tout en maintenant la sécurité de type.

Introduisons ceci en refactorisant nos tests.

```go
func TestAire(t *testing.T) {

	verifierAire := func(t testing.TB, forme Forme, attendu float64) {
		t.Helper()
		resultat := forme.Aire()
		if resultat != attendu {
			t.Errorf("reçu %g attendu %g", resultat, attendu)
		}
	}

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		verifierAire(t, rectangle, 72.0)
	})

	t.Run("cercles", func(t *testing.T) {
		cercle := Cercle{10}
		verifierAire(t, cercle, 314.1592653589793)
	})

}
```

Nous créons une fonction d'aide comme nous l'avons fait dans d'autres exercices mais cette fois nous demandons qu'une `Forme` soit passée. Si nous essayons d'appeler ceci avec quelque chose qui n'est pas une forme, alors ça ne compilera pas.

Comment quelque chose devient-il une forme ? Nous disons juste à Go ce qu'est une `Forme` en utilisant une déclaration d'interface

```go
type Forme interface {
	Aire() float64
}
```

Nous créons un nouveau `type` tout comme nous l'avons fait avec `Rectangle` et `Cercle` mais cette fois c'est une `interface` plutôt qu'une `struct`.

Une fois que vous ajoutez ceci au code, les tests passeront.

### Attendez, quoi ?

C'est assez différent des interfaces dans la plupart des autres langages de programmation. Normalement vous devez écrire du code pour dire `Mon type Foo implémente l'interface Bar`.

Mais dans notre cas

* `Rectangle` a une méthode appelée `Aire` qui retourne un `float64` donc elle satisfait l'interface `Forme`
* `Cercle` a une méthode appelée `Aire` qui retourne un `float64` donc elle satisfait l'interface `Forme`
* `string` n'a pas une telle méthode, donc elle ne satisfait pas l'interface
* etc.

En Go **la résolution d'interface est implicite**. Si le type que vous passez correspond à ce que l'interface demande, ça compilera.

### Découplage

Remarquez comment notre aide n'a pas besoin de se préoccuper de savoir si la forme est un `Rectangle` ou un `Cercle` ou un `Triangle`. En déclarant une interface, l'aide est _découplée_ des types concrets et a seulement la méthode dont elle a besoin pour faire son travail.

Ce genre d'approche d'utiliser des interfaces pour déclarer **seulement ce dont vous avez besoin** est très important dans la conception de logiciels et sera couvert plus en détail dans les sections ultérieures.

## Refactorisation supplémentaire

Maintenant que vous avez une certaine compréhension des structs, nous pouvons introduire les "tests dirigés par table".

Les [tests dirigés par table](https://go.dev/wiki/TableDrivenTests) sont utiles quand vous voulez construire une liste de cas de test qui peuvent être testés de la même manière.

```go
func TestAire(t *testing.T) {

	testsAire := []struct {
		forme Forme
		attendu  float64
	}{
		{Rectangle{12, 6}, 72.0},
		{Cercle{10}, 314.1592653589793},
	}

	for _, tt := range testsAire {
		resultat := tt.forme.Aire()
		if resultat != tt.attendu {
			t.Errorf("reçu %g attendu %g", resultat, tt.attendu)
		}
	}

}
```

La seule nouvelle syntaxe ici est de créer une "struct anonyme", `testsAire`. Nous déclarons un slice de structs en utilisant `[]struct` avec deux champs, la `forme` et l'`attendu`. Puis nous remplissons le slice avec des cas.

Nous itérons ensuite sur eux juste comme nous le faisons avec tout autre slice, en utilisant les champs de struct pour exécuter nos tests.

Vous pouvez voir comme il serait très facile pour un développeur d'introduire une nouvelle forme, d'implémenter `Aire` et puis de l'ajouter aux cas de test. De plus, si un bug est trouvé avec `Aire`, il est très facile d'ajouter un nouveau cas de test pour l'exercer avant de le corriger.

Les tests dirigés par table peuvent être un excellent élément dans votre boîte à outils, mais assurez-vous d'avoir besoin du bruit supplémentaire dans les tests.
Ils sont un excellent ajustement quand vous souhaitez tester diverses implémentations d'une interface, ou si les données passées à une fonction ont beaucoup d'exigences différentes qui doivent être testées.

Démontrons tout ceci en ajoutant une autre forme et en la testant ; un triangle.

## Écrivez le test d'abord

Ajouter un nouveau test pour notre nouvelle forme est très facile. Ajoutez juste `{Triangle{12, 6}, 36.0},` à notre liste.

```go
func TestAire(t *testing.T) {

	testsAire := []struct {
		forme Forme
		attendu  float64
	}{
		{Rectangle{12, 6}, 72.0},
		{Cercle{10}, 314.1592653589793},
		{Triangle{12, 6}, 36.0},
	}

	for _, tt := range testsAire {
		resultat := tt.forme.Aire()
		if resultat != tt.attendu {
			t.Errorf("reçu %g attendu %g", resultat, tt.attendu)
		}
	}

}
```

## Essayez d'exécuter le test

Rappelez-vous, continuez à essayer d'exécuter le test et laissez le compilateur vous guider vers une solution.

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

`./formes_test.go:25:4: undefined: Triangle`

Nous n'avons pas encore défini `Triangle`

```go
type Triangle struct {
	Base   float64
	Hauteur float64
}
```

Essayez à nouveau

```text
./formes_test.go:25:8: cannot use Triangle literal (type Triangle) as type Forme in field value:
    Triangle does not implement Forme (missing Aire method)
```

Il nous dit que nous ne pouvons pas utiliser un `Triangle` comme une forme parce qu'il n'a pas de méthode `Aire()`, alors ajoutez une implémentation vide pour faire fonctionner le test

```go
func (t Triangle) Aire() float64 {
	return 0
}
```

Finalement le code compile et nous obtenons notre erreur

`formes_test.go:31: reçu 0.00 attendu 36.00`

## Écrivez assez de code pour le faire passer

```go
func (t Triangle) Aire() float64 {
	return (t.Base * t.Hauteur) * 0.5
}
```

Et nos tests passent !

## Refactoriser

Encore une fois, l'implémentation est correcte mais nos tests pourraient avoir quelques améliorations.

Quand vous scannez ceci

```
{Rectangle{12, 6}, 72.0},
{Cercle{10}, 314.1592653589793},
{Triangle{12, 6}, 36.0},
```

Il n'est pas immédiatement clair ce que tous les nombres représentent et vous devriez viser à ce que vos tests soient facilement compris.

Jusqu'à présent, on ne vous a montré que la syntaxe pour créer des instances de structs `MaStruct{val1, val2}` mais vous pouvez optionnellement nommer les champs.

Voyons à quoi ça ressemble

```
        {forme: Rectangle{Largeur: 12, Hauteur: 6}, attendu: 72.0},
        {forme: Cercle{Rayon: 10}, attendu: 314.1592653589793},
        {forme: Triangle{Base: 12, Hauteur: 6}, attendu: 36.0},
```

Dans [Test-Driven Development by Example](https://g.co/kgs/yCzDLF) Kent Beck refactorise quelques tests à un point et affirme :

> Le test nous parle plus clairement, comme s'il était une assertion de vérité, **pas une séquence d'opérations**

\(l'emphase dans la citation est la mienne\)

Maintenant nos tests - plutôt, la liste de cas de test - font des assertions de vérité sur les formes et leurs aires.

## Assurez-vous que la sortie de votre test est utile

Rappelez-vous plus tôt quand nous implémentions `Triangle` et nous avions le test qui échouait ? Il a imprimé `formes_test.go:31: reçu 0.00 attendu 36.00`.

Nous savions que c'était en relation avec `Triangle` parce que nous travaillions juste avec.
Mais que se passerait-il si un bug se glissait dans le système dans un des 20 cas dans la table ?
Comment un développeur saurait-il quel cas a échoué ?
Ce n'est pas une grande expérience pour le développeur, il devra regarder manuellement à travers les cas pour découvrir quel cas a réellement échoué.

Nous pouvons changer notre message d'erreur en `%#v reçu %g attendu %g`. La chaîne de format `%#v` imprimera notre struct avec les valeurs dans ses champs, donc le développeur peut voir d'un coup d'œil les propriétés qui sont testées.

Pour augmenter davantage la lisibilité de nos cas de test, nous pouvons renommer le champ `attendu` en quelque chose de plus descriptif comme `aAire`.

Un dernier conseil avec les tests dirigés par table est d'utiliser `t.Run` et de nommer les cas de test.

En enveloppant chaque cas dans un `t.Run`, vous aurez une sortie de test plus claire sur les échecs car elle imprimera le nom du cas

```text
--- FAIL: TestAire (0.00s)
    --- FAIL: TestAire/Rectangle (0.00s)
        formes_test.go:33: main.Rectangle{Largeur:12, Hauteur:6} reçu 72.00 attendu 72.10
```

Et vous pouvez exécuter des tests spécifiques dans votre table avec `go test -run TestAire/Rectangle`.

Voici notre code de test final qui capture ceci

```go
func TestAire(t *testing.T) {

	testsAire := []struct {
		nom    string
		forme   Forme
		aAire float64
	}{
		{nom: "Rectangle", forme: Rectangle{Largeur: 12, Hauteur: 6}, aAire: 72.0},
		{nom: "Cercle", forme: Cercle{Rayon: 10}, aAire: 314.1592653589793},
		{nom: "Triangle", forme: Triangle{Base: 12, Hauteur: 6}, aAire: 36.0},
	}

	for _, tt := range testsAire {
		// utilisation de tt.nom du cas pour l'utiliser comme nom de test `t.Run`
		t.Run(tt.nom, func(t *testing.T) {
			resultat := tt.forme.Aire()
			if resultat != tt.aAire {
				t.Errorf("%#v reçu %g attendu %g", tt.forme, resultat, tt.aAire)
			}
		})

	}

}
```

## Conclusion

C'était plus de pratique TDD, itérant sur nos solutions à des problèmes mathématiques de base et apprenant de nouvelles fonctionnalités du langage motivées par nos tests.

* Déclarer des structs pour créer vos propres types de données qui vous permet de regrouper des données connexes ensemble et rendre l'intention de votre code plus claire
* Déclarer des interfaces pour que vous puissiez définir des fonctions qui peuvent être utilisées par différents types \([polymorphisme paramétrique](https://en.wikipedia.org/wiki/Parametric_polymorphism)\)
* Ajouter des méthodes pour que vous puissiez ajouter des fonctionnalités à vos types de données et pour que vous puissiez implémenter des interfaces
* Tests dirigés par table pour rendre vos assertions plus claires et vos suites de tests plus faciles à étendre et maintenir

C'était un chapitre important parce que nous commençons maintenant à définir nos propres types. Dans les langages statiquement typés comme Go, être capable de concevoir vos propres types est essentiel pour construire des logiciels faciles à comprendre, à assembler et à tester.

Les interfaces sont un excellent outil pour cacher la complexité d'autres parties du système. Dans notre cas, notre code d'aide de test n'avait pas besoin de connaître la forme exacte sur laquelle il faisait des assertions, seulement comment "demander" son aire.

À mesure que vous devenez plus familier avec Go, vous commencerez à voir la vraie force des interfaces et de la bibliothèque standard. Vous apprendrez sur les interfaces définies dans la bibliothèque standard qui sont utilisées _partout_ et en les implémentant contre vos propres types, vous pouvez très rapidement réutiliser beaucoup de grandes fonctionnalités.
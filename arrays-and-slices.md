# Tableaux et slices

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/arrays)**

Les tableaux vous permettent de stocker plusieurs éléments du même type dans une variable dans un ordre particulier.

Quand vous avez des tableaux, il est très courant de devoir les parcourir. Alors utilisons [nos nouvelles connaissances de `for`](iteration.md) pour faire une fonction `Somme`. `Somme` prendra un tableau de nombres et retournera le total.

Utilisons nos compétences TDD

## Écrivez le test d'abord

Créez un nouveau dossier pour travailler. Créez un nouveau fichier appelé `somme_test.go` et insérez ce qui suit :

```go
package main

import "testing"

func TestSomme(t *testing.T) {

	nombres := [5]int{1, 2, 3, 4, 5}

	resultat := Somme(nombres)
	attendu := 15

	if resultat != attendu {
		t.Errorf("reçu %d attendu %d donné, %v", resultat, attendu, nombres)
	}
}
```

Les tableaux ont une _capacité fixe_ que vous définissez quand vous déclarez la variable.
Nous pouvons initialiser un tableau de deux façons :

* \[N\]type{valeur1, valeur2, ..., valeurN} ex. `nombres := [5]int{1, 2, 3, 4, 5}`
* \[...\]type{valeur1, valeur2, ..., valeurN} ex. `nombres := [...]int{1, 2, 3, 4, 5}`

Il est parfois utile d'imprimer aussi les entrées de la fonction dans le message d'erreur.
Ici, nous utilisons l'espace réservé `%v` pour imprimer le format "par défaut", qui fonctionne bien pour les tableaux.

[Lire plus sur les chaînes de format](https://golang.org/pkg/fmt/)

## Essayez d'exécuter le test

Si vous avez initialisé go mod avec `go mod init main`, vous recevrez une erreur
`_testmain.go:13:2: cannot import "main"`. C'est parce que selon la pratique courante,
le package main ne contiendra que l'intégration d'autres packages et pas de code testable unitairement et
donc Go ne vous permettra pas d'importer un package avec le nom `main`.

Pour corriger cela, vous pouvez renommer le module principal dans `go.mod` à n'importe quel autre nom.

Une fois l'erreur ci-dessus corrigée, si vous exécutez `go test`, le compilateur échouera avec l'erreur familière
`./somme_test.go:10:19: undefined: Somme`. Maintenant nous pouvons procéder à l'écriture de la méthode actuelle à tester.

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Dans `somme.go`

```go
package main

func Somme(nombres [5]int) int {
	return 0
}
```

Votre test devrait maintenant échouer avec _un message d'erreur clair_

`somme_test.go:13: reçu 0 attendu 15 donné, [1 2 3 4 5]`

## Écrivez assez de code pour le faire passer

```go
func Somme(nombres [5]int) int {
	somme := 0
	for i := 0; i < 5; i++ {
		somme += nombres[i]
	}
	return somme
}
```

Pour obtenir la valeur d'un tableau à un index particulier, utilisez juste la syntaxe `tableau[index]`.
Dans ce cas, nous utilisons `for` pour itérer 5 fois pour parcourir le tableau et ajouter chaque élément à `somme`.

## Refactoriser

Introduisons [`range`](https://gobyexample.com/range) pour aider à nettoyer notre code

```go
func Somme(nombres [5]int) int {
	somme := 0
	for _, nombre := range nombres {
		somme += nombre
	}
	return somme
}
```

`range` vous permet d'itérer sur un tableau. À chaque itération, `range` retourne deux valeurs - l'index et la valeur.
Nous choisissons d'ignorer la valeur d'index en utilisant `_` [identificateur vide](https://golang.org/doc/effective_go.html#blank).

### Les tableaux et leur type

Une propriété intéressante des tableaux est que la taille est encodée dans son type. Si vous essayez
de passer un `[4]int` dans une fonction qui attend un `[5]int`, ça ne compilera pas.
Ce sont des types différents donc c'est exactement comme essayer de passer une `string` dans
une fonction qui veut un `int`.

Vous pensez peut-être que c'est assez encombrant que les tableaux aient une longueur fixe, et la plupart
du temps vous n'en utiliserez probablement pas !

Go a des _slices_ qui n'encodent pas la taille de la collection et peuvent
avoir n'importe quelle taille.

La prochaine exigence sera de sommer des collections de tailles variables.

## Écrivez le test d'abord

Nous utiliserons maintenant le [type slice][slice] qui nous permet d'avoir des collections de
n'importe quelle taille. La syntaxe est très similaire aux tableaux, vous omettez juste la taille quand
vous les déclarez

`monSlice := []int{1,2,3}` plutôt que `monTableau := [3]int{1,2,3}`

```go
func TestSomme(t *testing.T) {

	t.Run("collection de 5 nombres", func(t *testing.T) {
		nombres := [5]int{1, 2, 3, 4, 5}

		resultat := Somme(nombres)
		attendu := 15

		if resultat != attendu {
			t.Errorf("reçu %d attendu %d donné, %v", resultat, attendu, nombres)
		}
	})

	t.Run("collection de n'importe quelle taille", func(t *testing.T) {
		nombres := []int{1, 2, 3}

		resultat := Somme(nombres)
		attendu := 6

		if resultat != attendu {
			t.Errorf("reçu %d attendu %d donné, %v", resultat, attendu, nombres)
		}
	})

}
```

## Essayez d'exécuter le test

Cela ne compile pas

`./somme_test.go:22:18: cannot use nombres (type []int) as type [5]int in argument to Somme`

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Le problème ici est que nous pouvons soit

* Casser l'API existante en changeant l'argument de `Somme` pour être un slice plutôt
  qu'un tableau. Quand nous faisons cela, nous ruinerons potentiellement
  la journée de quelqu'un parce que notre _autre_ test ne compilera plus !
* Créer une nouvelle fonction

Dans notre cas, personne d'autre n'utilise notre fonction, donc plutôt que d'avoir deux fonctions à maintenir, ayons juste une.

```go
func Somme(nombres []int) int {
	somme := 0
	for _, nombre := range nombres {
		somme += nombre
	}
	return somme
}
```

Si vous essayez d'exécuter les tests, ils ne compileront toujours pas, vous devrez changer le premier test pour passer un slice plutôt qu'un tableau.

## Écrivez assez de code pour le faire passer

Il s'avère que corriger les problèmes du compilateur était tout ce dont nous avions besoin ici et les tests passent !

## Refactoriser

Nous avons déjà refactorisé `Somme` - tout ce que nous avons fait était de remplacer les tableaux par des slices, donc aucun changement supplémentaire n'est requis.
Rappelez-vous que nous ne devons pas négliger notre code de test dans l'étape de refactorisation - nous pouvons améliorer davantage nos tests `Somme`.

```go
func TestSomme(t *testing.T) {

	t.Run("collection de 5 nombres", func(t *testing.T) {
		nombres := []int{1, 2, 3, 4, 5}

		resultat := Somme(nombres)
		attendu := 15

		if resultat != attendu {
			t.Errorf("reçu %d attendu %d donné, %v", resultat, attendu, nombres)
		}
	})

	t.Run("collection de n'importe quelle taille", func(t *testing.T) {
		nombres := []int{1, 2, 3}

		resultat := Somme(nombres)
		attendu := 6

		if resultat != attendu {
			t.Errorf("reçu %d attendu %d donné, %v", resultat, attendu, nombres)
		}
	})

}
```

Il est important de questionner la valeur de vos tests. Ce ne devrait pas être un objectif d'avoir
autant de tests que possible, mais plutôt d'avoir autant de _confiance_ que
possible dans votre base de code. Avoir trop de tests peut devenir un vrai problème
et cela ajoute juste plus de surcharge dans la maintenance. **Chaque test a un coût**.

Dans notre cas, vous pouvez voir qu'avoir deux tests pour cette fonction est redondant.
Si ça fonctionne pour un slice d'une taille, il est très probable que ça fonctionnera pour un slice de
n'importe quelle taille \(dans la limite du raisonnable\).

La boîte à outils de test intégrée de Go propose un [outil de couverture](https://blog.golang.org/cover).
Bien que viser 100% de couverture ne devrait pas être votre objectif final, l'outil de couverture peut aider
à identifier les zones de votre code non couvertes par les tests. Si vous avez été strict avec le TDD,
il est assez probable que vous ayez près de 100% de couverture de toute façon.

Essayez d'exécuter

`go test -cover`

Vous devriez voir

```bash
PASS
coverage: 100.0% of statements
```

Maintenant supprimez un des tests et vérifiez la couverture à nouveau.

Maintenant que nous sommes satisfaits d'avoir une fonction bien testée, vous devriez committer votre
excellent travail avant de relever le prochain défi.

Nous avons besoin d'une nouvelle fonction appelée `SommeTout` qui prendra un nombre variable de
slices, retournant un nouveau slice contenant les totaux pour chaque slice passé.

Par exemple

`SommeTout([]int{1,2}, []int{0,9})` retournerait `[]int{3, 9}`

ou

`SommeTout([]int{1,1,1})` retournerait `[]int{3}`

## Écrivez le test d'abord

```go
func TestSommeTout(t *testing.T) {

	resultat := SommeTout([]int{1, 2}, []int{0, 9})
	attendu := []int{3, 9}

	if resultat != attendu {
		t.Errorf("reçu %v attendu %v", resultat, attendu)
	}
}
```

## Essayez d'exécuter le test

`./somme_test.go:23:14: undefined: SommeTout`

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Nous devons définir `SommeTout` selon ce que notre test veut.

Go peut vous laisser écrire des [_fonctions variadiques_](https://gobyexample.com/variadic-functions) qui peuvent prendre un nombre variable d'arguments.

```go
func SommeTout(nombresASommer ...[]int) []int {
	return nil
}
```

C'est valide, mais nos tests ne compileront toujours pas !

`./somme_test.go:26:20: invalid operation: resultat != attendu (slice can only be compared to nil)`

Go ne vous permet pas d'utiliser les opérateurs d'égalité avec les slices. Vous _pourriez_ écrire
une fonction pour itérer sur chaque slice `resultat` et `attendu` et vérifier leurs valeurs
mais par commodité, nous pouvons utiliser [`reflect.DeepEqual`][deepEqual] qui est
utile pour voir si _deux_ variables quelconques sont identiques.

```go
func TestSommeTout(t *testing.T) {

	resultat := SommeTout([]int{1, 2}, []int{0, 9})
	attendu := []int{3, 9}

	if !reflect.DeepEqual(resultat, attendu) {
		t.Errorf("reçu %v attendu %v", resultat, attendu)
	}
}
```

\(assurez-vous d'`import reflect` en haut de votre fichier pour avoir accès à `DeepEqual`\)

Il est important de noter que `reflect.DeepEqual` n'est pas "type safe" - le code
compilera même si vous faites quelque chose d'un peu idiot. Pour voir cela en action,
changez temporairement le test vers :

```go
func TestSommeTout(t *testing.T) {

	resultat := SommeTout([]int{1, 2}, []int{0, 9})
	attendu := "bob"

	if !reflect.DeepEqual(resultat, attendu) {
		t.Errorf("reçu %v attendu %v", resultat, attendu)
	}
}
```

Ce que nous avons fait ici est d'essayer de comparer un `slice` avec une `string`. Cela n'a
aucun sens, mais le test compile ! Donc bien qu'utiliser `reflect.DeepEqual` soit
un moyen pratique de comparer des slices \(et d'autres choses\) vous devez faire attention
quand vous l'utilisez.

(Depuis Go 1.21, le package standard [slices](https://pkg.go.dev/slices#pkg-overview) est disponible, qui a une fonction [slices.Equal](https://pkg.go.dev/slices#Equal) pour faire une comparaison simple shallow sur les slices, où vous n'avez pas besoin de vous inquiéter des types comme le cas ci-dessus. Notez que cette fonction attend que les éléments soient [comparables](https://pkg.go.dev/builtin#comparable). Donc, elle ne peut pas être appliquée aux slices avec des éléments non-comparables comme les slices 2D.)

Changez le test à nouveau et exécutez-le. Vous devriez avoir une sortie de test comme la suivante

`somme_test.go:30: reçu [] attendu [3 9]`

## Écrivez assez de code pour le faire passer

Ce que nous devons faire est itérer sur les varargs, calculer la somme en utilisant notre
fonction `Somme` existante, puis l'ajouter au slice que nous retournerons

```go
func SommeTout(nombresASommer ...[]int) []int {
	longueurDesNombres := len(nombresASommer)
	sommes := make([]int, longueurDesNombres)

	for i, nombres := range nombresASommer {
		sommes[i] = Somme(nombres)
	}

	return sommes
}
```

Beaucoup de nouvelles choses à apprendre !

Il y a une nouvelle façon de créer un slice. `make` vous permet de créer un slice avec
une capacité de départ du `len` des `nombresASommer` que nous devons traiter. La longueur d'un slice est le nombre d'éléments qu'il contient `len(monSlice)`, tandis que la capacité est le nombre d'éléments qu'il peut contenir dans le tableau sous-jacent `cap(monSlice)`, ex., `make([]int, 0, 5)` crée un slice avec la longueur 0 et la capacité 5.

Vous pouvez indexer les slices comme les tableaux avec `monSlice[N]` pour obtenir la valeur ou
lui assigner une nouvelle valeur avec `=`

Les tests devraient maintenant passer.

## Refactoriser

Comme mentionné, les slices ont une capacité. Si vous avez un slice avec une capacité de
2 et essayez de faire `monSlice[10] = 1`, vous obtiendrez une erreur _runtime_.

Cependant, vous pouvez utiliser la fonction `append` qui prend un slice et une nouvelle valeur,
puis retourne un nouveau slice avec tous les éléments dedans.

```go
func SommeTout(nombresASommer ...[]int) []int {
	var sommes []int
	for _, nombres := range nombresASommer {
		sommes = append(sommes, Somme(nombres))
	}

	return sommes
}
```

Dans cette implémentation, nous nous soucions moins de la capacité. Nous commençons avec un
slice vide `sommes` et y ajoutons le résultat de `Somme` tandis que nous travaillons à travers les varargs.

Notre prochaine exigence est de changer `SommeTout` en `SommeToutesQueues`, où elle
calculera les totaux des "queues" de chaque slice. La queue d'une collection est
tous les éléments dans la collection sauf le premier \(la "tête"\).

## Écrivez le test d'abord

```go
func TestSommeToutesQueues(t *testing.T) {
	resultat := SommeToutesQueues([]int{1, 2}, []int{0, 9})
	attendu := []int{2, 9}

	if !reflect.DeepEqual(resultat, attendu) {
		t.Errorf("reçu %v attendu %v", resultat, attendu)
	}
}
```

## Essayez d'exécuter le test

`./somme_test.go:26:14: undefined: SommeToutesQueues`

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Renommez la fonction en `SommeToutesQueues` et relancez le test

`somme_test.go:30: reçu [3 9] attendu [2 9]`

## Écrivez assez de code pour le faire passer

```go
func SommeToutesQueues(nombresASommer ...[]int) []int {
	var sommes []int
	for _, nombres := range nombresASommer {
		queue := nombres[1:]
		sommes = append(sommes, Somme(queue))
	}

	return sommes
}
```

Les slices peuvent être slicés ! La syntaxe est `slice[bas:haut]`. Si vous omettez la valeur sur
un des côtés du `:`, elle capture tout de ce côté. Dans notre
cas, nous disons "prendre de 1 à la fin" avec `nombres[1:]`. Vous pourriez vouloir
passer du temps à écrire d'autres tests autour des slices et expérimenter avec l'
opérateur slice pour vous familiariser davantage avec.

## Refactoriser

Pas grand chose à refactoriser cette fois.

Que pensez-vous qu'il arriverait si vous passiez un slice vide dans notre
fonction ? Quelle est la "queue" d'un slice vide ? Que se passe-t-il quand vous dites à Go de
capturer tous les éléments depuis `monSliceVide[1:]` ?

## Écrivez le test d'abord

```go
func TestSommeToutesQueues(t *testing.T) {

	t.Run("faire les sommes de quelques slices", func(t *testing.T) {
		resultat := SommeToutesQueues([]int{1, 2}, []int{0, 9})
		attendu := []int{2, 9}

		if !reflect.DeepEqual(resultat, attendu) {
			t.Errorf("reçu %v attendu %v", resultat, attendu)
		}
	})

	t.Run("sommer en sécurité des slices vides", func(t *testing.T) {
		resultat := SommeToutesQueues([]int{}, []int{3, 4, 5})
		attendu := []int{0, 9}

		if !reflect.DeepEqual(resultat, attendu) {
			t.Errorf("reçu %v attendu %v", resultat, attendu)
		}
	})

}
```

## Essayez d'exécuter le test

```text
panic: runtime error: slice bounds out of range [recovered]
    panic: runtime error: slice bounds out of range
```

Oh non ! Il est important de noter que bien que le test _ait compilé_, il _a une erreur runtime_.

Les erreurs de temps de compilation sont nos amies parce qu'elles nous aident à écrire des logiciels qui fonctionnent,
les erreurs runtime sont nos ennemies parce qu'elles affectent nos utilisateurs.

## Écrivez assez de code pour le faire passer

```go
func SommeToutesQueues(nombresASommer ...[]int) []int {
	var sommes []int
	for _, nombres := range nombresASommer {
		if len(nombres) == 0 {
			sommes = append(sommes, 0)
		} else {
			queue := nombres[1:]
			sommes = append(sommes, Somme(queue))
		}
	}

	return sommes
}
```

## Refactoriser

Nos tests ont du code répété autour des assertions à nouveau, alors extrayons celles-ci dans une fonction.

```go
func TestSommeToutesQueues(t *testing.T) {

	verifierSommes := func(t testing.TB, resultat, attendu []int) {
		t.Helper()
		if !reflect.DeepEqual(resultat, attendu) {
			t.Errorf("reçu %v attendu %v", resultat, attendu)
		}
	}

	t.Run("faire les sommes de queues de", func(t *testing.T) {
		resultat := SommeToutesQueues([]int{1, 2}, []int{0, 9})
		attendu := []int{2, 9}
		verifierSommes(t, resultat, attendu)
	})

	t.Run("sommer en sécurité des slices vides", func(t *testing.T) {
		resultat := SommeToutesQueues([]int{}, []int{3, 4, 5})
		attendu := []int{0, 9}
		verifierSommes(t, resultat, attendu)
	})

}
```

Nous aurions pu créer une nouvelle fonction `verifierSommes` comme nous le faisons normalement, mais dans ce cas, nous montrons une nouvelle technique, assigner une fonction à une variable. Cela pourrait paraître étrange mais, ce n'est pas différent d'assigner une variable à une `string`, ou un `int`, les fonctions en effet sont aussi des valeurs.

Ce n'est pas montré ici, mais cette technique peut être utile quand vous voulez lier une fonction à d'autres variables locales dans la "portée" (ex entre quelques `{}`). Elle vous permet aussi de réduire la surface de votre API.

En définissant cette fonction à l'intérieur du test, elle ne peut pas être utilisée par d'autres fonctions dans ce package. Cacher des variables et fonctions qui n'ont pas besoin d'être exportées est une considération de design importante.

Une conséquence pratique de ceci est que cela ajoute un peu de *type-safety* à notre code. Si
un développeur ajoute par erreur un nouveau test avec `verifierSommes(t, resultat, "dave")`, le compilateur
l'arrêtera dans ses traces.

```bash
$ go test
./somme_test.go:52:46: cannot use "dave" (type string) as type []int in argument to verifierSommes
```

## Conclusion

Nous avons couvert

* Tableaux
* Slices
  * Les diverses façons de les créer
  * Comment ils ont une capacité _fixe_ mais vous pouvez créer de nouveaux slices depuis les anciens
    en utilisant `append`
  * Comment slicer les slices !
* `len` pour obtenir la longueur d'un tableau ou slice
* Outil de couverture de test
* `reflect.DeepEqual` et pourquoi c'est utile mais peut réduire la type-safety de votre code

Nous avons utilisé des slices et tableaux avec des entiers mais ils fonctionnent avec tout autre type
aussi, incluant des tableaux/slices eux-mêmes. Donc vous pouvez déclarer une variable de
`[][]string` si vous en avez besoin.

[Consultez le post du blog Go sur les slices][blog-slice] pour un regard en profondeur sur
les slices. Essayez d'écrire plus de tests pour solidifier ce que vous apprenez de sa lecture.

Un autre moyen pratique d'expérimenter avec Go autre qu'écrire des tests est le Go
playground. Vous pouvez essayer la plupart des choses et vous pouvez facilement partager votre code si
vous devez poser des questions. [J'ai fait un go playground avec un slice dedans pour que vous puissiez expérimenter avec.](https://play.golang.org/p/ICCWcRGIO68)

[Voici un exemple](https://play.golang.org/p/bTrRmYfNYCp) de slicer un tableau
et comment changer le slice affecte le tableau original ; mais une "copie" du slice
n'affectera pas le tableau original.
[Un autre exemple](https://play.golang.org/p/Poth8JS28sc) de pourquoi c'est une bonne idée
de faire une copie d'un slice après avoir slicé un très grand slice.

[for]: ../iteration.md#
[blog-slice]: https://blog.golang.org/go-slices-usage-and-internals
[deepEqual]: https://golang.org/pkg/reflect/#DeepEqual
[slice]: https://golang.org/doc/effective_go.html#slices
# Revisiter les tableaux et les slices avec les génériques

**[Le code pour ce chapitre est une continuation des Tableaux et Slices, disponible ici](https://github.com/quii/learn-go-with-tests/tree/main/arrays)**

Examinez à la fois `SumAll` et `SumAllTails` que nous avons écrits dans [tableaux et slices](arrays-and-slices.md). Si vous n'avez pas votre version, veuillez copier le code du chapitre [tableaux et slices](arrays-and-slices.md) ainsi que les tests.

```go
// Sum calcule le total à partir d'une slice de nombres.
func Sum(numbers []int) int {
	var sum int
	for _, number := range numbers {
		sum += number
	}
	return sum
}

// SumAllTails calcule les sommes de tous les nombres sauf le premier, étant donné une collection de slices.
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		if len(numbers) == 0 {
			sums = append(sums, 0)
		} else {
			tail := numbers[1:]
			sums = append(sums, Sum(tail))
		}
	}

	return sums
}
```

Voyez-vous un motif récurrent ?

- Créer une sorte de valeur de résultat "initiale".
- Itérer sur la collection, en appliquant une sorte d'opération (ou fonction) au résultat et à l'élément suivant dans la slice, en définissant une nouvelle valeur pour le résultat
- Retourner le résultat.

Cette idée est couramment évoquée dans les cercles de programmation fonctionnelle, souvent appelée 'reduce' ou [fold](https://fr.wikipedia.org/wiki/Fold_(higher-order_function)).

> En programmation fonctionnelle, fold (également appelé reduce, accumulate, aggregate, compress, ou inject) fait référence à une famille de fonctions d'ordre supérieur qui analysent une structure de données récursive et, par l'utilisation d'une opération de combinaison donnée, recombinent les résultats du traitement récursif de ses parties constitutives, construisant une valeur de retour. Typiquement, un fold est présenté avec une fonction de combinaison, un nœud supérieur d'une structure de données, et éventuellement certaines valeurs par défaut à utiliser dans certaines conditions. Le fold procède ensuite à combiner des éléments de la hiérarchie de la structure de données, en utilisant la fonction de manière systématique.

Go a toujours eu des fonctions d'ordre supérieur, et depuis la version 1.18, il dispose également de [génériques](./generics.md), il est donc maintenant possible de définir certaines de ces fonctions dont on parle dans notre domaine plus large. Il n'y a aucune raison de se cacher la tête dans le sable, c'est une abstraction très courante en dehors de l'écosystème Go et il sera bénéfique de la comprendre.

Maintenant, je sais que certains d'entre vous grimacent probablement.

> Go est censé être simple

**Ne confondez pas facilité et simplicité**. Faire des boucles et copier-coller du code est facile, mais ce n'est pas nécessairement simple. Pour en savoir plus sur la simplicité et la facilité, regardez [le chef-d'œuvre de Rich Hickey - Simple Made Easy](https://www.youtube.com/watch?v=SxdOUGdseq4).

**Ne confondez pas non-familiarité et complexité**. Fold/reduce peut initialement sembler effrayant et informatique, mais ce n'est en réalité qu'une abstraction d'une opération très courante. Prendre une collection et la combiner en un seul élément. Quand vous prenez du recul, vous réaliserez que vous faites probablement cela _très souvent_.

## Un refactoring générique

Une erreur que les gens font souvent avec les nouvelles fonctionnalités de langage est qu'ils commencent par les utiliser sans avoir de cas d'utilisation concret. Ils s'appuient sur des conjectures et des suppositions pour guider leurs efforts.

Heureusement, nous avons écrit nos fonctions "utiles" et avons des tests autour d'elles, donc nous sommes maintenant libres d'expérimenter des idées dans la phase de refactoring du TDD et de savoir que quoi que nous essayions, il y a une vérification de sa valeur via nos tests unitaires.

Utiliser les génériques comme outil pour simplifier le code via l'étape de refactoring est beaucoup plus susceptible de vous guider vers des améliorations utiles, plutôt que des abstractions prématurées.

Nous sommes libres d'essayer des choses, de relancer nos tests, si nous aimons le changement, nous pouvons le valider. Sinon, il suffit d'annuler le changement. Cette liberté d'expérimenter est l'une des valeurs vraiment énormes du TDD.

Vous devriez être familier avec la syntaxe des génériques [du chapitre précédent](generics.md), essayez d'écrire votre propre fonction `Reduce` et utilisez-la dans `Sum` et `SumAllTails`.

### Indices

Si vous réfléchissez d'abord aux arguments de votre fonction, cela vous donnera un très petit ensemble de solutions valides
  - Le tableau que vous souhaitez réduire
  - Une sorte de fonction de combinaison

"Reduce" est un modèle incroyablement bien documenté, il n'est pas nécessaire de réinventer la roue. [Lisez le wiki, en particulier la section sur les listes](https://en.wikipedia.org/wiki/Fold_(higher-order_function)), cela devrait vous suggérer un autre argument dont vous aurez besoin.

> En pratique, il est pratique et naturel d'avoir une valeur initiale

### Ma première version de `Reduce`

```go
func Reduce[A any](collection []A, f func(A, A) A, initialValue A) A {
	var result = initialValue
	for _, x := range collection {
		result = f(result, x)
	}
	return result
}
```

Reduce capture l'_essence_ du modèle, c'est une fonction qui prend une collection, une fonction d'accumulation, une valeur initiale, et renvoie une seule valeur. Il n'y a pas de distractions désordonnées autour des types concrets.

Si vous comprenez la syntaxe des génériques, vous ne devriez avoir aucun problème à comprendre ce que fait cette fonction. En utilisant le terme reconnu `Reduce`, les programmeurs d'autres langages comprennent également l'intention.

### L'utilisation

```go
// Sum calcule le total à partir d'une slice de nombres.
func Sum(numbers []int) int {
	add := func(acc, x int) int { return acc + x }
	return Reduce(numbers, add, 0)
}

// SumAllTails calcule les sommes de tous les nombres sauf le premier, étant donné une collection de slices.
func SumAllTails(numbers ...[]int) []int {
	sumTail := func(acc, x []int) []int {
		if len(x) == 0 {
			return append(acc, 0)
		} else {
			tail := x[1:]
			return append(acc, Sum(tail))
		}
	}

	return Reduce(numbers, sumTail, []int{})
}
```

`Sum` et `SumAllTails` décrivent maintenant le comportement de leurs calculs comme les fonctions déclarées sur leurs premières lignes respectives. L'acte d'exécuter le calcul sur la collection est abstrait dans `Reduce`.

## Autres applications de reduce

En utilisant des tests, nous pouvons jouer avec notre fonction reduce pour voir à quel point elle est réutilisable. J'ai copié nos fonctions d'assertion génériques du chapitre précédent.

```go
func TestReduce(t *testing.T) {
	t.Run("multiplication de tous les éléments", func(t *testing.T) {
		multiply := func(x, y int) int {
			return x * y
		}

		AssertEqual(t, Reduce([]int{1, 2, 3}, multiply, 1), 6)
	})

	t.Run("concaténation de chaînes", func(t *testing.T) {
		concatenate := func(x, y string) string {
			return x + y
		}

		AssertEqual(t, Reduce([]string{"a", "b", "c"}, concatenate, ""), "abc")
	})
}
```

### La valeur zéro

Dans l'exemple de multiplication, nous montrons la raison d'avoir une valeur par défaut comme argument pour `Reduce`. Si nous nous appuyions sur la valeur par défaut de Go de 0 pour `int`, nous multiplierions notre valeur initiale par 0, puis les suivantes, donc vous n'obtiendriez jamais que 0. En la définissant à 1, le premier élément de la slice restera le même, et le reste se multipliera par les éléments suivants.

Si vous souhaitez paraître intelligent avec vos amis nerds, vous appelleriez cela [L'élément neutre](https://fr.wikipedia.org/wiki/Élément_neutre).

> En mathématiques, un élément neutre, ou élément identité, d'une opération binaire opérant sur un ensemble est un élément de l'ensemble qui laisse inchangé tout élément de l'ensemble lorsque l'opération est appliquée.

Dans l'addition, l'élément neutre est 0.

`1 + 0 = 1`

Avec la multiplication, c'est 1.

`1 * 1 = 1`

## Et si nous voulons réduire dans un type différent de `A` ?

Supposons que nous ayons une liste de transactions `Transaction` et que nous voulions une fonction qui les prendrait, plus un nom pour calculer leur solde bancaire.

Suivons le processus TDD.

## Écrire le test d'abord

```go
func TestBadBank(t *testing.T) {
	transactions := []Transaction{
		{
			From: "Chris",
			To:   "Riya",
			Sum:  100,
		},
		{
			From: "Adil",
			To:   "Chris",
			Sum:  25,
		},
	}

	AssertEqual(t, BalanceFor(transactions, "Riya"), 100)
	AssertEqual(t, BalanceFor(transactions, "Chris"), -75)
	AssertEqual(t, BalanceFor(transactions, "Adil"), -25)
}
```

## Essayer d'exécuter le test
```
# github.com/quii/learn-go-with-tests/arrays/v8 [github.com/quii/learn-go-with-tests/arrays/v8.test]
./bad_bank_test.go:6:20: undefined: Transaction
./bad_bank_test.go:18:14: undefined: BalanceFor
```

## Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test en échec

Nous n'avons pas encore nos types ou fonctions, ajoutons-les pour faire fonctionner le test.

```go
type Transaction struct {
	From string
	To   string
	Sum  float64
}

func BalanceFor(transactions []Transaction, name string) float64 {
	return 0.0
}
```

Lorsque vous exécutez le test, vous devriez voir ce qui suit :

```
=== RUN   TestBadBank
    bad_bank_test.go:19: got 0, want 100
    bad_bank_test.go:20: got 0, want -75
    bad_bank_test.go:21: got 0, want -25
--- FAIL: TestBadBank (0.00s)
```

## Écrire suffisamment de code pour le faire passer

Écrivons d'abord le code comme si nous n'avions pas de fonction `Reduce`.

```go
func BalanceFor(transactions []Transaction, name string) float64 {
	var balance float64
	for _, t := range transactions {
		if t.From == name {
			balance -= t.Sum
		}
		if t.To == name {
			balance += t.Sum
		}
	}
	return balance
}
```

## Refactoring

À ce stade, ayez une discipline de contrôle de source et validez votre travail. Nous avons un logiciel fonctionnel, prêt à défier Monzo, Barclays, et al.

Maintenant que notre travail est validé, nous sommes libres de jouer avec, et d'essayer des idées différentes dans la phase de refactoring. Pour être honnête, le code que nous avons n'est pas vraiment mauvais, mais pour les besoins de cet exercice, je veux démontrer le même code en utilisant `Reduce`.

```go
func BalanceFor(transactions []Transaction, name string) float64 {
	adjustBalance := func(currentBalance float64, t Transaction) float64 {
		if t.From == name {
			return currentBalance - t.Sum
		}
		if t.To == name {
			return currentBalance + t.Sum
		}
		return currentBalance
	}
	return Reduce(transactions, adjustBalance, 0.0)
}
```

Mais cela ne compilera pas.

```
./bad_bank.go:19:35: type func(acc float64, t Transaction) float64 of adjustBalance does not match inferred type func(Transaction, Transaction) Transaction for func(A, A) A
```

La raison est que nous essayons de réduire à un type _différent_ du type de la collection. Cela semble effrayant, mais en réalité, il suffit d'ajuster la signature de type de `Reduce` pour que cela fonctionne. Nous n'aurons pas à changer le corps de la fonction, et nous n'aurons pas à changer nos appelants existants.

```go
func Reduce[A, B any](collection []A, f func(B, A) B, initialValue B) B {
	var result = initialValue
	for _, x := range collection {
		result = f(result, x)
	}
	return result
}
```

Nous avons ajouté une seconde contrainte de type qui nous a permis d'assouplir les contraintes sur `Reduce`. Cela permet aux gens de `Reduce` d'une collection de `A` en un `B`. Dans notre cas, de `Transaction` à `float64`.

Cela rend `Reduce` plus polyvalent et réutilisable, tout en restant sûr au niveau des types. Si vous essayez de relancer les tests, ils devraient compiler et passer.

## Extension de la banque

Pour le plaisir, j'ai voulu améliorer l'ergonomie du code bancaire. J'ai omis le processus TDD par souci de brièveté.

```go
func TestBadBank(t *testing.T) {
	var (
		riya  = Account{Name: "Riya", Balance: 100}
		chris = Account{Name: "Chris", Balance: 75}
		adil  = Account{Name: "Adil", Balance: 200}

		transactions = []Transaction{
			NewTransaction(chris, riya, 100),
			NewTransaction(adil, chris, 25),
		}
	)

	newBalanceFor := func(account Account) float64 {
		return NewBalanceFor(account, transactions).Balance
	}

	AssertEqual(t, newBalanceFor(riya), 200)
	AssertEqual(t, newBalanceFor(chris), 0)
	AssertEqual(t, newBalanceFor(adil), 175)
}
```

Et voici le code mis à jour

```go
package main

type Transaction struct {
	From string
	To   string
	Sum  float64
}

func NewTransaction(from, to Account, sum float64) Transaction {
	return Transaction{From: from.Name, To: to.Name, Sum: sum}
}

type Account struct {
	Name    string
	Balance float64
}

func NewBalanceFor(account Account, transactions []Transaction) Account {
	return Reduce(
		transactions,
		applyTransaction,
		account,
	)
}

func applyTransaction(a Account, transaction Transaction) Account {
	if transaction.From == a.Name {
		a.Balance -= transaction.Sum
	}
	if transaction.To == a.Name {
		a.Balance += transaction.Sum
	}
	return a
}
```

Je pense que cela montre vraiment la puissance de l'utilisation de concepts comme `Reduce`. Le `NewBalanceFor` semble plus _déclaratif_, décrivant _ce qui_ se passe, plutôt que _comment_. Souvent, lorsque nous lisons du code, nous parcourons de nombreux fichiers, et nous essayons de comprendre _ce qui_ se passe, plutôt que _comment_, et ce style de code facilite bien cela.

Si je souhaite approfondir les détails, je peux, et je peux voir la _logique métier_ de `applyTransaction` sans me soucier des boucles et de l'état changeant; `Reduce` s'en occupe séparément.


### Fold/reduce sont assez universels

Les possibilités sont infinies™️ avec `Reduce` (ou `Fold`). C'est un modèle commun pour une raison, ce n'est pas seulement pour l'arithmétique ou la concaténation de chaînes. Essayez quelques autres applications.

- Pourquoi ne pas mélanger certains `color.RGBA` en une seule couleur ?
- Totaliser le nombre de votes dans un sondage, ou d'articles dans un panier d'achat.
- Plus ou moins tout ce qui implique le traitement d'une liste.

## Find

Maintenant que Go a des génériques, en les combinant avec des fonctions d'ordre supérieur, nous pouvons réduire beaucoup de code passe-partout dans nos projets, pour aider nos systèmes à être plus faciles à comprendre et à gérer.

Vous n'avez plus besoin d'écrire des fonctions `Find` spécifiques pour chaque type de collection que vous souhaitez rechercher, réutilisez plutôt ou écrivez une fonction `Find`. Si vous avez compris la fonction `Reduce` ci-dessus, écrire une fonction `Find` sera trivial.

Voici un test

```go
func TestFind(t *testing.T) {
	t.Run("trouver le premier nombre pair", func(t *testing.T) {
		numbers := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

		firstEvenNumber, found := Find(numbers, func(x int) bool {
			return x%2 == 0
		})
		AssertTrue(t, found)
		AssertEqual(t, firstEvenNumber, 2)
	})
}
```

Et voici l'implémentation

```go
func Find[A any](items []A, predicate func(A) bool) (value A, found bool) {
	for _, v := range items {
		if predicate(v) {
			return v, true
		}
	}
	return
}
```

Encore une fois, comme elle prend un type générique, nous pouvons la réutiliser de nombreuses façons

```go
type Person struct {
	Name string
}

t.Run("Trouver le meilleur programmeur", func(t *testing.T) {
	people := []Person{
		Person{Name: "Kent Beck"},
		Person{Name: "Martin Fowler"},
		Person{Name: "Chris James"},
	}

	king, found := Find(people, func(p Person) bool {
		return strings.Contains(p.Name, "Chris")
	})

	AssertTrue(t, found)
	AssertEqual(t, king, Person{Name: "Chris James"})
})
```

Comme vous pouvez le voir, ce code est sans faille.

## Conclusion

Lorsqu'elles sont utilisées avec goût, les fonctions d'ordre supérieur comme celles-ci rendront votre code plus simple à lire et à maintenir, mais rappelez-vous la règle empirique :

Utilisez le processus TDD pour développer un comportement réel et spécifique dont vous avez réellement besoin, dans la phase de refactoring, vous _pourriez_ alors découvrir des abstractions utiles pour aider à nettoyer le code.

Pratiquez la combinaison du TDD avec de bonnes habitudes de contrôle de source. Validez votre travail lorsque votre test passe, _avant_ d'essayer de refactoriser. De cette façon, si vous faites un gâchis, vous pouvez facilement revenir à votre état de fonctionnement.

### Les noms sont importants

Faites un effort pour faire des recherches en dehors de Go, afin de ne pas réinventer des modèles qui existent déjà avec un nom déjà établi.

Écrire une fonction qui prend une collection de `A` et les convertit en `B` ? Ne l'appelez pas `Convert`, c'est [`Map`](https://en.wikipedia.org/wiki/Map_(higher-order_function)). Utiliser le nom "propre" pour ces éléments réduira la charge cognitive pour les autres et rendra plus facile la recherche pour en apprendre davantage.

### Cela ne semble pas idiomatique ?

Essayez d'avoir l'esprit ouvert.

Bien que les idiomes de Go ne changeront pas, et ne devraient pas changer _radicalement_ en raison de la sortie des génériques, les idiomes _vont_ changer - en raison du changement de langage ! Ce ne devrait pas être un point controversé.

Dire

> Ce n'est pas idiomatique

Sans plus de détails, n'est pas une chose actionnable ou utile à dire. Surtout lorsqu'on discute de nouvelles fonctionnalités de langage.

Discutez avec vos collègues des modèles et du style de code en fonction de leurs mérites plutôt que du dogme. Tant que vous avez des tests bien conçus, vous pourrez toujours refactoriser et faire évoluer les choses au fur et à mesure que vous comprendrez ce qui fonctionne bien pour vous et votre équipe.

### Ressources

Fold est un véritable fondamental en informatique. Voici quelques ressources intéressantes si vous souhaitez en savoir plus
- [Wikipedia: Fold](https://en.wikipedia.org/wiki/Fold)
- [Un tutoriel sur l'universalité et l'expressivité de fold](http://www.cs.nott.ac.uk/~pszgmh/fold.pdf)
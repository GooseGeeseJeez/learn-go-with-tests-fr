# Itération

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/for)**

Pour faire des choses de manière répétée en Go, vous aurez besoin de `for`. En Go, il n'y a pas de mots-clés `while`, `do`, `until`, vous ne pouvez utiliser que `for`. Ce qui est une bonne chose !

Écrivons un test pour une fonction qui répète un caractère 5 fois.

Il n'y a rien de nouveau jusqu'à présent, alors essayez de l'écrire vous-même pour pratiquer.

## Écrivez le test d'abord

```go
package iteration

import "testing"

func TestRepeter(t *testing.T) {
	repete := Repeter("a")
	attendu := "aaaaa"

	if repete != attendu {
		t.Errorf("attendu %q mais reçu %q", attendu, repete)
	}
}
```

## Essayez d'exécuter le test

`./repeter_test.go:6:11: undefined: Repeter`

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

_Restez discipliné !_ Vous n'avez pas besoin de connaître quoi que ce soit de nouveau maintenant pour faire échouer le test correctement.

Tout ce que vous devez faire maintenant est suffisant pour le faire compiler afin que vous puissiez vérifier que votre test est bien écrit.

```go
package iteration

func Repeter(caractere string) string {
	return ""
}
```

N'est-ce pas agréable de savoir que vous connaissez déjà assez de Go pour écrire des tests pour quelques problèmes de base ? Cela signifie que vous pouvez maintenant jouer avec le code de production autant que vous le souhaitez et savoir qu'il se comporte comme vous l'espériez.

`repeter_test.go:10: attendu 'aaaaa' mais reçu ''`

## Écrivez assez de code pour le faire passer

La syntaxe `for` n'a rien de spécial et suit la plupart des langages de type C.

```go
func Repeter(caractere string) string {
	var repete string
	for i := 0; i < 5; i++ {
		repete = repete + caractere
	}
	return repete
}
```

Contrairement à d'autres langages comme C, Java ou JavaScript, il n'y a pas de parenthèses autour des trois composants de l'instruction for et les accolades `{ }` sont toujours requises. Vous pourriez vous demander ce qui se passe dans la ligne

```go
	var repete string
```

car nous avons utilisé `:=` jusqu'à présent pour déclarer et initialiser des variables. Cependant, `:=` est simplement [un raccourci pour les deux étapes](https://gobyexample.com/variables). Ici, nous déclarons seulement une variable `string`. D'où la version explicite. Nous pouvons aussi utiliser `var` pour déclarer des fonctions, comme nous le verrons plus tard.

Exécutez le test et il devrait passer.

Les variantes supplémentaires de la boucle for sont décrites [ici](https://gobyexample.com/for).

## Refactoriser

Maintenant il est temps de refactoriser et d'introduire une autre construction : `+=`, l'opérateur d'affectation.

```go
const nombreRepetitions = 5

func Repeter(caractere string) string {
	var repete string
	for i := 0; i < nombreRepetitions; i++ {
		repete += caractere
	}
	return repete
}
```

`+=` appelé _"l'opérateur d'addition ET d'affectation"_, il ajoute l'opérande de droite à l'opérande de gauche et affecte le résultat à l'opérande de gauche. Il fonctionne avec d'autres types comme les entiers.

### Benchmarking

Écrire des [benchmarks](https://golang.org/pkg/testing/#hdr-Benchmarks) en Go est une autre fonctionnalité de première classe du langage et c'est très similaire à l'écriture de tests.

```go
func BenchmarkRepeter(b *testing.B) {
	for b.Loop() {
		Repeter("a")
	}
}
```

Vous verrez que le code est très similaire à un test.

Le `testing.B` vous donne accès à la fonction de boucle. `Loop()` retourne true tant que le benchmark devrait continuer à s'exécuter.

Quand le code de benchmark est exécuté, il mesure combien de temps cela prend. Après que `Loop()` retourne false, `b.N` contient le nombre total d'itérations qui ont été exécutées.

Le nombre de fois que le code est exécuté ne devrait pas vous importer, le framework déterminera quelle est une "bonne" valeur pour cela pour vous permettre d'avoir quelques résultats décents.

Pour exécuter les benchmarks, faites `go test -bench=.` (ou si vous êtes dans Windows Powershell `go test -bench="."`)

```text
goos: darwin
goarch: amd64
pkg: github.com/quii/learn-go-with-tests/for/v4
10000000           136 ns/op
PASS
```

Ce que `136 ns/op` signifie, c'est que notre fonction prend en moyenne 136 nanosecondes à s'exécuter \(sur mon ordinateur\). Ce qui est plutôt correct ! Pour tester cela, elle l'a exécuté 10000000 fois.

**Note :** Par défaut, les benchmarks sont exécutés séquentiellement.

Seul le corps de la boucle est chronométré ; il exclut automatiquement le code de configuration et de nettoyage du chronométrage du benchmark. Un benchmark typique est structuré comme :

```go
func Benchmark(b *testing.B) {
	//... configuration ...
	for b.Loop() {
		//... code à mesurer ...
	}
	//... nettoyage ...
}
```

Les chaînes en Go sont immuables, ce qui signifie que chaque concaténation, comme dans notre fonction `Repeter`, implique de copier de la mémoire pour accommoder la nouvelle chaîne. Cela impacte les performances, particulièrement lors de concaténations lourdes de chaînes.

La bibliothèque standard fournit le type `strings.Builder` (voir [stringsBuilder]) qui minimise la copie de mémoire.
Il implémente une méthode `WriteString` que nous pouvons utiliser pour concaténer des chaînes :

```go
const nombreRepetitions = 5

func Repeter(caractere string) string {
	var repete strings.Builder
	for i := 0; i < nombreRepetitions; i++ {
		repete.WriteString(caractere)
	}
	return repete.String()
}
```

**Note** : Nous devons appeler la méthode `String` pour récupérer le résultat final.

Nous pouvons utiliser `BenchmarkRepeter` pour confirmer que `strings.Builder` améliore significativement les performances.
Exécutez `go test -bench=. -benchmem` :

```text
goos: darwin
goarch: amd64
pkg: github.com/quii/learn-go-with-tests/for/v4
10000000           25.70 ns/op           8 B/op           1 allocs/op
PASS
```

Le flag `-benchmem` rapporte des informations sur les allocations mémoire :

* `B/op` : le nombre d'octets alloués par itération
* `allocs/op` : le nombre d'allocations mémoire par itération

## Exercices d'entraînement

* Changez le test pour qu'un appelant puisse spécifier combien de fois le caractère est répété et puis corrigez le code
* Écrivez `ExampleRepeter` pour documenter votre fonction
* Jetez un œil au package [strings](https://golang.org/pkg/strings). Trouvez des fonctions que vous pensez pourraient être utiles et expérimentez avec elles en écrivant des tests comme nous l'avons fait ici. Investir du temps à apprendre la bibliothèque standard sera vraiment payant avec le temps.

## Conclusion

* Plus de pratique TDD
* Appris `for`
* Appris comment écrire des benchmarks

[stringsBuilder]: https://pkg.go.dev/strings#Builder
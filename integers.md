# Entiers

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/integers)**

Les entiers fonctionnent comme vous vous y attendriez. Écrivons une fonction `Additionner` pour tester les choses. Créez un fichier de test appelé `additionneurs_test.go` et écrivez ce code.

**Note :** Les fichiers sources Go ne peuvent avoir qu'un seul `package` par répertoire. Assurez-vous que vos fichiers sont organisés dans leurs propres packages. [Voici une bonne explication sur ce sujet.](https://dave.cheney.net/2014/12/01/five-suggestions-for-setting-up-a-go-project)

Votre répertoire de projet pourrait ressembler à quelque chose comme ceci :

```
apprendreGoAvecTests
    |
    |-> bonjourmonde
    |    |- bonjour.go
    |    |- bonjour_test.go
    |
    |-> entiers
    |    |- additionneurs_test.go
    |
    |- go.mod
    |- README.md
```

## Écrivez le test d'abord

```go
package entiers

import "testing"

func TestAdditionneurs(t *testing.T) {
	somme := Additionner(2, 2)
	attendu := 4

	if somme != attendu {
		t.Errorf("attendu '%d' mais reçu '%d'", attendu, somme)
	}
}
```

Vous remarquerez que nous utilisons `%d` comme chaînes de format plutôt que `%q`. C'est parce que nous voulons qu'il imprime un entier plutôt qu'une chaîne.

Notez aussi que nous n'utilisons plus le package main, au lieu de cela nous avons défini un package nommé `entiers`, comme le nom le suggère, cela regroupera les fonctions pour travailler avec les entiers telles que `Additionner`.

## Essayez d'exécuter le test

Exécutez le test `go test`

Inspectez l'erreur de compilation

`./additionneurs_test.go:6:10: undefined: Additionner`

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Écrivez assez de code pour satisfaire le compilateur _et c'est tout_ - rappelez-vous que nous voulons vérifier que nos tests échouent pour la bonne raison.

```go
package entiers

func Additionner(x, y int) int {
	return 0
}
```

Rappelez-vous, quand vous avez plus d'un argument du même type \(dans notre cas deux entiers\) plutôt que d'avoir `(x int, y int)` vous pouvez le raccourcir à `(x, y int)`.

Maintenant exécutez les tests, et nous devrions être heureux que le test rapporte correctement ce qui ne va pas.

`additionneurs_test.go:10: attendu '4' mais reçu '0'`

Si vous avez remarqué, nous avons appris sur les _valeurs de retour nommées_ dans la [dernière](hello-world.md#un...dernier...refactoring?) section mais nous n'utilisons pas la même chose ici. Elle devrait généralement être utilisée quand le sens du résultat n'est pas clair dans le contexte, dans notre cas il est assez clair que la fonction `Additionner` va additionner les paramètres. Vous pouvez vous référer à [ce](https://go.dev/wiki/CodeReviewComments#named-result-parameters) wiki pour plus de détails.

## Écrivez assez de code pour le faire passer

Dans le sens le plus strict du TDD, nous devrions maintenant écrire la _quantité minimale de code pour faire passer le test_. Un programmeur pointilleux pourrait faire ceci

```go
func Additionner(x, y int) int {
	return 4
}
```

Ah ha ! Déjoué encore une fois, le TDD est une arnaque n'est-ce pas ?

Nous pourrions écrire un autre test, avec quelques nombres différents pour forcer ce test à échouer mais cela ressemble à [un jeu de chat et souris](https://en.m.wikipedia.org/wiki/Cat_and_mouse).

Une fois que nous serons plus familiers avec la syntaxe de Go, j'introduirai une technique appelée _"Property Based Testing"_, qui arrêterait d'énerver les développeurs et vous aiderait à trouver des bugs.

Pour l'instant, réparons-le correctement

```go
func Additionner(x, y int) int {
	return x + y
}
```

Si vous relancez les tests, ils devraient passer.

## Refactoriser

Il n'y a pas beaucoup dans le code _réel_ que nous pouvons vraiment améliorer ici.

Nous avons exploré plus tôt comment en nommant l'argument de retour il apparaît dans la documentation mais aussi dans la plupart des éditeurs de texte des développeurs.

C'est génial parce que cela aide l'utilisabilité du code que vous écrivez. Il est préférable qu'un utilisateur puisse comprendre l'utilisation de votre code en regardant juste la signature du type et la documentation.

Vous pouvez ajouter de la documentation aux fonctions avec des commentaires, et ceux-ci apparaîtront dans Go Doc tout comme quand vous regardez la documentation de la bibliothèque standard.

```go
// Additionner prend deux entiers et retourne leur somme.
func Additionner(x, y int) int {
	return x + y
}
```

### Exemples testables

Si vous voulez vraiment aller plus loin, vous pouvez faire des [Exemples testables](https://blog.golang.org/examples). Vous trouverez de nombreux exemples dans la documentation de la bibliothèque standard.

Souvent, les exemples de code qui peuvent être trouvés en dehors de la base de code, comme un fichier README, deviennent obsolètes et incorrects par rapport au code réel parce qu'ils ne sont pas vérifiés.

Les fonctions d'exemple sont compilées chaque fois que les tests sont exécutés. Parce que de tels exemples sont validés par le compilateur Go, vous pouvez être confiant que les exemples de votre documentation reflètent toujours le comportement actuel du code.

Les fonctions d'exemple commencent par `Example` (un peu comme les fonctions de test commencent par `Test`), et résident dans les fichiers `_test.go` d'un package. Ajoutez la fonction `ExampleAdditionner` suivante au fichier `additionneurs_test.go`.

```go
func ExampleAdditionner() {
	somme := Additionner(1, 5)
	fmt.Println(somme)
	// Output: 6
}
```

(Si votre éditeur n'importe pas automatiquement les packages pour vous, l'étape de compilation échouera parce qu'il vous manquera `import "fmt"` dans `additionneurs_test.go`. Il est fortement recommandé de rechercher comment avoir ce genre d'erreurs corrigées automatiquement dans l'éditeur que vous utilisez.)

Ajouter ce code fera que l'exemple apparaisse dans votre documentation, rendant votre code encore plus accessible. Si jamais votre code change pour que l'exemple ne soit plus valide, votre build échouera.

En exécutant la suite de tests du package, nous pouvons voir que la fonction d'exemple `ExampleAdditionner` est exécutée sans arrangement supplémentaire de notre part :

```bash
$ go test -v
=== RUN   TestAdditionneurs
--- PASS: TestAdditionneurs (0.00s)
=== RUN   ExampleAdditionner
--- PASS: ExampleAdditionner (0.00s)
```

Remarquez le format spécial du commentaire, `// Output: 6`. Bien que l'exemple sera toujours compilé, ajouter ce commentaire signifie que l'exemple sera aussi exécuté. Allez-y et retirez temporairement le commentaire `// Output: 6`, puis exécutez `go test`, et vous verrez que `ExampleAdditionner` n'est plus exécuté.

Les exemples sans commentaires de sortie sont utiles pour démontrer du code qui ne peut pas s'exécuter comme des tests unitaires, comme celui qui accède au réseau, tout en garantissant que l'exemple compile au moins.

Pour voir la documentation d'exemple, jetons un coup d'œil rapide à `pkgsite`. Naviguez vers le répertoire de votre projet, puis exécutez `pkgsite -open .`, ce qui devrait ouvrir un navigateur web pour vous, pointant vers `http://localhost:8080`. À l'intérieur, vous verrez une liste de tous les packages de la bibliothèque standard de Go, plus les packages tiers que vous avez installés, sous lesquels vous devriez voir votre documentation d'exemple pour `github.com/quii/learn-go-with-tests`. Suivez ce lien, et puis regardez sous `Entiers`, puis sous `func Additionner`, puis développez `Example` et vous devriez voir l'exemple que vous avez ajouté pour `somme := Additionner(1, 5)`.

Si vous publiez votre code avec des exemples sur une URL publique, vous pouvez partager la documentation de votre code sur [pkg.go.dev](https://pkg.go.dev/). Par exemple, [ici](https://pkg.go.dev/github.com/quii/learn-go-with-tests/integers/v2) est l'API finalisée pour ce chapitre. Cette interface web vous permet de rechercher la documentation des packages de la bibliothèque standard et des packages tiers.

## Conclusion

Ce que nous avons couvert :

*   Plus de pratique du workflow TDD
*   Entiers, addition
*   Écrire une meilleure documentation pour que les utilisateurs de notre code puissent comprendre son utilisation rapidement
*   Exemples de comment utiliser notre code, qui sont vérifiés dans le cadre de nos tests
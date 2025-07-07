# Maps

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/maps)**

Dans [tableaux et slices](arrays-and-slices.md), vous avez vu comment stocker des valeurs de façon ordonnée. Maintenant, nous allons examiner une façon de stocker des éléments par une `clé` et les rechercher rapidement.

Les maps vous permettent de stocker des éléments d'une manière similaire à un dictionnaire. Vous pouvez considérer la `clé` comme le mot et la `valeur` comme la définition. Et quelle meilleure façon d'apprendre sur les Maps que de construire notre propre dictionnaire ?

Tout d'abord, en supposant que nous avons déjà quelques mots avec leurs définitions dans le dictionnaire, si nous recherchons un mot, il devrait retourner sa définition.

## Écrivez le test d'abord

Dans `dictionary_test.go`

```go
package main

import "testing"

func TestRecherche(t *testing.T) {
	dictionnaire := Dictionnaire{"test": "ceci est juste un test"}

	resultat := dictionnaire.Recherche("test")
	attendu := "ceci est juste un test"

	if resultat != attendu {
		t.Errorf("resultat '%s' attendu '%s'", resultat, attendu)
	}
}
```

Déclaration du type la plus simple ici, un `map[string]string`. Les maps vous permettent de stocker des éléments de façon similaire à un dictionnaire, où vous pouvez faire des recherches basées sur une `clé` pour obtenir une `valeur`.

Les clés et les valeurs peuvent être de n'importe quel type. Vous pourriez par exemple faire un `map[int]string` si vous vouliez rechercher des chaînes par entier ou `map[string]int` si vous vouliez rechercher des entiers par chaîne.

Vous pourriez faire `map[string]Utilisateur` pour rechercher des objets `Utilisateur` par nom, par exemple. La seule exigence est que la clé soit comparable. Consultez [le langage Go spec](https://golang.org/ref/spec#Comparison_operators) pour en savoir plus.

Ce n'est peut-être pas immédiatement évident à première vue, mais nous stockons une variable de type `Dictionnaire` (notre `map[string]string`), puis nous essayons d'appeler une méthode `Recherche` dessus.

Nous avons déjà vu ce genre de choses dans le chapitre [interfaces structurées et méthodes](structs-methods-and-interfaces.md), mais cette fois nous allons utiliser un _alias de type_. Un alias de type vous permet de créer un nouveau type à partir d'un type existant.

La syntaxe est `type MonType ExistantType`.

Pourquoi utiliser un alias de type plutôt qu'une structure ? Un map est en soi un type de référence. Qu'est-ce que cela signifie ? Si vous passez un map à une fonction ou à une méthode et que vous le modifiez, le changement sera reflété dans le appelant. D'un autre côté, un type comme `int` est un type de valeur. Lorsque vous passez une valeur à une fonction ou une méthode, Go fait une copie de cette valeur.

L'aliasing de notre map avec un nouveau type spécifique nous permet d'enrichir notre type en y attachant des méthodes.

## Essayez d'exécuter le test

```text
./dictionnaire_test.go:8:3: undefined: Dictionnaire
```

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Dans `dictionnaire.go`

```go
package main

type Dictionnaire map[string]string

func (d Dictionnaire) Recherche(mot string) string {
	return ""
}
```

Notre test échoue maintenant avec une sortie beaucoup plus claire.

```text
dictionnaire_test.go:12: resultat '' attendu 'ceci est juste un test'
```

## Écrivez assez de code pour le faire passer

```go
func (d Dictionnaire) Recherche(mot string) string {
	return d[mot]
}
```

Accéder à un map est identique à l'accès à un tableau, c'est par le biais d'une notation entre crochets.

La différence essentielle est qu'avec un tableau vous pouvez uniquement passer des entiers comme clés, tandis qu'avec les maps vous pouvez avoir d'autres types comme les chaînes.

Un autre aspect différent est que vous pouvez passer une clé qui n'existe pas. Dans le cas des tableaux, vous obtiendrez une compilation impossible si vous essayez d'écrire un élément de tableau qui est hors limites, mais dans un map vous obtiendrez simplement une valeur zéro de ce type. Par exemple, si vous avez un `map[string]int` et tentez d'obtenir une clé qui n'existe pas, vous obtiendrez 0.

Cette propriété nous permet de simplement retourner le résultat d'accès à la map directement et que le test passe.

## Refactoriser

Il n'y a pas grand-chose à refactoriser dans notre implémentation, mais nous pourrions apporter quelques ajustements à notre test. Une chose que nous pourrions faire est de créer une fonction d'aide `assertChainesEgales` pour faciliter l'écriture de tests futurs.

```go
func TestRecherche(t *testing.T) {
	dictionnaire := Dictionnaire{"test": "ceci est juste un test"}

	resultat := dictionnaire.Recherche("test")
	attendu := "ceci est juste un test"

	assertChainesEgales(t, resultat, attendu)
}

func assertChainesEgales(t testing.TB, recu, attendu string) {
	t.Helper()

	if recu != attendu {
		t.Errorf("recu '%s' attendu '%s'", recu, attendu)
	}
}
```

Nous avons maintenant une méthode `Recherche` très simple. Cependant, nous ne savons pas vraiment ce qui arrive lorsque l'on recherche une clé qui n'existe pas. Nous allons écrire un nouveau test pour cela.

## Écrivez le test d'abord

```go
func TestRecherche(t *testing.T) {
	dictionnaire := Dictionnaire{"test": "ceci est juste un test"}

	t.Run("mot connu", func(t *testing.T) {
		resultat, _ := dictionnaire.Recherche("test")
		attendu := "ceci est juste un test"

		assertChainesEgales(t, resultat, attendu)
	})

	t.Run("mot inconnu", func(t *testing.T) {
		_, resultat := dictionnaire.Recherche("inconnu")
		attendu := "impossible de trouver le mot que vous recherchez"

		if resultat == nil {
			t.Fatal("attendu une erreur")
		}

		assertChainesEgales(t, resultat.Error(), attendu)
	})
}
```

L'utilisation de sous-tests nous aide à construire un test complet pour notre fonction `Recherche`.

Dans le sous-test `mot connu`, nous continuons à vérifier que l'on peut récupérer une définition pour un mot dans le dictionnaire.

Cependant, dans le sous-test `mot inconnu`, nous vérifions deux choses :
1. `Recherche` retourne une erreur
2. L'erreur contient un message indiquant pourquoi la recherche a échoué

En adaptant nos tests, nous avons également modifié notre fonction `Recherche` pour retourner une deuxième valeur, une `erreur`. En Go, la façon idiomatique de gérer cette situation est de retourner la valeur _zéro_ du type (une chaîne vide dans ce cas) et une `erreur` avec un message descriptif.

Cette façon permet à l'appelant de vérifier si une erreur s'est produite et de décider quoi faire à partir de là. Dans notre cas, nous voulons renvoyer un dictionnaire sans aucune sortie d'erreur si tout va bien, ou la valeur zéro et une erreur si la clé n'est pas présente dans le dictionnaire.

## Essayez d'exécuter le test

```text
./dictionnaire_test.go:15:23: assignment mismatch: 2 variables but 1 values
./dictionnaire_test.go:19:24: assignment mismatch: 2 variables but 1 values
```

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

```go
func (d Dictionnaire) Recherche(mot string) (string, error) {
	return d[mot], nil
}
```

C'est la première fois que nous retournons plusieurs valeurs dans notre code. Nous pouvons le faire simplement en ajoutant des parenthèses autour des valeurs de retour.

`(valeur1, valeur2)`

Maintenant, nous devons créer une erreur à retourner lorsque la clé n'est pas trouvée. Nous utiliserons un moyen courant en Go pour créer des erreurs personnalisées en utilisant `errors.New`.

## Écrivez assez de code pour le faire passer

```go
func (d Dictionnaire) Recherche(mot string) (string, error) {
	definition, ok := d[mot]
	if !ok {
		return "", errors.New("impossible de trouver le mot que vous recherchez")
	}

	return definition, nil
}
```

Pour améliorer notre gestion d'erreur, nous utilisons une fonctionnalité intéressante de Go en prenant une "deuxième" valeur de retour lorsque nous accédons au map.

La deuxième valeur est un booléen qui indique si la clé a été trouvée ou non.

Cette opération est à peu près la même que

```go
if d[mot] != "" {
    // do something
}
```

Cependant, ce qui est problématique avec cette approche, c'est que si la valeur (dans ce cas, la définition) était vide, nous ne saurions pas si c'était vraiment stocké comme vide ou si la clé n'existait pas dans le map.

En utilisant la syntaxe `value, ok := map[key]`, nous obtenons la valeur (qui sera la valeur zéro si la clé n'est pas présente) et un booléen qui nous indique si la clé a été trouvée.

Cela nous permet de distinguer entre une clé qui n'existe pas (ok sera faux) et une clé qui est présente avec une valeur vide (ok sera vrai).

## Refactoriser

Notre code semble bon, mais nous pourrions améliorer la façon dont nous gérons les erreurs. Il est généralement bon de créer des erreurs constantes que vous pouvez utiliser pour comparer plus facilement avec `==` qu'avec une chaîne comme `err.Error()`.

```go
var ErrMotInexistant = errors.New("impossible de trouver le mot que vous recherchez")

func (d Dictionnaire) Recherche(mot string) (string, error) {
	definition, ok := d[mot]
	if !ok {
		return "", ErrMotInexistant
	}

	return definition, nil
}
```

Et maintenant, nous pouvons refactoriser notre test pour utiliser cette constante d'erreur.

```go
t.Run("mot inconnu", func(t *testing.T) {
	_, err := dictionnaire.Recherche("inconnu")

	assertErreur(t, err, ErrMotInexistant)
})

func assertErreur(t testing.TB, recu, attendu error) {
	t.Helper()

	if recu != attendu {
		t.Errorf("recu erreur '%s' attendu '%s'", recu, attendu)
	}
}
```

Nous avons créé une fonction d'aide `assertErreur` pour que nos tests restent légers.

## Écrivez le test d'abord

Nous avons un excellent moyen de rechercher dans le dictionnaire. Il est maintenant temps de donner à notre dictionnaire la possibilité d'ajouter de nouveaux mots.

```go
func TestAjouter(t *testing.T) {
	dictionnaire := Dictionnaire{}
	mot := "test"
	definition := "ceci est juste un test"

	dictionnaire.Ajouter(mot, definition)

	attendu := "ceci est juste un test"
	resultat, err := dictionnaire.Recherche(mot)
	if err != nil {
		t.Fatal("ne devrait pas obtenir d'erreur:", err)
	}

	if attendu != resultat {
		t.Errorf("resultat '%s' attendu '%s'", resultat, attendu)
	}
}
```

Notre test crée un dictionnaire vide et s'assure qu'un mot est ajouté.

## Essayez d'exécuter le test

```
./dictionnaire_test.go:31:13: dictionnaire.Ajouter undefined (type Dictionnaire has no field or method Ajouter)
```

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Dans `dictionnaire.go`

```go
func (d Dictionnaire) Ajouter(mot, definition string) {

}
```

Notre test échoue parce que nous ne changeons pas les données.

```
dictionnaire_test.go:40: ne devrait pas obtenir d'erreur: impossible de trouver le mot que vous recherchez
```

## Écrivez assez de code pour le faire passer

```go
func (d Dictionnaire) Ajouter(mot, definition string) {
	d[mot] = definition
}
```

Ajouter à un map est presque comme accéder à un élément, vous utilisez simplement `=` pour affecter une valeur à la clé.

## Refactoriser

Notre fonction `Ajouter` est très simple et notre test passe. Mais notre test utilise des vérifications multiples qui rendent le test un peu verbeux. Nous pouvons le simplifier en créant une nouvelle fonction d'aide `assertDefinition`.

```go
func TestAjouter(t *testing.T) {
	dictionnaire := Dictionnaire{}
	mot := "test"
	definition := "ceci est juste un test"
	dictionnaire.Ajouter(mot, definition)

	assertDefinition(t, dictionnaire, mot, definition)
}

func assertDefinition(t testing.TB, dictionnaire Dictionnaire, mot, definition string) {
	t.Helper()

	resultat, err := dictionnaire.Recherche(mot)
	if err != nil {
		t.Fatal("ne devrait pas obtenir d'erreur:", err)
	}

	if definition != resultat {
		t.Errorf("resultat '%s' attendu '%s'", resultat, definition)
	}
}
```

Notre test est maintenant plus compact et la méthode `Ajouter` est simple. Cependant, qu'advient-il si on redéfinit un mot ? Notre fonction actuelle permettrait à un utilisateur d'écraser une définition. Cette fonctionnalité peut être acceptable dans certains cas, mais pour ce cas, nous allons dire qu'un mot n'est ajouté que s'il n'existe pas déjà.

## Écrivez le test d'abord

```go
func TestAjouter(t *testing.T) {
	t.Run("nouveau mot", func(t *testing.T) {
		dictionnaire := Dictionnaire{}
		mot := "test"
		definition := "ceci est juste un test"

		err := dictionnaire.Ajouter(mot, definition)

		assertErreur(t, err, nil)
		assertDefinition(t, dictionnaire, mot, definition)
	})

	t.Run("mot existant", func(t *testing.T) {
		mot := "test"
		definition := "ceci est juste un test"
		dictionnaire := Dictionnaire{mot: definition}
		err := dictionnaire.Ajouter(mot, "nouveau test")

		assertErreur(t, err, ErrMotExistant)
		assertDefinition(t, dictionnaire, mot, definition)
	})
}
```

Pour ce test, nous avons modifié notre fonction `Ajouter` pour renvoyer une erreur, qui est la valeur `nil` (nil étant l'équivalent de null en Go) pour un scénario réussi.

Nous avons également créé un dictionnaire avec un mot afin de tester ce qui se passe si nous essayons d'ajouter un mot qui existe déjà.

## Essayez d'exécuter le test

```
./dictionnaire_test.go:40:17: dictionnaire.Ajouter(mot, definition) used as value
./dictionnaire_test.go:46:23: undefined: ErrMotExistant
./dictionnaire_test.go:54:22: dictionnaire.Ajouter(mot, "nouveau test") used as value
```

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Nous avons besoin de définir notre nouvelle erreur constante et modifier notre fonction `Ajouter` pour retourner des erreurs.

```go
var (
	ErrMotInexistant = errors.New("impossible de trouver le mot que vous recherchez")
	ErrMotExistant = errors.New("impossible d'ajouter le mot car il existe déjà")
)

func (d Dictionnaire) Ajouter(mot, definition string) error {
	d[mot] = definition
	return nil
}
```

Maintenant, nous avons besoin de vérifier si le mot existe déjà.

## Écrivez assez de code pour le faire passer

```go
func (d Dictionnaire) Ajouter(mot, definition string) error {
	_, existe := d[mot]
	if existe {
		return ErrMotExistant
	}

	d[mot] = definition
	return nil
}
```

Ici, nous utilisons à nouveau la deuxième valeur de retour de la recherche de map pour vérifier si le mot existe déjà avant de l'ajouter.

## Refactoriser

Notre implémentation est bonne et les tests passent. Mais il y a une petite optimisation que nous pouvons faire. Nous avons déjà une fonction `Recherche` qui renvoie une erreur si le mot existe. Nous pouvons l'utiliser dans notre fonction `Ajouter` pour vérifier si le mot existe ou non.

```go
func (d Dictionnaire) Ajouter(mot, definition string) error {
	_, err := d.Recherche(mot)

	switch err {
	case ErrMotInexistant:
		d[mot] = definition
	case nil:
		return ErrMotExistant
	default:
		return err
	}

	return nil
}
```

Ici, nous sommes utilisant une instruction `switch` pour gérer les résultats potentiels de notre appel à `Recherche`. Remarquez que nous avons aussi géré le cas où `Recherche` pourrait retourner une erreur différente. Bien que nous ne prévoyons pas d'autres erreurs de `Recherche` actuellement, en gérant ce cas, nous rendons notre code plus robuste aux évolutions futures.

## Écrivez le test d'abord

Maintenant, il est temps d'ajouter une fonctionnalité de mise à jour pour notre dictionnaire. Par "mise à jour", nous voulons dire changer la définition d'un mot.

```go
func TestMettreAJour(t *testing.T) {
	mot := "test"
	definition := "ceci est juste un test"
	dictionnaire := Dictionnaire{mot: definition}
	nouvelleDefinition := "nouvelle définition"

	dictionnaire.MettreAJour(mot, nouvelleDefinition)

	assertDefinition(t, dictionnaire, mot, nouvelleDefinition)
}
```

La fonction `MettreAJour` est très similaire à `Ajouter`, à part que cette fois nous attendons que la fonctionnalité remplace l'ancienne définition d'un mot par une nouvelle.

## Essayez d'exécuter le test

```
./dictionnaire_test.go:68:13: dictionnaire.MettreAJour undefined (type Dictionnaire has no field or method MettreAJour)
```

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Nous ajoutons simplement une nouvelle fonction `MettreAJour` à notre fichier `dictionnaire.go`.

```go
func (d Dictionnaire) MettreAJour(mot, definition string) {

}
```

Maintenant, nous voyons notre test échouer :

```
dictionnaire_test.go:67: resultat 'ceci est juste un test' attendu 'nouvelle définition'
```

## Écrivez assez de code pour le faire passer

```go
func (d Dictionnaire) MettreAJour(mot, definition string) {
	d[mot] = definition
}
```

Notre test passe, mais l'implémentation est la même que notre fonction `Ajouter`. Ce n'est pas idéal, car nous savons que la fonction `Ajouter` échouera si le mot existe déjà, ce qui signifie que nous n'utilisons pas correctement notre fonctionnalité `MettreAJour`.

## Refactoriser

Notre fonction `MettreAJour` fonctionne, mais la façon dont nous l'avons implémentée n'est pas idéale. Nous devrions vérifier si le mot existe d'abord, et si ce n'est pas le cas, retourner une erreur.

```go
func TestMettreAJour(t *testing.T) {
	t.Run("mot existant", func(t *testing.T) {
		mot := "test"
		definition := "ceci est juste un test"
		nouvelleDef := "nouvelle définition"
		dictionnaire := Dictionnaire{mot: definition}

		err := dictionnaire.MettreAJour(mot, nouvelleDef)

		assertErreur(t, err, nil)
		assertDefinition(t, dictionnaire, mot, nouvelleDef)
	})

	t.Run("nouveau mot", func(t *testing.T) {
		mot := "nouveau"
		definition := "ceci est juste un test"
		dictionnaire := Dictionnaire{}

		err := dictionnaire.MettreAJour(mot, definition)

		assertErreur(t, err, ErrMotInexistant)
	})
}
```

Nous avons maintenant deux sous-tests, l'un qui vérifie que nous pouvons mettre à jour un mot existant, et l'autre qui vérifie que `MettreAJour` renvoie une erreur s'il est appelé avec un mot qui n'existe pas dans le dictionnaire.

## Essayez d'exécuter le test

```
./dictionnaire_test.go:75:17: dictionnaire.MettreAJour(mot, nouvelleDef) used as value
./dictionnaire_test.go:87:23: dictionnaire.MettreAJour(mot, definition) used as value
```

Nous recevons une erreur de compilation car notre fonction `MettreAJour` ne retourne pas de valeur. Nous devons la mettre à jour pour qu'elle le fasse.

```go
func (d Dictionnaire) MettreAJour(mot, definition string) error {
	d[mot] = definition
	return nil
}
```

Maintenant, il s'exécute, mais nous avons un échec de test :

```
dictionnaire_test.go:88: recu erreur '<nil>' attendu 'impossible de trouver le mot que vous recherchez'
```

## Écrivez assez de code pour le faire passer

```go
func (d Dictionnaire) MettreAJour(mot, definition string) error {
	_, err := d.Recherche(mot)
	switch err {
	case ErrMotInexistant:
		return ErrMotInexistant
	case nil:
		d[mot] = definition
	default:
		return err
	}

	return nil
}
```

Nous avons maintenant une implémentation correcte de `MettreAJour`. Encore une fois, nous avons utilisé une instruction `switch` pour gérer les cas où `Recherche` pourrait retourner différents types d'erreurs.

## Écrivez le test d'abord

Notre dictionnaire est presque complet. Nous avons `Recherche`, `Ajouter` et `MettreAJour`. Il ne reste plus qu'à implémenter `Supprimer`.

```go
func TestSupprimer(t *testing.T) {
	mot := "test"
	dictionnaire := Dictionnaire{mot: "définition de test"}

	dictionnaire.Supprimer(mot)

	_, err := dictionnaire.Recherche(mot)
	if err != ErrMotInexistant {
		t.Errorf("attendu '%s' à être supprimé", mot)
	}
}
```

Notre test vérifie que la clé a été supprimée en s'assurant que `Recherche` renvoie notre erreur pour un mot inexistant.

## Essayez d'exécuter le test

```
./dictionnaire_test.go:95:13: dictionnaire.Supprimer undefined (type Dictionnaire has no field or method Supprimer)
```

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

```go
func (d Dictionnaire) Supprimer(mot string) {

}
```

Le test échoue maintenant parce que le mot n'a pas été supprimé :

```
dictionnaire_test.go:98: attendu 'test' à être supprimé
```

## Écrivez assez de code pour le faire passer

```go
func (d Dictionnaire) Supprimer(mot string) {
	delete(d, mot)
}
```

Go a une fonction intégrée `delete` qui fonctionne sur les maps. Elle prend un map et une clé en entrée, et supprime l'élément correspondant.

## Conclusion

Dans ce chapitre, nous avons abordé beaucoup de concepts importants liés aux maps en Go.

Récapitulons ce que nous avons appris :

1. Création d'un type basé sur un map avec un alias de type
2. Comment travailler avec maps, y compris l'accès, l'ajout, la mise à jour et la suppression d'éléments
3. Comment gérer les erreurs lors de l'interaction avec des maps
4. Comment utiliser les méthodes avec notre type d'alias pour créer un type plus riche
5. Comment utiliser des constantes pour améliorer la gestion des erreurs

Nos fonctions finales sont :

```go
func (d Dictionnaire) Recherche(mot string) (string, error)
func (d Dictionnaire) Ajouter(mot, definition string) error
func (d Dictionnaire) MettreAJour(mot, definition string) error
func (d Dictionnaire) Supprimer(mot string)
```

Maintenant, notre `Dictionnaire` est complètement fonctionnel et offre des méthodes idiomatiques qui permettent d'interagir avec le dictionnaire de manière claire et sans ambiguïté.
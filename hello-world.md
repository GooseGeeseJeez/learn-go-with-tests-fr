# Bonjour, Monde

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/hello-world)**

Traditionnellement, votre premier programme dans un nouveau langage est le [Bonjour, Monde](https://fr.wikipedia.org/wiki/Hello_world).

- Créez un dossier où vous le désirez
- Mettez un nouveau fichier dedans appelé `bonjour.go` et mettez le code suivant à l'intérieur

```go
package main

import "fmt"

func main() {
	fmt.Println("Bonjour, monde")
}
```

Pour l'exécuter, tapez `go run bonjour.go`.

## Comment ça marche

Quand vous écrivez un programme en Go, vous aurez un package `main` défini avec une fonction `main` à l'intérieur. Les packages sont des moyens de regrouper du code Go connexe ensemble.

Le mot-clé `func` définit une fonction avec un nom et un corps.

Avec `import "fmt"` nous importons un package qui contient la fonction `Println` que nous utilisons pour imprimer.

## Comment tester

Comment teste-t-on ceci ? Il est bon de séparer votre code "domaine" du monde extérieur \(effets de bord\, ou plus communément *side effects*). Le `fmt.Println` est un effet de bord \(imprimer sur stdout\), et la chaîne que nous envoyons est notre domaine.

Alors séparons ces deux éléments pour que ce soit plus facile à tester

```go
package main

import "fmt"

func Bonjour() string {
	return "Bonjour, monde"
}

func main() {
	fmt.Println(Bonjour())
}
```

Nous avons créé une nouvelle fonction avec `func`, mais cette fois, nous avons ajouté un autre mot-clé, `string`, à la définition. Cela signifie que cette fonction retourne une `string`.

Maintenant créez un nouveau fichier appelé `bonjour_test.go` où nous allons écrire un test pour notre fonction `Bonjour`

```go
package main

import "testing"

func TestBonjour(t *testing.T) {
	resultat := Bonjour()
	attendu := "Bonjour, monde"

	if resultat != attendu {
		t.Errorf("reçu %q attendu %q", resultat, attendu)
	}
}
```

## Des Modules en Go ?

L'étape suivante est d'exécuter les tests. Entrez `go test` dans votre terminal. Si les tests passent, alors vous utilisez probablement une version antérieure de Go. Cependant, si vous utilisez Go 1.16 ou plus récent, les tests ne s'exécuteront probablement pas. À la place, vous verrez un message d'erreur comme celui-ci dans le terminal :

```shell
$ go test
go: cannot find main module; see 'go help modules'
```

Quel est le problème ? En un mot, les [modules](https://blog.golang.org/go116-module-changes). Heureusement, le problème est facile à corriger. Entrez `go mod init exemple.com/bonjour` dans votre terminal. Cela créera un nouveau fichier avec le contenu suivant :

```
module exemple.com/bonjour

go 1.16
```

Ce fichier dit aux outils `go` des informations essentielles sur votre code. Si vous prévoyez de distribuer votre application, vous incluriez où le code était disponible pour téléchargement ainsi que des informations sur les dépendances. Le nom du module, exemple\.com\/bonjour, se réfère habituellement à une URL où le module peut être trouvé et téléchargé. Pour la compatibilité avec les outils que nous utiliserons bientôt, assurez-vous que le nom de votre module a un point quelque part, comme le point dans .com de exemple\.com/bonjour. Pour l'instant, votre fichier de module est minimal, et vous pouvez le laisser comme ça. Pour lire plus sur les modules, [vous pouvez consulter la référence dans la documentation Golang](https://golang.org/doc/modules/gomod-ref). Nous pouvons revenir aux tests et à l'apprentissage de Go maintenant puisque les tests devraient s'exécuter, même sur Go 1.16.

Dans les futurs chapitres, vous devrez exécuter `go mod init NOMQUELCONQUE` dans chaque nouveau dossier avant d'exécuter des commandes comme `go test` ou `go build`.

## Retour aux tests

Exécutez `go test` dans votre terminal. Ça devrait avoir réussi ! Juste pour vérifier, essayez de délibérément casser le test en changeant la chaîne `attendu`.

Notez comment vous n'avez pas eu à choisir entre plusieurs frameworks de test puis comprendre comment les installer. Tout ce dont vous avez besoin est intégré dans le langage, et la syntaxe est la même que le reste du code que vous écrirez.

### Écrire des tests

Écrire un test, c'est juste comme écrire une fonction, avec quelques règles :

* Il doit être dans un fichier avec un nom comme `xxx_test.go`
* La fonction de test doit commencer par le mot `Test`
* La fonction de test ne prend qu'un seul argument `t *testing.T`
* Pour utiliser le type `*testing.T`, vous devez `import "testing"`, comme nous l'avons fait avec `fmt` dans l'autre fichier

Pour l'instant, il suffit de savoir que votre `t` de type `*testing.T` est votre "crochet" dans le framework de test pour que vous puissiez faire des choses comme `t.Fail()` quand vous voulez faire échouer.

Nous avons couvert quelques nouveaux sujets :

#### `if`
Les instructions if en Go sont très similaires à d'autres langages de programmation.

#### Déclarer des variables

Nous déclarons quelques variables avec la syntaxe `nomVar := valeur`, ce qui nous permet de réutiliser certaines valeurs dans notre test pour la lisibilité.

#### `t.Errorf`

Nous appelons la _méthode_ `Errorf` sur notre `t`, qui va imprimer un message et faire échouer le test. Le `f` signifie format, ce qui nous permet de construire une chaîne avec des valeurs insérées dans les valeurs d'espace réservé `%q`. Quand vous faites échouer le test, il devrait être clair comment ça fonctionne.

Vous pouvez lire plus sur les chaînes d'espace réservé dans la [documentation fmt](https://pkg.go.dev/fmt#hdr-Printing). Pour les tests, `%q` est très utile car il entoure vos valeurs de guillemets doubles.

Nous explorerons plus tard la différence entre les méthodes et les fonctions.

### Documentation de Go

Une autre fonctionnalité de qualité de vie de Go est la documentation. Nous venons de voir la documentation pour le package fmt sur le site officiel de visualisation de packages, et Go fournit également des moyens pour accéder rapidement à la documentation hors ligne.

Go a un outil intégré, doc, qui vous permet d'examiner tout package installé sur votre système, ou le module sur lequel vous travaillez actuellement. Pour voir cette même documentation pour les verbes d'impression :

```
$ go doc fmt
package fmt // import "fmt"

Package fmt implements formatted I/O with functions analogous to C's printf and
scanf. The format 'verbs' are derived from C's but are simpler.

# Printing

The verbs:

General:

    %v	the value in a default format
    	when printing structs, the plus flag (%+v) adds field names
    %#v	a Go-syntax representation of the value
    %T	a Go-syntax representation of the type of the value
    %%	a literal percent sign; consumes no value
...
```

Le second outil de Go pour voir la documentation est la commande pkgsite, qui alimente le site officiel de visualisation de packages de Go. Vous pouvez installer pkgsite avec `go install golang.org/x/pkgsite/cmd/pkgsite@latest`, puis l'exécuter avec `pkgsite -open .`. La commande install de Go téléchargera les fichiers sources de ce dépôt et les construira en un binaire exécutable. Pour une installation par défaut de Go, cet exécutable sera dans `$HOME/go/bin` pour Linux et macOS, et `%USERPROFILE%\go\bin` pour Windows. Si vous n'avez pas encore ajouté ces chemins à votre variable $PATH, vous voudrez peut-être le faire pour rendre l'exécution des commandes installées par go plus facile.

La grande majorité de la bibliothèque standard a une excellente documentation avec des exemples. Naviguer vers [http://localhost:8080/testing](http://localhost:8080/testing) serait intéressant pour voir ce qui est disponible pour vous.

### Bonjour, TOI

Maintenant que nous avons un test, nous pouvons itérer sur notre logiciel en toute sécurité.

Dans le dernier exemple, nous avons écrit le test _après_ que le code ait été écrit pour que vous puissiez avoir un exemple de comment écrire un test et déclarer une fonction. À partir de maintenant, nous écrirons les tests en premier.

Notre prochaine exigence est de nous permettre de spécifier le destinataire du salut.

Commençons par capturer ceci dans un test. C'est du développement piloté par les tests de base et nous permet de nous assurer que notre test teste _réellement_ ce que nous voulons. Quand vous écrivez des tests rétrospectivement, il y a le risque que votre test puisse continuer à passer même si le code ne fonctionne pas comme prévu.

```go
package main

import "testing"

func TestBonjour(t *testing.T) {
	resultat := Bonjour("Chris")
	attendu := "Bonjour, Chris"

	if resultat != attendu {
		t.Errorf("reçu %q attendu %q", resultat, attendu)
	}
}
```

Maintenant exécutez `go test`, vous devriez avoir une erreur de compilation

```text
./bonjour_test.go:6:25: too many arguments in call to Bonjour
    have (string)
    want ()
```

Quand vous utilisez un langage statiquement typé comme Go, il est important d'_écouter le compilateur_. Le compilateur comprend comment votre code devrait s'assembler et fonctionner pour que vous n'ayez pas à le faire.

Dans ce cas, le compilateur vous dit ce que vous devez faire pour continuer. Nous devons changer notre fonction `Bonjour` pour accepter un argument.

Éditez la fonction `Bonjour` pour accepter un argument de type string

```go
func Bonjour(nom string) string {
	return "Bonjour, monde"
}
```

Si vous essayez d'exécuter vos tests maintenant, votre `main.go` ne se compilera pas parce que vous n'envoyez pas d'argument. Envoyez "monde" pour qu'il se compile.

```go
func main() {
	fmt.Println(Bonjour("monde"))
}
```

Maintenant quand vous exécutez vos tests, vous devriez voir quelque chose comme

```text
bonjour_test.go:10: reçu "Bonjour, monde" attendu "Bonjour, Chris"
```

Nous avons finalement un test qui compile mais qui ne passe pas. Maintenant écrivons juste assez de code pour le faire passer. Dans la discipline TDD, nous devrions écrire le _minimum de code pour faire passer le test_.

```go
func Bonjour(nom string) string {
	return "Bonjour, " + nom
}
```

Quand vous exécutez les tests, ils devraient maintenant passer. Normalement, dans le cadre du cycle TDD, nous devrions maintenant _refactoriser_.

### Une note sur le *source control*

À ce stade, si vous utilisez le *source control* \(ce que vous devriez !\), je committrais le code tel qu'il est. Nous avons un logiciel qui fonctionne soutenu par un test.

Je ne committrais pas le code qui ne fonctionne pas, au cas où je voudrais revenir en arrière plus tard.

Il n'y a pas grand chose à refactoriser ici, mais nous pouvons introduire une autre fonctionnalité du langage Go : les _constantes_.

### Constantes

Les constantes sont définies comme suit

```go
const anglais = "Anglais"
const francais = "Français"
const espagnol = "Espagnol"

const prefixeSalutAnglais = "Hello, "
const prefixeSalutFrancais = "Bonjour, "
const prefixeSalutEspagnol = "Hola, "
```

Nous pouvons maintenant refactoriser notre fonction `Bonjour`

```go
func Bonjour(nom string) string {
	return prefixeSalutFrancais + nom
}
```

Après refactorisation, relancez vos tests pour vous assurer que vous n'avez rien cassé.

Les constantes devraient améliorer les performances de votre application car cela évite de créer l'instance de chaîne `"Bonjour, "` à chaque fois que la fonction `Bonjour` est appelée.

Pour être clair, l'impact sur les performances pour cet exemple serait négligeable, mais il vaut la peine de penser à créer des constantes pour capturer la signification des valeurs et parfois pour aider les performances.

## Bonjour, monde... encore

L'exigence suivante est quand notre fonction est appelée avec une chaîne vide, elle devrait par défaut afficher "Bonjour, Monde" au lieu de "Bonjour, ".

Commençons par écrire un nouveau test qui échoue

```go
func TestBonjour(t *testing.T) {
	t.Run("dire bonjour aux personnes", func(t *testing.T) {
		resultat := Bonjour("Chris")
		attendu := "Bonjour, Chris"

		if resultat != attendu {
			t.Errorf("reçu %q attendu %q", resultat, attendu)
		}
	})
	t.Run("dire 'Bonjour, Monde' quand une chaîne vide est fournie", func(t *testing.T) {
		resultat := Bonjour("")
		attendu := "Bonjour, Monde"

		if resultat != attendu {
			t.Errorf("reçu %q attendu %q", resultat, attendu)
		}
	})
}
```

Ici nous introduisons un autre outil dans notre arsenal de test, les sous-tests. Parfois il est utile de regrouper des tests autour d'une "chose" et ensuite d'avoir des sous-tests décrivant différents scénarios.

Un avantage de cette approche est que vous pouvez configurer du code partagé qui peut être utilisé dans les autres tests.

Il y a une syntaxe répétée quand nous vérifions si le message est ce que nous attendons.

La refactorisation n'est pas _seulement_ pour le code de production !

Il est important que vos tests soient clairs à comprendre car ils agissent comme de la documentation de votre code.

Nous pouvons et devons refactoriser nos tests.

```go
func TestBonjour(t *testing.T) {
	verifierMessageCorrect := func(t testing.TB, resultat, attendu string) {
		t.Helper()
		if resultat != attendu {
			t.Errorf("reçu %q attendu %q", resultat, attendu)
		}
	}

	t.Run("dire bonjour aux personnes", func(t *testing.T) {
		resultat := Bonjour("Chris")
		attendu := "Bonjour, Chris"
		verifierMessageCorrect(t, resultat, attendu)
	})

	t.Run("dire 'Bonjour, Monde' quand une chaîne vide est fournie", func(t *testing.T) {
		resultat := Bonjour("")
		attendu := "Bonjour, Monde"
		verifierMessageCorrect(t, resultat, attendu)
	})
}
```

Que faisons-nous ici ?

Nous refactorisons notre assertion en une fonction. Cela réduit la duplication et améliore la lisibilité de nos tests. Nous devons passer `t *testing.T` pour que nous puissions dire au runtime de test qu'il a échoué quand nous devons le faire.

`t.Helper()` est nécessaire pour dire à la suite de tests que cette méthode est une fonction helper. En faisant cela, quand elle échoue, le numéro de ligne rapporté sera dans notre _appel de fonction_ plutôt que dans notre fonction helper de test. Cela vous aidera à localiser les problèmes plus facilement. Si vous ne comprenez toujours pas, commentez cette ligne, essayez de faire échouer le test et observez les erreurs rapportées. Les commentaires en Go sont une très bonne façon d'ajouter de l'information à votre code, ou dans ce cas, de dire rapidement au compilateur d'ignorer une ligne. Vous pouvez commenter la ligne t.Helper() en ajoutant deux slashs // à son début. Vous devriez voir que cette ligne est devenue grise ou a changé de couleur pour indiquer qu'elle est désormais commentée.

Lorsque vous avez plus d'un argument d'un même type (deux strings dans notre cas), n'écrivez pas `(got string, want string)` mais plutôt la version abrégée `(got, want string)`.

### Retour au *source control*

Maintenant que le code semble bon, je commiterais pour seulement envoyer en ligne la "jolie" version de notre code avec son test.

### Discipline

Revoyons le cycle jusqu'à présent

* Écrire un test
* Faire compiler le compilateur
* Exécuter le test, voir qu'il échoue et vérifier que le message d'erreur a du sens
* Écrire juste assez de code pour faire passer le test
* Refactoriser

En surface, cela peut sembler fastidieux, mais suivre cette boucle d'actions est important.

Non seulement cela garantit que vous avez _des tests pertinents_, mais cela garantit que vous _concevez un bon logiciel_ en refactorisatant avec la sécurité des tests.

Voir le test échouer est une étape importante car cela vous permet également de voir à quoi ressemble le message d'erreur. En tant que développeur, il peut être très difficile de travailler avec une base de code quand les tests qui échouent ne donnent pas une idée claire de ce qui ne va pas.

En vous assurant que vos tests sont _rapides_ et en configurant vos outils pour que l'exécution des tests soit simple, vous pouvez entrer dans un état de flow lors de l'écriture de votre code.

En n'écrivant pas de tests, vous vous engagez à vérifier manuellement votre code en exécutant votre logiciel, ce qui brise votre *état de flow*. Cela ne vous fera pas gagner du temps, surtout sur le long terme.

## Continuez ! Plus d'exigences

Bon sang, nous avons une autre exigence. Nous devons maintenant supporter un second paramètre, spécifiant la langue du salut. Si une langue que nous ne reconnaissons pas est passée, faites juste par défaut à l'anglais.

Nous devons être confiants que nous pouvons utiliser le TDD (*Test-Driven Development*) pour résoudre ce problème facilement !

Écrivons un test pour un utilisateur exigeant de l'espagnol. Ajoutez ceci à votre suite de tests.

```go
t.Run("en espagnol", func(t *testing.T) {
	resultat := Bonjour("Elodie", "Espagnol")
	attendu := "Hola, Elodie"
	verifierMessageCorrect(t, resultat, attendu)
})
```

Pas de triche ! Lancez le test pour vous assurer qu'il compile et échoue correctement \(`./bonjour_test.go:27:19: too many arguments in call to Bonjour`\), puis utilisez le compilateur pour vous guider.

```go
func Bonjour(nom string, langue string) string {
	if nom == "" {
		nom = "Monde"
	}
	return prefixeSalutFrancais + nom
}
```

Quand vous essayez d'exécuter le test, il se plaindra que vous ne passez pas assez d'arguments à `Bonjour` dans vos autres tests et dans `main.go`. Utilisez le compilateur pour vous guider sur ce qu'il faut faire.

Une fois que vous avez terminé, tous vos tests devraient compiler et passer, sauf le nouveau que nous venons d'ajouter :

```text
bonjour_test.go:29: reçu "Bonjour, Elodie" attendu "Hola, Elodie"
```

Nous pouvons utiliser `if` ici pour vérifier la langue

```go
func Bonjour(nom string, langue string) string {
	if nom == "" {
		nom = "Monde"
	}

	if langue == espagnol {
		return prefixeSalutEspagnol + nom
	}

	return prefixeSalutFrancais + nom
}
```

Les tests devraient maintenant passer.

Maintenant il est temps de _refactoriser_. Vous devriez voir quelques problèmes dans le code, "valeurs magiques" de chaînes, du code quelque peu répétitif. Essayez de le refactoriser vous-même, avec chaque changement s'assurer que vous relancez les tests pour vous assurer que vous n'avez rien cassé.

```go
const espagnol = "Espagnol"
const francais = "Français"
const anglais = "Anglais"

const prefixeSalutAnglais = "Hello, "
const prefixeSalutFrancais = "Bonjour, "
const prefixeSalutEspagnol = "Hola, "

func Bonjour(nom string, langue string) string {
	if nom == "" {
		nom = "Monde"
	}

	if langue == espagnol {
		return prefixeSalutEspagnol + nom
	}

	if langue == francais {
		return prefixeSalutFrancais + nom
	}

	return prefixeSalutAnglais + nom
}
```

### Anglais

* Écrivez un test en affirmant que si vous passez `"Anglais"` vous obtenez `"Bonjour, "`
* Voyez-le échouer, écrivez le code pour le faire passer

### `switch`

Quand vous avez beaucoup d'instructions `if` sur la même valeur, il est courant d'utiliser une instruction `switch` à la place. Nous pouvons utiliser `switch` pour refactoriser le code pour le rendre plus facile à lire et plus extensible si nous voulons ajouter plus de support linguistique plus tard.

```go
func Bonjour(nom string, langue string) string {
	if nom == "" {
		nom = "Monde"
	}

	prefixe := prefixeSalutAnglais

	switch langue {
	case Anglais:
		prefixe = prefixeSalutAnglais
	case espagnol:
		prefixe = prefixeSalutEspagnol
	case anglais:
		prefixe = prefixeSalutAnglais
	}

	return prefixe + nom
}
```

Écrivez un test pour vous assurer que cela fonctionne toujours. Cela devrait être facile à modifier maintenant pour ajouter plus de langues.

### un...dernier...refactoring ?

Vous pourriez argumenter que peut-être nos fonctions deviennent un peu grosses. La chose la plus simple que je puisse faire est d'extraire quelques fonctionnalités dans une autre fonction.

```go
func Bonjour(nom string, langue string) string {
	if nom == "" {
		nom = "Monde"
	}

	return prefixeSalut(langue) + nom
}

func prefixeSalut(langue string) (prefixe string) {
	switch langue {
	case espagnol:
		prefixe = prefixeSalutEspagnol
	case anglais:
		prefixe = prefixeSalutAnglais
	default:
		prefixe = prefixeSalutFrancais
	}
	return
}
```

Quelques nouveaux concepts :

* Dans notre signature de fonction, nous avons fait un _retour nommé_ `(prefixe string)`.
* Cela créera une variable appelée `prefixe` dans votre fonction.
  * Elle sera assignée à la valeur "zéro". Cela dépend du type, pour les int c'est 0 et pour les strings c'est `""`.
    * Vous pouvez retourner ce que vous avez défini en appelant simplement `return` plutôt que `return prefixe`.
  * Cela s'affichera dans la documentation de Go, donc cela peut rendre l'intention de votre code plus claire.
* `default` dans l'instruction switch sera branché si aucun des autres cas `case` ne correspond.
* Le nom de la fonction commence par une lettre minuscule. En Go, les fonctions publiques commencent par une lettre majuscule et les fonctions privées commencent par une lettre minuscule. Nous ne voulons pas que les internes de notre algorithme soient exposés au monde, donc nous rendons cette fonction privée.

## Conclusion

Qui aurait pensé que vous pourriez apprendre autant de `Bonjour, monde` ?

À ce stade, vous devriez avoir une bonne compréhension de :

### Une partie de la syntaxe de Go :

* Écrire des tests
* Déclarer des fonctions, avec des arguments et des types de retour
* `if`, `const` et `switch`
* Déclarer des variables et des constantes

### Le processus TDD et _pourquoi_ les étapes sont importantes

* _Écrire un test qui échoue et voir qu'il échoue_ pour que nous sachions que nous avons écrit un test _pertinent_ pour nos exigences et avons vu que cela produit un _message d'erreur facile à comprendre_
* Écrire la plus petite quantité de code pour le faire passer pour que nous sachions que nous avons un logiciel qui fonctionne
* _Puis_ refactoriser, soutenu par la sécurité de nos tests pour nous assurer que nous avons du code bien conçu qui est facile à travailler

Dans nos cas, nous avons traversé de `Bonjour()` vers `Bonjour("nom")`, vers `Bonjour("nom", "Français")` en petites étapes faciles à comprendre.

Bien sûr, c'est trivial par rapport aux logiciels "réels", mais les principes restent les mêmes. TDD est une compétence qui demande de la pratique pour se développer, mais en décomposant les problèmes en petits morceaux testables, vous obtiendrez des logiciels plus robustes.
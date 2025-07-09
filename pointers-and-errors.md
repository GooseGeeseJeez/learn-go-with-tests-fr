# Pointeurs et erreurs

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/pointers)**

Nous avons appris les structs dans la section précédente qui nous permettent de capturer un certain nombre de valeurs liées autour d'un concept.

À un moment donné, vous pourriez souhaiter utiliser des structs pour gérer l'état, en exposant des méthodes pour permettre aux utilisateurs de changer l'état d'une manière que vous pouvez contrôler.

**La fintech adore Go** et, euh, les bitcoins ? Alors montrons quel système bancaire incroyable nous pouvons créer.

Créons une struct `Portefeuille` qui nous permet de déposer des `Bitcoin`.

## Écrivez le test d'abord

```go
func TestPortefeuille(t *testing.T) {

	portefeuille := Portefeuille{}

	portefeuille.Deposer(10)

	solde := portefeuille.Solde()

	attendu := 10

	if solde != attendu {
		t.Errorf("solde %d, attendu %d", solde, attendu)
	}
}
```

## Essayez d'exécuter le test

`./portefeuille_test.go:7:9: undefined: Portefeuille`

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Nous avons besoin de définir un type `Portefeuille`.

```go
type Portefeuille struct {}

func (p Portefeuille) Deposer(montant int) {

}

func (p Portefeuille) Solde() int {
	return 0
}
```

Le test échoue maintenant avec un message clair

`portefeuille_test.go:13: solde 0, attendu 10`

## Écrivez assez de code pour le faire passer

Nous avons besoin d'une variable à l'intérieur de notre struct pour stocker le solde.

```go
type Portefeuille struct {
	solde int
}

func (p *Portefeuille) Deposer(montant int) {
	p.solde += montant
}

func (p Portefeuille) Solde() int {
	return p.solde
}
```

`solde += montant` est une façon concise de dire `solde = solde + montant`.

Le test échoue toujours, mais étonnamment.

```
portefeuille_test.go:13: solde 0, attendu 10
```

C'est bizarre ! Nous sommes censés avoir mis à jour le solde, mais il semblerait que nous ne l'ayons pas fait. 

En Go, lorsque vous appelez une fonction ou une méthode, les arguments sont _copiés_.

Quand `Deposer` est appelé, la struct `Portefeuille` est copiée, donc la méthode opère sur une copie des données.

Pour résoudre ce problème, nous avons besoin de la struct `Portefeuille` elle-même, pas d'une copie. Nous devons donc utiliser un **pointer** pour la méthode `Deposer`.

## Pointeurs

Un pointeur fait _référence_ à un endroit en mémoire où une valeur est stockée, plutôt que de copier la valeur elle-même.

En ajoutant un `*` avant le type, vous obtenez accès à une valeur par pointeur.

Donc maintenant, au lieu de faire une copie de la struct `Portefeuille`, `Deposer` a accès aux données originales de la struct.

Exécutez à nouveau les tests et ils devraient maintenant passer.

### Une dernière précision sur les pointeurs

En Go, lorsque vous passez une valeur comme paramètre à une fonction/méthode, cette valeur est copiée.

Quand nous appelons `func (p Portefeuille) Deposer(montant int)`, le compilateur Go copie la struct `Portefeuille` dans `p`, donc les modifications ne sont faites que sur `p` et non sur ce qui a été copié.

Si nous changeons la définition en `func (p *Portefeuille) Deposer(montant int)`, la méthode `Deposer` va fonctionner avec un pointeur vers la `Portefeuille`.

En Go, cela reste clair car vous pouvez voir qu'il s'agit d'un pointeur car nous devons préfixer avec un symbole `*`.

Mais lorsque vous appelez la méthode à l'aide du récepteur, vous n'avez pas besoin de vous soucier des détails de la façon dont elle est implémentée, que ce soit avec une valeur ou un pointeur. Vous n'avez pas besoin de dire dans votre code `&portefeuille.Deposer(10)`.

Cette dernière partie peut sembler un peu déroutante, mais rappelez-vous que lorsque vous appelez une méthode sur une variable comme `portefeuille.Deposer(10)`, l'objet `portefeuille` est toujours le receveur, il est juste copié différemment selon que la méthode a un receveur par valeur ou par pointeur.

## Refactoriser

Nous avons découvert que sans un receveur par pointeur pour la méthode `Deposer`, nous ne pouvons pas modifier l'état de la valeur.

Il convient de faire de même pour `Solde` afin de rester cohérent, même si techniquement elle n'a pas besoin d'être un pointeur car elle ne modifie pas l'état. Mais dans l'ensemble, il est plus courant de garder vos receveurs de méthode cohérents (soit tous des pointeurs, soit tous des valeurs).

```go
func (p *Portefeuille) Solde() int {
	return p.solde
}
```

Ce changement ne modifie pas le comportement, donc les tests devraient toujours passer.

Je souhaite maintenant introduire un nouveau type, `Bitcoin` qui s'ajoutera à notre travail de simulation d'un système bancaire de premier ordre.

```go
type Bitcoin int

type Portefeuille struct {
	solde Bitcoin
}

func (p *Portefeuille) Deposer(montant Bitcoin) {
	p.solde += montant
}

func (p *Portefeuille) Solde() Bitcoin {
	return p.solde
}
```

Pour utiliser notre nouveau type `Bitcoin`, nous devons mettre à jour notre test.

```go
func TestPortefeuille(t *testing.T) {

	portefeuille := Portefeuille{}

	portefeuille.Deposer(Bitcoin(10))

	solde := portefeuille.Solde()

	attendu := Bitcoin(10)

	if solde != attendu {
		t.Errorf("solde %d, attendu %d", solde, attendu)
	}
}
```

Pour pouvoir utiliser `Bitcoin` avec le code du test, nous devons convertir un entier en `Bitcoin` avec la syntaxe `Bitcoin(999)`.

Un aspect intéressant de Go est que la méthode `Stringer` du package `fmt` vous permet de définir comment votre type est imprimé lorsqu'il est utilisé avec la chaîne de format `%s` (%s pour "string").

Ajoutons cette fonctionnalité à `Bitcoin`:

```go
func (b Bitcoin) String() string {
	return fmt.Sprintf("%d BTC", b)
}
```

À mesure que notre code devient plus complexe, nous voulons que nos tests restent simples à écrire et à lire. Nous avons déjà vu comment les méthodes d'aide à l'assertion peuvent rendre les tests plus clairs.

```go
func TestPortefeuille(t *testing.T) {

	portefeuille := Portefeuille{}
	portefeuille.Deposer(Bitcoin(10))

	verifieSolde := func(t testing.TB, portefeuille Portefeuille, attendu Bitcoin) {
		t.Helper()
		solde := portefeuille.Solde()

		if solde != attendu {
			t.Errorf("solde %s, attendu %s", solde, attendu)
		}
	}

	verifieSolde(t, portefeuille, Bitcoin(10))
}
```

Maintenant, nous avons une méthode réutilisable `verifieSolde` qui utilise notre nouvelle fonction `String()` pour nos bitcoins.

Implémentons maintenant une fonction `Retirer` pour notre portefeuille.

## Écrivez le test d'abord

```go
func TestPortefeuille(t *testing.T) {

	t.Run("Deposer", func(t *testing.T) {
		portefeuille := Portefeuille{}
		portefeuille.Deposer(Bitcoin(10))
		verifieSolde(t, portefeuille, Bitcoin(10))
	})

	t.Run("Retirer", func(t *testing.T) {
		portefeuille := Portefeuille{solde: Bitcoin(20)}
		portefeuille.Retirer(Bitcoin(10))
		verifieSolde(t, portefeuille, Bitcoin(10))
	})

}
```

## Essayez d'exécuter le test

`./portefeuille_test.go:18:9: portefeuille.Retirer undefined (type Portefeuille has no field or method Retirer)`

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

```go
func (p *Portefeuille) Retirer(montant Bitcoin) {

}
```

```
portefeuille_test.go:33: solde 20 BTC, attendu 10 BTC
```

## Écrivez assez de code pour le faire passer

```go
func (p *Portefeuille) Retirer(montant Bitcoin) {
	p.solde -= montant
}
```

## Refactoriser

Nous avons maintenant une fonction `Retirer` qui fonctionne. Cependant, que se passe-t-il si un utilisateur essaie de retirer plus de bitcoins qu'il n'en a ? Il finira par avoir un solde négatif.

Nous devons ajouter une validation pour empêcher cela et retourner une erreur dans ce cas.

## Écrivez le test d'abord

```go
t.Run("Retirer avec des fonds insuffisants", func(t *testing.T) {
	soldeInitial := Bitcoin(20)
	portefeuille := Portefeuille{soldeInitial}
	erreur := portefeuille.Retirer(Bitcoin(100))

	verifieSolde(t, portefeuille, soldeInitial)

	if erreur == nil {
		t.Errorf("une erreur aurait dû être retournée")
	}
})
```

## Essayez d'exécuter le test

`./portefeuille_test.go:37:15: portefeuille.Retirer(Bitcoin(100)) used as value`

Dans Go, les fonctions peuvent retourner plusieurs valeurs. Ici, nous devons changer la signature de notre fonction `Retirer` pour renvoyer une erreur en plus de l'opération de retrait.

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

```go
func (p *Portefeuille) Retirer(montant Bitcoin) error {
	p.solde -= montant
	return nil
}
```

```
portefeuille_test.go:40: une erreur aurait dû être retournée
```

## Écrivez assez de code pour le faire passer

```go
func (p *Portefeuille) Retirer(montant Bitcoin) error {
	if montant > p.solde {
		return errors.New("oh non")
	}

	p.solde -= montant
	return nil
}
```

Le test devrait maintenant passer parce que nous vérifions si nous essayons de retirer plus que nous avons et retournons une erreur dans ce cas.

## Refactoriser

### Améliorer les messages d'erreur

Ça n'est pas génial que nous retournions juste une chaîne un peu aléatoire comme erreur. 

Nous pouvons améliorer cela en créant une erreur spécifique pour ce cas :

```go
var ErrFondsInsuffisants = errors.New("impossible de retirer: fonds insuffisants")

func (p *Portefeuille) Retirer(montant Bitcoin) error {
	if montant > p.solde {
		return ErrFondsInsuffisants
	}

	p.solde -= montant
	return nil
}
```

C'est une bonne pratique d'exposer des variables contenant des erreurs comme celle-ci pour que l'utilisateur de votre API puisse vérifier le type d'erreur qu'il a reçue.

### Vérifier la présence d'erreurs spécifiques

```go
t.Run("Retirer avec des fonds insuffisants", func(t *testing.T) {
	soldeInitial := Bitcoin(20)
	portefeuille := Portefeuille{soldeInitial}
	erreur := portefeuille.Retirer(Bitcoin(100))

	verifieSolde(t, portefeuille, soldeInitial)

	if erreur != ErrFondsInsuffisants {
		t.Errorf("erreur attendue %s, mais reçu %s", ErrFondsInsuffisants, erreur)
	}
})
```

Maintenant nous vérifions que l'erreur a la bonne valeur, ce qui est plus précis et rend notre code plus compréhensible.

### Factorisation de la fonctionnalité de test

Notre test a encore des aspects qui pourraient être améliorés. Les vérifications d'erreur sont communes, donc nous pouvons faire une fonction Helper pour cela :

```go
verifieErreur := func(t testing.TB, erreur error, attendu error) {
	t.Helper()
	if erreur != attendu {
		t.Errorf("erreur attendue %q, mais reçu %q", attendu, erreur)
	}
}
```

Et maintenant nous pouvons mettre à jour notre code pour utiliser cette fonction :

```go
t.Run("Retirer avec des fonds insuffisants", func(t *testing.T) {
	soldeInitial := Bitcoin(20)
	portefeuille := Portefeuille{soldeInitial}
	erreur := portefeuille.Retirer(Bitcoin(100))

	verifieSolde(t, portefeuille, soldeInitial)
	verifieErreur(t, erreur, ErrFondsInsuffisants)
})
```

## Conclusion

### Pointeurs

* Go passe les valeurs par copie, donc si vous utilisez une valeur (comme `Portefeuille`) plutôt qu'un pointeur (*`Portefeuille`), les méthodes travailleront sur une copie des données.
* Les pointeurs vous permettent de partager une référence à un endroit spécifique de la mémoire pour que vous puissiez partager des données.
* Lorsque vous utilisez des receveurs par pointeur, l'utilisation par les appelants ressemble à l'utilisation de valeurs, ils n'ont pas à se soucier des détails d'implémentation.

### Erreurs

* Exposez les erreurs que les utilisateurs de votre API peuvent vérifier avec des variables.
* Utilisez les types personnalisés pour ajouter un niveau supplémentaire d'informations à vos valeurs et permettre un code plus expressif.

## Refactoriser

Nous avons maintenant une fonction `Retirer` qui fonctionne. Cependant, que se passe-t-il si un utilisateur essaie de retirer plus de bitcoins qu'il n'en a ? Il finira par avoir un solde négatif.

Nous devons ajouter une validation pour empêcher cela et retourner une erreur dans ce cas.


	resultat := portefeuille.Solde()
	attendu := 10

	if resultat != attendu {
		t.Errorf("reçu %d attendu %d", resultat, attendu)
	}
}
```

Dans l'[exemple précédent](./structs-methods-and-interfaces.md) nous accédions aux champs directement avec le nom du champ, cependant dans notre _portefeuille très sécurisé_ nous ne voulons pas exposer notre état interne au reste du monde. Nous voulons contrôler l'accès via des méthodes.

## Essayez d'exécuter le test

`./portefeuille_test.go:7:17: undefined: Portefeuille`

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Le compilateur ne sait pas ce qu'est un `Portefeuille` alors disons-le lui.

```go
type Portefeuille struct{}
```

Maintenant que nous avons créé notre portefeuille, essayez d'exécuter le test à nouveau

```
./portefeuille_test.go:9:18: portefeuille.Deposer undefined (type Portefeuille has no field or method Deposer)
./portefeuille_test.go:11:23: portefeuille.Solde undefined (type Portefeuille has no field or method Solde)
```

Nous devons définir ces méthodes.

Rappelez-vous de faire seulement assez pour faire fonctionner les tests. Nous devons nous assurer que notre test échoue correctement avec un message d'erreur clair.

```go
func (p Portefeuille) Deposer(montant int) {

}

func (p Portefeuille) Solde() int {
	return 0
}
```

Si cette syntaxe vous semble inconnue, retournez lire la section sur les structs.

Les tests devraient maintenant compiler et s'exécuter

`portefeuille_test.go:15: reçu 0 attendu 10`

## Écrivez assez de code pour le faire passer

Nous aurons besoin d'une variable _solde_ dans notre struct pour stocker l'état

```go
type Portefeuille struct {
	solde int
}
```

En Go, si un symbole (variables, types, fonctions etc.) commence par un symbole en minuscule alors il est privé _en dehors du package dans lequel il est défini_.

Dans notre cas, nous voulons que nos méthodes puissent manipuler cette valeur, mais personne d'autre.

Rappelez-vous que nous pouvons accéder au champ interne `solde` dans la struct en utilisant la variable "récepteur".

```go
func (p Portefeuille) Deposer(montant int) {
	p.solde += montant
}

func (p Portefeuille) Solde() int {
	return p.solde
}
```

Avec notre carrière en fintech assurée, exécutez la suite de tests et prélassez-vous dans le test qui passe

`portefeuille_test.go:15: reçu 0 attendu 10`

### Ce n'est pas tout à fait correct

Eh bien c'est confus, notre code ressemble à ce qu'il devrait fonctionner.
Nous ajoutons le nouveau montant à notre solde et puis la méthode solde devrait retourner l'état actuel de celui-ci.

En Go, **quand vous appelez une fonction ou une méthode les arguments sont** _**copiés**_.

Quand on appelle `func (p Portefeuille) Deposer(montant int)` le `p` est une copie de ce sur quoi nous avons appelé la méthode.

Sans devenir trop informatique, quand vous créez une valeur - comme un portefeuille, elle est stockée quelque part en mémoire. Vous pouvez découvrir quelle est l'_adresse_ de ce bout de mémoire avec `&maVal`.

Expérimentez en ajoutant quelques impressions à votre code

```go
func TestPortefeuille(t *testing.T) {

	portefeuille := Portefeuille{}

	portefeuille.Deposer(10)

	resultat := portefeuille.Solde()

	fmt.Printf("adresse de solde dans test est %p \n", &portefeuille.solde)

	attendu := 10

	if resultat != attendu {
		t.Errorf("reçu %d attendu %d", resultat, attendu)
	}
}
```

```go
func (p Portefeuille) Deposer(montant int) {
	fmt.Printf("adresse de solde dans Deposer est %p \n", &p.solde)
	p.solde += montant
}
```

L'espace réservé `%p` imprime les adresses mémoire en notation base 16 avec des `0x` en tête et le caractère d'échappement `\n` imprime une nouvelle ligne.
Notez que nous obtenons le pointeur (adresse mémoire) de quelque chose en plaçant un caractère `&` au début du symbole.

Maintenant relancez le test

```text
adresse de solde dans Deposer est 0xc420012268
adresse de solde dans test est 0xc420012260
```

Vous pouvez voir que les adresses des deux soldes sont différentes. Donc quand nous changeons la valeur du solde dans le code, nous travaillons sur une copie de ce qui vient du test. Par conséquent, le solde dans le test est inchangé.

Nous pouvons corriger cela avec des _pointeurs_. Les [Pointeurs](https://gobyexample.com/pointers) nous permettent de _pointer_ vers des valeurs et puis nous permettent de les changer.
Donc plutôt que de prendre une copie de tout le Portefeuille, nous prenons à la place un pointeur vers ce portefeuille pour que nous puissions changer les valeurs originales à l'intérieur.

```go
func (p *Portefeuille) Deposer(montant int) {
	p.solde += montant
}

func (p *Portefeuille) Solde() int {
	return p.solde
}
```

La différence est que le type de récepteur est `*Portefeuille` plutôt que `Portefeuille` que vous pouvez lire comme "un pointeur vers un portefeuille".

Essayez de relancer les tests et ils devraient passer.

Maintenant vous pourriez vous demander, pourquoi ont-ils passé ? Nous n'avons pas déréférencé le pointeur dans la fonction, comme ceci :

```go
func (p *Portefeuille) Solde() int {
	return (*p).solde
}
```

et avons apparemment adressé l'objet directement. En fait, le code ci-dessus utilisant `(*p)` est absolument valide. Cependant, les créateurs de Go ont jugé cette notation encombrante, donc le langage nous permet d'écrire `p.solde`, sans déréférencement explicite.
Ces pointeurs vers des structs ont même leur propre nom : _pointeurs de struct_ et ils sont [automatiquement déréférencés](https://golang.org/ref/spec#Method_values).

Techniquement vous n'avez pas besoin de changer `Solde` pour utiliser un récepteur pointeur car prendre une copie du solde va bien. Cependant, par convention vous devriez garder vos types de récepteur de méthode identiques pour la cohérence.

## Refactoriser

Nous avons dit que nous faisions un portefeuille Bitcoin mais nous ne les avons pas mentionnés jusqu'à présent. Nous avons utilisé `int` parce que c'est un bon type pour compter les choses !

Il semble un peu excessif de créer une `struct` pour cela. `int` est bien en termes de la façon dont ça fonctionne mais ce n'est pas descriptif.

Go vous permet de créer de nouveaux types à partir de types existants.

La syntaxe est `type MonNom TypeOriginal`

```go
type Bitcoin int

type Portefeuille struct {
	solde Bitcoin
}

func (p *Portefeuille) Deposer(montant Bitcoin) {
	p.solde += montant
}

func (p *Portefeuille) Solde() Bitcoin {
	return p.solde
}
```

```go
func TestPortefeuille(t *testing.T) {

	portefeuille := Portefeuille{}

	portefeuille.Deposer(Bitcoin(10))

	resultat := portefeuille.Solde()

	attendu := Bitcoin(10)

	if resultat != attendu {
		t.Errorf("reçu %d attendu %d", resultat, attendu)
	}
}
```

Pour faire un `Bitcoin` vous utilisez juste la syntaxe `Bitcoin(999)`.

En faisant cela nous créons un nouveau type et nous pouvons déclarer des _méthodes_ sur eux. Cela peut être très utile quand vous voulez ajouter une fonctionnalité spécifique au domaine au-dessus de types existants.

Implémentons [Stringer](https://golang.org/pkg/fmt/#Stringer) sur Bitcoin

```go
type Stringer interface {
	String() string
}
```

Cette interface est définie dans le package `fmt` et vous permet de définir comment votre type est imprimé quand utilisé avec la chaîne de format `%s` dans les impressions.

```go
func (b Bitcoin) String() string {
	return fmt.Sprintf("%d BTC", b)
}
```

Comme vous pouvez le voir, la syntaxe pour créer une méthode sur une déclaration de type est la même que sur une struct.

Ensuite nous devons mettre à jour nos chaînes de format de test pour qu'elles utilisent `String()` à la place.

```go
	if resultat != attendu {
		t.Errorf("reçu %s attendu %s", resultat, attendu)
	}
```

Pour voir cela en action, cassez délibérément le test pour que nous puissions le voir

`portefeuille_test.go:18: reçu 10 BTC attendu 20 BTC`

Cela rend plus clair ce qui se passe dans notre test.

La prochaine exigence est pour une fonction `Retirer`.

## Écrivez le test d'abord

Presque l'opposé de `Deposer()`

```go
func TestPortefeuille(t *testing.T) {

	t.Run("déposer", func(t *testing.T) {
		portefeuille := Portefeuille{}

		portefeuille.Deposer(Bitcoin(10))

		resultat := portefeuille.Solde()

		attendu := Bitcoin(10)

		if resultat != attendu {
			t.Errorf("reçu %s attendu %s", resultat, attendu)
		}
	})

	t.Run("retirer", func(t *testing.T) {
		portefeuille := Portefeuille{solde: Bitcoin(20)}

		portefeuille.Retirer(Bitcoin(10))

		resultat := portefeuille.Solde()

		attendu := Bitcoin(10)

		if resultat != attendu {
			t.Errorf("reçu %s attendu %s", resultat, attendu)
		}
	})
}
```

## Essayez d'exécuter le test

`./portefeuille_test.go:26:19: portefeuille.Retirer undefined (type Portefeuille has no field or method Retirer)`

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

```go
func (p *Portefeuille) Retirer(montant Bitcoin) {

}
```

`portefeuille_test.go:33: reçu 20 BTC attendu 10 BTC`

## Écrivez assez de code pour le faire passer

```go
func (p *Portefeuille) Retirer(montant Bitcoin) {
	p.solde -= montant
}
```

## Refactoriser

Il y a de la duplication dans nos tests, refactorisons cela.

```go
func TestPortefeuille(t *testing.T) {

	verifierSolde := func(t testing.TB, portefeuille Portefeuille, attendu Bitcoin) {
		t.Helper()
		resultat := portefeuille.Solde()

		if resultat != attendu {
			t.Errorf("reçu %s attendu %s", resultat, attendu)
		}
	}

	t.Run("déposer", func(t *testing.T) {
		portefeuille := Portefeuille{}
		portefeuille.Deposer(Bitcoin(10))
		verifierSolde(t, portefeuille, Bitcoin(10))
	})

	t.Run("retirer", func(t *testing.T) {
		portefeuille := Portefeuille{solde: Bitcoin(20)}
		portefeuille.Retirer(Bitcoin(10))
		verifierSolde(t, portefeuille, Bitcoin(10))
	})

}
```

Que devrait-il se passer si vous essayez de `Retirer` plus que ce qui reste dans le compte ? Pour l'instant, notre exigence est de supposer qu'il n'y a pas de facilité de découvert.

Comment signalons-nous un problème quand nous utilisons `Retirer` ?

En Go, si vous voulez indiquer une erreur, il est idiomatique que votre fonction retourne une `err` pour que l'appelant vérifie et agisse dessus.

Essayons cela dans un test.

## Écrivez le test d'abord

```go
t.Run("retirer fonds insuffisants", func(t *testing.T) {
	soldeDepart := Bitcoin(20)
	portefeuille := Portefeuille{soldeDepart}
	err := portefeuille.Retirer(Bitcoin(100))

	verifierSolde(t, portefeuille, soldeDepart)

	if err == nil {
		t.Error("voulait une erreur mais n'en a pas eu")
	}
})
```

Nous voulons que `Retirer` retourne une erreur _si_ vous essayez de prendre plus que ce que vous avez et le solde devrait rester le même.

Nous vérifions ensuite qu'une erreur a été retournée en faisant échouer le test si elle est `nil`.

`nil` est synonyme de `null` d'autres langages de programmation. Les erreurs peuvent être `nil` parce que le type de retour de `Retirer` sera `error`, qui est une interface. Si vous voyez une fonction qui prend des arguments ou retourne des valeurs qui sont des interfaces, elles peuvent être nillables.

Comme `null` si vous essayez d'accéder à une valeur qui est `nil` cela lancera une **panique d'exécution**. C'est mauvais ! Vous devriez vous assurer de vérifier les nils.

## Essayez d'exécuter le test

`./portefeuille_test.go:31:25: portefeuille.Retirer(Bitcoin(100)) used as value`

Le libellé est peut-être un peu peu clair, mais notre intention précédente avec `Retirer` était juste de l'appeler, elle ne retournera jamais de valeur. Pour que cela compile nous devrons la changer pour qu'elle ait un type de retour.

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

```go
func (p *Portefeuille) Retirer(montant Bitcoin) error {
	p.solde -= montant
	return nil
}
```

Encore une fois, il est très important d'écrire juste assez de code pour satisfaire le compilateur. Nous corrigeons notre méthode `Retirer` pour retourner `error` et pour l'instant nous devons retourner _quelque chose_ alors retournons juste `nil`.

## Écrivez assez de code pour le faire passer

```go
func (p *Portefeuille) Retirer(montant Bitcoin) error {

	if montant > p.solde {
		return errors.New("oh non")
	}

	p.solde -= montant
	return nil
}
```

Rappelez-vous d'importer `errors` dans votre code.

`errors.New` crée une nouvelle `error` avec un message de votre choix.

## Refactoriser

Créons un Helper de test rapide pour notre vérification d'erreur pour améliorer la lisibilité du test

```go
verifierErreur := func(t testing.TB, err error) {
	t.Helper()
	if err == nil {
		t.Error("voulait une erreur mais n'en a pas eu")
	}
}
```

Et dans notre test

```go
t.Run("retirer fonds insuffisants", func(t *testing.T) {
	soldeDepart := Bitcoin(20)
	portefeuille := Portefeuille{soldeDepart}
	err := portefeuille.Retirer(Bitcoin(100))

	verifierErreur(t, err)
	verifierSolde(t, portefeuille, soldeDepart)
})
```

Espérons qu'en retournant une erreur de "oh non" vous pensiez que nous _pourrions_ itérer sur cela parce que ça ne semble pas très utile de retourner.

En supposant que l'erreur finit par être retournée à l'utilisateur, mettons à jour notre test pour affirmer sur une sorte de message d'erreur plutôt que juste l'existence d'une erreur.

## Écrivez le test d'abord

Mettons à jour notre Helper pour une `string` à comparer.

```go
verifierErreur := func(t testing.TB, recu error, attendu string) {
	t.Helper()

	if recu == nil {
		t.Fatal("n'a pas eu d'erreur mais en voulait une")
	}

	if recu.Error() != attendu {
		t.Errorf("reçu %q, attendu %q", recu, attendu)
	}
}
```

Comme vous pouvez le voir, les `Error`s peuvent être converties en chaîne avec la méthode `.Error()`, que nous faisons pour la comparer avec la chaîne que nous voulons. Nous nous assurons aussi que l'erreur n'est pas `nil` pour nous assurer de ne pas appeler `.Error()` sur `nil`.

Et puis mettons à jour l'appelant

```go
t.Run("retirer fonds insuffisants", func(t *testing.T) {
	soldeDepart := Bitcoin(20)
	portefeuille := Portefeuille{soldeDepart}
	err := portefeuille.Retirer(Bitcoin(100))

	verifierErreur(t, err, "ne peut pas retirer, fonds insuffisants")
	verifierSolde(t, portefeuille, soldeDepart)
})
```

Nous avons introduit `t.Fatal` qui arrêtera le test s'il est appelé. C'est parce que nous ne voulons pas faire plus d'assertions sur l'erreur retournée s'il n'y en a pas une. Sans cela le test continuerait à l'étape suivante et paniquerait à cause d'un pointeur nil.

## Essayez d'exécuter le test

`portefeuille_test.go:61: reçu err 'oh non' attendu 'ne peut pas retirer, fonds insuffisants'`

## Écrivez assez de code pour le faire passer

```go
func (p *Portefeuille) Retirer(montant Bitcoin) error {

	if montant > p.solde {
		return errors.New("ne peut pas retirer, fonds insuffisants")
	}

	p.solde -= montant
	return nil
}
```

## Refactoriser

Nous avons de la duplication du message d'erreur à la fois dans le code de test et le code `Retirer`.

Ce serait vraiment ennuyeux que le test échoue si quelqu'un voulait reformuler l'erreur et c'est juste trop de détail pour notre test. Nous ne nous _soucions_ pas vraiment du libellé exact, juste qu'une sorte d'erreur significative autour du retrait soit retournée étant donné une certaine condition.

En Go, les erreurs sont des valeurs, donc nous pouvons la refactoriser en une variable et avoir une seule source de vérité pour cela.

```go
var ErrFondsInsuffisants = errors.New("ne peut pas retirer, fonds insuffisants")

func (p *Portefeuille) Retirer(montant Bitcoin) error {

	if montant > p.solde {
		return ErrFondsInsuffisants
	}

	p.solde -= montant
	return nil
}
```

Le mot-clé `var` nous permet de définir des valeurs globales au package.

C'est un changement positif en soi parce que maintenant notre fonction `Retirer` semble très claire.

Ensuite nous pouvons refactoriser notre code de test pour utiliser cette valeur au lieu de chaînes spécifiques.

```go
func TestPortefeuille(t *testing.T) {

	t.Run("déposer", func(t *testing.T) {
		portefeuille := Portefeuille{}
		portefeuille.Deposer(Bitcoin(10))
		verifierSolde(t, portefeuille, Bitcoin(10))
	})

	t.Run("retirer avec fonds", func(t *testing.T) {
		portefeuille := Portefeuille{Bitcoin(20)}
		portefeuille.Retirer(Bitcoin(10))
		verifierSolde(t, portefeuille, Bitcoin(10))
	})

	t.Run("retirer fonds insuffisants", func(t *testing.T) {
		portefeuille := Portefeuille{Bitcoin(20)}
		err := portefeuille.Retirer(Bitcoin(100))

		verifierErreur(t, err, ErrFondsInsuffisants)
		verifierSolde(t, portefeuille, Bitcoin(20))
	})
}

func verifierSolde(t testing.TB, portefeuille Portefeuille, attendu Bitcoin) {
	t.Helper()
	resultat := portefeuille.Solde()

	if resultat != attendu {
		t.Errorf("reçu %q attendu %q", resultat, attendu)
	}
}

func verifierErreur(t testing.TB, recu, attendu error) {
	t.Helper()
	if recu == nil {
		t.Fatal("n'a pas eu d'erreur mais en voulait une")
	}

	if recu != attendu {
		t.Errorf("reçu %q, attendu %q", recu, attendu)
	}
}
```

Et maintenant le test est plus facile à suivre aussi.

J'ai déplacé les Helpers hors de la fonction de test principale juste pour que quand quelqu'un ouvre un fichier, il puisse commencer à lire nos assertions d'abord, plutôt que quelques Helpers.

Une autre propriété utile des tests est qu'ils nous aident à comprendre l'usage _réel_ de notre code pour que nous puissions faire du code sympathique. Nous pouvons voir ici qu'un développeur peut simplement appeler notre code et faire une vérification d'égalité avec `ErrFondsInsuffisants` et agir en conséquence.

### Erreurs non vérifiées

Bien que le compilateur Go vous aide beaucoup, parfois il y a des choses que vous pouvez encore manquer et la gestion d'erreurs peut parfois être délicate.

Il y a un scénario que nous n'avons pas testé. Pour le trouver, exécutez ce qui suit dans un terminal pour installer `errcheck`, un des nombreux linters disponibles pour Go.

`go install github.com/kisielk/errcheck@latest`

Puis, dans le répertoire avec votre code exécutez `errcheck .`

Vous devriez obtenir quelque chose comme

`portefeuille_test.go:17:18: portefeuille.Retirer(Bitcoin(10))`

Ce que cela nous dit c'est que nous n'avons pas vérifié l'erreur retournée sur cette ligne de code. Cette ligne de code sur mon ordinateur correspond à notre scénario de retrait normal parce que nous n'avons pas vérifié que si le `Retirer` réussit qu'une erreur n'est _pas_ retournée.

Voici le code de test final qui prend en compte cela.

```go
func TestPortefeuille(t *testing.T) {

	t.Run("déposer", func(t *testing.T) {
		portefeuille := Portefeuille{}
		portefeuille.Deposer(Bitcoin(10))

		verifierSolde(t, portefeuille, Bitcoin(10))
	})

	t.Run("retirer avec fonds", func(t *testing.T) {
		portefeuille := Portefeuille{Bitcoin(20)}
		err := portefeuille.Retirer(Bitcoin(10))

		verifierAucuneErreur(t, err)
		verifierSolde(t, portefeuille, Bitcoin(10))
	})

	t.Run("retirer fonds insuffisants", func(t *testing.T) {
		portefeuille := Portefeuille{Bitcoin(20)}
		err := portefeuille.Retirer(Bitcoin(100))

		verifierErreur(t, err, ErrFondsInsuffisants)
		verifierSolde(t, portefeuille, Bitcoin(20))
	})
}

func verifierSolde(t testing.TB, portefeuille Portefeuille, attendu Bitcoin) {
	t.Helper()
	resultat := portefeuille.Solde()

	if resultat != attendu {
		t.Errorf("reçu %s attendu %s", resultat, attendu)
	}
}

func verifierAucuneErreur(t testing.TB, recu error) {
	t.Helper()
	if recu != nil {
		t.Fatal("a eu une erreur mais n'en voulait pas")
	}
}

func verifierErreur(t testing.TB, recu error, attendu error) {
	t.Helper()
	if recu == nil {
		t.Fatal("n'a pas eu d'erreur mais en voulait une")
	}

	if recu != attendu {
		t.Errorf("reçu %s, attendu %s", recu, attendu)
	}
}
```

## Conclusion

### Pointeurs

* Go copie les valeurs quand vous les passez aux fonctions/méthodes, donc si vous écrivez une fonction qui a besoin de muter l'état vous aurez besoin qu'elle prenne un pointeur vers la chose que vous voulez changer.
* Le fait que Go prenne une copie des valeurs est utile beaucoup de fois mais parfois vous ne voudrez pas que votre système fasse une copie de quelque chose, auquel cas vous avez besoin de passer une référence. Les exemples incluent référencer de très grandes structures de données ou des choses où seule une instance est nécessaire \(comme les pools de connexion de base de données\).

### nil

* Les pointeurs peuvent être nil
* Quand une fonction retourne un pointeur vers quelque chose, vous devez vous assurer de vérifier s'il est nil ou vous pourriez lever une exception d'exécution - le compilateur ne vous aidera pas ici.
* Utile quand vous voulez décrire une valeur qui pourrait manquer

### Erreurs

* Les erreurs sont la façon de signifier l'échec quand on appelle une fonction/méthode.
* En écoutant nos tests nous avons conclu que vérifier une chaîne dans une erreur résulterait en un test fragile. Donc nous avons refactorisé notre implémentation pour utiliser une valeur significative à la place et cela a résulté en du code plus facile à tester et nous avons conclu que ce serait plus facile pour les utilisateurs de notre API aussi.
* Ce n'est pas la fin de l'histoire avec la gestion d'erreurs, vous pouvez faire des choses plus sophistiquées mais c'est juste une introduction. Les sections ultérieures couvriront plus de stratégies.
* [Ne vérifiez pas juste les erreurs, gérez-les gracieusement](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)

### Créer de nouveaux types à partir de types existants

* Utile pour ajouter plus de signification spécifique au domaine aux valeurs
* Peut vous permettre d'implémenter des interfaces

Les pointeurs et les erreurs sont une grande partie de l'écriture de Go avec laquelle vous devez être à l'aise. Heureusement le compilateur vous aidera _habituellement_ si vous faites quelque chose de mal, prenez juste votre temps et lisez l'erreur.
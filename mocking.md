# Mocking

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/mocking)**

On vous a demandé d'écrire un programme qui compte à rebours à partir de 3, en affichant chaque nombre sur une nouvelle ligne (avec une pause d'une seconde), et lorsqu'il atteint zéro, il affiche "Go!" et se termine.

```
3
2
1
Go!
```

Nous allons aborder ce problème en écrivant une fonction appelée `Compte` que nous intégrerons ensuite dans un programme `main` pour qu'il ressemble à quelque chose comme :

```go
package main

func main() {
	Compte()
}
```

Bien que ce soit un programme assez simple, pour le tester complètement, nous devrons, comme toujours, adopter une approche _itérative_ et _pilotée par les tests_.

Que veux-je dire par itératif ? Nous nous assurons de prendre les plus petites étapes possibles pour avoir un _logiciel utile_.

Nous ne voulons pas passer beaucoup de temps avec du code qui fonctionnera théoriquement après quelques bricolages, car c'est souvent ainsi que les développeurs tombent dans des pièges. **C'est une compétence importante que de pouvoir découper les exigences en morceaux aussi petits que possible afin d'avoir un _logiciel fonctionnel_.**

Voici comment nous pouvons diviser notre travail et l'itérer :

- Afficher 3
- Afficher 3, 2, 1 et Go!
- Attendre une seconde entre chaque ligne

## Écrivez le test d'abord

Notre logiciel doit imprimer sur stdout et nous avons vu comment nous pouvons utiliser l'Injection de Dépendances (DI) pour faciliter le test dans la section DI.

```go
func TestCompte(t *testing.T) {
	buffer := &bytes.Buffer{}

	Compte(buffer)

	obtenu := buffer.String()
	attendu := "3"

	if obtenu != attendu {
		t.Errorf("obtenu %q attendu %q", obtenu, attendu)
	}
}
```

Si quelque chose comme `buffer` ne vous est pas familier, relisez [la section précédente](dependency-injection.md).

Nous savons que nous voulons que notre fonction `Compte` écrive des données quelque part et `io.Writer` est la manière de facto de capturer cela en tant qu'interface en Go.

- Dans `main`, nous enverrons vers `os.Stdout` pour que nos utilisateurs voient le compte à rebours affiché sur le terminal.
- Dans le test, nous enverrons vers `bytes.Buffer` pour que nos tests puissent capturer les données générées.

## Essayez d'exécuter le test

`./compte_test.go:11:2: undefined: Compte`

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Définissez `Compte`

```go
func Compte() {}
```

Essayez à nouveau

```
./compte_test.go:11:11: too many arguments in call to Compte
    have (*bytes.Buffer)
    want ()
```

Le compilateur vous indique ce que pourrait être la signature de votre fonction, alors mettez-la à jour.

```go
func Compte(out *bytes.Buffer) {}
```

`compte_test.go:17: obtenu '' attendu '3'`

Parfait !

## Écrivez assez de code pour le faire passer

```go
func Compte(out *bytes.Buffer) {
	fmt.Fprint(out, "3")
}
```

Nous utilisons `fmt.Fprint` qui prend un `io.Writer` (comme `*bytes.Buffer`) et lui envoie une `string`. Le test devrait passer.

## Refactoriser

Nous savons que bien que `*bytes.Buffer` fonctionne, il serait préférable d'utiliser une interface à usage général à la place.

```go
func Compte(out io.Writer) {
	fmt.Fprint(out, "3")
}
```

Relancez les tests et ils devraient passer.

Pour finaliser, connectons maintenant notre fonction à un `main` pour avoir un logiciel fonctionnel qui nous rassure sur notre progression.

```go
package main

import (
	"fmt"
	"io"
	"os"
)

func Compte(out io.Writer) {
	fmt.Fprint(out, "3")
}

func main() {
	Compte(os.Stdout)
}
```

Essayez d'exécuter le programme et admirez votre travail.

Oui, cela semble trivial, mais c'est l'approche que je recommanderais pour tout projet. **Prenez une fine tranche de fonctionnalité et faites-la fonctionner de bout en bout, soutenue par des tests.**

Ensuite, nous pouvons faire afficher 2, 1 et puis "Go!".

## Écrivez le test d'abord

En investissant pour que la tuyauterie globale fonctionne correctement, nous pouvons itérer sur notre solution en toute sécurité et facilement. Nous n'aurons plus besoin d'arrêter et de redémarrer le programme pour être sûrs qu'il fonctionne, car toute la logique est testée.

```go
func TestCompte(t *testing.T) {
	buffer := &bytes.Buffer{}

	Compte(buffer)

	obtenu := buffer.String()
	attendu := `3
2
1
Go!`

	if obtenu != attendu {
		t.Errorf("obtenu %q attendu %q", obtenu, attendu)
	}
}
```

La syntaxe des backticks est une autre façon de créer une `string`, mais elle permet d'inclure des éléments comme les sauts de ligne, ce qui est parfait pour notre test.

## Essayez d'exécuter le test

```
compte_test.go:21: obtenu '3' attendu '3
        2
        1
        Go!'
```
## Écrivez assez de code pour le faire passer

```go
func Compte(out io.Writer) {
	for i := 3; i > 0; i-- {
		fmt.Fprintln(out, i)
	}
	fmt.Fprint(out, "Go!")
}
```

Utilisez une boucle `for` qui compte à rebours avec `i--` et utilisez `fmt.Fprintln` pour imprimer sur `out` avec notre nombre suivi d'un caractère de nouvelle ligne. Enfin, utilisez `fmt.Fprint` pour envoyer "Go!" à la fin.

## Refactoriser

Il n'y a pas grand-chose à refactoriser à part transformer certaines valeurs magiques en constantes nommées.

```go
const motFinal = "Go!"
const debutCompteARebours = 3

func Compte(out io.Writer) {
	for i := debutCompteARebours; i > 0; i-- {
		fmt.Fprintln(out, i)
	}
	fmt.Fprint(out, motFinal)
}
```

Si vous exécutez le programme maintenant, vous devriez obtenir le résultat souhaité, mais nous n'avons pas encore le compte à rebours dramatique avec les pauses d'une seconde.

Go vous permet d'y parvenir avec `time.Sleep`. Essayez de l'ajouter à notre code.

```go
func Compte(out io.Writer) {
	for i := debutCompteARebours; i > 0; i-- {
		fmt.Fprintln(out, i)
		time.Sleep(1 * time.Second)
	}

	fmt.Fprint(out, motFinal)
}
```

Si vous exécutez le programme, il fonctionne comme nous le souhaitons.

## Mocking

Les tests passent toujours et le logiciel fonctionne comme prévu, mais nous avons quelques problèmes :
- Nos tests prennent 3 secondes à s'exécuter.
    - Chaque article prospectif sur le développement logiciel souligne l'importance des boucles de retour rapides.
    - **Les tests lents ruinent la productivité des développeurs**.
    - Imaginez si les exigences deviennent plus sophistiquées, nécessitant plus de tests. Sommes-nous satisfaits d'ajouter 3 secondes au temps d'exécution des tests pour chaque nouveau test de `Compte` ?
- Nous n'avons pas testé une propriété importante de notre fonction.

Nous avons une dépendance à `Sleep` que nous devons extraire pour pouvoir la contrôler dans nos tests.

Si nous pouvons _mocker_ `time.Sleep`, nous pouvons utiliser _l'injection de dépendances_ pour l'utiliser à la place d'un "vrai" `time.Sleep` et ensuite nous pouvons **espionner les appels** pour faire des assertions sur eux.

## Écrivez le test d'abord

Définissons notre dépendance comme une interface. Cela nous permet ensuite d'utiliser une _vraie_ Sleeper dans `main` et une _Sleeper espion_ dans nos tests. En utilisant une interface, notre fonction `Compte` n'en a pas conscience et cela ajoute de la flexibilité pour l'appelant.

```go
type Sleeper interface {
	Dormir()
}
```

J'ai pris la décision de conception que notre fonction `Compte` ne serait pas responsable de la durée du sommeil. Cela simplifie un peu notre code pour l'instant et signifie qu'un utilisateur de notre fonction peut configurer cette somnolence comme il le souhaite.

Maintenant, nous devons créer un _mock_ pour nos tests.

```go
type SleeperEspion struct {
	Appels int
}

func (s *SleeperEspion) Dormir() {
	s.Appels++
}
```

Les _espions_ sont un type de _mock_ qui peut enregistrer comment une dépendance est utilisée. Ils peuvent enregistrer les arguments envoyés, combien de fois elle a été appelée, etc. Dans notre cas, nous gardons une trace du nombre de fois où `Dormir()` est appelé afin de pouvoir le vérifier dans notre test.

Mettez à jour les tests pour injecter une dépendance à notre Espion et affirmer que le sommeil a été appelé 3 fois.

```go
func TestCompte(t *testing.T) {
	buffer := &bytes.Buffer{}
	SleeperEspion := &SleeperEspion{}

	Compte(buffer, SleeperEspion)

	obtenu := buffer.String()
	attendu := `3
2
1
Go!`

	if obtenu != attendu {
		t.Errorf("obtenu %q attendu %q", obtenu, attendu)
	}

	if SleeperEspion.Appels != 3 {
		t.Errorf("pas assez d'appels à Sleeper, attendu 3 obtenu %d", SleeperEspion.Appels)
	}
}
```

## Essayez d'exécuter le test

```
too many arguments in call to Compte
    have (*bytes.Buffer, *SleeperEspion)
    want (io.Writer)
```

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Nous devons mettre à jour `Compte` pour accepter notre `Sleeper`

```go
func Compte(out io.Writer, Sleeper Sleeper) {
	for i := debutCompteARebours; i > 0; i-- {
		fmt.Fprintln(out, i)
		time.Sleep(1 * time.Second)
	}

	fmt.Fprint(out, motFinal)
}
```

Si vous essayez à nouveau, votre `main` ne compilera plus pour la même raison

```
./main.go:26:11: not enough arguments in call to Compte
    have (*os.File)
    want (io.Writer, Sleeper)
```

Créons une _vraie_ Sleeper qui implémente l'interface dont nous avons besoin

```go
type SleeperParDefaut struct{}

func (d *SleeperParDefaut) Dormir() {
	time.Sleep(1 * time.Second)
}
```

Nous pouvons ensuite l'utiliser dans notre vraie application comme ceci

```go
func main() {
	Sleeper := &SleeperParDefaut{}
	Compte(os.Stdout, Sleeper)
}
```

## Écrivez assez de code pour le faire passer

Le test compile maintenant mais ne passe pas car nous appelons toujours le `time.Sleep` plutôt que la dépendance injectée. Corrigeons cela.

```go
func Compte(out io.Writer, Sleeper Sleeper) {
	for i := debutCompteARebours; i > 0; i-- {
		fmt.Fprintln(out, i)
		Sleeper.Dormir()
	}

	fmt.Fprint(out, motFinal)
}
```

Le test devrait passer et ne plus prendre 3 secondes.

### Encore quelques problèmes

Il y a encore une autre propriété importante que nous n'avons pas testée.

`Compte` devrait dormir avant chaque impression suivante, par exemple :

- `Imprimer N`
- `Dormir`
- `Imprimer N-1`
- `Dormir`
- `Imprimer Go!`
- etc

Notre dernier changement affirme seulement qu'il a dormi 3 fois, mais ces sommeils pourraient se produire hors séquence.

Lorsque vous écrivez des tests, si vous n'êtes pas sûr que vos tests vous donnent une confiance suffisante, cassez-les simplement ! (assurez-vous d'avoir d'abord validé vos modifications dans le contrôle de source). Modifiez le code comme suit

```go
func Compte(out io.Writer, Sleeper Sleeper) {
	for i := debutCompteARebours; i > 0; i-- {
		Sleeper.Dormir()
	}

	for i := debutCompteARebours; i > 0; i-- {
		fmt.Fprintln(out, i)
	}

	fmt.Fprint(out, motFinal)
}
```

Si vous exécutez vos tests, ils devraient toujours passer même si l'implémentation est incorrecte.

Utilisons à nouveau l'espionnage avec un nouveau test pour vérifier que l'ordre des opérations est correct.

Nous avons deux dépendances différentes et nous voulons enregistrer toutes leurs opérations dans une seule liste. Nous allons donc créer _un espion pour les deux_.

```go
type EspionOperationsCompte struct {
	Appels []string
}

func (s *EspionOperationsCompte) Dormir() {
	s.Appels = append(s.Appels, dormir)
}

func (s *EspionOperationsCompte) Write(p []byte) (n int, err error) {
	s.Appels = append(s.Appels, ecrire)
	return
}

const ecrire = "écrire"
const dormir = "dormir"
```

Notre `EspionOperationsCompte` implémente à la fois `io.Writer` et `Sleeper`, en enregistrant chaque appel dans une seule tranche. Dans ce test, nous ne nous soucions que de l'ordre des opérations, donc les enregistrer sous forme de liste d'opérations nommées est suffisant.

Nous pouvons maintenant ajouter un sous-test à notre suite de tests qui vérifie que nos sommeils et impressions fonctionnent dans l'ordre que nous espérons

```go
t.Run("dormir avant chaque impression", func(t *testing.T) {
	espionDormirImprimer := &EspionOperationsCompte{}
	Compte(espionDormirImprimer, espionDormirImprimer)

	attendu := []string{
		ecrire,
		dormir,
		ecrire,
		dormir,
		ecrire,
		dormir,
		ecrire,
	}

	if !reflect.DeepEqual(attendu, espionDormirImprimer.Appels) {
		t.Errorf("appels attendus %v obtenus %v", attendu, espionDormirImprimer.Appels)
	}
})
```

Ce test devrait maintenant échouer. Remettez `Compte` à ce qu'il était pour corriger le test.

Nous avons maintenant deux tests espionnant la `Sleeper`, nous pouvons donc refactoriser notre test pour que l'un teste ce qui est imprimé et l'autre s'assure que nous dormons entre les impressions. Enfin, nous pouvons supprimer notre premier espion car il n'est plus utilisé.

```go
func TestCompte(t *testing.T) {

	t.Run("imprime 3 jusqu'à Go!", func(t *testing.T) {
		buffer := &bytes.Buffer{}
		Compte(buffer, &EspionOperationsCompte{})

		obtenu := buffer.String()
		attendu := `3
2
1
Go!`

		if obtenu != attendu {
			t.Errorf("obtenu %q attendu %q", obtenu, attendu)
		}
	})

	t.Run("dormir avant chaque impression", func(t *testing.T) {
		espionDormirImprimer := &EspionOperationsCompte{}
		Compte(espionDormirImprimer, espionDormirImprimer)

		attendu := []string{
			ecrire,
			dormir,
			ecrire,
			dormir,
			ecrire,
			dormir,
			ecrire,
		}

		if !reflect.DeepEqual(attendu, espionDormirImprimer.Appels) {
			t.Errorf("appels attendus %v obtenus %v", attendu, espionDormirImprimer.Appels)
		}
	})
}
```

Nous avons maintenant notre fonction et ses 2 propriétés importantes correctement testées.

## Étendre Sleeper pour la rendre configurable

Une fonctionnalité intéressante serait que la `Sleeper` soit configurable. Cela signifie que nous pouvons ajuster le temps de sommeil dans notre programme principal.

### Écrivez le test d'abord

Créons d'abord un nouveau type pour `SleeperConfigurable` qui accepte ce dont nous avons besoin pour la configuration et les tests.

```go
type SleeperConfigurable struct {
	duree time.Duration
	dormir func(time.Duration)
}
```

Nous utilisons `duree` pour configurer le temps de sommeil et `dormir` comme moyen de passer une fonction de sommeil. La signature de `dormir` est la même que pour `time.Sleep`, nous permettant d'utiliser `time.Sleep` dans notre implémentation réelle et l'espion suivant dans nos tests :

```go
type EspionTemps struct {
	dureeDormie time.Duration
}

func (s *EspionTemps) Dormir(duree time.Duration) {
	s.dureeDormie = duree
}
```

Avec notre espion en place, nous pouvons créer un nouveau test pour la Sleeper configurable.

```go
func TestSleeperConfigurable(t *testing.T) {
	tempsDormir := 5 * time.Second

	espionTemps := &EspionTemps{}
	Sleeper := SleeperConfigurable{tempsDormir, espionTemps.Dormir}
	Sleeper.Dormir()

	if espionTemps.dureeDormie != tempsDormir {
		t.Errorf("aurait dû dormir pendant %v mais a dormi pendant %v", tempsDormir, espionTemps.dureeDormie)
	}
}
```

Il ne devrait y avoir rien de nouveau dans ce test et il est configuré de manière très similaire aux tests de mock précédents.

### Essayez d'exécuter le test
```
Sleeper.Dormir undefined (type SleeperConfigurable has no field or method Dormir, but does have dormir)

```

Vous devriez voir un message d'erreur très clair indiquant que nous n'avons pas créé de méthode `Dormir` sur notre `SleeperConfigurable`.

### Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue
```go
func (c *SleeperConfigurable) Dormir() {
}
```

Avec notre nouvelle fonction `Dormir` implémentée, nous avons un test qui échoue.

```
compte_test.go:56: aurait dû dormir pendant 5s mais a dormi pendant 0s
```

### Écrivez assez de code pour le faire passer

Tout ce que nous devons faire maintenant est d'implémenter la fonction `Dormir` pour `SleeperConfigurable`.

```go
func (c *SleeperConfigurable) Dormir() {
	c.dormir(c.duree)
}
```

Avec ce changement, tous les tests devraient à nouveau passer et vous pourriez vous demander pourquoi tant d'efforts alors que le programme principal n'a pas du tout changé. Espérons que cela deviendra clair après la section suivante.

### Nettoyage et refactorisation

La dernière chose que nous devons faire est d'utiliser notre `SleeperConfigurable` dans la fonction principale.

```go
func main() {
	Sleeper := &SleeperConfigurable{1 * time.Second, time.Sleep}
	Compte(os.Stdout, Sleeper)
}
```

Si nous exécutons les tests et le programme manuellement, nous pouvons voir que tout le comportement reste le même.

Puisque nous utilisons la `SleeperConfigurable`, il est maintenant sûr de supprimer l'implémentation `SleeperParDefaut`. Finalisant notre programme et ayant une Sleeper plus [générique](https://stackoverflow.com/questions/19291776/whats-the-difference-between-abstraction-and-generalization) avec des comptes à rebours de durée arbitraire.

## Mais le mocking n'est-il pas mauvais ?

Vous avez peut-être entendu dire que le mocking est mauvais. Comme tout dans le développement logiciel, il peut être utilisé à mauvais escient, tout comme [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself).

Les gens se retrouvent généralement dans une mauvaise situation lorsqu'ils n'_écoutent pas leurs tests_ et ne _respectent pas l'étape de refactorisation_.

Si votre code de mocking devient compliqué ou si vous devez mocker beaucoup de choses pour tester quelque chose, vous devriez _écouter_ ce mauvais sentiment et réfléchir à votre code. Généralement, c'est le signe que :

- La chose que vous testez doit faire trop de choses (car elle a trop de dépendances à mocker)
  - Décomposez le module pour qu'il en fasse moins
- Ses dépendances sont trop granulaires
  - Réfléchissez à la façon dont vous pouvez consolider certaines de ces dépendances en un module significatif
- Votre test est trop préoccupé par les détails d'implémentation
  - Privilégiez le test du comportement attendu plutôt que de l'implémentation

Normalement, beaucoup de mocking indique une _mauvaise abstraction_ dans votre code.

**Ce que les gens voient ici est une faiblesse du TDD, mais c'est en fait une force**. Le plus souvent, un code de test médiocre est le résultat d'une mauvaise conception ou, pour le dire plus gentiment, un code bien conçu est facile à tester.

### Mais les mocks et les tests me compliquent toujours la vie !

Vous êtes-vous déjà retrouvé dans cette situation ?

- Vous voulez faire un peu de refactorisation
- Pour ce faire, vous finissez par changer beaucoup de tests
- Vous remettez en question le TDD et publiez un article sur Medium intitulé "Le mocking est nocif"

C'est généralement le signe que vous testez trop de _détails d'implémentation_. Essayez de faire en sorte que vos tests testent le _comportement utile_, sauf si l'implémentation est vraiment importante pour le fonctionnement du système.

Il est parfois difficile de savoir _à quel niveau_ tester exactement, mais voici quelques réflexions et règles que j'essaie de suivre :

- **La définition du refactoring est que le code change mais que le comportement reste le même**. Si vous avez décidé de faire un refactoring, en théorie, vous devriez pouvoir faire le commit sans aucun changement de test. Alors, lorsque vous écrivez un test, demandez-vous :
  - Est-ce que je teste le comportement que je veux, ou les détails d'implémentation ?
  - Si je devais refactoriser ce code, devrais-je apporter beaucoup de changements aux tests ?
- Bien que Go vous permette de tester des fonctions privées, je les éviterais car les fonctions privées sont des détails d'implémentation pour soutenir le comportement public. Testez le comportement public. Sandi Metz décrit les fonctions privées comme étant "moins stables" et vous ne voulez pas que vos tests y soient couplés.
- J'ai l'impression que si un test fonctionne avec **plus de 3 mocks, c'est un signal d'alarme** - il est temps de repenser la conception.
- Utilisez les espions avec prudence. Les espions vous permettent de voir l'intérieur de l'algorithme que vous écrivez, ce qui peut être très utile, mais cela signifie un couplage plus étroit entre votre code de test et l'implémentation. **Assurez-vous que vous vous souciez réellement de ces détails si vous allez les espionner**.

#### Ne puis-je pas simplement utiliser un framework de mocking ?

Le mocking ne nécessite pas de magie et est relativement simple ; l'utilisation d'un framework peut rendre le mocking plus compliqué qu'il ne l'est. Nous n'utilisons pas l'automocking dans ce chapitre afin d'obtenir :

- une meilleure compréhension de la façon de mocker
- la pratique de l'implémentation des interfaces

Dans les projets collaboratifs, il y a une valeur à auto-générer des mocks. Dans une équipe, un outil de génération de mocks codifie la cohérence autour des doubles de test. Cela évitera les doubles de test mal écrits, ce qui peut se traduire par des tests mal écrits.

Vous ne devriez utiliser qu'un générateur de mock qui génère des doubles de test contre une interface. Tout outil qui dicte trop la façon dont les tests sont écrits, ou qui utilise beaucoup de "magie", peut aller au diable.

## Récapitulatif

### Plus sur l'approche TDD

- Face à des exemples moins triviaux, décomposez le problème en "tranches verticales minces". Essayez d'arriver à un point où vous avez _un logiciel fonctionnel soutenu par des tests_ aussi rapidement que possible, pour éviter de vous retrouver dans des impasses et d'adopter une approche "big bang".
- Une fois que vous avez un logiciel qui fonctionne, il devrait être plus facile de _l'itérer par petites étapes_ jusqu'à ce que vous arriviez au logiciel dont vous avez besoin.

> "Quand utiliser le développement itératif ? Vous devriez utiliser le développement itératif uniquement sur les projets que vous voulez réussir."
>
> Martin Fowler.

### Mocking

- **Sans mocking, des domaines importants de votre code resteront non testés**. Dans notre cas, nous ne serions pas en mesure de tester que notre code faisait une pause entre chaque impression, mais il existe d'innombrables autres exemples. Appeler un service qui _peut_ échouer ? Vouloir tester votre système dans un état particulier ? Il est très difficile de tester ces scénarios sans mocking.
- Sans mocks, vous pourriez avoir à configurer des bases de données et d'autres éléments tiers juste pour tester des règles métier simples. Vous risquez d'avoir des tests lents, ce qui entraîne des **boucles de rétroaction lentes**.
- En devant lancer une base de données ou un service web pour tester quelque chose, vous risquez d'avoir des **tests fragiles** en raison du manque de fiabilité de ces services.

Une fois qu'un développeur a appris le mocking, il devient très facile de sur-tester chaque facette d'un système en termes de _façon dont il fonctionne_ plutôt que de _ce qu'il fait_. Soyez toujours conscient de **la valeur de vos tests** et de l'impact qu'ils auraient sur les refactorisations futures.

Dans ce chapitre sur le mocking, nous n'avons couvert que les **Spies**, qui sont un type de mock. Les mocks sont un type de "double de test".

> [Test Double est un terme générique pour tout cas où vous remplacez un objet de production à des fins de test.](https://martinfowler.com/bliki/TestDouble.html)

Parmi les doubles de test, il existe différents types comme les stubs, les spies et les mocks ! Consultez [l'article de Martin Fowler](https://martinfowler.com/bliki/TestDouble.html) pour plus de détails.

## Bonus - Exemple d'itérateurs de Go 1.23

Dans Go 1.23, [les itérateurs ont été introduits](https://tip.golang.org/doc/go1.23). Nous pouvons utiliser les itérateurs de diverses façons, dans ce cas, nous pouvons créer un itérateur `compteARebroursDepuis`, qui retournera les nombres du compte à rebours dans l'ordre inverse.

Avant d'aborder la façon dont nous écrivons des itérateurs personnalisés, voyons comment nous les utilisons. Plutôt que d'écrire une boucle assez impérative pour compter à rebours à partir d'un nombre, nous pouvons rendre ce code plus expressif en utilisant `range` sur notre itérateur personnalisé `compteARebroursDepuis`.

```go
func Compte(out io.Writer, Sleeper Sleeper) {
	for i := range compteARebroursDepuis(3) {
		fmt.Fprintln(out, i)
		Sleeper.Dormir()
	}

	fmt.Fprint(out, motFinal)
}
```

Pour écrire un itérateur comme `compteARebroursDepuis`, vous devez écrire une fonction d'une manière particulière. D'après la documentation :

    La clause "range" dans une boucle "for-range" accepte maintenant des fonctions d'itérateur des types suivants
        func(func() bool)
        func(func(K) bool)
        func(func(K, V) bool)

(Les `K` et `V` représentent respectivement les types de clé et de valeur.)

Dans notre cas, nous n'avons pas de clés, juste des valeurs. Go fournit également un type pratique `iter.Seq[T]` qui est un alias de type pour `func(func(T) bool)`.

```go
func compteARebroursDepuis(depuis int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := depuis; i > 0; i-- {
			if !yield(i) {
				return
			}
		}
	}
}
```

C'est un itérateur simple, qui générera des nombres dans l'ordre inverse, en commençant par `depuis` - parfait pour notre cas d'utilisation.
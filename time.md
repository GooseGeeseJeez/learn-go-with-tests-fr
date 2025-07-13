# Time (Temps)

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/time)**

Le propriétaire du produit souhaite que nous élargissions les fonctionnalités de notre application en ligne de commande en aidant un groupe de personnes à jouer au Poker Texas-Holdem.

## Juste assez d'informations sur le poker

Vous n'aurez pas besoin de connaître grand-chose sur le poker, seulement qu'à certains intervalles de temps, tous les joueurs doivent être informés d'une valeur croissante de "blind" (mise obligatoire).

Notre application aidera à suivre quand la blind doit augmenter, et de combien elle doit être.

- Au démarrage, elle demande combien de joueurs participent. Cela détermine le temps qui s'écoule avant que la mise "blind" n'augmente.
  - Il y a un temps de base de 5 minutes.
  - Pour chaque joueur, 1 minute est ajoutée.
  - Par exemple, 6 joueurs équivalent à 11 minutes pour la blind.
- Après l'expiration du temps de la blind, le jeu doit alerter les joueurs du nouveau montant de la mise blind.
- La blind commence à 100 jetons, puis 200, 400, 600, 1000, 2000 et continue à doubler jusqu'à la fin de la partie (notre fonctionnalité précédente de "Ruth wins" doit toujours terminer la partie)

## Rappel du code

Dans le chapitre précédent, nous avons commencé à créer une application en ligne de commande qui accepte déjà une commande de type `{name} wins`. Voici à quoi ressemble le code actuel de `CLI`, mais assurez-vous de vous familiariser avec le reste du code avant de commencer.

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
}

func NewCLI(store PlayerStore, in io.Reader) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
	}
}

func (cli *CLI) PlayPoker() {
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins", "", 1)
}

func (cli *CLI) readLine() string {
	cli.in.Scan()
	return cli.in.Text()
}
```


### `time.AfterFunc`

Nous voulons pouvoir programmer notre application pour qu'elle affiche les valeurs des mises blind à certaines durées dépendant du nombre de joueurs.

Pour limiter la portée de ce que nous devons faire, nous allons oublier pour l'instant la partie concernant le nombre de joueurs et supposer simplement qu'il y a 5 joueurs, donc nous testerons que _toutes les 10 minutes, la nouvelle valeur de la mise blind est imprimée_.

Comme d'habitude, la bibliothèque standard nous couvre avec [`func AfterFunc(d Duration, f func()) *Timer`](https://golang.org/pkg/time/#AfterFunc)

> `AfterFunc` attend que la durée s'écoule puis appelle f dans sa propre goroutine. Il renvoie un `Timer` qui peut être utilisé pour annuler l'appel en utilisant sa méthode Stop.

### [`time.Duration`](https://golang.org/pkg/time/#Duration)

> Une Duration représente le temps écoulé entre deux instants sous forme d'un comptage de nanosecondes en int64.

La bibliothèque time possède un certain nombre de constantes pour vous permettre de multiplier ces nanosecondes afin qu'elles soient un peu plus lisibles pour le type de scénarios que nous allons réaliser.

```
5 * time.Second
```

Lorsque nous appellerons `PlayPoker`, nous programmerons toutes nos alertes blind.

Tester cela peut être un peu délicat. Nous voudrons vérifier que chaque période est programmée avec le bon montant de blind, mais si vous regardez la signature de `time.AfterFunc`, son deuxième argument est la fonction qu'il exécutera. Vous ne pouvez pas comparer des fonctions en Go, nous ne pourrions donc pas tester quelle fonction a été envoyée. Nous devrons donc créer une sorte d'enveloppe autour de `time.AfterFunc` qui prendra le temps d'exécution et le montant à imprimer pour que nous puissions les espionner.

## Écrivez d'abord le test

Ajoutez un nouveau test à notre suite

```go
t.Run("it schedules printing of blind values", func(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &poker.StubPlayerStore{}
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(playerStore, in, blindAlerter)
	cli.PlayPoker()

	if len(blindAlerter.alerts) != 1 {
		t.Fatal("expected a blind alert to be scheduled")
	}
})
```

Vous remarquerez que nous avons créé un `SpyBlindAlerter` que nous essayons d'injecter dans notre `CLI` puis nous vérifions qu'après avoir appelé `PlayPoker`, une alerte est programmée.

(Rappelez-vous que nous visons d'abord le scénario le plus simple, puis nous itérerons.)

Voici la définition de `SpyBlindAlerter`

```go
type SpyBlindAlerter struct {
	alerts []struct {
		scheduledAt time.Duration
		amount      int
	}
}

func (s *SpyBlindAlerter) ScheduleAlertAt(duration time.Duration, amount int) {
	s.alerts = append(s.alerts, struct {
		scheduledAt time.Duration
		amount      int
	}{duration, amount})
}

```


## Essayez d'exécuter le test

```
./CLI_test.go:32:27: too many arguments in call to poker.NewCLI
	have (*poker.StubPlayerStore, *strings.Reader, *SpyBlindAlerter)
	want (poker.PlayerStore, io.Reader)
```

## Écrivez le minimum de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Nous avons ajouté un nouvel argument et le compilateur se plaint. _Strictement parlant_, le minimum de code est de faire en sorte que `NewCLI` accepte un `*SpyBlindAlerter`, mais trompons-nous un peu et définissons simplement la dépendance comme une interface.

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}
```

Puis ajoutez-le au constructeur

```go
func NewCLI(store PlayerStore, in io.Reader, alerter BlindAlerter) *CLI
```

Vos autres tests échoueront maintenant car ils n'ont pas de `BlindAlerter` passé à `NewCLI`.

L'espionnage sur BlindAlerter n'est pas pertinent pour les autres tests, donc dans le fichier de test, ajoutez

```go
var dummySpyAlerter = &SpyBlindAlerter{}
```

Utilisez-le ensuite dans les autres tests pour résoudre les problèmes de compilation. En l'étiquetant comme "dummy", il est clair pour le lecteur du test qu'il n'est pas important.

[> Les objets dummy sont passés mais jamais réellement utilisés. Habituellement, ils sont juste utilisés pour remplir les listes de paramètres.](https://martinfowler.com/articles/mocksArentStubs.html)

Les tests devraient maintenant se compiler et notre nouveau test échoue.

```
=== RUN   TestCLI
=== RUN   TestCLI/it_schedules_printing_of_blind_values
--- FAIL: TestCLI (0.00s)
    --- FAIL: TestCLI/it_schedules_printing_of_blind_values (0.00s)
    	CLI_test.go:38: expected a blind alert to be scheduled
```

## Écrivez suffisamment de code pour le faire passer

Nous devrons ajouter le `BlindAlerter` comme champ sur notre `CLI` pour pouvoir y faire référence dans notre méthode `PlayPoker`.

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
	alerter     BlindAlerter
}

func NewCLI(store PlayerStore, in io.Reader, alerter BlindAlerter) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
		alerter:     alerter,
	}
}
```

Pour faire passer le test, nous pouvons appeler notre `BlindAlerter` avec ce que nous voulons

```go
func (cli *CLI) PlayPoker() {
	cli.alerter.ScheduleAlertAt(5*time.Second, 100)
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

Ensuite, nous voudrons vérifier qu'il programme toutes les alertes que nous espérons pour 5 joueurs

## Écrivez d'abord le test

```go
	t.Run("it schedules printing of blind values", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &poker.StubPlayerStore{}
		blindAlerter := &SpyBlindAlerter{}

		cli := poker.NewCLI(playerStore, in, blindAlerter)
		cli.PlayPoker()

		cases := []struct {
			expectedScheduleTime time.Duration
			expectedAmount       int
		}{
			{0 * time.Second, 100},
			{10 * time.Minute, 200},
			{20 * time.Minute, 300},
			{30 * time.Minute, 400},
			{40 * time.Minute, 500},
			{50 * time.Minute, 600},
			{60 * time.Minute, 800},
			{70 * time.Minute, 1000},
			{80 * time.Minute, 2000},
			{90 * time.Minute, 4000},
			{100 * time.Minute, 8000},
		}

		for i, c := range cases {
			t.Run(fmt.Sprintf("%d scheduled for %v", c.expectedAmount, c.expectedScheduleTime), func(t *testing.T) {

				if len(blindAlerter.alerts) <= i {
					t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
				}

				alert := blindAlerter.alerts[i]

				amountGot := alert.amount
				if amountGot != c.expectedAmount {
					t.Errorf("got amount %d, want %d", amountGot, c.expectedAmount)
				}

				gotScheduledTime := alert.scheduledAt
				if gotScheduledTime != c.expectedScheduleTime {
					t.Errorf("got scheduled time of %v, want %v", gotScheduledTime, c.expectedScheduleTime)
				}
			})
		}
	})
```

Les tests basés sur des tableaux fonctionnent bien ici et illustrent clairement ce que sont nos exigences. Nous parcourons le tableau et vérifions le `SpyBlindAlerter` pour voir si l'alerte a été programmée avec les bonnes valeurs.

## Essayez d'exécuter le test

Vous devriez avoir beaucoup d'échecs qui ressemblent à ceci

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_schedules_printing_of_blind_values
    --- FAIL: TestCLI/it_schedules_printing_of_blind_values (0.00s)
=== RUN   TestCLI/it_schedules_printing_of_blind_values/100_scheduled_for_0s
        --- FAIL: TestCLI/it_schedules_printing_of_blind_values/100_scheduled_for_0s (0.00s)
        	CLI_test.go:71: got scheduled time of 5s, want 0s
=== RUN   TestCLI/it_schedules_printing_of_blind_values/200_scheduled_for_10m0s
        --- FAIL: TestCLI/it_schedules_printing_of_blind_values/200_scheduled_for_10m0s (0.00s)
        	CLI_test.go:59: alert 1 was not scheduled [{5000000000 100}]
```

## Écrivez suffisamment de code pour le faire passer

```go
func (cli *CLI) PlayPoker() {

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + 10*time.Minute
	}

	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

Ce n'est pas beaucoup plus compliqué que ce que nous avions déjà. Nous itérons maintenant sur un tableau de `blinds` et appelons le planificateur sur un `blindTime` croissant

## Refactorisation

Nous pouvons encapsuler nos alertes programmées dans une méthode juste pour rendre la lecture de `PlayPoker` un peu plus claire.

```go
func (cli *CLI) PlayPoker() {
	cli.scheduleBlindAlerts()
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func (cli *CLI) scheduleBlindAlerts() {
	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + 10*time.Minute
	}
}
```

Enfin, nos tests semblent un peu encombrants. Nous avons deux structs anonymes représentant la même chose, une `ScheduledAlert`. Refactorisons cela en un nouveau type puis créons des helpers pour les comparer.

```go
type scheduledAlert struct {
	at     time.Duration
	amount int
}

func (s scheduledAlert) String() string {
	return fmt.Sprintf("%d chips at %v", s.amount, s.at)
}

type SpyBlindAlerter struct {
	alerts []scheduledAlert
}

func (s *SpyBlindAlerter) ScheduleAlertAt(at time.Duration, amount int) {
	s.alerts = append(s.alerts, scheduledAlert{at, amount})
}
```

Nous avons ajouté une méthode `String()` à notre type pour qu'il s'affiche joliment si le test échoue

Mettez à jour notre test pour utiliser notre nouveau type

```go
t.Run("it schedules printing of blind values", func(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &poker.StubPlayerStore{}
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(playerStore, in, blindAlerter)
	cli.PlayPoker()

	cases := []scheduledAlert{
		{0 * time.Second, 100},
		{10 * time.Minute, 200},
		{20 * time.Minute, 300},
		{30 * time.Minute, 400},
		{40 * time.Minute, 500},
		{50 * time.Minute, 600},
		{60 * time.Minute, 800},
		{70 * time.Minute, 1000},
		{80 * time.Minute, 2000},
		{90 * time.Minute, 4000},
		{100 * time.Minute, 8000},
	}

	for i, want := range cases {
		t.Run(fmt.Sprint(want), func(t *testing.T) {

			if len(blindAlerter.alerts) <= i {
				t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
			}

			got := blindAlerter.alerts[i]
			assertScheduledAlert(t, got, want)
		})
	}
})
```

Implémentez vous-même `assertScheduledAlert`.

Nous avons passé pas mal de temps ici à écrire des tests et avons été un peu vilains en ne nous intégrant pas à notre application. Résolvons ce problème avant d'ajouter d'autres exigences.

Essayez d'exécuter l'application et elle ne se compilera pas, se plaignant de ne pas avoir assez d'arguments pour `NewCLI`.

Créons une implémentation de `BlindAlerter` que nous pourrons utiliser dans notre application.

Créez `blind_alerter.go` et déplacez notre interface `BlindAlerter` et ajoutez les nouvelles choses ci-dessous

```go
package poker

import (
	"fmt"
	"os"
	"time"
)

type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}

type BlindAlerterFunc func(duration time.Duration, amount int)

func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int) {
	a(duration, amount)
}

func StdOutAlerter(duration time.Duration, amount int) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(os.Stdout, "Blind is now %d\n", amount)
	})
}
```

Rappelez-vous que n'importe quel _type_ peut implémenter une interface, pas seulement les `structs`. Si vous créez une bibliothèque qui expose une interface avec une fonction définie, il est courant d'exposer également un type `MyInterfaceFunc`.

Ce type sera une `func` qui implémentera également votre interface. De cette façon, les utilisateurs de votre interface ont la possibilité d'implémenter votre interface avec juste une fonction, plutôt que de devoir créer un type `struct` vide.

Nous créons ensuite la fonction `StdOutAlerter` qui a la même signature que la fonction et utilisons simplement `time.AfterFunc` pour la programmer pour imprimer sur `os.Stdout`.

Mettez à jour `main` où nous créons `NewCLI` pour voir cela en action

```go
poker.NewCLI(store, os.Stdin, poker.BlindAlerterFunc(poker.StdOutAlerter)).PlayPoker()
```

Avant d'exécuter, vous voudrez peut-être changer l'incrément de `blindTime` dans `CLI` à 10 secondes plutôt que 10 minutes juste pour que vous puissiez le voir en action.

Vous devriez voir l'impression des valeurs de blind comme nous l'espérions toutes les 10 secondes. Remarquez que vous pouvez toujours taper `Shaun wins` dans le CLI et il arrêtera le programme comme nous l'attendons.

Le jeu ne sera pas toujours joué avec 5 personnes, nous devons donc demander à l'utilisateur d'entrer un nombre de joueurs avant le début du jeu.

## Écrivez d'abord le test

Pour vérifier que nous demandons le nombre de joueurs, nous voudrons enregistrer ce qui est écrit sur StdOut. Nous l'avons fait plusieurs fois maintenant, nous savons que `os.Stdout` est un `io.Writer` donc nous pouvons vérifier ce qui est écrit si nous utilisons l'injection de dépendance pour passer un `bytes.Buffer` dans notre test et voir ce que notre code va écrire.

Nous ne nous soucions pas de nos autres collaborateurs dans ce test pour le moment, donc nous avons créé des doublures (dummies) dans notre fichier de test.

Nous devrions être un peu méfiants du fait que nous avons maintenant 4 dépendances pour `CLI`, cela semble peut-être commencer à avoir trop de responsabilités. Vivons avec pour l'instant et voyons si une refactorisation émerge au fur et à mesure que nous ajoutons cette nouvelle fonctionnalité.

```go
var dummyBlindAlerter = &SpyBlindAlerter{}
var dummyPlayerStore = &poker.StubPlayerStore{}
var dummyStdIn = &bytes.Buffer{}
var dummyStdOut = &bytes.Buffer{}
```

Voici notre nouveau test

```go
t.Run("it prompts the user to enter the number of players", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	cli := poker.NewCLI(dummyPlayerStore, dummyStdIn, stdout, dummyBlindAlerter)
	cli.PlayPoker()

	got := stdout.String()
	want := "Please enter the number of players: "

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
})
```

Nous passons ce qui sera `os.Stdout` dans `main` et voyons ce qui est écrit.

## Essayez d'exécuter le test

```
./CLI_test.go:38:27: too many arguments in call to poker.NewCLI
	have (*poker.StubPlayerStore, *bytes.Buffer, *bytes.Buffer, *SpyBlindAlerter)
	want (poker.PlayerStore, io.Reader, poker.BlindAlerter)
```

## Écrivez le minimum de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Nous avons une nouvelle dépendance, nous devrons donc mettre à jour `NewCLI`

```go
func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI
```

Maintenant, les _autres_ tests ne se compileront pas car ils n'ont pas d'`io.Writer` passé à `NewCLI`.

Ajoutez `dummyStdout` pour les autres tests.

Le nouveau test devrait échouer comme ceci

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players
    --- FAIL: TestCLI/it_prompts_the_user_to_enter_the_number_of_players (0.00s)
    	CLI_test.go:46: got '', want 'Please enter the number of players: '
FAIL
```

## Écrivez suffisamment de code pour le faire passer

Nous devons ajouter notre nouvelle dépendance à notre `CLI` pour pouvoir y faire référence dans `PlayPoker`

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
	out         io.Writer
	alerter     BlindAlerter
}

func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
		out:         out,
		alerter:     alerter,
	}
}
```

Enfin, nous pouvons écrire notre invite au début du jeu

```go
func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, "Please enter the number of players: ")
	cli.scheduleBlindAlerts()
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

## Refactorisation

Nous avons une chaîne dupliquée pour l'invite que nous devrions extraire en une constante

```go
const PlayerPrompt = "Please enter the number of players: "
```

Utilisez ceci dans le code de test et dans `CLI`.

Maintenant, nous devons envoyer un nombre et l'extraire. La seule façon de savoir si cela a eu l'effet désiré est de voir quelles alertes blind ont été programmées.

## Écrivez d'abord le test

```go
t.Run("it prompts the user to enter the number of players", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	in := strings.NewReader("7\n")
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(dummyPlayerStore, in, stdout, blindAlerter)
	cli.PlayPoker()

	got := stdout.String()
	want := poker.PlayerPrompt

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}

	cases := []scheduledAlert{
		{0 * time.Second, 100},
		{12 * time.Minute, 200},
		{24 * time.Minute, 300},
		{36 * time.Minute, 400},
	}

	for i, want := range cases {
		t.Run(fmt.Sprint(want), func(t *testing.T) {

			if len(blindAlerter.alerts) <= i {
				t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
			}

			got := blindAlerter.alerts[i]
			assertScheduledAlert(t, got, want)
		})
	}
})
```

Ouch ! Beaucoup de changements.

- Nous supprimons notre doublure pour StdIn et envoyons plutôt une version simulée représentant notre utilisateur entrant 7
- Nous supprimons également notre doublure sur le blind alerter afin de voir que le nombre de joueurs a eu un effet sur la planification
- Nous testons quelles alertes sont programmées

## Essayez d'exécuter le test

Le test devrait toujours se compiler et échouer en indiquant que les temps programmés sont incorrects car nous avons codé en dur pour que le jeu soit basé sur 5 joueurs

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players
    --- FAIL: TestCLI/it_prompts_the_user_to_enter_the_number_of_players (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players/100_chips_at_0s
        --- PASS: TestCLI/it_prompts_the_user_to_enter_the_number_of_players/100_chips_at_0s (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players/200_chips_at_12m0s
```

## Écrivez suffisamment de code pour le faire passer

Rappelez-vous, nous sommes libres de commettre tous les péchés dont nous avons besoin pour que cela fonctionne. Une fois que nous avons un logiciel fonctionnel, nous pouvons ensuite travailler sur la refactorisation du désordre que nous sommes sur le point de créer !

```go
func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, PlayerPrompt)
	numberOfPlayers, _ := strconv.Atoi(cli.readLine())

	cli.scheduleBlindAlerts(numberOfPlayers)

	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func (cli *CLI) scheduleBlindAlerts(numberOfPlayers int) {
	blindIncrement := time.Duration(5+numberOfPlayers) * time.Minute

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + blindIncrement
	}
}
```

- Nous lisons le `numberOfPlayersInput` dans une chaîne
- Nous utilisons `cli.readLine()` pour obtenir l'entrée de l'utilisateur, puis appelons `Atoi` pour la convertir en entier - en ignorant tous les scénarios d'erreur. Nous devrons écrire un test pour ce scénario plus tard.
- À partir de là, nous modifions `scheduleBlindAlerts` pour accepter un nombre de joueurs. Nous calculons ensuite un temps `blindIncrement` à utiliser pour ajouter à `blindTime` au fur et à mesure que nous itérons sur les montants de blind

Bien que notre nouveau test ait été corrigé, beaucoup d'autres ont échoué car maintenant notre système ne fonctionne que si le jeu commence par un utilisateur entrant un nombre. Vous devrez corriger les tests en changeant les entrées utilisateur de sorte qu'un nombre suivi d'un saut de ligne soit ajouté (cela met en évidence encore plus de défauts dans notre approche actuelle).

## Refactorisation

Tout cela semble un peu horrible, non ? **Écoutons nos tests**.

- Pour tester que nous programmons des alertes, nous avons configuré 4 dépendances différentes. Chaque fois que vous avez beaucoup de dépendances pour une _chose_ dans votre système, cela implique qu'elle fait trop. Visuellement, nous pouvons le voir à quel point notre test est encombré.
- Pour moi, il semble que **nous devons faire une abstraction plus claire entre la lecture de l'entrée utilisateur et la logique métier que nous voulons faire**
- Un meilleur test serait _étant donné cette entrée utilisateur, appelons-nous un nouveau type `Game` avec le bon nombre de joueurs_.
- Nous extrairions ensuite le test de la planification dans les tests pour notre nouveau `Game`.

Nous pouvons d'abord refactoriser vers notre `Game` et notre test devrait continuer à passer. Une fois que nous avons fait les changements structurels que nous voulons, nous pouvons réfléchir à la façon dont nous pouvons refactoriser les tests pour refléter notre nouvelle séparation des préoccupations

Rappelez-vous que lors des changements de refactorisation, essayez de les garder aussi petits que possible et continuez à réexécuter les tests.

Essayez-le vous-même d'abord. Réfléchissez aux limites de ce qu'un `Game` offrirait et de ce que notre `CLI` devrait faire.

Pour l'instant **ne changez pas** l'interface externe de `NewCLI` car nous ne voulons pas changer le code de test et le code client en même temps, c'est trop à jongler et nous pourrions finir par casser des choses.

Voici ce que j'ai proposé :

```go
// game.go
type Game struct {
	alerter BlindAlerter
	store   PlayerStore
}

func (p *Game) Start(numberOfPlayers int) {
	blindIncrement := time.Duration(5+numberOfPlayers) * time.Minute

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		p.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + blindIncrement
	}
}

func (p *Game) Finish(winner string) {
	p.store.RecordWin(winner)
}

// cli.go
type CLI struct {
	in   *bufio.Scanner
	out  io.Writer
	game *Game
}

func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI {
	return &CLI{
		in:  bufio.NewScanner(in),
		out: out,
		game: &Game{
			alerter: alerter,
			store:   store,
		},
	}
}

const PlayerPrompt = "Please enter the number of players: "

func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, PlayerPrompt)

	numberOfPlayersInput := cli.readLine()
	numberOfPlayers, _ := strconv.Atoi(strings.Trim(numberOfPlayersInput, "\n"))

	cli.game.Start(numberOfPlayers)

	winnerInput := cli.readLine()
	winner := extractWinner(winnerInput)

	cli.game.Finish(winner)
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins\n", "", 1)
}

func (cli *CLI) readLine() string {
	cli.in.Scan()
	return cli.in.Text()
}
```

Du point de vue du "domaine" :
- Nous voulons `Start` (démarrer) un `Game`, en indiquant combien de personnes jouent
- Nous voulons `Finish` (terminer) un `Game`, en déclarant le gagnant

Le nouveau type `Game` encapsule cela pour nous.

Avec ce changement, nous avons passé `BlindAlerter` et `PlayerStore` à `Game` car il est maintenant responsable des alertes et du stockage des résultats.

Notre `CLI` ne s'occupe maintenant que de :

- Construire `Game` avec ses dépendances existantes (que nous refactoriserons ensuite)
- Interpréter l'entrée utilisateur comme des invocations de méthode pour `Game`

Nous voulons essayer d'éviter de faire de "grandes" refactorisations qui nous laissent dans un état de tests défaillants pendant des périodes prolongées, car cela augmente les chances d'erreurs. (Si vous travaillez dans une équipe grande/distribuée, c'est encore plus important)

La première chose que nous ferons est de refactoriser `Game` pour que nous l'injectons dans `CLI`. Nous ferons les plus petits changements dans nos tests pour faciliter cela, puis nous verrons comment nous pouvons décomposer les tests dans les thèmes de l'analyse de l'entrée utilisateur et de la gestion du jeu.

Tout ce que nous devons faire maintenant est de changer `NewCLI`

```go
func NewCLI(in io.Reader, out io.Writer, game *Game) *CLI {
	return &CLI{
		in:   bufio.NewScanner(in),
		out:  out,
		game: game,
	}
}
```

Cela semble déjà être une amélioration. Nous avons moins de dépendances et _notre liste de dépendances reflète notre objectif global de conception_ du CLI concerné par l'entrée/sortie et déléguant des actions spécifiques au jeu à un `Game`.

Si vous essayez de compiler, il y a des problèmes. Vous devriez être capable de résoudre ces problèmes vous-même. Ne vous inquiétez pas de créer des mocks pour `Game` pour l'instant, initialisez simplement de _vrais_ `Game` juste pour que tout se compile et que les tests soient verts.

Pour ce faire, vous devrez faire un constructeur

```go
func NewGame(alerter BlindAlerter, store PlayerStore) *Game {
	return &Game{
		alerter: alerter,
		store:   store,
	}
}
```

Voici un exemple d'une des configurations pour les tests étant corrigée

```go
stdout := &bytes.Buffer{}
in := strings.NewReader("7\n")
blindAlerter := &SpyBlindAlerter{}
game := poker.NewGame(blindAlerter, dummyPlayerStore)

cli := poker.NewCLI(in, stdout, game)
cli.PlayPoker()
```

Il ne devrait pas falloir beaucoup d'efforts pour corriger les tests et revenir au vert (c'est le but !) mais assurez-vous de corriger également `main.go` avant la prochaine étape.

```go
// main.go
game := poker.NewGame(poker.BlindAlerterFunc(poker.StdOutAlerter), store)
cli := poker.NewCLI(os.Stdin, os.Stdout, game)
cli.PlayPoker()
```

Maintenant que nous avons extrait `Game`, nous devrions déplacer nos assertions spécifiques au jeu dans des tests séparés de CLI.

C'est juste un exercice de copie de nos tests `CLI` mais avec moins de dépendances

```go
func TestGame_Start(t *testing.T) {
	t.Run("schedules alerts on game start for 5 players", func(t *testing.T) {
		blindAlerter := &poker.SpyBlindAlerter{}
		game := poker.NewGame(blindAlerter, dummyPlayerStore)

		game.Start(5)

		cases := []poker.ScheduledAlert{
			{At: 0 * time.Second, Amount: 100},
			{At: 10 * time.Minute, Amount: 200},
			{At: 20 * time.Minute, Amount: 300},
			{At: 30 * time.Minute, Amount: 400},
			{At: 40 * time.Minute, Amount: 500},
			{At: 50 * time.Minute, Amount: 600},
			{At: 60 * time.Minute, Amount: 800},
			{At: 70 * time.Minute, Amount: 1000},
			{At: 80 * time.Minute, Amount: 2000},
			{At: 90 * time.Minute, Amount: 4000},
			{At: 100 * time.Minute, Amount: 8000},
		}

		checkSchedulingCases(cases, t, blindAlerter)
	})

	t.Run("schedules alerts on game start for 7 players", func(t *testing.T) {
		blindAlerter := &poker.SpyBlindAlerter{}
		game := poker.NewGame(blindAlerter, dummyPlayerStore)

		game.Start(7)

		cases := []poker.ScheduledAlert{
			{At: 0 * time.Second, Amount: 100},
			{At: 12 * time.Minute, Amount: 200},
			{At: 24 * time.Minute, Amount: 300},
			{At: 36 * time.Minute, Amount: 400},
		}

		checkSchedulingCases(cases, t, blindAlerter)
	})

}

func TestGame_Finish(t *testing.T) {
	store := &poker.StubPlayerStore{}
	game := poker.NewGame(dummyBlindAlerter, store)
	winner := "Ruth"

	game.Finish(winner)
	poker.AssertPlayerWin(t, store, winner)
}
```

L'intention derrière ce qui se passe lorsqu'un jeu de poker démarre est maintenant beaucoup plus claire.

Assurez-vous de déplacer également le test pour quand le jeu se termine.

Une fois que nous sommes satisfaits d'avoir déplacé les tests pour la logique du jeu, nous pouvons simplifier nos tests CLI afin qu'ils reflètent plus clairement nos responsabilités prévues

- Traiter l'entrée utilisateur et appeler les méthodes de `Game` au moment approprié
- Envoyer la sortie
- Crucialement, il ne connaît pas le fonctionnement réel des jeux

Pour ce faire, nous devrons faire en sorte que `CLI` ne dépende plus d'un type concret `Game` mais accepte une interface avec `Start(numberOfPlayers)` et `Finish(winner)`. Nous pouvons ensuite créer un espion de ce type et vérifier que les appels corrects sont effectués.

C'est ici que nous nous rendons compte que le nommage est parfois maladroit. Renommez `Game` en `TexasHoldem` (car c'est le _genre_ de jeu que nous jouons) et la nouvelle interface s'appellera `Game`. Cela reste fidèle à l'idée que notre CLI ne connaît pas le jeu réel que nous jouons et ce qui se passe lorsque vous `Start` et `Finish`.

```go
type Game interface {
	Start(numberOfPlayers int)
	Finish(winner string)
}
```

Remplacez toutes les références à `*Game` dans `CLI` et remplacez-les par `Game` (notre nouvelle interface). Comme toujours, continuez à réexécuter les tests pour vérifier que tout est vert pendant que nous refactorisons.

Maintenant que nous avons découplé `CLI` de `TexasHoldem`, nous pouvons utiliser des espions pour vérifier que `Start` et `Finish` sont appelés quand nous nous y attendons, avec les bons arguments.

Créez un espion qui implémente `Game`

```go
type GameSpy struct {
	StartedWith  int
	FinishedWith string
}

func (g *GameSpy) Start(numberOfPlayers int) {
	g.StartedWith = numberOfPlayers
}

func (g *GameSpy) Finish(winner string) {
	g.FinishedWith = winner
}
```

Remplacez tout test `CLI` qui teste une logique spécifique au jeu par des vérifications sur la façon dont notre `GameSpy` est appelé. Cela reflétera alors clairement les responsabilités du CLI dans nos tests.

Voici un exemple de l'un des tests étant corrigé ; essayez de faire le reste vous-même et consultez le code source si vous êtes coincé.

```go
	t.Run("it prompts the user to enter the number of players and starts the game", func(t *testing.T) {
		stdout := &bytes.Buffer{}
		in := strings.NewReader("7\n")
		game := &GameSpy{}

		cli := poker.NewCLI(in, stdout, game)
		cli.PlayPoker()

		gotPrompt := stdout.String()
		wantPrompt := poker.PlayerPrompt

		if gotPrompt != wantPrompt {
			t.Errorf("got %q, want %q", gotPrompt, wantPrompt)
		}

		if game.StartedWith != 7 {
			t.Errorf("wanted Start called with 7 but got %d", game.StartedWith)
		}
	})
```

Maintenant que nous avons une séparation claire des préoccupations, vérifier les cas limites autour de l'IO dans notre `CLI` devrait être plus facile.

Nous devons aborder le scénario où un utilisateur met une valeur non numérique lorsqu'on lui demande le nombre de joueurs :

Notre code ne devrait pas démarrer le jeu et il devrait afficher une erreur utile à l'utilisateur puis quitter.

## Écrivez d'abord le test

Nous commencerons par nous assurer que le jeu ne démarre pas

```go
t.Run("it prints an error when a non numeric value is entered and does not start the game", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	in := strings.NewReader("Pies\n")
	game := &GameSpy{}

	cli := poker.NewCLI(in, stdout, game)
	cli.PlayPoker()

	if game.StartCalled {
		t.Errorf("game should not have started")
	}
})
```

Vous devrez ajouter à notre `GameSpy` un champ `StartCalled` qui n'est défini que si `Start` est appelé

## Essayez d'exécuter le test
```
=== RUN   TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game
    --- FAIL: TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game (0.00s)
        CLI_test.go:62: game should not have started
```

## Écrivez suffisamment de code pour le faire passer

Là où nous appelons `Atoi`, nous devons juste vérifier l'erreur

```go
numberOfPlayers, err := strconv.Atoi(cli.readLine())

if err != nil {
	return
}
```

Ensuite, nous devons informer l'utilisateur de ce qu'il a fait de mal, donc nous allons affirmer ce qui est imprimé sur `stdout`.

## Écrivez d'abord le test

Nous avons déjà affirmé ce qui a été imprimé sur `stdout` avant, donc nous pouvons copier ce code pour l'instant

```go
gotPrompt := stdout.String()

wantPrompt := poker.PlayerPrompt + "you're so silly"

if gotPrompt != wantPrompt {
	t.Errorf("got %q, want %q", gotPrompt, wantPrompt)
}
```

Nous stockons _tout_ ce qui est écrit sur stdout, nous attendons donc toujours le `poker.PlayerPrompt`. Nous vérifions ensuite qu'une chose supplémentaire est imprimée. Nous ne sommes pas trop préoccupés par la formulation exacte pour l'instant, nous l'aborderons lors de la refactorisation.

## Essayez d'exécuter le test

```
=== RUN   TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game
    --- FAIL: TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game (0.00s)
        CLI_test.go:70: got 'Please enter the number of players: ', want 'Please enter the number of players: you're so silly'
```

## Écrivez suffisamment de code pour le faire passer

Changez le code de gestion des erreurs

```go
if err != nil {
	fmt.Fprint(cli.out, "you're so silly")
	return
}
```

## Refactorisation

Maintenant, refactorisez le message dans une constante comme `PlayerPrompt`

```go
wantPrompt := poker.PlayerPrompt + poker.BadPlayerInputErrMsg
```

et mettez un message plus approprié

```go
const BadPlayerInputErrMsg = "Bad value received for number of players, please try again with a number"
```

Enfin, nos tests autour de ce qui a été envoyé à `stdout` sont assez verbeux, écrivons une fonction d'assertion pour les nettoyer.

```go
func assertMessagesSentToUser(t testing.TB, stdout *bytes.Buffer, messages ...string) {
	t.Helper()
	want := strings.Join(messages, "")
	got := stdout.String()
	if got != want {
		t.Errorf("got %q sent to stdout but expected %+v", got, messages)
	}
}
```

L'utilisation de la syntaxe vararg (`...string`) est pratique ici car nous devons affirmer sur des quantités variables de messages.

Utilisez cet assistant dans les deux tests où nous affirmons sur les messages envoyés à l'utilisateur.

Il y a un certain nombre de tests qui pourraient être aidés avec des fonctions `assertX`, alors entraînez-vous à la refactorisation en nettoyant nos tests pour qu'ils se lisent bien.

Prenez le temps et réfléchissez à la valeur de certains des tests que nous avons mis en place. Rappelez-vous que nous ne voulons pas plus de tests que nécessaire, pouvez-vous en refactoriser/supprimer certains _et toujours être confiant que tout fonctionne_ ?

Voici ce que j'ai proposé

```go
func TestCLI(t *testing.T) {

	t.Run("start game with 3 players and finish game with 'Chris' as winner", func(t *testing.T) {
		game := &GameSpy{}
		stdout := &bytes.Buffer{}

		in := userSends("3", "Chris wins")
		cli := poker.NewCLI(in, stdout, game)

		cli.PlayPoker()

		assertMessagesSentToUser(t, stdout, poker.PlayerPrompt)
		assertGameStartedWith(t, game, 3)
		assertFinishCalledWith(t, game, "Chris")
	})

	t.Run("start game with 8 players and record 'Cleo' as winner", func(t *testing.T) {
		game := &GameSpy{}

		in := userSends("8", "Cleo wins")
		cli := poker.NewCLI(in, dummyStdOut, game)

		cli.PlayPoker()

		assertGameStartedWith(t, game, 8)
		assertFinishCalledWith(t, game, "Cleo")
	})

	t.Run("it prints an error when a non numeric value is entered and does not start the game", func(t *testing.T) {
		game := &GameSpy{}

		stdout := &bytes.Buffer{}
		in := userSends("pies")

		cli := poker.NewCLI(in, stdout, game)
		cli.PlayPoker()

		assertGameNotStarted(t, game)
		assertMessagesSentToUser(t, stdout, poker.PlayerPrompt, poker.BadPlayerInputErrMsg)
	})
}
```
Les tests reflètent maintenant les principales capacités du CLI, il est capable de lire l'entrée utilisateur en termes de nombre de personnes qui jouent et qui a gagné, et gère lorsqu'une mauvaise valeur est entrée pour le nombre de joueurs. En faisant cela, il est clair pour le lecteur ce que fait `CLI`, mais aussi ce qu'il ne fait pas.

Que se passe-t-il si au lieu de mettre `Ruth wins`, l'utilisateur met `Lloyd is a killer` ?

Terminez ce chapitre en écrivant un test pour ce scénario et en le faisant passer.

## Récapitulation

### Un rapide récapitulatif du projet

Au cours des 5 derniers chapitres, nous avons lentement TDD'd une bonne quantité de code

- Nous avons deux applications, une application en ligne de commande et un serveur web.
- Ces deux applications s'appuient sur un `PlayerStore` pour enregistrer les gagnants
- Le serveur web peut également afficher un classement de qui gagne le plus de jeux
- L'application en ligne de commande aide les joueurs à jouer à un jeu de poker en suivant la valeur actuelle de la blind.

### time.Afterfunc

Une façon très pratique de programmer un appel de fonction après une durée spécifique. Il vaut vraiment la peine de prendre le temps [de consulter la documentation pour `time`](https://golang.org/pkg/time/) car elle contient beaucoup de fonctions et méthodes qui vous feront gagner du temps.

Certaines de mes préférées sont

- `time.After(duration)` renvoie un `chan Time` quand la durée a expiré. Donc si vous souhaitez faire quelque chose _après_ un temps spécifique, cela peut vous aider.
- `time.NewTicker(duration)` renvoie un `Ticker` qui est similaire à ce qui précède en ce qu'il renvoie un canal mais celui-ci "tique" à chaque durée, plutôt qu'une seule fois. C'est très pratique si vous voulez exécuter du code toutes les `N durée`.

### Plus d'exemples de bonne séparation des préoccupations

_Généralement_, il est de bonne pratique de séparer les responsabilités de traitement des entrées et réponses de l'utilisateur du code de domaine. Vous le voyez ici dans notre application en ligne de commande et aussi dans notre serveur web.

Nos tests sont devenus désordonnés. Nous avions trop d'assertions (vérifier cette entrée, programmer ces alertes, etc.) et trop de dépendances. Nous pouvions visuellement voir qu'il était encombré ; il est **si important d'écouter vos tests**.

- Si vos tests semblent désordonnés, essayez de les refactoriser.
- Si vous avez fait cela et qu'ils sont toujours désordonnés, il est très probable que cela pointe vers un défaut dans votre conception
- C'est l'une des vraies forces des tests.

Même si les tests et le code de production étaient un peu encombrés, nous pouvions librement refactoriser soutenus par nos tests.

Rappelez-vous que lorsque vous vous trouvez dans ces situations, prenez toujours de petites mesures et réexécutez les tests après chaque changement.

Il aurait été dangereux de refactoriser à la fois le code de test _et_ le code de production en même temps, nous avons donc d'abord refactorisé le code de production (dans l'état actuel, nous ne pouvions pas améliorer beaucoup les tests) sans changer son interface pour pouvoir nous fier à nos tests autant que possible tout en changeant des choses. _Ensuite_, nous avons refactorisé les tests après l'amélioration de la conception.

Après la refactorisation, la liste des dépendances reflétait notre objectif de conception. C'est un autre avantage de l'injection de dépendances en ce qu'elle documente souvent l'intention. Lorsque vous vous appuyez sur des variables globales, les responsabilités deviennent très peu claires.

## Un exemple de fonction implémentant une interface

Lorsque vous définissez une interface avec une méthode, vous pourriez envisager de définir un type `MyInterfaceFunc` pour la compléter afin que les utilisateurs puissent implémenter votre interface avec juste une fonction.

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}

// BlindAlerterFunc vous permet d'implémenter BlindAlerter avec une fonction
type BlindAlerterFunc func(duration time.Duration, amount int)

// ScheduleAlertAt est l'implémentation BlindAlerterFunc de BlindAlerter
func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int) {
	a(duration, amount)
}
```

En faisant cela, les personnes utilisant votre bibliothèque peuvent implémenter votre interface avec juste une fonction. Ils peuvent utiliser la [conversion de type](https://go.dev/tour/basics/13) pour convertir leur fonction en un `BlindAlerterFunc` puis l'utiliser comme un BlindAlerter (car `BlindAlerterFunc` implémente `BlindAlerter`).

```go
game := poker.NewTexasHoldem(poker.BlindAlerterFunc(poker.StdOutAlerter), store)
```

Le point plus large ici est qu'en Go, vous pouvez ajouter des méthodes à des _types_, pas seulement à des structs. C'est une fonctionnalité très puissante, et vous pouvez l'utiliser pour implémenter des interfaces de manière plus pratique.

Considérez que vous pouvez non seulement définir des types de fonctions, mais aussi définir des types autour d'autres types, afin de pouvoir leur ajouter des méthodes.

```go
type Blog map[string]string

func (b Blog) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, b[r.URL.Path])
}
```

Ici, nous avons créé un gestionnaire HTTP qui implémente un "blog" très simple où il utilisera les chemins URL comme clés pour les publications stockées dans une map.
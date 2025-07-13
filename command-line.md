# Ligne de commande et structure de projet

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/command-line)**

Notre product owner souhaite maintenant faire un _pivot_ en introduisant une seconde application - une application en ligne de commande.

Pour l'instant, elle devra simplement être capable d'enregistrer la victoire d'un joueur lorsque l'utilisateur tape `Ruth wins`. L'objectif est qu'elle devienne éventuellement un outil pour aider les utilisateurs à jouer au poker.

Le product owner souhaite que la base de données soit partagée entre les deux applications afin que la ligue se mette à jour en fonction des victoires enregistrées dans la nouvelle application.

## Un rappel du code

Nous avons une application avec un fichier `main.go` qui lance un serveur HTTP. Le serveur HTTP ne nous intéressera pas pour cet exercice mais l'abstraction qu'il utilise, oui. Il dépend d'un `PlayerStore`.

```go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() League
}
```

Nous avons implémenté cela avec un `FileSystemPlayerStore` qui sauvegarde les données dans un fichier JSON.

## Quelques refactorisations de projet d'abord

Avant d'aller trop loin, nous devons faire une pause et réfléchir à la structure de notre projet. Jusqu'à présent, tout notre code était dans le package `main` qui était acceptable, mais maintenant que nous ajoutons une autre application, nous avons besoin de restructurer les choses.

Nous voulons partager le code `FileSystemPlayerStore` entre deux applications.

Créez un nouveau répertoire pour notre projet et créez quelques sous-répertoires :
- `cmd/cli` pour notre nouvelle application
- `cmd/webserver` pour notre application de serveur web existante
- `pkg/poker` pour le code que nous allons partager

La convention en Go est de mettre votre code d'application dans le répertoire `cmd` et le code que d'autres développeurs peuvent importer dans le répertoire `pkg`. Pour plus de détails sur la structure des projets, consultez [ce guide](https://github.com/golang-standards/project-layout).

Déplaçons notre serveur existant et la logique de stockage dans le répertoire `pkg/poker`. Ces codes ne devraient plus faire partie du package `main`, renommez-les en package `poker`.

```bash
mkdir -p cmd/{cli,webserver} pkg/poker
cp serveur.go pkg/poker
cp serveur_test.go pkg/poker
cp stockage_systeme_fichiers.go pkg/poker
cp stockage_systeme_fichiers_test.go pkg/poker
cp league.go pkg/poker
cp test_utilitaires.go pkg/poker
```

Ensuite, nous devrons mettre à jour les imports et le package dans ces fichiers. La partie facile est de renommer les packages ; la partie difficile est de s'assurer que tout est encore importé correctement (n'oubliez pas de renommer les fichiers _test pour utiliser le nouveau nom du package également). Il est préférable de mettre un nom simple correspondant au nom du répertoire, alors utilisons `poker`.

Une fois que vous avez déplacé le code et mis à jour les packages, vous pouvez maintenant vous concentrer sur l'application du serveur web en créant un petit fichier main dans `cmd/webserver/main.go`.

```go
package main

import (
	"log"
	"net/http"
	"os"
	"github.com/votrenom/poker"
)

const dbFileName = "game.db.json"

func main() {
	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	server := poker.NewPlayerServer(store)

	if err := http.ListenAndServe(":5000", server); err != nil {
		log.Fatalf("could not listen on port 5000 %v", err)
	}
}
```

Le code ne change pas beaucoup mais nous devons maintenant importer notre propre code depuis le nouveau package que nous avons créé.

### Vérifications finales

Vous devriez être capable de lancer le serveur web avec `go run cmd/webserver/main.go` et faire des requêtes curl comme avant pour vous assurer que tout fonctionne toujours.

Vous pouvez également exécuter les tests dans le répertoire `pkg/poker` pour vous assurer que nos tests fonctionnent toujours.

## Application en ligne de commande

Maintenant que nous avons restructuré notre code, ajoutons notre application en ligne de commande qui rencontrera les exigences suivantes :

> Pour l'instant, elle devra simplement être capable d'enregistrer la victoire d'un joueur lorsque l'utilisateur tape `Ruth wins`. L'intention est qu'elle devienne éventuellement un outil pour aider les utilisateurs à jouer au poker.

### Squelette de base

Créons d'abord un simple fichier main dans `cmd/cli/main.go`

```go
package main

import "fmt"

func main() {
	fmt.Println("Let's play poker")
}
```

Vous pouvez le lancer avec `go run cmd/cli/main.go` et il devrait afficher un message à l'écran. Nous avons notre squelette de base !

## Écrivons d'abord le test

Notre application aura une fonction que nous appellerons lorsqu'elle démarre, qui lira la ligne utilisateur et l'enverra à quelque chose pour l'enregistrer. Pour faciliter les tests, il serait préférable de ne pas intégrer réellement la lecture de l'entrée standard (stdin) dans notre logique d'application. Injectons plutôt le `io.Reader` de l'entrée standard dans notre application pour pouvoir la tester avec différentes entrées.

Créons un nouveau fichier `CLI_test.go` dans notre package `poker`.

```go
package poker_test

import (
	"strings"
	"testing"

	"github.com/votrenom/poker"
)

func TestCLI(t *testing.T) {
	playerStore := &poker.StubPlayerStore{}
	in := strings.NewReader("Chris wins\n")
	cli := poker.CLI{playerStore, in}
	cli.PlayPoker()

	if len(playerStore.WinCalls) != 1 {
		t.Fatal("expected a win call but didn't get any")
	}

	got := playerStore.WinCalls[0]
	want := "Chris"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

## Essayons d'exécuter le test

Nous devrons créer les types et les fonctions mentionnés dans le test.

## Écrivons la quantité minimale de code pour que le test s'exécute et vérifier la sortie du test échouant

Tout d'abord, créons notre type `CLI` dans un nouveau fichier `CLI.go` dans le package `poker`.

```go
package poker

import (
	"io"
)

type CLI struct {
	playerStore PlayerStore
	in          io.Reader
}

func (cli *CLI) PlayPoker() {}
```

La compilation échoue toujours car nous n'avons pas de stub `StubPlayerStore` dans notre fichier de test. C'est parce que nous avions des fichiers de test différents qui avaient chacun leur propre version de `StubPlayerStore` et maintenant nous avons mis tout dans un même package.

C'est l'une des irritations liées au développement Go. Si deux fichiers de test ont des helpers comme `StubPlayerStore`, ils ne peuvent pas le voir les uns les autres. Vous avez alors le choix entre :
- Déplacer ces helpers vers un fichier dans le package principal (et les exporter)
- Déplacer les helpers dans un fichier dans le package `xxx_test`.

Dans notre cas précédent, ce n'était pas trop un problème car nos tests étaient dans des fichiers distincts pour des packages distincts, mais maintenant que nous les avons tous réunis, nous devrions probablement résoudre ce problème. Nous reviendrons sur cette question plus tard, mais pour l'instant, ajoutons juste notre fonction d'aide de test dans le fichier principal.

## Écrivons suffisamment de code pour le faire passer

```go
func (cli *CLI) PlayPoker() {
	cli.playerStore.RecordWin("Chris")
}
```

Ce n'est pas la bonne logique, mais le test passe. Nous devons maintenant gérer les autres entrées possibles.

## Écrivons d'abord le test

```go
func TestCLI(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &poker.StubPlayerStore{}

	cli := &poker.CLI{playerStore, in}
	cli.PlayPoker()

	if len(playerStore.WinCalls) != 1 {
		t.Fatal("expected a win call but didn't get any")
	}

	got := playerStore.WinCalls[0]
	want := "Chris"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

Mais maintenant étendons-le pour gérer d'autres entrées.

```go
func TestCLI(t *testing.T) {

	t.Run("record chris win from user input", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &poker.StubPlayerStore{}

		cli := poker.CLI{playerStore, in}
		cli.PlayPoker()

		poker.AssertPlayerWin(t, playerStore, "Chris")
	})

	t.Run("record cleo win from user input", func(t *testing.T) {
		in := strings.NewReader("Cleo wins\n")
		playerStore := &poker.StubPlayerStore{}

		cli := poker.CLI{playerStore, in}
		cli.PlayPoker()

		poker.AssertPlayerWin(t, playerStore, "Cleo")
	})

}
```

## Essayons d'exécuter le test

```
--- FAIL: TestCLI (0.00s)
    --- FAIL: TestCLI/record_chris_win_from_user_input (0.00s)
        CLI_test.go:23: got [Chris] function calls, want [Chris]
    --- FAIL: TestCLI/record_cleo_win_from_user_input (0.00s)
        CLI_test.go:32: got [Chris] function calls, want [Cleo]
```

## Écrivons suffisamment de code pour le faire passer

Nous avons besoin de lire l'entrée de l'utilisateur et de l'analyser.

```go
func (cli *CLI) PlayPoker() {
	reader := bufio.NewReader(cli.in)
	line, _ := reader.ReadString('\n')
	cli.playerStore.RecordWin(extractWinner(line))
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins", "", 1)
}
```

Le test passe maintenant.

## Refactorison

Déplaçons notre fonction d'aide de test dans un fichier d'utilitaires :

```go
func AssertPlayerWin(t testing.TB, store *StubPlayerStore, winner string) {
	t.Helper()

	if len(store.WinCalls) != 1 {
		t.Fatalf("got %d calls to RecordWin want %d", len(store.WinCalls), 1)
	}

	if store.WinCalls[0] != winner {
		t.Errorf("got %q function calls, want %q", store.WinCalls[0], winner)
	}
}
```

Et utilisons cette fonction d'aide dans notre test :

```go
func TestCLI(t *testing.T) {

	t.Run("record chris win from user input", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &poker.StubPlayerStore{}

		cli := poker.CLI{playerStore, in}
		cli.PlayPoker()

		AssertPlayerWin(t, playerStore, "Chris")
	})

	t.Run("record cleo win from user input", func(t *testing.T) {
		in := strings.NewReader("Cleo wins\n")
		playerStore := &poker.StubPlayerStore{}

		cli := poker.CLI{playerStore, in}
		cli.PlayPoker()

		AssertPlayerWin(t, playerStore, "Cleo")
	})

}
```

## Écrivons d'abord le test

Notre application ne tient pas compte des entrées incorrectes. Ajoutons un test pour cela.

```go
func TestCLI(t *testing.T) {

	t.Run("record chris win from user input", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &StubPlayerStore{}

		cli := CLI{playerStore, in}
		cli.PlayPoker()

		AssertPlayerWin(t, playerStore, "Chris")
	})

	t.Run("record cleo win from user input", func(t *testing.T) {
		in := strings.NewReader("Cleo wins\n")
		playerStore := &StubPlayerStore{}

		cli := CLI{playerStore, in}
		cli.PlayPoker()

		AssertPlayerWin(t, playerStore, "Cleo")
	})

	t.Run("do not read beyond the first newline", func(t *testing.T) {
		in := strings.NewReader("Chris wins\nCleo wins\n")
		playerStore := &StubPlayerStore{}

		cli := CLI{playerStore, in}
		cli.PlayPoker()

		AssertPlayerWin(t, playerStore, "Chris")
	})
}
```

## Essayons d'exécuter le test

```
=== RUN   TestCLI
=== RUN   TestCLI/record_chris_win_from_user_input
=== RUN   TestCLI/record_cleo_win_from_user_input
=== RUN   TestCLI/do_not_read_beyond_the_first_newline
--- PASS: TestCLI (0.00s)
    --- PASS: TestCLI/record_chris_win_from_user_input (0.00s)
    --- PASS: TestCLI/record_cleo_win_from_user_input (0.00s)
    --- PASS: TestCLI/do_not_read_beyond_the_first_newline (0.00s)
```

Le test passe déjà, car notre `bufio.Reader.ReadString` ne lit que jusqu'au premier `\n`.

Maintenant, intégrons notre code dans la fonction `main` de notre application.

## Écrivons suffisamment de code pour le faire passer

Modifions notre structure `CLI` pour avoir un constructeur qui nous permet d'initialiser la structure facilement.

```go
type CLI struct {
	playerStore PlayerStore
	in          io.Reader
}

func NewCLI(playerStore PlayerStore, in io.Reader) *CLI {
	return &CLI{
		playerStore: playerStore,
		in:          in,
	}
}

func (cli *CLI) PlayPoker() {
	reader := bufio.NewReader(cli.in)
	line, _ := reader.ReadString('\n')
	cli.playerStore.RecordWin(extractWinner(line))
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins", "", 1)
}
```

Et maintenant nous pouvons utiliser ce constructeur dans notre fichier `main.go` :

```go
package main

import (
	"fmt"
	"github.com/votrenom/poker"
	"log"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	fmt.Println("Let's play poker")
	fmt.Println("Type {Name} wins to record a win")

	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	game := poker.NewCLI(store, os.Stdin)
	game.PlayPoker()
}
```

Si vous exécutez l'application maintenant avec `go run cmd/cli/main.go`, vous devriez pouvoir entrer un nom de joueur suivi de "wins" et cela enregistrera la victoire.

### Refactorison

Nos deux applications ont besoin d'un accès au `FileSystemStore`. Dans les deux cas, nous écrivons :

```go
db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

if err != nil {
	log.Fatalf("problem opening %s %v", dbFileName, err)
}

store, err := poker.NewFileSystemPlayerStore(db)

if err != nil {
	log.Fatalf("problem creating file system player store, %v ", err)
}
```

Nous pouvons extraire cette logique en une seule fonction.

```go
func FileSystemPlayerStoreFromFile(path string) (*FileSystemPlayerStore, func(), error) {
	db, err := os.OpenFile(path, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		return nil, nil, fmt.Errorf("problem opening %s %v", path, err)
	}

	closeFunc := func() {
		db.Close()
	}

	store, err := NewFileSystemPlayerStore(db)

	if err != nil {
		return nil, nil, fmt.Errorf("problem creating file system player store, %v ", err)
	}

	return store, closeFunc, nil
}
```

Cette fonction prend un chemin et renvoie :

- `*FileSystemPlayerStore`
- Une fonction pour fermer le fichier
- Toute erreur qui pourrait survenir dans le processus de création

C'est une bonne utilisation de l'idiome Go d'encapsuler la gestion des ressources et de renvoyer une fonction qui peut être utilisée pour fermer cette ressource.

Maintenant, nous pouvons simplifier notre application CLI :

```go
package main

import (
	"fmt"
	"log"
	"os"

	"github.com/votrenom/poker"
)

const dbFileName = "game.db.json"

func main() {
	store, close, err := poker.FileSystemPlayerStoreFromFile(dbFileName)

	if err != nil {
		log.Fatal(err)
	}
	defer close()

	fmt.Println("Let's play poker")
	fmt.Println("Type {Name} wins to record a win")

	poker.NewCLI(store, os.Stdin).PlayPoker()
}
```

Et notre application de serveur web :

```go
package main

import (
	"log"
	"net/http"

	"github.com/votrenom/poker"
)

const dbFileName = "game.db.json"

func main() {
	store, close, err := poker.FileSystemPlayerStoreFromFile(dbFileName)

	if err != nil {
		log.Fatal(err)
	}
	defer close()

	server := poker.NewPlayerServer(store)

	if err := http.ListenAndServe(":5000", server); err != nil {
		log.Fatalf("could not listen on port 5000 %v", err)
	}
}
```

## Conclusion

### Structure de package

Nous avons restructuré notre code en utilisant plusieurs packages pour faciliter le partage de code entre nos deux applications. Les avantages de cette approche incluent :

- Un seul endroit pour maintenir le code partagé
- Séparation des préoccupations
- Plus facile à tester
- Plus facile à travailler en équipe

Cela rend notre code plus modulaire et plus facile à étendre à l'avenir.

### Lecture des entrées utilisateur

L'entrée utilisateur est simplement de l'`io.Reader` et nous pouvons utiliser la bibliothèque standard Go pour la lire. Dans notre cas, nous avons utilisé `bufio.NewReader` pour lire une ligne de texte de l'entrée utilisateur.

### Des abstractions simples mènent à une réutilisation de code plus simple

En utilisant des interfaces simples comme `io.Reader` et `io.Writer`, nous pouvons facilement tester notre code sans avoir à utiliser des entrées/sorties réelles. Cela rend nos tests plus rapides, plus fiables et plus faciles à écrire.

Par exemple, notre `CLI` peut être testée en utilisant un `strings.Reader` au lieu de dépendre de l'entrée réelle de l'utilisateur. De même, notre `PlayerStore` abstrait la manière dont les données sont stockées, ce qui nous permet de l'implémenter de différentes manières (en mémoire, système de fichiers, base de données, etc.).
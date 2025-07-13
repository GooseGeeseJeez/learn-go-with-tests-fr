# E/S et tri

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/io)**

[Dans le chapitre précédent](json.md), nous avons continué à faire évoluer notre application en ajoutant un nouvel endpoint `/league`. En cours de route, nous avons appris à gérer le JSON, à intégrer des types et à faire du routage.

Notre *product owner* est quelque peu contrariée par le logiciel qui perd les scores lorsque le serveur est redémarré. Cela est dû au fait que notre implémentation du magasin est en mémoire vive (RAM). Elle n'est pas non plus satisfaite que nous n'ayons pas interprété que l'endpoint `/league` devrait renvoyer les joueurs classés par nombre de victoires !

## Le code jusqu'à présent

Voici notre interface `PlayerStore` :

```go
type PlayerStore interface {
    GetPlayerScore(name string) int
    RecordWin(name string)
    GetLeague() []Player
}
```

Et voici les types que nous utilisons pour `Player` et `PlayerServer` :

```go
type Player struct {
    Name string
    Wins int
}

type PlayerServer struct {
    store PlayerStore
    http.Handler
}
```

Dans ce chapitre, nous allons implémenter une version du `PlayerStore` qui écrit les données sur le disque et nous réglerons le problème de tri.

## Pérennité

Il y a de nombreux moyens d'implémenter un système de stockage persistant pour nos données. Nous pourrions utiliser une base de données comme Postgres ou MySQL, mais ce serait probablement exagéré pour nos besoins.

Un moyen simple de stocker des données est de les écrire dans un fichier et c'est ce que nous allons faire.

Le package [encoding/json](https://godoc.org/encoding/json) du standard Go comprend des fonctions de décalage qui facilitent l'encodage et le décodage des données.

[io.Writer](https://godoc.org/io#Writer) est l'interface utilisée dans l'ensemble de la bibliothèque standard pour écrire des données dans quelque chose. Par exemple, quand nous appelons `json.NewEncoder(writer)`, nous créons un `Encoder` qui peut écrire JSON dans `writer`. De même, il existe un `io.Reader` pour lire des données.

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

Donc nous savons qu'il existe une fonction pour encoder de la JSON et que cette fonction prend un `io.Writer`, mais comment écrivons-nous sur le disque ?

[os.File](https://godoc.org/os#File) implémente `io.Writer` (et aussi `io.Reader`), ce qui nous permet de manipuler des fichiers.

C'est une bonne occasion d'utiliser notre interface `PlayerStore` car nous pouvons faire une autre implémentation avec notre nouveau stockage basé sur des fichiers sans que notre serveur HTTP ait à savoir ce qui se passe.

Nous allons appeler cette implémentation `FileSystemPlayerStore`.

## Réfléchir à notre approche

Nous devons réfléchir à comment nous allons stocker les données et comment l'accès sera coordonné.

Nous utiliserons un fichier JSON par souci de simplicité, mais comment allons-nous stocker et récupérer les données du fichier ?

Nous pourrions :

- Tout lire à partir du disque chaque fois que nous en avons besoin.
- Avoir des données en mémoire et écrire dans le disque lorsqu'elles sont modifiées.

Le premier est plus simple mais très inefficace. Le second est un peu plus compliqué mais plus efficace. Nous utiliserons le second, mais nous implémenterons d'abord le premier pour ensuite refactoriser, de sorte que nous n'ayons pas à gérer trop de choses à la fois.

Nous allons aussi introduire une autre idée pour aider à tester le code avec les fichiers.

Comme expliqué, `os.File` implémente `io.Writer` et `io.Reader`. La majorité des fonctions qui manipulent des fichiers prennent des interfaces plutôt que des types concrets. C'est pourquoi si nous voulons passer un `os.File` à une autre fonction, nous pouvons simplement passer un `io.Reader` ou un `io.Writer`.

De même, si nous écrivons une fonction qui lit un fichier, nous n'avons pas besoin que notre fonction accepte un `os.File`, elle devrait probablement accepter un `io.Reader`.

Cette idée rend les tests beaucoup plus simples car nous n'avons pas à nous soucier de créer des fichiers réels. Tant que nous pouvons passer un type qui peut correspondre à l'interface que nous voulons, nous pouvons simplement envoyer des données de test au lieu d'avoir à créer des fichiers réels.

## Écrivons d'abord le test

Notre approche sera donc :

- Création d'un fichier de test dans lequel nous stockerons les scores initiaux.
- Création du type `FileSystemPlayerStore` avec une fonction d'initialisation qui prendra le fichier système pour stocker les données.
- Implémentation de `PlayerStore`.

```go
func TestFileSystemStore(t *testing.T) {
    t.Run("league from a reader", func(t *testing.T) {
        database := strings.NewReader(`[
            {"Name": "Cleo", "Wins": 10},
            {"Name": "Chris", "Wins": 33}]`)

        store := NewFileSystemPlayerStore(database)

        got := store.GetLeague()

        want := []Player{
            {"Cleo", 10},
            {"Chris", 33},
        }

        assertLeague(t, got, want)
    })
}
```

Nous utilisons `strings.NewReader` qui retourne un `io.Reader` à partir d'une chaîne de caractères, qui contient notre base de données JSON souhaitée. Nous le passons ensuite à une fonction `NewFileSystemPlayerStore` que nous devrons créer. Ensuite, nous vérifions que la fonction `GetLeague` retourne le même que celui qui se trouve dans notre base de données.

Avant de continuer, nous devons implémenter une fonction helper `assertLeague`. Nous aurions pu la copier à partir de notre test précédent mais il est plus pratique de la réutiliser en la déplaçant dans un fichier test helper.

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "net/http/httptest"
    "reflect"
    "testing"
)

type StubPlayerStore struct {
    scores   map[string]int
    winCalls []string
    league   []Player
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
    score := s.scores[name]
    return score
}

func (s *StubPlayerStore) RecordWin(name string) {
    s.winCalls = append(s.winCalls, name)
}

func (s *StubPlayerStore) GetLeague() []Player {
    return s.league
}

func AssertPlayerWin(t testing.TB, store *StubPlayerStore, winner string) {
    t.Helper()

    if len(store.winCalls) != 1 {
        t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
    }

    if store.winCalls[0] != winner {
        t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], winner)
    }
}

func AssertScoreEquals(t testing.TB, got, want int) {
    t.Helper()
    if got != want {
        t.Errorf("got %d want %d", got, want)
    }
}

func AssertResponseBody(t testing.TB, got, want string) {
    t.Helper()
    if got != want {
        t.Errorf("response body is wrong, got %q want %q", got, want)
    }
}

func AssertStatus(t testing.TB, got, want int) {
    t.Helper()
    if got != want {
        t.Errorf("did not get correct status, got %d, want %d", got, want)
    }
}

func AssertLeague(t testing.TB, got, want []Player) {
    t.Helper()
    if !reflect.DeepEqual(got, want) {
        t.Errorf("got %v want %v", got, want)
    }
}

func AssertContentType(t testing.TB, response *httptest.ResponseRecorder, want string) {
    t.Helper()
    if response.Result().Header.Get("content-type") != want {
        t.Errorf("response did not have content-type of %s, got %v", want, response.Result().Header)
    }
}

func GetLeagueFromResponse(t testing.TB, body io.Reader) (league []Player) {
    t.Helper()
    err := json.NewDecoder(body).Decode(&league)

    if err != nil {
        t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", body, err)
    }

    return
}

func NewLeagueRequest() *http.Request {
    req, _ := http.NewRequest(http.MethodGet, "/league", nil)
    return req
}

func NewGetScoreRequest(name string) *http.Request {
    req, _ := http.NewRequest(http.MethodGet, fmt.Sprintf("/players/%s", name), nil)
    return req
}

func NewPostWinRequest(name string) *http.Request {
    req, _ := http.NewRequest(http.MethodPost, fmt.Sprintf("/players/%s", name), nil)
    return req
}
```

Quelques changements ont été apportés :

- Le fichier s'appelle maintenant `testing.go` et non `server_test.go` car nous voulons réutiliser ces helpers dans d'autres tests.
- Nous avons changé le package en `main` car c'est le package que nous utilisons et vous ne pouvez pas utiliser les fonctions d'un autre package dans votre test.
- Nous avons exporté les fonctions et le type `StubPlayerStore` en renommant les fonctions avec un nom en majuscules.

## Essayons d'exécuter le test

```
# github.com/quii/learn-go-with-tests/io/v1
./file_system_store_test.go:15:12: undefined: NewFileSystemPlayerStore
```

## Écrivons suffisamment de code pour le faire fonctionner

Créez un nouveau fichier appelé `file_system_store.go` et ajoutez le code suivant :

```go
package main

import (
    "io"
)

func NewFileSystemPlayerStore(database io.Reader) *FileSystemPlayerStore {
    return &FileSystemPlayerStore{}
}

type FileSystemPlayerStore struct {
}

func (f *FileSystemPlayerStore) GetLeague() []Player {
    return nil
}
```

Essayons à nouveau :

```
--- FAIL: TestFileSystemStore (0.00s)
    --- FAIL: TestFileSystemStore/league_from_a_reader (0.00s)
        file_system_store_test.go:21: got [] want [{Cleo 10} {Chris 33}]
```

Nous avons besoin d'analyser la base de données JSON à partir du lecteur et la renvoyer quand on appelle `GetLeague`.

```go
package main

import (
    "encoding/json"
    "io"
)

func NewFileSystemPlayerStore(database io.Reader) *FileSystemPlayerStore {
    league, _ := NewLeague(database)
    return &FileSystemPlayerStore{
        league,
    }
}

type FileSystemPlayerStore struct {
    league []Player
}

func (f *FileSystemPlayerStore) GetLeague() []Player {
    return f.league
}

func NewLeague(rdr io.Reader) ([]Player, error) {
    var league []Player
    err := json.NewDecoder(rdr).Decode(&league)
    if err != nil {
        err = fmt.Errorf("problem parsing league, %v", err)
    }

    return league, err
}
```

En plus d'implémenter `GetLeague()`, nous avons créé une fonction utilitaire `NewLeague` qui contiendra la logique pour analyser notre base de données. Nous avons séparé ces responsabilités pour qu'elles soient plus faciles à tester.

Passons maintenant aux autres fonctions dans `PlayerStore` :

```go
func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {
    return 0
}

func (f *FileSystemPlayerStore) RecordWin(name string) {
}
```

Bien que ces méthodes vides devraient suffire pour que notre premier test passe, essayons de construire sur ce que nous avons pour terminer notre implémentation.

## Écrivons d'abord le test

```go
t.Run("get player score", func(t *testing.T) {
    database := strings.NewReader(`[
        {"Name": "Cleo", "Wins": 10},
        {"Name": "Chris", "Wins": 33}]`)

    store := NewFileSystemPlayerStore(database)

    got := store.GetPlayerScore("Chris")
    want := 33
    assertScoreEquals(t, got, want)
})
```

## Essayons d'exécuter le test

```
=== RUN   TestFileSystemStore/get_player_score
    --- FAIL: TestFileSystemStore/get_player_score (0.00s)
        file_system_store_test.go:34: got 0 want 33
```

## Écrivons suffisamment de code pour le faire passer

Nous devons parcourir notre "ligue" pour trouver le joueur et renvoyer ses scores.

```go
func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {
    for _, player := range f.league {
        if player.Name == name {
            return player.Wins
        }
    }
    return 0
}
```

## Écrivons d'abord le test

Enfin, nous devons pouvoir enregistrer des victoires.

```go
t.Run("store wins for existing players", func(t *testing.T) {
    database := strings.NewReader(`[
        {"Name": "Cleo", "Wins": 10},
        {"Name": "Chris", "Wins": 33}]`)

    store := NewFileSystemPlayerStore(database)

    store.RecordWin("Chris")

    got := store.GetPlayerScore("Chris")
    want := 34
    assertScoreEquals(t, got, want)
})
```

## Essayons d'exécuter le test

```
=== RUN   TestFileSystemStore/store_wins_for_existing_players
    --- FAIL: TestFileSystemStore/store_wins_for_existing_players (0.00s)
        file_system_store_test.go:47: got 33 want 34
```

## Écrivons suffisamment de code pour le faire passer

```go
func (f *FileSystemPlayerStore) RecordWin(name string) {
    for i, player := range f.league {
        if player.Name == name {
            f.league[i].Wins++
        }
    }
}
```

Notre test passe maintenant mais il y a quelques problèmes avec ce que nous avons fait :

- Notre store n'écrit pas les données sur le disque à la fin de chaque opération, donc les données ne persistent pas.
- Nous ne gérons pas le cas où un nouveau joueur est ajouté à la ligue.

Je pense qu'on peut ignorer le point sur la persistance pour l'instant, mais nous devons gérer le cas où un joueur n'existe pas dans la ligue. Continuons à itérer sur notre solution.

## Écrivons d'abord le test

```go
t.Run("store wins for new players", func(t *testing.T) {
    database := strings.NewReader(`[
        {"Name": "Cleo", "Wins": 10},
        {"Name": "Chris", "Wins": 33}]`)

    store := NewFileSystemPlayerStore(database)

    store.RecordWin("Pepper")

    got := store.GetPlayerScore("Pepper")
    want := 1
    assertScoreEquals(t, got, want)
})
```

## Essayons d'exécuter le test

```
=== RUN   TestFileSystemStore/store_wins_for_new_players
    --- FAIL: TestFileSystemStore/store_wins_for_new_players (0.00s)
        file_system_store_test.go:60: got 0 want 1
```

## Écrivons suffisamment de code pour le faire passer

```go
func (f *FileSystemPlayerStore) RecordWin(name string) {
    for i, player := range f.league {
        if player.Name == name {
            f.league[i].Wins++
            return
        }
    }

    f.league = append(f.league, Player{name, 1})
}
```

## Refactorisation

Une option à considérer est d'ajouter une méthode à `Player` pour incrémenter ses victoires. Mais cette fois, pour notre refactorisation, nous examinerons plutôt comment nous pouvons rendre `FileSystemStore` plus robuste.

Bien que les tests passent pour le moment, notre implémentation n'écrit rien sur le disque. Cependant, nous avons implémenté la logique requise pour le reste de notre application et nous pouvons maintenant nous concentrer sur la persistance.

Nous avons également vu notre code mesurer les joueurs, ce qui pourrait être réutilisé par rapport à la tâche de tri dont nous devons également nous occuper.

Pour éviter d'en faire trop à la fois, reprenons l'approche initiale qui consistait à lire à partir du disque à chaque appel de méthode, puis à le refactoriser plus tard pour qu'il soit plus efficace.

Notre `FileSystemStore` ne devrait pas se soucier de la création de fichiers, il devrait se concentrer sur la lecture et l'écriture des données. Il serait logique de créer une interface pour encapsuler les opérations d'E/S que nous souhaitons effectuer pour que notre magasin puisse s'en servir.

L'interface `io.ReadWriteSeeker` regroupe `io.Reader`, `io.Writer` et `io.Seeker` (facultatif). Cela signifie que nous pouvons découpler notre code des détails d'implémentation de la façon dont les données sont stockées (en mémoire, sur le disque, etc.) et nous donne également la flexibilité de rechercher dans le fichier quand nous en avons besoin.

```go
type ReadWriteSeeker interface {
    Reader
    Writer
    Seeker
}
```

Repensons à ce que nous voulons faire :

- Pour `GetLeague` et `GetPlayerScore`, nous devons rembobiner le lecteur jusqu'au début (en utilisant Seek), analyser les données et ensuite renvoyer les données.
- Pour `RecordWin`, nous devons faire comme ci-dessus, mais ensuite écrire un nouveau JSON avec les données mises à jour.

Mettons à jour nos tests pour refléter cela. Nous nous attendons à ce que si nous avons un `ReadWriteSeeker` avec des données valides, tout doit fonctionner comme prévu.

```go
func TestFileSystemStore(t *testing.T) {

    t.Run("league sorted", func(t *testing.T) {
        database, cleanDatabase := createTempFile(t, `[
            {"Name": "Cleo", "Wins": 10},
            {"Name": "Chris", "Wins": 33}]`)
        defer cleanDatabase()

        store := NewFileSystemPlayerStore(database)

        got := store.GetLeague()

        want := []Player{
            {"Chris", 33},
            {"Cleo", 10},
        }

        assertLeague(t, got, want)

        // read again to ensure data is not lost
        got = store.GetLeague()
        assertLeague(t, got, want)
    })

    t.Run("get player score", func(t *testing.T) {
        database, cleanDatabase := createTempFile(t, `[
            {"Name": "Cleo", "Wins": 10},
            {"Name": "Chris", "Wins": 33}]`)
        defer cleanDatabase()

        store := NewFileSystemPlayerStore(database)

        got := store.GetPlayerScore("Chris")
        want := 33
        assertScoreEquals(t, got, want)
    })

    t.Run("store wins for existing players", func(t *testing.T) {
        database, cleanDatabase := createTempFile(t, `[
            {"Name": "Cleo", "Wins": 10},
            {"Name": "Chris", "Wins": 33}]`)
        defer cleanDatabase()

        store := NewFileSystemPlayerStore(database)

        store.RecordWin("Chris")

        got := store.GetPlayerScore("Chris")
        want := 34
        assertScoreEquals(t, got, want)
    })

    t.Run("store wins for new players", func(t *testing.T) {
        database, cleanDatabase := createTempFile(t, `[
            {"Name": "Cleo", "Wins": 10},
            {"Name": "Chris", "Wins": 33}]`)
        defer cleanDatabase()

        store := NewFileSystemPlayerStore(database)

        store.RecordWin("Pepper")

        got := store.GetPlayerScore("Pepper")
        want := 1
        assertScoreEquals(t, got, want)
    })

    t.Run("league sorted", func(t *testing.T) {
        database, cleanDatabase := createTempFile(t, `[
            {"Name": "Cleo", "Wins": 10},
            {"Name": "Chris", "Wins": 33}]`)
        defer cleanDatabase()

        store := NewFileSystemPlayerStore(database)

        got := store.GetLeague()

        want := []Player{
            {"Chris", 33},
            {"Cleo", 10},
        }

        assertLeague(t, got, want)

        // read again to ensure data is not lost
        got = store.GetLeague()
        assertLeague(t, got, want)
    })
}
```

Remarquez également que nous avons introduit une fonction `createTempFile` pour créer un véritable fichier temporaire pour tester. Elle renverra un `*os.File` qui implémente `ReadWriteSeeker` et une fonction de nettoyage pour que nous puissions nettoyer après nos tests.

```go
func createTempFile(t testing.TB, initialData string) (*os.File, func()) {
    t.Helper()

    tmpfile, err := ioutil.TempFile("", "db")

    if err != nil {
        t.Fatalf("could not create temp file %v", err)
    }

    tmpfile.Write([]byte(initialData))

    removeFile := func() {
        tmpfile.Close()
        os.Remove(tmpfile.Name())
    }

    return tmpfile, removeFile
}
```

Cela semble être un changement considérable, mais tout ce que nous faisons est de créer un fichier temporaire avec des données et de le passer à notre magasin. Nous utilisons `defer cleanDatabase()` pour nous assurer que le fichier est supprimé après le test.

Si vous exécutez les tests, ils échoueront car notre code ne lit pas et n'écrit pas dans le magasin correctement.

Pour que cela fonctionne, nous avons besoin de modifier notre implémentation :

```go
type FileSystemPlayerStore struct {
    database io.ReadWriteSeeker
    league   League
}

func NewFileSystemPlayerStore(database io.ReadWriteSeeker) *FileSystemPlayerStore {
    database.Seek(0, 0)
    league, _ := NewLeague(database)
    return &FileSystemPlayerStore{
        database: database,
        league:   league,
    }
}

func (f *FileSystemPlayerStore) GetLeague() League {
    f.database.Seek(0, 0)
    league, _ := NewLeague(f.database)
    return league
}

func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {
    player := f.GetLeague().Find(name)

    if player != nil {
        return player.Wins
    }

    return 0
}

func (f *FileSystemPlayerStore) RecordWin(name string) {
    league := f.GetLeague()
    player := league.Find(name)

    if player != nil {
        player.Wins++
    } else {
        league = append(league, Player{name, 1})
    }

    f.database.Seek(0, 0)
    json.NewEncoder(f.database).Encode(league)
}
```

Remarquez comment nous avons utilisé la composition sur notre type `League` pour implémenter la fonctionnalité `Find`. Ajoutons cette fonctionnalité :

```go
type League []Player

func (l League) Find(name string) *Player {
    for i, p := range l {
        if p.Name == name {
            return &l[i]
        }
    }
    return nil
}
```

Notez que nous renvoyons un pointeur vers le joueur réel, de sorte que lorsque nous incrémentons leurs victoires, cela modifie la ligue que nous renvoyons et que nous encodons plus tard dans `RecordWin`.

Essayez d'exécuter les tests et ils devraient passer !

## Refactorisation

Notre première approche consistant à réécrire le fichier à chaque appel à `RecordWin` est loin d'être idéale, mais elle est fonctionnelle et nous pouvons refactoriser cela.

Nous ne traitons pas non plus le besoin de trier la ligue. Occupons-nous d'abord de cela, car c'est plus simple.

Puisque nous avons notre `League` sous forme de type, nous pouvons ajouter une méthode de tri :

```go
func (l League) Sort() {
    sort.Slice(l, func(i, j int) bool {
        return l[i].Wins > l[j].Wins
    })
}
```

Maintenant nous pouvons mettre à jour notre test pour vérifier que les joueurs sont triés par nombre de victoires :

```go
t.Run("league sorted", func(t *testing.T) {
    database, cleanDatabase := createTempFile(t, `[
        {"Name": "Cleo", "Wins": 10},
        {"Name": "Chris", "Wins": 33}]`)
    defer cleanDatabase()

    store := NewFileSystemPlayerStore(database)

    got := store.GetLeague()

    want := []Player{
        {"Chris", 33},
        {"Cleo", 10},
    }

    assertLeague(t, got, want)

    // read again to ensure data is not lost
    got = store.GetLeague()
    assertLeague(t, got, want)
})
```

Il échouera car les données ne sont pas triées. Mettons à jour `GetLeague` :

```go
func (f *FileSystemPlayerStore) GetLeague() League {
    f.database.Seek(0, 0)
    league, _ := NewLeague(f.database)
    league.Sort()
    return league
}
```

Maintenant que nous avons résolu le problème de tri, passons à la refactorisation de notre magasin.

Notre première version a un problème de performance évident : à chaque appel d'une méthode, nous lisons le fichier entier. Nous devrions plutôt stocker la dernière version de la ligue en mémoire pour éviter ces lectures répétées. Avec cette approche, il nous suffit de lire le fichier une seule fois lorsque nous créons le magasin, puis de maintenir le contenu à jour en mémoire. Nous n'écrivons dans le fichier qu'en cas de modification des données.

Voici la solution refactorisée :

```go
type FileSystemPlayerStore struct {
    database io.ReadWriteSeeker
    league   League
}

func NewFileSystemPlayerStore(database io.ReadWriteSeeker) *FileSystemPlayerStore {
    database.Seek(0, 0)
    league, _ := NewLeague(database)
    return &FileSystemPlayerStore{
        database: database,
        league:   league,
    }
}

func (f *FileSystemPlayerStore) GetLeague() League {
    sort.Slice(f.league, func(i, j int) bool {
        return f.league[i].Wins > f.league[j].Wins
    })
    return f.league
}

func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {
    player := f.league.Find(name)

    if player != nil {
        return player.Wins
    }

    return 0
}

func (f *FileSystemPlayerStore) RecordWin(name string) {
    player := f.league.Find(name)

    if player != nil {
        player.Wins++
    } else {
        f.league = append(f.league, Player{name, 1})
    }

    f.database.Seek(0, 0)
    json.NewEncoder(f.database).Encode(f.league)
}
```

En réalité, nous avions déjà presque cela dans notre implémentation précédente, nous avons simplement besoin d'enlever les appels redondants pour lire le fichier dans `GetLeague` et `GetPlayerScore`.

## Le problème de fichier tronqué

Notre code actuel fonctionne mais il y a un problème subtil. Quand nous écrivons dans le fichier et que la longueur du nouveau fichier est inférieure à la longueur du fichier existant, les octets supplémentaires ne sont pas effacés. Cela peut conduire à un fichier JSON invalide.

Par exemple, si notre fichier contient initialement `[{"Name": "Chris", "Wins": 33}]` et que nous écrivons `[{"Name": "Cleo", "Wins": 10}]`, le fichier résultant contiendrait `[{"Name": "Cleo", "Wins": 10}]}]` car les derniers caractères du fichier précédent ne seraient pas écrasés.

Pour résoudre ce problème, nous devons tronquer le fichier avant d'y écrire. Nous pouvons utiliser la méthode `Truncate` de `os.File` pour cela :

```go
func (f *FileSystemPlayerStore) RecordWin(name string) {
    player := f.league.Find(name)

    if player != nil {
        player.Wins++
    } else {
        f.league = append(f.league, Player{name, 1})
    }

    f.database.Seek(0, 0)
    f.database.Truncate(0)
    json.NewEncoder(f.database).Encode(f.league)
}
```

Cependant, cette méthode n'est pas disponible sur notre interface `io.ReadWriteSeeker`. Nous devons donc adapter notre approche.

Nous pourrions transformer notre fonction `FileSystemPlayerStore` pour n'accepter que les `*os.File`, mais cela rendrait nos tests plus difficiles à écrire, car nous devrions toujours créer des fichiers réels.

Une meilleure approche est de créer une nouvelle interface qui étend `ReadWriteSeeker` avec les méthodes supplémentaires dont nous avons besoin :

```go
type FileSystemPlayerStoreFunc interface {
    io.ReadWriteSeeker
    Truncate(size int64) error
}
```

Maintenant nous pouvons mettre à jour notre code pour utiliser cette interface :

```go
func NewFileSystemPlayerStore(file FileSystemPlayerStoreFunc) (*FileSystemPlayerStore, error) {
    file.Seek(0, 0)
    
    info, err := file.Stat()

    if err != nil {
        return nil, fmt.Errorf("problem getting file info from file %s, %v", file.Name(), err)
    }

    if info.Size() == 0 {
        file.Write([]byte("[]"))
        file.Seek(0, 0)
    }

    league, err := NewLeague(file)

    if err != nil {
        return nil, fmt.Errorf("problem loading player store from file %s, %v", file.Name(), err)
    }
    
    return &FileSystemPlayerStore{
        database: file,
        league:   league,
    }, nil
}
```

Nous avons ajouté une vérification pour créer un fichier vide si nécessaire, et nous gérons également les erreurs correctement.

Maintenant, nous devons mettre à jour notre fonction `RecordWin` pour utiliser `Truncate` :

```go
func (f *FileSystemPlayerStore) RecordWin(name string) {
    player := f.league.Find(name)

    if player != nil {
        player.Wins++
    } else {
        f.league = append(f.league, Player{name, 1})
    }

    f.database.Seek(0, 0)
    f.database.(FileSystemPlayerStoreFunc).Truncate(0)
    json.NewEncoder(f.database).Encode(f.league)
}
```

Nous utilisons une assertion de type pour convertir notre `database` en `FileSystemPlayerStoreFunc` afin de pouvoir appeler `Truncate`.

## Intégration avec notre serveur

Maintenant que nous avons une implémentation fonctionnelle de notre magasin qui écrit dans un fichier, nous devons l'intégrer à notre serveur.

Commençons par créer une fonction principale pour notre jeu :

```go
func main() {
    db, err := os.OpenFile("game.db.json", os.O_RDWR|os.O_CREATE, 0666)

    if err != nil {
        log.Fatalf("problem opening %s %v", dbFileName, err)
    }

    store, err := NewFileSystemPlayerStore(db)

    if err != nil {
        log.Fatalf("problem creating file system player store, %v ", err)
    }

    server := NewPlayerServer(store)

    if err := http.ListenAndServe(":5000", server); err != nil {
        log.Fatalf("could not listen on port 5000 %v", err)
    }
}
```

Cela crée un fichier pour notre base de données (ou l'ouvre s'il existe déjà), crée notre magasin et notre serveur, puis commence à écouter le trafic HTTP.

## Conclusion

Dans ce chapitre, nous avons :

- Créé une implémentation de stockage sur le système de fichiers pour notre application
- Appris à utiliser `io.Reader`, `io.Writer`, et `io.Seeker` pour travailler avec des fichiers
- Utilisé la composition pour créer des types plus complexes et ajouter des fonctionnalités
- Utilisé l'encapsulation pour cacher les détails d'implémentation et rendre notre code plus testable
- Refactorisé notre code pour le rendre plus performant tout en maintenant sa fonctionnalité
- Appris à trier des données en Go
- Résolu le problème de la troncature des fichiers lors de l'écriture

Notre application peut maintenant stocker les scores des joueurs dans un fichier, les récupérer et afficher une ligue triée par nombre de victoires. Dans le prochain chapitre, nous ajouterons une interface en ligne de commande à notre application pour la rendre plus conviviale.
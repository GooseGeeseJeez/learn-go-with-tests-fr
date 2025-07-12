# JSON, routage et intégration

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/json)**

[Dans le chapitre précédent](http-server.md), nous avons créé un serveur web pour stocker le nombre de parties gagnées par les joueurs.

Notre product owner a une nouvelle exigence : avoir un nouvel endpoint appelé `/league` qui renvoie une liste de tous les joueurs stockés. Elle souhaite que cette liste soit renvoyée au format JSON.

## Voici le code que nous avons jusqu'à présent

```go
// server.go
package main

import (
	"fmt"
	"net/http"
	"strings"
)

type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
}

type PlayerServer struct {
	store PlayerStore
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, player)
	case http.MethodGet:
		p.showScore(w, player)
	}
}

func (p *PlayerServer) showScore(w http.ResponseWriter, player string) {
	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}

func (p *PlayerServer) processWin(w http.ResponseWriter, player string) {
	p.store.RecordWin(player)
	w.WriteHeader(http.StatusAccepted)
}
```

```go
// in_memory_player_store.go
package main

func NewInMemoryPlayerStore() *InMemoryPlayerStore {
	return &InMemoryPlayerStore{map[string]int{}}
}

type InMemoryPlayerStore struct {
	store map[string]int
}

func (i *InMemoryPlayerStore) RecordWin(name string) {
	i.store[name]++
}

func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return i.store[name]
}

```

```go
// main.go
package main

import (
	"log"
	"net/http"
)

func main() {
	server := &PlayerServer{NewInMemoryPlayerStore()}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

You can find the corresponding tests in the link at the top of the chapter.

We'll start by making the league table endpoint.

## Écrivons d'abord le test

Nous allons étendre la suite de tests existante car nous disposons déjà de fonctions de test utiles et d'un faux `PlayerStore` à utiliser.

```go
//server_test.go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := &PlayerServer{&store}

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

Avant de nous préoccuper des scores réels et du JSON, nous essaierons de garder les changements petits avec l'intention d'itérer vers notre objectif. Le début le plus simple est de vérifier que nous pouvons atteindre `/league` et obtenir un `OK` en retour.

## Essayons d'exécuter le test

```
    --- FAIL: TestLeague/it_returns_200_on_/league (0.00s)
        server_test.go:101: status code is wrong: got 404, want 200
FAIL
FAIL	playerstore	0.221s
FAIL
```

Notre `PlayerServer` renvoie une erreur `404 Not Found`, comme si nous essayions d'obtenir les victoires d'un joueur inconnu. En examinant comment `server.go` implémente `ServeHTTP`, nous constatons qu'il suppose toujours être appelé avec une URL pointant vers un joueur spécifique :

```go
player := strings.TrimPrefix(r.URL.Path, "/players/")
```

Dans le chapitre précédent, nous avons mentionné que c'était une manière assez naïve de faire notre routage. Notre test nous informe correctement que nous avons besoin d'un concept pour gérer différents chemins de requête.

## Écrivons suffisamment de code pour le faire passer

Go dispose d'un mécanisme de routage intégré appelé [`ServeMux`](https://golang.org/pkg/net/http/#ServeMux) (multiplexeur de requêtes) qui vous permet d'attacher des `http.Handler` à des chemins de requête particuliers.

Commettons quelques péchés et faisons passer les tests de la manière la plus rapide possible, sachant que nous pourrons le refactoriser en toute sécurité une fois que nous saurons que les tests passent.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	router := http.NewServeMux()

	router.Handle("/league", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	router.Handle("/players/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		player := strings.TrimPrefix(r.URL.Path, "/players/")

		switch r.Method {
		case http.MethodPost:
			p.processWin(w, player)
		case http.MethodGet:
			p.showScore(w, player)
		}
	}))

	router.ServeHTTP(w, r)
}
```

- Lorsque la requête commence, nous créons un routeur puis nous lui indiquons pour le chemin `x` d'utiliser le gestionnaire `y`.
- Ainsi, pour notre nouvel endpoint, nous utilisons `http.HandlerFunc` et une _fonction anonyme_ pour appeler `w.WriteHeader(http.StatusOK)` lorsque `/league` est demandé afin de faire passer notre nouveau test.
- Pour la route `/players/`, nous coupons et collons simplement notre code dans un autre `http.HandlerFunc`.
- Enfin, nous traitons la requête qui est arrivée en appelant le `ServeHTTP` de notre nouveau routeur (remarquez comment `ServeMux` est _aussi_ un `http.Handler` ?)

Les tests devraient maintenant passer.

## Refactorisation

`ServeHTTP` commence à être assez volumineux, nous pouvons séparer les choses en refactorisant nos gestionnaires en méthodes distinctes.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	router.ServeHTTP(w, r)
}

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}

func (p *PlayerServer) playersHandler(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, player)
	case http.MethodGet:
		p.showScore(w, player)
	}
}
```

Il est assez étrange (et inefficace) de configurer un routeur à chaque arrivée d'une requête, puis de l'appeler. Ce que nous voulons idéalement faire, c'est avoir une sorte de fonction `NewPlayerServer` qui prendra nos dépendances et effectuera la configuration unique de création du routeur. Chaque requête pourra alors simplement utiliser cette unique instance du routeur.

```go
//server.go
type PlayerServer struct {
	store  PlayerStore
	router *http.ServeMux
}

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := &PlayerServer{
		store,
		http.NewServeMux(),
	}

	p.router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	p.router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	return p
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	p.router.ServeHTTP(w, r)
}
```

- `PlayerServer` a maintenant besoin de stocker un routeur.
- Nous avons déplacé la création du routage hors de `ServeHTTP` et dans notre `NewPlayerServer` afin que cela ne soit fait qu'une seule fois, et non à chaque requête.
- Vous devrez mettre à jour tout le code de test et de production où nous utilisions `PlayerServer{&store}` avec `NewPlayerServer(&store)`.

### Une dernière refactorisation

Essayez de modifier le code comme suit.

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := new(PlayerServer)

	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	p.Handler = router

	return p
}
```

Ensuite, remplacez `server := &PlayerServer{&store}` par `server := NewPlayerServer(&store)` dans `server_test.go`, `server_integration_test.go` et `main.go`.

Enfin, assurez-vous de **supprimer** `func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request)` car elle n'est plus nécessaire !

## Intégration (Embedding)

Nous avons changé la deuxième propriété de `PlayerServer`, en supprimant la propriété nommée `router http.ServeMux` et en la remplaçant par `http.Handler` ; c'est ce qu'on appelle _l'intégration_ (embedding).

> Go ne fournit pas la notion typique d'héritage basée sur les types, mais il a la capacité d'« emprunter » des morceaux d'une implémentation en intégrant des types au sein d'une structure ou d'une interface.

[Effective Go - Embedding](https://golang.org/doc/effective_go.html#embedding)

Cela signifie que notre `PlayerServer` possède maintenant toutes les méthodes que `http.Handler` possède, à savoir uniquement `ServeHTTP`.

Pour "remplir" le `http.Handler`, nous lui assignons le `router` que nous créons dans `NewPlayerServer`. Nous pouvons faire cela parce que `http.ServeMux` possède la méthode `ServeHTTP`.

Cela nous permet de supprimer notre propre méthode `ServeHTTP`, car nous en exposons déjà une via le type intégré.

L'intégration est une fonctionnalité de langage très intéressante. Vous pouvez l'utiliser avec des interfaces pour composer de nouvelles interfaces.

```go
type Animal interface {
	Eater
	Sleeper
}
```

Et vous pouvez également l'utiliser avec des types concrets, pas seulement des interfaces. Comme vous pouvez vous y attendre, si vous intégrez un type concret, vous aurez accès à toutes ses méthodes et champs publics.

### Des inconvénients ?

Vous devez être prudent avec l'intégration des types car vous exposerez toutes les méthodes et champs publics du type que vous intégrez. Dans notre cas, c'est acceptable car nous avons intégré uniquement _l'interface_ que nous voulions exposer (`http.Handler`).

Si nous avions été paresseux et que nous avions intégré `http.ServeMux` à la place (le type concret), cela fonctionnerait toujours _mais_ les utilisateurs de `PlayerServer` pourraient ajouter de nouvelles routes à notre serveur car `Handle(path, handler)` serait public.

**Lors de l'intégration de types, réfléchissez vraiment à l'impact que cela a sur votre API publique.**

C'est une erreur _très_ courante de mal utiliser l'intégration et de finir par polluer vos API et exposer les parties internes de votre type.

Maintenant que nous avons restructuré notre application, nous pouvons facilement ajouter de nouvelles routes et avoir le début de l'endpoint `/league`. Nous devons maintenant lui faire renvoyer des informations utiles.

Nous devrions renvoyer du JSON qui ressemble à quelque chose comme ceci.

```json
[
   {
      "Name":"Bill",
      "Wins":10
   },
   {
      "Name":"Alice",
      "Wins":15
   }
]
```

## Écrivons d'abord le test

Nous allons commencer par essayer d'analyser la réponse pour en tirer quelque chose de significatif.

```go
//server_test.go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := NewPlayerServer(&store)

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		var got []Player

		err := json.NewDecoder(response.Body).Decode(&got)

		if err != nil {
			t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", response.Body, err)
		}

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

### Pourquoi ne pas tester la chaîne JSON ?

On pourrait argumenter qu'une étape initiale plus simple serait simplement de vérifier que le corps de la réponse contient une chaîne JSON particulière.

D'après mon expérience, les tests qui vérifient des chaînes JSON présentent les problèmes suivants :

- *Fragilité*. Si vous modifiez le modèle de données, vos tests échoueront.
- *Difficile à déboguer*. Il peut être délicat de comprendre quel est le problème réel lors de la comparaison de deux chaînes JSON.
- *Intention peu claire*. Bien que la sortie doive être du JSON, ce qui est vraiment important, c'est exactement quelles sont les données, plutôt que la façon dont elles sont encodées.
- *Re-tester la bibliothèque standard*. Il n'est pas nécessaire de tester comment la bibliothèque standard produit du JSON, elle est déjà testée. Ne testez pas le code des autres.

Au lieu de cela, nous devrions analyser le JSON dans des structures de données pertinentes pour nos tests.

### Modélisation des données

Étant donné le modèle de données JSON, il semble que nous ayons besoin d'un tableau de `Player` avec certains champs, nous avons donc créé un nouveau type pour représenter cela.

```go
//server.go
type Player struct {
	Name string
	Wins int
}
```

### Décodage JSON

```go
//server_test.go
var got []Player
err := json.NewDecoder(response.Body).Decode(&got)
```

Pour analyser le JSON dans notre modèle de données, nous créons un `Decoder` à partir du package `encoding/json` puis appelons sa méthode `Decode`. Pour créer un `Decoder`, il a besoin d'un `io.Reader` pour lire, qui dans notre cas est le `Body` de notre espion de réponse.

`Decode` prend l'adresse de l'objet dans lequel nous essayons de décoder, c'est pourquoi nous déclarons une tranche vide de `Player` à la ligne précédente.

L'analyse du JSON peut échouer, donc `Decode` peut renvoyer une `error`. Il est inutile de continuer le test si cela échoue, nous vérifions donc l'erreur et arrêtons le test avec `t.Fatalf` si elle se produit. Remarquez que nous affichons le corps de la réponse avec l'erreur, car il est important pour la personne qui exécute le test de voir quelle chaîne ne peut pas être analysée.

## Essayons d'exécuter le test

```
=== RUN   TestLeague/it_returns_200_on_/league
    --- FAIL: TestLeague/it_returns_200_on_/league (0.00s)
        server_test.go:107: Unable to parse response from server '' into slice of Player, 'unexpected end of JSON input'
```

Notre endpoint ne renvoie actuellement pas de corps, il ne peut donc pas être analysé en JSON.

## Écrivons suffisamment de code pour le faire passer

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	leagueTable := []Player{
		{"Chris", 20},
	}

	json.NewEncoder(w).Encode(leagueTable)

	w.WriteHeader(http.StatusOK)
}
```

Le test passe maintenant.

### Encodage et décodage

Remarquez la belle symétrie dans la bibliothèque standard.

- Pour créer un `Encoder`, vous avez besoin d'un `io.Writer`, ce que `http.ResponseWriter` implémente.
- Pour créer un `Decoder`, vous avez besoin d'un `io.Reader`, ce que le champ `Body` de notre espion de réponse implémente.

Tout au long de ce livre, nous avons utilisé `io.Writer` et c'est une autre démonstration de sa prévalence dans la bibliothèque standard et de la façon dont de nombreuses bibliothèques fonctionnent facilement avec.

## Refactorisation

Il serait bon d'introduire une séparation des préoccupations entre notre gestionnaire et l'obtention du `leagueTable`, car nous savons que nous n'allons pas le coder en dur très bientôt.

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(p.getLeagueTable())
	w.WriteHeader(http.StatusOK)
}

func (p *PlayerServer) getLeagueTable() []Player {
	return []Player{
		{"Chris", 20},
	}
}
```

Ensuite, nous voudrons étendre notre test afin de pouvoir contrôler exactement quelles données nous voulons récupérer.

## Écrivons d'abord le test

Nous pouvons mettre à jour le test pour vérifier que la table de ligue contient des joueurs que nous allons simuler dans notre store.

Mettons à jour `StubPlayerStore` pour lui permettre de stocker une ligue, qui n'est qu'une tranche de `Player`. Nous y stockerons nos données attendues.

```go
//server_test.go
type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
	league   []Player
}
```

Ensuite, mettons à jour notre test actuel en mettant quelques *players* dans la proprété *league* de notre stub et assurons-nous qu'ils soient reçus de notre serveur.

```go
//server_test.go
func TestLeague(t *testing.T) {

	t.Run("it returns the league table as JSON", func(t *testing.T) {
		wantedLeague := []Player{
			{"Cleo", 32},
			{"Chris", 20},
			{"Tiest", 14},
		}

		store := StubPlayerStore{nil, nil, wantedLeague}
		server := NewPlayerServer(&store)

		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		var got []Player

		err := json.NewDecoder(response.Body).Decode(&got)

		if err != nil {
			t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", response.Body, err)
		}

		assertStatus(t, response.Code, http.StatusOK)

		if !reflect.DeepEqual(got, wantedLeague) {
			t.Errorf("got %v want %v", got, wantedLeague)
		}
	})
}
```

## Essayons d'exécuter le test

```
./server_test.go:33:3: too few values in struct initializer
./server_test.go:70:3: too few values in struct initializer
```

## Écrivons le minimum de code pour que le test s'exécute et vérifions la sortie du test en échec

Vous devrez mettre à jour les autres tests car nous avons un nouveau champ dans `StubPlayerStore` ; définissez-le à nil pour les autres tests.

Essayez d'exécuter à nouveau les tests et vous devriez obtenir

```
=== RUN   TestLeague/it_returns_the_league_table_as_JSON
    --- FAIL: TestLeague/it_returns_the_league_table_as_JSON (0.00s)
        server_test.go:124: got [{Chris 20}] want [{Cleo 32} {Chris 20} {Tiest 14}]
```

## Écrivons suffisamment de code pour le faire passer

Nous savons que les données se trouvent dans notre `StubPlayerStore` et nous les avons abstraites dans une interface `PlayerStore`. Nous devons mettre à jour cela afin que quiconque nous passe un `PlayerStore` puisse nous fournir les données pour les ligues.

```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() []Player
}
```

Maintenant, nous pouvons mettre à jour notre code de gestionnaire pour appeler cette méthode plutôt que de renvoyer une liste codée en dur. Supprimons notre méthode `getLeagueTable()` puis mettons à jour `leagueHandler` pour appeler `GetLeague()`.

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(p.store.GetLeague())
	w.WriteHeader(http.StatusOK)
}
```

Try and run the tests.

```
# github.com/quii/learn-go-with-tests/json-and-io/v4
./main.go:9:50: cannot use NewInMemoryPlayerStore() (type *InMemoryPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *InMemoryPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_integration_test.go:11:27: cannot use store (type *InMemoryPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *InMemoryPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:36:28: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:74:28: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:106:29: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
```

The compiler is complaining because `InMemoryPlayerStore` and `StubPlayerStore` do not have the new method we added to our interface.

For `StubPlayerStore` it's pretty easy, just return the `league` field we added earlier.

```go
//server_test.go
func (s *StubPlayerStore) GetLeague() []Player {
	return s.league
}
```

Here's a reminder of how `InMemoryStore` is implemented.

```go
//in_memory_player_store.go
type InMemoryPlayerStore struct {
	store map[string]int
}
```

Bien qu'il serait assez simple d'implémenter `GetLeague` "correctement" en itérant sur la map, rappelez-vous que nous essayons simplement d'_écrire la quantité minimale de code pour faire passer les tests_.

Alors, contentons-nous de satisfaire le compilateur pour l'instant et vivons avec le sentiment inconfortable d'une implémentation incomplète dans notre `InMemoryStore`.

```go
//in_memory_player_store.go
func (i *InMemoryPlayerStore) GetLeague() []Player {
	return nil
}
```

Ce que cela nous dit vraiment, c'est que _plus tard_, nous allons vouloir tester cela, mais mettons cela de côté pour l'instant.

Essayez d'exécuter les tests, le compilateur devrait être satisfait et les tests devraient passer !

## Refactorisation

Le code de test ne transmet pas très bien notre intention et contient beaucoup de code standard que nous pouvons refactoriser.

```go
//server_test.go
t.Run("it returns the league table as JSON", func(t *testing.T) {
	wantedLeague := []Player{
		{"Cleo", 32},
		{"Chris", 20},
		{"Tiest", 14},
	}

	store := StubPlayerStore{nil, nil, wantedLeague}
	server := NewPlayerServer(&store)

	request := newLeagueRequest()
	response := httptest.NewRecorder()

	server.ServeHTTP(response, request)

	got := getLeagueFromResponse(t, response.Body)
	assertStatus(t, response.Code, http.StatusOK)
	assertLeague(t, got, wantedLeague)
})
```

Here are the new helpers

```go
//server_test.go
func getLeagueFromResponse(t testing.TB, body io.Reader) (league []Player) {
	t.Helper()
	err := json.NewDecoder(body).Decode(&league)

	if err != nil {
		t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", body, err)
	}

	return
}

func assertLeague(t testing.TB, got, want []Player) {
	t.Helper()
	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}

func newLeagueRequest() *http.Request {
	req, _ := http.NewRequest(http.MethodGet, "/league", nil)
	return req
}
```

Une dernière chose que nous devons faire pour que notre serveur fonctionne est de nous assurer que nous renvoyons un en-tête `content-type` dans la réponse afin que les machines puissent reconnaître que nous renvoyons du `JSON`.

## Écrivons d'abord le test

Add this assertion to the existing test

```go
//server_test.go
if response.Result().Header.Get("content-type") != "application/json" {
	t.Errorf("response did not have content-type of application/json, got %v", response.Result().Header)
}
```

## Essayons d'exécuter le test

```
=== RUN   TestLeague/it_returns_the_league_table_as_JSON
    --- FAIL: TestLeague/it_returns_the_league_table_as_JSON (0.00s)
        server_test.go:124: response did not have content-type of application/json, got map[Content-Type:[text/plain; charset=utf-8]]
```

## Écrivons suffisamment de code pour le faire passer

Mettons à jour `leagueHandler`

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", "application/json")
	json.NewEncoder(w).Encode(p.store.GetLeague())
}
```

The test should pass.

## Refactorisation

Créons une constante pour "application/json" et utilisons-la dans `leagueHandler`

```go
//server.go
const jsonContentType = "application/json"

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", jsonContentType)
	json.NewEncoder(w).Encode(p.store.GetLeague())
}
```

Then add a helper for `assertContentType`.

```go
//server_test.go
func assertContentType(t testing.TB, response *httptest.ResponseRecorder, want string) {
	t.Helper()
	if response.Result().Header.Get("content-type") != want {
		t.Errorf("response did not have content-type of %s, got %v", want, response.Result().Header)
	}
}
```

Use it in the test.

```go
//server_test.go
assertContentType(t, response, jsonContentType)
```

Now that we have sorted out `PlayerServer` for now we can turn our attention to `InMemoryPlayerStore` because right now if we tried to demo this to the product owner `/league` will not work.

La façon la plus rapide d'obtenir une certaine confiance est d'ajouter à notre test d'intégration : nous pouvons atteindre le nouvel endpoint et vérifier que nous obtenons la bonne réponse de `/league`.

## Écrivons d'abord le test

Nous pouvons utiliser `t.Run` pour décomposer un peu ce test et nous pouvons réutiliser les helpers de nos tests de serveur - montrant encore une fois l'importance de refactoriser les tests.

```go
//server_integration_test.go
func TestRecordingWinsAndRetrievingThem(t *testing.T) {
	store := NewInMemoryPlayerStore()
	server := NewPlayerServer(store)
	player := "Pepper"

	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))

	t.Run("get score", func(t *testing.T) {
		response := httptest.NewRecorder()
		server.ServeHTTP(response, newGetScoreRequest(player))
		assertStatus(t, response.Code, http.StatusOK)

		assertResponseBody(t, response.Body.String(), "3")
	})

	t.Run("get league", func(t *testing.T) {
		response := httptest.NewRecorder()
		server.ServeHTTP(response, newLeagueRequest())
		assertStatus(t, response.Code, http.StatusOK)

		got := getLeagueFromResponse(t, response.Body)
		want := []Player{
			{"Pepper", 3},
		}
		assertLeague(t, got, want)
	})
}
```

## Essayons d'exécuter le test

```
=== RUN   TestRecordingWinsAndRetrievingThem/get_league
    --- FAIL: TestRecordingWinsAndRetrievingThem/get_league (0.00s)
        server_integration_test.go:35: got [] want [{Pepper 3}]
```

## Écrivons suffisamment de code pour le faire passer

`InMemoryPlayerStore` renvoie `nil` lorsque vous appelez `GetLeague()`, nous devrons donc corriger cela.

```go
//in_memory_player_store.go
func (i *InMemoryPlayerStore) GetLeague() []Player {
	var league []Player
	for name, wins := range i.store {
		league = append(league, Player{name, wins})
	}
	return league
}
```

Tout ce que nous avons à faire est d'itérer sur la map et de convertir chaque clé/valeur en `Player`.

Le test devrait maintenant passer.

## Conclusion

Nous avons continué à itérer en toute sécurité sur notre programme en utilisant le TDD, en lui permettant de prendre en charge de nouveaux endpoints de manière maintenable avec un routeur et il peut maintenant renvoyer du JSON pour nos consommateurs. Dans le chapitre suivant, nous aborderons la persistance des données et le tri de notre ligue.

Ce que nous avons couvert :

- **Routage**. La bibliothèque standard vous offre un type facile à utiliser pour le routage. Elle adopte pleinement l'interface `http.Handler` en ce sens que vous assignez des routes à des `Handler`s et le routeur lui-même est également un `Handler`. Elle ne dispose cependant pas de certaines fonctionnalités auxquelles vous pourriez vous attendre, comme les variables de chemin (par exemple `/users/{id}`). Vous pouvez facilement analyser cette information vous-même, mais vous voudrez peut-être envisager d'utiliser d'autres bibliothèques de routage si cela devient une contrainte. La plupart des bibliothèques populaires respectent la philosophie de la bibliothèque standard en implémentant également `http.Handler`.
- **Intégration de types**. Nous avons abordé un peu cette technique, mais vous pouvez [en apprendre davantage dans Effective Go](https://golang.org/doc/effective_go.html#embedding). S'il y a une chose à retenir, c'est que cette technique peut être extrêmement utile, mais _pensez toujours à votre API publique, n'exposez que ce qui est approprié_.
- **Sérialisation et désérialisation JSON**. La bibliothèque standard rend très simple la sérialisation et la désérialisation de vos données. Elle est également ouverte à la configuration et vous pouvez personnaliser le fonctionnement de ces transformations de données si nécessaire.

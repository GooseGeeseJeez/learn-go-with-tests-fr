# Serveur HTTP

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/http-server)**

On vous a demandé de créer un serveur web où les utilisateurs peuvent suivre combien de parties les joueurs ont gagnées.

-   `GET /players/{name}` devrait renvoyer un nombre indiquant le nombre total de victoires
-   `POST /players/{name}` devrait enregistrer une victoire pour ce nom, incrémentant à chaque `POST` ultérieur

Nous suivrons l'approche TDD, obtenant un logiciel fonctionnel aussi rapidement que possible puis faisant de petites améliorations itératives jusqu'à avoir la solution. En adoptant cette approche nous :

-   Gardons l'espace du problème restreint à tout moment
-   N'allons pas dans des impasses
-   Si jamais nous sommes bloqués/perdus, faire un retour arrière ne perdrait pas beaucoup de travail.

## Rouge, vert, refactoriser

Tout au long de ce livre, nous avons mis l'accent sur le processus TDD d'écrire un test et le regarder échouer (rouge), écrire la _quantité minimale_ de code pour le faire fonctionner (vert) puis refactoriser.

Cette discipline d'écrire la quantité minimale de code est importante en termes de sécurité que le TDD vous procure. Vous devriez vous efforcer de sortir du "rouge" aussi vite que possible.

Kent Beck le décrit comme :

> Faire fonctionner le test rapidement, en commettant tous les péchés nécessaires dans le processus.

Vous pouvez commettre ces péchés car vous refactoriserez ensuite, soutenu par la sécurité des tests.

### Que se passe-t-il si vous ne le faites pas ?

Plus vous effectuez de changements en étant dans le rouge, plus vous risquez d'ajouter davantage de problèmes, non couverts par des tests.

L'idée est d'écrire de façon itérative du code utile par petites étapes, guidé par des tests afin de ne pas tomber dans une impasse pendant des heures.

### L'œuf et la poule

Comment pouvons-nous construire cela de manière incrémentale ? Nous ne pouvons pas `GET` un joueur sans avoir stocké quelque chose et il semble difficile de savoir si `POST` a fonctionné sans que le point d'extrémité `GET` existe déjà.

C'est là que le _mocking_ brille.

-   `GET` aura besoin d'une _chose_ `PlayerStore` pour obtenir les scores d'un joueur. Ce devrait être une interface afin que lorsque nous testons, nous puissions créer un simple stub pour tester notre code sans avoir besoin d'implémenter un quelconque code de stockage réel.
-   Pour `POST`, nous pouvons _espionner_ ses appels à `PlayerStore` pour nous assurer qu'il stocke les joueurs correctement. Notre implémentation de la sauvegarde ne sera pas couplée à la récupération.
-   Pour avoir rapidement un logiciel fonctionnel, nous pouvons faire une implémentation en mémoire très simple, puis plus tard nous pourrons créer une implémentation soutenue par le mécanisme de stockage de notre choix.

## Écrire le test d'abord

Nous pouvons écrire un test et le faire passer en renvoyant une valeur codée en dur pour commencer. Kent Beck appelle cela "Faking it". Une fois que nous avons un test qui fonctionne, nous pouvons alors écrire plus de tests pour nous aider à éliminer cette constante.

En faisant cette très petite étape, nous pouvons faire le début important de mettre en place une structure de projet globale qui fonctionne correctement sans avoir à nous soucier trop de notre logique d'application.

Pour créer un serveur web en Go, vous allez généralement appeler [ListenAndServe](https://golang.org/pkg/net/http/#ListenAndServe).

```go
func ListenAndServe(addr string, handler Handler) error
```

Cela démarrera un serveur web écoutant sur un port, créant une goroutine pour chaque requête et l'exécutant contre un [`Handler`](https://golang.org/pkg/net/http/#Handler).

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

Un type implémente l'interface Handler en implémentant la méthode `ServeHTTP` qui attend deux arguments, le premier est l'endroit où nous _écrivons notre réponse_ et le second est la requête HTTP qui a été envoyée au serveur.

Créons un fichier nommé `server_test.go` et écrivons un test pour une fonction `PlayerServer` qui prend ces deux arguments. La requête envoyée sera pour obtenir le score d'un joueur, que nous attendons à être `"20"`.
```go
func TestGETPlayers(t *testing.T) {
	t.Run("returns Pepper's score", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/players/Pepper", nil)
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		got := response.Body.String()
		want := "20"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})
}
```

Pour tester notre serveur, nous aurons besoin d'une `Request` à envoyer et nous voudrons _espionner_ ce que notre handler écrit dans le `ResponseWriter`.

-   Nous utilisons `http.NewRequest` pour créer une requête. Le premier argument est la méthode de la requête et le second est le chemin de la requête. L'argument `nil` fait référence au corps de la requête, que nous n'avons pas besoin de définir dans ce cas.
-   `net/http/httptest` a un espion déjà fait pour nous appelé `ResponseRecorder` que nous pouvons utiliser. Il a de nombreuses méthodes utiles pour inspecter ce qui a été écrit comme réponse.

## Essayer d'exécuter le test

`./server_test.go:13:2: undefined: PlayerServer`

## Écrire la quantité minimale de code pour que le test s'exécute et vérifier la sortie du test échouant

Le compilateur est là pour aider, écoutez-le simplement.

Créez un fichier nommé `server.go` et définissez `PlayerServer`

```go
func PlayerServer() {}
```

Essayez à nouveau

```
./server_test.go:13:14: too many arguments in call to PlayerServer
    have (*httptest.ResponseRecorder, *http.Request)
    want ()
```

Ajoutez les arguments à notre fonction

```go
import "net/http"

func PlayerServer(w http.ResponseWriter, r *http.Request) {

}
```

Le code compile maintenant et le test échoue

```
=== RUN   TestGETPlayers/returns_Pepper's_score
    --- FAIL: TestGETPlayers/returns_Pepper's_score (0.00s)
        server_test.go:20: got '', want '20'
```

## Écrire suffisamment de code pour le faire passer

Dans le chapitre sur l'injection de dépendances, nous avons abordé les serveurs HTTP avec une fonction `Greet`. Nous avons appris que `ResponseWriter` de net/http implémente également `Writer` de io, nous pouvons donc utiliser `fmt.Fprint` pour envoyer des chaînes de caractères comme réponses HTTP.

```go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "20")
}
```

Le test devrait maintenant passer.

## Compléter la structure

Nous voulons intégrer cela dans une application. C'est important car :

-   Nous aurons un _logiciel réellement fonctionnel_, nous ne voulons pas écrire des tests pour le plaisir, c'est bien de voir le code en action.
-   Au fur et à mesure que nous refactorisons notre code, il est probable que nous changerons la structure du programme. Nous voulons nous assurer que cela se reflète également dans notre application dans le cadre de l'approche incrémentale.

Créez un nouveau fichier `main.go` pour notre application et mettez-y ce code :

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(PlayerServer)
	log.Fatal(http.ListenAndServe(":5000", handler))
}
```

Jusqu'à présent, tout notre code d'application a été dans un seul fichier, cependant, ce n'est pas une bonne pratique pour les projets plus importants où vous voudrez séparer les choses dans différents fichiers.

Pour l'exécuter, faites `go build` qui prendra tous les fichiers `.go` dans le répertoire et vous construira un programme. Vous pouvez ensuite l'exécuter avec `./myprogram`.

### `http.HandlerFunc`

Plus tôt, nous avons exploré que l'interface `Handler` est ce que nous devons implémenter pour créer un serveur. _Typiquement_, nous faisons cela en créant une `struct` et en lui faisant implémenter l'interface en implémentant sa propre méthode ServeHTTP. Cependant, l'utilisation des structs est pour contenir des données mais _actuellement_ nous n'avons pas d'état, donc cela ne semble pas juste d'en créer une.

[HandlerFunc](https://golang.org/pkg/net/http/#HandlerFunc) nous permet d'éviter cela.

> Le type HandlerFunc est un adaptateur qui permet l'utilisation de fonctions ordinaires comme gestionnaires HTTP. Si f est une fonction avec la signature appropriée, HandlerFunc(f) est un Handler qui appelle f.

```go
type HandlerFunc func(ResponseWriter, *Request)
```

D'après la documentation, nous voyons que le type `HandlerFunc` a déjà implémenté la méthode `ServeHTTP`.
En convertissant notre fonction `PlayerServer` avec ce type, nous avons maintenant implémenté le `Handler` requis.

### `http.ListenAndServe(":5000"...)`

`ListenAndServe` prend un port à écouter et un `Handler`. S'il y a un problème, le serveur web renverra une erreur, un exemple de cela pourrait être le port déjà écouté. Pour cette raison, nous enveloppons l'appel dans `log.Fatal` pour enregistrer l'erreur pour l'utilisateur.

Ce que nous allons faire maintenant est écrire _un autre_ test pour nous forcer à faire un changement positif pour essayer de nous éloigner de la valeur codée en dur.

## Écrire le test d'abord

Nous ajouterons un autre sous-test à notre suite qui essaie d'obtenir le score d'un joueur différent, ce qui brisera notre approche codée en dur.

```go
t.Run("returns Floyd's score", func(t *testing.T) {
	request, _ := http.NewRequest(http.MethodGet, "/players/Floyd", nil)
	response := httptest.NewRecorder()

	PlayerServer(response, request)

	got := response.Body.String()
	want := "10"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
})
```

Vous avez peut-être pensé

> Nous avons sûrement besoin d'une sorte de concept de stockage pour contrôler quel joueur obtient quel score. C'est étrange que les valeurs semblent si arbitraires dans nos tests.

Rappelez-vous que nous essayons simplement de faire des pas aussi petits que raisonnablement possible, donc nous essayons simplement de casser la constante pour l'instant.

## Essayer d'exécuter le test

```
=== RUN   TestGETPlayers/returns_Pepper's_score
    --- PASS: TestGETPlayers/returns_Pepper's_score (0.00s)
=== RUN   TestGETPlayers/returns_Floyd's_score
    --- FAIL: TestGETPlayers/returns_Floyd's_score (0.00s)
        server_test.go:34: got '20', want '10'
```

## Écrire suffisamment de code pour le faire passer

```go
//server.go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	if player == "Pepper" {
		fmt.Fprint(w, "20")
		return
	}

	if player == "Floyd" {
		fmt.Fprint(w, "10")
		return
	}
}
```

Ce test nous a forcés à examiner réellement l'URL de la requête et à prendre une décision. Donc même si dans nos têtes, nous nous inquiétions peut-être des magasins de joueurs et des interfaces, la prochaine étape logique semble en fait concerner le _routage_.

Si nous avions commencé avec le code du magasin, la quantité de changements que nous aurions à faire serait très grande par rapport à cela. **C'est un plus petit pas vers notre objectif final et il a été guidé par des tests**.

Nous résistons à la tentation d'utiliser des bibliothèques de routage pour l'instant, juste le plus petit pas pour faire passer notre test.

`r.URL.Path` renvoie le chemin de la requête que nous pouvons ensuite utiliser [`strings.TrimPrefix`](https://golang.org/pkg/strings/#TrimPrefix) pour éliminer `/players/` et obtenir le joueur demandé. Ce n'est pas très robuste mais fera l'affaire pour l'instant.

## Refactoriser

Nous pouvons simplifier le `PlayerServer` en séparant la récupération du score dans une fonction :

```go
//server.go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	fmt.Fprint(w, GetPlayerScore(player))
}

func GetPlayerScore(name string) string {
	if name == "Pepper" {
		return "20"
	}

	if name == "Floyd" {
		return "10"
	}

	return ""
}
```

Et nous pouvons éviter la répétition de code dans les tests en créant quelques fonctions d'aide :

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}

func newGetScoreRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodGet, fmt.Sprintf("/players/%s", name), nil)
	return req
}

func assertResponseBody(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("response body is wrong, got %q want %q", got, want)
	}
}
```

Cependant, nous ne devrions toujours pas être satisfaits. Cela ne semble pas correct que notre serveur connaisse les scores.

Notre refactorisation a rendu assez clair ce qu'il faut faire.

Nous avons déplacé le calcul du score hors du corps principal de notre gestionnaire dans une fonction `GetPlayerScore`. Cela semble être le bon endroit pour séparer les préoccupations en utilisant des interfaces.

Déplaçons notre fonction refactorisée pour qu'elle soit une interface à la place :

```go
type PlayerStore interface {
	GetPlayerScore(name string) int
}
```

Pour que notre `PlayerServer` puisse utiliser un `PlayerStore`, il aura besoin d'une référence à celui-ci. Maintenant semble être le bon moment pour changer notre architecture afin que notre `PlayerServer` soit maintenant une `struct`.

```go
type PlayerServer struct {
	store PlayerStore
}
```

Enfin, nous allons maintenant implémenter l'interface `Handler` en ajoutant une méthode à notre nouvelle struct et en y mettant notre code de gestionnaire existant.

```go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

Le seul autre changement est que nous appelons maintenant notre `store.GetPlayerScore` pour obtenir le score, plutôt que la fonction locale que nous avons définie (que nous pouvons maintenant supprimer).

Voici la liste complète du code de notre serveur :

```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
}

type PlayerServer struct {
	store PlayerStore
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

### Régler les problèmes

C'était pas mal de changements et nous savons que nos tests et notre application ne vont plus compiler, mais détendez-vous simplement et laissez le compilateur faire son travail.

`./main.go:9:58: type PlayerServer is not an expression`

Nous devons changer nos tests pour créer plutôt une nouvelle instance de notre `PlayerServer` puis appeler sa méthode `ServeHTTP`.

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	server := &PlayerServer{}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}
```

Notez que nous ne nous soucions toujours pas de créer des magasins _pour l'instant_, nous voulons simplement que le compilateur passe dès que possible.

Vous devriez avoir l'habitude de prioriser d'avoir un code qui compile et ensuite un code qui passe les tests.

En ajoutant plus de fonctionnalités (comme des stores stub) alors que le code ne compile pas, nous nous ouvrons potentiellement à _plus_ de problèmes de compilation.

Maintenant, `main.go` ne compilera pas pour la même raison.

```go
func main() {
	server := &PlayerServer{}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Finalement, tout compile mais les tests échouent :

```
=== RUN   TestGETPlayers/returns_the_Pepper's_score
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
    panic: runtime error: invalid memory address or nil pointer dereference
```

C'est parce que nous n'avons pas passé un `PlayerStore` dans nos tests. Nous devrons en créer un factice.

```go
//server_test.go
type StubPlayerStore struct {
	scores map[string]int
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}
```

Une `map` est un moyen rapide et facile de créer un magasin clé/valeur factice pour nos tests. Maintenant, créons un de ces magasins pour nos tests et envoyons-le dans notre `PlayerServer`.

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{
			"Pepper": 20,
			"Floyd":  10,
		},
	}
	server := &PlayerServer{&store}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}
```

Nos tests passent maintenant et sont meilleurs. L'_intention_ derrière notre code est plus claire maintenant en raison de l'introduction du magasin. Nous disons au lecteur que parce que nous avons _ces données dans un `PlayerStore`_, lorsque vous l'utilisez avec un `PlayerServer`, vous devriez obtenir les réponses suivantes.

### Exécuter l'application

Maintenant que nos tests passent, la dernière chose que nous devons faire pour terminer cette refactorisation est de vérifier si notre application fonctionne. Le programme devrait démarrer mais vous obtiendrez une réponse horrible si vous essayez d'atteindre le serveur à `http://localhost:5000/players/Pepper`.

La raison en est que nous n'avons pas transmis un `PlayerStore`.

Nous devrons en faire une implémentation, mais c'est difficile en ce moment car nous ne stockons aucune donnée significative, donc ce sera codé en dur pour le moment.

```go
//main.go
type InMemoryPlayerStore struct{}

func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return 123
}

func main() {
	server := &PlayerServer{&InMemoryPlayerStore{}}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Si vous exécutez `go build` à nouveau et accédez à la même URL, vous devriez obtenir `"123"`. Pas génial, mais jusqu'à ce que nous stockions des données, c'est le mieux que nous puissions faire.
De plus, il n'était pas très satisfaisant que notre application principale démarre mais ne fonctionne pas réellement. Nous avons dû tester manuellement pour voir le problème.

Nous avons quelques options quant à ce qu'il faut faire ensuite

-   Gérer le scénario où le joueur n'existe pas
-   Gérer le scénario `POST /players/{name}`

Bien que le scénario `POST` nous rapproche du "chemin heureux", je pense qu'il sera plus facile de s'attaquer d'abord au scénario du joueur manquant car nous sommes déjà dans ce contexte. Nous arriverons au reste plus tard.

## Écrire le test d'abord

Ajoutez un scénario de joueur manquant à notre suite existante :

```go
//server_test.go
t.Run("returns 404 on missing players", func(t *testing.T) {
	request := newGetScoreRequest("Apollo")
	response := httptest.NewRecorder()

	server.ServeHTTP(response, request)

	got := response.Code
	want := http.StatusNotFound

	if got != want {
		t.Errorf("got status %d want %d", got, want)
	}
})
```

## Essayer d'exécuter le test

```
=== RUN   TestGETPlayers/returns_404_on_missing_players
    --- FAIL: TestGETPlayers/returns_404_on_missing_players (0.00s)
        server_test.go:56: got status 200 want 404
```

## Écrire suffisamment de code pour le faire passer

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	w.WriteHeader(http.StatusNotFound)

	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

Parfois, je lève fortement les yeux lorsque les défenseurs du TDD disent "assurez-vous d'écrire juste la quantité minimale de code pour le faire passer" car cela peut sembler très pédant.

Mais ce scénario illustre bien l'exemple. J'ai fait le strict minimum (en sachant que ce n'est pas correct), qui consiste à écrire un `StatusNotFound` sur **toutes les réponses** mais tous nos tests passent !

**En faisant le minimum pour faire passer les tests, cela peut mettre en évidence des lacunes dans vos tests**. Dans notre cas, nous ne vérifions pas que nous devrions obtenir un `StatusOK` quand les joueurs _existent_ dans le magasin.

Mettez à jour les deux autres tests pour affirmer le statut et corriger le code.

Voici les nouveaux tests :

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{
			"Pepper": 20,
			"Floyd":  10,
		},
	}
	server := &PlayerServer{&store}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
		assertResponseBody(t, response.Body.String(), "10")
	})

	t.Run("returns 404 on missing players", func(t *testing.T) {
		request := newGetScoreRequest("Apollo")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusNotFound)
	})
}

func assertStatus(t testing.TB, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("did not get correct status, got %d, want %d", got, want)
	}
}

func newGetScoreRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodGet, fmt.Sprintf("/players/%s", name), nil)
	return req
}

func assertResponseBody(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("response body is wrong, got %q want %q", got, want)
	}
}
```

Nous vérifions maintenant le statut dans tous nos tests, donc j'ai créé un helper `assertStatus` pour faciliter cela.

Maintenant, nos deux premiers tests échouent à cause du 404 au lieu de 200, donc nous pouvons corriger `PlayerServer` pour ne renvoyer "not found" que si le score est 0.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}
```

### Stockage des scores

Maintenant que nous pouvons récupérer des scores d'un magasin, il est logique de pouvoir stocker de nouveaux scores.

## Écrire le test d'abord

```go
//server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
	}
	server := &PlayerServer{&store}

	t.Run("it returns accepted on POST", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodPost, "/players/Pepper", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)
	})
}
```

Pour commencer, vérifions simplement que nous obtenons le bon code d'état si nous accédons à la route particulière avec POST. Cela nous permet de développer la fonctionnalité d'accepter un type de requête différent et de le traiter différemment de `GET /players/{name}`. Une fois que cela fonctionne, nous pourrons alors commencer à affirmer sur l'interaction de notre gestionnaire avec le magasin.

## Essayer d'exécuter le test

```
=== RUN   TestStoreWins/it_returns_accepted_on_POST
    --- FAIL: TestStoreWins/it_returns_accepted_on_POST (0.00s)
        server_test.go:70: did not get correct status, got 404, want 202
```

## Écrire suffisamment de code pour le faire passer

N'oubliez pas que nous commettons délibérément des péchés, donc une instruction `if` basée sur la méthode de la requête fera l'affaire.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	if r.Method == http.MethodPost {
		w.WriteHeader(http.StatusAccepted)
		return
	}

	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}
```

## Refactoriser

Le gestionnaire est un peu confus maintenant. Séparons le code pour le rendre plus facile à suivre et isoler les différentes fonctionnalités dans de nouvelles fonctions.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	switch r.Method {
	case http.MethodPost:
		p.processWin(w)
	case http.MethodGet:
		p.showScore(w, r)
	}

}

func (p *PlayerServer) showScore(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}

func (p *PlayerServer) processWin(w http.ResponseWriter) {
	w.WriteHeader(http.StatusAccepted)
}
```

Cela rend l'aspect de routage de `ServeHTTP` un peu plus clair et signifie que nos prochaines itérations sur le stockage peuvent simplement être à l'intérieur de `processWin`.

Ensuite, nous voulons vérifier que lorsque nous faisons notre `POST /players/{name}`, notre `PlayerStore` est informé d'enregistrer la victoire.

## Écrire le test d'abord

Nous pouvons accomplir cela en étendant notre `StubPlayerStore` avec une nouvelle méthode `RecordWin` et en espionnant ses invocations.

```go
//server_test.go
type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}

func (s *StubPlayerStore) RecordWin(name string) {
	s.winCalls = append(s.winCalls, name)
}
```

Maintenant, étendons notre test pour vérifier le nombre d'invocations pour commencer :

```go
//server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
	}
	server := &PlayerServer{&store}

	t.Run("it records wins when POST", func(t *testing.T) {
		request := newPostWinRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)

		if len(store.winCalls) != 1 {
			t.Errorf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
		}
	})
}

func newPostWinRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodPost, fmt.Sprintf("/players/%s", name), nil)
	return req
}
```

## Essayer d'exécuter le test

```
./server_test.go:26:20: too few values in struct initializer
./server_test.go:65:20: too few values in struct initializer
```

## Écrire la quantité minimale de code pour que le test s'exécute et vérifier la sortie du test échouant

Nous devons mettre à jour notre code où nous créons un `StubPlayerStore` car nous avons ajouté un nouveau champ :

```go
//server_test.go
store := StubPlayerStore{
	map[string]int{},
	nil,
}
```

```
--- FAIL: TestStoreWins (0.00s)
    --- FAIL: TestStoreWins/it_records_wins_when_POST (0.00s)
        server_test.go:80: got 0 calls to RecordWin want 1
```

## Écrire suffisamment de code pour le faire passer

Comme nous n'affirmons que le nombre d'appels plutôt que les valeurs spécifiques, cela rend notre itération initiale un peu plus petite.

Nous devons mettre à jour l'idée de `PlayerServer` de ce qu'est un `PlayerStore` en changeant l'interface si nous allons pouvoir appeler `RecordWin`.

```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
}
```

En faisant cela, `main` ne compile plus :

```
./main.go:17:46: cannot use InMemoryPlayerStore literal (type *InMemoryPlayerStore) as type PlayerStore in field value:
    *InMemoryPlayerStore does not implement PlayerStore (missing RecordWin method)
```

Le compilateur nous dit ce qui ne va pas. Mettons à jour `InMemoryPlayerStore` pour avoir cette méthode.

```go
//main.go
type InMemoryPlayerStore struct{}

func (i *InMemoryPlayerStore) RecordWin(name string) {}
```

Essayez d'exécuter les tests et nous devrions être de retour à la compilation du code - mais le test échoue toujours.

Maintenant que `PlayerStore` a `RecordWin`, nous pouvons l'appeler dans notre `PlayerServer` :

```go
//server.go
func (p *PlayerServer) processWin(w http.ResponseWriter) {
	p.store.RecordWin("Bob")
	w.WriteHeader(http.StatusAccepted)
}
```

Exécutez les tests et ils devraient passer ! Évidemment, `"Bob"` n'est pas exactement ce que nous voulons envoyer à `RecordWin`, alors affinons davantage le test.

## Écrire le test d'abord

```go
//server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
		nil,
	}
	server := &PlayerServer{&store}

	t.Run("it records wins on POST", func(t *testing.T) {
		player := "Pepper"

		request := newPostWinRequest(player)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)

		if len(store.winCalls) != 1 {
			t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
		}

		if store.winCalls[0] != player {
			t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], player)
		}
	})
}
```

Maintenant que nous savons qu'il y a un élément dans notre slice `winCalls`, nous pouvons référencer en toute sécurité le premier et vérifier qu'il est égal à `player`.

## Essayer d'exécuter le test

```
=== RUN   TestStoreWins/it_records_wins_on_POST
    --- FAIL: TestStoreWins/it_records_wins_on_POST (0.00s)
        server_test.go:86: did not store correct winner got 'Bob' want 'Pepper'
```

## Écrire suffisamment de code pour le faire passer

```go
//server.go
func (p *PlayerServer) processWin(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	p.store.RecordWin(player)
	w.WriteHeader(http.StatusAccepted)
}
```

Nous avons modifié `processWin` pour prendre `http.Request` afin que nous puissions regarder l'URL pour extraire le nom du joueur. Une fois que nous avons cela, nous pouvons appeler notre `store` avec la valeur correcte pour faire passer le test.

## Refactoriser

Nous pouvons DRY ce code un peu car nous extrayons le nom du joueur de la même façon à deux endroits :

```go
//server.go
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

Même si nos tests passent, nous n'avons pas vraiment de logiciel fonctionnel. Si vous essayez d'exécuter `main` et d'utiliser le logiciel comme prévu, cela ne fonctionne pas car nous n'avons pas encore implémenté correctement `PlayerStore`. C'est bien cependant ; en nous concentrant sur notre gestionnaire, nous avons identifié l'interface dont nous avons besoin, plutôt que d'essayer de la concevoir à l'avance.

Nous _pourrions_ commencer à écrire des tests autour de notre `InMemoryPlayerStore` mais il n'est là que temporairement jusqu'à ce que nous implémentions une façon plus robuste de persister les scores des joueurs (c'est-à-dire une base de données).

Ce que nous ferons pour l'instant est d'écrire un _test d'intégration_ entre notre `PlayerServer` et `InMemoryPlayerStore` pour terminer la fonctionnalité. Cela nous permettra d'atteindre notre objectif d'être confiants que notre application fonctionne, sans avoir à tester directement `InMemoryPlayerStore`. Non seulement cela, mais lorsque nous en arriverons à implémenter `PlayerStore` avec une base de données, nous pourrons tester cette implémentation avec le même test d'intégration.

### Tests d'intégration

Les tests d'intégration peuvent être utiles pour tester que des zones plus importantes de votre système fonctionnent, mais vous devez garder à l'esprit :

-   Ils sont plus difficiles à écrire
-   Quand ils échouent, il peut être difficile de savoir pourquoi (généralement c'est un bug dans un composant du test d'intégration) et donc plus difficile à corriger
-   Ils sont parfois plus lents à exécuter (car ils sont souvent utilisés avec des composants "réels", comme une base de données)

Pour cette raison, il est recommandé de rechercher _La Pyramide des Tests_.

## Écrire le test d'abord

Dans un souci de concision, je vais vous montrer le test d'intégration final refactorisé.

```go
// server_integration_test.go
package main

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestRecordingWinsAndRetrievingThem(t *testing.T) {
	store := InMemoryPlayerStore{}
	server := PlayerServer{&store}
	player := "Pepper"

	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))

	response := httptest.NewRecorder()
	server.ServeHTTP(response, newGetScoreRequest(player))
	assertStatus(t, response.Code, http.StatusOK)

	assertResponseBody(t, response.Body.String(), "3")
}
```

-   Nous créons nos deux composants que nous essayons d'intégrer : `InMemoryPlayerStore` et `PlayerServer`.
-   Nous envoyons ensuite 3 requêtes pour enregistrer 3 victoires pour `player`. Nous ne sommes pas trop préoccupés par les codes d'état dans ce test car ils ne sont pas pertinents pour savoir s'ils s'intègrent bien.
-   La réponse suivante nous importe (donc nous stockons une variable `response`) car nous allons essayer d'obtenir le score de `player`.

## Essayer d'exécuter le test

```
--- FAIL: TestRecordingWinsAndRetrievingThem (0.00s)
    server_integration_test.go:24: response body is wrong, got '123' want '3'
```

## Écrire suffisamment de code pour le faire passer

Je vais prendre quelques libertés ici et écrire plus de code sans écrire de test.

_C'est permis !_ Nous avons toujours un test vérifiant que les choses devraient fonctionner correctement, mais ce n'est pas autour de l'unité spécifique avec laquelle nous travaillons (`InMemoryPlayerStore`).

Si je me retrouvais coincé dans ce scénario, je reviendrais à mon état de test défaillant et j'écrirais alors des tests unitaires plus spécifiques autour de `InMemoryPlayerStore` pour m'aider à élaborer une solution.

```go
//in_memory_player_store.go
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

-   Nous devons stocker les données, j'ai donc ajouté un `map[string]int` à la struct `InMemoryPlayerStore`
-   Pour plus de commodité, j'ai créé `NewInMemoryPlayerStore` pour initialiser le magasin, et j'ai mis à jour le test d'intégration pour l'utiliser :
    ```go
    //server_integration_test.go
    store := NewInMemoryPlayerStore()
    server := PlayerServer{store}
    ```
-   Le reste du code est juste une enveloppe autour de la `map`

Le test d'intégration passe, maintenant nous devons juste changer `main` pour utiliser `NewInMemoryPlayerStore()` :

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

Compilez-le, exécutez-le et utilisez `curl` pour le tester.

-   Exécutez ceci plusieurs fois, changez les noms des joueurs si vous le souhaitez : `curl -X POST http://localhost:5000/players/Pepper`
-   Vérifiez les scores avec `curl http://localhost:5000/players/Pepper`

Parfait ! Vous avez créé un service de type REST. Pour aller plus loin, vous voudriez choisir un magasin de données pour persister les scores plus longtemps que la durée d'exécution du programme.

-   Choisissez un magasin (Bolt ? Mongo ? Postgres ? Système de fichiers ?)
-   Faites en sorte que `PostgresPlayerStore` implémente `PlayerStore`
-   TDD la fonctionnalité pour être sûr que cela fonctionne
-   Branchez-le dans le test d'intégration, vérifiez que c'est toujours ok
-   Enfin, branchez-le dans `main`

## Refactoriser

Nous y sommes presque ! Prenons quelques efforts pour prévenir les erreurs de concurrence comme celles-ci :

```
fatal error: concurrent map read and map write
```

En ajoutant des mutexes, nous imposons la sécurité de concurrence, en particulier pour le compteur dans notre fonction `RecordWin`. Lisez-en plus sur les mutexes dans le chapitre sur la synchronisation.

## En résumé

### `http.Handler`

-   Implémentez cette interface pour créer des serveurs web
-   Utilisez `http.HandlerFunc` pour transformer des fonctions ordinaires en `http.Handler`
-   Utilisez `httptest.NewRecorder` pour passer en tant que `ResponseWriter` pour vous permettre d'espionner les réponses que votre gestionnaire envoie
-   Utilisez `http.NewRequest` pour construire les requêtes que vous vous attendez à voir arriver dans votre système

### Interfaces, Mocking et DI

-   Vous permet de construire progressivement le système en plus petits morceaux
-   Vous permet de développer un gestionnaire qui a besoin d'un stockage sans avoir besoin d'un stockage réel
-   TDD pour faire ressortir les interfaces dont vous avez besoin

### Commettez des péchés, puis refactorisez (et ensuite validez dans le contrôle de source)

-   Vous devez considérer avoir une compilation défaillante ou des tests défaillants comme une situation rouge dont vous devez sortir dès que possible.
-   Écrivez juste le code nécessaire pour y arriver. _Ensuite_ refactorisez et rendez le code agréable.
-   En essayant de faire trop de changements alors que le code ne compile pas ou que les tests échouent, vous risquez d'aggraver les problèmes.
-   S'en tenir à cette approche vous force à écrire de petits tests, ce qui signifie de petits changements, ce qui aide à garder le travail sur des systèmes complexes gérable.
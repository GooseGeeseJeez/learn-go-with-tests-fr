# WebSockets

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/websockets)**

Dans ce chapitre, nous allons apprendre à utiliser les WebSockets pour améliorer notre application.

## Rappel du projet

Nous avons deux applications dans notre code poker :

- *Application en ligne de commande*. Elle demande à l'utilisateur d'entrer le nombre de joueurs dans une partie. Ensuite, elle informe les joueurs de la valeur de la "mise obligatoire" (blind bet), qui augmente avec le temps. À tout moment, un utilisateur peut entrer `"{NomJoueur} wins"` pour terminer la partie et enregistrer le vainqueur dans un stockage.
- *Application web*. Elle permet aux utilisateurs d'enregistrer les gagnants des parties et affiche un classement. Elle partage le même stockage que l'application en ligne de commande.

## Prochaines étapes

Le propriétaire du produit est ravi de l'application en ligne de commande mais préférerait que nous puissions apporter cette fonctionnalité au navigateur. Elle imagine une page web avec une zone de texte permettant à l'utilisateur d'entrer le nombre de joueurs et, lorsqu'il soumet le formulaire, la page affiche la valeur de la mise obligatoire et la met à jour automatiquement au moment approprié. Comme pour l'application en ligne de commande, l'utilisateur peut déclarer le gagnant qui sera enregistré dans la base de données.

À première vue, cela semble assez simple, mais comme toujours, nous devons insister sur l'approche _itérative_ du développement logiciel.

Tout d'abord, nous devrons servir du HTML. Jusqu'à présent, tous nos points de terminaison HTTP ont renvoyé soit du texte brut, soit du JSON. Nous _pourrions_ utiliser les mêmes techniques que nous connaissons (car ce sont toutes des chaînes de caractères au final), mais nous pouvons également utiliser le package [html/template](https://golang.org/pkg/html/template/) pour une solution plus élégante.

Nous devons également pouvoir envoyer de manière asynchrone des messages à l'utilisateur disant `La mise obligatoire est maintenant *y*` sans avoir à rafraîchir le navigateur. Nous pouvons utiliser les [WebSockets](https://en.wikipedia.org/wiki/WebSocket) pour faciliter cela.

> WebSocket est un protocole de communication informatique, fournissant des canaux de communication full-duplex via une seule connexion TCP

Étant donné que nous adoptons un certain nombre de techniques, il est encore plus important que nous fassions d'abord le minimum de travail utile possible, puis que nous itérions.

Pour cette raison, la première chose que nous ferons sera de créer une page web avec un formulaire permettant à l'utilisateur d'enregistrer un gagnant. Au lieu d'utiliser un formulaire simple, nous utiliserons WebSockets pour envoyer ces données à notre serveur afin qu'il puisse les enregistrer.

Après cela, nous travaillerons sur les alertes de mise obligatoire, à quel moment nous aurons mis en place un peu de code d'infrastructure.

### Qu'en est-il des tests pour le JavaScript ?

Il y aura du JavaScript écrit pour cela, mais je n'entrerai pas dans l'écriture de tests.

C'est bien sûr possible, mais pour des raisons de concision, je n'inclurai aucune explication à ce sujet.

Désolé. Faites pression sur O'Reilly pour qu'ils me payent pour créer un "Learn JavaScript with tests".

## Écrivez d'abord le test

La première chose que nous devons faire est de servir du HTML aux utilisateurs lorsqu'ils accèdent à `/game`.

Voici un rappel du code pertinent dans notre serveur web :

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

const jsonContentType = "application/json"

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

La chose la plus _simple_ que nous puissions faire pour l'instant est de vérifier que lorsque nous faisons un `GET /game`, nous obtenons un `200`.

```go
func TestGame(t *testing.T) {
	t.Run("GET /game returns 200", func(t *testing.T) {
		server := NewPlayerServer(&StubPlayerStore{})

		request, _ := http.NewRequest(http.MethodGet, "/game", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

## Essayez d'exécuter le test
```
--- FAIL: TestGame (0.00s)
=== RUN   TestGame/GET_/game_returns_200
    --- FAIL: TestGame/GET_/game_returns_200 (0.00s)
    	server_test.go:109: did not get correct status, got 404, want 200
```

## Écrivez suffisamment de code pour le faire passer

Notre serveur a un routeur configuré, il est donc relativement facile à corriger.

Ajoutez à notre routeur :

```go
router.Handle("/game", http.HandlerFunc(p.game))
```

Et écrivez ensuite la méthode `game` :

```go
func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}
```

## Refactorisation

Le code du serveur est déjà bien structuré grâce au fait que nous insérons plus de code dans le code existant bien factorisé très facilement.

Nous pouvons nettoyer un peu le test en ajoutant une fonction d'aide `newGameRequest` pour faire la requête à `/game`. Essayez de l'écrire vous-même.

```go
func TestGame(t *testing.T) {
	t.Run("GET /game returns 200", func(t *testing.T) {
		server := NewPlayerServer(&StubPlayerStore{})

		request := newGameRequest()
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response, http.StatusOK)
	})
}
```

Vous remarquerez également que j'ai changé `assertStatus` pour accepter `response` plutôt que `response.Code` car je trouve que c'est plus lisible.

Maintenant, nous devons faire en sorte que le point de terminaison renvoie du HTML, le voici :

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Let's play poker</title>
</head>
<body>
<section id="game">
    <div id="declare-winner">
        <label for="winner">Winner</label>
        <input type="text" id="winner"/>
        <button id="winner-button">Declare winner</button>
    </div>
</section>
</body>
<script type="application/javascript">

    const submitWinnerButton = document.getElementById('winner-button')
    const winnerInput = document.getElementById('winner')

    if (window['WebSocket']) {
        const conn = new WebSocket('ws://' + document.location.host + '/ws')

        submitWinnerButton.onclick = event => {
            conn.send(winnerInput.value)
        }
    }
</script>
</html>
```

Nous avons une page web très simple :

 - Une entrée de texte pour que l'utilisateur puisse saisir le gagnant
 - Un bouton sur lequel il peut cliquer pour déclarer le gagnant
 - Du JavaScript pour ouvrir une connexion WebSocket à notre serveur et gérer l'appui sur le bouton de soumission

`WebSocket` est intégré à la plupart des navigateurs modernes, nous n'avons donc pas à nous soucier d'importer des bibliothèques. La page web ne fonctionnera pas pour les navigateurs plus anciens, mais c'est acceptable pour ce scénario.

### Comment tester que nous renvoyons le bon balisage ?

Il y a plusieurs façons. Comme cela a été souligné tout au long du livre, il est important que les tests que vous écrivez aient une valeur suffisante pour justifier le coût.

1. Écrire un test basé sur un navigateur, en utilisant quelque chose comme Selenium. Ces tests sont les plus "réalistes" de toutes les approches car ils démarrent un vrai navigateur web et simulent un utilisateur interagissant avec lui. Ces tests peuvent vous donner beaucoup de confiance que votre système fonctionne, mais ils sont plus difficiles à écrire que les tests unitaires et beaucoup plus lents à exécuter. Pour les besoins de notre produit, c'est exagéré.
2. Faire une correspondance exacte de chaîne. Cela _peut_ être correct, mais ces types de tests finissent par être très fragiles. Dès que quelqu'un change le balisage, vous aurez un test qui échoue alors qu'en pratique rien n'est _réellement cassé_.
3. Vérifier que nous appelons le bon modèle. Nous utiliserons une bibliothèque de modèles de la bibliothèque standard pour servir le HTML (discuté sous peu) et nous pourrions injecter la _chose_ pour générer le HTML et espionner son appel pour vérifier que nous le faisons correctement. Cela aurait un impact sur la conception de notre code mais ne teste pas grand-chose ; seulement que nous l'appelons avec le bon fichier de modèle. Étant donné que nous n'aurons qu'un seul modèle dans notre projet, le risque d'échec semble faible.

Donc, dans le livre "Learn Go with Tests" pour la première fois, nous n'allons pas écrire de test.

Placez le balisage dans un fichier appelé `game.html`

Ensuite, modifiez le point de terminaison que nous venons d'écrire comme suit :

```go
func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	tmpl, err := template.ParseFiles("game.html")

	if err != nil {
		http.Error(w, fmt.Sprintf("problem loading template %s", err.Error()), http.StatusInternalServerError)
		return
	}

	tmpl.Execute(w, nil)
}
```

[`html/template`](https://golang.org/pkg/html/template/) est un package Go pour créer du HTML. Dans notre cas, nous appelons `template.ParseFiles`, en donnant le chemin de notre fichier html. En supposant qu'il n'y ait pas d'erreur, vous pouvez ensuite `Execute` le modèle, qui l'écrit dans un `io.Writer`. Dans notre cas, nous voulons qu'il `Write` sur internet, nous lui donnons donc notre `http.ResponseWriter`.

Comme nous n'avons pas écrit de test, il serait prudent de tester manuellement notre serveur web juste pour s'assurer que les choses fonctionnent comme nous l'espérions. Allez dans `cmd/webserver` et exécutez le fichier `main.go`. Visitez `http://localhost:5000/game`.

Vous _devriez_ avoir obtenu une erreur concernant l'impossibilité de trouver le modèle. Vous pouvez soit modifier le chemin pour qu'il soit relatif à votre dossier, soit avoir une copie du fichier `game.html` dans le répertoire `cmd/webserver`. J'ai choisi de créer un lien symbolique (`ln -s ../../game.html game.html`) vers le fichier à la racine du projet afin que si je fais des modifications, elles soient reflétées lors de l'exécution du serveur.

Si vous effectuez cette modification et exécutez à nouveau, vous devriez voir notre interface utilisateur.

Maintenant, nous devons tester que lorsque nous recevons une chaîne via une connexion WebSocket à notre serveur, nous la déclarons comme gagnante d'une partie.

## Écrivez d'abord le test

Pour la première fois, nous allons utiliser une bibliothèque externe afin de pouvoir travailler avec les WebSockets.

Exécutez `go get github.com/gorilla/websocket`

Cela récupérera le code de l'excellente bibliothèque [Gorilla WebSocket](https://github.com/gorilla/websocket). Maintenant, nous pouvons mettre à jour nos tests pour notre nouvelle exigence.

```go
t.Run("when we get a message over a websocket it is a winner of a game", func(t *testing.T) {
	store := &StubPlayerStore{}
	winner := "Ruth"
	server := httptest.NewServer(NewPlayerServer(store))
	defer server.Close()

	wsURL := "ws" + strings.TrimPrefix(server.URL, "http") + "/ws"

	ws, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
	if err != nil {
		t.Fatalf("could not open a ws connection on %s %v", wsURL, err)
	}
	defer ws.Close()

	if err := ws.WriteMessage(websocket.TextMessage, []byte(winner)); err != nil {
		t.Fatalf("could not send message over ws connection %v", err)
	}

	AssertPlayerWin(t, store, winner)
})
```

Assurez-vous que vous avez un import pour la bibliothèque `websocket`. Mon IDE l'a fait automatiquement pour moi, le vôtre devrait le faire aussi.

Pour tester ce qui se passe depuis le navigateur, nous devons ouvrir notre propre connexion WebSocket et y écrire.

Nos tests précédents concernant notre serveur se contentaient d'appeler des méthodes sur notre serveur, mais maintenant nous devons avoir une connexion persistante à notre serveur. Pour ce faire, nous utilisons `httptest.NewServer` qui prend un `http.Handler` et le démarrera pour écouter les connexions.

En utilisant `websocket.DefaultDialer.Dial`, nous essayons de nous connecter à notre serveur, puis nous essayerons d'envoyer un message avec notre `winner`.

Enfin, nous affirmons sur le magasin de joueurs pour vérifier que le gagnant a été enregistré.

## Essayez d'exécuter le test
```
=== RUN   TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game
    --- FAIL: TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game (0.00s)
        server_test.go:124: could not open a ws connection on ws://127.0.0.1:55838/ws websocket: bad handshake
```

Nous n'avons pas modifié notre serveur pour accepter les connexions WebSocket sur `/ws`, nous ne faisons donc pas encore de poignée de main.

## Écrivez suffisamment de code pour le faire passer

Ajoutez une autre entrée à notre routeur :

```go
router.Handle("/ws", http.HandlerFunc(p.webSocket))
```

Puis ajoutez notre nouveau gestionnaire `webSocket` :

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	upgrader := websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
	}
	upgrader.Upgrade(w, r, nil)
}
```

Pour accepter une connexion WebSocket, nous `Upgrade` la requête. Si vous réexécutez maintenant le test, vous devriez passer à l'erreur suivante.

```
=== RUN   TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game
    --- FAIL: TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game (0.00s)
        server_test.go:132: got 0 calls to RecordWin want 1
```

Maintenant que nous avons ouvert une connexion, nous voudrons écouter un message puis l'enregistrer comme gagnant.

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	upgrader := websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
	}
	conn, _ := upgrader.Upgrade(w, r, nil)
	_, winnerMsg, _ := conn.ReadMessage()
	p.store.RecordWin(string(winnerMsg))
}
```

(Oui, nous ignorons beaucoup d'erreurs pour l'instant !)

`conn.ReadMessage()` bloque en attendant un message sur la connexion. Une fois que nous en obtenons un, nous l'utilisons pour `RecordWin`. Cela fermerait finalement la connexion WebSocket.

Si vous essayez d'exécuter le test, il échoue toujours.

Le problème est le timing. Il y a un délai entre notre connexion WebSocket lisant le message et enregistrant la victoire, et notre test se termine avant que cela ne se produise. Vous pouvez tester cela en mettant un court `time.Sleep` avant l'assertion finale.

Allons-y pour l'instant mais reconnaissons que mettre des pauses arbitraires dans les tests **est une très mauvaise pratique**.

```go
time.Sleep(10 * time.Millisecond)
AssertPlayerWin(t, store, winner)
```

## Refactorisation

Nous avons commis de nombreux péchés pour faire fonctionner ce test, à la fois dans le code du serveur et dans le code de test, mais souvenez-vous que c'est la façon la plus simple de travailler.

Nous avons un logiciel désagréable, horrible, mais _fonctionnel_ soutenu par un test, donc maintenant nous sommes libres de l'améliorer et de savoir que nous ne casserons rien accidentellement.

Commençons par le code du serveur.

Nous pouvons déplacer l'`upgrader` vers une valeur privée à l'intérieur de notre package car nous n'avons pas besoin de le redéclarer à chaque demande de connexion WebSocket :

```go
var wsUpgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
}

func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	conn, _ := wsUpgrader.Upgrade(w, r, nil)
	_, winnerMsg, _ := conn.ReadMessage()
	p.store.RecordWin(string(winnerMsg))
}
```

Notre appel à `template.ParseFiles("game.html")` s'exécutera à chaque `GET /game`, ce qui signifie que nous irons sur le système de fichiers à chaque requête alors que nous n'avons pas besoin de réanalyser le modèle. Refactorisons notre code pour que nous analysions le modèle une fois dans `NewPlayerServer` à la place. Nous devrons faire en sorte que cette fonction puisse maintenant renvoyer une erreur au cas où nous aurions des problèmes pour récupérer le modèle sur le disque ou l'analyser.

Voici les modifications pertinentes à `PlayerServer` :

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
	template *template.Template
}

const htmlTemplatePath = "game.html"

func NewPlayerServer(store PlayerStore) (*PlayerServer, error) {
	p := new(PlayerServer)

	tmpl, err := template.ParseFiles(htmlTemplatePath)

	if err != nil {
		return nil, fmt.Errorf("problem opening %s %v", htmlTemplatePath, err)
	}

	p.template = tmpl
	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))
	router.Handle("/game", http.HandlerFunc(p.game))
	router.Handle("/ws", http.HandlerFunc(p.webSocket))

	p.Handler = router

	return p, nil
}

func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	p.template.Execute(w, nil)
}
```

En changeant la signature de `NewPlayerServer`, nous avons maintenant des problèmes de compilation. Essayez de les résoudre vous-même ou référez-vous au code source si vous avez des difficultés.

Pour le code de test, j'ai créé un helper appelé `mustMakePlayerServer(t *testing.T, store PlayerStore) *PlayerServer` pour pouvoir cacher le bruit d'erreur des tests.

```go
func mustMakePlayerServer(t *testing.T, store PlayerStore) *PlayerServer {
	server, err := NewPlayerServer(store)
	if err != nil {
		t.Fatal("problem creating player server", err)
	}
	return server
}
```

De même, j'ai créé un autre helper `mustDialWS` pour pouvoir cacher le bruit d'erreur désagréable lors de la création de la connexion WebSocket.

```go
func mustDialWS(t *testing.T, url string) *websocket.Conn {
	ws, _, err := websocket.DefaultDialer.Dial(url, nil)

	if err != nil {
		t.Fatalf("could not open a ws connection on %s %v", url, err)
	}

	return ws
}
```

Enfin, dans notre code de test, nous pouvons créer un helper pour faciliter l'envoi de messages :

```go
func writeWSMessage(t testing.TB, conn *websocket.Conn, message string) {
	t.Helper()
	if err := conn.WriteMessage(websocket.TextMessage, []byte(message)); err != nil {
		t.Fatalf("could not send message over ws connection %v", err)
	}
}
```

Maintenant que les tests passent, essayez d'exécuter le serveur et déclarez quelques gagnants dans `/game`. Vous devriez les voir enregistrés dans `/league`. N'oubliez pas qu'à chaque fois que nous obtenons un gagnant, nous _fermons la connexion_, vous devrez actualiser la page pour ouvrir à nouveau la connexion.

Nous avons créé un formulaire web trivial qui permet aux utilisateurs d'enregistrer le gagnant d'un jeu. Itérons dessus pour permettre à l'utilisateur de démarrer une partie en fournissant un nombre de joueurs, et le serveur enverra des messages au client les informant de la valeur de la mise aveugle au fil du temps.

Mettez d'abord à jour `game.html` pour mettre à jour notre code côté client pour les nouvelles exigences :

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Lets play poker</title>
</head>
<body>
<section id="game">
    <div id="game-start">
        <label for="player-count">Number of players</label>
        <input type="number" id="player-count"/>
        <button id="start-game">Start</button>
    </div>

    <div id="declare-winner">
        <label for="winner">Winner</label>
        <input type="text" id="winner"/>
        <button id="winner-button">Declare winner</button>
    </div>

    <div id="blind-value"/>
</section>

<section id="game-end">
    <h1>Another great game of poker everyone!</h1>
    <p><a href="/league">Go check the league table</a></p>
</section>

</body>
<script type="application/javascript">
    const startGame = document.getElementById('game-start')

    const declareWinner = document.getElementById('declare-winner')
    const submitWinnerButton = document.getElementById('winner-button')
    const winnerInput = document.getElementById('winner')

    const blindContainer = document.getElementById('blind-value')

    const gameContainer = document.getElementById('game')
    const gameEndContainer = document.getElementById('game-end')

    declareWinner.hidden = true
    gameEndContainer.hidden = true

    document.getElementById('start-game').addEventListener('click', event => {
        startGame.hidden = true
        declareWinner.hidden = false

        const numberOfPlayers = document.getElementById('player-count').value

        if (window['WebSocket']) {
            const conn = new WebSocket('ws://' + document.location.host + '/ws')

            submitWinnerButton.onclick = event => {
                conn.send(winnerInput.value)
                gameEndContainer.hidden = false
                gameContainer.hidden = true
            }

            conn.onclose = evt => {
                blindContainer.innerText = 'Connection closed'
            }

            conn.onmessage = evt => {
                blindContainer.innerText = evt.data
            }

            conn.onopen = function () {
                conn.send(numberOfPlayers)
            }
        }
    })
</script>
</html>
```

Les principaux changements sont l'introduction d'une section pour entrer le nombre de joueurs et d'une section pour afficher la valeur de la mise aveugle. Nous avons une petite logique pour afficher/masquer l'interface utilisateur en fonction de l'étape du jeu.

Tout message que nous recevons via `conn.onmessage` est supposé être des alertes aveugles, nous définissons donc le `blindContainer.innerText` en conséquence.

Comment envoyer les alertes aveugles ? Dans le chapitre précédent, nous avons introduit l'idée de `Game` pour que notre code CLI puisse appeler un `Game` et tout le reste serait pris en charge, y compris la programmation des alertes aveugles. Cela s'est avéré être une bonne séparation des préoccupations.

```go
type Game interface {
	Start(numberOfPlayers int)
	Finish(winner string)
}
```

Lorsque l'utilisateur était invité dans le CLI à saisir le nombre de joueurs, le système `Start` démarrait le jeu, ce qui lançait les alertes aveugles, et lorsque l'utilisateur déclarait le gagnant, ils `Finish`. Ce sont les mêmes exigences que nous avons maintenant, juste une façon différente d'obtenir les entrées ; nous devrions donc chercher à réutiliser ce concept si nous le pouvons.

Notre implémentation "réelle" de `Game` est `TexasHoldem` :

```go
type TexasHoldem struct {
	alerter BlindAlerter
	store   PlayerStore
}
```

En envoyant un `BlindAlerter`, `TexasHoldem` peut programmer des alertes aveugles à envoyer _où que ce soit_ :

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}
```

Et pour rappel, voici notre implémentation du `BlindAlerter` que nous utilisons dans le CLI :

```go
func StdOutAlerter(duration time.Duration, amount int) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(os.Stdout, "Blind is now %d\n", amount)
	})
}
```

Cela fonctionne dans le CLI car nous _voulons toujours envoyer les alertes à `os.Stdout`_, mais cela ne fonctionnera pas pour notre serveur web. Pour chaque requête, nous obtenons un nouveau `http.ResponseWriter` que nous mettons ensuite à niveau vers `*websocket.Conn`. Nous ne pouvons donc pas savoir lors de la construction de nos dépendances où nos alertes doivent aller.

Pour cette raison, nous devons changer `BlindAlerter.ScheduleAlertAt` pour qu'il prenne une destination pour les alertes afin que nous puissions le réutiliser dans notre serveur web.

Ouvrez `blind_alerter.go` et ajoutez le paramètre à `io.Writer` :

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int, to io.Writer)
}

type BlindAlerterFunc func(duration time.Duration, amount int, to io.Writer)

func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int, to io.Writer) {
	a(duration, amount, to)
}
```

L'idée d'un `StdoutAlerter` ne correspond pas à notre nouveau modèle, donc renommez-le simplement en `Alerter` :

```go
func Alerter(duration time.Duration, amount int, to io.Writer) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(to, "Blind is now %d\n", amount)
	})
}
```

Si vous essayez de compiler, cela échouera dans `TexasHoldem` car il appelle `ScheduleAlertAt` sans destination, pour que les choses se compilent à nouveau _pour l'instant_, codez en dur `os.Stdout`.

Essayez d'exécuter les tests et ils échoueront car `SpyBlindAlerter` n'implémente plus `BlindAlerter`, corrigez cela en mettant à jour la signature de `ScheduleAlertAt`, exécutez les tests et nous devrions toujours être au vert.

Il n'est pas logique que `TexasHoldem` sache où envoyer les alertes aveugles. Mettons maintenant à jour `Game` pour que lorsque vous commencez une partie, vous déclariez _où_ les alertes doivent aller.

```go
type Game interface {
	Start(numberOfPlayers int, alertsDestination io.Writer)
	Finish(winner string)
}
```

Laissez le compilateur vous dire ce que vous devez corriger. Le changement n'est pas si mauvais :

- Mettez à jour `TexasHoldem` pour qu'il implémente correctement `Game`
- Dans `CLI` quand nous commençons le jeu, passez notre propriété `out` (`cli.game.Start(numberOfPlayers, cli.out)`)
- Dans le test de `TexasHoldem`, j'utilise `game.Start(5, io.Discard)` pour résoudre le problème de compilation et configurer la sortie d'alerte pour être jetée

Si vous avez bien fait les choses, tout devrait être au vert ! Maintenant, nous pouvons essayer d'utiliser `Game` dans `Server`.

## Écrivez d'abord le test

Les exigences de `CLI` et de `Server` sont les mêmes ! C'est juste le mécanisme de livraison qui est différent.

Jetons un coup d'œil à notre test `CLI` pour nous inspirer.

```go
t.Run("start game with 3 players and finish game with 'Chris' as winner", func(t *testing.T) {
	game := &GameSpy{}

	out := &bytes.Buffer{}
	in := userSends("3", "Chris wins")

	poker.NewCLI(in, out, game).PlayPoker()

	assertMessagesSentToUser(t, out, poker.PlayerPrompt)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, "Chris")
})
```

Il semble que nous devrions pouvoir aboutir à un résultat similaire en utilisant `GameSpy`.

Remplacez l'ancien test websocket par ce qui suit :

```go
t.Run("start a game with 3 players and declare Ruth the winner", func(t *testing.T) {
	game := &poker.GameSpy{}
	winner := "Ruth"
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(10 * time.Millisecond)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)
})
```

- Comme discuté, nous créons un espion `Game` et le transmettons à `mustMakePlayerServer` (assurez-vous de mettre à jour l'assistant pour prendre cela en charge).
- Nous envoyons ensuite les messages websocket pour une partie.
- Enfin, nous vérifions que le jeu est démarré et terminé avec ce que nous attendons.

## Essayez d'exécuter le test

Vous aurez un certain nombre d'erreurs de compilation autour de `mustMakePlayerServer` dans d'autres tests. Introduisez une variable non exportée `dummyGame` et utilisez-la dans tous les tests qui ne se compilent pas :

```go
var (
	dummyGame = &GameSpy{}
)
```

La dernière erreur est là où nous essayons de passer `Game` à `NewPlayerServer` mais il ne le supporte pas encore :

```
./server_test.go:21:38: too many arguments in call to "github.com/quii/learn-go-with-tests/WebSockets/v2".NewPlayerServer
	have ("github.com/quii/learn-go-with-tests/WebSockets/v2".PlayerStore, "github.com/quii/learn-go-with-tests/WebSockets/v2".Game)
	want ("github.com/quii/learn-go-with-tests/WebSockets/v2".PlayerStore)
```

## Écrivez le minimum de code pour que le test s'exécute et vérifiez la sortie du test échoué

Ajoutez-le simplement comme argument pour l'instant juste pour faire fonctionner le test :

```go
func NewPlayerServer(store PlayerStore, game Game) (*PlayerServer, error)
```

Enfin !

```
=== RUN   TestGame/start_a_game_with_3_players_and_declare_Ruth_the_winner
--- FAIL: TestGame (0.01s)
    --- FAIL: TestGame/start_a_game_with_3_players_and_declare_Ruth_the_winner (0.01s)
    	server_test.go:146: wanted Start called with 3 but got 0
    	server_test.go:147: expected finish called with 'Ruth' but got ''
FAIL
```

## Écrivez suffisamment de code pour le faire passer

Nous devons ajouter `Game` comme champ à `PlayerServer` pour qu'il puisse l'utiliser lorsqu'il reçoit des requêtes.

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
	template *template.Template
	game     Game
}
```

(Nous avons déjà une méthode appelée `game` donc renommez-la en `playGame`)

Ensuite, attribuons-la dans notre constructeur :

```go
func NewPlayerServer(store PlayerStore, game Game) (*PlayerServer, error) {
	p := new(PlayerServer)

	tmpl, err := template.ParseFiles(htmlTemplatePath)

	if err != nil {
		return nil, fmt.Errorf("problem opening %s %v", htmlTemplatePath, err)
	}

	p.game = game

	// etc
}
```

Maintenant, nous pouvons utiliser notre `Game` dans `webSocket`.

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	conn, _ := wsUpgrader.Upgrade(w, r, nil)

	_, numberOfPlayersMsg, _ := conn.ReadMessage()
	numberOfPlayers, _ := strconv.Atoi(string(numberOfPlayersMsg))
	p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!

	_, winner, _ := conn.ReadMessage()
	p.game.Finish(string(winner))
}
```

Hourra ! Les tests passent.

Nous n'allons pas envoyer les messages d'alerte aveugle _pour l'instant_ car nous devons y réfléchir. Quand nous appelons `game.Start`, nous envoyons `io.Discard` qui va simplement jeter tous les messages qui lui sont écrits.

Pour l'instant, démarrez le serveur web. Vous devrez mettre à jour le `main.go` pour passer un `Game` au `PlayerServer` :

```go
func main() {
	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	game := poker.NewTexasHoldem(poker.BlindAlerterFunc(poker.Alerter), store)

	server, err := poker.NewPlayerServer(store, game)

	if err != nil {
		log.Fatalf("problem creating player server %v", err)
	}

	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Sans tenir compte du fait que nous ne recevons pas encore d'alertes aveugles, l'application fonctionne ! Nous avons réussi à réutiliser `Game` avec `PlayerServer` et il a pris en charge tous les détails. Une fois que nous aurons trouvé comment envoyer nos alertes aveugles à travers les websockets plutôt que de les jeter, _cela devrait_ tout fonctionner.

Mais avant cela, nettoyons un peu le code.

## Refactorisation

La façon dont nous utilisons les WebSockets est assez basique et la gestion des erreurs est assez naïve, j'ai donc voulu encapsuler cela dans un type juste pour supprimer ce désordre du code du serveur. Nous pourrons peut-être le revoir plus tard, mais pour l'instant, cela va un peu nettoyer les choses :

```go
type playerServerWS struct {
	*websocket.Conn
}

func newPlayerServerWS(w http.ResponseWriter, r *http.Request) *playerServerWS {
	conn, err := wsUpgrader.Upgrade(w, r, nil)

	if err != nil {
		log.Printf("problem upgrading connection to WebSockets %v\n", err)
	}

	return &playerServerWS{conn}
}

func (w *playerServerWS) WaitForMsg() string {
	_, msg, err := w.ReadMessage()
	if err != nil {
		log.Printf("error reading from websocket %v\n", err)
	}
	return string(msg)
}
```

Maintenant, le code du serveur est un peu simplifié :

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!

	winner := ws.WaitForMsg()
	p.game.Finish(winner)
}
```

Une fois que nous aurons trouvé comment ne pas jeter les messages d'alerte aveugle, nous aurons terminé.

### N'écrivons _pas_ de test !

Parfois, lorsque nous ne sommes pas sûrs de la façon de faire quelque chose, il est préférable de jouer et d'essayer des choses ! Assurez-vous que votre travail est d'abord validé car une fois que nous aurons trouvé un moyen, nous devrions le faire passer par un test.

La ligne de code problématique que nous avons est :

```go
p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!
```

Nous devons passer un `io.Writer` pour que le jeu puisse y écrire les alertes aveugles.

Ne serait-il pas agréable si nous pouvions passer notre `playerServerWS` d'avant ? C'est notre enveloppe autour de notre WebSocket, donc il _semble_ que nous devrions pouvoir l'envoyer à notre `Game` pour envoyer des messages.

Essayez :

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, ws)
	//etc...
}
```

Le compilateur se plaint :

```
./server.go:71:14: cannot use ws (type *playerServerWS) as type io.Writer in argument to p.game.Start:
	*playerServerWS does not implement io.Writer (missing Write method)
```

Il semble que la chose évidente à faire serait de faire en sorte que `playerServerWS` _implémente_ `io.Writer`. Pour ce faire, nous utilisons le `*websocket.Conn` sous-jacent pour utiliser `WriteMessage` pour envoyer le message à travers le websocket :

```go
func (w *playerServerWS) Write(p []byte) (n int, err error) {
	err = w.WriteMessage(websocket.TextMessage, p)

	if err != nil {
		return 0, err
	}

	return len(p), nil
}
```

Cela semble trop facile ! Essayez d'exécuter l'application et voyez si elle fonctionne.

Auparavant, modifiez `TexasHoldem` pour que le temps d'incrémentation des blinds soit plus court afin que vous puissiez le voir en action :

```go
blindIncrement := time.Duration(5+numberOfPlayers) * time.Second // (plutôt qu'une minute)
```

Vous devriez le voir fonctionner ! Le montant de la mise aveugle s'incrémente dans le navigateur comme par magie.

Maintenant, revenons au code et réfléchissons à la façon de le tester. Pour _l'implémenter_, tout ce que nous avons fait était de passer à `StartGame` `playerServerWS` plutôt que `io.Discard`, ce qui pourrait vous faire penser que nous devrions peut-être espionner l'appel pour vérifier qu'il fonctionne.

L'espionnage est génial et nous aide à vérifier les détails d'implémentation, mais nous devrions toujours essayer de favoriser le test du _vrai_ comportement si nous le pouvons, car lorsque vous décidez de refactoriser, ce sont souvent les tests d'espionnage qui commencent à échouer car ils vérifient généralement les détails d'implémentation que vous essayez de changer.

Notre test ouvre actuellement une connexion websocket à notre serveur en cours d'exécution et envoie des messages pour lui faire faire des choses. De même, nous devrions pouvoir tester les messages que notre serveur renvoie via la connexion websocket.

## Écrivez d'abord le test

Nous allons modifier notre test existant.

Actuellement, notre `GameSpy` n'envoie aucune donnée à `out` lorsque vous appelez `Start`. Nous devrions le modifier pour que nous puissions le configurer pour envoyer un message en conserve, puis nous pouvons vérifier que ce message est envoyé au websocket. Cela devrait nous donner confiance que nous avons configuré les choses correctement tout en exerçant le comportement réel que nous voulons.

```go
type GameSpy struct {
	StartCalled     bool
	StartCalledWith int
	BlindAlert      []byte

	FinishedCalled   bool
	FinishCalledWith string
}
```

Ajoutez le champ `BlindAlert`.

Mettez à jour `GameSpy` `Start` pour envoyer le message en conserve à `out`.

```go
func (g *GameSpy) Start(numberOfPlayers int, out io.Writer) {
	g.StartCalled = true
	g.StartCalledWith = numberOfPlayers
	out.Write(g.BlindAlert)
}
```

Cela signifie maintenant que lorsque nous exerçons `PlayerServer` lorsqu'il essaie de `Start` le jeu, il devrait finir par envoyer des messages via le websocket si les choses fonctionnent correctement.

Enfin, nous pouvons mettre à jour le test :

```go
t.Run("start a game with 3 players, send some blind alerts down WS and declare Ruth the winner", func(t *testing.T) {
	wantedBlindAlert := "Blind is 100"
	winner := "Ruth"

	game := &GameSpy{BlindAlert: []byte(wantedBlindAlert)}
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(10 * time.Millisecond)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)

	_, gotBlindAlert, _ := ws.ReadMessage()

	if string(gotBlindAlert) != wantedBlindAlert {
		t.Errorf("got blind alert %q, want %q", string(gotBlindAlert), wantedBlindAlert)
	}
})
```

- Nous avons ajouté un `wantedBlindAlert` et configuré notre `GameSpy` pour l'envoyer à `out` si `Start` est appelé.
- Nous espérons qu'il est envoyé dans la connexion websocket, nous avons donc ajouté un appel à `ws.ReadMessage()` pour attendre qu'un message soit envoyé, puis vérifier que c'est celui que nous attendions.

## Essayez d'exécuter le test

Vous devriez constater que le test est bloqué indéfiniment. C'est parce que `ws.ReadMessage()` bloquera jusqu'à ce qu'il obtienne un message, ce qui n'arrivera jamais.

## Écrivez le minimum de code pour que le test s'exécute et vérifiez la sortie du test échoué

Nous ne devrions jamais avoir de tests qui se bloquent, donc introduisons un moyen de gérer le code que nous voulons temporiser.

```go
func within(t testing.TB, d time.Duration, assert func()) {
	t.Helper()

	done := make(chan struct{}, 1)

	go func() {
		assert()
		done <- struct{}{}
	}()

	select {
	case <-time.After(d):
		t.Error("timed out")
	case <-done:
	}
}
```

Ce que fait `within`, c'est prendre une fonction `assert` comme argument puis l'exécuter dans une go routine. Si/Quand la fonction se termine, elle signalera qu'elle a terminé via le canal `done`.

Pendant ce temps, nous utilisons une instruction `select` qui nous permet d'attendre qu'un canal envoie un message. À partir de là, c'est une course entre la fonction `assert` et `time.After` qui enverra un signal lorsque la durée sera écoulée.

Enfin, j'ai créé une fonction d'aide pour notre assertion juste pour rendre les choses un peu plus propres :

```go
func assertWebsocketGotMsg(t *testing.T, ws *websocket.Conn, want string) {
	_, msg, _ := ws.ReadMessage()
	if string(msg) != want {
		t.Errorf(`got "%s", want "%s"`, string(msg), want)
	}
}
```

Voici comment se lit maintenant le test :

```go
t.Run("start a game with 3 players, send some blind alerts down WS and declare Ruth the winner", func(t *testing.T) {
	wantedBlindAlert := "Blind is 100"
	winner := "Ruth"

	game := &GameSpy{BlindAlert: []byte(wantedBlindAlert)}
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(tenMS)

	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)
	within(t, tenMS, func() { assertWebsocketGotMsg(t, ws, wantedBlindAlert) })
})
```

Maintenant, si vous exécutez le test...

```
=== RUN   TestGame
=== RUN   TestGame/start_a_game_with_3_players,_send_some_blind_alerts_down_WS_and_declare_Ruth_the_winner
--- FAIL: TestGame (0.02s)
    --- FAIL: TestGame/start_a_game_with_3_players,_send_some_blind_alerts_down_WS_and_declare_Ruth_the_winner (0.02s)
    	server_test.go:143: timed out
    	server_test.go:150: got "", want "Blind is 100"
```

## Écrivez suffisamment de code pour le faire passer

Enfin, nous pouvons maintenant changer notre code serveur, pour qu'il envoie notre connexion WebSocket au jeu quand il démarre :

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, ws)

	winner := ws.WaitForMsg()
	p.game.Finish(winner)
}
```

## Refactorisation

Le code du serveur était un très petit changement, il n'y a donc pas beaucoup à changer ici, mais le code de test contient toujours un appel `time.Sleep` car nous devons attendre que notre serveur fasse son travail de manière asynchrone.

Nous pouvons refactoriser nos helpers `assertGameStartedWith` et `assertFinishCalledWith` pour qu'ils puissent réessayer leurs assertions pendant une courte période avant d'échouer.

Voici comment vous pouvez le faire pour `assertFinishCalledWith` et vous pouvez utiliser la même approche pour l'autre helper :

```go
func assertFinishCalledWith(t testing.TB, game *GameSpy, winner string) {
	t.Helper()

	passed := retryUntil(500*time.Millisecond, func() bool {
		return game.FinishCalledWith == winner
	})

	if !passed {
		t.Errorf("expected finish called with %q but got %q", winner, game.FinishCalledWith)
	}
}
```

Voici comment `retryUntil` est défini :

```go
func retryUntil(d time.Duration, f func() bool) bool {
	deadline := time.Now().Add(d)
	for time.Now().Before(deadline) {
		if f() {
			return true
		}
	}
	return false
}
```

## Récapitulation

Notre application est maintenant complète. Un jeu de poker peut être lancé via un navigateur web et les utilisateurs sont informés de la valeur de la mise aveugle au fil du temps via WebSockets. Lorsque la partie se termine, ils peuvent enregistrer le gagnant qui est persisté en utilisant le code que nous avons écrit il y a quelques chapitres. Les joueurs peuvent découvrir qui est le meilleur (ou le plus chanceux) joueur de poker en utilisant le point de terminaison `/league` du site web.

Au cours de ce voyage, nous avons commis des erreurs, mais avec le flux TDD, nous n'avons jamais été très loin d'un logiciel fonctionnel. Nous étions libres de continuer à itérer et à expérimenter.

Le dernier chapitre fera une rétrospective sur l'approche, la conception à laquelle nous sommes arrivés et réglera certains détails.

Nous avons couvert plusieurs choses dans ce chapitre :

### WebSockets

- Une façon pratique d'envoyer des messages entre les clients et les serveurs qui ne nécessite pas que le client interroge constamment le serveur. Le code client et serveur que nous avons est très simple.
- Trivial à tester, mais vous devez faire attention à la nature asynchrone des tests.

### Gérer le code dans les tests qui peut être retardé ou ne jamais se terminer

- Créez des fonctions d'aide pour réessayer les assertions et ajouter des délais d'expiration.
- Nous pouvons utiliser des goroutines pour garantir que les assertions ne bloquent rien, puis utiliser des canaux pour leur permettre de signaler qu'elles ont terminé, ou non.
- Le package `time` contient des fonctions utiles qui envoient également des signaux via des canaux sur des événements dans le temps afin que nous puissions définir des délais d'expiration.
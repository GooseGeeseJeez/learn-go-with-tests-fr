# Context

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/context)**

Les logiciels démarrent souvent des processus longs et intensifs en ressources (souvent dans des goroutines). Si l'action qui a provoqué cela est annulée ou échoue pour une raison quelconque, vous devez arrêter ces processus de manière cohérente dans votre application.

Si vous ne gérez pas cela, votre application Go rapide dont vous êtes si fier pourrait commencer à avoir des problèmes de performance difficiles à déboguer.

Dans ce chapitre, nous utiliserons le package `context` pour nous aider à gérer les processus de longue durée.

Nous allons commencer par un exemple classique d'un serveur web qui, lorsqu'il est sollicité, démarre un processus potentiellement long pour récupérer des données à renvoyer dans la réponse.

Nous allons tester un scénario où un utilisateur annule la requête avant que les données ne puissent être récupérées et nous nous assurerons que le processus est informé d'abandonner.

J'ai mis en place du code pour le cas nominal pour nous permettre de démarrer. Voici notre code serveur :

```go
func Serveur(magasin Magasin) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, magasin.Recuperer())
	}
}
```

C'est une fonction qui prend un `Magasin` et retourne une fonction qui est appelée lorsque la fonction `Serveur` est appelée.

`Magasin` est une interface qui nous permet d'extraire le récupérateur de données et de le simuler pour nos tests.

```go
type Magasin interface {
	Recuperer() string
}
```

Voici notre test pour vérifier que tout fonctionne.

```go
func TestServeur(t *testing.T) {
	data := "hello, world"
	svr := Serveur(&MagasinStub{data})

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`reçu "%s", attendu "%s"`, response.Body.String(), data)
	}
}

type MagasinStub struct {
	reponse string
}

func (s *MagasinStub) Recuperer() string {
	return s.reponse
}
```

Pour nos premiers tests, nous utilisons un `MagasinStub` pour contrôler les données renvoyées par la méthode `Recuperer`. Nous créons également un serveur web, faisons une requête puis vérifions que nous recevons la réponse que nous attendons.

Voyons maintenant un scénario plus réaliste où la fonction prend du temps à s'exécuter. Pour ce faire, nous allons modifier notre méthode `Recuperer` pour qu'elle prenne un peu de temps à être exécutée.

```go
func (s *MagasinLent) Recuperer() string {
	time.Sleep(100 * time.Millisecond)
	return s.reponse
}
```

Et puis ajoutons un nouveau test

```go
t.Run("indique au magasin d'annuler le travail si la requête est annulée", func(t *testing.T) {
	data := "hello, world"
	magasin := &MagasinEspion{reponse: data}
	svr := Serveur(magasin)

	request := httptest.NewRequest(http.MethodGet, "/", nil)

	cancellingCtx, cancel := context.WithCancel(request.Context())
	time.AfterFunc(5*time.Millisecond, cancel)
	request = request.WithContext(cancellingCtx)

	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if magasin.annuleCalled {
		t.Error("magasin.Annuler n'a pas été appelé")
	}
})
```

Si vous vous rappelez de la section sur la concurrence, nous avons utilisé le test helper de Go pour faire des requêtes HTTP. Un des arguments de la fonction handler est `*http.Request`, qui a un champ `Context`.

L'idée est que vous puissiez créer une hiérarchie de contextes pour que votre code puisse être informé de l'annulation de manière cohérente, depuis l'utilisateur annulant la requête HTTP jusqu'à la goroutine au plus profond de votre code. Lorsque l'utilisateur annule la requête, le contexte est annulé et toutes les goroutines utilisant ce contexte sont informées de l'annulation. 

Notre test crée un contexte qui est annulé après un certain délai (beaucoup plus court que le temps nécessaire pour que notre magasin termine son travail) et vérifie que notre magasin est informé de l'annulation. 

Le test ne va pas compiler car notre `MagasinEspion` n'a pas encore de champ `annuleCalled`. Créons maintenant une nouvelle implémentation de `Magasin` qui enregistre si `Annuler` a été appelé.

```go
type MagasinEspion struct {
	reponse string
	annuleCalled bool
}

func (s *MagasinEspion) Recuperer() string {
	time.Sleep(100 * time.Millisecond)
	return s.reponse
}

func (s *MagasinEspion) Annuler() {
	s.annuleCalled = true
}
```

Nous implémentons également une méthode `Annuler` qui enregistre si elle a été appelée.

Notre test compile maintenant mais échoue car nous n'avons pas encore mis à jour notre fonction `Serveur` pour informer le magasin de l'annulation.

```
=== RUN   TestServeur/indique_au_magasin_d'annuler_le_travail_si_la_requête_est_annulée
    --- FAIL: TestServeur/indique_au_magasin_d'annuler_le_travail_si_la_requête_est_annulée (0.01s)
        context_test.go:48: magasin.Annuler n'a pas été appelé
```

Nous devons d'abord changer notre interface `Magasin` pour refléter les nouvelles exigences.

```go
type Magasin interface {
	Recuperer() string
	Annuler()
}
```

Ensuite, nous pouvons mettre à jour notre fonction `Serveur` pour vérifier si le contexte a été annulé et appeler `Annuler` si nécessaire.

```go
func Serveur(magasin Magasin) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()

		data := make(chan string, 1)

		go func() {
			data <- magasin.Recuperer()
		}()

		select {
		case d := <-data:
			fmt.Fprint(w, d)
		case <-ctx.Done():
			magasin.Annuler()
		}
	}
}
```

Voici ce qui se passe maintenant :

1. Nous extrayons le contexte de la requête.
2. Nous créons un canal pour communiquer avec la goroutine qui appellera `Recuperer`.
3. Nous lançons une goroutine qui appelle `Recuperer` et envoie le résultat sur le canal.
4. Nous utilisons `select` pour attendre soit que le canal `data` reçoive une valeur, soit que le contexte soit annulé.
5. Si le contexte est annulé, nous appelons `Annuler` sur le magasin.

Maintenant, notre test devrait passer, car nous appelons `Annuler` sur le magasin lorsque le contexte est annulé.

Cependant, il y a un problème avec notre implémentation actuelle. Si vous regardez attentivement, vous verrez que nous avons une goroutine qui appelle `Recuperer`, mais cette goroutine n'est pas informée de l'annulation du contexte. 

C'est là que l'interface `context.Context` devient vraiment utile. Elle nous permet de transmettre le contexte à travers les appels de fonction, afin que chaque fonction puisse être informée de l'annulation.

Modifions notre interface `Magasin` pour accepter un contexte.

```go
type Magasin interface {
	Recuperer(ctx context.Context) string
}
```

Nous n'avons plus besoin de la méthode `Annuler` car nous pouvons simplement vérifier si le contexte a été annulé à l'intérieur de `Recuperer`.

Maintenant, mettons à jour notre fonction `Serveur` pour passer le contexte à `Recuperer`.

```go
func Serveur(magasin Magasin) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		data := magasin.Recuperer(r.Context())
		fmt.Fprint(w, data)
	}
}
```

C'est beaucoup plus simple ! Nous n'avons plus besoin de créer de goroutine ou d'utiliser un canal, car c'est maintenant la responsabilité de `Recuperer` de gérer l'annulation.

Maintenant, mettons à jour notre stub et nos espions pour implémenter la nouvelle interface.

```go
type MagasinStub struct {
	reponse string
}

func (s *MagasinStub) Recuperer(ctx context.Context) string {
	return s.reponse
}

type MagasinEspion struct {
	reponse      string
	annuleCalled bool
}

func (s *MagasinEspion) Recuperer(ctx context.Context) string {
	data := make(chan string, 1)

	go func() {
		var result string
		for _, c := range s.reponse {
			select {
			case <-ctx.Done():
				s.annuleCalled = true
				return
			default:
				time.Sleep(10 * time.Millisecond)
				result += string(c)
			}
		}
		data <- result
	}()

	select {
	case <-ctx.Done():
		return ""
	case res := <-data:
		return res
	}
}
```

Dans `MagasinEspion`, nous simulons un travail long en parcourant chaque caractère de la réponse et en attendant un peu. Nous vérifions régulièrement si le contexte a été annulé, et si c'est le cas, nous définissons `annuleCalled` à `true` et retournons.

Cependant, avec ces changements, notre test ne teste plus vraiment ce que nous voulons. Il ne vérifie plus si `Annuler` a été appelé, car nous n'avons plus cette méthode. Au lieu de cela, nous voulons vérifier que `Recuperer` a été informé de l'annulation du contexte.

Modifions notre test pour vérifier cela.

```go
t.Run("indique au magasin d'annuler le travail si la requête est annulée", func(t *testing.T) {
	data := "hello, world"
	magasin := &MagasinEspion{reponse: data}
	svr := Serveur(magasin)

	request := httptest.NewRequest(http.MethodGet, "/", nil)

	cancellingCtx, cancel := context.WithCancel(request.Context())
	time.AfterFunc(5*time.Millisecond, cancel)
	request = request.WithContext(cancellingCtx)

	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if !magasin.annuleCalled {
		t.Error("le magasin n'a pas été informé de l'annulation")
	}
})
```

Maintenant, nous vérifions que `annuleCalled` est `true`, ce qui indique que `Recuperer` a été informé de l'annulation du contexte.

Dans cette approche, nous utilisons le pattern "context-aware" pour gérer les annulations. En transmettant le contexte à travers les appels de fonction, nous permettons à chaque fonction de vérifier si elle doit continuer son travail ou s'arrêter.

## Résumé

- Le package `context` est très utile pour gérer les annulations dans les applications Go, en particulier dans les serveurs web.
- En transmettant un contexte à travers les appels de fonction, vous pouvez vous assurer que toutes les goroutines sont informées lorsqu'une opération est annulée.
- C'est un moyen propre et cohérent de gérer les annulations à travers votre application.
- Utilisez le pattern "context-aware" en faisant accepter un contexte par vos fonctions et en vérifiant régulièrement si le contexte a été annulé.
- N'oubliez pas que le contexte doit être le premier paramètre d'une fonction par convention.

## Ressources supplémentaires

- [Go Concurrency Patterns: Context](https://blog.golang.org/context)
- [Package context](https://golang.org/pkg/context/)
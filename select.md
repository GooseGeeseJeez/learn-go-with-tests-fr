# Select

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/select)**

On vous a demandé de créer une fonction appelée `CourseDesSites` qui prend deux URLs et les fait "s'affronter" en effectuant une requête HTTP GET et en retournant l'URL qui a répondu en premier. Si aucune ne répond dans un délai de 10 secondes, alors la fonction doit retourner une `error`.

Pour cela, nous utiliserons :

- `net/http` pour effectuer les appels HTTP.
- `net/http/httptest` pour nous aider à les tester.
- Les goroutines.
- `select` pour synchroniser les processus.

## Écrivez le test d'abord

Commençons par quelque chose de simple pour nous lancer.

```go
func TestCourseDesSites(t *testing.T) {
	siteRapide := "https://www.quii.dev"
	siteLent := "http://www.facebook.com"

	attendu := siteRapide
	resultat := CourseDesSites(siteRapide, siteLent)

	if resultat != attendu {
		t.Errorf("resultat '%s', attendu '%s'", resultat, attendu)
	}
}
```

Nous savons que ce n'est pas parfait et qu'il y a des hypothèses discutables dans notre test, mais c'est suffisant pour commencer. Il est important de ne pas trop se préoccuper de l'exactitude dès le départ.

## Essayons d'exécuter le test

```
./racer_test.go:14:9: undefined: CourseDesSites
```

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

```go
func CourseDesSites(a, b string) (gagnant string) {
	return
}
```

```
--- FAIL: TestCourseDesSites (0.00s)
    racer_test.go:12: resultat '', attendu 'https://www.quii.dev'
```

## Écrivez assez de code pour le faire passer

```go
func CourseDesSites(a, b string) (gagnant string) {
	demarreA := time.Now()
	http.Get(a)
	dureeA := time.Since(demarreA)

	demarreB := time.Now()
	http.Get(b)
	dureeB := time.Since(demarreB)

	if dureeA < dureeB {
		return a
	}

	return b
}
```

Pour ce test nous :

1. Mesurons le temps qu'il faut pour faire `Get(a)`
2. Mesurons le temps qu'il faut pour faire `Get(b)`
3. Si `a` est plus rapide, on retourne `a`, sinon on retourne `b`

Refactorisons-le un peu, puisqu'il est assez répétitif.

```go
func CourseDesSites(a, b string) (gagnant string) {
	aDuration := mesurerTempsDAppel(a)
	bDuration := mesurerTempsDAppel(b)

	if aDuration < bDuration {
		return a
	}

	return b
}

func mesurerTempsDAppel(url string) time.Duration {
	debut := time.Now()
	http.Get(url)
	return time.Since(debut)
}
```

### Problèmes avec notre approche

Nous avons une solution qui fonctionnera la _plupart du temps_ mais elle est un peu naïve.

Quels sont les problèmes potentiels ?

- Si un des sites n'existe pas, nous allons toujours obtenir une réponse mais cela se manifestera par une erreur de type HTTP. Nous n'utilisons pas le retour des valeurs de `http.Get`.
- Nous avons du mal à les tester car nous dépendons de sites web externes.
- Et même s'ils existent, il est évidemment risqué de dépendre d'un site web externe pour nos tests - il faut, au minimum, accepter qu'ils soient un peu flaky.

Commençons par résoudre le problème que nous avons créé en dépendant de sites web du monde réel en créant nos propres serveurs web pour les tester.

## Écrivez le test d'abord

Supprimons notre test précédent et écrivons un nouveau qui utilise des serveurs HTTP que nous allons contrôler.

```go
func TestCourseDesSites(t *testing.T) {
	serveurLent := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(20 * time.Millisecond)
		w.WriteHeader(http.StatusOK)
	}))

	serveurRapide := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	serveurLent.URL, serveurRapide.URL
	attendu := serveurRapide.URL
	resultat := CourseDesSites(serveurLent.URL, serveurRapide.URL)

	if resultat != attendu {
		t.Errorf("resultat '%s', attendu '%s'", resultat, attendu)
	}

	serveurLent.Close()
	serveurRapide.Close()
}
```

Le package `httptest` est vraiment pratique pour tester du code qui utilise HTTP. Dans notre cas, nous utilisons [`NewServer`](https://golang.org/pkg/net/http/httptest/#NewServer) pour créer deux serveurs web. Une des fonctions qui simule un serveur lent qui va attendre 20 millisecondes avant de répondre, et l'autre qui retourne immédiatement une réponse.

Quand nous créons un serveur web avec `NewServer`, il lance un serveur réel qui écoute sur une adresse HTTP réelle, que vous pouvez récupérer à partir de la propriété `URL`. Nous utilisons cette adresse dans nos tests.

Ne manquez pas de `defer` la fermeture des serveurs à la fin des tests, sinon ils continueront à écouter sur les adresses HTTP après la fin des tests.

## Essayons d'exécuter le test

```
=== RUN   TestCourseDesSites
--- FAIL: TestCourseDesSites (0.05s)
    racer_test.go:31: resultat 'http://127.0.0.1:57354', attendu 'http://127.0.0.1:57355'
```

Il est difficile de savoir exactement pourquoi il échoue en raison de la façon dont le test a été écrit. Ce n'est pas très clair quelle URL doit être la plus rapide. Créons une fonction d'aide pour rendre cela plus explicite.

```go
func TestCourseDesSites(t *testing.T) {
	serveurLent := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(20 * time.Millisecond)
		w.WriteHeader(http.StatusOK)
	}))

	serveurRapide := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	defer serveurLent.Close()
	defer serveurRapide.Close()

	URLLente := serveurLent.URL
	URLRapide := serveurRapide.URL

	attendu := URLRapide
	resultat := CourseDesSites(URLLente, URLRapide)

	if resultat != attendu {
		t.Errorf("resultat '%s', attendu '%s'", resultat, attendu)
	}
}
```

C'est beaucoup plus clair maintenant quant à ce qui devrait se passer.

## Essayons d'exécuter le test

```
=== RUN   TestCourseDesSites
--- FAIL: TestCourseDesSites (0.05s)
    racer_test.go:33: resultat 'http://127.0.0.1:57106', attendu 'http://127.0.0.1:57105'
```

Nous échouons toujours, mais pourquoi ? Je crois que c'est parce que, même si le serveur "lent" a un délai de 20 millisecondes, c'est si rapide que le problème n'apparaît pas toujours.

Renforçons le délai pour être sûr.

```go
serveurLent := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	time.Sleep(20 * time.Millisecond)
	w.WriteHeader(http.StatusOK)
}))
```

Si vous exécutez le test à plusieurs reprises, vous devriez voir qu'il passe. Étant donné que le problème vient de notre test, ajouter un "fix" comme celui-ci est acceptable. Maintenant, nous avons un test qui échoue si nous n'obtenons pas le bon résultat.

### Problème avec notre approche

Notre test a un risque de flakiness (instabilité). Il risque d'échouer parfois. Ce n'est évidemment pas idéal. Qu'est-ce que nous pourrions faire pour l'améliorer ?

- La fonction `CourseDesSites` est en train de mesurer le temps de réponse en effectuant des requêtes d'une manière séquentielle. C'est plutôt inefficace et risque de ne pas refléter comment les serveurs se comporteraient sur un ordinateur différent. Si vous aviez une machine assez lente, le premier appel pourrait ne pas être terminé avant que le second n'ait fini, même avec le délai.
- Nous ne voulons pas attendre 20 ms à chaque fois que nous exécutons ce test. Et que se passerait-il si nous ajoutons des tests qui vérifient des requêtes plus lentes ?

En réalité, nous voulons tester que notre fonction effectue les requêtes de manière concurrente, car c'est plus efficace que de les faire séquentiellement. Nous nous soucions moins du temps réel qu'il faut pour répondre à l'appel, mais plutôt de l'ordre dans lequel les requêtes sont terminées. Notre nouvelle solution devrait faire les appels HTTP simultanément et retourner l'URL du plus rapide.

## Écrivez assez de code pour le faire passer

```go
func CourseDesSites(a, b string) (gagnant string) {
	premierARepondre := make(chan string)

	go func() {
		http.Get(a)
		premierARepondre <- a
	}()

	go func() {
		http.Get(b)
		premierARepondre <- b
	}()

	return <-premierARepondre
}
```

- Nous avons créé un `chan string` appelé `premierARepondre`.
- Pour chaque URL, nous avons lancé une goroutine qui va tenter de récupérer l'URL et ensuite l'écrire dans le canal `premierARepondre`.
- Nous retournons la première réponse qui est écrite dans notre canal.

### Problèmes avec notre approche

Notre problème est que c'est toujours un peu lent. Chaque requête vers un serveur fictif peut répondre à des vitesses différentes, ce qui peut provoquer des flakiness dans nos tests. Nous voulons avoir un contrôle total, mais sans attendre des secondes pour que nos tests s'exécutent.

C'est là qu'entre en jeu `net/http/httptest`.

Modifions notre fonction `mesurerTempsDAppel` pour qu'elle n'utilise pas de vrais serveurs HTTP.

```go
func mesurerTempsDAppel(url string) time.Duration {
	debut := time.Now()
	http.Get(url)
	return time.Since(debut)
}
```

Nous voulons injecter (au sens d'injection de dépendances) du code qui peut être appelé par la fonction `CourseDesSites` et qui retournera un code HTTP. Pour ce faire, nous pouvons utiliser une interface.

```go
type Fetcher interface {
	Fetch() (string, error)
}
```

Maintenant nous pouvons changer notre fonction `CourseDesSites` pour qu'elle prenne en paramètre cette interface plutôt que des chaînes de caractères.

```go
func CourseDesSites(a, b Fetcher) (gagnant string, err error) {
	select {
	case <-a.Fetch():
		return a.URL(), nil
	case <-b.Fetch():
		return b.URL(), nil
	case <-time.After(10 * time.Second):
		return "", fmt.Errorf("délai d'attente dépassé pour %s et %s", a.URL(), b.URL())
	}
}
```

Le mot-clé `select` est comme un switch, mais pour les canaux. Les `case` définissent ce qui sera retourné lorsqu'un canal est écrit. Dans ce cas, nous renvoyons le nom de l'URL qui a écrit en premier dans son canal.

### Un nouveau besoin

Avant de pouvoir terminer ce travail, nous avons besoin d'une nouvelle exigence : nous voulons que `CourseDesSites` retourne une erreur si les deux sites web mettent plus de 10 secondes à répondre.

## Écrivez le test d'abord

```go
t.Run("retourne une erreur si un serveur ne répond pas dans le délai imparti", func(t *testing.T) {
	serveurA := creerServeurQuiMetLongtempsARepondre(11 * time.Second)
	serveurB := creerServeurQuiMetLongtempsARepondre(12 * time.Second)

	defer serveurA.Close()
	defer serveurB.Close()

	_, err := CourseDesSites(serveurA.URL, serveurB.URL)

	if err == nil {
		t.Error("on s'attendait à une erreur mais on n'en a pas reçu")
	}
})

func creerServeurQuiMetLongtempsARepondre(delai time.Duration) *httptest.Server {
	return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(delai)
		w.WriteHeader(http.StatusOK)
	}))
}
```

Nous avons défini deux serveurs qui prennent plus que le délai d'attente pour répondre. Ensuite, nous vérifions que notre fonction retourne une erreur.

## Essayons d'exécuter le test

```
=== RUN   TestCourseDesSites
    --- FAIL: TestCourseDesSites (0.00s)
        --- FAIL: TestCourseDesSites/retourne_une_erreur_si_un_serveur_ne_répond_pas_dans_le_délai_imparti (0.00s)
            racer_test.go:40: on s'attendait à une erreur mais on n'en a pas reçu
```

## Écrivez assez de code pour le faire passer

```go
func CourseDesSites(a, b string) (gagnant string, err error) {
	return ConfigCourseDesSites(a, b, 10*time.Second)
}

func ConfigCourseDesSites(a, b string, timeout time.Duration) (gagnant string, err error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(timeout):
		return "", fmt.Errorf("délai d'attente dépassé pour %s et %s", a, b)
	}
}

func ping(url string) chan struct{} {
	ch := make(chan struct{})
	go func() {
		http.Get(url)
		close(ch)
	}()
	return ch
}
```

Le mot-clé `select` est comme un switch, mais pour les canaux. Les `case` définissent ce qui sera retourné lorsqu'un canal est écrit. Dans ce cas, nous renvoyons le nom de l'URL qui a écrit en premier dans son canal.

Notre fonction `ping` crée un canal et ensuite lance une goroutine qui essaie de récupérer l'URL puis ferme le canal après avoir terminé. Remarquez que le canal a un type `chan struct{}` qui est le plus petit type disponible à partir de la mémoire. Nous ne sommes pas intéressés par le contenu du canal, seulement par sa propriété de signalisation.

La troisième instruction `case` utilise `time.After` qui est une fonction très pratique lors de l'utilisation de canaux. Elle renvoie un `chan` et écrira dessus après la durée spécifiée. Dans notre cas, on l'utilise pour définir un délai d'attente.

## Refactoriser

Nous pouvons revoir légèrement l'API pour ajouter une flexibilité supplémentaire à la fonction `CourseDesSites` :

```go
var timeout = 10 * time.Second

func CourseDesSites(a, b string) (gagnant string, err error) {
	return ConfigCourseDesSites(a, b, timeout)
}

func ConfigCourseDesSites(a, b string, timeout time.Duration) (gagnant string, err error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(timeout):
		return "", fmt.Errorf("délai d'attente dépassé pour %s et %s", a, b)
	}
}
```

Nous avons ajouté une variable de timeout pour éviter la magie des nombres, et nous avons fourni une fonction supplémentaire qui peut être utilisée par les utilisateurs qui sont plus préoccupés par les timeouts.

## Conclusion

### `select`

- Utile pour gérer plusieurs canaux.
- Le premier canal qui envoie une valeur "gagne".
- Peut être utilisé avec `time.After` pour une gestion élégante des délais d'attente.

### `httptest`

- Une bibliothèque pratique pour tester le code qui utilise HTTP.
- Utilise la même interface que les serveurs HTTP "réels", ce qui rend vos tests complets mais rapides.
- Capable de créer des serveurs HTTP factices que nous pouvons injecter dans notre code et contrôler leur temps de réponse.

### Concurrence

- Goroutines - fonctions concurrentes.
- Canaux - pour communiquer et synchroniser entre goroutines.
- Select - pour gérer les canaux de manière élégante.

### Liens intéressants

- [Concurrency is not parallelism](https://blog.golang.org/concurrency-is-not-parallelism) par Rob Pike
- [Go by Example: Select](https://gobyexample.com/select)
- [Advanced Go Concurrency Patterns](https://blog.golang.org/advanced-go-concurrency-patterns) par Sameer Ajmani
- [sync.WaitGroup](https://golang.org/pkg/sync/#WaitGroup) est une autre façon de synchroniser les goroutines.

## Récapitulatif

En travaillant avec `select`, nous avons exploré:

1. L'utilisation de goroutines et de canaux pour réaliser une concurrence efficace.
2. L'utilisation de `select` pour coordonner des opérations concurrentes.
3. L'utilisation de `httptest` pour tester le code HTTP de manière fiable.
4. L'implémentation d'un mécanisme de timeout élégant avec `time.After`.

La concurrence en Go est une fonctionnalité puissante, mais elle nécessite une réflexion attentive. Les outils que nous avons explorés ici font partie des mécanismes fondamentaux pour construire des programmes concurrents efficaces et fiables en Go.
# Introduction aux tests d'acceptation

À mon travail (`$WORK`), nous avons rencontré le besoin d'avoir un "arrêt gracieux" (graceful shutdown) pour nos services. L'arrêt gracieux s'assure que votre système termine correctement son travail avant d'être arrêté. Une analogie de la vie réelle serait quelqu'un qui essaie de terminer correctement un appel téléphonique avant de passer à la réunion suivante, plutôt que de raccrocher au milieu d'une phrase.

Ce chapitre présentera une introduction à l'arrêt gracieux dans le contexte d'un serveur HTTP et comment écrire des "tests d'acceptation" pour vous donner confiance dans le comportement de votre code.

Après avoir lu ce chapitre, vous saurez comment partager des packages avec d'excellents tests, réduire les efforts de maintenance et augmenter la confiance dans la qualité de votre travail.

## Juste assez d'informations sur Kubernetes

Nous exécutons notre logiciel sur [Kubernetes](https://kubernetes.io/) (K8s). K8s mettra fin aux "pods" (en pratique, notre logiciel) pour diverses raisons, et une raison courante est lorsque nous poussons du nouveau code que nous voulons déployer.

Nous nous fixons des normes élevées concernant les [métriques DORA](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance), nous travaillons donc de manière à déployer de petites améliorations et fonctionnalités incrémentielles en production plusieurs fois par jour.

Lorsque K8s souhaite mettre fin à un pod, il lance un ["cycle de vie de terminaison"](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace), et une partie de ce processus consiste à envoyer un signal SIGTERM à notre logiciel. C'est K8s qui dit à notre code :

> Vous devez vous arrêter, terminer tout travail en cours car après une certaine "période de grâce", j'enverrai `SIGKILL`, et ce sera la fin pour vous.

Lors d'un `SIGKILL`, tout travail que votre programme pourrait être en train de faire sera immédiatement arrêté.

## Si vous n'avez pas de grâce

Selon la nature de votre logiciel, si vous ignorez `SIGTERM`, vous pouvez rencontrer des problèmes.

Notre problème spécifique concernait les requêtes HTTP en cours. Lorsqu'un test automatisé exerçait notre API, si K8s décidait d'arrêter le pod, le serveur mourait, le test ne recevait pas de réponse du serveur et le test échouait.

Cela déclencherait une alerte dans notre canal d'incidents, ce qui nécessite qu'un développeur arrête ce qu'il fait et s'attaque au problème. Ces échecs intermittents sont une distraction ennuyeuse pour notre équipe.

Ces problèmes ne sont pas uniques à nos tests. Si un utilisateur envoie une requête à votre système et que le processus est interrompu en plein vol, il sera probablement accueilli par une erreur HTTP 5xx, ce qui n'est pas le genre d'expérience utilisateur que vous souhaitez offrir.

## Quand vous avez de la grâce

Ce que nous voulons faire, c'est écouter le signal `SIGTERM`, et plutôt que de tuer instantanément le serveur, nous voulons :

- Arrêter d'écouter de nouvelles requêtes
- Permettre à toutes les requêtes en cours de se terminer
- *Ensuite* terminer le processus

## Comment avoir de la grâce

Heureusement, Go dispose déjà d'un mécanisme pour arrêter gracieusement un serveur avec [net/http/Server.Shutdown](https://pkg.go.dev/net/http#Server.Shutdown).

> Shutdown arrête gracieusement le serveur sans interrompre les connexions actives. Shutdown fonctionne en fermant d'abord tous les écouteurs ouverts, puis en fermant toutes les connexions inactives, puis en attendant indéfiniment que les connexions reviennent à l'état inactif avant de les fermer. Si le contexte fourni expire avant que l'arrêt ne soit terminé, Shutdown renvoie l'erreur du contexte, sinon il renvoie toute erreur renvoyée par la fermeture des écouteurs sous-jacents du serveur.

Pour gérer `SIGTERM`, nous pouvons utiliser [os/signal.Notify](https://pkg.go.dev/os/signal#Notify), qui enverra tous les signaux entrants à un canal que nous fournissons.

En utilisant ces deux fonctionnalités de la bibliothèque standard, vous pouvez écouter `SIGTERM` et vous arrêter gracieusement.

## Package d'arrêt gracieux

À cette fin, j'ai écrit [https://pkg.go.dev/github.com/quii/go-graceful-shutdown](https://pkg.go.dev/github.com/quii/go-graceful-shutdown). Il fournit une fonction décorateur pour un `*http.Server` pour appeler sa méthode `Shutdown` lorsqu'un signal `SIGTERM` est détecté.

```go
func main() {
	var (
		ctx        = context.Background()
		httpServer = &http.Server{Addr: ":8080", Handler: http.HandlerFunc(acceptancetests.SlowHandler)}
		server     = gracefulshutdown.NewServer(httpServer)
	)

	if err := server.ListenAndServe(ctx); err != nil {
		// cela se produira généralement si nos réponses ne sont pas écrites avant la date limite du ctx, pas grand-chose à faire
		log.Fatalf("oh non, l'arrêt n'a pas été gracieux, certaines réponses ont peut-être été perdues %v", err)
	}

	// espérons que vous verrez toujours ceci à la place
	log.Println("arrêt gracieux ! toutes les réponses ont été envoyées")
}
```

Les détails concernant le code ne sont pas trop importants pour cette lecture, mais il vaut la peine de jeter un coup d'œil rapide au code avant de continuer.

## Tests et boucles de feedback

Lorsque nous avons écrit le package `gracefulshutdown`, nous avions des tests unitaires pour prouver qu'il se comporte correctement, ce qui nous a donné la confiance nécessaire pour refactoriser de manière agressive. Cependant, nous ne nous sentions toujours pas "confiants" qu'il fonctionnait **vraiment**.

Nous avons ajouté un package `cmd` et créé un vrai programme pour utiliser le package que nous écrivions. Nous le lancions manuellement, envoyions une requête HTTP, puis envoyions un `SIGTERM` pour voir ce qui se passerait.

**L'ingénieur en vous devrait se sentir mal à l'aise avec les tests manuels**.
C'est ennuyeux, ça ne s'adapte pas à l'échelle, c'est imprécis et c'est du gaspillage. Si vous écrivez un package que vous avez l'intention de partager, mais que vous voulez aussi le garder simple et peu coûteux à modifier, les tests manuels ne suffiront pas.

## Tests d'acceptation

Si vous avez lu le reste de ce livre, vous avez principalement écrit des "tests unitaires". Les tests unitaires sont un outil fantastique pour permettre une refactorisation sans crainte, favoriser une bonne conception modulaire, éviter les régressions et faciliter un retour d'information rapide.

Par leur nature, ils ne testent que de petites parties de votre système. Généralement, les tests unitaires seuls ne sont *pas suffisants* pour une stratégie de test efficace. Rappelez-vous, nous voulons que nos systèmes soient **toujours livrables**. Nous ne pouvons pas nous fier aux tests manuels, nous avons donc besoin d'un autre type de test : les **tests d'acceptation**.

### Que sont-ils ?

Les tests d'acceptation sont une sorte de "test en boîte noire". Ils sont parfois appelés "tests fonctionnels". Ils devraient exercer le système comme le ferait un utilisateur du système.

Le terme "boîte noire" fait référence à l'idée que le code de test n'a pas accès aux éléments internes du système, il ne peut utiliser que son interface publique et faire des assertions sur les comportements qu'il observe. Cela signifie qu'ils ne peuvent tester que le système dans son ensemble.

C'est un trait avantageux car cela signifie que les tests exercent le système de la même manière qu'un utilisateur le ferait, ils ne peuvent pas utiliser de contournements spéciaux qui pourraient faire passer un test, mais qui ne prouvent pas réellement ce que vous devez prouver. C'est similaire au principe de préférer que vos fichiers de test unitaires vivent dans un package de test séparé, par exemple, `package mypkg_test` plutôt que `package mypkg`.

### Avantages des tests d'acceptation

- Lorsqu'ils passent, vous savez que votre système entier se comporte comme vous le souhaitez.
- Ils sont plus précis, plus rapides et nécessitent moins d'efforts que les tests manuels.
- Lorsqu'ils sont bien écrits, ils agissent comme une documentation précise et vérifiée de votre système. Ils ne tombent pas dans le piège de la documentation qui diverge du comportement réel du système.
- Pas de mocking ! Tout est réel.

### Inconvénients potentiels par rapport aux tests unitaires

- Ils sont coûteux à écrire.
- Ils prennent plus de temps à s'exécuter.
- Ils dépendent de la conception du système.
- Lorsqu'ils échouent, ils ne donnent généralement pas la cause première et peuvent être difficiles à déboguer.
- Ils ne vous donnent pas de retour sur la qualité interne de votre système. Vous pourriez écrire n'importe quoi et faire passer un test d'acceptation.
- Tous les scénarios ne sont pas pratiques à exercer en raison de la nature de la boîte noire.

Pour cette raison, il est insensé de ne compter que sur les tests d'acceptation. Ils n'ont pas beaucoup des qualités que possèdent les tests unitaires, et un système avec un grand nombre de tests d'acceptation aura tendance à souffrir en termes de coûts de maintenance et de délai d'exécution médiocre.

#### Délai d'exécution ?

Le délai d'exécution fait référence au temps qu'il faut entre le moment où un commit est fusionné dans votre branche principale et son déploiement en production. Ce nombre peut varier de semaines, voire de mois pour certaines équipes, à quelques minutes. Encore une fois, à `$WORK`, nous valorisons les conclusions de DORA et voulons maintenir notre délai d'exécution à moins de 10 minutes.

Une approche de test équilibrée est nécessaire pour un système fiable avec un excellent délai d'exécution, et ceci est généralement décrit en termes de [Pyramide de Test](https://martinfowler.com/articles/practical-test-pyramid.html).

## Comment écrire des tests d'acceptation de base

Comment cela se rapporte-t-il au problème initial ? Nous venons d'écrire un package ici, et il est entièrement testable unitairement.

Comme je l'ai mentionné, les tests unitaires ne nous donnaient pas tout à fait la confiance dont nous avions besoin. Nous voulons être *vraiment* sûrs que le package fonctionne lorsqu'il est intégré à un programme réel et en cours d'exécution. Nous devrions pouvoir automatiser les vérifications manuelles que nous faisions.

Examinons le programme de test :

```go
func main() {
	var (
		ctx        = context.Background()
		httpServer = &http.Server{Addr: ":8080", Handler: http.HandlerFunc(acceptancetests.SlowHandler)}
		server     = gracefulshutdown.NewServer(httpServer)
	)

	if err := server.ListenAndServe(ctx); err != nil {
		// cela se produira généralement si nos réponses ne sont pas écrites avant la date limite du ctx, pas grand-chose à faire
		log.Fatalf("oh non, l'arrêt n'a pas été gracieux, certaines réponses ont peut-être été perdues %v", err)
	}

	// espérons que vous verrez toujours ceci à la place
	log.Println("arrêt gracieux ! toutes les réponses ont été envoyées")
}
```

Vous avez peut-être deviné que `SlowHandler` a un `time.Sleep` pour retarder la réponse, afin que j'aie le temps d'envoyer un `SIGTERM` et voir ce qui se passe. Le reste est assez standard :

- Créer un `net/http/Server` ;
- L'envelopper dans la bibliothèque (voir : [Patron Décorateur](https://en.wikipedia.org/wiki/Decorator_pattern)) ;
- Utiliser la version enveloppée pour `ListenAndServe`.

### Étapes de haut niveau pour le test d'acceptation

- Compiler le programme
- L'exécuter (et attendre qu'il écoute sur le port `8080`)
- Envoyer une requête HTTP au serveur
- Avant que le serveur n'ait la possibilité d'envoyer une réponse HTTP, envoyer `SIGTERM`
- Vérifier si nous recevons toujours une réponse

### Compiler et exécuter le programme

```go
package acceptancetests

import (
	"fmt"
	"math/rand"
	"net"
	"os"
	"os/exec"
	"path/filepath"
	"syscall"
	"time"
)

const (
	baseBinName = "temp-testbinary"
)

func LaunchTestProgram(port string) (cleanup func(), sendInterrupt func() error, err error) {
	binName, err := buildBinary()
	if err != nil {
		return nil, nil, err
	}

	sendInterrupt, kill, err := runServer(binName, port)

	cleanup = func() {
		if kill != nil {
			kill()
		}
		os.Remove(binName)
	}

	if err != nil {
		cleanup() // même s'il n'écoute pas correctement, le programme pourrait toujours être en cours d'exécution
		return nil, nil, err
	}

	return cleanup, sendInterrupt, nil
}

func buildBinary() (string, error) {
	binName := randomString(10) + "-" + baseBinName

	build := exec.Command("go", "build", "-o", binName)

	if err := build.Run(); err != nil {
		return "", fmt.Errorf("impossible de compiler l'outil %s: %s", binName, err)
	}
	return binName, nil
}

func runServer(binName string, port string) (sendInterrupt func() error, kill func(), err error) {
	dir, err := os.Getwd()
	if err != nil {
		return nil, nil, err
	}

	cmdPath := filepath.Join(dir, binName)

	cmd := exec.Command(cmdPath)

	if err := cmd.Start(); err != nil {
		return nil, nil, fmt.Errorf("impossible d'exécuter le convertisseur temp: %s", err)
	}

	kill = func() {
		_ = cmd.Process.Kill()
	}

	sendInterrupt = func() error {
		return cmd.Process.Signal(syscall.SIGTERM)
	}

	err = waitForServerListening(port)

	return
}

func waitForServerListening(port string) error {
	for i := 0; i < 30; i++ {
		conn, _ := net.Dial("tcp", net.JoinHostPort("localhost", port))
		if conn != nil {
			conn.Close()
			return nil
		}
		time.Sleep(100 * time.Millisecond)
	}
	return fmt.Errorf("rien ne semble écouter sur localhost:%s", port)
}

func randomString(n int) string {
	var letters = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")

	s := make([]rune, n)
	for i := range s {
		s[i] = letters[rand.Intn(len(letters))]
	}
	return string(s)
}
```

`LaunchTestProgram` est responsable de :
- compiler le programme
- lancer le programme
- attendre qu'il écoute sur le port `8080`
- fournir une fonction `cleanup` pour tuer le programme et le supprimer afin de s'assurer que lorsque nos tests se terminent, nous sommes laissés dans un état propre
- fournir une fonction `interrupt` pour envoyer au programme un `SIGTERM` afin de nous permettre de tester le comportement

Certes, ce n'est pas le plus beau code du monde, mais concentrez-vous simplement sur la fonction exportée `LaunchTestProgram`, les fonctions non exportées qu'elle appelle sont des boilerplates sans intérêt.

Comme nous l'avons vu, les tests d'acceptation ont tendance à être plus difficiles à mettre en place. Ce code rend le code de *test* substantiellement plus simple à lire, et souvent avec les tests d'acceptation, une fois que vous avez écrit le code cérémonial, c'est fait, et vous pouvez l'oublier.

### Le(s) test(s) d'acceptation

Nous voulions avoir deux tests d'acceptation pour deux programmes, un avec arrêt gracieux et un sans, afin que nous et les lecteurs puissions voir la différence de comportement. Avec `LaunchTestProgram` pour compiler et exécuter les programmes, il est assez simple d'écrire des tests d'acceptation pour les deux, et nous bénéficions de la réutilisation avec certaines fonctions d'aide.

Voici le test pour le serveur *avec* un arrêt gracieux, [vous pouvez trouver le test sans sur GitHub](https://github.com/quii/go-graceful-shutdown/blob/main/acceptancetests/withoutgracefulshutdown/main_test.go)

```go
package main

import (
	"testing"
	"time"

	"github.com/quii/go-graceful-shutdown/acceptancetests"
	"github.com/quii/go-graceful-shutdown/assert"
)

const (
	port = "8080"
	url  = "http://localhost:" + port
)

func TestGracefulShutdown(t *testing.T) {
	cleanup, sendInterrupt, err := acceptancetests.LaunchTestProgram(port)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(cleanup)

	// vérifions simplement que le serveur fonctionne avant d'arrêter les choses
	assert.CanGet(t, url)

	// lancer une requête, et avant qu'elle n'ait la possibilité de répondre, envoyer SIGTERM.
	time.AfterFunc(50*time.Millisecond, func() {
		assert.NoError(t, sendInterrupt())
	})
	// Sans arrêt gracieux, cela échouerait
	assert.CanGet(t, url)

	// après l'interruption, le serveur devrait être arrêté et aucune autre requête ne fonctionnera
	assert.CantGet(t, url)
}
```

Avec la configuration encapsulée, les tests sont complets, décrivent le comportement et sont relativement faciles à suivre.

`assert.CanGet/CantGet` sont des fonctions d'aide que j'ai créées pour ne pas répéter (DRY) cette assertion commune pour cette suite.

```go
func CanGet(t testing.TB, url string) {
	errChan := make(chan error)

	go func() {
		res, err := http.Get(url)
		if err != nil {
			errChan <- err
			return
		}
		res.Body.Close()
		errChan <- nil
	}()

	select {
	case err := <-errChan:
		NoError(t, err)
	case <-time.After(3 * time.Second):
		t.Errorf("délai d'attente dépassé pour la requête vers %q", url)
	}
}
```

Cela lancera un `GET` vers `URL` sur une goroutine, et s'il répond sans erreur avant 3 secondes, alors il ne échouera pas. `CantGet` est omis pour plus de concision, [mais vous pouvez le voir sur GitHub ici](https://github.com/quii/go-graceful-shutdown/blob/main/assert/assert.go#L61).

Il est important de noter à nouveau que Go dispose de tous les outils dont vous avez besoin pour écrire des tests d'acceptation prêts à l'emploi. Vous n'avez *pas besoin* d'un framework spécial pour créer des tests d'acceptation.

### Petit investissement avec un grand rendement

Avec ces tests, les lecteurs peuvent examiner les programmes d'exemple et être confiants que l'exemple fonctionne *réellement*, ils peuvent donc avoir confiance dans les affirmations du package.

Plus important encore, en tant qu'auteur, nous obtenons un **retour rapide** et une **confiance massive** que le package fonctionne dans un environnement réel.

```shell
go test -count=1 ./...
ok  	github.com/quii/go-graceful-shutdown	0.196s
?   	github.com/quii/go-graceful-shutdown/acceptancetests	[no test files]
ok  	github.com/quii/go-graceful-shutdown/acceptancetests/withgracefulshutdown	4.785s
ok  	github.com/quii/go-graceful-shutdown/acceptancetests/withoutgracefulshutdown	2.914s
?   	github.com/quii/go-graceful-shutdown/assert	[no test files]
```

## Conclusion

Dans ce billet de blog, nous avons introduit les tests d'acceptation dans votre boîte à outils de test. Ils sont inestimables lorsque vous commencez à construire de vrais systèmes et sont un complément important à vos tests unitaires.

La nature de *comment* écrire des tests d'acceptation dépend du système que vous construisez, mais les principes restent les mêmes. Traitez votre système comme une "boîte noire". Si vous créez un site web, vos tests devraient agir comme un utilisateur, vous voudrez donc utiliser un navigateur web headless comme [Selenium](https://www.selenium.dev/), pour cliquer sur des liens, remplir des formulaires, etc. Pour une API RESTful, vous enverrez des requêtes HTTP à l'aide d'un client.

### Aller plus loin pour des systèmes plus complexes

Les systèmes non triviaux n'ont pas tendance à être des applications à processus unique comme celle dont nous avons discuté. Généralement, vous dépendrez d'autres systèmes tels qu'une base de données. Pour ces scénarios, vous devrez automatiser un environnement local pour tester. Des outils comme [docker-compose](https://docs.docker.com/compose/) sont utiles pour démarrer des conteneurs de l'environnement dont vous avez besoin pour exécuter votre système localement.

### Le prochain chapitre

Dans ce billet, le test d'acceptation a été écrit rétrospectivement. Cependant, dans [Growing Object-Oriented Software](http://www.growing-object-oriented-software.com), les auteurs montrent que nous pouvons utiliser les tests d'acceptation dans une approche dirigée par les tests pour agir comme une "étoile du nord" pour guider nos efforts.

À mesure que les systèmes deviennent plus complexes, les coûts d'écriture et de maintenance des tests d'acceptation peuvent rapidement s'envoler. Il existe d'innombrables histoires d'équipes de développement entravées par des suites de tests d'acceptation coûteuses.

Le prochain chapitre présentera l'utilisation des tests d'acceptation pour guider notre conception ainsi que les principes et techniques pour gérer les coûts des tests d'acceptation.

### Améliorer la qualité de l'open-source

Si vous écrivez des packages que vous avez l'intention de partager, je vous encourage à créer des programmes d'exemple simples démontrant ce que fait votre package et à investir du temps dans la création de tests d'acceptation simples à suivre pour donner à vous-même et aux utilisateurs potentiels de votre travail la confiance nécessaire.

Comme les [Exemples testables](https://go.dev/blog/examples), voir ce petit effort supplémentaire dans l'expérience du développeur contribue grandement à instaurer la confiance dans votre travail et réduira vos propres coûts de maintenance.

## Plug de recrutement pour `$WORK`

Si vous avez envie de travailler dans un environnement avec d'autres ingénieurs résolvant des problèmes intéressants, que vous vivez près ou autour de Londres ou Porto, et que vous appréciez le contenu de ce chapitre et de ce livre - n'hésitez pas à [me contacter sur Twitter](https://twitter.com/quii), et peut-être que nous pourrons travailler ensemble bientôt !
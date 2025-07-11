# Apprendre Go avec les tests - Dimensionner les tests d'acceptation (et une légère introduction à gRPC)

Ce chapitre est une suite de [Introduction aux tests d'acceptation](https://quii.gitbook.io/learn-go-with-tests/testing-fundamentals/intro-to-acceptance-tests). Vous pouvez trouver [le code complet de ce chapitre sur GitHub](https://github.com/quii/go-specs-greet).

Les tests d'acceptation sont essentiels et impactent directement votre capacité à faire évoluer votre système avec confiance au fil du temps, à un coût de changement raisonnable.

C'est aussi un outil fantastique pour vous aider à travailler avec du code hérité. Face à une base de code médiocre sans tests, résistez à la tentation de commencer le refactoring. Au lieu de cela, écrivez des tests d'acceptation pour vous donner un filet de sécurité qui vous permettra de modifier librement les éléments internes du système sans affecter son comportement externe fonctionnel. Les tests d'acceptation n'ont pas besoin de se préoccuper de la qualité interne, ils sont donc parfaitement adaptés à ces situations.

Après cette lecture, vous apprécierez que les tests d'acceptation sont utiles pour la vérification et peuvent également être utilisés dans le processus de développement en nous aidant à modifier notre système de manière plus délibérée et méthodique, réduisant ainsi les efforts gaspillés.

## Prérequis

L'inspiration pour ce chapitre est née de nombreuses années de frustration avec les tests d'acceptation. Voici deux vidéos que je vous recommande de regarder :

- Dave Farley - [Comment écrire des tests d'acceptation](https://www.youtube.com/watch?v=JDD5EEJgpHU)
- Nat Pryce - [Tests fonctionnels de bout en bout qui peuvent s'exécuter en millisecondes](https://www.youtube.com/watch?v=Fk4rCn4YLLU)

"Growing Object Oriented Software" (GOOS) est un livre très important pour de nombreux ingénieurs logiciels, y compris moi-même. L'approche qu'il préconise est celle que j'enseigne aux ingénieurs avec qui je travaille.

- [GOOS](http://www.growing-object-oriented-software.com) - Nat Pryce & Steve Freeman

Enfin, [Riya Dattani](https://twitter.com/dattaniriya) et moi avons parlé de ce sujet dans le contexte du BDD dans notre présentation, [Tests d'acceptation, BDD et Go](https://www.youtube.com/watch?v=ZMWJCk_0WrY).

## Récapitulatif

Nous parlons de tests en "boîte noire" qui vérifient que votre système se comporte comme prévu de l'extérieur, d'un **point de vue métier**. Les tests n'ont pas accès aux éléments internes du système qu'ils testent ; ils ne se préoccupent que de **ce que** fait votre système plutôt que de **comment** il le fait.

## Anatomie des mauvais tests d'acceptation

Au cours de nombreuses années, j'ai travaillé pour plusieurs entreprises et équipes. Chacune d'entre elles a reconnu la nécessité de tests d'acceptation, une manière de tester un système du point de vue de l'utilisateur et de vérifier qu'il fonctionne comme prévu, mais presque sans exception, le coût de ces tests est devenu un réel problème pour l'équipe.

- Lents à exécuter
- Fragiles
- Instables
- Coûteux à maintenir, et semblent rendre les modifications du logiciel plus difficiles qu'elles ne devraient l'être
- Ne peuvent s'exécuter que dans un environnement particulier, ce qui entraîne des boucles de feedback lentes et médiocres

Supposons que vous ayez l'intention d'écrire un test d'acceptation pour un site web que vous construisez. Vous décidez d'utiliser un navigateur sans interface graphique (comme [Selenium](https://www.selenium.dev)) pour simuler un utilisateur cliquant sur des boutons de votre site web afin de vérifier qu'il fait ce qu'il est censé faire.

Au fil du temps, le balisage de votre site web doit changer à mesure que de nouvelles fonctionnalités sont découvertes, et les ingénieurs débattent pour savoir si quelque chose doit être un `<article>` ou une `<section>` pour la milliardième fois.

Même si votre équipe n'apporte que des modifications mineures au système, à peine perceptibles pour l'utilisateur réel, vous vous retrouvez à perdre beaucoup de temps à mettre à jour vos tests d'acceptation.

### Couplage fort

Réfléchissez à ce qui déclenche la modification des tests d'acceptation :

- Un changement de comportement externe. Si vous souhaitez modifier ce que fait le système, il semble raisonnable, voire souhaitable, de modifier la suite de tests d'acceptation.
- Un changement de détail d'implémentation / refactoring. Idéalement, cela ne devrait pas nécessiter de changement, ou si c'est le cas, un changement mineur.

Trop souvent, cependant, la deuxième raison est celle pour laquelle les tests d'acceptation doivent être modifiés. Au point où les ingénieurs deviennent même réticents à modifier leur système en raison de l'effort perçu pour mettre à jour les tests !

![Riya et moi-même parlant de la séparation des préoccupations dans nos tests](https://i.imgur.com/bbG6z57.png)

Ces problèmes découlent du non-respect des habitudes d'ingénierie bien établies et pratiquées, écrites par les auteurs mentionnés ci-dessus. **Vous ne pouvez pas écrire des tests d'acceptation comme des tests unitaires** ; ils nécessitent plus de réflexion et des pratiques différentes.

## Anatomie des bons tests d'acceptation

Si nous voulons des tests d'acceptation qui ne changent que lorsque nous modifions le comportement et non les détails d'implémentation, il est logique que nous devions séparer ces préoccupations.

### Sur les types de complexité

En tant qu'ingénieurs logiciels, nous devons faire face à deux types de complexité.

- **La complexité accidentelle** est la complexité à laquelle nous devons faire face parce que nous travaillons avec des ordinateurs, des choses comme les réseaux, les disques, les API, etc.

- **La complexité essentielle** est parfois appelée "logique de domaine". Ce sont les règles et vérités particulières dans votre domaine.
  - Par exemple, "si un titulaire de compte retire plus d'argent que ce dont il dispose, il est à découvert". Cette déclaration ne dit rien sur les ordinateurs ; cette affirmation était vraie avant même que les ordinateurs ne soient utilisés dans les banques !

La complexité essentielle devrait être exprimable pour une personne non technique, et il est utile de l'avoir modélisée dans notre code "domaine" et dans nos tests d'acceptation.

### Séparation des préoccupations

Ce que Dave Farley a proposé dans la vidéo précédente, et ce dont Riya et moi avons également discuté, c'est que nous devrions avoir l'idée de **spécifications**. Les spécifications décrivent le comportement du système que nous voulons sans être couplées à la complexité accidentelle ou aux détails d'implémentation.

Cette idée devrait vous paraître raisonnable. Dans le code de production, nous nous efforçons fréquemment de séparer les préoccupations et de découpler les unités de travail. N'hésiteriez-vous pas à introduire une `interface` pour permettre à votre gestionnaire `HTTP` de le découpler des préoccupations non HTTP ? Appliquons cette même ligne de pensée à nos tests d'acceptation.

Dave Farley décrit une structure spécifique.

![Dave Farley sur les tests d'acceptation](https://i.imgur.com/nPwpihG.png)

À GopherconUK, Riya et moi avons mis cela en termes de Go.

![Séparation des préoccupations](https://i.imgur.com/qdY4RJe.png)

### Tests sur stéroïdes

Découpler la façon dont la spécification est exécutée nous permet de la réutiliser dans différents scénarios. Nous pouvons :

#### Rendre nos pilotes configurables

Cela signifie que vous pouvez exécuter vos tests d'acceptation localement, dans votre environnement de préproduction et (idéalement) de production.
- Trop d'équipes conçoivent leurs systèmes de telle sorte que les tests d'acceptation sont impossibles à exécuter localement. Cela introduit une boucle de feedback intolérablement lente. Ne préféreriez-vous pas être sûr que vos tests d'acceptation réussiront _avant_ d'intégrer votre code ? Si les tests commencent à échouer, est-il acceptable que vous ne puissiez pas reproduire l'échec localement et que vous deviez plutôt soumettre des modifications et croiser les doigts pour que cela passe 20 minutes plus tard dans un environnement différent ?
- N'oubliez pas que le fait que vos tests passent en préproduction ne signifie pas que votre système fonctionnera. La parité dev/prod est, au mieux, un petit mensonge. [Je teste en production](https://increment.com/testing/i-test-in-production/).
- Il y a toujours des différences entre les environnements qui peuvent affecter le *comportement* de votre système. Un CDN pourrait avoir des en-têtes de cache mal configurés ; un service en aval dont vous dépendez peut se comporter différemment ; une valeur de configuration peut être incorrecte. Mais ne serait-il pas agréable de pouvoir exécuter vos spécifications en production pour repérer rapidement ces problèmes ?

#### Connecter _différents_ pilotes pour tester d'autres parties de votre système

Cette flexibilité nous permet de tester les comportements à différents niveaux d'abstraction et architecturaux, ce qui nous permet d'avoir des tests plus ciblés au-delà des tests en boîte noire.
- Par exemple, vous pouvez avoir une page web avec une API derrière. Pourquoi ne pas utiliser la même spécification pour tester les deux ? Vous pouvez utiliser un navigateur sans interface graphique pour la page web et des appels HTTP pour l'API.
- En poussant cette idée plus loin, idéalement, nous voulons que le **code modélise la complexité essentielle** (comme code de "domaine"), nous devrions donc également pouvoir utiliser nos spécifications pour les tests unitaires. Cela donnera un retour rapide sur le fait que la complexité essentielle de notre système est modélisée et se comporte correctement.

### Les tests d'acceptation changent pour les bonnes raisons

Avec cette approche, la seule raison pour que vos spécifications changent est si le comportement du système change, ce qui est raisonnable.

- Si votre API HTTP doit changer, vous avez un endroit évident pour la mettre à jour, le pilote.
- Si votre balisage change, là encore, mettez à jour le pilote spécifique.

Au fur et à mesure que votre système se développe, vous vous retrouverez à réutiliser des pilotes pour plusieurs tests, ce qui signifie à nouveau que si les détails d'implémentation changent, vous n'avez qu'un seul endroit à mettre à jour, généralement évident.

Lorsqu'elle est bien faite, cette approche nous donne une flexibilité dans nos détails d'implémentation et une stabilité dans nos spécifications. Surtout, elle fournit une structure simple et évidente pour gérer les changements, ce qui devient essentiel à mesure qu'un système et son équipe grandissent.

### Les tests d'acceptation comme méthode de développement logiciel

Dans notre présentation, Riya et moi avons discuté des tests d'acceptation et de leur relation avec le BDD. Nous avons parlé de la façon dont le fait de commencer votre travail en essayant de _comprendre le problème que vous essayez de résoudre_ et de l'exprimer comme une spécification aide à concentrer votre intention et est une excellente façon de commencer votre travail.

J'ai été initié à cette façon de travailler dans GOOS. Il y a quelque temps, j'ai résumé les idées sur mon blog. Voici un extrait de mon article [Pourquoi le TDD](https://quii.dev/The_Why_of_TDD)

---

Le TDD est axé sur la conception du comportement dont vous avez précisément besoin, de manière itérative. Lorsque vous commencez une nouvelle zone, vous devez identifier un comportement clé, nécessaire et réduire agressivement la portée.

Suivez une approche "de haut en bas", en commençant par un test d'acceptation (AT) qui exerce le comportement de l'extérieur. Cela servira d'étoile du nord pour vos efforts. Tout ce sur quoi vous devriez vous concentrer, c'est de faire passer ce test. Ce test échouera probablement pendant un certain temps tandis que vous développez suffisamment de code pour le faire passer.

![](https://i.imgur.com/pxTaYu4.png)

Une fois que votre AT est configuré, vous pouvez passer au processus TDD pour produire suffisamment d'unités pour faire passer l'AT. L'astuce consiste à ne pas trop s'inquiéter de la conception à ce stade ; obtenez suffisamment de code pour faire passer l'AT car vous êtes encore en train d'apprendre et d'explorer le problème.

Cette première étape est souvent plus étendue que vous ne le pensez, en mettant en place des serveurs web, du routage, de la configuration, etc., c'est pourquoi il est essentiel de maintenir la portée du travail réduite. Nous voulons faire ce premier pas positif sur notre toile vierge et l'avoir soutenu par un AT réussi afin de pouvoir continuer à itérer rapidement et en toute sécurité.

![](https://i.imgur.com/t5y5opw.png)

Au fur et à mesure que vous développez, écoutez vos tests, ils devraient vous donner des signaux pour vous aider à orienter votre conception dans une meilleure direction, mais, encore une fois, ancrée dans le comportement plutôt que dans notre imagination.

Généralement, votre première "unité" qui fait le travail difficile pour faire passer l'AT deviendra trop grande pour être confortable, même pour cette petite quantité de comportement. C'est à ce moment que vous pouvez commencer à réfléchir à la façon de décomposer le problème et d'introduire de nouveaux collaborateurs.

![](https://i.imgur.com/UYqd7Cq.png)

C'est là que les doubles de test (par exemple, les fakes, les mocks) sont utiles car la plupart de la complexité qui réside en interne dans le logiciel ne réside généralement pas dans les détails d'implémentation mais "entre" les unités et la façon dont elles interagissent.

#### Les périls de l'approche ascendante

Il s'agit d'une approche "descendante" plutôt qu'"ascendante". L'approche ascendante a ses utilisations, mais elle comporte un élément de risque. En construisant des "services" et du code sans qu'ils soient intégrés rapidement à votre application et sans vérifier un test de haut niveau, **vous risquez de gaspiller beaucoup d'efforts sur des idées non validées**.

C'est une propriété cruciale de l'approche dirigée par les tests d'acceptation, utilisant des tests pour obtenir une validation réelle de notre code.

Trop souvent, j'ai rencontré des ingénieurs qui ont créé un morceau de code, en isolation, de bas en haut, qu'ils pensent résoudre un travail, mais qui :

- Ne fonctionne pas comme nous le voulons
- Fait des choses dont nous n'avons pas besoin
- Ne s'intègre pas facilement
- Nécessite une tonne de réécriture de toute façon

C'est du gaspillage.

## Assez parlé, passons au code

Contrairement aux autres chapitres, vous aurez besoin de [Docker](https://www.docker.com) installé car nous exécuterons nos applications dans des conteneurs. À ce stade du livre, on suppose que vous êtes à l'aise pour écrire du code Go, importer depuis différents packages, etc.

Créez un nouveau projet avec `go mod init github.com/quii/go-specs-greet` (vous pouvez mettre ce que vous voulez ici, mais si vous changez le chemin, vous devrez changer toutes les importations internes pour qu'elles correspondent)

Créez un dossier `specifications` pour contenir nos spécifications, et ajoutez un fichier `greet.go`

```go
package specifications

import (
	"testing"

	"github.com/alecthomas/assert/v2"
)

type Greeter interface {
	Greet() (string, error)
}

func GreetSpecification(t testing.TB, greeter Greeter) {
	got, err := greeter.Greet()
	assert.NoError(t, err)
	assert.Equal(t, got, "Hello, world")
}
```

Mon IDE (Goland) se charge des tracas d'ajout de dépendances pour moi, mais si vous avez besoin de le faire manuellement, vous feriez

`go get github.com/alecthomas/assert/v2`

Étant donné la conception de test d'acceptation de Farley (Spécification->DSL->Pilote->Système), nous avons maintenant une spécification découplée de l'implémentation. Elle ne sait pas et ne se soucie pas de _comment_ nous faisons `Greet` ; elle ne se préoccupe que de la complexité essentielle de notre domaine. Il faut admettre que cette complexité n'est pas très importante pour le moment, mais nous développerons la spécification pour ajouter plus de fonctionnalités au fur et à mesure que nous itérerons. Il est toujours important de commencer petit !

Vous pourriez voir l'interface comme notre première étape d'un DSL ; au fur et à mesure que le projet grandit, vous pourriez trouver la nécessité d'abstraire différemment, mais pour l'instant, c'est bien.

À ce stade, ce niveau de cérémonie pour découpler notre spécification de l'implémentation pourrait amener certaines personnes à nous accuser de "trop abstraire". **Je vous promets que les tests d'acceptation qui sont trop couplés à l'implémentation deviennent un véritable fardeau pour les équipes d'ingénierie**. Je suis convaincu que la plupart des tests d'acceptation dans la nature sont coûteux à maintenir en raison de ce couplage inapproprié, plutôt que l'inverse d'être trop abstraits.

Nous pouvons utiliser cette spécification pour vérifier n'importe quel "système" capable de faire `Greet`.

### Premier système : API HTTP

Nous devons fournir un "service de salutation" via HTTP. Nous devrons donc créer :

1. Un **pilote**. Dans ce cas, un qui fonctionne avec un système HTTP en utilisant un **client HTTP**. Ce code saura comment travailler avec notre API. Les pilotes traduisent les DSL en appels spécifiques au système ; dans notre cas, le pilote implémentera l'interface que les spécifications définissent.
2. Un **serveur HTTP** avec une API de salutation
3. Un **test**, qui est responsable de la gestion du cycle de vie du démarrage du serveur, puis de la connexion du pilote à la spécification pour l'exécuter comme un test

## Écrire le test d'abord

Le processus initial de création d'un test en boîte noire qui compile et exécute votre programme, exécute le test puis nettoie tout peut être assez laborieux. C'est pourquoi il est préférable de le faire au début de votre projet avec une fonctionnalité minimale. Je commence généralement tous mes projets avec une implémentation de serveur "hello world", avec tous mes tests configurés et prêts à ce que je construise rapidement la fonctionnalité réelle.

Le modèle mental des "spécifications", des "pilotes" et des "tests d'acceptation" peut prendre un peu de temps pour s'habituer, alors suivez attentivement. Il peut être utile de "travailler à rebours" en essayant d'abord d'appeler la spécification.

Créez une structure pour héberger le programme que nous avons l'intention de livrer.

`mkdir -p cmd/httpserver`

À l'intérieur du nouveau dossier, créez un nouveau fichier `greeter_server_test.go`, et ajoutez ce qui suit.

```go
package main_test

import (
	"testing"

	"github.com/quii/go-specs-greet/specifications"
)

func TestGreeterServer(t *testing.T) {
	specifications.GreetSpecification(t, nil)
}
```

Nous souhaitons exécuter notre spécification dans un test Go. Nous avons déjà accès à un `*testing.T`, c'est donc le premier argument, mais qu'en est-il du second ?

`specifications.Greeter` est une interface, que nous allons implémenter avec un `Driver` en modifiant le nouveau code TestGreeterServer comme suit :

```go
import (
	go_specs_greet "github.com/quii/go-specs-greet"
)

func TestGreeterServer(t *testing.T) {
	driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080"}
	specifications.GreetSpecification(t, driver)
}
```

Il serait préférable que notre `Driver` soit configurable pour l'exécuter dans différents environnements, y compris localement, nous avons donc ajouté un champ `BaseURL`.

## Essayer d'exécuter le test

```
./greeter_server_test.go:46:12: undefined: go_specs_greet.Driver
```

Nous pratiquons toujours le TDD ici ! C'est un grand premier pas que nous devons faire ; nous devons créer quelques fichiers et écrire peut-être plus de code que ce à quoi nous sommes généralement habitués, mais lorsque vous commencez, c'est souvent le cas. Il est très important d'essayer de se rappeler les règles de l'étape rouge.

> Commettez autant de péchés que nécessaire pour faire passer le test

## Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test qui échoue

Bouchez-vous le nez pour l'instant ; n'oubliez pas, nous pourrons refactoriser une fois le test passé. Voici le code du pilote dans `driver.go` que nous placerons à la racine du projet :

```go
package go_specs_greet

import (
	"io"
	"net/http"
)

type Driver struct {
	BaseURL string
}

func (d Driver) Greet() (string, error) {
	res, err := http.Get(d.BaseURL + "/greet")
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

Remarques :

- On pourrait dire que je devrais écrire des tests pour produire les divers `if err != nil`, mais selon mon expérience, tant que vous ne faites rien avec le `err`, les tests qui disent "vous renvoyez l'erreur que vous obtenez" ont relativement peu de valeur.
- **Vous ne devriez pas utiliser le client HTTP par défaut**. Plus tard, nous passerons un client HTTP pour le configurer avec des délais d'attente, etc., mais pour l'instant, nous essayons simplement d'arriver à un test qui passe.
- Dans notre `greeter_server_test.go`, nous avons appelé la fonction Driver du package `go_specs_greet` que nous avons maintenant créé, n'oubliez pas d'ajouter `github.com/quii/go-specs-greet` à ses importations.

Essayez de relancer les tests ; ils devraient maintenant se compiler mais pas passer.

```
Get "http://localhost:8080/greet": dial tcp [::1]:8080: connect: connection refused
```

Nous avons un `Driver`, mais nous n'avons pas encore démarré notre application, il ne peut donc pas faire de requête HTTP. Nous devons que notre test d'acceptation coordonne la construction, l'exécution et enfin l'arrêt de notre système pour que le test s'exécute.

### Exécuter notre application

Il est courant pour les équipes de construire des images Docker de leurs systèmes à déployer, nous ferons donc de même pour notre test

Pour nous aider à utiliser Docker dans nos tests, nous utiliserons [Testcontainers](https://golang.testcontainers.org). Testcontainers nous donne un moyen programmatique de construire des images Docker et de gérer les cycles de vie des conteneurs.

`go get github.com/testcontainers/testcontainers-go`

Maintenant, vous pouvez modifier `cmd/httpserver/greeter_server_test.go` pour qu'il se lise comme suit :

```go
package main_test

import (
	"context"
	"testing"

	"github.com/alecthomas/assert/v2"
	go_specs_greet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func TestGreeterServer(t *testing.T) {
	ctx := context.Background()

	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:    "../../.",
			Dockerfile: "./cmd/httpserver/Dockerfile",
			// réglez sur false si vous voulez moins de spam, mais c'est utile si vous avez des problèmes
			PrintBuildLog: true,
		},
		ExposedPorts: []string{"8080:8080"},
		WaitingFor:   wait.ForHTTP("/").WithPort("8080"),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})

	driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080"}
	specifications.GreetSpecification(t, driver)
}
```

Essayez d'exécuter le test.

```
=== RUN   TestGreeterHandler
2022/09/10 18:49:44 Starting container id: 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
2022/09/10 18:49:45 Waiting for container id 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
2022/09/10 18:49:45 Container is ready id: 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
    greeter_server_test.go:32: Did not expect an error but got:
        Error response from daemon: Cannot locate specified Dockerfile: ./cmd/httpserver/Dockerfile: failed to create container
--- FAIL: TestGreeterHandler (0.59s)
```

Nous devons créer un Dockerfile pour notre programme. À l'intérieur de notre dossier `httpserver`, créez un `Dockerfile` et ajoutez ce qui suit.

```dockerfile
# Assurez-vous de spécifier la même version de Go que celle dans le fichier go.mod.
# Par exemple, golang:1.22.1-alpine.
FROM golang:1.18-alpine

WORKDIR /app

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/httpserver/*.go

EXPOSE 8080
CMD [ "./svr" ]
```

Ne vous inquiétez pas trop des détails ici ; il peut être raffiné et optimisé, mais pour cet exemple, il suffira. L'avantage de notre approche ici est que nous pourrons plus tard améliorer notre Dockerfile et avoir un test pour prouver qu'il fonctionne comme nous le souhaitons. C'est une vraie force d'avoir des tests en boîte noire !

Essayez de relancer le test ; il devrait se plaindre de ne pas pouvoir construire l'image. Bien sûr, c'est parce que nous n'avons pas encore écrit de programme à construire !

Pour que le test s'exécute complètement, nous devrons créer un programme qui écoute sur `8080`, mais **c'est tout**. Respectez la discipline TDD, n'écrivez pas le code de production qui ferait passer le test avant d'avoir vérifié que le test échoue comme nous l'attendons.

Créez un `main.go` dans notre dossier `httpserver` avec ce qui suit

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(func(writer http.ResponseWriter, request *http.Request) {
	})
	if err := http.ListenAndServe(":8080", handler); err != nil {
		log.Fatal(err)
	}
}
```

Essayez d'exécuter le test à nouveau, et il devrait échouer avec ce qui suit.

```
    greet.go:16: Expected values to be equal:
        +Hello, World
        \ No newline at end of file
--- FAIL: TestGreeterHandler (2.09s)
```

## Écrire suffisamment de code pour le faire passer

Mettez à jour le gestionnaire pour qu'il se comporte comme notre spécification le souhaite

```go
import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
		fmt.Fprint(w, "Hello, world")
	})
	if err := http.ListenAndServe(":8080", handler); err != nil {
		log.Fatal(err)
	}
}
```

## Refactoriser

Bien que techniquement ce ne soit pas une refactorisation, nous ne devrions pas nous fier au client HTTP par défaut. Modifions donc notre pilote pour pouvoir en fournir un, que notre test donnera.

```go
import (
	"io"
	"net/http"
)

type Driver struct {
	BaseURL string
	Client  *http.Client
}

func (d Driver) Greet() (string, error) {
	res, err := d.Client.Get(d.BaseURL + "/greet")
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

Dans notre test dans `cmd/httpserver/greeter_server_test.go`, mettez à jour la création du pilote pour passer un client.

```go
client := http.Client{
	Timeout: 1 * time.Second,
}

driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080", Client: &client}
specifications.GreetSpecification(t, driver)
```

C'est une bonne pratique de garder `main.go` aussi simple que possible ; il ne devrait se préoccuper que d'assembler les blocs de construction que vous transformez en application.

Créez un fichier à la racine du projet appelé `handler.go` et déplacez notre code là-bas.

```go
package go_specs_greet

import (
	"fmt"
	"net/http"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Hello, world")
}
```

Mettez à jour `main.go` pour importer et utiliser le gestionnaire à la place.

```go
package main

import (
	"net/http"

	go_specs_greet "github.com/quii/go-specs-greet"
)

func main() {
	handler := http.HandlerFunc(go_specs_greet.Handler)
	http.ListenAndServe(":8080", handler)
}
```

## Réfléchir

La première étape a semblé être un effort. Nous avons créé plusieurs fichiers `go` pour créer et tester un gestionnaire HTTP qui renvoie une chaîne codée en dur. Cette cérémonie d'"itération 0" et cette configuration nous serviront bien pour les itérations suivantes.

Modifier les fonctionnalités devrait être simple et contrôlé en les pilotant à travers la spécification et en traitant tous les changements qu'elle nous oblige à faire. Maintenant que le `DockerFile` et les `testcontainers` sont configurés pour notre test d'acceptation, nous ne devrions pas avoir à modifier ces fichiers à moins que la façon dont nous construisons notre application ne change.

Nous verrons cela avec notre exigence suivante, saluer une personne particulière.

## Écrire le test d'abord

Modifions notre spécification

```go
package specifications

import (
	"testing"

	"github.com/alecthomas/assert/v2"
)

type Greeter interface {
	Greet(name string) (string, error)
}

func GreetSpecification(t testing.TB, greeter Greeter) {
	got, err := greeter.Greet("Mike")
	assert.NoError(t, err)
	assert.Equal(t, got, "Hello, Mike")
}
```

Pour nous permettre de saluer des personnes spécifiques, nous devons modifier l'interface de notre système pour accepter un paramètre `name`.

## Essayer d'exécuter le test

```
./greeter_server_test.go:48:39: cannot use driver (variable of type go_specs_greet.Driver) as type specifications.Greeter in argument to specifications.GreetSpecification:
	go_specs_greet.Driver does not implement specifications.Greeter (wrong type for Greet method)
		have Greet() (string, error)
		want Greet(name string) (string, error)
```

Le changement dans la spécification signifie que notre pilote doit être mis à jour.

## Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test qui échoue

Mettez à jour le pilote pour qu'il spécifie une valeur de requête `name` dans la demande pour demander qu'un `name` particulier soit salué.

```go
import "io"

func (d Driver) Greet(name string) (string, error) {
	res, err := d.Client.Get(d.BaseURL + "/greet?name=" + name)
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

Le test devrait maintenant s'exécuter et échouer.

```
    greet.go:16: Expected values to be equal:
        -Hello, world
        \ No newline at end of file
        +Hello, Mike
        \ No newline at end of file
---- FAIL: TestGreeterHandler (1.92s)
```

## Écrire suffisamment de code pour le faire passer

Extrayez le `name` de la requête et saluez.

```go
import (
	"fmt"
	"net/http"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, %s", r.URL.Query().Get("name"))
}
```

Le test devrait maintenant passer.

## Refactoriser

Dans [HTTP Handlers Revisited,](https://github.com/quii/learn-go-with-tests/blob/main/http-handlers-revisited.md), nous avons discuté de l'importance pour les gestionnaires HTTP de n'être responsables que de la gestion des préoccupations HTTP ; toute "logique de domaine" devrait vivre en dehors du gestionnaire. Cela nous permet de développer une logique de domaine isolée de HTTP, la rendant plus simple à tester et à comprendre.

Séparons ces préoccupations.

Mettez à jour notre gestionnaire dans `./handler.go` comme suit :

```go
func Handler(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query().Get("name")
	fmt.Fprint(w, Greet(name))
}
```

Créez un nouveau fichier `./greet.go` :
```go
package go_specs_greet

import "fmt"

func Greet(name string) string {
	return fmt.Sprintf("Hello, %s", name)
}
```

## Une légère digression dans le Design Pattern (patron de conception) "adaptateur"

Maintenant que nous avons séparé notre logique de domaine de salutation des personnes dans une fonction séparée, nous sommes maintenant libres d'écrire des tests unitaires pour notre fonction de salutation. C'est sans aucun doute beaucoup plus simple que de la tester à travers une spécification qui passe par un pilote qui frappe un serveur web, pour obtenir une chaîne !

Ne serait-il pas agréable si nous pouvions réutiliser notre spécification ici aussi ? Après tout, le point de la spécification est découplé des détails d'implémentation. Si la spécification capture notre **complexité essentielle** et que notre code de "domaine" est censé la modéliser, nous devrions pouvoir les utiliser ensemble.

Essayons en créant `./greet_test.go` comme suit :

```go
package go_specs_greet_test

import (
	"testing"

	go_specs_greet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreet(t *testing.T) {
	specifications.GreetSpecification(t, go_specs_greet.Greet)
}

```

Ce serait bien, mais ça ne fonctionne pas

```
./greet_test.go:11:39: cannot use go_specs_greet.Greet (value of type func(name string) string) as type specifications.Greeter in argument to specifications.GreetSpecification:
	func(name string) string does not implement specifications.Greeter (missing Greet method)
```

Notre spécification veut quelque chose qui a une méthode `Greet()`, pas une fonction.

L'erreur de compilation est frustrante ; nous avons une chose que nous "savons" être un `Greeter`, mais elle n'est pas tout à fait dans la bonne **forme** pour que le compilateur nous laisse l'utiliser. C'est à cela que répond le patron **adaptateur**.

> En [génie logiciel](https://en.wikipedia.org/wiki/Software_engineering), le **patron adaptateur** est un [patron de conception logicielle](https://en.wikipedia.org/wiki/Software_design_pattern) (également connu sous le nom de [wrapper](https://en.wikipedia.org/wiki/Wrapper_function), un nom alternatif partagé avec le [patron décorateur](https://en.wikipedia.org/wiki/Decorator_pattern)) qui permet à l'[interface](https://en.wikipedia.org/wiki/Interface_(computer_science)) d'une [classe](https://en.wikipedia.org/wiki/Class_(computer_science)) existante d'être utilisée comme une autre interface. Il est souvent utilisé pour faire fonctionner des classes existantes avec d'autres sans modifier leur [code source](https://en.wikipedia.org/wiki/Source_code).

Beaucoup de mots sophistiqués pour quelque chose de relativement simple, ce qui est souvent le cas avec les patrons de conception, c'est pourquoi les gens ont tendance à lever les yeux au ciel. La valeur des patrons de conception n'est pas dans des implémentations spécifiques, mais dans un langage pour décrire des solutions spécifiques aux problèmes courants auxquels les ingénieurs sont confrontés. Si vous avez une équipe qui partage un vocabulaire commun, cela réduit les frictions dans la communication.

Ajoutez ce code dans `./specifications/adapters.go`

```go
type GreetAdapter func(name string) string

func (g GreetAdapter) Greet(name string) (string, error) {
	return g(name), nil
}
```

Nous pouvons maintenant utiliser notre adaptateur dans notre test pour brancher notre fonction `Greet` dans la spécification.

```go
package go_specs_greet_test

import (
	"testing"

	gospecsgreet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreet(t *testing.T) {
	specifications.GreetSpecification(
		t,
		specifications.GreetAdapter(gospecsgreet.Greet),
	)
}
```

Le patron adaptateur est pratique lorsque vous avez un type qui présente le comportement qu'une interface souhaite, mais qui n'est pas dans la bonne forme.

## Réfléchir

Le changement de comportement semblait simple, non ? OK, c'était peut-être simplement dû à la nature du problème, mais cette méthode de travail vous donne de la discipline et une façon simple et répétable de changer votre système de haut en bas :

- Analysez votre problème et identifiez une légère amélioration à votre système qui vous pousse dans la bonne direction
- Capturez la nouvelle complexité essentielle dans une spécification
- Suivez les erreurs de compilation jusqu'à ce que l'AT s'exécute
- Mettez à jour votre implémentation pour que le système se comporte selon la spécification
- Refactorisez

Après la douleur de la première itération, nous n'avons pas eu à modifier notre code de test d'acceptation car nous avons la séparation des spécifications, des pilotes et de l'implémentation. Changer notre spécification nous a obligés à mettre à jour notre pilote et finalement notre implémentation, mais le code standard autour de _comment_ démarrer le système comme un conteneur n'a pas été affecté.

Même avec la surcharge de construction d'une image docker pour notre application et de démarrage du conteneur, la boucle de rétroaction pour tester notre **entire** application est très serrée :

```
quii@Chriss-MacBook-Pro go-specs-greet % go test ./...
ok  	github.com/quii/go-specs-greet	0.181s
ok  	github.com/quii/go-specs-greet/cmd/httpserver	2.221s
?   	github.com/quii/go-specs-greet/specifications	[no test files]
```

Maintenant, imaginez que votre CTO a décidé que gRPC est _l'avenir_. Elle veut que vous exposiez cette même fonctionnalité sur un serveur gRPC tout en maintenant le serveur HTTP existant.

C'est un exemple de **complexité accidentelle**. Rappelez-vous, la complexité accidentelle est la complexité à laquelle nous devons faire face parce que nous travaillons avec des ordinateurs, des choses comme les réseaux, les disques, les API, etc. **La complexité essentielle n'a pas changé**, nous ne devrions donc pas avoir à changer nos spécifications.

De nombreuses structures de dépôt et patrons de conception traitent principalement de la séparation des types de complexité. Par exemple, "ports et adaptateurs" demande que vous sépariez votre code de domaine de tout ce qui a trait à la complexité accidentelle ; ce code vit dans un dossier "adaptateurs".

### Faciliter le changement

Parfois, il est logique de faire une refactorisation _avant_ de faire un changement.

> D'abord, facilitez le changement, puis faites le changement facile

~Kent Beck

Pour cette raison, déplaçons notre code `http` - `driver.go` et `handler.go` - dans un package appelé `httpserver` dans un dossier `adapters` et changeons leurs noms de package en `httpserver`.

Vous devrez maintenant importer le package racine dans `handler.go` pour faire référence à la méthode Greet...

```go
package httpserver

import (
	"fmt"
	"net/http"

	go_specs_greet "github.com/quii/go-specs-greet/domain/interactions"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query().Get("name")
	fmt.Fprint(w, go_specs_greet.Greet(name))
}

```

Importez votre adaptateur httpserver dans main.go :

```go
package main

import (
	"net/http"

	"github.com/quii/go-specs-greet/adapters/httpserver"
)

func main() {
	handler := http.HandlerFunc(httpserver.Handler)
	http.ListenAndServe(":8080", handler)
}
```

et mettez à jour l'importation et la référence à `Driver` dans greeter_server_test.go :

```go
driver := httpserver.Driver{BaseURL: "http://localhost:8080", Client: &client}
```

Enfin, il est utile de rassembler notre code de niveau domaine dans son propre dossier également. Ne soyez pas paresseux et n'ayez pas un dossier `domain` dans vos projets avec des centaines de types et de fonctions sans rapport. Faites un effort pour réfléchir à votre domaine et regrouper les idées qui vont ensemble. Cela rendra votre projet plus facile à comprendre et améliorera la qualité de vos importations.

Plutôt que de voir

```go
domain.Greet
```

Ce qui est juste un peu bizarre, préférez

```go
interactions.Greet
```

Créez un dossier `domain` pour héberger tout votre code de domaine, et à l'intérieur, un dossier `interactions`. Selon vos outils, vous devrez peut-être mettre à jour certaines importations et codes.

L'arborescence de notre projet devrait maintenant ressembler à ceci :

```
quii@Chriss-MacBook-Pro go-specs-greet % tree
.
├── Makefile
├── README.md
├── adapters
│   └── httpserver
│       ├── driver.go
│       └── handler.go
├── cmd
│   └── httpserver
|       ├── Dockerfile
│       ├── greeter_server_test.go
│       └── main.go
├── domain
│   └── interactions
│       ├── greet.go
│       └── greet_test.go
├── go.mod
├── go.sum
└── specifications
    └── adapters.go
    └── greet.go

```

Notre code de domaine, **complexité essentielle**, vit à la racine de notre module go, et le code qui nous permettra de les utiliser dans "le monde réel" est organisé en **adaptateurs**. Le dossier `cmd` est l'endroit où nous pouvons composer ces groupements logiques en applications pratiques, qui ont des tests en boîte noire pour vérifier que tout fonctionne. Sympa !

Enfin, nous pouvons faire un _petit_ nettoyage de notre test d'acceptation. Si vous considérez les étapes de haut niveau de notre test d'acceptation :

- Construire une image docker
- Attendre qu'elle écoute sur _un_ port
- Créer un pilote qui comprend comment traduire le DSL en appels spécifiques au système
- Brancher le pilote dans la spécification

... vous réaliserez que nous avons les mêmes exigences pour un test d'acceptation pour le serveur gRPC !

Le dossier `adapters` semble un bon endroit, donc dans un fichier appelé `docker.go`, encapsulez les deux premières étapes dans une fonction que nous réutiliserons ensuite.

```go
package adapters

import (
	"context"
	"fmt"
	"testing"
	"time"

	"github.com/alecthomas/assert/v2"
	"github.com/docker/go-connections/nat"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func StartDockerServer(
	t testing.TB,
	port string,
	dockerFilePath string,
) {
	ctx := context.Background()
	t.Helper()
	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:       "../../.",
			Dockerfile:    dockerFilePath,
			PrintBuildLog: true,
		},
		ExposedPorts: []string{fmt.Sprintf("%s:%s", port, port)},
		WaitingFor:   wait.ForListeningPort(nat.Port(port)).WithStartupTimeout(5 * time.Second),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})
}
```

Cela nous donne l'occasion de nettoyer un peu notre test d'acceptation

```go
func TestGreeterServer(t *testing.T) {
	var (
		port           = "8080"
		dockerFilePath = "./cmd/httpserver/Dockerfile"
		baseURL        = fmt.Sprintf("http://localhost:%s", port)
		driver         = httpserver.Driver{BaseURL: baseURL, Client: &http.Client{
			Timeout: 1 * time.Second,
		}}
	)

	adapters.StartDockerServer(t, port, dockerFilePath)
	specifications.GreetSpecification(t, driver)
}
```

Cela devrait rendre l'écriture du _prochain_ test plus simple.

## Écrire le test d'abord

Cette nouvelle fonctionnalité peut être accomplie en créant un nouvel `adaptateur` pour interagir avec notre code de domaine. Pour cette raison, nous :

- Ne devrions pas avoir à changer la spécification ;
- Devrions pouvoir réutiliser la spécification ;
- Devrions pouvoir réutiliser le code de domaine.

Créez un nouveau dossier `grpcserver` à l'intérieur de `cmd` pour héberger notre nouveau programme et le test d'acceptation correspondant. À l'intérieur de `cmd/grpc_server/greeter_server_test.go`, ajoutez un test d'acceptation, qui ressemble beaucoup à notre test de serveur HTTP, non par coïncidence mais par conception.

```go
package main_test

import (
	"fmt"
	"testing"

	"github.com/quii/go-specs-greet/adapters"
	"github.com/quii/go-specs-greet/adapters/grpcserver"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreeterServer(t *testing.T) {
	var (
		port           = "50051"
		dockerFilePath = "./cmd/grpcserver/Dockerfile"
		driver         = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	adapters.StartDockerServer(t, port, dockerFilePath)
	specifications.GreetSpecification(t, &driver)
}
```

Les seules différences sont :

- Nous utilisons un fichier docker différent, car nous construisons un programme différent
- Cela signifie que nous aurons besoin d'un nouveau `Driver`, qui utilisera `gRPC` pour interagir avec notre nouveau programme

## Essayer d'exécuter le test

```
./greeter_server_test.go:26:12: undefined: grpcserver
```

Nous n'avons pas encore créé de `Driver`, donc ça ne compilera pas.

## Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test qui échoue

Créez un dossier `grpcserver` à l'intérieur de `adapters` et créez-y `driver.go`

```go
package grpcserver

type Driver struct {
	Addr string
}

func (d Driver) Greet(name string) (string, error) {
	return "", nil
}
```

Si vous l'exécutez à nouveau, il devrait maintenant _compiler_ mais ne pas passer car nous n'avons pas créé de Dockerfile et de programme correspondant à exécuter.

Créez un nouveau `Dockerfile` à l'intérieur de `cmd/grpcserver`.

```dockerfile
# Assurez-vous de spécifier la même version de Go que celle dans le fichier go.mod.
FROM golang:1.18-alpine

WORKDIR /app

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/grpcserver/*.go

EXPOSE 50051
CMD [ "./svr" ]
```

Et un `main.go`

```go
package main

import "fmt"

func main() {
	fmt.Println("implement me")
}
```

Vous devriez maintenant constater que le test échoue car notre serveur n'écoute pas sur le port. Il est temps de commencer à construire notre client et notre serveur avec gRPC.

## Écrire suffisamment de code pour le faire passer

### gRPC

Si vous n'êtes pas familier avec gRPC, je vous suggère de commencer par consulter le [site web de gRPC](https://grpc.io). Néanmoins, pour ce chapitre, c'est juste un autre type d'adaptateur dans notre système, une façon pour d'autres systèmes d'appeler (**r**emote **p**rocedure **c**all) notre excellent code de domaine.

La particularité est que vous définissez une "définition de service" en utilisant Protocol Buffers. Vous générez ensuite du code serveur et client à partir de la définition. Cela fonctionne non seulement pour Go mais aussi pour la plupart des langages courants. Cela signifie que vous pouvez partager une définition avec d'autres équipes de votre entreprise qui n'écrivent peut-être même pas en Go et peuvent toujours faire de la communication service-à-service en douceur.

Si vous n'avez jamais utilisé gRPC auparavant, vous devrez installer un **compilateur de Protocol buffer** et certains **plugins Go**. [Le site web de gRPC a des instructions claires sur la façon de le faire](https://grpc.io/docs/languages/go/quickstart/).

Dans le même dossier que notre nouveau pilote, ajoutez un fichier `greet.proto` avec ce qui suit

```protobuf
syntax = "proto3";

option go_package = "github.com/quii/adapters/grpcserver";

package grpcserver;

service Greeter {
  rpc Greet (GreetRequest) returns (GreetReply) {}
}

message GreetRequest {
  string name = 1;
}

message GreetReply {
  string message = 1;
}
```

Pour comprendre cette définition, vous n'avez pas besoin d'être un expert en Protocol Buffers. Nous définissons un service avec une méthode Greet, puis décrivons les types de messages entrants et sortants.

À l'intérieur de `adapters/grpcserver`, exécutez ce qui suit pour générer le code client et serveur

```
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    greet.proto
```

Si cela a fonctionné, nous aurions du code généré pour nous à utiliser. Commençons par utiliser le code client généré à l'intérieur de notre `Driver`.

```go
package grpcserver

import (
	"context"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type Driver struct {
	Addr string
}

func (d Driver) Greet(name string) (string, error) {
	//todo: we shouldn't redial every time we call greet, refactor out when we're green
	conn, err := grpc.Dial(d.Addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		return "", err
	}
	defer conn.Close()

	client := NewGreeterClient(conn)
	greeting, err := client.Greet(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}
```

Maintenant que nous avons un client, nous devons mettre à jour notre `main.go` pour créer un serveur. Rappelez-vous, à ce stade, nous essayons simplement de faire passer notre test et ne nous soucions pas de la qualité du code.

```go
package main

import (
	"context"
	"log"
	"net"

	"github.com/quii/go-specs-greet/adapters/grpcserver"
	"google.golang.org/grpc"
)

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatal(err)
	}
	s := grpc.NewServer()
	grpcserver.RegisterGreeterServer(s, &GreetServer{})

	if err := s.Serve(lis); err != nil {
		log.Fatal(err)
	}
}

type GreetServer struct {
	grpcserver.UnimplementedGreeterServer
}

func (g GreetServer) Greet(ctx context.Context, request *grpcserver.GreetRequest) (*grpcserver.GreetReply, error) {
	return &grpcserver.GreetReply{Message: "fixme"}, nil
}
```

Pour créer notre serveur gRPC, nous devons implémenter l'interface qu'il a générée pour nous

```go
// GreeterServer is the server API for Greeter service.
// All implementations must embed UnimplementedGreeterServer
// for forward compatibility
type GreeterServer interface {
	Greet(context.Context, *GreetRequest) (*GreetReply, error)
	mustEmbedUnimplementedGreeterServer()
}
```

Notre fonction `main` :

- Écoute sur un port
- Crée un `GreetServer` qui implémente l'interface, puis l'enregistre avec `grpcServer.RegisterGreeterServer`, ainsi qu'un `grpc.Server`.
- Utilise le serveur avec l'écouteur

Ce ne serait pas un effort supplémentaire massif d'appeler notre code de domaine à l'intérieur de `greetServer.Greet` plutôt que de coder en dur `fix-me` dans le message, mais j'aimerais d'abord exécuter notre test d'acceptation pour voir si tout fonctionne au niveau du transport et vérifier la sortie du test qui échoue.

```
greet.go:16: Expected values to be equal:
-fixme
\ No newline at end of file
+Hello, Mike
\ No newline at end of file
```

Parfait ! Nous pouvons voir que notre pilote est capable de se connecter à notre serveur gRPC dans le test.

Maintenant, appelez notre code de domaine à l'intérieur de notre `GreetServer`

```go
type GreetServer struct {
	grpcserver.UnimplementedGreeterServer
}

func (g GreetServer) Greet(ctx context.Context, request *grpcserver.GreetRequest) (*grpcserver.GreetReply, error) {
	return &grpcserver.GreetReply{Message: interactions.Greet(request.Name)}, nil
}
```

Enfin, ça passe ! Nous avons un test d'acceptation qui prouve que notre serveur de salutation gRPC se comporte comme nous le souhaitons.

## Refactoriser

Nous avons commis plusieurs péchés pour faire passer le test, mais maintenant qu'ils passent, nous avons le filet de sécurité pour refactoriser.

### Simplifier main

Comme précédemment, nous ne voulons pas que `main` ait trop de code à l'intérieur. Nous pouvons déplacer notre nouveau `GreetServer` dans `adapters/grpcserver` car c'est là qu'il devrait vivre. En termes de cohésion, si nous changeons la définition du service, nous voulons que le "rayon d'explosion" du changement soit confiné à cette zone de notre code.

### Ne pas recomposer dans notre pilote à chaque fois

Nous n'avons qu'un seul test, mais si nous étendons notre spécification (nous le ferons), il n'est pas logique que le Driver recompose pour chaque appel RPC.

```go
package grpcserver

import (
	"context"
	"sync"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type Driver struct {
	Addr string

	connectionOnce sync.Once
	conn           *grpc.ClientConn
	client         GreeterClient
}

func (d *Driver) Greet(name string) (string, error) {
	client, err := d.getClient()
	if err != nil {
		return "", err
	}

	greeting, err := client.Greet(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}

func (d *Driver) getClient() (GreeterClient, error) {
	var err error
	d.connectionOnce.Do(func() {
		d.conn, err = grpc.Dial(d.Addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
		d.client = NewGreeterClient(d.conn)
	})
	return d.client, err
}
```

Ici, nous montrons comment nous pouvons utiliser [`sync.Once`](https://pkg.go.dev/sync#Once) pour garantir que notre `Driver` ne tente de créer une connexion à notre serveur qu'une seule fois.

Jetons un coup d'œil à l'état actuel de notre structure de projet avant de continuer.

```
quii@Chriss-MacBook-Pro go-specs-greet % tree
.
├── Makefile
├── README.md
├── adapters
│   ├── docker.go
│   ├── grpcserver
│   │   ├── driver.go
│   │   ├── greet.pb.go
│   │   ├── greet.proto
│   │   ├── greet_grpc.pb.go
│   │   └── server.go
│   └── httpserver
│       ├── driver.go
│       └── handler.go
├── cmd
│   ├── grpcserver
│   │   ├── Dockerfile
│   │   ├── greeter_server_test.go
│   │   └── main.go
│   └── httpserver
│       ├── Dockerfile
│       ├── greeter_server_test.go
│       └── main.go
├── domain
│   └── interactions
│       ├── greet.go
│       └── greet_test.go
├── go.mod
├── go.sum
└── specifications
    └── greet.go
```

- Les `adapters` ont des unités cohésives de fonctionnalité regroupées ensemble
- `cmd` contient nos applications et les tests d'acceptation correspondants
- Notre code est totalement découplé de toute complexité accidentelle

### Consolidation des `Dockerfile`

Vous avez probablement remarqué que les deux `Dockerfiles` sont presque identiques au-delà du chemin vers le binaire que nous souhaitons construire.

Les `Dockerfiles` peuvent accepter des arguments pour nous permettre de les réutiliser dans différents contextes, ce qui semble parfait. Nous pouvons supprimer nos 2 Dockerfiles et en avoir un à la racine du projet avec ce qui suit

```dockerfile
# Assurez-vous de spécifier la même version de Go que celle dans le fichier go.mod.
FROM golang:1.18-alpine

WORKDIR /app

ARG bin_to_build

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/${bin_to_build}/main.go

CMD [ "./svr" ]
```

Nous devrons mettre à jour notre fonction `StartDockerServer` pour passer l'argument lors de la construction des images

```go
func StartDockerServer(
	t testing.TB,
	port string,
	binToBuild string,
) {
	ctx := context.Background()
	t.Helper()
	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:    "../../.",
			Dockerfile: "Dockerfile",
			BuildArgs: map[string]*string{
				"bin_to_build": &binToBuild,
			},
			PrintBuildLog: true,
		},
		ExposedPorts: []string{fmt.Sprintf("%s:%s", port, port)},
		WaitingFor:   wait.ForListeningPort(nat.Port(port)).WithStartupTimeout(5 * time.Second),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})
}
```

Et enfin, mettez à jour nos tests pour passer l'image à construire (faites-le aussi pour l'autre test et changez `grpcserver` en `httpserver`).

```go
func TestGreeterServer(t *testing.T) {
	var (
		port   = "50051"
		driver = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	adapters.StartDockerServer(t, port, "grpcserver")
	specifications.GreetSpecification(t, &driver)
}
```

### Séparation des différents types de tests

Les tests d'acceptation sont excellents en ce qu'ils testent le fonctionnement de l'ensemble du système d'un point de vue purement orienté utilisateur et comportemental, mais ils présentent des inconvénients par rapport aux tests unitaires :

- Plus lents
- La qualité du retour d'information n'est souvent pas aussi ciblée qu'un test unitaire
- N'aide pas à la qualité interne ou à la conception

[La Pyramide de Test](https://martinfowler.com/articles/practical-test-pyramid.html) nous guide sur le type de mélange que nous voulons pour notre suite de tests, vous devriez lire l'article de Fowler pour plus de détails, mais le résumé très simplifié pour ce billet est "beaucoup de tests unitaires et quelques tests d'acceptation".

Pour cette raison, à mesure qu'un projet grandit, vous pouvez souvent vous trouver dans des situations où les tests d'acceptation peuvent prendre quelques minutes à s'exécuter. Pour offrir une expérience conviviale aux développeurs qui consultent votre projet, vous pouvez permettre aux développeurs d'exécuter les différents types de tests séparément.

Il est préférable que l'exécution de `go test ./...` soit possible sans configuration supplémentaire de la part d'un ingénieur, à part quelques dépendances clés comme le compilateur Go (évidemment) et peut-être Docker.

Go fournit un mécanisme permettant aux ingénieurs d'exécuter uniquement des tests "courts" avec le [drapeau short](https://pkg.go.dev/testing#Short)

`go test -short ./...`

Nous pouvons ajouter à nos tests d'acceptation pour voir si l'utilisateur souhaite exécuter nos tests d'acceptation en inspectant la valeur du drapeau

```go
if testing.Short() {
	t.Skip()
}
```

J'ai créé un `Makefile` pour montrer cette utilisation

```makefile
build:
	golangci-lint run
	go test ./...

unit-tests:
	go test -short ./...
```

### Quand devrais-je écrire des tests d'acceptation ?

La meilleure pratique consiste à privilégier beaucoup de tests unitaires rapides et quelques tests d'acceptation, mais comment décider quand écrire un test d'acceptation, par rapport aux tests unitaires ?

Il est difficile de donner une règle concrète, mais les questions que je me pose généralement sont les suivantes :

- S'agit-il d'un cas particulier ? Je préférerais les tester unitairement
- Est-ce quelque chose dont les personnes non informatiques parlent beaucoup ? Je préférerais avoir une grande confiance que l'élément clé fonctionne "réellement", donc j'ajouterais un test d'acceptation
- Est-ce que je décris un parcours utilisateur, plutôt qu'une fonction spécifique ? Test d'acceptation
- Les tests unitaires me donneraient-ils assez de confiance ? Parfois, vous prenez un parcours existant qui a déjà un test d'acceptation, mais vous ajoutez d'autres fonctionnalités pour traiter différents scénarios en raison d'entrées différentes. Dans ce cas, l'ajout d'un autre test d'acceptation ajoute un coût mais apporte peu de valeur, je préférerais donc quelques tests unitaires.

## Itérer sur notre travail

Avec tout cet effort, vous espérez que l'extension de notre système sera désormais simple. Créer un système sur lequel il est simple de travailler n'est pas nécessairement facile, mais cela vaut la peine d'y consacrer du temps, et c'est sensiblement plus facile à faire lorsque vous commencez un projet.

Étendons notre API pour inclure une fonctionnalité de "malédiction".

## Écrire le test d'abord

Il s'agit d'un tout nouveau comportement, nous devrions donc commencer par un test d'acceptation. Dans notre fichier de spécification, ajoutez ce qui suit

```go
type MeanGreeter interface {
	Curse(name string) (string, error)
}

func CurseSpecification(t *testing.T, meany MeanGreeter) {
	got, err := meany.Curse("Chris")
	assert.NoError(t, err)
	assert.Equal(t, got, "Go to hell, Chris!")
}
```

Choisissez l'un de nos tests d'acceptation et essayez d'utiliser la spécification

```go
func TestGreeterServer(t *testing.T) {
	if testing.Short() {
		t.Skip()
	}
	var (
		port   = "50051"
		driver = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	t.Cleanup(driver.Close)
	adapters.StartDockerServer(t, port, "grpcserver")
	specifications.GreetSpecification(t, &driver)
	specifications.CurseSpecification(t, &driver)
}
```

## Essayer d'exécuter le test

```
# github.com/quii/go-specs-greet/cmd/grpcserver_test [github.com/quii/go-specs-greet/cmd/grpcserver.test]
./greeter_server_test.go:27:39: cannot use &driver (value of type *grpcserver.Driver) as type specifications.MeanGreeter in argument to specifications.CurseSpecification:
	*grpcserver.Driver does not implement specifications.MeanGreeter (missing Curse method)
```

Notre `Driver` ne prend pas encore en charge `Curse`.

## Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test qui échoue

Rappelez-vous que nous essayons simplement de faire fonctionner le test, alors ajoutez la méthode à `Driver`

```go
func (d *Driver) Curse(name string) (string, error) {
	return "", nil
}
```

Si vous essayez à nouveau, le test devrait compiler, s'exécuter et échouer

```
greet.go:26: Expected values to be equal:
+Go to hell, Chris!
\ No newline at end of file
```

## Écrire suffisamment de code pour le faire passer

Nous devrons mettre à jour notre spécification de protocole buffer pour avoir une méthode `Curse`, puis régénérer notre code.

```protobuf
service Greeter {
  rpc Greet (GreetRequest) returns (GreetReply) {}
  rpc Curse (GreetRequest) returns (GreetReply) {}
}
```

On pourrait dire que la réutilisation des types `GreetRequest` et `GreetReply` est un couplage inapproprié, mais nous pouvons nous en occuper lors de la phase de refactorisation. Comme je le souligne constamment, nous essayons simplement de faire passer le test, afin de vérifier que le logiciel fonctionne, _puis_ nous pouvons le rendre agréable.

Régénérez notre code avec (à l'intérieur de `adapters/grpcserver`).

```
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    greet.proto
```

### Mettre à jour le pilote

Maintenant que le code client a été mis à jour, nous pouvons maintenant appeler `Curse` dans notre `Driver`

```go
func (d *Driver) Curse(name string) (string, error) {
	client, err := d.getClient()
	if err != nil {
		return "", err
	}

	greeting, err := client.Curse(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}
```

### Mettre à jour le serveur

Enfin, nous devons ajouter la méthode `Curse` à notre `Server`

```go
package grpcserver

import (
	"context"
	"fmt"

	"github.com/quii/go-specs-greet/domain/interactions"
)

type GreetServer struct {
	UnimplementedGreeterServer
}

func (g GreetServer) Curse(ctx context.Context, request *GreetRequest) (*GreetReply, error) {
	return &GreetReply{Message: fmt.Sprintf("Go to hell, %s!", request.Name)}, nil
}

func (g GreetServer) Greet(ctx context.Context, request *GreetRequest) (*GreetReply, error) {
	return &GreetReply{Message: interactions.Greet(request.Name)}, nil
}
```

Les tests devraient maintenant passer.

## Refactoriser

Essayez de le faire vous-même.

- Extrayez la logique de domaine `Curse` du serveur grpc, comme nous l'avons fait pour `Greet`. Utilisez la spécification comme un test unitaire contre votre logique de domaine
- Utilisez différents types dans le protobuf pour garantir que les types de message pour `Greet` et `Curse` sont découplés.

## Implémentation de `Curse` pour le serveur HTTP

Encore une fois, un exercice pour vous, le lecteur. Nous avons notre spécification au niveau du domaine et notre logique au niveau du domaine bien séparées. Si vous avez suivi ce chapitre, cela devrait être très simple.

- Ajoutez la spécification au test d'acceptation existant pour le serveur HTTP
- Mettez à jour votre `Driver`
- Ajoutez le nouvel endpoint au serveur et réutilisez le code de domaine pour implémenter la fonctionnalité. Vous pouvez utiliser `http.NewServeMux` pour gérer le routage vers les différents endpoints.

N'oubliez pas de travailler par petites étapes, de valider et d'exécuter vos tests fréquemment. Si vous êtes vraiment bloqué, [vous pouvez trouver mon implémentation sur GitHub](https://github.com/quii/go-specs-greet).

## Améliorer les deux systèmes en mettant à jour la logique de domaine avec un test unitaire

Comme mentionné, tous les changements d'un système ne doivent pas être pilotés via un test d'acceptation. Les permutations de règles métier et de cas particuliers devraient être simples à piloter via un test unitaire si vous avez bien séparé les préoccupations.

Ajoutez un test unitaire à notre fonction `Greet` pour définir par défaut le `name` à `World` s'il est vide. Vous devriez voir à quel point c'est simple, et ensuite les règles métier sont reflétées dans les deux applications "gratuitement".

## Conclusion

Construire des systèmes avec un coût de changement raisonnable vous oblige à avoir des AT conçus pour vous aider, et non à devenir un fardeau de maintenance. Ils peuvent être utilisés comme un moyen de guider, ou comme le dit GOOS, de "faire croître" votre logiciel méthodiquement.

J'espère qu'avec cet exemple, vous pouvez voir le flux de travail prévisible et structuré de notre application pour conduire le changement et comment vous pourriez l'utiliser pour votre travail.

Vous pouvez imaginer parler à une partie prenante qui veut étendre d'une certaine manière le système sur lequel vous travaillez. Capturez-le de manière centrée sur le domaine et indépendante de l'implémentation dans une spécification, et utilisez-le comme une étoile du nord pour vos efforts. Riya et moi décrivons l'utilisation de techniques BDD comme "Example Mapping" [dans notre présentation à GopherconUK](https://www.youtube.com/watch?v=ZMWJCk_0WrY) pour vous aider à comprendre plus profondément la complexité essentielle et vous permettre d'écrire des spécifications plus détaillées et significatives.

Séparer les préoccupations de complexité essentielle et accidentelle rendra votre travail moins ad hoc et plus structuré et délibéré ; cela garantit la résilience de vos tests d'acceptation et les aide à devenir moins un fardeau de maintenance.

Dave Farley donne un excellent conseil :

> Imaginez la personne la moins technique à laquelle vous pouvez penser, qui comprend le domaine du problème, lisant vos tests d'acceptation. Les tests devraient avoir du sens pour cette personne.

Les spécifications devraient alors faire office de documentation. Elles devraient spécifier clairement comment un système doit se comporter. Cette idée est le principe autour d'outils comme [Cucumber](https://cucumber.io), qui vous offre un DSL pour capturer des comportements sous forme de code, puis vous convertissez ce DSL en appels système, tout comme nous l'avons fait ici.

### Ce qui a été couvert

- L'écriture de spécifications abstraites vous permet d'exprimer la complexité essentielle du problème que vous résolvez et d'éliminer la complexité accidentelle. Cela vous permet de réutiliser les spécifications dans différents contextes.
- Comment utiliser [Testcontainers](https://golang.testcontainers.org) pour gérer le cycle de vie de votre système pour les AT. Cela vous permet de tester minutieusement l'image que vous avez l'intention d'expédier sur votre ordinateur, vous donnant un feedback rapide et de la confiance.
- Une brève introduction à la conteneurisation de votre application avec Docker
- gRPC
- Plutôt que de poursuivre des structures de dossiers préétablies, vous pouvez utiliser votre approche de développement pour faire émerger naturellement la structure de votre application, en fonction de vos propres besoins

### Matériel supplémentaire

- Dans cet exemple, notre "DSL" n'est pas vraiment un DSL ; nous avons simplement utilisé des interfaces pour découpler notre spécification du monde réel et nous permettre d'exprimer la logique de domaine de manière claire. À mesure que votre système grandit, ce niveau d'abstraction pourrait devenir maladroit et peu clair. [Renseignez-vous sur le "Screenplay Pattern"](https://cucumber.io/blog/bdd/understanding-screenplay-(part-1)/) si vous voulez trouver plus d'idées sur la façon de structurer vos spécifications.
- Pour souligner, [Growing Object-Oriented Software, Guided by Tests,](http://www.growing-object-oriented-software.com) est un classique. Il démontre l'application de cette approche "style Londres", "de haut en bas" pour écrire des logiciels. Quiconque a apprécié Learn Go with Tests devrait tirer beaucoup de valeur de la lecture de GOOS.
- [Dans le dépôt de code d'exemple](https://github.com/quii/go-specs-greet), il y a plus de code et d'idées dont je n'ai pas parlé ici, comme la construction docker multi-étapes, vous pourriez vouloir vérifier cela.
  - En particulier, *pour le plaisir*, j'ai créé un **troisième programme**, un site web avec des formulaires HTML pour `Greet` et `Curse`. Le `Driver` utilise l'excellent module [https://github.com/go-rod/rod](https://github.com/go-rod/rod), qui lui permet de travailler avec le site web avec un navigateur, tout comme le ferait un utilisateur. En regardant l'historique git, vous pouvez voir comment j'ai commencé par ne pas utiliser d'outils de modélisation "juste pour que ça fonctionne", puis, une fois que j'ai passé mon test d'acceptation, j'avais la liberté de le faire sans craindre de casser les choses.
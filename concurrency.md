# Concurrence

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/concurrency)**

Voici le contexte : un collègue a écrit une fonction, `VerifieLesSites`, qui vérifie l'état d'une liste d'URLs.

```go
package concurrency

type VerificateurSite func(string) bool

func VerifieLesSites(vs VerificateurSite, urls []string) map[string]bool {
	resultats := make(map[string]bool)

	for _, url := range urls {
		resultats[url] = vs(url)
	}

	return resultats
}
```

Il est utilisé pour vérifier l'état d'une liste de sites web. Pour chaque URL, il appellera une fonction `VerificateurSite` qui retournera `true` si le site est bon, et `false` s'il est mauvais.

Il retourne un map des résultats de la vérification pour chaque URL.

Votre collègue veut que vous l'aidiez à rendre la fonction plus rapide. Il est actuellement satisfait de la façon dont elle fonctionne, mais il veut que vous trouviez un moyen de la rendre plus rapide car il a une longue liste d'URLs qu'il veut vérifier régulièrement.

## Écrivez le test d'abord

Voici le test que votre collègue a écrit :

```go
package concurrency

import (
	"reflect"
	"testing"
)

func TestVerifieLesSites(t *testing.T) {
	sites := []string{
		"http://google.com",
		"http://blog.gypsydave5.com",
		"waat://furhurterwe.geds",
	}

	attendu := map[string]bool{
		"http://google.com":          true,
		"http://blog.gypsydave5.com": true,
		"waat://furhurterwe.geds":    false,
	}

	resultat := VerifieLesSites(mockVerificateurSite, sites)

	if !reflect.DeepEqual(attendu, resultat) {
		t.Fatalf("Attendu %v, obtenu %v", attendu, resultat)
	}
}

func mockVerificateurSite(url string) bool {
	if url == "waat://furhurterwe.geds" {
		return false
	}
	return true
}
```

Le test est le suivant :

- Définir une slice d'URLs à vérifier
- Définir un "mock" qui va nous donner des résultats connus pour chaque URL
- Appeler la fonction que notre collègue a écrite
- Vérifier qu'elle retourne un map de nos URLs avec leurs résultats de vérification

Maintenant, nous avons un test qui vérifie que la fonction fait ce qu'elle est censée faire.

Le problème, c'est que ce n'est pas chronométré - nous ne pouvons pas savoir si nos changements accélèrent ou ralentissent la fonction. Avant de la changer, nous devrions réfléchir à la façon de tester qu'elle s'améliore.

## Ajout d'un benchmark

Un [benchmark](https://golang.org/pkg/testing/#hdr-Benchmarks) est une fonction qui teste la performance d'une fonction. Tout comme les fonctions de test, elles se trouvent dans les fichiers `*_test.go` et sont exécutées à l'aide de la commande `go test`. Toutefois, contrairement aux fonctions de test, les fonctions de benchmark ont pour but de montrer la vitesse d'exécution.

```go
package concurrency

import (
	"reflect"
	"testing"
	"time"
)

func mockVerificateurSite(url string) bool {
	if url == "waat://furhurterwe.geds" {
		return false
	}
	return true
}

func TestVerifieLesSites(t *testing.T) {
	sites := []string{
		"http://google.com",
		"http://blog.gypsydave5.com",
		"waat://furhurterwe.geds",
	}

	attendu := map[string]bool{
		"http://google.com":          true,
		"http://blog.gypsydave5.com": true,
		"waat://furhurterwe.geds":    false,
	}

	resultat := VerifieLesSites(mockVerificateurSite, sites)

	if !reflect.DeepEqual(attendu, resultat) {
		t.Fatalf("Attendu %v, obtenu %v", attendu, resultat)
	}
}

func slowVerificateurSite(_ string) bool {
	time.Sleep(20 * time.Millisecond)
	return true
}

func BenchmarkVerifieLesSites(b *testing.B) {
	urls := make([]string, 100)
	for i := 0; i < len(urls); i++ {
		urls[i] = "une url"
	}

	for i := 0; i < b.N; i++ {
		VerifieLesSites(slowVerificateurSite, urls)
	}
}
```

Le benchmark est un peu plus compliqué qu'un test :

- un benchmark a accès à un paramètre `b *testing.B` qui fournit une méthode `Loop` pour la boucle benchmark
- le paramètre `b` contient également un champ `N` qui est utilisé pour déterminer combien de fois la boucle s'exécute
- nous avons également créé une liste d'URLs de test artificiellement longue et une version lente de notre `VerificateurSite` pour simuler une vraie utilisation de la fonction
- le `time.Sleep` dans `slowVerificateurSite` signifie que cette fonction prendra 20 millisecondes à retourner

Lorsque nous exécutons un benchmark, il est exécuté plusieurs fois par le test runner. La valeur de `b.N` augmentera à chaque exécution jusqu'à ce que le test runner soit satisfait de la stabilité du temps de référence.

Essayez-le ! Exécutez `go test -bench=.` sur votre terminal. Cela exécutera les tests, puis les benchmarks :

```text
pkg: github.com/gypsydave5/learn-go-with-tests/concurrency/v0
BenchmarkVerifieLesSites-4               1        2571051988 ns/op
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v0        2.573s
```

Fantastique ! Mais qu'est-ce que tous ces chiffres signifient ?

- `-4` est le nombre de processeurs utilisés par le test runner
- `1` est le nombre de fois que le test runner a estimé qu'il fallait exécuter le benchmark pour obtenir un bon échantillon
- `2571051988 ns/op` est le temps moyen d'exécution de `VerifieLesSites` en nanosecondes.

Ok, donc il nous faut actuellement environ 2,5 secondes pour vérifier 100 sites. Votre collègue pense que vous pouvez y arriver plus rapidement et que vous devriez utiliser une "goroutine". Vous avez du travail à faire.

## Écrivez assez de code pour le faire passer

Nous savons que la fonction fonctionne, mais nous voulons qu'elle s'exécute plus rapidement. Nous allons utiliser les [goroutines](https://golang.org/doc/effective_go.html#goroutines), qui sont une façon de faire fonctionner le code en Go de manière concurrente.

La concurrence n'est pas le parallélisme. Nous _pourrions_ accomplir ce travail de manière parallèle - vérifier chaque URL en même temps qu'une autre - mais ce n'est pas nécessairement ce que nous voulons faire.

Ce que nous voulons faire, c'est déplacer la vérification des sites web sur des goroutines séparées et puis rassembler les résultats. Cela nous permettra d'avoir une concurrence - les goroutines s'exécuteront lorsque le processus attendra que les requêtes HTTP reviennent.

Modifions notre fonction `VerifieLesSites` :

```go
func VerifieLesSites(vs VerificateurSite, urls []string) map[string]bool {
	resultats := make(map[string]bool)

	for _, url := range urls {
		go func() {
			resultats[url] = vs(url)
		}()
	}

	time.Sleep(2 * time.Second)

	return resultats
}
```

Voyons ce qui se passe :

- `go` est une expression Go qui démarre une goroutine, qui est une fonction qui s'exécute simultanément avec d'autres fonctions. Elle démarre une nouvelle goroutine à chaque itération de la boucle for.
- `func()` est une fonction anonyme, aussi connue sous le nom de fermeture (closure). Cette fermeture n'accepte aucun paramètre.
- Tout ce que nous voulions faire à l'origine à l'intérieur de la boucle, nous le faisons maintenant à l'intérieur de la goroutine.
- Nous avons également ajouté un `time.Sleep(2 * time.Second)` à la fin, juste pour donner aux goroutines le temps de terminer leur travail avant que la fonction ne renvoie les `resultats`.

Mais maintenant, lorsque nous exécutons les tests...

```text
--- FAIL: TestVerifieLesSites (2.00s)
    CheckWebsites_test.go:31: Attendu map[http://google.com:true http://blog.gypsydave5.com:true waat://furhurterwe.geds:false], obtenu map[]
```

Oh non ! Le test échoue - et de manière assez spectaculaire - pas de sites web ont été vérifiés. Que s'est-il passé ?

### Race conditions

L'erreur vient de la fermeture que nous avons créée au sein de notre boucle for. Lorsque nous itérons sur la boucle, `url` change à chaque itération. Mais chaque goroutine a une référence à la même variable `url`, dont la valeur finale est `waat://furhurterwe.geds`. Ainsi, chaque goroutine vérifie ce site web, après avoir été lancée à différents moments, et essaie de mettre son résultat dans le map.

Mais cela n'explique pas vraiment pourquoi nous n'obtenons pas de résultats du tout.

Le problème est dans la façon dont nous utilisons le map `resultats`. Écrire dans un map n'est pas une opération sûre pour les goroutines en Go. En lançant plusieurs goroutines, nous provoquons une _race condition_ (*condition de course* en français) où plusieurs processus tentent d'écrire dans le même endroit en même temps.

Exécutons à nouveau les tests, mais cette fois avec le détecteur de race de Go activé : `go test -race`.

```text
==================
WARNING: DATA RACE
    Read at 0x00c000096820 by main goroutine:
    ...

    Previous write at 0x00c000096820 by goroutine 8:
    ...
==================
```

Ce n'est pas grave et ce n'est pas fatal - si ça l'était, vous verriez parfois des choses étranges dans les résultats, comme certains des sites web qui manquent ou donnent les mauvais résultats. Les *race conditions* sont difficiles à reproduire car leur comportement est par nature non déterministe.

### Channels

Nous pouvons corriger notre code en utilisant les _channels_. Les channels sont une structure de données Go qui permet d'envoyer et de recevoir des valeurs entre les goroutines, en garantissant que l'envoi et la réception sont des opérations synchronisées. Cela signifie que nous pouvons utiliser un channel pour coordonner le travail de nos goroutines.

Voici notre fonction corrigée :

```go
func VerifieLesSites(vs VerificateurSite, urls []string) map[string]bool {
	resultats := make(map[string]bool)
	channel := make(chan string)

	for _, url := range urls {
		go func(u string) {
			channel <- u
		}(url)
	}

	for i := 0; i < len(urls); i++ {
		url := <-channel
		resultats[url] = vs(url)
	}

	return resultats
}
```

Mais maintenant nous obtenons un comportement étrange et nos tests ne passent toujours pas.

```text
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
...
```

Nous avons un problème parce que nous utilisons notre channel de la mauvaise façon. Lorsque nous envoyons quelque chose sur un channel (avec `channel <- someValue`), ce processus _attend_ qu'un autre processus reçoive de ce channel. Dans notre code, nous essayons d'envoyer chaque URL sur le channel avant de commencer à recevoir quoi que ce soit du channel.

Essayons de corriger cela :

```go
func VerifieLesSites(vs VerificateurSite, urls []string) map[string]bool {
	resultats := make(map[string]bool)
	channel := make(chan string)

	for _, url := range urls {
		go func(u string) {
			resultats[u] = vs(u)
			channel <- u
		}(url)
	}

	for i := 0; i < len(urls); i++ {
		<-channel
	}

	return resultats
}
```

Maintenant, chaque goroutine vérifie une URL, stocke le résultat dans le map, puis envoie l'URL sur le channel pour indiquer qu'elle a terminé. Dans la deuxième boucle, nous recevons simplement depuis le channel une fois pour chaque URL, en ignorant le message envoyé, ce qui permet aux goroutines de s'achever.

Mais il y a encore un problème - nous avons toujours une race condition avec les goroutines qui écrivent dans le map. Et nous ne voulons vraiment pas être affectés par les races.

Nous devons modifier notre approche pour éviter ce problème. Plutôt que d'écrire directement dans le map depuis les goroutines, nous allons utiliser le channel pour envoyer des "résultats" contenant l'URL et le résultat de la vérification. Puis, dans la fonction principale, nous recevrons ces résultats et les stockerons dans le map.

Voici notre version corrigée et sans race condition :

```go
type resultat struct {
	string
	bool
}

func VerifieLesSites(vs VerificateurSite, urls []string) map[string]bool {
	resultats := make(map[string]bool)
	channel := make(chan resultat)

	for _, url := range urls {
		go func(u string) {
			channel <- resultat{u, vs(u)}
		}(url)
	}

	for i := 0; i < len(urls); i++ {
		resultat := <-channel
		resultats[resultat.string] = resultat.bool
	}

	return resultats
}
```

Maintenant nous avons :
- Créé un type `resultat` qui contient le résultat d'un appel à `VerificateurSite`.
- Créé un channel de type `resultat`.
- Envoyé le résultat sur le channel dans chaque goroutine.
- Reçu chaque résultat depuis le channel et stocké le résultat dans le map.

Lançons les tests et regardons ce qui se passe.

```text
go test
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v1        0.015s
```

Fantastique ! Maintenant, lançons le benchmark :

```text
go test -bench=.
testing: warning: no tests to run
BenchmarkVerifieLesSites-8               1        2568052759 ns/op
PASS
```

Hmm, c'est à peu près le même que notre test de référence, non ? Voyons pourquoi.

Notre benchmark est globalement correct : il crée une grande liste d'URL, puis une implémentation lente d'un `VerificateurSite` qui prend environ 20 millisecondes à se terminer, puis l'exécute à travers notre fonction.

Le problème, c'est qu'on fait une évaluation naïve de ce que notre fonction est censée faire. Oui, elle vérifie chaque URL, mais elle ne fait rien avec le résultat. La raison pour laquelle notre fonction lente est lente est qu'elle a un `time.Sleep(20 * time.Millisecond)` à l'intérieur. Dans un environnement de test, on ne se soucie pas vraiment du résultat, on veut juste savoir que le code fait ce qu'il est censé faire.

Il ne s'agit pas vraiment de savoir combien de temps prend notre fonction `VerificateurSite` dans des conditions de test, mais de savoir si notre fonction `VerifieLesSites` est plus rapide lorsqu'elle utilise la concurrence pour des entrées connues. Modifions notre benchmark pour simuler une fonction `VerificateurSite` lente.

```go
func slowVerificateurSite(_ string) bool {
	time.Sleep(20 * time.Millisecond)
	return true
}

func BenchmarkVerifieLesSites(b *testing.B) {
	urls := make([]string, 100)
	for i := 0; i < len(urls); i++ {
		urls[i] = "une url"
	}

	for i := 0; i < b.N; i++ {
		VerifieLesSites(slowVerificateurSite, urls)
	}
}
```

Maintenant, exécutons à nouveau le benchmark :

```text
pkg: github.com/gypsydave5/learn-go-with-tests/concurrency/v1
BenchmarkVerifieLesSites-8             100          20890591 ns/op
PASS
```

C'est beaucoup mieux ! Notre fonction met maintenant environ 0.02 seconde à vérifier 100 sites.

Exécutons le benchmark sans notre implémentation concurrente. Revenons à l'implémentation originale :

```go
func VerifieLesSites(vs VerificateurSite, urls []string) map[string]bool {
	resultats := make(map[string]bool)

	for _, url := range urls {
		resultats[url] = vs(url)
	}

	return resultats
}
```

```text
pkg: github.com/gypsydave5/learn-go-with-tests/concurrency/v0
BenchmarkVerifieLesSites-8               1        2410465949 ns/op
PASS
```

Environ 2,4 secondes, soit 115 fois plus lent! C'est probablement assez rapide pour que votre collègue soit satisfait.

## Rétrospective

### `goroutines`

Créer une nouvelle routine pour qu'une fonction s'exécute simultanément à d'autres fonctions est incroyablement facile. Nous créons une goroutine en ajoutant simplement le mot-clé `go` devant une fonction :

```go
go doSomething()
```

Mais attendez - notre fonction anonyme a un paramètre : comment avons-nous pu faire cela ? Lorsque nous avons essayé de faire référence à la variable de boucle directement, nous avons eu une race condition.

Nous avons utilisé une _fermeture_ - un concept venant de la programmation fonctionnelle. Une fermeture est une fonction qui utilise des variables de son environnement extérieur. En Go, une fermeture a accès aux variables de la fonction à laquelle elle est attachée.

```go
go func(u string) {
	canal <- resultat{u, vs(u)}
}(url)
```

Nous créons une fermeture qui prend un paramètre `u`, et nous transmettons la variable `url` à cette fermeture au moment où nous la créons : `}(url)`.

### `channels`

Les channels sont des tubes qui connectent des goroutines en concurrence, et fournissent un mécanisme pour que les goroutines communiquent entre elles. Dans notre fonction, nous avons utilisé un channel pour collecter les résultats de chaque goroutine.

Nous avons créé un channel avec la fonction `make` :

```go
canal := make(chan resultat)
```

Ensuite, nous avons écrit dans le channel dans chaque goroutine avec l'opérateur d'envoi, `<-` :

```go
canal <- resultat{u, vs(u)}
```

Et enfin, nous avons lu à partir du channel dans notre fonction principale, encore une fois avec l'opérateur `<-` :

```go
resultat := <-canal
```

Les channels sont un concept important en Go. Ils nous permettent de coordonner des goroutines en concurrence. Si vous êtes intéressé à en savoir plus, il y a un excellent article sur le [blog Go](https://blog.golang.org/pipelines).

### Synchroniser les données avec des maps

Nous avons rencontré un problème lors de la mise à jour d'un map en concurrence. Ce problème est appelé une race condition, et il se produit lorsque plusieurs goroutines tentent d'accéder à la même partie de la mémoire en même temps.

Nous avons résolu ce problème en nous assurant que nous ne mettions à jour le map que dans notre fonction principale, et en utilisant un channel pour collecter les résultats de chaque goroutine.

Une autre approche possible serait d'utiliser une structure de données synchronisée, comme un map synchronisé. Cependant, cette approche est généralement plus lente que l'utilisation de canaux, et elle est plus difficile à comprendre.

### Effectuer un benchmark sur le code

Nous avons utilisé le package `testing` de Go pour effectuer un benchmark sur notre code. Les fonctions de benchmark sont similaires aux fonctions de test, mais elles sont utilisées pour mesurer la performance du code plutôt que sa correction.

Pour effectuer un benchmark sur une fonction, nous créons une fonction `BenchmarkXxx` dans un fichier de test, où `Xxx` est le nom de la fonction que nous souhaitons tester. Cette fonction doit prendre un paramètre `b *testing.B`.

Ensuite, nous exécutons le benchmark avec la commande `go test -bench=.`. L'indicateur `-bench` indique au test runner d'exécuter les benchmarks, et `.` est une expression régulière qui correspond à tous les benchmarks du package.

### Résumé

Ce chapitre a couvert :

- L'utilisation de goroutines pour exécuter du code concurremment
- L'utilisation de fermetures (closures) pour encapsuler des goroutines
- L'utilisation de canaux pour communiquer entre goroutines
- Éviter les *race conditions* lors de l'accès aux données partagées
- Effectuer des benchmarks sur le code pour mesurer les améliorations de performance
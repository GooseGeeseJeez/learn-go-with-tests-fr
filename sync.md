# Sync

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/sync)**

Nous voulons créer un compteur qui peut être utilisé en toute sécurité de manière concurrente.

Nous commencerons par un compteur non sécurisé et vérifierons que son comportement fonctionne dans un environnement mono-thread.

Ensuite, nous testerons son manque de sécurité avec plusieurs goroutines essayant d'utiliser le compteur via un test, puis nous le corrigerons.

## Écrivez le test d'abord

Nous voulons que notre API nous fournisse une méthode pour incrémenter le compteur, puis récupérer sa valeur.

```go
func TestCompteur(t *testing.T) {
	t.Run("incrémenter le compteur 3 fois le laisse à 3", func(t *testing.T) {
		compteur := Compteur{}
		compteur.Inc()
		compteur.Inc()
		compteur.Inc()

		if compteur.Valeur() != 3 {
			t.Errorf("attendu 3, obtenu %d", compteur.Valeur())
		}
	})
}
```

## Essayez d'exécuter le test

```
./sync_test.go:9:11: undefined: Counter
```

## Écrivez la quantité minimale de code pour que le test s'exécute et vérifiez la sortie du test qui échoue

Définissons notre type `Compteur`.

```go
type Compteur struct {
	valeur int
}

func (c *Compteur) Inc() {
	c.valeur++
}

func (c *Compteur) Valeur() int {
	return c.valeur
}
```

Le test devrait maintenant échouer avec une erreur claire.

```
=== RUN   TestCompteur
--- FAIL: TestCompteur (0.00s)
=== RUN   TestCompteur/incrémenter_le_compteur_3_fois_le_laisse_à_3
    --- FAIL: TestCompteur/incrémenter_le_compteur_3_fois_le_laisse_à_3 (0.00s)
        sync_test.go:13: attendu 3, obtenu 0
```

Attendez, nous n'avons toujours pas d'erreur. Nous incrémentons la valeur, nous l'avons même vérifiée... mais alors pourquoi obtenons-nous 0?

Nous devons faire attention lorsque nous utilisons des pointeurs vers les structures.
Lorsque nous avons écrit `compteur := Compteur{}`, nous avons créé une instance de la struct `Compteur` avec une valeur de 0 pour le champ `valeur`.

Cependant, lorsque nous appelons `compteur.Inc()`, nous passons la _valeur_ de `compteur` à la méthode. Cela signifie que la méthode reçoit une _copie_ de `compteur` (car les arguments de fonction en Go sont passés par valeur). La méthode modifie alors la copie, pas l'original.

Nous devons prendre le pointeur de `compteur` et l'utiliser :

```go
func TestCompteur(t *testing.T) {
	t.Run("incrémenter le compteur 3 fois le laisse à 3", func(t *testing.T) {
		compteur := &Compteur{}
		compteur.Inc()
		compteur.Inc()
		compteur.Inc()

		if compteur.Valeur() != 3 {
			t.Errorf("attendu 3, obtenu %d", compteur.Valeur())
		}
	})
}
```

## Écrivez assez de code pour le faire passer

Les tests passent maintenant. Mais nous voulions aussi tester si notre compteur est sûr du point de vue concurrentiel, lorsque plusieurs goroutines tentent de l'incrémenter en même temps.

Si nous avons 3 goroutines qui incrémentent un compteur commençant à 0, nous nous attendrions à ce que la valeur finale soit de 3. Mais ce n'est pas toujours le cas dans un code non sécurisé pour la concurrence.

## Écrivez le test d'abord

```go
t.Run("fonctionne en concurrence en toute sécurité", func(t *testing.T) {
	compteurAttendu := 1000
	compteur := &Compteur{}

	var wg sync.WaitGroup
	wg.Add(compteurAttendu)

	for i := 0; i < compteurAttendu; i++ {
		go func() {
			compteur.Inc()
			wg.Done()
		}()
	}
	wg.Wait()

	if compteur.Valeur() != compteurAttendu {
		t.Errorf("attendu %d, obtenu %d", compteurAttendu, compteur.Valeur())
	}
})
```

Ce test lance 1000 goroutines, chacune incrémentant le compteur une fois.

Nous utilisons `sync.WaitGroup` pour nous aider à attendre que toutes les goroutines aient fini. Considérez que c'est comme un compteur :

- Vous appelez `Add` avec le nombre de goroutines à attendre.
- Chaque goroutine appelle `Done()` lorsqu'elle a terminé son travail.
- Pendant ce temps, vous pouvez appeler `Wait()` qui bloquera jusqu'à ce que toutes les goroutines signalent qu'elles ont terminé (c'est-à-dire que le compteur interne du WaitGroup atteint 0).

## Essayez d'exécuter le test

```
=== RUN   TestCompteur/fonctionne_en_concurrence_en_toute_sécurité
--- FAIL: TestCompteur/fonctionne_en_concurrence_en_toute_sécurité (0.00s)
    sync_test.go:26: attendu 1000, obtenu 940
```

Le test échoue. Il s'avère que notre compteur n'est pas sûr pour une utilisation concurrente car le résultat n'est pas fiable. Nous nous attendions à 1000 incrémentations, mais nous n'en avons obtenu que 940.

Si vous exécutez le test plusieurs fois, vous obtiendrez probablement des résultats différents. 

Le problème ici est que nous avons plusieurs goroutines essayant d'accéder à la même mémoire en même temps sans aucune coordination. C'est ce qu'on appelle une "condition de course".

## Écrivez assez de code pour le faire passer

Nous devons protéger l'accès à la valeur partagée en utilisant un mutex. Go fournit un mutex dans le package `sync`.

```go
type Compteur struct {
	mu    sync.Mutex
	valeur int
}

func (c *Compteur) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.valeur++
}
```

Ce que nous faisons ici c'est que nous verrouillons le mutex avant de modifier la valeur, puis nous le déverrouillons après. `defer` s'assure que le mutex est déverrouillé même si la fonction panique.

L'utilisation d'un mutex signifie qu'une seule goroutine peut accéder au bloc de code verrouillé à la fois. Toutes les autres goroutines doivent attendre que le mutex soit déverrouillé avant de pouvoir procéder.

Maintenant, notre test devrait passer de manière fiable. Si vous l'exécutez plusieurs fois, vous obtiendrez toujours 1000 incrémentations.

## Refactoriser

Notre code est déjà assez propre. Cependant, il y a une chose importante à noter : notre méthode `Valeur()` n'est pas protégée par un mutex. Si une goroutine appelle `Valeur()` pendant qu'une autre appelle `Inc()`, nous pourrions toujours avoir une condition de course.

C'est parce que `Valeur()` lit la valeur, et `Inc()` écrit la valeur. Si ces opérations se produisent en même temps, nous pourrions lire une valeur incohérente.

Corrigeons cela :

```go
func (c *Compteur) Valeur() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.valeur
}
```

## Problèmes liés à l'embarquement

Une autre approche serait d'utiliser l'embarquement de structs en Go pour inclure `sync.Mutex` directement dans notre `Compteur`.

```go
type Compteur struct {
	sync.Mutex
	valeur int
}

func (c *Compteur) Inc() {
	c.Lock()
	defer c.Unlock()
	c.valeur++
}

func (c *Compteur) Valeur() int {
	c.Lock()
	defer c.Unlock()
	return c.valeur
}
```

Maintenant, nous n'avons pas besoin de définir notre propre champ `mu`. Nous pouvons simplement appeler les méthodes `Lock` et `Unlock` directement sur l'instance `Compteur`.

Bien que cela puisse sembler plus propre, il y a un problème potentiel avec cette approche. En embarquant `sync.Mutex`, nous exposons ses méthodes (`Lock`, `Unlock`) comme méthodes de notre type `Compteur`. Cela signifie que le code qui utilise notre `Compteur` pourrait appeler `compteur.Lock()` ou `compteur.Unlock()` directement, ce qui pourrait briser notre logique de synchronisation.

Pour cette raison, il est généralement préférable de garder le mutex comme un champ non exporté, comme dans notre première solution.

## Conclusion

Nous avons appris comment rendre un compteur sûr pour une utilisation concurrente en utilisant un mutex pour protéger l'accès à la valeur partagée.

Le package `sync` fournit plusieurs primitives de synchronisation, dont :

- `Mutex` : pour protéger l'accès à une ressource partagée.
- `RWMutex` : similaire à `Mutex`, mais permet plusieurs lectures simultanées tant qu'aucune écriture n'est en cours.
- `WaitGroup` : pour attendre que plusieurs goroutines terminent.
- `Once` : pour s'assurer qu'une fonction n'est exécutée qu'une seule fois.
- `Cond` : pour attendre ou annoncer l'occurrence d'un événement.

Lorsque vous travaillez avec des goroutines et des données partagées, il est important de s'assurer que l'accès à ces données est correctement synchronisé pour éviter les conditions de course.

Rappelez-vous le dicton en Go : "Ne communiquez pas en partageant la mémoire ; partagez la mémoire en communiquant". Bien que nous ayons utilisé un mutex pour partager la mémoire en toute sécurité dans cet exemple, dans de nombreux cas, il est préférable d'utiliser des canaux pour coordonner les goroutines.
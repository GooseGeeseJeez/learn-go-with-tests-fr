# Readers sensibles au `context`e

**[Vous pouvez trouver tout le code ici](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/context-aware-reader)**

Ce chapitre démontre comment développer un `io.Reader` sensible au contexte par la méthode TDD, tel que décrit par Mat Ryer et David Hernandez dans [The Pace Dev Blog](https://pace.dev/blog/2020/02/03/context-aware-ioreader-for-golang-by-mat-ryer).

## Un lecteur sensible au contexte ?

Tout d'abord, un rappel rapide sur `io.Reader`.

Si vous avez lu d'autres chapitres de ce livre, vous avez déjà rencontré `io.Reader` lors de l'ouverture de fichiers, de l'encodage JSON et de diverses autres tâches courantes. C'est une abstraction simple pour lire des données à partir de _quelque chose_

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

En utilisant `io.Reader`, vous pouvez bénéficier d'une grande réutilisation de code de la bibliothèque standard, c'est une abstraction très couramment utilisée (avec son homologue `io.Writer`)

### Sensible au contexte ?

[Dans un chapitre précédent](context.md), nous avons discuté de la façon dont nous pouvons utiliser `context` pour permettre l'annulation. C'est particulièrement utile si vous effectuez des tâches qui peuvent être coûteuses en termes de calcul et que vous souhaitez pouvoir les arrêter.

Lorsque vous utilisez un `io.Reader`, vous n'avez aucune garantie sur la vitesse, cela pourrait prendre 1 nanoseconde ou des centaines d'heures. Il pourrait être utile de pouvoir annuler ce genre de tâches dans votre propre application, et c'est ce que Mat et David ont essayé d'accomplir.

Ils ont combiné deux abstractions simples (`context.Context` et `io.Reader`) pour résoudre ce problème.

Essayons de développer par TDD une fonctionnalité qui nous permette d'envelopper un `io.Reader` pour qu'il puisse être annulé.

Tester cela pose un défi intéressant. Normalement, lorsque vous utilisez un `io.Reader`, vous le fournissez généralement à une autre fonction et vous ne vous préoccupez pas des détails, comme avec `json.NewDecoder` ou `io.ReadAll`.

Ce que nous voulons démontrer, c'est quelque chose comme

> Étant donné un `io.Reader` avec "ABCDEF", lorsque j'envoie un signal d'annulation à mi-parcours, si j'essaie de continuer à lire, je n'obtiens rien d'autre, donc tout ce que j'obtiens est "ABC"

Examinons à nouveau l'interface.

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

La méthode `Read` du `Reader` lira le contenu qu'il possède dans un `[]byte` que nous fournissons.

Donc plutôt que de tout lire, nous pourrions :

 - Fournir un tableau d'octets de taille fixe qui ne contient pas tout le contenu
 - Envoyer un signal d'annulation
 - Essayer de lire à nouveau et cela devrait renvoyer une erreur avec 0 octet lu

Pour l'instant, écrivons simplement un test de "bon déroulement" où il n'y a pas d'annulation, juste pour nous familiariser avec le problème sans avoir à écrire de code de production pour le moment.

```go
func TestContextAwareReader(t *testing.T) {
	t.Run("lets just see how a normal reader works", func(t *testing.T) {
		rdr := strings.NewReader("123456")
		got := make([]byte, 3)
		_, err := rdr.Read(got)

		if err != nil {
			t.Fatal(err)
		}

		assertBufferHas(t, got, "123")

		_, err = rdr.Read(got)

		if err != nil {
			t.Fatal(err)
		}

		assertBufferHas(t, got, "456")
	})
}

func assertBufferHas(t testing.TB, buf []byte, want string) {
	t.Helper()
	got := string(buf)
	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

- Créer un `io.Reader` à partir d'une chaîne avec des données
- Un tableau d'octets dans lequel lire, plus petit que le contenu du lecteur
- Appeler read, vérifier le contenu, répéter.

À partir de là, nous pouvons imaginer envoyer une sorte de signal d'annulation avant la deuxième lecture pour changer le comportement.

Maintenant que nous avons vu comment cela fonctionne, nous allons développer le reste de la fonctionnalité en TDD.

## Écrire le test d'abord

Nous voulons pouvoir composer un `io.Reader` avec un `context.Context`.

Avec le TDD, il est préférable de commencer par imaginer l'API souhaitée et d'écrire un test pour celle-ci.

À partir de là, le compilateur et la sortie du test en échec peuvent nous guider vers une solution

```go
t.Run("behaves like a normal reader", func(t *testing.T) {
	rdr := NewCancellableReader(strings.NewReader("123456"))
	got := make([]byte, 3)
	_, err := rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "123")

	_, err = rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "456")
})
```

## Essayer d'exécuter le test

```
./cancel_readers_test.go:12:10: undefined: NewCancellableReader
```

## Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test en échec

Nous devrons définir cette fonction et elle devrait renvoyer un `io.Reader`

```go
func NewCancellableReader(rdr io.Reader) io.Reader {
	return nil
}
```

Si vous essayez de l'exécuter

```
=== RUN   TestCancelReaders
=== RUN   TestCancelReaders/behaves_like_a_normal_reader
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
	panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x10f8fb5]
```

Comme prévu

## Écrire suffisamment de code pour le faire passer

Pour l'instant, nous allons simplement renvoyer le `io.Reader` que nous avons passé

```go
func NewCancellableReader(rdr io.Reader) io.Reader {
	return rdr
}
```

Le test devrait maintenant passer.

Je sais, je sais, cela semble stupide et pédant, mais avant de se lancer dans le travail sophistiqué, il est important que nous ayons _une certaine_ vérification que nous n'avons pas cassé le comportement "normal" d'un `io.Reader`, et ce test nous donnera confiance à mesure que nous avancerons.

## Écrire le test d'abord

Ensuite, nous devons essayer d'annuler.

```go
t.Run("stops reading when cancelled", func(t *testing.T) {
	ctx, cancel := context.WithCancel(context.Background())
	rdr := NewCancellableReader(ctx, strings.NewReader("123456"))
	got := make([]byte, 3)
	_, err := rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "123")

	cancel()

	n, err := rdr.Read(got)

	if err == nil {
		t.Error("expected an error after cancellation but didn't get one")
	}

	if n > 0 {
		t.Errorf("expected 0 bytes to be read after cancellation but %d were read", n)
	}
})
```

Nous pouvons plus ou moins copier le premier test mais maintenant nous :
- Créons un `context.Context` avec annulation pour pouvoir `cancel` après la première lecture
- Pour que notre code fonctionne, nous devrons passer `ctx` à notre fonction
- Nous affirmons ensuite qu'après `cancel`, rien n'a été lu

## Essayer d'exécuter le test

```
./cancel_readers_test.go:33:30: too many arguments in call to NewCancellableReader
	have (context.Context, *strings.Reader)
	want (io.Reader)
```

## Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test en échec

Le compilateur nous dit quoi faire ; mettre à jour notre signature pour accepter un contexte

```go
func NewCancellableReader(ctx context.Context, rdr io.Reader) io.Reader {
	return rdr
}
```

(Vous devrez également mettre à jour le premier test pour passer `context.Background` aussi)

Vous devriez maintenant voir une sortie de test en échec très claire

```
=== RUN   TestCancelReaders
=== RUN   TestCancelReaders/stops_reading_when_cancelled
--- FAIL: TestCancelReaders (0.00s)
    --- FAIL: TestCancelReaders/stops_reading_when_cancelled (0.00s)
        cancel_readers_test.go:48: expected an error but didn't get one
        cancel_readers_test.go:52: expected 0 bytes to be read after cancellation but 3 were read
```

## Écrire suffisamment de code pour le faire passer

À ce stade, c'est du copier-coller de l'article original de Mat et David, mais nous allons quand même procéder lentement et de manière itérative.

Nous savons que nous avons besoin d'un type qui encapsule le `io.Reader` à partir duquel nous lisons et le `context.Context`, alors créons-le et essayons de le renvoyer depuis notre fonction au lieu du `io.Reader` original

```go
func NewCancellableReader(ctx context.Context, rdr io.Reader) io.Reader {
	return &readerCtx{
		ctx:      ctx,
		delegate: rdr,
	}
}

type readerCtx struct {
	ctx      context.Context
	delegate io.Reader
}
```

Comme je l'ai souligné à plusieurs reprises dans ce livre, allez lentement et laissez le compilateur vous aider

```
./cancel_readers_test.go:60:3: cannot use &readerCtx literal (type *readerCtx) as type io.Reader in return argument:
	*readerCtx does not implement io.Reader (missing Read method)
```

L'abstraction semble correcte, mais elle n'implémente pas l'interface dont nous avons besoin (`io.Reader`), alors ajoutons la méthode.

```go
func (r *readerCtx) Read(p []byte) (n int, err error) {
	panic("implement me")
}
```

Exécutez les tests et ils devraient _compiler_ mais paniquer. C'est quand même un progrès.

Faisons passer le premier test en _déléguant_ simplement l'appel à notre `io.Reader` sous-jacent

```go
func (r readerCtx) Read(p []byte) (n int, err error) {
	return r.delegate.Read(p)
}
```

À ce stade, notre test de bon déroulement passe à nouveau et nous avons l'impression que nos éléments sont bien abstraits

Pour que notre deuxième test passe, nous devons vérifier le `context.Context` pour voir s'il a été annulé.

```go
func (r readerCtx) Read(p []byte) (n int, err error) {
	if err := r.ctx.Err(); err != nil {
		return 0, err
	}
	return r.delegate.Read(p)
}
```

Tous les tests devraient maintenant passer. Vous remarquerez que nous renvoyons l'erreur du `context.Context`. Cela permet aux appelants du code d'examiner les différentes raisons pour lesquelles l'annulation s'est produite, ce qui est traité plus en détail dans l'article original.

## Conclusion

- Les petites interfaces sont bonnes et sont facilement composées
- Lorsque vous essayez d'augmenter une chose (par exemple `io.Reader`) avec une autre, vous voulez généralement utiliser le [modèle de délégation](https://en.wikipedia.org/wiki/Delegation_pattern)

> En génie logiciel, le modèle de délégation est un modèle de conception orienté objet qui permet à la composition d'objets d'obtenir la même réutilisation de code que l'héritage.

- Une façon simple de commencer ce type de travail est d'envelopper votre délégué et d'écrire un test qui affirme qu'il se comporte comme le délégué normalement avant de commencer à composer d'autres parties pour changer le comportement. Cela vous aidera à maintenir le bon fonctionnement des choses pendant que vous codez vers votre objectif
# Types d'erreurs

**[Vous pouvez trouver tout le code ici](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/error-types)**

**Créer vos propres types pour les erreurs peut être une façon élégante de rendre votre code plus propre, plus facile à utiliser et à tester.**

Pedro sur le Slack Gopher demande

> Si je crée une erreur comme `fmt.Errorf("%s must be foo, got %s", bar, baz)`, existe-t-il un moyen de tester l'égalité sans comparer la valeur de la chaîne ?

Créons une fonction pour explorer cette idée.

```go
// DumbGetter va récupérer le corps en chaîne de caractères de l'url s'il obtient un 200
func DumbGetter(url string) (string, error) {
	res, err := http.Get(url)

	if err != nil {
		return "", fmt.Errorf("problème lors de la récupération depuis %s, %v", url, err)
	}

	if res.StatusCode != http.StatusOK {
		return "", fmt.Errorf("n'a pas obtenu 200 de %s, mais %d", url, res.StatusCode)
	}

	defer res.Body.Close()
	body, _ := io.ReadAll(res.Body) // ignoring err for brevity

	return string(body), nil
}
```

Il n'est pas rare d'écrire une fonction qui pourrait échouer pour différentes raisons, et nous voulons nous assurer de gérer correctement chaque scénario.

Comme Pedro le dit, nous _pourrions_ écrire un test pour l'erreur de statut comme ceci.

```go
t.Run("lorsque vous n'obtenez pas un 200, vous obtenez une erreur de statut", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("une erreur était attendue")
	}

	want := fmt.Sprintf("n'a pas obtenu 200 de %s, mais %d", svr.URL, http.StatusTeapot)
	got := err.Error()

	if got != want {
		t.Errorf(`got "%v", want "%v"`, got, want)
	}
})
```

Ce test crée un serveur qui renvoie toujours `StatusTeapot` et nous utilisons ensuite son URL comme argument pour `DumbGetter` afin de vérifier qu'il gère correctement les réponses autres que `200`.

## Problèmes avec cette façon de tester

Ce livre essaie d'insister sur _l'écoute de vos tests_ et ce test ne _semble_ pas bon :

- Nous construisons la même chaîne de caractères que le code de production pour la tester
- C'est ennuyeux à lire et à écrire
- Est-ce que le message d'erreur exact est ce qui nous _préoccupe vraiment_ ?

Qu'est-ce que cela nous dit ? L'ergonomie de notre test se refléterait sur un autre morceau de code essayant d'utiliser notre code.

Comment un utilisateur de notre code réagit-il aux types spécifiques d'erreurs que nous renvoyons ? Le mieux qu'il puisse faire est d'examiner la chaîne d'erreur, ce qui est extrêmement sujet aux erreurs et horrible à écrire.

## Ce que nous devrions faire

Avec le TDD, nous avons l'avantage d'adopter l'état d'esprit suivant :

> Comment _voudrais-je_ utiliser ce code ?

Ce que nous pourrions faire pour `DumbGetter`, c'est fournir un moyen aux utilisateurs d'utiliser le système de types pour comprendre quel genre d'erreur s'est produite.

Et si `DumbGetter` pouvait nous renvoyer quelque chose comme

```go
type BadStatusError struct {
	URL    string
	Status int
}
```

Plutôt qu'une chaîne magique, nous avons de véritables _données_ avec lesquelles travailler.

Modifions notre test existant pour refléter ce besoin

```go
t.Run("lorsque vous n'obtenez pas un 200, vous obtenez une erreur de statut", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("une erreur était attendue")
	}

	got, isStatusErr := err.(BadStatusError)

	if !isStatusErr {
		t.Fatalf("ce n'était pas une BadStatusError, mais %T", err)
	}

	want := BadStatusError{URL: svr.URL, Status: http.StatusTeapot}

	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

Nous devrons faire en sorte que `BadStatusError` implémente l'interface error.

```go
func (b BadStatusError) Error() string {
	return fmt.Sprintf("n'a pas obtenu 200 de %s, mais %d", b.URL, b.Status)
}
```

### Que fait le test ?

Au lieu de vérifier la chaîne exacte de l'erreur, nous faisons une [assertion de type](https://tour.golang.org/methods/15) sur l'erreur pour voir si c'est une `BadStatusError`. Cela reflète plus clairement notre désir concernant le _type_ d'erreur. En supposant que l'assertion réussisse, nous pouvons alors vérifier que les propriétés de l'erreur sont correctes.

Lorsque nous exécutons le test, il nous indique que nous n'avons pas renvoyé le bon type d'erreur

```
--- FAIL: TestDumbGetter (0.00s)
    --- FAIL: TestDumbGetter/when_you_dont_get_a_200_you_get_a_status_error (0.00s)
    	error-types_test.go:56: was not a BadStatusError, got *errors.errorString
```

Corrigeons `DumbGetter` en mettant à jour notre code de gestion d'erreurs pour utiliser notre type

```go
if res.StatusCode != http.StatusOK {
	return "", BadStatusError{URL: url, Status: res.StatusCode}
}
```

Ce changement a eu des _effets positifs réels_

- Notre fonction `DumbGetter` est devenue plus simple, elle n'est plus concernée par les subtilités d'une chaîne d'erreur, elle crée simplement une `BadStatusError`.
- Nos tests reflètent maintenant (et documentent) ce qu'un utilisateur de notre code _pourrait_ faire s'il décidait de vouloir faire une gestion d'erreurs plus sophistiquée que simplement journaliser. Il suffit de faire une assertion de type et vous avez facilement accès aux propriétés de l'erreur.
- C'est toujours "juste" une `error`, donc s'ils le choisissent, ils peuvent la transmettre dans la pile d'appels ou la journaliser comme toute autre `error`.

## Conclusion

Si vous vous retrouvez à tester plusieurs conditions d'erreur, ne tombez pas dans le piège de comparer les messages d'erreur.

Cela conduit à des tests instables et difficiles à lire/écrire, et cela reflète les difficultés que les utilisateurs de votre code auront s'ils doivent également commencer à faire les choses différemment selon le type d'erreurs qui se sont produites.

Assurez-vous toujours que vos tests reflètent la façon dont _vous_ aimeriez utiliser votre code. À cet égard, envisagez de créer des types d'erreur pour encapsuler vos types d'erreurs. Cela facilite la gestion de différents types d'erreurs pour les utilisateurs de votre code et rend également l'écriture de votre code de gestion d'erreurs plus simple et plus facile à lire.

## Addendum

À partir de Go 1.13, il existe de nouvelles façons de travailler avec les erreurs dans la bibliothèque standard, ce qui est couvert dans le [Blog Go](https://blog.golang.org/go1.13-errors)

```go
t.Run("lorsque vous n'obtenez pas un 200, vous obtenez une erreur de statut", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("une erreur était attendue")
	}

	var got BadStatusError
	isBadStatusError := errors.As(err, &got)
	want := BadStatusError{URL: svr.URL, Status: http.StatusTeapot}

	if !isBadStatusError {
		t.Fatalf("ce n'était pas une BadStatusError, mais %T", err)
	}

	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

Dans ce cas, nous utilisons [`errors.As`](https://pkg.go.dev/errors#example-As) pour essayer d'extraire notre erreur dans notre type personnalisé. Il renvoie un `bool` pour indiquer le succès et l'extrait dans `got` pour nous.
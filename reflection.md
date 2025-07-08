# Réflexion

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/reflection)**

[De Twitter](https://twitter.com/peterbourgon/status/1011403901419937792?s=09)

> défi golang : écrivez une fonction `walk(x interface{}, fn func(string))` qui prend une struct `x` et appelle `fn` pour tous les champs de type string trouvés à l'intérieur. niveau de difficulté : récursivement.

Pour faire cela, nous aurons besoin d'utiliser la _réflexion_.

> La réflexion en informatique est la capacité d'un programme à examiner sa propre structure, particulièrement à travers les types ; c'est une forme de métaprogrammation. C'est aussi une grande source de confusion.

De [The Go Blog: Reflection](https://blog.golang.org/laws-of-reflection)

## Qu'est-ce que `interface{}`?

Nous avons apprécié la sécurité de typage que Go nous a offerte en termes de fonctions qui travaillent avec des types connus, comme `string`, `int` et nos propres types comme `CompteBancaire`.

Cela signifie que nous obtenons une documentation gratuite et que le compilateur se plaindra si vous essayez de passer le mauvais type à une fonction.

Un `interface{}` peut être utilisé pour représenter n'importe quel type, ce qui est très utile pour des fonctions qui doivent être capable de manipuler des types inconnus au moment de la compilation, comme notre fonction `walk`.

### Problèmes avec `interface{}`

Cette flexibilité a un coût.

Si vous avez déjà une interface comme :

```go
type Stringer interface {
    String() string
}
```

Et une fonction qui prend un `Stringer`, le compilateur refusera de compiler le code si vous essayez de passer un type qui n'implémente pas l'interface.

```go
type PersonStruct struct {
    Name string
}

func PrintString(s fmt.Stringer) {
    fmt.Println(s.String())
}

func main() {
    p := PersonStruct{Name: "Alice"}
    PrintString(p)
}
```

Produira un message d'erreur comme celui-ci à la compilation :

```
cannot use p (type PersonStruct) as type fmt.Stringer in argument to PrintString:
    PersonStruct does not implement fmt.Stringer (missing String method)
```

C'est très utile, mais avec `interface{}` le compilateur vous laissera passer n'importe quoi parce que tout peut être représenté comme un `interface{}`.

C'est une des raisons pour lesquelles vous devez utiliser `interface{}` avec prudence.

Mais si vous essayez d'interagir avec ces valeurs `interface{}` comme si c'était un type en particulier, comme si c'était un `string` par exemple, vous obtiendrez une erreur d'exécution :

```go
func main() {
    var x interface{} = "hello"
    var y string = x.(string) // OK
    var z int = x.(int) // Panic: interface conversion: interface {} is string, not int
}
```

Donc pour utiliser `interface{}` vous devrez soit :
- Savoir de quel type est la valeur et faire une assertion de type comme l'exemple ci-dessus, ou
- Utiliser la réflexion pour examiner dynamiquement la valeur et son type.

## Écrivez le test d'abord

Nous avons besoin d'écrire une fonction avec la signature suivante :

```go
func walk(x interface{}, fn func(input string))
```

`walk` prend `x` de type `interface{}` (c'est-à-dire n'importe quel type) et une fonction `fn` qui prend un `string` en entrée.

Notre fonction doit parcourir tous les champs d'une struct et appeler `fn` avec la valeur de tout champ qui est un `string`.

Le nom de la fonction `walk` est un indice. Dans le domaine informatique, le "tree walking" (parcours d'arbre) est un processus qui consiste à visiter toutes les valeurs dans une structure de données, comme un arbre, en explorant méthodiquement les nœuds de cet arbre.

Dans ce cas, nous voulons "parcourir" une structure arbitraire, explorer tous ses champs et appeler `fn` pour les champs de type `string`.

Commençons par un cas simple, une fonction qui extrait un seul champ `string` d'une struct.

```go
func TestMarche(t *testing.T) {

	cases := []struct {
		Nom          string
		Entree       interface{}
		AppelsPrevus []string
	}{
		{
			"Struct avec un champ string",
			struct {
				Nom string
			}{"Chris"},
			[]string{"Chris"},
		},
	}

	for _, test := range cases {
		t.Run(test.Nom, func(t *testing.T) {
			var resultat []string

			marche(test.Entree, func(entree string) {
				resultat = append(resultat, entree)
			})

			if !reflect.DeepEqual(resultat, test.AppelsPrevus) {
				t.Errorf("resultat %v, attendu %v", resultat, test.AppelsPrevus)
			}
		})
	}
}
```

Si ce test paraît déroutant, revoyez le chapitre sur les [tableaux et slices](arrays-and-slices.md) pour vous rappeler comment fonctionnent les tests utilisant des tables de tests.

En résumé, nous créons une suite de cas de tests où nous pouvons décrire l'entrée pour notre fonction `marche` et la sortie attendue. 

Notre cas de test est une struct anonyme avec un seul champ. Nous passerons cela à `marche` ainsi qu'une fonction qui ajoutera la valeur reçue dans une slice `resultat` que nous pouvons vérifier ensuite.

## Essayez d'exécuter le test

```
./reflection_test.go:21:2: undefined: marche
```

## Écrivez le code minimal pour faire passer le test

```go
func marche(x interface{}, fn func(entrée string)) {
	val := reflect.ValueOf(x)
	champ := val.Field(0)
	fn(champ.String())
}
```

Le code de cette fonction peut sembler un peu mystérieux, mais voici comment il fonctionne :

1. Nous utilisons `reflect.ValueOf` pour obtenir une `Value` qui représente la valeur contenue dans l'interface `x`. Cela nous donne un point d'entrée pour utiliser l'API de réflexion de Go.
2. Nous supposons que `x` est une struct avec au moins un champ, et nous accédons au premier champ avec `val.Field(0)`.
3. Nous supposons que ce champ est un string (ou du moins qu'il peut être converti en string), et nous appelons la méthode `String()` sur la `Value` pour obtenir une représentation string de ce champ.
4. Enfin, nous appelons la fonction `fn` avec cette string.

Évidemment, ces suppositions sont très limitées, mais elles suffisent pour faire passer notre premier test.

## Refactoriser

Notre solution est clairement insuffisante et très limitée, mais elle fait passer notre premier test. Nous allons l'étendre au fur et à mesure que nous ajouterons plus de cas de test.

Mettons à jour notre test pour tester une struct avec deux champs de type string.

```go
{
	"Struct avec deux champs string",
	struct {
		Nom      string
		Ville    string
	}{"Chris", "Londres"},
	[]string{"Chris", "Londres"},
}
```

## Essayez d'exécuter le test

```
=== RUN   TestMarche/Struct_avec_deux_champs_string
    --- FAIL: TestMarche/Struct_avec_deux_champs_string (0.00s)
        reflection_test.go:40: resultat [Chris], attendu [Chris Londres]
```

Le test échoue car notre code ne prend en compte que le premier champ. Modifions notre fonction pour traiter tous les champs.

```go
func marche(x interface{}, fn func(entrée string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		champ := val.Field(i)
		fn(champ.String())
	}
}
```

Maintenant, notre code va :
1. Obtenir le nombre total de champs dans la struct avec `val.NumField()`.
2. Itérer sur tous les champs de 0 à `NumField()-1`.
3. Pour chaque champ, obtenir sa valeur et appeler `fn` avec la représentation string de cette valeur.

Avec cette modification, notre test pour une struct avec deux champs string devrait passer.

Mais que se passe-t-il si nous avons une struct avec des champs qui ne sont pas des strings ?

Ajoutons un cas de test pour une struct avec des champs de différents types.

```go
{
	"Struct avec champs string et int",
	struct {
		Nom        string
		Age        int
		Ville      string
	}{"Chris", 33, "Londres"},
	[]string{"Chris", "Londres"},
},
```

## Essayez d'exécuter le test

```
=== RUN   TestMarche/Struct_avec_champs_string_et_int
    --- FAIL: TestMarche/Struct_avec_champs_string_et_int (0.00s)
        reflection_test.go:41: resultat [Chris 33 Londres], attendu [Chris Londres]
```

Notre test échoue car notre fonction actuelle essaie de traiter tous les champs comme des strings, y compris le champ `Age` qui est un entier. 

Nous devons modifier notre code pour n'appeler `fn` que sur les champs de type string.

```go
func marche(x interface{}, fn func(entrée string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		champ := val.Field(i)

		if champ.Kind() == reflect.String {
			fn(champ.String())
		}
	}
}
```

`champ.Kind()` nous donne le type du champ, et nous pouvons le comparer à `reflect.String` pour vérifier si c'est un string.

Maintenant, notre test pour une struct avec des champs de différents types devrait passer.

Mais que se passe-t-il si nous avons une struct imbriquée ?

```go
{
	"Structs imbriquées",
	Personne{
		"Chris",
		Profil{33, "Londres"},
	},
	[]string{"Chris", "Londres"},
},

type Personne struct {
	Nom    string
	Profil Profil
}

type Profil struct {
	Age   int
	Ville string
}
```

## Essayez d'exécuter le test

```
=== RUN   TestMarche/Structs_imbriquées
    --- FAIL: TestMarche/Structs_imbriquées (0.00s)
        reflection_test.go:54: resultat [Chris], attendu [Chris Londres]
```

Notre fonction actuelle ne sait pas comment traiter les structs imbriquées. Nous devons la modifier pour qu'elle puisse parcourir récursivement les champs de type struct.

```go
func marche(x interface{}, fn func(entrée string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		champ := val.Field(i)

		switch champ.Kind() {
		case reflect.String:
			fn(champ.String())
		case reflect.Struct:
			marche(champ.Interface(), fn)
		}
	}
}
```

Nous avons remplacé notre instruction `if` par un `switch` pour pouvoir gérer différents types de champs.

Si le champ est une struct, nous appelons récursivement `marche` sur ce champ, en convertissant d'abord la `Value` en `interface{}` avec `champ.Interface()`.

Cela devrait faire passer notre test pour les structs imbriquées.

Mais que se passe-t-il si notre struct est un pointeur ?

```go
{
	"Pointeurs vers des trucs",
	&Personne{
		"Chris",
		Profil{33, "Londres"},
	},
	[]string{"Chris", "Londres"},
},
```

## Essayez d'exécuter le test

```
=== RUN   TestMarche/Pointeurs_vers_des_trucs
panic: reflect: call of reflect.Value.NumField on ptr Value [recovered]
	panic: reflect: call of reflect.Value.NumField on ptr Value
```

Notre test panique parce que nous essayons d'appeler `NumField()` sur un pointeur, ce qui n'est pas valide. Nous devons d'abord déréférencer le pointeur.

```go
func marche(x interface{}, fn func(entrée string)) {
	val := reflect.ValueOf(x)

	if val.Kind() == reflect.Ptr {
		val = val.Elem()
	}

	for i := 0; i < val.NumField(); i++ {
		champ := val.Field(i)

		switch champ.Kind() {
		case reflect.String:
			fn(champ.String())
		case reflect.Struct:
			marche(champ.Interface(), fn)
		case reflect.Ptr:
			marche(champ.Interface(), fn)
		}
	}
}
```

Nous avons ajouté deux modifications :
1. Si `x` est un pointeur, nous utilisons `val.Elem()` pour obtenir la valeur pointée.
2. Si un champ est un pointeur, nous appelons récursivement `marche` sur ce champ.

Cela devrait faire passer notre test pour les pointeurs.

Mais que se passe-t-il si nous avons des slices ou des tableaux ?

```go
{
	"Slices",
	[]Profil{
		{33, "Londres"},
		{34, "Paris"},
	},
	[]string{"Londres", "Paris"},
},
```

## Essayez d'exécuter le test

```
=== RUN   TestMarche/Slices
panic: reflect: call of reflect.Value.NumField on slice Value [recovered]
	panic: reflect: call of reflect.Value.NumField on slice Value
```

Notre fonction actuelle ne sait pas comment traiter les slices. Nous devons la modifier pour parcourir les éléments d'une slice.

```go
func marche(x interface{}, fn func(entrée string)) {
	val := reflect.ValueOf(x)

	if val.Kind() == reflect.Ptr {
		val = val.Elem()
	}

	switch val.Kind() {
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			marche(val.Field(i).Interface(), fn)
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			marche(val.Index(i).Interface(), fn)
		}
	case reflect.String:
		fn(val.String())
	}
}
```

Nous avons restructuré notre fonction pour qu'elle traite différents types de valeurs :
1. Si c'est une struct, nous parcourons ses champs.
2. Si c'est une slice ou un tableau, nous parcourons ses éléments.
3. Si c'est une string, nous appelons `fn` sur cette string.

Avec ces modifications, notre test pour les slices devrait passer.

Et les maps ?

```go
{
	"Maps",
	map[string]string{
		"Foo": "Bar",
		"Baz": "Boz",
	},
	[]string{"Bar", "Boz"},
},
```

## Essayez d'exécuter le test

```
=== RUN   TestMarche/Maps
panic: reflect: call of reflect.Value.NumField on map Value [recovered]
	panic: reflect: call of reflect.Value.NumField on map Value
```

Nous devons ajouter le support pour les maps.

```go
func marche(x interface{}, fn func(entrée string)) {
	val := reflect.ValueOf(x)

	if val.Kind() == reflect.Ptr {
		val = val.Elem()
	}

	switch val.Kind() {
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			marche(val.Field(i).Interface(), fn)
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			marche(val.Index(i).Interface(), fn)
		}
	case reflect.Map:
		for _, clé := range val.MapKeys() {
			marche(val.MapIndex(clé).Interface(), fn)
		}
	case reflect.String:
		fn(val.String())
	}
}
```

Si c'est une map, nous parcourons ses valeurs (ignorant les clés).

Avec ces modifications, notre test pour les maps devrait passer.

Et que se passe-t-il si nous avons un canal ?

```go
{
	"Canaux",
	func() interface{} {
		canal := make(chan Profil)
		go func() {
			canal <- Profil{33, "Berlin"}
			canal <- Profil{34, "Katowice"}
			close(canal)
		}()
		return canal
	}(),
	[]string{"Berlin", "Katowice"},
},
```

## Essayez d'exécuter le test

```
=== RUN   TestMarche/Canaux
panic: reflect: call of reflect.Value.NumField on chan Value [recovered]
	panic: reflect: call of reflect.Value.NumField on chan Value
```

Nous devons ajouter le support pour les canaux.

```go
func marche(x interface{}, fn func(entrée string)) {
	val := reflect.ValueOf(x)

	if val.Kind() == reflect.Ptr {
		val = val.Elem()
	}

	switch val.Kind() {
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			marche(val.Field(i).Interface(), fn)
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			marche(val.Index(i).Interface(), fn)
		}
	case reflect.Map:
		for _, clé := range val.MapKeys() {
			marche(val.MapIndex(clé).Interface(), fn)
		}
	case reflect.Chan:
		for v, ok := val.Recv(); ok; v, ok = val.Recv() {
			marche(v.Interface(), fn)
		}
	case reflect.String:
		fn(val.String())
	}
}
```

Si c'est un canal, nous lisons les valeurs du canal jusqu'à ce qu'il soit fermé, et nous appelons récursivement `marche` sur chaque valeur.

Avec ces modifications, notre test pour les canaux devrait passer.

Et que se passe-t-il avec les fonctions qui retournent des valeurs ?

```go
{
	"Fonctions",
	func() string { return "hello world" },
	[]string{"hello world"},
},
```

## Essayez d'exécuter le test

```
=== RUN   TestMarche/Fonctions
panic: reflect: call of reflect.Value.NumField on func Value [recovered]
	panic: reflect: call of reflect.Value.NumField on func Value
```

Nous devons ajouter le support pour les fonctions.

```go
func marche(x interface{}, fn func(entrée string)) {
	val := reflect.ValueOf(x)

	if val.Kind() == reflect.Ptr {
		val = val.Elem()
	}

	switch val.Kind() {
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			marche(val.Field(i).Interface(), fn)
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			marche(val.Index(i).Interface(), fn)
		}
	case reflect.Map:
		for _, clé := range val.MapKeys() {
			marche(val.MapIndex(clé).Interface(), fn)
		}
	case reflect.Chan:
		for v, ok := val.Recv(); ok; v, ok = val.Recv() {
			marche(v.Interface(), fn)
		}
	case reflect.Func:
		valFnResult := val.Call(nil)
		for _, res := range valFnResult {
			marche(res.Interface(), fn)
		}
	case reflect.String:
		fn(val.String())
	}
}
```

Si c'est une fonction, nous l'appelons avec `val.Call(nil)` (sans arguments), et nous parcourons les résultats.

Avec ces modifications, notre test pour les fonctions devrait passer.

## Conclusion

La réflexion en Go est un outil puissant, mais qui doit être utilisé avec prudence. Elle est utile lorsque vous devez travailler avec des types inconnus à la compilation, mais elle rend votre code plus difficile à comprendre et à maintenir.

Les lois de la réflexion en Go, selon le blog officiel, sont :

1. La réflexion va de l'interface à la réflexion.
2. La réflexion va de la réflexion à l'interface.
3. Pour modifier une valeur de réflexion, la valeur doit être modifiable.

En utilisant la réflexion, nous avons créé une fonction générique `marche` qui peut parcourir n'importe quelle structure de données et appeler une fonction sur toutes les valeurs string qu'elle contient.

Cette fonction pourrait être utile dans des scénarios comme :
- Validation de données
- Sérialisation
- Logging
- Tout autre cas où vous devez traiter des structures de données dynamiques.

Mais rappelez-vous que la réflexion est lente et difficile à déboguer, donc utilisez-la uniquement lorsque c'est nécessaire.
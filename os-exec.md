# OS Exec

**[Vous pouvez trouver tout le code ici](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/os-exec)**

[keith6014](https://www.reddit.com/user/keith6014) demande sur [reddit](https://www.reddit.com/r/golang/comments/aaz8ji/testdata_and_function_setup_help/)

> J'exécute une commande en utilisant os/exec.Command() qui génère des données XML. La commande sera exécutée dans une fonction appelée GetData().

> Pour tester GetData(), j'ai créé des données de test.

> Dans mon fichier _test.go, j'ai un TestGetData qui appelle GetData() mais cela utilisera os.exec, alors que j'aimerais plutôt utiliser mes données de test.

> Quelle est une bonne façon d'y parvenir ? En appelant GetData, devrais-je avoir un mode "test" pour qu'il lise un fichier, par exemple GetData(mode string) ?

Quelques observations :

- Quand quelque chose est difficile à tester, c'est souvent parce que la séparation des préoccupations n'est pas tout à fait correcte
- N'ajoutez pas de "modes de test" dans votre code, utilisez plutôt [l'Injection de Dépendances](./dependency-injection.md) afin de pouvoir modéliser vos dépendances et séparer les préoccupations.

J'ai pris la liberté de deviner à quoi pourrait ressembler le code

```go
type Payload struct {
	Message string `xml:"message"`
}

func GetData() string {
	cmd := exec.Command("cat", "msg.xml")

	out, _ := cmd.StdoutPipe()
	var payload Payload
	decoder := xml.NewDecoder(out)

	// ces 3 fonctions peuvent renvoyer des erreurs mais je les ignore par souci de concision
	cmd.Start()
	decoder.Decode(&payload)
	cmd.Wait()

	return strings.ToUpper(payload.Message)
}
```

- Il utilise `exec.Command` qui vous permet d'exécuter une commande externe au processus
- Nous capturons la sortie dans `cmd.StdoutPipe` qui nous renvoie un `io.ReadCloser` (cela est important pour plus tard)
- Le reste du code est plus ou moins copié-collé de la [documentation excellente](https://golang.org/pkg/os/exec/#example_Cmd_StdoutPipe).
    - Nous capturons toute sortie de stdout dans un `io.ReadCloser`, puis nous `Start` (démarrons) la commande et attendons que toutes les données soient lues en appelant `Wait`. Entre ces deux appels, nous décodons les données dans notre structure `Payload`.

Voici ce que contient `msg.xml`

```xml
<payload>
    <message>Happy New Year!</message>
</payload>
```

J'ai écrit un test simple pour montrer son fonctionnement

```go
func TestGetData(t *testing.T) {
	got := GetData()
	want := "HAPPY NEW YEAR!"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

## Code testable

Un code testable est découplé et à but unique. Pour moi, il semble y avoir deux préoccupations principales pour ce code :

1. Récupérer les données XML brutes
2. Décoder les données XML et appliquer notre logique métier (dans ce cas `strings.ToUpper` sur le `<message>`)

La première partie consiste simplement à copier l'exemple de la bibliothèque standard.

La deuxième partie est celle où se trouve notre logique métier et en regardant le code, nous pouvons voir où commence la "couture" dans notre logique : c'est là où nous obtenons notre `io.ReadCloser`. Nous pouvons utiliser cette abstraction existante pour séparer les préoccupations et rendre notre code testable.

**Le problème avec GetData est que la logique métier est couplée avec le moyen d'obtenir le XML. Pour améliorer notre conception, nous devons les découpler**

Notre `TestGetData` peut servir de test d'intégration entre nos deux préoccupations, donc nous le conserverons pour nous assurer que tout continue à fonctionner.

Voici à quoi ressemble le code nouvellement séparé

```go
type Payload struct {
	Message string `xml:"message"`
}

func GetData(data io.Reader) string {
	var payload Payload
	xml.NewDecoder(data).Decode(&payload)
	return strings.ToUpper(payload.Message)
}

func getXMLFromCommand() io.Reader {
	cmd := exec.Command("cat", "msg.xml")
	out, _ := cmd.StdoutPipe()

	cmd.Start()
	data, _ := io.ReadAll(out)
	cmd.Wait()

	return bytes.NewReader(data)
}

func TestGetDataIntegration(t *testing.T) {
	got := GetData(getXMLFromCommand())
	want := "HAPPY NEW YEAR!"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

Maintenant que `GetData` prend son entrée à partir d'un simple `io.Reader`, nous l'avons rendu testable et il ne se préoccupe plus de la façon dont les données sont récupérées ; les gens peuvent réutiliser la fonction avec n'importe quoi qui renvoie un `io.Reader` (ce qui est extrêmement courant). Par exemple, nous pourrions commencer à récupérer le XML depuis une URL au lieu de la ligne de commande.

```go
func TestGetData(t *testing.T) {
	input := strings.NewReader(`
<payload>
    <message>Cats are the best animal</message>
</payload>`)

	got := GetData(input)
	want := "CATS ARE THE BEST ANIMAL"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}

```

Voici un exemple de test unitaire pour `GetData`.

En séparant les préoccupations et en utilisant les abstractions existantes dans Go, tester notre logique métier importante devient un jeu d'enfant.
# Injection de dépendances

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/di)**

Il est supposé que vous avez lu la [section sur les structs](./structs-methods-and-interfaces.md) auparavant, car une certaine compréhension des interfaces sera nécessaire pour cela.

Il y a _beaucoup_ de malentendus autour de l'injection de dépendances dans la communauté de programmation. Nous espérons que ce guide vous montrera que :

* Vous n'avez pas besoin d'un framework
* Cela ne complique pas excessivement votre conception
* Cela facilite les tests
* Cela vous permet d'écrire d'excellentes fonctions à usage général.

Nous voulons écrire une fonction qui salue quelqu'un, tout comme nous l'avons fait dans le chapitre hello-world, mais cette fois, nous allons tester _l'impression réelle_.

Pour rappel, voici à quoi pourrait ressembler cette fonction :

```go
func Saluer(nom string) {
	fmt.Printf("Bonjour, %s", nom)
}
```

Mais comment tester cette fonction ? En appelant `Saluer`, elle va écrire sur la sortie standard, qui est difficile à capturer pour un test.

En Go, lorsque vous voulez afficher quelque chose, vous l'appelez par `fmt.Printf` qui, sous le capot, utilise `os.Stdout`, un [*os.File](https://golang.org/pkg/os/#File).

Pour savoir quels sont les défauts de cette approche, commençons par écrire un test :

## Écrivez le test d'abord

```go
func TestSaluer(t *testing.T) {
	Saluer("Chris")
}
```

Puis exécutez-le :

```text
=== RUN   TestSaluer
Bonjour, Chris--- PASS: TestSaluer (0.00s)
PASS
```

Maintenant, imaginez-vous qu'il y a des centaines de tests exécutant cette fonction, ce qui entraînerait des centaines de "Bonjour, Chris" à l'écran. À ce moment, vous pourriez vous dire "j'aimerais pouvoir capter ce que la fonction écrit".

Gardez à l'esprit que nous voulons également posséder la manière dont notre fonction affiche. Par exemple, nous pourrions vouloir un jour que la fonction écrire dans une file d'attente de messages plutôt que sur la sortie standard.

Nous avons besoin de :

1. Utiliser l'injection de dépendances pour injecter (ou passer) notre dépendance (à savoir ce qui affiche le contenu)
2. Intercepter ce qui est écrit, afin que nous puissions tester sa sortie

## Refactoriser

```go
func Saluer(f *os.File, nom string) {
	fmt.Fprintf(f, "Bonjour, %s", nom)
}
```

Nous avons modifié `Saluer` pour qu'elle ne puisse plus écrire sur le terminal directement, et qu'à la place nous lui passions un `File` sur lequel elle écrira. Ce qui est intéressant, c'est que `os.Stdout` est un `*os.File` donc pour utiliser notre fonction normalement, nous appelons `Saluer(os.Stdout, "world")`.

Comment tester cela ?

```go
func TestSaluer(t *testing.T) {
	buffer := bytes.Buffer{}
	Saluer(&buffer, "Chris")

	resultat := buffer.String()
	attendu := "Bonjour, Chris"

	if resultat != attendu {
		t.Errorf("resultat '%s', attendu '%s'", resultat, attendu)
	}
}
```

Nous utilisons la struct `bytes.Buffer` qui implémente l'interface `io.Writer`, ce qui nous permet de l'utiliser comme "capteur" d'écriture.

Ce test passera, parfait. Cela semble plutôt simple pour l'instant, mais nous avons introduit une dépendance dans notre fonction qui était auparavant masquée.

Cette façon de faire n'est pas idéale en cas d'utilisation depuis d'autres endroits du codebase :

```go
func main() {
	Saluer(os.Stdout, "Elodie")
}
```

Nous avons toujours besoin de savoir quoi utiliser pour écrire, mais ça n'est pas très important pour notre fonction `main`.

## Interface orientée

Notre fonction est conforme à l'interface `io.Writer`, qui est définie dans le package standard comme :

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

Si nous changeons notre fonction pour qu'elle utilise cette interface au lieu de la struct concrète `*os.File`, alors nous nous ouvrons à un usage plus polyvalent de notre fonction :

```go
func Saluer(writer io.Writer, nom string) {
	fmt.Fprintf(writer, "Bonjour, %s", nom)
}
```

`fmt.Fprintf` est comme `fmt.Printf` mais prend un `io.Writer` comme premier argument. Ce changement signifie que la fonction `Saluer` sera capable d'écrire sur n'importe quel objet supportant l'interface `Writer`, pas uniquement les fichiers.

Notre test reste inchangé, et notre utilisation par le biais de la fonction `main` est également simple :

```go
func main() {
	Saluer(os.Stdout, "Elodie")
}
```

Nous savons qu'`os.Stdout` implémente `io.Writer` donc notre code compilera.

## Étendre l'utilisation

Grâce à cette abstraction simple et à bien comprendre ce que nous essayons d'injecter (l'affichage d'un texte), nous pouvons maintenant rajouter d'autres cas d'usage qu'il aurait été difficile d'implémenter avant.

### Serveur HTTP

Dans un nouveau fichier, nous pouvons créer un serveur HTTP qui utilise notre fonction `Saluer` pour dire bonjour à des utilisateurs.

```go
func HandlerMonSalut(w http.ResponseWriter, r *http.Request) {
	Saluer(w, "monde")
}

func main() {
	http.ListenAndServe(":5000", http.HandlerFunc(HandlerMonSalut))
}
```

Nous avons introduit `HandlerMonSalut` qui prend un `http.ResponseWriter` et un `http.Request`. Lorsque nous implémentons des serveurs HTTP en Go, nous devons écrire une fonction ayant cette signature.

`http.ResponseWriter` implémente aussi l'interface `io.Writer`, ce qui signifie que nous pouvons réutiliser notre fonction `Saluer` dans notre gestionnaire. Pour que ce serveur web fonctionne, nous devons encore l'attacher à un port, ce que fait la ligne suivante. `http.HandlerFunc` convertit notre gestionnaire en un `http.Handler`, puis nous l'associons au serveur.

### Log synchronisé

Nous pourrions également vouloir utiliser notre fonction `Saluer` sur un logger synchronisé.

```go
func main() {
	Saluer(os.Stdout, "Elodie")

	logger := log.New(os.Stdout, "INFO: ", log.LstdFlags)
	Saluer(logger.Writer(), "Elodie")
}
```

Vous pouvez trouver plus d'informations sur le package `log` [ici](https://golang.org/pkg/log/).

#### La différence entre Fprintf et Fprint

Mais qu'est-ce que c'est que ce `fmt.Fprintf`? Et en quoi est-ce différent de `fmt.Fprint`?

* `fmt.Fprintf` accepte un format au milieu et des arguments à la fin, permettant d'injecter des variables dans la chaîne formatée.
* `fmt.Fprint` accepte simplement un Writer et une chaîne.

Il y a aussi des variantes comme `fmt.Printf` et `fmt.Print`. Celles-ci n'acceptent pas de `Writer` et écrivent par défaut sur la sortie standard. Elles sont très pratiques dans les applications simples mais, comme nous l'avons vu, finissent par limiter la testabilité.

## Conclusion

Notre première itération n'était pas testable car nous écrivions directement sur la sortie standard, un endroit difficile à surveiller.

En utilisant l'injection de dépendances, nous avons pu :

* **Tester notre code** : En introduisant la possibilité d'injecter notre dépendance d'écriture, nous pouvons contrôler ce que la fonction écrit pour pouvoir tester son comportement.
* **Séparer nos préoccupations** : Mettre à jour nos fonctions pour accepter des dépendances plutôt que de les instancier à l'intérieur clarifie ce qui est important pour cette fonction et ce qui ne l'est pas.
* **Autoriser notre code à être utilisé dans différents contextes** : Grâce à l'utilisation de l'interface `io.Writer` comme abstraction (plutôt que `*os.File`), notre fonction `Saluer` peut être utilisée dans diverses situations, comme le serveur web, les logs et les tests.

Cette approche est beaucoup plus simple et explicite que beaucoup de frameworks d'injection de dépendances qui existent pour d'autres langages, qui utilisent souvent la réflexion et parfois des fichiers de configuration pour déterminer quelle fonction a quelle dépendance. Au lieu de cela, nous utilisons simplement la technique de base du langage Go, à savoir les fonctions, les interfaces et la lisibilité pour atteindre les mêmes résultats.

### À retenir
* L'injection de dépendances est l'approche où l'on passe (ou injecte) une dépendance dans du code qui en a besoin, plutôt que le code qui crée ou recherche lui-même la dépendance.
* Cela rend le code plus flexible et testable.
* Go est particulièrement adapté à cette approche en raison de ses interfaces.
* Votre code devrait reposer sur des abstractions bien définies, pas sur des détails d'implémentation.
* Votre code devrait s'articuler autour des interfaces, pas autour de leurs implémentations concrètes.
# Lecture de fichiers

- **[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/reading-files)**
- [Voici une vidéo de moi travaillant sur le problème et répondant aux questions du stream Twitch](https://www.youtube.com/watch?v=nXts4dEJnkU)

Dans ce chapitre, nous allons apprendre comment lire des fichiers, en extraire des données et faire quelque chose d'utile.

Imaginez que vous travaillez avec un ami pour créer un logiciel de blog. L'idée est qu'un auteur écrira ses articles en markdown, avec des métadonnées en haut du fichier. Au démarrage, le serveur web lira un dossier pour créer des `Post`, puis une fonction `NewHandler` séparée utilisera ces `Post` comme source de données pour le serveur web du blog.

On nous a demandé de créer le package qui convertit un dossier donné de fichiers d'articles de blog en une collection de `Post`.

### Exemple de données

hello world.md
```markdown
Title: Hello, TDD world!
Description: First post on our wonderful blog
Tags: tdd, go
---
Hello world!

Le corps des articles commence après le `---`
```

### Données attendues

```go
type Post struct {
	Title, Description, Body string
	Tags                     []string
}
```

## Développement itératif et piloté par les tests

Nous adopterons une approche itérative où nous prenons toujours des étapes simples et sûres vers notre objectif.

Cela nous oblige à découper notre travail, mais nous devons veiller à ne pas tomber dans le piège d'une approche ["bottom-up"](https://en.wikipedia.org/wiki/Top-down_and_bottom-up_design).

Nous ne devrions pas faire confiance à notre imagination trop active lorsque nous commençons à travailler. Nous pourrions être tentés de créer une sorte d'abstraction qui n'est validée qu'une fois que nous avons tout assemblé, comme une sorte de `BlogPostFileParser`.

Ce n'est _pas_ itératif et cela nous prive des boucles de rétroaction serrées que le TDD est censé nous apporter.

Kent Beck dit :

> L'optimisme est un risque professionnel de la programmation. La rétroaction est le traitement.

Au lieu de cela, notre approche devrait s'efforcer d'être aussi proche que possible de la livraison d'une _réelle_ valeur pour le consommateur aussi rapidement que possible (souvent appelée "happy path"). Une fois que nous avons livré une petite quantité de valeur pour le consommateur de bout en bout, l'itération ultérieure du reste des exigences est généralement simple.

## Réfléchir au type de test que nous voulons voir

Rappelons-nous notre état d'esprit et nos objectifs au départ :

- **Écrire le test que nous voulons voir**. Réfléchissons à la façon dont nous aimerions utiliser le code que nous allons écrire du point de vue d'un consommateur.
- Se concentrer sur le _quoi_ et le _pourquoi_, mais ne pas se laisser distraire par le _comment_.

Notre package doit offrir une fonction qui peut être dirigée vers un dossier et nous retourner des articles.

```go
var posts []blogposts.Post
posts = blogposts.NewPostsFromFS("some-folder")
```

Pour écrire un test à ce sujet, nous aurions besoin d'une sorte de dossier de test avec quelques exemples d'articles. _Il n'y a rien de terriblement mal à cela_, mais vous faites quelques compromis :

- pour chaque test, vous devrez peut-être créer de nouveaux fichiers pour tester un comportement particulier
- certains comportements seront difficiles à tester, comme l'échec du chargement des fichiers
- les tests s'exécuteront un peu plus lentement car ils devront accéder au système de fichiers

Nous nous couplons également inutilement à une implémentation spécifique du système de fichiers.

### Abstractions du système de fichiers introduites dans Go 1.16

Go 1.16 a introduit une abstraction pour les systèmes de fichiers ; le package [io/fs](https://golang.org/pkg/io/fs/).

> Le package fs définit des interfaces de base pour un système de fichiers. Un système de fichiers peut être fourni par le système d'exploitation hôte mais aussi par d'autres packages.

Cela nous permet de desserrer notre couplage à un système de fichiers spécifique, ce qui nous permettra ensuite d'injecter différentes implémentations selon nos besoins.

> [Du côté du producteur de l'interface, le nouveau type embed.FS implémente fs.FS, tout comme zip.Reader. La nouvelle fonction os.DirFS fournit une implémentation de fs.FS soutenue par un arbre de fichiers du système d'exploitation.](https://golang.org/doc/go1.16#fs)

Si nous utilisons cette interface, les utilisateurs de notre package disposent d'un certain nombre d'options intégrées à la bibliothèque standard à utiliser. Apprendre à tirer parti des interfaces définies dans la bibliothèque standard de Go (par exemple, `io.fs`, [`io.Reader`](https://golang.org/pkg/io/#Reader), [`io.Writer`](https://golang.org/pkg/io/#Writer)) est vital pour écrire des packages faiblement couplés. Ces packages peuvent ensuite être réutilisés dans des contextes différents de ceux que vous avez imaginés, avec un minimum de tracas pour vos consommateurs.

Dans notre cas, peut-être que notre consommateur souhaite que les articles soient intégrés dans le binaire Go plutôt que dans des fichiers d'un système de fichiers "réel" ? Quoi qu'il en soit, _notre code n'a pas besoin de s'en soucier_.

Pour nos tests, le package [testing/fstest](https://golang.org/pkg/testing/fstest/) nous offre une implémentation de [io/FS](https://golang.org/pkg/io/fs/#FS) à utiliser, similaire aux outils que nous connaissons dans [net/http/httptest](https://golang.org/pkg/net/http/httptest/).

Compte tenu de ces informations, l'approche suivante semble meilleure,

```go
var posts []blogposts.Post
posts = blogposts.NewPostsFromFS(someFS)
```


## Écrire le test d'abord

Nous devrions garder la portée aussi petite et utile que possible. Si nous prouvons que nous pouvons lire tous les fichiers d'un répertoire, ce sera un bon début. Cela nous donnera confiance dans le logiciel que nous écrivons. Nous pouvons vérifier que le nombre de `[]Post` retournés est le même que le nombre de fichiers dans notre système de fichiers fictif.

Créez un nouveau projet pour travailler sur ce chapitre.

- `mkdir blogposts`
- `cd blogposts`
- `go mod init github.com/{votre-nom}/blogposts`
- `touch blogposts_test.go`

```go
package blogposts_test

import (
	"testing"
	"testing/fstest"
)

func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("hi")},
		"hello-world2.md": {Data: []byte("hola")},
	}

	posts := blogposts.NewPostsFromFS(fs)

	if len(posts) != len(fs) {
		t.Errorf("got %d posts, wanted %d posts", len(posts), len(fs))
	}
}

```

Notez que le package de notre test est `blogposts_test`. Rappelez-vous, lorsque le TDD est bien pratiqué, nous adoptons une approche _pilotée par le consommateur_ : nous ne voulons pas tester les détails internes car les _consommateurs_ ne s'en soucient pas. En ajoutant `_test` à notre nom de package prévu, nous n'accédons qu'aux membres exportés de notre package - tout comme un vrai utilisateur de notre package.

Nous avons importé [`testing/fstest`](https://golang.org/pkg/testing/fstest/) qui nous donne accès au type [`fstest.MapFS`](https://golang.org/pkg/testing/fstest/#MapFS). Notre système de fichiers fictif passera `fstest.MapFS` à notre package.

> Un MapFS est un système de fichiers simple en mémoire à utiliser dans les tests, représenté comme une carte des noms de chemins (arguments à Open) aux informations sur les fichiers ou répertoires qu'ils représentent.

Cela semble plus simple que de maintenir un dossier de fichiers de test, et cela s'exécutera plus rapidement.

Enfin, nous avons codifié l'utilisation de notre API du point de vue d'un consommateur, puis vérifié si elle crée le bon nombre d'articles.

## Essayer d'exécuter le test

```
./blogpost_test.go:15:12: undefined: blogposts
```

## Écrire le minimum de code pour que le test s'exécute et _vérifier la sortie du test qui échoue_

Le package n'existe pas. Créez un nouveau fichier `blogposts.go` et mettez `package blogposts` à l'intérieur. Vous devrez ensuite importer ce package dans vos tests. Pour moi, les imports ressemblent maintenant à :

```go
import (
	blogposts "github.com/quii/learn-go-with-tests/reading-files"
	"testing"
	"testing/fstest"
)
```

Maintenant, les tests ne compileront pas car notre nouveau package n'a pas de fonction `NewPostsFromFS`, qui renvoie un type de collection.

```
./blogpost_test.go:16:12: undefined: blogposts.NewPostsFromFS
```

Cela nous oblige à créer le squelette de notre fonction pour faire fonctionner le test. N'oubliez pas de ne pas trop réfléchir au code à ce stade ; nous essayons simplement d'obtenir un test qui s'exécute, et de nous assurer qu'il échoue comme prévu. Si nous sautons cette étape, nous risquons de passer sur des hypothèses et d'écrire un test qui n'est pas utile.

```go
package blogposts

import "testing/fstest"

type Post struct {
}

func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	return nil
}
```

Le test devrait maintenant échouer correctement

```
=== RUN   TestNewBlogPosts
    blogposts_test.go:48: got 0 posts, wanted 2 posts
```

## Écrire suffisamment de code pour le faire passer

Nous _pourrions_ ["slimer"](https://deniseyu.github.io/leveling-up-tdd/) pour le faire passer :

```go
func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	return []Post{{}, {}}
}
```

Mais, comme l'a écrit Denise Yu :

>Le sliming est utile pour donner un "squelette" à votre objet. Concevoir une interface et exécuter une logique sont deux préoccupations, et slimer les tests de manière stratégique vous permet de vous concentrer sur l'une à la fois.

Nous avons déjà notre structure. Alors, que faisons-nous à la place ?

Comme nous avons réduit la portée, tout ce que nous devons faire est de lire le répertoire et de créer un article pour chaque fichier que nous rencontrons. Nous n'avons pas à nous soucier d'ouvrir des fichiers et de les analyser pour le moment.

```go
func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	dir, _ := fs.ReadDir(fileSystem, ".")
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts
}
```

[`fs.ReadDir`](https://golang.org/pkg/io/fs/#ReadDir) lit un répertoire dans un `fs.FS` donné et renvoie [`[]DirEntry`](https://golang.org/pkg/io/fs/#DirEntry).

Notre vision idéalisée du monde a déjà été déjouée car des erreurs peuvent se produire, mais rappelez-vous que notre objectif est maintenant _de faire passer le test_, pas de changer la conception, nous ignorerons donc l'erreur pour l'instant.

Le reste du code est simple : parcourir les entrées, créer un `Post` pour chacune et retourner la slice.

## Refactoriser

Même si nos tests réussissent, nous ne pouvons pas utiliser notre nouveau package en dehors de ce contexte, car il est couplé à une implémentation concrète `fstest.MapFS`. Mais ce n'est pas obligatoire. Modifiez l'argument de notre fonction `NewPostsFromFS` pour accepter l'interface de la bibliothèque standard.

```go
func NewPostsFromFS(fileSystem fs.FS) []Post {
	dir, _ := fs.ReadDir(fileSystem, ".")
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts
}
```

Relancez les tests : tout devrait fonctionner.

### Gestion des erreurs

Nous avons mis de côté la gestion des erreurs plus tôt lorsque nous nous sommes concentrés sur le fonctionnement du happy path. Avant de continuer à itérer sur la fonctionnalité, nous devrions reconnaître que des erreurs peuvent se produire lors du travail avec des fichiers. Au-delà de la lecture du répertoire, nous pouvons rencontrer des problèmes lors de l'ouverture de fichiers individuels. Modifions notre API (via nos tests d'abord, naturellement) pour qu'elle puisse renvoyer une `error`.

```go
func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("hi")},
		"hello-world2.md": {Data: []byte("hola")},
	}

	posts, err := blogposts.NewPostsFromFS(fs)

	if err != nil {
		t.Fatal(err)
	}

	if len(posts) != len(fs) {
		t.Errorf("got %d posts, wanted %d posts", len(posts), len(fs))
	}
}
```

Exécutez le test : il devrait se plaindre du mauvais nombre de valeurs de retour. La correction du code est simple.

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts, nil
}
```

Cela fera passer le test. Le praticien TDD en vous pourrait être agacé que nous n'ayons pas vu un test échouer avant d'écrire le code pour propager l'erreur de `fs.ReadDir`. Pour faire cela "correctement", nous aurions besoin d'un nouveau test où nous injectons un double de test `fs.FS` défaillant pour faire en sorte que `fs.ReadDir` renvoie une `error`.

```go
type StubFailingFS struct {
}

func (s StubFailingFS) Open(name string) (fs.File, error) {
	return nil, errors.New("oh non, j'échoue toujours")
}
```
```go
// plus tard
_, err := blogposts.NewPostsFromFS(StubFailingFS{})
```

Cela devrait vous donner confiance dans notre approche. L'interface que nous utilisons a une seule méthode, ce qui rend la création de doubles de test pour tester différents scénarios triviale.

Dans certains cas, tester la gestion des erreurs est la chose pragmatique à faire mais, dans notre cas, nous ne faisons rien d'_intéressant_ avec l'erreur, nous la propageons simplement, donc cela ne vaut pas la peine d'écrire un nouveau test.

Logiquement, nos prochaines itérations porteront sur l'expansion de notre type `Post` pour qu'il contienne des données utiles.

## Écrire le test d'abord

Nous allons commencer par la première ligne du schéma d'article de blog proposé, le champ titre.

Nous devons modifier le contenu des fichiers de test pour qu'ils correspondent à ce qui a été spécifié, puis nous pouvons faire une assertion qu'il est analysé correctement.
```go
func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("Title: Post 1")},
		"hello-world2.md": {Data: []byte("Title: Post 2")},
	}

	// le reste du code de test est coupé par souci de brièveté
	got := posts[0]
	want := blogposts.Post{Title: "Post 1"}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %+v, want %+v", got, want)
	}
}
```

## Essayer d'exécuter le test
```
./blogpost_test.go:58:26: unknown field 'Title' in struct literal of type blogposts.Post
```

## Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test qui échoue

Ajoutez le nouveau champ à notre type `Post` pour que le test s'exécute

```go
type Post struct {
	Title string
}
```

Relancez le test, et vous devriez obtenir un test qui échoue clairement

```
=== RUN   TestNewBlogPosts
=== RUN   TestNewBlogPosts/parses_the_post
    blogpost_test.go:61: got {Title:}, want {Title:Post 1}
```

## Écrire suffisamment de code pour le faire passer

Nous devrons ouvrir chaque fichier puis extraire le titre

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for _, f := range dir {
		post, err := getPost(fileSystem, f)
		if err != nil {
			return nil, err //todo: needs clarification, should we totally fail if one file fails? or just ignore?
		}
		posts = append(posts, post)
	}
	return posts, nil
}

func getPost(fileSystem fs.FS, f fs.DirEntry) (Post, error) {
	postFile, err := fileSystem.Open(f.Name())
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()

	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

Rappelez-vous que notre objectif à ce stade n'est pas d'écrire un code élégant, c'est juste d'arriver à un point où nous avons un logiciel qui fonctionne.

Même si cela semble une petite avancée, cela nous a quand même obligé à écrire une bonne quantité de code et à faire des suppositions concernant la gestion des erreurs. Ce serait un point où vous devriez parler à vos collègues et décider de la meilleure approche.

L'approche itérative nous a donné un retour rapide que notre compréhension des exigences est incomplète.

`fs.FS` nous donne un moyen d'ouvrir un fichier à l'intérieur par nom avec sa méthode `Open`. À partir de là, nous lisons les données du fichier et, pour l'instant, nous n'avons pas besoin d'une analyse sophistiquée, juste de couper le texte `Title: ` en découpant la chaîne.

## Refactoriser

Séparer le 'code d'ouverture de fichier' du 'code d'analyse du contenu du fichier' rendra le code plus simple à comprendre et à utiliser.

```go
func getPost(fileSystem fs.FS, f fs.DirEntry) (Post, error) {
	postFile, err := fileSystem.Open(f.Name())
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()
	return newPost(postFile)
}

func newPost(postFile fs.File) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

Lorsque vous refactorisez de nouvelles fonctions ou méthodes, faites attention et réfléchissez aux arguments. Vous êtes en train de concevoir ici, et vous êtes libre de réfléchir profondément à ce qui est approprié car vous avez des tests qui passent. Pensez au couplage et à la cohésion. Dans ce cas, vous devriez vous demander :

> Est-ce que `newPost` doit être couplé à un `fs.File` ? Utilisons-nous toutes les méthodes et données de ce type ? De quoi avons-nous _vraiment_ besoin ?

Dans notre cas, nous l'utilisons uniquement comme argument pour `io.ReadAll` qui a besoin d'un `io.Reader`. Nous devrions donc assouplir le couplage dans notre fonction et demander un `io.Reader`.

```go
func newPost(postFile io.Reader) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

Vous pouvez faire un argument similaire pour notre fonction `getPost`, qui prend un argument `fs.DirEntry` mais appelle simplement `Name()` pour obtenir le nom du fichier. Nous n'avons pas besoin de tout cela ; découplons de ce type et passons le nom du fichier comme une chaîne. Voici le code entièrement refactorisé :

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for _, f := range dir {
		post, err := getPost(fileSystem, f.Name())
		if err != nil {
			return nil, err //todo: needs clarification, should we totally fail if one file fails? or just ignore?
		}
		posts = append(posts, post)
	}
	return posts, nil
}

func getPost(fileSystem fs.FS, fileName string) (Post, error) {
	postFile, err := fileSystem.Open(fileName)
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()
	return newPost(postFile)
}

func newPost(postFile io.Reader) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

À partir de maintenant, la plupart de nos efforts peuvent être soigneusement contenus dans `newPost`. Les préoccupations d'ouverture et d'itération sur les fichiers sont terminées, et maintenant nous pouvons nous concentrer sur l'extraction des données pour notre type `Post`. Bien que ce ne soit pas techniquement nécessaire, les fichiers sont un bon moyen de regrouper logiquement des choses connexes, donc j'ai déplacé le type `Post` et `newPost` dans un nouveau fichier `post.go`.

### Fonction d'aide pour les tests

Nous devrions également prendre soin de nos tests. Nous allons faire beaucoup d'assertions sur `Posts`, nous devrions donc écrire du code pour nous aider avec cela

```go
func assertPost(t *testing.T, got blogposts.Post, want blogposts.Post) {
	t.Helper()
	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %+v, want %+v", got, want)
	}
}
```

```go
assertPost(t, posts[0], blogposts.Post{Title: "Post 1"})
```

## Écrire le test d'abord

Étendons notre test pour extraire la ligne suivante du fichier, la description. Jusqu'à ce que cela passe, cela devrait maintenant sembler confortable et familier.

```go
func TestNewBlogPosts(t *testing.T) {
	const (
		firstBody = `Title: Post 1
Description: Description 1`
		secondBody = `Title: Post 2
Description: Description 2`
	)

	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte(firstBody)},
		"hello-world2.md": {Data: []byte(secondBody)},
	}

	// le reste du code de test est coupé par souci de brièveté
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
	})

}
```

## Essayer d'exécuter le test

```
./blogpost_test.go:47:58: unknown field 'Description' in struct literal of type blogposts.Post
```

## Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test qui échoue

Ajoutez le nouveau champ à `Post`.

```go
type Post struct {
	Title       string
	Description string
}
```

Les tests devraient maintenant compiler et échouer.

```
=== RUN   TestNewBlogPosts
    blogpost_test.go:47: got {Title:Post 1
        Description: Description 1 Description:}, want {Title:Post 1 Description:Description 1}
```

## Écrire suffisamment de code pour le faire passer

La bibliothèque standard a une bibliothèque pratique pour vous aider à parcourir les données, ligne par ligne ; [`bufio.Scanner`](https://golang.org/pkg/bufio/#Scanner)

> Scanner fournit une interface pratique pour lire des données telles qu'un fichier de lignes de texte délimitées par des sauts de ligne.

```go
func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	scanner.Scan()
	titleLine := scanner.Text()

	scanner.Scan()
	descriptionLine := scanner.Text()

	return Post{Title: titleLine[7:], Description: descriptionLine[13:]}, nil
}
```

Heureusement, il prend également un `io.Reader` à lire (merci encore, couplage lâche), nous n'avons pas besoin de changer nos arguments de fonction.

Appelez `Scan` pour lire une ligne, puis extrayez les données en utilisant `Text`.

Cette fonction ne pourrait jamais renvoyer d'`error`. Il serait tentant à ce stade de la supprimer du type de retour, mais nous savons que nous devrons gérer des structures de fichiers non valides plus tard, alors nous pouvons aussi bien la laisser.

## Refactoriser

Nous avons une répétition autour de la numérisation d'une ligne puis de la lecture du texte. Nous savons que nous allons effectuer cette opération au moins une fois de plus, c'est une refactorisation simple pour sécher, alors commençons par là.

```go
func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	readLine := func() string {
		scanner.Scan()
		return scanner.Text()
	}

	title := readLine()[7:]
	description := readLine()[13:]

	return Post{Title: title, Description: description}, nil
}
```

Cela a à peine économisé des lignes de code, mais ce n'est rarement le but de la refactorisation. Ce que j'essaie de faire ici, c'est juste de séparer le _quoi_ du _comment_ de la lecture des lignes pour rendre le code un peu plus déclaratif pour le lecteur.

Bien que les nombres magiques 7 et 13 fassent le travail, ils ne sont pas terriblement descriptifs.

```go
const (
	titleSeparator       = "Title: "
	descriptionSeparator = "Description: "
)

func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	readLine := func() string {
		scanner.Scan()
		return scanner.Text()
	}

	title := readLine()[len(titleSeparator):]
	description := readLine()[len(descriptionSeparator):]

	return Post{Title: title, Description: description}, nil
}
```

Maintenant que je fixe le code avec mon esprit de refactorisation créatif, j'aimerais essayer de faire en sorte que notre fonction readLine s'occupe de supprimer la balise. Il existe également une façon plus lisible de supprimer un préfixe d'une chaîne avec la fonction `strings.TrimPrefix`.

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
	}, nil
}
```

Vous pourriez aimer cette idée ou non, mais moi, je l'aime. Le point est que dans l'état de refactorisation, nous sommes libres de jouer avec les détails internes, et vous pouvez continuer à exécuter vos tests pour vérifier que les choses fonctionnent toujours correctement. Nous pouvons toujours revenir à des états précédents si nous ne sommes pas satisfaits. L'approche TDD nous donne cette licence pour expérimenter fréquemment des idées, nous avons donc plus de chances d'écrire du code de qualité.

L'exigence suivante est d'extraire les tags de l'article. Si vous suivez, je vous recommande d'essayer de l'implémenter vous-même avant de continuer à lire. Vous devriez maintenant avoir un bon rythme itératif et vous sentir confiant pour extraire la ligne suivante et analyser les données.

Par souci de brièveté, je ne passerai pas par toutes les étapes du TDD, mais voici le test avec les tags ajoutés.

```go
func TestNewBlogPosts(t *testing.T) {
	const (
		firstBody = `Title: Post 1
Description: Description 1
Tags: tdd, go`
		secondBody = `Title: Post 2
Description: Description 2
Tags: rust, borrow-checker`
	)

	// le reste du code de test est coupé par souci de brièveté
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
		Tags:        []string{"tdd", "go"},
	})
}
```

Vous ne vous rendez service à personne si vous vous contentez de copier et coller ce que j'écris. Pour nous assurer que nous sommes tous sur la même page, voici mon code qui inclut l'extraction des tags.

```go
const (
	titleSeparator       = "Title: "
	descriptionSeparator = "Description: "
	tagsSeparator        = "Tags: "
)

func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
		Tags:        strings.Split(readMetaLine(tagsSeparator), ", "),
	}, nil
}
```

Espérons qu'il n'y ait pas de surprises ici. Nous avons pu réutiliser `readMetaLine` pour obtenir la ligne suivante pour les tags, puis les diviser en utilisant `strings.Split`.

La dernière itération sur notre happy path est d'extraire le corps de l'article.

Voici un rappel du format de fichier proposé.

```markdown
Title: Hello, TDD world!
Description: First post on our wonderful blog
Tags: tdd, go
---
Hello world!

Le corps des articles commence après le `---`
```

Nous avons déjà lu les 3 premières lignes. Nous devons ensuite lire une ligne de plus, la rejeter, puis le reste du fichier contient le corps de l'article.

## Écrire le test d'abord

Modifiez les données de test pour avoir le séparateur, et un corps avec quelques sauts de ligne pour vérifier que nous capturons tout le contenu.

```go
	const (
		firstBody = `Title: Post 1
Description: Description 1
Tags: tdd, go
---
Hello
World`
		secondBody = `Title: Post 2
Description: Description 2
Tags: rust, borrow-checker
---
B
L
M`
	)
```

Ajoutez à notre assertion comme les autres

```go
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
		Tags:        []string{"tdd", "go"},
		Body: `Hello
World`,
	})
```

## Essayer d'exécuter le test

```
./blogpost_test.go:60:3: unknown field 'Body' in struct literal of type blogposts.Post
```

Comme nous nous y attendions.

## Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test qui échoue

Ajoutez `Body` à `Post` et le test devrait échouer.

```
=== RUN   TestNewBlogPosts
    blogposts_test.go:38: got {Title:Post 1 Description:Description 1 Tags:[tdd go] Body:}, want {Title:Post 1 Description:Description 1 Tags:[tdd go] Body:Hello
        World}
```

## Écrire suffisamment de code pour le faire passer

1. Scannez la ligne suivante pour ignorer le séparateur `---`.
2. Continuez à scanner jusqu'à ce qu'il n'y ait plus rien à scanner.

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	title := readMetaLine(titleSeparator)
	description := readMetaLine(descriptionSeparator)
	tags := strings.Split(readMetaLine(tagsSeparator), ", ")

	scanner.Scan() // ignorer une ligne

	buf := bytes.Buffer{}
	for scanner.Scan() {
		fmt.Fprintln(&buf, scanner.Text())
	}
	body := strings.TrimSuffix(buf.String(), "\n")

	return Post{
		Title:       title,
		Description: description,
		Tags:        tags,
		Body:        body,
	}, nil
}
```

- `scanner.Scan()` renvoie un `bool` qui indique s'il y a plus de données à scanner, nous pouvons donc l'utiliser avec une boucle `for` pour continuer à lire les données jusqu'à la fin.
- Après chaque `Scan()`, nous écrivons les données dans le tampon en utilisant `fmt.Fprintln`. Nous utilisons la version qui ajoute un saut de ligne car le scanner supprime les sauts de ligne de chaque ligne, mais nous devons les conserver.
- En raison de ce qui précède, nous devons supprimer le saut de ligne final, afin de ne pas en avoir un en trop.

## Refactoriser

Encapsuler l'idée d'obtenir le reste des données dans une fonction aidera les futurs lecteurs à comprendre rapidement _ce qui_ se passe dans `newPost`, sans avoir à se préoccuper des détails d'implémentation.

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
		Tags:        strings.Split(readMetaLine(tagsSeparator), ", "),
		Body:        readBody(scanner),
	}, nil
}

func readBody(scanner *bufio.Scanner) string {
	scanner.Scan() // ignorer une ligne
	buf := bytes.Buffer{}
	for scanner.Scan() {
		fmt.Fprintln(&buf, scanner.Text())
	}
	return strings.TrimSuffix(buf.String(), "\n")
}
```

## Itérer davantage

Nous avons créé notre "fil d'acier" de fonctionnalité, en prenant le chemin le plus court pour arriver à notre happy path, mais il est clair qu'il y a encore du chemin à parcourir avant qu'il ne soit prêt pour la production.

Nous n'avons pas géré :

- quand le format du fichier n'est pas correct
- le fichier n'est pas un `.md`
- que se passe-t-il si l'ordre des champs de métadonnées est différent ? Cela devrait-il être autorisé ? Devrions-nous être en mesure de le gérer ?

Cependant, de façon cruciale, nous avons un logiciel qui fonctionne et nous avons défini notre interface. Les points ci-dessus ne sont que des itérations supplémentaires, plus de tests à écrire et à diriger notre comportement. Pour prendre en charge l'un des points ci-dessus, nous ne devrions pas avoir à changer notre _conception_, juste les détails d'implémentation.

Se concentrer sur l'objectif signifie que nous avons pris les décisions importantes et les avons validées par rapport au comportement souhaité, plutôt que de nous enliser dans des questions qui n'affecteront pas la conception globale.

## Conclusion

`fs.FS` et les autres changements dans Go 1.16 nous donnent des moyens élégants de lire des données à partir des systèmes de fichiers et de les tester simplement.

Si vous souhaitez essayer le code "pour de vrai" :

- Créez un dossier `cmd` dans le projet, ajoutez un fichier `main.go`
- Ajoutez le code suivant

```go
import (
	blogposts "github.com/quii/fstest-spike"
	"log"
	"os"
)

func main() {
	posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
	if err != nil {
		log.Fatal(err)
	}
	log.Println(posts)
}
```

- Ajoutez quelques fichiers markdown dans un dossier `posts` et exécutez le programme !

Remarquez la symétrie entre le code de production

```go
posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
```

Et les tests

```go
posts, err := blogposts.NewPostsFromFS(fs)
```

C'est à ce moment que le TDD descendant, piloté par le consommateur, _semble correct_.

Un utilisateur de notre package peut consulter nos tests et comprendre rapidement ce qu'il est censé faire et comment l'utiliser. En tant que mainteneurs, nous pouvons être _confiants que nos tests sont utiles car ils sont du point de vue d'un consommateur_. Nous ne testons pas les détails d'implémentation ou d'autres détails accessoires, nous pouvons donc être raisonnablement confiants que nos tests nous aideront, plutôt que de nous gêner lors de la refactorisation.

En s'appuyant sur de bonnes pratiques d'ingénierie logicielle comme [**l'injection de dépendances**](dependency-injection.md), notre code est simple à tester et à réutiliser.

Lorsque vous créez des packages, même s'ils sont uniquement internes à votre projet, préférez une approche descendante pilotée par le consommateur. Cela vous empêchera de sur-imaginer des conceptions et de faire des abstractions dont vous n'avez même pas besoin, et cela vous aidera à garantir que les tests que vous écrivez sont utiles.

L'approche itérative a maintenu chaque étape petite, et la rétroaction continue nous a aidés à découvrir des exigences peu claires, peut-être plus tôt qu'avec d'autres approches plus ad hoc.

### Écriture ?

Il est important de noter que ces nouvelles fonctionnalités n'ont d'opérations que pour _lire_ des fichiers. Si votre travail a besoin d'écrire, vous devrez chercher ailleurs. N'oubliez pas de continuer à réfléchir à ce que la bibliothèque standard offre actuellement, si vous écrivez des données, vous devriez probablement envisager d'utiliser des interfaces existantes telles que `io.Writer` pour garder votre code faiblement couplé et réutilisable.

### Lectures complémentaires

- C'était une introduction légère à `io/fs`. [Ben Congdon a fait un excellent article](https://benjamincongdon.me/blog/2021/01/21/A-Tour-of-Go-116s-iofs-package/) qui a été d'une grande aide pour écrire ce chapitre.
- [Discussion sur les interfaces du système de fichiers](https://github.com/golang/go/issues/41190)
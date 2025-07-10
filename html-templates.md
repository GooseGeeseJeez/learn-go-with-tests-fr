# Templates HTML

**[Vous pouvez trouver tout le code ici](https://github.com/quii/learn-go-with-tests/tree/main/blogrenderer)**

Nous vivons dans un monde où tout le monde veut construire des applications web avec le framework frontend à la mode du moment, construit sur des gigaoctets de JavaScript transpilé, fonctionnant avec un système de build byzantin ; [mais ce n'est peut-être pas toujours nécessaire](https://quii.dev/The_Web_I_Want).

Je dirais que la plupart des développeurs Go apprécient une chaîne d'outils simple, stable et rapide, mais le monde du frontend échoue souvent à répondre à ces attentes.

De nombreux sites web n'ont pas besoin d'être des [SPA](https://en.wikipedia.org/wiki/Single-page_application). **HTML et CSS sont d'excellents moyens de diffuser du contenu** et vous pouvez utiliser Go pour créer un site web qui diffuse du HTML.

Si vous souhaitez avoir des éléments dynamiques, vous pouvez toujours ajouter un peu de JavaScript côté client, ou vous pourriez même vouloir expérimenter avec [Hotwire](https://hotwired.dev) qui vous permet d'offrir une expérience dynamique avec une approche côté serveur.

Vous pouvez générer votre HTML en Go avec une utilisation élaborée de [`fmt.Fprintf`](https://pkg.go.dev/fmt#Fprintf), mais dans ce chapitre, vous apprendrez que la bibliothèque standard de Go dispose d'outils pour générer du HTML de manière plus simple et plus maintenable. Vous découvrirez également des méthodes plus efficaces pour tester ce type de code auxquelles vous n'avez peut-être pas encore pensé.

## Ce que nous allons construire

Dans le chapitre [Lecture de fichiers](/reading-files.md), nous avons écrit du code qui prend un [`fs.FS`](https://pkg.go.dev/io/fs) (un système de fichiers) et renvoie une slice de `Post` pour chaque fichier markdown rencontré.

```go
posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
```

Voici comment nous avons défini `Post`

```go
type Post struct {
	Title, Description, Body string
	Tags                     []string
}
```

Voici un exemple de l'un des fichiers markdown qui peuvent être analysés.

```markdown
Title: Bienvenue sur mon blog
Description: Introduction à mon blog
Tags: cuisine, famille, vivre-rire-aimer
---
# Première recette !
Bienvenue sur mon **incroyable blog de recettes**. Je vais écrire à propos des recettes de ma famille, et je vais m'assurer d'écrire une histoire longue, hors sujet et ennuyeuse sur ma famille avant que vous n'arriviez aux instructions réelles.
```

Si nous poursuivons notre parcours de création d'un logiciel de blog, nous prendrions ces données et générerions du HTML à partir d'elles pour que notre serveur web les renvoie en réponse aux requêtes HTTP.

Pour notre blog, nous voulons générer deux types de pages :

1. **Voir le post**. Affiche un post spécifique. Le champ `Body` dans `Post` est une chaîne contenant du markdown, il doit donc être converti en HTML.
2. **Index**. Liste tous les posts, avec des hyperliens pour consulter le post spécifique.

Nous voulons également une apparence cohérente sur notre site. Ainsi, pour chaque page, nous aurons les éléments HTML habituels comme `<html>` et un `<head>` contenant des liens vers des feuilles de style CSS et tout ce dont nous pourrions avoir besoin.

Lorsque vous créez un logiciel de blog, vous avez plusieurs options en termes d'approche pour construire et envoyer du HTML au navigateur de l'utilisateur.

Nous allons concevoir notre code pour qu'il accepte un `io.Writer`. Cela signifie que l'appelant de notre code a la flexibilité de :

- Les écrire dans un [os.File](https://pkg.go.dev/os#File), afin qu'ils puissent être servis statiquement
- Écrire le HTML directement dans un [`http.ResponseWriter`](https://pkg.go.dev/net/http#ResponseWriter)
- Ou les écrire dans n'importe quoi ! Tant que cela implémente `io.Writer`, l'utilisateur peut générer du HTML à partir d'un `Post`

## Écrire le test d'abord

Comme toujours, il est important de réfléchir aux exigences avant de plonger trop vite. Comment pouvons-nous prendre cet ensemble d'exigences assez large et le décomposer en une petite étape réalisable sur laquelle nous pouvons nous concentrer ?

À mon avis, la visualisation du contenu est prioritaire par rapport à une page d'index. Nous pourrions lancer ce produit et partager des liens directs vers notre merveilleux contenu. Une page d'index qui ne peut pas renvoyer au contenu réel n'est pas utile.

Pourtant, le rendu d'un post tel que décrit précédemment semble encore important. Tous les éléments HTML, la conversion du corps en markdown en HTML, la liste des tags, etc.

À ce stade, je ne suis pas trop préoccupé par le balisage spécifique, et une première étape facile serait de vérifier que nous pouvons rendre le titre du post en tant que `<h1>`. Cela *semble* être la plus petite première étape qui peut nous faire avancer un peu.

```go
package blogrenderer_test

import (
	"bytes"
	"github.com/quii/learn-go-with-tests/blogrenderer"
	"testing"
)

func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	t.Run("il convertit un post unique en HTML", func(t *testing.T) {
		buf := bytes.Buffer{}
		err := blogrenderer.Render(&buf, aPost)

		if err != nil {
			t.Fatal(err)
		}

		got := buf.String()
		want := `<h1>hello world</h1>`
		if got != want {
			t.Errorf("obtenu '%s' attendu '%s'", got, want)
		}
	})
}
```

Notre décision d'accepter un `io.Writer` simplifie également les tests. Dans ce cas, nous écrivons dans un [`bytes.Buffer`](https://pkg.go.dev/bytes#Buffer) dont nous pourrons ensuite inspecter le contenu.

## Essayer d'exécuter le test

Si vous avez lu les chapitres précédents de ce livre, vous devriez être bien entraîné à cette étape maintenant. Vous ne pourrez pas exécuter le test car nous n'avons pas défini le package ou la fonction `Render`. Essayez de suivre les messages du compilateur vous-même et d'arriver à un état où vous pouvez exécuter le test et voir qu'il échoue avec un message clair.

Il est vraiment important que vous testiez l'échec de vos tests, vous vous remercierez lorsque vous ferez accidentellement échouer un test 6 mois plus tard d'avoir fait l'effort *maintenant* de vérifier qu'il échoue avec un message clair.

## Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test échouant

Voici le code minimal pour que le test s'exécute

```go
package blogrenderer

// si vous continuez à partir du chapitre sur la lecture de fichiers, vous ne devriez pas redéfinir ceci
type Post struct {
	Title, Description, Body string
	Tags                     []string
}

func Render(w io.Writer, p Post) error {
	return nil
}
```

Le test devrait se plaindre qu'une chaîne vide n'est pas égale à ce que nous voulons.

## Écrire suffisamment de code pour le faire passer

```go
func Render(w io.Writer, p Post) error {
	_, err := fmt.Fprintf(w, "<h1>%s</h1>", p.Title)
	return err
}
```

Rappelez-vous, le développement logiciel est avant tout une activité d'apprentissage. Pour découvrir et apprendre au fur et à mesure de notre travail, nous devons travailler d'une manière qui nous donne des boucles de rétroaction fréquentes et de haute qualité, et la façon la plus simple de le faire est de travailler par petites étapes.

Nous ne nous inquiétons donc pas d'utiliser des bibliothèques de templates pour l'instant. Vous pouvez très bien créer du HTML avec un modèle de chaîne "normal", et en sautant la partie modèle, nous pouvons valider un petit morceau de comportement utile et nous avons fait un petit travail de conception pour l'API de notre package.

## Refactoring

Pas grand chose à refactorer pour l'instant, alors passons à l'itération suivante

## Écrire le test d'abord

Maintenant que nous avons une version très basique qui fonctionne, nous pouvons itérer sur le test pour étendre les fonctionnalités. Dans ce cas, rendre plus d'informations à partir du `Post`.

```go
	t.Run("il convertit un post unique en HTML", func(t *testing.T) {
		buf := bytes.Buffer{}
		err := blogrenderer.Render(&buf, aPost)

		if err != nil {
			t.Fatal(err)
		}

		got := buf.String()
		want := `<h1>hello world</h1>
<p>This is a description</p>
Tags: <ul><li>go</li><li>tdd</li></ul>`

		if got != want {
			t.Errorf("obtenu '%s' attendu '%s'", got, want)
		}
	})
```

Remarquez que l'écriture de ceci *semble* maladroite. Voir tout ce balisage dans le test semble mauvais, et nous n'avons même pas mis le corps, ou le HTML réel que nous voudrions avec tout le contenu `<head>` et tout le mobilier de page dont nous avons besoin.

Néanmoins, supportons la douleur *pour l'instant*.

## Essayer d'exécuter le test

Il devrait échouer, se plaignant de ne pas avoir la chaîne que nous attendons, car nous ne rendons pas la description et les tags.

## Écrire suffisamment de code pour le faire passer

Essayez de le faire vous-même plutôt que de copier le code. Ce que vous devriez constater, c'est que faire passer ce test _est un peu ennuyeux_ ! Quand j'ai essayé, ma première tentative a renvoyé cette erreur

```
=== RUN   TestRender
=== RUN   TestRender/it_converts_a_single_post_into_HTML
    renderer_test.go:32: got '<h1>hello world</h1><p>This is a description</p><ul><li>go</li><li>tdd</li></ul>' want '<h1>hello world</h1>
        <p>This is a description</p>
        Tags: <ul><li>go</li><li></li></ul>'
```

Les nouvelles lignes ! Qui s'en soucie ? Eh bien, notre test s'en soucie, car il correspond à une valeur de chaîne exacte. Devrait-il ? J'ai supprimé les nouvelles lignes pour l'instant juste pour faire passer le test.

```go
func Render(w io.Writer, p Post) error {
	_, err := fmt.Fprintf(w, "<h1>%s</h1><p>%s</p>", p.Title, p.Description)
	if err != nil {
		return err
	}

	_, err = fmt.Fprint(w, "Tags: <ul>")
	if err != nil {
		return err
	}

	for _, tag := range p.Tags {
		_, err = fmt.Fprintf(w, "<li>%s</li>", tag)
		if err != nil {
			return err
		}
	}

	_, err = fmt.Fprint(w, "</ul>")
	if err != nil {
		return err
	}

	return nil
}
```

**Ouah**. Ce n'est pas le plus beau code que j'ai écrit, et nous n'en sommes encore qu'à une implémentation très précoce de notre balisage. Nous aurons besoin de beaucoup plus de contenu et de choses sur notre page, nous voyons rapidement que cette approche n'est pas appropriée.

Mais, crucialement, nous avons un test qui passe ; nous avons un logiciel qui fonctionne.

## Refactoring

Avec le filet de sécurité d'un test réussi pour un code qui fonctionne, nous pouvons maintenant envisager de changer notre approche d'implémentation au stade du refactoring.

### Introduction aux templates

Go dispose de deux packages de templates [text/template](https://pkg.go.dev/text/template) et [html/template](https://pkg.go.dev/html/template) qui partagent la même interface. Ce qu'ils font tous deux, c'est vous permettre de combiner un modèle et des données pour produire une chaîne.

Quelle est la différence avec la version HTML ?

> Le package template (html/template) implémente des modèles basés sur les données pour générer une sortie HTML sécurisée contre l'injection de code. Il fournit la même interface que le package text/template et devrait être utilisé à la place de text/template chaque fois que la sortie est du HTML.

Le langage de templating est très similaire à [Mustache](https://mustache.github.io) et vous permet de générer du contenu dynamiquement de manière très propre avec une belle séparation des préoccupations. Comparé à d'autres langages de templating que vous avez peut-être utilisés, il est très contraint ou "sans logique" comme aime le dire Mustache. C'est une décision de conception importante **et délibérée**.

Bien que nous nous concentrions ici sur la génération de HTML, si votre projet effectue des concaténations et des incantations de chaînes complexes, vous pourriez vouloir utiliser `text/template` pour nettoyer votre code.

### Retour au code

Voici un template pour notre blog :

`<h1>{{.Title}}</h1><p>{{.Description}}</p>Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>`

Où définir cette chaîne ? Eh bien, nous avons quelques options, mais pour garder les étapes petites, commençons simplement par une chaîne ordinaire

```go
package blogrenderer

import (
	"html/template"
	"io"
)

const (
	postTemplate = `<h1>{{.Title}}</h1><p>{{.Description}}</p>Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>`
)

func Render(w io.Writer, p Post) error {
	templ, err := template.New("blog").Parse(postTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, p); err != nil {
		return err
	}

	return nil
}
```

Nous créons un nouveau template avec un nom, puis nous analysons notre chaîne de template. Nous pouvons ensuite utiliser la méthode `Execute` dessus, en passant nos données, dans ce cas le `Post`.

Le template remplacera des éléments comme `{{.Description}}` par le contenu de `p.Description`. Les templates vous offrent également des primitives de programmation comme `range` pour parcourir des valeurs, et `if`. Vous pouvez trouver plus de détails dans la [documentation de text/template](https://pkg.go.dev/text/template).

*Ceci devrait être un pur refactoring.* Nous ne devrions pas avoir besoin de changer nos tests et ils devraient continuer à passer. Surtout, notre code est plus facile à lire et comporte beaucoup moins de gestion d'erreurs ennuyeuse à gérer.

Fréquemment, les gens se plaignent de la verbosité de la gestion des erreurs en Go, mais vous pourriez trouver que vous pouvez trouver de meilleures façons d'écrire votre code afin qu'il soit moins sujet aux erreurs en premier lieu, comme ici.

### Plus de refactoring

L'utilisation de `html/template` a certainement été une amélioration, mais l'avoir comme une constante de chaîne dans notre code n'est pas génial :

- C'est encore assez difficile à lire.
- Ce n'est pas convivial pour l'IDE/l'éditeur. Pas de coloration syntaxique, pas de possibilité de reformater, refactoriser, etc.
- Cela ressemble à du HTML, mais vous ne pouvez pas vraiment travailler avec comme vous pourriez le faire avec un fichier HTML "normal"

Ce que nous aimerions faire, c'est avoir nos templates dans des fichiers séparés afin de mieux les organiser et de travailler avec eux comme s'il s'agissait de fichiers HTML.

Créez un dossier appelé "templates" et à l'intérieur, créez un fichier appelé `blog.gohtml`, collez notre template dans le fichier.

Maintenant, modifiez notre code pour intégrer les systèmes de fichiers en utilisant [la fonctionnalité d'intégration incluse dans go 1.16](https://pkg.go.dev/embed).

```go
package blogrenderer

import (
	"embed"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

func Render(w io.Writer, p Post) error {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return err
	}

	if err := templ.Execute(w, p); err != nil {
		return err
	}

	return nil
}
```

En intégrant un "système de fichiers" dans notre code, nous pouvons charger plusieurs templates et les combiner librement. Cela deviendra utile lorsque nous voudrons partager la logique de rendu entre différents templates, comme un en-tête pour le haut de la page HTML et un pied de page.

### Embed ?

Embed a été brièvement abordé dans [lecture de fichiers](reading-files.md). [La documentation de la bibliothèque standard explique](https://pkg.go.dev/embed)

> Le package embed fournit un accès aux fichiers intégrés dans le programme Go en cours d'exécution.
>
> Les fichiers source Go qui importent "embed" peuvent utiliser la directive //go:embed pour initialiser une variable de type string, []byte ou FS avec le contenu des fichiers lus depuis le répertoire du package ou les sous-répertoires au moment de la compilation.

Pourquoi voudrions-nous utiliser cela ? Eh bien, l'alternative est que nous _pouvons_ charger nos templates à partir d'un système de fichiers "normal". Cependant, cela signifie que nous devrions nous assurer que les templates se trouvent dans le bon chemin de fichier partout où nous voulons utiliser ce logiciel. Dans votre travail, vous pouvez avoir différents environnements comme le développement, la préproduction et la production. Pour que cela fonctionne, vous devriez vous assurer que vos templates sont copiés au bon endroit.

Avec embed, les fichiers sont inclus dans votre programme Go lorsque vous le construisez. Cela signifie qu'une fois que vous avez construit votre programme (ce que vous ne devriez faire qu'une seule fois), les fichiers sont toujours disponibles pour vous.

Ce qui est pratique, c'est que vous pouvez non seulement intégrer des fichiers individuels, mais aussi des systèmes de fichiers ; et ce système de fichiers implémente [io/fs](https://pkg.go.dev/io/fs), ce qui signifie que votre code n'a pas besoin de se soucier du type de système de fichiers avec lequel il travaille.

Si vous souhaitez utiliser différents templates en fonction de la configuration, vous pourriez préférer charger les templates à partir du disque de manière plus conventionnelle.

## Ensuite : Rendre le template "agréable"

Nous ne voulons pas vraiment que notre template soit défini comme une chaîne d'une ligne. Nous voulons pouvoir l'espacer pour le rendre plus facile à lire et à travailler, quelque chose comme ceci :

```handlebars
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
```

Mais si nous faisons cela, notre test échoue. C'est parce que notre test attend qu'une chaîne très spécifique soit renvoyée.

Mais en réalité, nous ne nous soucions pas vraiment des espaces blancs. Maintenir ce test deviendra un cauchemar si nous devons continuer à mettre à jour péniblement la chaîne d'assertion chaque fois que nous apportons des modifications mineures au balisage. À mesure que le template grandit, ce type de modifications devient plus difficile à gérer et les coûts de travail augmenteront en flèche.

## Introduction aux tests d'approbation

[Go Approval Tests](https://github.com/approvals/go-approval-tests)

> ApprovalTests permet de tester facilement des objets plus grands, des chaînes et tout ce qui peut être sauvegardé dans un fichier (images, sons, CSV, etc...)

L'idée est similaire aux fichiers "dorés" ou aux tests de snapshot. Plutôt que de maintenir maladroitement des chaînes dans un fichier de test, l'outil d'approbation peut comparer la sortie pour vous avec un fichier "approuvé" que vous avez créé. Vous copiez simplement la nouvelle version si vous l'approuvez. Relancez le test et vous êtes à nouveau au vert.

Ajoutez une dépendance à `"github.com/approvals/go-approval-tests"` à votre projet et modifiez le test comme suit

```go
func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	t.Run("il convertit un post unique en HTML", func(t *testing.T) {
		buf := bytes.Buffer{}

		if err := blogrenderer.Render(&buf, aPost); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
}
```

La première fois que vous l'exécutez, il échouera parce que nous n'avons encore rien approuvé

```
=== RUN   TestRender
=== RUN   TestRender/it_converts_a_single_post_into_HTML
    renderer_test.go:29: Failed Approval: received does not match approved.
```

Il aura créé deux fichiers, qui ressemblent à ce qui suit

- `renderer_test.TestRender.il_convertit_un_post_unique_en_HTML.received.txt`
- `renderer_test.TestRender.il_convertit_un_post_unique_en_HTML.approved.txt`

Le fichier reçu contient la nouvelle version non approuvée de la sortie. Copiez-la dans le fichier approuvé vide et relancez le test.

En copiant la nouvelle version, vous avez "approuvé" le changement, et le test passe maintenant.

Pour voir le flux de travail en action, modifiez le template comme nous l'avons discuté pour le rendre plus facile à lire (mais sémantiquement, c'est la même chose).

```handlebars
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
```

Relancez le test. Un nouveau fichier "reçu" sera généré car la sortie de notre code diffère de la version approuvée. Jetez-y un coup d'œil, et si vous êtes satisfait des changements, copiez simplement la nouvelle version et relancez le test. Assurez-vous de commiter les fichiers approuvés dans le contrôle de source.

Cette approche rend la gestion des modifications de grandes choses laides comme HTML beaucoup plus simple. Vous pouvez utiliser un outil de diff pour visualiser et gérer les différences, et cela garde votre code de test plus propre.

![Utiliser un outil de diff pour gérer les changements](https://i.imgur.com/0MoNdva.png)

C'est en fait une utilisation assez mineure des tests d'approbation, qui sont un outil extrêmement utile dans votre arsenal de tests. [Emily Bache](https://twitter.com/emilybache) a une [vidéo intéressante où elle utilise des tests d'approbation pour ajouter un ensemble de tests incroyablement étendu à une base de code compliquée qui n'a aucun test](https://www.youtube.com/watch?v=zyM2Ep28ED8). Les "tests combinatoires" sont définitivement quelque chose qui vaut la peine d'être exploré.

Maintenant que nous avons fait ce changement, nous bénéficions toujours d'un code bien testé, mais les tests ne seront pas trop gênants lorsque nous bidouillerons le balisage.

### Faisons-nous toujours du TDD ?

Un effet secondaire intéressant de cette approche est qu'elle nous éloigne du TDD. Bien sûr, vous _pourriez_ modifier manuellement les fichiers approuvés à l'état que vous souhaitez, exécuter vos tests, puis corriger les templates pour qu'ils produisent ce que vous avez défini.

Mais c'est simplement stupide ! Le TDD est une méthode de travail, spécifiquement de conception ; mais cela ne signifie pas que nous devons l'utiliser dogmatiquement pour **tout**.

L'important est que nous avons fait la bonne chose et utilisé le TDD comme un **outil de conception** pour concevoir l'API de notre package. Pour les modifications de templates, notre processus peut être :

- Faire un petit changement au template
- Exécuter le test d'approbation
- Examiner la sortie pour vérifier qu'elle semble correcte
- Faire l'approbation
- Répéter

Nous ne devrions toujours pas abandonner la valeur de travailler par petites étapes réalisables. Essayez de trouver des moyens de rendre les changements petits et continuez à relancer les tests pour obtenir un retour réel sur ce que vous faites.

Si nous commençons à faire des choses comme changer le code _autour_ des templates, alors bien sûr, cela peut justifier de revenir à notre méthode de travail TDD.

## Développer le balisage

La plupart des sites web ont un HTML plus riche que ce que nous avons maintenant. Pour commencer, un élément `html`, avec un `head`, peut-être aussi un `nav`. Habituellement, il y a aussi l'idée d'un pied de page.

Si notre site va avoir différentes pages, nous voudrions définir ces choses à un seul endroit pour garder notre site avec une apparence cohérente. Les templates Go nous permettent de définir des sections que nous pouvons ensuite importer dans d'autres templates.

Modifiez notre template existant pour importer un template supérieur et inférieur

```handlebars
{{template "top" .}}
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
{{template "bottom" .}}
```

Puis créez `top.gohtml` avec ce qui suit

```handlebars
{{define "top"}}
<!DOCTYPE html>
<html lang="fr">
<head>
    <title>Mon blog incroyable !</title>
    <meta charset="UTF-8"/>
    <meta name="description" content="Wow, likez et abonnez-vous, ça aide vraiment la chaîne les gars" lang="fr"/>
</head>
<body>
<nav role="navigation">
    <div>
        <h1>Blog du Gopher en herbe</h1>
        <ul>
            <li><a href="/">accueil</a></li>
            <li><a href="about">à propos</a></li>
            <li><a href="archive">archives</a></li>
        </ul>
    </div>
</nav>
<main>
{{end}}
```

Et `bottom.gohtml`

```handlebars
{{define "bottom"}}
</main>
<footer>
    <ul>
        <li><a href="https://twitter.com/quii">Twitter</a></li>
        <li><a href="https://github.com/quii">GitHub</a></li>
    </ul>
</footer>
</body>
</html>
{{end}}
```

(Évidemment, n'hésitez pas à mettre le balisage que vous voulez !)

Nous devons maintenant spécifier un template spécifique à exécuter. Dans le renderer de blog, changez la commande `Execute` en `ExecuteTemplate`

```go
if err := templ.ExecuteTemplate(w, "blog.gohtml", p); err != nil {
	return err
}
```

Relancez votre test. Un nouveau fichier "reçu" devrait être créé et le test échouera. Vérifiez-le et si vous êtes satisfait, approuvez-le en le copiant sur l'ancienne version. Relancez le test à nouveau et il devrait passer.

## Une excuse pour jouer avec le benchmarking

Avant de continuer, réfléchissons à ce que fait notre code.

```go
func Render(w io.Writer, p Post) error {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return err
	}

	if err := templ.ExecuteTemplate(w, "blog.gohtml", p); err != nil {
		return err
	}

	return nil
}
```

- Analyser les templates
- Utiliser le template pour rendre un post dans un `io.Writer`

Bien que l'impact sur les performances de la réanalyse des templates pour chaque post dans la plupart des cas soit assez négligeable, l'effort pour *ne pas* le faire est également assez négligeable et devrait également un peu nettoyer le code.

Pour voir l'impact de ne pas faire cette analyse encore et encore, nous pouvons utiliser l'outil de benchmarking pour voir à quelle vitesse notre fonction s'exécute.

```go
func BenchmarkRender(b *testing.B) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	for b.Loop() {
		blogrenderer.Render(io.Discard, aPost)
	}
}
```

Sur mon ordinateur, voici les résultats

```
BenchmarkRender-8 22124 53812 ns/op
```

Pour éviter d'avoir à réanalyser les templates encore et encore, nous allons créer un type qui contiendra le template analysé et qui aura une méthode pour effectuer le rendu

```go
type PostRenderer struct {
	templ *template.Template
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	return &PostRenderer{templ: templ}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {

	if err := r.templ.ExecuteTemplate(w, "blog.gohtml", p); err != nil {
		return err
	}

	return nil
}
```

Cela change l'interface de notre code, nous devrons donc mettre à jour notre test

```go
func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	postRenderer, err := blogrenderer.NewPostRenderer()

	if err != nil {
		t.Fatal(err)
	}

	t.Run("il convertit un post unique en HTML", func(t *testing.T) {
		buf := bytes.Buffer{}

		if err := postRenderer.Render(&buf, aPost); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
}
```

Et notre benchmark

```go
func BenchmarkRender(b *testing.B) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	postRenderer, err := blogrenderer.NewPostRenderer()

	if err != nil {
		b.Fatal(err)
	}

	for b.Loop() {
		postRenderer.Render(io.Discard, aPost)
	}
}
```

Le test devrait continuer à passer. Et notre benchmark ?

`BenchmarkRender-8 362124 3131 ns/op`. Les anciennes NS par opération étaient de `53812 ns/op`, c'est donc une amélioration décente ! Au fur et à mesure que nous ajoutons d'autres méthodes de rendu, par exemple une page d'index, cela devrait simplifier le code car nous n'avons pas besoin de dupliquer l'analyse du template.

## Retour au travail réel

En termes de rendu des posts, la partie importante restante est en fait le rendu du `Body`. Si vous vous souvenez, ce devrait être du markdown que l'auteur a écrit, il faudra donc le convertir en HTML.

Nous vous laissons cela comme exercice à vous, le lecteur. Vous devriez pouvoir trouver une bibliothèque Go pour le faire pour vous. Utilisez le test d'approbation pour valider ce que vous faites.

### Sur les tests des bibliothèques tierces

**Note**. Attention à ne pas trop vous soucier de tester explicitement comment une bibliothèque tierce se comporte dans les tests unitaires.

Écrire des tests contre du code que vous ne contrôlez pas est inutile et ajoute une charge de maintenance. Parfois, vous souhaiterez peut-être utiliser [l'injection de dépendance](./dependency-injection.md) pour contrôler une dépendance et simuler son comportement pour un test.

Dans ce cas cependant, je considère la conversion du markdown en HTML comme un détail d'implémentation du rendu, et nos tests d'approbation devraient nous donner suffisamment de confiance.

### Rendu de l'index

La prochaine fonctionnalité que nous allons réaliser est le rendu d'un index, listant les posts sous forme de liste ordonnée HTML.

Nous élargissons notre API, alors nous allons remettre notre chapeau TDD.

## Écrire le test d'abord

À première vue, une page d'index semble simple, mais l'écriture du test nous pousse toujours à faire quelques choix de conception

```go
t.Run("il rend un index de posts", func(t *testing.T) {
	buf := bytes.Buffer{}
	posts := []blogrenderer.Post{{Title: "Hello World"}, {Title: "Hello World 2"}}

	if err := postRenderer.RenderIndex(&buf, posts); err != nil {
		t.Fatal(err)
	}

	got := buf.String()
	want := `<ol><li><a href="/post/hello-world">Hello World</a></li><li><a href="/post/hello-world-2">Hello World 2</a></li></ol>`

	if got != want {
		t.Errorf("obtenu %q attendu %q", got, want)
	}
})
```

1. Nous utilisons le champ title du `Post` comme une partie du chemin de l'URL, mais nous ne voulons pas vraiment d'espaces dans l'URL, donc nous les remplaçons par des tirets.
2. Nous avons ajouté une méthode `RenderIndex` à notre `PostRenderer` qui prend à nouveau un `io.Writer` et une slice de `Post`.

Si nous nous en étions tenus à une approche de test après coup, les tests d'approbation ici ne répondraient pas à ces questions dans un environnement contrôlé. **Les tests nous donnent un espace pour réfléchir**.

## Essayer d'exécuter le test

```
./renderer_test.go:41:13: undefined: blogrenderer.RenderIndex
```

## Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test échouant

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return nil
}
```

Ce qui précède devrait obtenir l'échec de test suivant

```
=== RUN   TestRender
=== RUN   TestRender/it_renders_an_index_of_posts
    renderer_test.go:49: got "" want "<ol><li><a href=\"/post/hello-world\">Hello World</a></li><li><a href=\"/post/hello-world-2\">Hello World 2</a></li></ol>"
---- FAIL: TestRender (0.00s)
```

## Écrire suffisamment de code pour le faire passer

Même si cela _semble_ facile, c'est un peu gênant. Je l'ai fait en plusieurs étapes

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{.Title}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

Je ne voulais pas m'embêter avec des fichiers de template séparés au début, je voulais juste le faire fonctionner. Je considère l'analyse initiale du template et la séparation comme un refactoring que je peux faire plus tard.

Cela ne passe pas, mais c'est proche.

```
=== RUN   TestRender
=== RUN   TestRender/it_renders_an_index_of_posts
    renderer_test.go:49: got "<ol><li><a href=\"/post/Hello%20World\">Hello World</a></li><li><a href=\"/post/Hello%20World%202\">Hello World 2</a></li></ol>" want "<ol><li><a href=\"/post/hello-world\">Hello World</a></li><li><a href=\"/post/hello-world-2\">Hello World 2</a></li></ol>"
---- FAIL: TestRender (0.00s)
    --- FAIL: TestRender/it_renders_an_index_of_posts (0.00s)
```

Vous pouvez voir que le code de templating échappe aux espaces dans les attributs `href`. Nous avons besoin d'un moyen de remplacer les espaces par des tirets. Nous ne pouvons pas simplement parcourir le `[]Post` et les remplacer en mémoire car nous voulons toujours que les espaces soient affichés à l'utilisateur dans les ancres.

Nous avons quelques options. La première que nous allons explorer est de passer une fonction dans notre template.

### Passer des fonctions dans les templates

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{sanitiseTitle .Title}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Funcs(template.FuncMap{
		"sanitiseTitle": func(title string) string {
			return strings.ToLower(strings.Replace(title, " ", "-", -1))
		},
	}).Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

_Avant d'analyser un template_, vous pouvez ajouter un `template.FuncMap` dans votre template, ce qui vous permet de définir des fonctions qui peuvent être appelées dans votre template. Dans ce cas, nous avons créé une fonction `sanitiseTitle` que nous appelons ensuite dans notre template avec `{{sanitiseTitle .Title}}`.

C'est une fonctionnalité puissante, être capable d'envoyer des fonctions dans votre template vous permettra de faire des choses très cool, mais, devriez-vous ? En revenant aux principes de Mustache et des templates sans logique, pourquoi préconisaient-ils l'absence de logique ? **Qu'est-ce qui ne va pas avec la logique dans les templates ?**

Comme nous l'avons montré, pour tester nos templates, *nous avons dû introduire un tout autre type de test*.

Imaginez que vous introduisiez une fonction dans un template qui a quelques permutations différentes de comportement et de cas limites, **comment allez-vous la tester** ? Avec cette conception actuelle, votre seul moyen de tester cette logique est de _rendre du HTML et de comparer des chaînes_. Ce n'est pas une façon facile ou saine de tester la logique, et certainement pas ce que vous voudriez pour une logique métier _importante_.

Même si la technique des tests d'approbation a réduit le coût de maintenance de ces tests, ils sont toujours plus coûteux à maintenir que la plupart des tests unitaires que vous écrirez. Ils sont toujours sensibles à toute modification mineure du balisage que vous pourriez faire, c'est juste que nous l'avons rendu plus facile à gérer. Nous devrions toujours nous efforcer d'architecturer notre code pour ne pas avoir à écrire beaucoup de tests autour de nos templates, et essayer de séparer les préoccupations afin que toute logique qui n'a pas besoin de vivre à l'intérieur de notre code de rendu soit correctement séparée.

Ce que les moteurs de templating influencés par Mustache vous donnent, c'est une contrainte utile, n'essayez pas de la contourner trop souvent ; **n'allez pas à contre-courant**. Au lieu de cela, adoptez l'idée des [modèles de vue](https://stackoverflow.com/a/11074506/3193), où vous construisez des types spécifiques qui contiennent les données dont vous avez besoin pour le rendu, d'une manière qui est pratique pour le langage de templating.

De cette façon, quelle que soit la logique métier importante que vous utilisez pour générer ce sac de données, elle peut être testée unitairement séparément, loin du monde désordonné du HTML et du templating.

### Séparation des préoccupations

Alors que pourrions-nous faire à la place ?

#### Ajouter une méthode à `Post` et l'appeler dans le template

Nous pouvons appeler des méthodes dans notre code de templating sur les types que nous envoyons, nous pourrions donc ajouter une méthode `SanitisedTitle` à `Post`. Cela simplifierait le template et nous pourrions facilement tester unitairement cette logique séparément si nous le souhaitons. C'est probablement la solution la plus simple, bien que pas nécessairement la plus simple.

Un inconvénient de cette approche est que c'est toujours une logique de _vue_. Ce n'est pas intéressant pour le reste du système, mais cela devient une partie de l'API pour un objet de domaine principal. Ce type d'approche peut, au fil du temps, vous amener à créer des [God Objects (Objets omniscients)](https://en.wikipedia.org/wiki/God_object).

#### Créer un type de modèle de vue dédié, tel que `PostViewModel` avec exactement les données dont nous avons besoin

Plutôt que notre code de rendu soit couplé à l'objet de domaine, `Post`, il prend à la place un modèle de vue.

```go
type PostViewModel struct {
	Title, SanitisedTitle, Description, Body string
	Tags                                     []string
}
```

Les appelants de notre code devraient mapper de `[]Post` à `[]PostView`, générant le `SanitizedTitle`. Une façon de garder cela propre serait d'avoir un `func NewPostView(p Post) PostView` qui encapsulerait le mappage.

Cela garderait notre code de rendu sans logique et est probablement la séparation des préoccupations la plus stricte que nous pourrions faire, mais le compromis est un processus légèrement plus compliqué pour obtenir le rendu de nos posts.

Les deux options sont bonnes, dans ce cas, je suis tenté d'opter pour la première. Au fur et à mesure que vous faites évoluer le système, vous devriez vous méfier d'ajouter de plus en plus de méthodes ad hoc juste pour graisser les rouages du rendu ; des modèles de vue dédiés deviennent plus utiles lorsque la transformation entre l'objet de domaine et la vue devient plus impliquée.

Nous pouvons donc ajouter notre méthode à `Post`

```go
func (p Post) SanitisedTitle() string {
	return strings.ToLower(strings.Replace(p.Title, " ", "-", -1))
}
```

Et puis nous pouvons revenir à un monde plus simple dans notre code de rendu

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{.SanitisedTitle}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

## Refactoring

Finalement, le test devrait passer. Nous pouvons maintenant déplacer notre template dans un fichier (`templates/index.gohtml`) et le charger une fois, lors de la construction de notre renderer.

```go
package blogrenderer

import (
	"embed"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

type PostRenderer struct {
	templ *template.Template
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	return &PostRenderer{templ: templ}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {
	return r.templ.ExecuteTemplate(w, "blog.gohtml", p)
}

func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return r.templ.ExecuteTemplate(w, "index.gohtml", posts)
}
```

En analysant plus d'un template dans `templ`, nous devons maintenant appeler `ExecuteTemplate` et spécifier _quel_ template nous souhaitons rendre selon le cas, mais j'espère que vous conviendrez que le code auquel nous sommes arrivés est superbe.

Il y a un _léger_ risque si quelqu'un renomme l'un des fichiers de template, cela introduirait un bug, mais nos tests unitaires rapides à exécuter le détecteraient rapidement.

Maintenant que nous sommes satisfaits de la conception de l'API de notre package et que nous avons développé quelques comportements de base avec TDD, changeons notre test pour utiliser les approbations.

```go
	t.Run("il rend un index de posts", func(t *testing.T) {
		buf := bytes.Buffer{}
		posts := []blogrenderer.Post{{Title: "Hello World"}, {Title: "Hello World 2"}}

		if err := postRenderer.RenderIndex(&buf, posts); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
```

N'oubliez pas d'exécuter le test pour voir qu'il échoue, puis d'approuver le changement.

Enfin, nous pouvons ajouter notre mobilier de page à notre page d'index :

```handlebars
{{template "top" .}}
<ol>{{range .}}<li><a href="/post/{{.SanitisedTitle}}">{{.Title}}</a></li>{{end}}</ol>
{{template "bottom" .}}
```

Relancez le test, approuvez le changement et nous en avons fini avec l'index !

## Rendu du corps en markdown

Je vous ai encouragé à l'essayer vous-même, voici l'approche que j'ai fini par adopter.

```go
package blogrenderer

import (
	"embed"
	"github.com/gomarkdown/markdown"
	"github.com/gomarkdown/markdown/parser"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

type PostRenderer struct {
	templ    *template.Template
	mdParser *parser.Parser
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	extensions := parser.CommonExtensions | parser.AutoHeadingIDs
	parser := parser.NewWithExtensions(extensions)

	return &PostRenderer{templ: templ, mdParser: parser}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {
	return r.templ.ExecuteTemplate(w, "blog.gohtml", newPostVM(p, r))
}

func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return r.templ.ExecuteTemplate(w, "index.gohtml", posts)
}

type postViewModel struct {
	Post
	HTMLBody template.HTML
}

func newPostVM(p Post, r *PostRenderer) postViewModel {
	vm := postViewModel{Post: p}
	vm.HTMLBody = template.HTML(markdown.ToHTML([]byte(p.Body), r.mdParser, nil))
	return vm
}
```

J'ai utilisé l'excellente bibliothèque [gomarkdown](https://github.com/gomarkdown/markdown) qui a fonctionné exactement comme je l'espérais.

Si vous avez essayé de le faire vous-même, vous avez peut-être constaté que votre rendu de corps avait le HTML échappé. C'est une fonctionnalité de sécurité du package html/template de Go pour empêcher la sortie de HTML malveillant provenant de tiers.

Pour contourner cela, dans le type que vous envoyez au rendu, vous devrez envelopper votre HTML de confiance dans [template.HTML](https://pkg.go.dev/html/template#HTML)

> HTML encapsule un fragment de document HTML sûr connu. Il ne doit pas être utilisé pour du HTML provenant d'un tiers, ou du HTML avec des balises ou des commentaires non fermés. Les sorties d'un assainisseur HTML solide et d'un modèle échappé par ce package conviennent à l'utilisation avec HTML.
>
> L'utilisation de ce type présente un risque de sécurité : le contenu encapsulé doit provenir d'une source fiable, car il sera inclus textuellement dans la sortie du modèle.

J'ai donc créé un modèle de vue **non exporté** (`postViewModel`), car je considérais toujours cela comme un détail d'implémentation interne au rendu. Je n'ai pas besoin de le tester séparément et je ne veux pas qu'il pollue mon API.

J'en construis un lors du rendu afin de pouvoir analyser le `Body` en `HTMLBody` puis j'utilise ce champ dans le template pour rendre le HTML.

## Conclusion

Si vous combinez vos apprentissages du chapitre [lecture de fichiers](reading-files.md) et celui-ci, vous pouvez confortablement créer un générateur de site statique simple et bien testé et lancer votre propre blog. Trouvez quelques tutoriels CSS et vous pourrez aussi le rendre beau.

Cette approche s'étend au-delà des blogs. Prendre des données de n'importe quelle source, que ce soit une base de données, une API ou un système de fichiers et les convertir en HTML et les renvoyer depuis un serveur est une technique simple qui s'étend sur de nombreuses décennies. Les gens aiment se lamenter sur la complexité du développement web moderne, mais êtes-vous sûr que vous ne vous infligez pas la complexité à vous-même ?

Go est merveilleux pour le développement web, surtout lorsque vous réfléchissez clairement à vos véritables exigences pour le site web que vous créez. Générer du HTML sur le serveur est souvent une approche meilleure, plus simple et plus performante que de créer une "application web" avec des technologies comme React.

### Ce que nous avons appris

- Comment créer et rendre des templates HTML.
- Comment composer les templates ensemble et [DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself) le balisage associé et nous aider à maintenir une apparence cohérente.
- Comment passer des fonctions dans les templates, et pourquoi vous devriez y réfléchir à deux fois.
- Comment écrire des "Tests d'approbation", qui nous aident à tester la sortie grosse et laide de choses comme les renderers de templates.

### Sur les templates sans logique

Comme toujours, tout est une question de **séparation des préoccupations**. Il est important que nous considérions quelles sont les responsabilités des différentes parties de notre système. Trop souvent, les gens laissent fuir une logique métier importante dans les templates, mélangeant les préoccupations et rendant les systèmes difficiles à comprendre, à maintenir et à tester.

### Pas seulement pour HTML

N'oubliez pas que go a `text/template` pour générer d'autres types de données à partir d'un template. Si vous avez besoin de transformer des données en un type de sortie structuré, les techniques exposées dans ce chapitre peuvent être utiles.

### Références et matériel supplémentaire

- [John Calhoun's 'Learn Web Development with Go'](https://www.calhoun.io/intro-to-templates-p1-contextual-encoding/) contient un certain nombre d'excellents articles sur le templating.
- [Hotwire](https://hotwired.dev) - Vous pouvez utiliser ces techniques pour créer des applications web Hotwire. Il a été construit par Basecamp qui sont principalement une boutique Ruby on Rails, mais comme il est côté serveur, nous pouvons l'utiliser avec Go.
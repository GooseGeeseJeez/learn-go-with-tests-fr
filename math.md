# Mathématiques

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/math)**

Malgré toute la puissance des ordinateurs modernes pour effectuer d'énormes calculs à une vitesse fulgurante, le développeur moyen utilise rarement les mathématiques dans son travail. Mais pas aujourd'hui ! Aujourd'hui, nous allons utiliser les mathématiques pour résoudre un _vrai_ problème. Et pas des mathématiques ennuyeuses - nous allons utiliser la trigonométrie, les vecteurs et toutes sortes de choses dont vous avez toujours dit que vous n'auriez jamais à utiliser après le lycée.

## Le Problème

Vous voulez créer un SVG d'une horloge. Pas une horloge numérique - non, ce serait trop facile - une horloge _analogique_, avec des aiguilles. Vous ne cherchez rien de sophistiqué, juste une belle fonction qui prend un `Time` du package `time` et qui produit un SVG d'une horloge avec toutes les aiguilles - heure, minute et seconde - pointant dans la bonne direction. À quel point cela peut-il être difficile ?

D'abord, nous allons avoir besoin d'un SVG d'une horloge pour jouer avec. Les SVG sont un format d'image fantastique à manipuler par programmation car ils sont écrits comme une série de formes, décrites en XML. Donc cette horloge :

![un svg d'une horloge](math/example_clock.svg)

est décrite comme ceci :

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg"
     width="100%"
     height="100%"
     viewBox="0 0 300 300"
     version="2.0">

  <!-- lunette -->
  <circle cx="150" cy="150" r="100" style="fill:#fff;stroke:#000;stroke-width:5px;"/>

  <!-- aiguille des heures -->
  <line x1="150" y1="150" x2="114.150000" y2="132.260000"
        style="fill:none;stroke:#000;stroke-width:7px;"/>

  <!-- aiguille des minutes -->
  <line x1="150" y1="150" x2="101.290000" y2="99.730000"
        style="fill:none;stroke:#000;stroke-width:7px;"/>

  <!-- aiguille des secondes -->
  <line x1="150" y1="150" x2="77.190000" y2="202.900000"
        style="fill:none;stroke:#f00;stroke-width:3px;"/>
</svg>
```

C'est un cercle avec trois lignes, chacune des lignes commençant au milieu du cercle (x=150, y=150), et se terminant à une certaine distance.

Donc ce que nous allons faire, c'est reconstruire ce qui précède d'une manière ou d'une autre, mais changer les lignes pour qu'elles pointent dans les directions appropriées pour un temps donné.

## Un Test d'Acceptation

Avant de trop nous plonger dans le problème, réfléchissons à un test d'acceptation (*acceptance test*).

Attendez, vous ne savez pas encore ce qu'est un test d'acceptation. Laissez-moi essayer de l'expliquer.

Laissez-moi vous demander : à quoi ressemble la victoire ? Comment savons-nous que nous avons terminé le travail ? Le TDD fournit un bon moyen de savoir quand vous avez terminé : quand le test passe. Parfois, c'est agréable - en fait, presque tout le temps c'est agréable - d'écrire un test qui vous indique quand vous avez terminé d'écrire l'ensemble de la fonctionnalité utilisable. Pas seulement un test qui vous indique qu'une fonction particulière fonctionne de la manière attendue, mais un test qui vous indique que l'ensemble de ce que vous essayez d'accomplir - la "fonctionnalité" - est complet.

Ces tests sont parfois appelés "tests d'acceptation", parfois appelés "tests de fonctionnalité". L'idée est que vous écrivez un test de très haut niveau pour décrire ce que vous essayez d'accomplir - un utilisateur clique sur un bouton sur un site Web, et il voit une liste complète des Pokémon qu'il a attrapés, par exemple. Une fois que nous avons écrit ce test, nous pouvons ensuite écrire plus de tests - des tests unitaires - qui construisent vers un système fonctionnel qui passera le test d'acceptation. Donc pour notre exemple, ces tests pourraient concerner le rendu d'une page Web avec un bouton, le test des gestionnaires de routes sur un serveur Web, l'exécution de recherches dans une base de données, etc. Toutes ces choses seront faites avec le TDD, et toutes contribueront à faire passer le test d'acceptation original.

Quelque chose comme cette image _classique_ de Nat Pryce et Steve Freeman

![Boucles de feedback Outside-in dans le TDD](TDD-outside-in.jpg)

Quoi qu'il en soit, essayons d'écrire ce test d'acceptation - celui qui nous fera savoir quand nous aurons terminé.

Nous avons un exemple d'horloge, alors réfléchissons aux paramètres importants.


```
<line x1="150" y1="150" x2="114.150000" y2="132.260000"
        style="fill:none;stroke:#000;stroke-width:7px;"/>
```

Le centre de l'horloge (les attributs `x1` et `y1` pour cette ligne) est le même pour chaque aiguille de l'horloge. Les nombres qui doivent changer pour chaque aiguille de l'horloge - les paramètres de ce qui construit le SVG - sont les attributs `x2` et `y2`. Nous aurons besoin d'un X et d'un Y pour chacune des aiguilles de l'horloge.

Je _pourrais_ penser à plus de paramètres - le rayon du cercle de l'horloge, la taille du SVG, les couleurs des aiguilles, leur forme, etc... mais il est préférable de commencer par résoudre un problème simple et concret avec une solution simple et concrète, puis de commencer à ajouter des paramètres pour le généraliser.

Donc nous dirons que
- chaque horloge a un centre de (150, 150)
- l'aiguille des heures a une longueur de 50
- l'aiguille des minutes a une longueur de 80
- l'aiguille des secondes a une longueur de 90.

Une chose à noter à propos des SVG : l'origine - le point (0,0) - est dans le coin _supérieur gauche_, pas dans le coin _inférieur gauche_ comme on pourrait s'y attendre. Il sera important de s'en souvenir lorsque nous déterminerons quels nombres utiliser dans nos lignes.

Enfin, je ne décide pas _comment_ construire le SVG - nous pourrions utiliser un modèle du package `text/template`, ou nous pourrions simplement envoyer des octets dans un `bytes.Buffer` ou un writer. Mais nous savons que nous aurons besoin de ces nombres, alors concentrons-nous sur le test de quelque chose qui les crée.

### Écrire le test en premier

Donc mon premier test ressemble à ceci :

```go
package clockface_test

import (
	"projectpath/clockface"
	"testing"
	"time"
)

func TestSecondHandAtMidnight(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 0, 0, time.UTC)

	want := clockface.Point{X: 150, Y: 150 - 90}
	got := clockface.SecondHand(tm)

	if got != want {
		t.Errorf("Got %v, wanted %v", got, want)
	}
}
```

Vous souvenez-vous comment les SVG tracent leurs coordonnées à partir du coin supérieur gauche ? Pour placer l'aiguille des secondes à minuit, nous nous attendons à ce qu'elle n'ait pas bougé du centre du cadran sur l'axe X - toujours 150 - et l'axe Y est la longueur de l'aiguille 'vers le haut' depuis le centre ; 150 moins 90.

### Essayer d'exécuter le test

Cela fait ressortir les échecs attendus concernant les fonctions et types manquants :

```
--- FAIL: TestSecondHandAtMidnight (0.00s)
./clockface_test.go:13:10: undefined: clockface.Point
./clockface_test.go:14:9: undefined: clockface.SecondHand
```

Donc un `Point` où la pointe de l'aiguille des secondes devrait aller, et une fonction pour l'obtenir.

### Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test qui échoue

Implémentons ces types pour que le code compile

```go
package clockface

import "time"

// A Point représente une coordonnée cartésienne bidimensionnelle
type Point struct {
	X float64
	Y float64
}

// SecondHand est le vecteur unitaire de l'aiguille des secondes d'une horloge analogique à l'heure `t`
// représenté comme un Point.
func SecondHand(t time.Time) Point {
	return Point{}
}
```

et maintenant nous obtenons :

```
--- FAIL: TestSecondHandAtMidnight (0.00s)
    clockface_test.go:17: Got {0 0}, wanted {150 60}
FAIL
exit status 1
FAIL	learn-go-with-tests/math/clockface	0.006s
```

### Écrire suffisamment de code pour le faire passer

Maintenant, écrivons le code nécessaire pour faire passer notre test de minuit :

```go
func SecondHand(t time.Time) Point {
	return Point{150, 60}
}
```

Et le test passe !

```
PASS
ok  	learn-go-with-tests/math/clockface	0.006s
```

### Refactoriser

Il n'y a rien à refactoriser pour l'instant... mais attendez une minute. 

Notre test attend que l'aiguille des secondes pointe vers le haut à minuit. Mais est-ce vraiment ce que fait une vraie horloge ? Maintenant que j'y pense, à minuit (et à midi), l'aiguille des secondes pointe vers le 12, c'est-à-dire vers le haut.

Donc notre test et notre implémentation sont corrects. Mais je pense que nous pouvons rendre notre test un peu plus expressif :

```go
func TestSecondHandAtMidnight(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 0, 0, time.UTC)

	want := clockface.Point{X: 150, Y: 60}
	got := clockface.SecondHand(tm)

	if got != want {
		t.Errorf("Got %v, wanted %v", got, want)
	}
}
```

Nous avons supprimé le calcul `150 - 90` du test et utilisé directement la valeur attendue. Mais maintenant le test est moins explicite sur la raison pour laquelle nous attendons ces valeurs particulières. Nous pourrions ajouter un commentaire, mais nous réfléchirons à une meilleure façon de communiquer cela plus tard.

## Plus de tests

Ajoutons quelques tests supplémentaires pour d'autres positions de l'aiguille des secondes.

```go
func TestSecondHandAt30Seconds(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 30, 0, time.UTC)

	want := clockface.Point{X: 150 + 90, Y: 150}
	got := clockface.SecondHand(tm)

	if got != want {
		t.Errorf("Got %v, wanted %v", got, want)
	}
}
```

À 30 secondes, l'aiguille devrait pointer vers la droite, donc X augmente de la longueur de l'aiguille.

Pour le moment, notre test échoue :

```
--- FAIL: TestSecondHandAt30Seconds (0.00s)
    clockface_test.go:27: Got {150 60}, wanted {240 150}
```

Faisons en sorte que notre fonction gère ce cas également, même si c'est de manière simpliste pour l'instant :

```go
func SecondHand(t time.Time) Point {
	second := t.Second()
	
	// à 0 seconde, pointer vers le haut
	if second == 0 {
		return Point{150, 60}
	}
	
	// à 30 secondes, pointer vers la droite
	return Point{240, 150}
}
```

Et voilà ! Mais ce n'est clairement pas une solution évolutive. Nous allons devoir trouver une formule mathématique pour calculer la position de l'aiguille pour n'importe quelle seconde.

## Penser comme une horloge

Une horloge analogique est essentiellement un cercle divisé en 60 segments égaux pour les secondes. L'aiguille des secondes fait un tour complet en 60 secondes, donc elle se déplace de 360 degrés en 60 secondes, ce qui signifie 6 degrés par seconde.

Si nous savons à quel angle doit être l'aiguille, nous pouvons utiliser des fonctions trigonométriques pour calculer les coordonnées X et Y de l'extrémité de l'aiguille.

Ajoutons d'abord un test pour la conversion des secondes en angle en radians (les fonctions trigonométriques en Go utilisent des radians, pas des degrés) :

```go
func TestSecondsInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(0, 0, 30), math.Pi},
		{simpleTime(0, 0, 0), 0},
		{simpleTime(0, 0, 45), (math.Pi / 2) * 3},
		{simpleTime(0, 0, 7), (math.Pi / 30) * 7},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := clockface.SecondsInRadians(c.time)
			if !roughlyEqualFloat64(got, c.angle) {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}

func simpleTime(hours, minutes, seconds int) time.Time {
	return time.Date(312, time.October, 28, hours, minutes, seconds, 0, time.UTC)
}

func testName(t time.Time) string {
	return t.Format("15:04:05")
}

func roughlyEqualFloat64(a, b float64) bool {
	const equalityThreshold = 1e-7
	return math.Abs(a-b) < equalityThreshold
}
```

Nous avons créé une fonction d'aide `simpleTime` pour simplifier la création de valeurs `time.Time` pour nos tests, et une fonction `roughlyEqualFloat64` pour comparer des nombres à virgule flottante avec une tolérance, car les comparaisons exactes de nombres à virgule flottante peuvent être problématiques en raison d'erreurs d'arrondi.

Maintenant, implémentons la fonction `SecondsInRadians` :

```go
// SecondsInRadians retourne la position des secondes de l'heure en radians
func SecondsInRadians(t time.Time) float64 {
	return float64(t.Second()) * (math.Pi / 30)
}
```

Chaque seconde correspond à 6 degrés, ce qui équivaut à (math.Pi / 30) radians.

Maintenant, utilisons cette fonction pour calculer la position de l'aiguille des secondes :

```go
// SecondHand est le vecteur unitaire de l'aiguille des secondes d'une horloge analogique à l'heure `t`
// représenté comme un Point.
func SecondHand(t time.Time) Point {
	angle := SecondsInRadians(t)
	
	x := math.Sin(angle)
	y := math.Cos(angle)
	
	// Les coordonnées sont par rapport au centre de l'horloge (150, 150)
	// et nous utilisons une aiguille de 90 unités de long
	return Point{
		X: 150 + x*90,
		Y: 150 - y*90,
	}
}
```

Notez que nous soustrayons pour Y car dans un SVG, l'axe Y va du haut vers le bas.

Et voilà ! Notre horloge peut maintenant afficher l'aiguille des secondes à n'importe quelle position.

## Ajouter les aiguilles des minutes et des heures

Nous pouvons suivre la même approche pour les aiguilles des minutes et des heures. La seule différence est la période de rotation et la longueur des aiguilles.

- L'aiguille des minutes fait un tour complet en 60 minutes, donc elle se déplace de 6 degrés par minute.
- L'aiguille des heures fait un tour complet en 12 heures, donc elle se déplace de 30 degrés par heure. Mais elle se déplace également légèrement en fonction des minutes écoulées.

Implémentons ces fonctions en suivant la même approche que pour l'aiguille des secondes.

## Conclusion

Dans ce chapitre, nous avons utilisé nos connaissances en mathématiques pour résoudre un problème concret : dessiner une horloge analogique avec des aiguilles pointant dans les bonnes directions. Nous avons utilisé la trigonométrie pour calculer les positions des aiguilles sur un cercle.

Cette approche montre comment le TDD peut être utilisé même pour des problèmes qui semblent nécessiter une compréhension mathématique. Nous avons commencé par des tests simples et concrets, puis nous avons progressivement développé notre solution.

N'ayez pas peur d'utiliser les mathématiques dans votre code lorsque c'est approprié. Les bibliothèques standard de Go, comme le package `math`, offrent de nombreuses fonctions qui facilitent les calculs complexes.
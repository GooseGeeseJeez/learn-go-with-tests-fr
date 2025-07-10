# Chiffres Romains

**[Vous pouvez trouver tout le code de ce chapitre ici](https://github.com/quii/learn-go-with-tests/tree/main/roman-numerals)**

Certaines entreprises vous demanderont de réaliser le [Kata des Chiffres Romains](http://codingdojo.org/kata/RomanNumerals/) dans le cadre du processus d'entretien. Ce chapitre vous montrera comment l'aborder avec le TDD.

Nous allons écrire une fonction qui convertit un [nombre arabe](https://fr.wikipedia.org/wiki/Chiffres_arabes) (chiffres 0 à 9) en un chiffre romain.

Si vous n'avez jamais entendu parler des [chiffres romains](https://fr.wikipedia.org/wiki/Num%C3%A9ration_romaine), c'est ainsi que les Romains écrivaient les nombres.

Vous les construisez en collant des symboles ensemble et ces symboles représentent des nombres.

Ainsi, `I` signifie "un". `III` est trois.

Cela semble facile mais il y a quelques règles intéressantes. `V` signifie cinq, mais `IV` est 4 (pas `IIII`).

`MCMLXXXIV` est 1984. Cela semble compliqué et il est difficile d'imaginer comment nous pouvons écrire du code pour comprendre cela dès le départ.

Comme ce livre le souligne, une compétence clé pour les développeurs de logiciels est d'essayer d'identifier des "tranches verticales fines" de fonctionnalités _utiles_ puis **d'itérer**. Le workflow TDD aide à faciliter le développement itératif.

Donc plutôt que 1984, commençons par 1.

## Écrire le test en premier

```go
func TestRomanNumerals(t *testing.T) {
	got := ConvertToRoman(1)
	want := "I"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

Si vous êtes arrivé jusqu'ici dans le livre, cela devrait vous sembler très ennuyeux et routinier. C'est une bonne chose.

## Essayer d'exécuter le test

```console
./numeral_test.go:6:9: undefined: ConvertToRoman
```

Laissez le compilateur vous guider.

## Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test qui échoue

Créons notre fonction mais ne faisons pas passer le test encore, assurons-nous toujours que le test échoue comme prévu.

```go
func ConvertToRoman(arabic int) string {
	return ""
}
```

Le test devrait maintenant s'exécuter

```console
=== RUN   TestRomanNumerals
--- FAIL: TestRomanNumerals (0.00s)
    numeral_test.go:10: got '', want 'I'
FAIL
```

## Écrire suffisamment de code pour le faire passer

```go
func ConvertToRoman(arabic int) string {
	return "I"
}
```

## Refactoriser

Pas grand-chose à refactoriser pour l'instant.

_Je sais_ que ça semble bizarre de simplement coder en dur le résultat, mais avec le TDD, nous voulons rester hors de la "zone rouge" aussi longtemps que possible. Cela peut _sembler_ que nous n'avons pas accompli grand-chose, mais nous avons défini notre API et obtenu un test capturant l'une de nos règles ; même si le code "réel" est assez simpliste.

Maintenant, utilisez ce sentiment d'inconfort pour écrire un nouveau test qui nous force à écrire un code légèrement moins simpliste.

## Écrire le test en premier

Nous pouvons utiliser des sous-tests pour regrouper joliment nos tests.

```go
func TestRomanNumerals(t *testing.T) {
	t.Run("1 est converti en I", func(t *testing.T) {
		got := ConvertToRoman(1)
		want := "I"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})

	t.Run("2 est converti en II", func(t *testing.T) {
		got := ConvertToRoman(2)
		want := "II"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})
}
```

## Essayer d'exécuter le test

```console
=== RUN   TestRomanNumerals/2_est_converti_en_II
    --- FAIL: TestRomanNumerals/2_est_converti_en_II (0.00s)
        numeral_test.go:20: got 'I', want 'II'
```

Pas de grande surprise ici.

## Écrire suffisamment de code pour le faire passer

```go
func ConvertToRoman(arabic int) string {
	if arabic == 2 {
		return "II"
	}
	return "I"
}
```

Oui, on a toujours l'impression de ne pas vraiment aborder le problème. Nous devons donc écrire plus de tests pour avancer.

## Refactoriser

Nous avons des répétitions dans nos tests. Lorsque vous testez quelque chose qui semble être une question de "étant donné l'entrée X, nous attendons Y", vous devriez probablement utiliser des tests basés sur des tableaux.

```go
func TestRomanNumerals(t *testing.T) {
	cases := []struct {
		Description string
		Arabic      int
		Want        string
	}{
		{"1 est converti en I", 1, "I"},
		{"2 est converti en II", 2, "II"},
	}

	for _, test := range cases {
		t.Run(test.Description, func(t *testing.T) {
			got := ConvertToRoman(test.Arabic)
			if got != test.Want {
				t.Errorf("got %q, want %q", got, test.Want)
			}
		})
	}
}
```

Nous pouvons maintenant facilement ajouter d'autres cas sans avoir à écrire plus de code de test.

Continuons et testons pour 3.

## Écrire le test en premier

Ajoutez ce qui suit à nos cas

```
{"3 est converti en III", 3, "III"},
```

## Essayer d'exécuter le test

```console
=== RUN   TestRomanNumerals/3_est_converti_en_III
    --- FAIL: TestRomanNumerals/3_est_converti_en_III (0.00s)
        numeral_test.go:20: got 'I', want 'III'
```

## Écrire suffisamment de code pour le faire passer

```go
func ConvertToRoman(arabic int) string {
	if arabic == 3 {
		return "III"
	}
	if arabic == 2 {
		return "II"
	}
	return "I"
}
```

## Refactoriser

OK, donc je commence à ne plus apprécier ces instructions if, et si vous regardez attentivement le code, vous pouvez voir que nous construisons une chaîne de `I` basée sur la taille de `arabic`.

Nous "savons" que pour des nombres plus complexes, nous ferons une sorte d'arithmétique et de concaténation de chaînes.

Essayons une refactorisation avec ces idées en tête, cela _pourrait ne pas_ convenir à la solution finale, mais c'est OK. Nous pouvons toujours jeter notre code et recommencer avec les tests que nous avons pour nous guider.

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i := 0; i < arabic; i++ {
		result.WriteString("I")
	}

	return result.String()
}
```

Vous vous souvenez peut-être de [`strings.Builder`](https://golang.org/pkg/strings/#Builder) de notre discussion sur [le benchmarking](iteration.md#benchmarking)

> Un Builder est utilisé pour construire efficacement une chaîne à l'aide des méthodes Write. Il minimise la copie de mémoire.

Normalement, je ne me soucierais pas de telles optimisations tant que je n'ai pas un véritable problème de performance, mais la quantité de code n'est pas beaucoup plus grande qu'un ajout "manuel" à une chaîne, donc autant utiliser l'approche la plus rapide.

Le code me semble meilleur et décrit le domaine _tel que nous le connaissons maintenant_.

### Les Romains étaient aussi adeptes du DRY...

Les choses commencent à se compliquer maintenant. Les Romains, dans leur sagesse, pensaient que répéter des caractères deviendrait difficile à lire et à compter. Ainsi, une règle avec les chiffres romains est que vous ne pouvez pas avoir le même caractère répété plus de 3 fois d'affilée.

Au lieu de cela, vous prenez le symbole supérieur suivant puis vous "soustrayez" en mettant un symbole à sa gauche. Tous les symboles ne peuvent pas être utilisés comme soustracteurs ; seulement I (1), X (10) et C (100).

Par exemple, `5` en chiffres romains est `V`. Pour créer 4, vous ne faites pas `IIII`, mais plutôt `IV`.

## Écrire le test en premier

```
{"4 est converti en IV (impossible de répéter plus de 3 fois)", 4, "IV"},
```

## Essayer d'exécuter le test

```console
=== RUN   TestRomanNumerals/4_est_converti_en_IV_(impossible_de_répéter_plus_de_3_fois)
    --- FAIL: TestRomanNumerals/4_est_converti_en_IV_(impossible_de_répéter_plus_de_3_fois) (0.00s)
        numeral_test.go:24: got 'IIII', want 'IV'
```

## Écrire suffisamment de code pour le faire passer

```go
func ConvertToRoman(arabic int) string {

	if arabic == 4 {
		return "IV"
	}

	var result strings.Builder

	for i := 0; i < arabic; i++ {
		result.WriteString("I")
	}

	return result.String()
}
```

## Refactoriser

Je n'aime pas que nous ayons rompu notre modèle de construction de chaînes et je veux continuer avec celui-ci.

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i := arabic; i > 0; i-- {
		if i == 4 {
			result.WriteString("IV")
			break
		}
		result.WriteString("I")
	}

	return result.String()
}
```

Pour que 4 "s'intègre" dans ma pensée actuelle, je compte maintenant à rebours à partir du nombre arabe, en ajoutant des symboles à notre chaîne au fur et à mesure que nous progressons. Je ne suis pas sûr que cela fonctionnera à long terme, mais voyons !

Faisons fonctionner 5.

## Écrire le test en premier

```
{"5 est converti en V", 5, "V"},
```

## Essayer d'exécuter le test

```console
=== RUN   TestRomanNumerals/5_est_converti_en_V
    --- FAIL: TestRomanNumerals/5_est_converti_en_V (0.00s)
        numeral_test.go:25: got 'IIV', want 'V'
```

## Écrire suffisamment de code pour le faire passer

Copions simplement l'approche que nous avons utilisée pour 4.

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i := arabic; i > 0; i-- {
		if i == 5 {
			result.WriteString("V")
			break
		}
		if i == 4 {
			result.WriteString("IV")
			break
		}
		result.WriteString("I")
	}

	return result.String()
}
```

## Refactoriser

Les répétitions dans les boucles comme celle-ci sont généralement le signe d'une abstraction qui attend d'être appelée. Court-circuiter les boucles peut être un outil efficace pour la lisibilité, mais cela pourrait aussi vous indiquer autre chose.

Nous bouclons sur notre nombre arabe et si nous rencontrons certains symboles, nous appelons `break`, mais ce que nous faisons _réellement_ est de soustraire `i` d'une manière maladroite.

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for arabic > 0 {
		switch {
		case arabic > 4:
			result.WriteString("V")
			arabic -= 5
		case arabic > 3:
			result.WriteString("IV")
			arabic -= 4
		default:
			result.WriteString("I")
			arabic--
		}
	}

	return result.String()
}
```

- Étant donné les signaux que je lis dans notre code, issus de nos tests sur des scénarios très basiques, je peux voir que pour construire un chiffre romain, je dois soustraire de `arabic` au fur et à mesure que j'applique des symboles.
- La boucle `for` ne repose plus sur un `i`, mais à la place, nous continuerons à construire notre chaîne jusqu'à ce que nous ayons soustrait suffisamment de symboles de `arabic`.

Je suis plutôt sûr que cette approche sera valide pour 6 (VI), 7 (VII) et 8 (VIII) aussi. Néanmoins, ajoutez les cas à notre suite de tests et vérifiez (je n'inclurai pas le code par souci de brièveté, consultez GitHub pour des exemples si vous n'êtes pas sûr).

9 suit la même règle que 4 en ce sens que nous devrions soustraire `I` de la représentation du nombre suivant. 10 est représenté en chiffres romains par `X` ; donc 9 devrait être `IX`.

## Écrire le test en premier

```
{"9 est converti en IX", 9, "IX"},
```
## Essayer d'exécuter le test

```console
=== RUN   TestRomanNumerals/9_est_converti_en_IX
    --- FAIL: TestRomanNumerals/9_est_converti_en_IX (0.00s)
        numeral_test.go:29: got 'VIV', want 'IX'
```

## Écrire suffisamment de code pour le faire passer

Nous devrions pouvoir adopter la même approche qu'auparavant.

```
case arabic > 8:
    result.WriteString("IX")
    arabic -= 9
```

## Refactoriser

Il _semble_ que le code nous indique toujours qu'il y a une refactorisation quelque part, mais ce n'est pas totalement évident pour moi, alors continuons.

Je vais également passer le code pour cela, mais ajoutez à vos cas de test un test pour `10` qui devrait être `X` et faites-le passer avant de lire la suite.

Voici quelques tests que j'ai ajoutés car je suis confiant que jusqu'à 39, notre code devrait fonctionner.

```
{"10 est converti en X", 10, "X"},
{"14 est converti en XIV", 14, "XIV"},
{"18 est converti en XVIII", 18, "XVIII"},
{"20 est converti en XX", 20, "XX"},
{"39 est converti en XXXIX", 39, "XXXIX"},
```

Si vous avez déjà fait de la programmation OO, vous savez que vous devriez considérer les instructions `switch` avec un peu de méfiance. Généralement, vous capturez un concept ou des données dans du code impératif alors qu'en fait, cela pourrait être capturé dans une structure de classe à la place.

Go n'est pas strictement OO, mais cela ne signifie pas que nous ignorons complètement les leçons que l'OO offre (autant que certains voudraient vous le dire).

Notre instruction switch décrit certaines vérités sur les chiffres romains ainsi que leur comportement.

Nous pouvons refactoriser cela en découplant les données du comportement.

```go
type RomanNumeral struct {
	Value  int
	Symbol string
}

var allRomanNumerals = []RomanNumeral{
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}

func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for _, numeral := range allRomanNumerals {
		for arabic >= numeral.Value {
			result.WriteString(numeral.Symbol)
			arabic -= numeral.Value
		}
	}

	return result.String()
}
```

Cela me semble beaucoup mieux. Nous avons déclaré certaines règles autour des chiffres comme des données plutôt que cachées dans un algorithme, et nous pouvons voir comment nous travaillons simplement à travers le nombre arabe, essayant d'ajouter des symboles à notre résultat s'ils conviennent.

Cette abstraction fonctionne-t-elle pour des nombres plus grands ? Étendez la suite de tests pour qu'elle fonctionne pour le nombre romain de 50, qui est `L`.

Voici quelques cas de test, essayez de les faire passer.

```
{"40 est converti en XL", 40, "XL"},
{"47 est converti en XLVII", 47, "XLVII"},
{"49 est converti en XLIX", 49, "XLIX"},
{"50 est converti en L", 50, "L"},
```

Besoin d'aide ? Vous pouvez voir quels symboles ajouter dans [ce gist](https://gist.github.com/pamelafox/6c7b948213ba55332d86efd0f0b037de).

## Et le reste !

Voici les symboles restants

| Arabe | Romain |
| ------ | :---: |
| 100    |   C   |
| 500    |   D   |
| 1000   |   M   |

Adoptez la même approche pour les symboles restants, il devrait s'agir simplement d'ajouter des données aux tests et à notre tableau de symboles.

Votre code fonctionne-t-il pour `1984` : `MCMLXXXIV` ?

Voici ma suite de tests finale

```go
func TestRomanNumerals(t *testing.T) {
	cases := []struct {
		Arabic int
		Roman  string
	}{
		{Arabic: 1, Roman: "I"},
		{Arabic: 2, Roman: "II"},
		{Arabic: 3, Roman: "III"},
		{Arabic: 4, Roman: "IV"},
		{Arabic: 5, Roman: "V"},
		{Arabic: 6, Roman: "VI"},
		{Arabic: 7, Roman: "VII"},
		{Arabic: 8, Roman: "VIII"},
		{Arabic: 9, Roman: "IX"},
		{Arabic: 10, Roman: "X"},
		{Arabic: 14, Roman: "XIV"},
		{Arabic: 18, Roman: "XVIII"},
		{Arabic: 20, Roman: "XX"},
		{Arabic: 39, Roman: "XXXIX"},
		{Arabic: 40, Roman: "XL"},
		{Arabic: 47, Roman: "XLVII"},
		{Arabic: 49, Roman: "XLIX"},
		{Arabic: 50, Roman: "L"},
		{Arabic: 100, Roman: "C"},
		{Arabic: 90, Roman: "XC"},
		{Arabic: 400, Roman: "CD"},
		{Arabic: 500, Roman: "D"},
		{Arabic: 900, Roman: "CM"},
		{Arabic: 1000, Roman: "M"},
		{Arabic: 1984, Roman: "MCMLXXXIV"},
		{Arabic: 3999, Roman: "MMMCMXCIX"},
		{Arabic: 2014, Roman: "MMXIV"},
		{Arabic: 1006, Roman: "MVI"},
		{Arabic: 798, Roman: "DCCXCVIII"},
	}
	for _, test := range cases {
		t.Run(fmt.Sprintf("%d est converti en %q", test.Arabic, test.Roman), func(t *testing.T) {
			got := ConvertToRoman(test.Arabic)
			if got != test.Roman {
				t.Errorf("got %q, want %q", got, test.Roman)
			}
		})
	}
}
```

- J'ai supprimé `description` car j'ai estimé que les _données_ décrivaient suffisamment l'information.
- J'ai ajouté quelques autres cas limites que j'ai trouvés juste pour me donner un peu plus de confiance. Avec les tests basés sur des tableaux, c'est très peu coûteux.

Je n'ai pas changé l'algorithme, tout ce que j'ai eu à faire était de mettre à jour le tableau `allRomanNumerals`.

```go
var allRomanNumerals = []RomanNumeral{
	{1000, "M"},
	{900, "CM"},
	{500, "D"},
	{400, "CD"},
	{100, "C"},
	{90, "XC"},
	{50, "L"},
	{40, "XL"},
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}
```

## Analyser les chiffres romains

Nous n'avons pas encore terminé. Ensuite, nous allons écrire une fonction qui convertit _depuis_ un chiffre romain vers un `int`.

## Écrire le test en premier

Nous pouvons réutiliser nos cas de test ici avec un peu de refactorisation.

Déplacez la variable `cases` en dehors du test en tant que variable de package dans un bloc `var`.

```go
func TestConvertingToArabic(t *testing.T) {
	for _, test := range cases[:1] {
		t.Run(fmt.Sprintf("%q est converti en %d", test.Roman, test.Arabic), func(t *testing.T) {
			got := ConvertToArabic(test.Roman)
			if got != test.Arabic {
				t.Errorf("got %d, want %d", got, test.Arabic)
			}
		})
	}
}
```

Notez que j'utilise la fonctionnalité de slice pour n'exécuter qu'un seul des tests pour l'instant (`cases[:1]`), car essayer de faire passer tous ces tests en une seule fois est un trop grand pas.

## Essayer d'exécuter le test

```console
./numeral_test.go:60:11: undefined: ConvertToArabic
```

## Écrire le minimum de code pour que le test s'exécute et vérifier la sortie du test qui échoue

Ajoutez notre nouvelle définition de fonction

```go
func ConvertToArabic(roman string) int {
	return 0
}
```

Le test devrait maintenant s'exécuter et échouer

```console
--- FAIL: TestConvertingToArabic (0.00s)
    --- FAIL: TestConvertingToArabic/'I'_est_converti_en_1 (0.00s)
        numeral_test.go:62: got 0, want 1
```

## Écrire suffisamment de code pour le faire passer

Vous savez quoi faire

```go
func ConvertToArabic(roman string) int {
	return 1
}
```

Ensuite, changez l'indice de slice dans notre test pour passer au cas de test suivant (par exemple, `cases[:2]`). Faites-le passer vous-même avec le code le plus simpliste auquel vous pouvez penser, continuez à écrire du code simpliste (meilleur livre jamais, non ?) pour le troisième cas aussi. Voici mon code simpliste.

```go
func ConvertToArabic(roman string) int {
	if roman == "III" {
		return 3
	}
	if roman == "II" {
		return 2
	}
	return 1
}
```

À travers la simplicité du _vrai code qui fonctionne_, nous pouvons commencer à voir un modèle similaire à celui construit précédemment. Nous devons parcourir l'entrée et construire _quelque chose_, dans ce cas un total.

```go
func ConvertToArabic(roman string) int {
	total := 0
	for range roman {
		total++
	}
	return total
}
```

## Écrire le test en premier

Passons maintenant à `cases[:4]` (`IV`), qui échoue maintenant car il obtient 2 en retour, ce qui est la longueur de la chaîne.

## Écrire suffisamment de code pour le faire passer

```go
// plus tôt..
var allRomanNumerals = RomanNumerals{
	{1000, "M"},
	{900, "CM"},
	{500, "D"},
	{400, "CD"},
	{100, "C"},
	{90, "XC"},
	{50, "L"},
	{40, "XL"},
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}

// plus tard..
func ConvertToArabic(roman string) int {
	var arabic = 0

	for _, numeral := range allRomanNumerals {
		for strings.HasPrefix(roman, numeral.Symbol) {
			arabic += numeral.Value
			roman = strings.TrimPrefix(roman, numeral.Symbol)
		}
	}

	return arabic
}
```

C'est essentiellement l'algorithme de `ConvertToRoman(int)` implémenté à l'envers. Ici, nous parcourons la chaîne de chiffres romains donnée :
- Nous recherchons des symboles de chiffres romains pris dans `allRomanNumerals`, du plus élevé au plus bas, au début de la chaîne.
- Si nous trouvons le préfixe, nous ajoutons sa valeur à `arabic` et supprimons le préfixe.

À la fin, nous retournons la somme comme nombre arabe.

La fonction `HasPrefix(s, prefix)` vérifie si la chaîne `s` commence par `prefix` et `TrimPrefix(s, prefix)` supprime le `prefix` de `s`, afin que nous puissions continuer avec les symboles de chiffres romains restants. Cela fonctionne avec `IV` et tous les autres cas de test.

Vous pouvez implémenter cela comme une fonction récursive, qui est plus élégante (à mon avis) mais pourrait être plus lente. Je vous laisse le soin de le faire et de réaliser quelques tests `Benchmark...`.

Maintenant que nous avons nos fonctions pour convertir un nombre arabe en chiffre romain et inversement, nous pouvons pousser nos tests un peu plus loin :

## Une introduction aux tests basés sur les propriétés

Il y a eu quelques règles dans le domaine des chiffres romains avec lesquelles nous avons travaillé dans ce chapitre

- On ne peut pas avoir plus de 3 symboles consécutifs
- Seuls I (1), X (10) et C (100) peuvent être des "soustracteurs"
- Prendre le résultat de `ConvertToRoman(N)` et le passer à `ConvertToArabic` devrait nous retourner `N`

Les tests que nous avons écrits jusqu'à présent peuvent être décrits comme des tests basés sur des "exemples" où nous fournissons des _exemples_ pour que l'outil les vérifie.

Et si nous pouvions prendre ces règles que nous connaissons sur notre domaine et les appliquer d'une manière ou d'une autre à notre code ?

Les tests basés sur les propriétés vous aident à le faire en lançant des données aléatoires à votre code et en vérifiant que les règles que vous décrivez sont toujours vraies. Beaucoup de gens pensent que les tests basés sur les propriétés concernent principalement les données aléatoires, mais ils se tromperaient. Le véritable défi des tests basés sur les propriétés est d'avoir une _bonne_ compréhension de votre domaine afin que vous puissiez écrire ces propriétés.

Assez de mots, voyons un peu de code

```go
func TestPropertiesOfConversion(t *testing.T) {
	assertion := func(arabic int) bool {
		roman := ConvertToRoman(arabic)
		fromRoman := ConvertToArabic(roman)
		return fromRoman == arabic
	}

	if err := quick.Check(assertion, nil); err != nil {
		t.Error("échec des vérifications", err)
	}
}
```

### Logique de la propriété

Notre premier test vérifiera que si nous transformons un nombre en romain, quand nous utilisons notre autre fonction pour le reconvertir en nombre, nous obtenons ce que nous avions au départ.

- Étant donné un nombre aléatoire (par exemple `4`).
- Appeler `ConvertToRoman` avec ce nombre aléatoire (devrait retourner `IV` si `4`).
- Prendre le résultat ci-dessus et le passer à `ConvertToArabic`.
- Ce qui précède devrait nous donner notre entrée d'origine (`4`).

Cela semble être un bon test pour nous donner confiance car il devrait échouer s'il y a un bogue dans l'un ou l'autre. La seule façon dont il pourrait passer, c'est s'ils ont le même type de bogue ; ce qui n'est pas impossible mais semble peu probable.

### Explication technique

 Nous utilisons le package [testing/quick](https://golang.org/pkg/testing/quick/) de la bibliothèque standard.

 En lisant à partir du bas, nous fournissons à `quick.Check` une fonction qu'il exécutera sur un certain nombre d'entrées aléatoires, si la fonction retourne `false`, cela sera considéré comme un échec de la vérification.

 Notre fonction `assertion` ci-dessus prend un nombre aléatoire et exécute nos fonctions pour tester la propriété.

### Exécutez notre test

 Essayez de l'exécuter ; votre ordinateur peut se bloquer pendant un moment, alors tuez-le quand vous vous ennuyez :)

 Que se passe-t-il ? Essayez d'ajouter ce qui suit au code d'assertion.

 ```go
assertion := func(arabic int) bool {
	if arabic < 0 || arabic > 3999 {
		log.Println(arabic)
		return true
	}
	roman := ConvertToRoman(arabic)
	fromRoman := ConvertToArabic(roman)
	return fromRoman == arabic
}
```

Vous devriez voir quelque chose comme ceci :

```console
=== RUN   TestPropertiesOfConversion
2019/07/09 14:41:27 6849766357708982977
2019/07/09 14:41:27 -7028152357875163913
2019/07/09 14:41:27 -6752532134903680693
2019/07/09 14:41:27 4051793897228170080
2019/07/09 14:41:27 -1111868396280600429
2019/07/09 14:41:27 8851967058300421387
2019/07/09 14:41:27 562755830018219185
```

Simplement exécuter cette propriété très simple a révélé une faille dans notre implémentation. Nous avons utilisé `int` comme entrée mais :
- Vous ne pouvez pas faire de nombres négatifs avec les chiffres romains
- Étant donné notre règle d'un maximum de 3 symboles consécutifs, nous ne pouvons pas représenter une valeur supérieure à 3999 ([enfin, plus ou moins](https://www.quora.com/Which-is-the-maximum-number-in-Roman-numerals)) et `int` a une valeur maximale beaucoup plus élevée que 3999.

C'est génial ! Nous avons été forcés de réfléchir plus profondément à notre domaine, ce qui est une véritable force des tests basés sur les propriétés.

Clairement, `int` n'est pas un excellent type pour cette tâche. Et si nous essayions quelque chose d'un peu plus approprié ?

### [`uint16`](https://golang.org/pkg/builtin/#uint16)

Go possède des types pour les _entiers non signés_, ce qui signifie qu'ils ne peuvent pas être négatifs ; cela élimine donc immédiatement une classe de bugs dans notre code. En ajoutant 16, cela signifie qu'il s'agit d'un entier 16 bits qui peut stocker un maximum de `65535`, ce qui est encore trop grand mais nous rapproche de ce dont nous avons besoin.

Essayez de mettre à jour le code pour utiliser `uint16` plutôt que `int`. J'ai mis à jour `assertion` dans le test pour donner un peu plus de visibilité.

```go
assertion := func(arabic uint16) bool {
	if arabic > 3999 {
		return true
	}
	t.Log("testing", arabic)
	roman := ConvertToRoman(arabic)
	fromRoman := ConvertToArabic(roman)
	return fromRoman == arabic
}
```
Notez que nous enregistrons maintenant l'entrée en utilisant la méthode `log` du framework de test. Assurez-vous d'exécuter la commande `go test` avec le drapeau `-v` pour imprimer la sortie supplémentaire (`go test -v`).

Si vous exécutez le test, ils s'exécutent réellement maintenant et vous pouvez voir ce qui est testé. Vous pouvez exécuter plusieurs fois pour voir que notre code résiste bien aux différentes valeurs ! Cela me donne beaucoup de confiance que notre code fonctionne comme nous le voulons.

Le nombre par défaut d'exécutions que `quick.Check` effectue est de 100, mais vous pouvez le modifier avec une configuration.

```go
if err := quick.Check(assertion, &quick.Config{
	MaxCount: 1000,
}); err != nil {
	t.Error("échec des vérifications", err)
}
```

### Travaux supplémentaires

- Pouvez-vous écrire des tests basés sur les propriétés qui vérifient les autres propriétés que nous avons décrites ?
- Pouvez-vous trouver un moyen de rendre impossible pour quelqu'un d'appeler notre code avec un nombre supérieur à 3999 ?
    - Vous pourriez renvoyer une erreur
    - Ou créer un nouveau type qui ne peut pas représenter > 3999
        - Qu'en pensez-vous ?

## Conclusion

### Plus de pratique TDD avec le développement itératif

L'idée d'écrire du code qui convertit 1984 en MCMLXXXIV vous a-t-elle semblé intimidante au début ? C'était le cas pour moi, et je développe des logiciels depuis assez longtemps.

L'astuce, comme toujours, est de **commencer par quelque chose de simple** et de prendre de **petites étapes**.

À aucun moment dans ce processus nous n'avons fait de grands sauts, ni de refactorisations énormes, ni ne nous sommes emmêlés.

Je peux entendre quelqu'un dire cyniquement "ce n'est qu'un kata". Je ne peux pas le contredire, mais j'adopte toujours la même approche pour chaque projet sur lequel je travaille. Je ne livre jamais un grand système distribué en une seule étape, je trouve la chose la plus simple que l'équipe pourrait livrer (généralement un site web "Hello world") puis j'itère sur de petits morceaux de fonctionnalités en blocs gérables, tout comme nous l'avons fait ici.

La compétence consiste à savoir _comment_ diviser le travail, et cela vient avec la pratique et avec un peu de TDD pour vous aider en chemin.

### Tests basés sur les propriétés

- Intégrés dans la bibliothèque standard
- Si vous pouvez trouver des moyens de décrire les règles de votre domaine dans le code, ce sont d'excellents outils pour vous donner plus de confiance
- Vous forcent à réfléchir profondément à votre domaine
- Potentiellement un bon complément à votre suite de tests

## Post-scriptum

Ce livre dépend des précieux retours de la communauté. [Dave](http://github.com/gypsydave5) est d'une aide énorme dans pratiquement chaque chapitre. Mais il a vraiment râlé à propos de mon utilisation des "chiffres arabes" dans ce chapitre, donc, dans un souci de transparence totale, voici ce qu'il a dit.

> Je vais juste expliquer pourquoi une valeur de type `int` n'est pas vraiment un "chiffre arabe". C'est peut-être moi qui suis trop précis, donc je comprendrai parfaitement si vous me dites d'aller me faire voir.
>
> Un _chiffre_ est un caractère utilisé dans la représentation des nombres - du latin pour "doigt", car nous en avons généralement dix. Dans le système de numération arabe (également appelé hindou-arabe), il y en a dix. Ces chiffres arabes sont :
>
> ```console
>   0 1 2 3 4 5 6 7 8 9
> ```
>
> Un _numéral_ est la représentation d'un nombre à l'aide d'une collection de chiffres. Un numéral arabe est un nombre représenté par des chiffres arabes dans un système de numération positionnel en base 10. Nous disons "positionnel" car chaque chiffre a une valeur différente selon sa position dans le numéral. Ainsi
>
> ```console
>   1337
> ```
>
> Le `1` a une valeur de mille car c'est le premier chiffre d'un numéral à quatre chiffres.
>
> Les chiffres romains sont construits en utilisant un nombre réduit de chiffres (`I`, `V` etc...) principalement comme valeurs pour produire le numéral. Il y a un peu de positionnement mais c'est surtout `I` qui représente toujours "un".
>
> Donc, étant donné cela, est-ce que `int` est un "nombre arabe" ? L'idée d'un nombre n'est pas du tout liée à sa représentation - nous pouvons le voir si nous nous demandons quelle est la représentation correcte de ce nombre :
>
> ```console
> 255
> 11111111
> two-hundred and fifty-five
> FF
> 377
> ```
>
> Oui, c'est une question piège. Elles sont toutes correctes. Ce sont les représentations du même nombre respectivement dans les systèmes de numération décimal, binaire, anglais, hexadécimal et octal.
>
> La représentation d'un nombre sous forme de numéral est _indépendante_ de ses propriétés en tant que nombre - et nous pouvons le voir lorsque nous regardons les littéraux entiers en Go :
>
> ```go
> 	0xFF == 255 // true
> ```
>
> Et comment nous pouvons imprimer des entiers dans une chaîne de format :
>
> ```go
> n := 255
> fmt.Printf("%b %c %d %o %q %x %X %U", n, n, n, n, n, n, n, n)
> // 11111111 ÿ 255 377 'ÿ' ff FF U+00FF
> ```
>
> Nous pouvons écrire le même entier à la fois en hexadécimal et en numéral arabe (décimal).
>
> Donc lorsque la signature de la fonction ressemble à `ConvertToRoman(arabic int) string`, elle fait une petite supposition sur la façon dont elle est appelée. Parce que parfois `arabic` sera écrit comme un littéral d'entier décimal
>
> ```go
> 	ConvertToRoman(255)
> ```
>
> Mais il pourrait tout aussi bien être écrit
>
> ```go
> 	ConvertToRoman(0xFF)
> ```
>
> En réalité, nous ne "convertissons" pas du tout à partir d'un numéral arabe, nous "imprimons" - représentons - un `int` en tant que numéral romain - et les `int` ne sont pas des numérals, arabes ou autres ; ce sont juste des nombres. La fonction `ConvertToRoman` est plus comme `strconv.Itoa` en ce sens qu'elle transforme un `int` en une `string`.
>
> Mais toutes les autres versions du kata ne se soucient pas de cette distinction, alors :shrug:
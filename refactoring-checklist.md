# Checklist pour la refactorisation

la refactorisation est une compétence qui, une fois suffisamment pratiquée, devient, dans la plupart des cas, une seconde nature relativement facile à acquérir.

Cette activité est souvent confondue avec des changements de conception plus importants, mais ce sont des activités distinctes. Faire la distinction entre la refactorisation et les autres activités de programmation est utile car cela permet de travailler avec clarté et discipline.

## Refactoring vs autres activités

la refactorisation consiste simplement à améliorer du code existant <u>sans changer son comportement</u> ; par conséquent, les tests ne devraient pas avoir à changer.

C'est pourquoi c'est la 3ème étape du cycle TDD. Une fois que vous avez ajouté un comportement et un test pour le valider, la refactorisation devrait être une activité qui ne nécessite aucun changement dans votre code de test. **Vous faites autre chose** si vous "refactorisez" du code et que vous devez modifier des tests en même temps.

De nombreux refactorings très utiles sont simples à apprendre et faciles à réaliser (votre IDE en automatise presque entièrement plusieurs), mais, avec le temps, ils ont un impact énorme sur la qualité de votre système.

### Autres activités, comme la "grande" conception

> Je ne change pas le comportement "réel", mais je dois changer mes tests ? Qu'est-ce que ça signifie ?

Supposons que vous travaillez sur un type et que vous souhaitez améliorer la qualité de son code. *la refactorisation ne devrait pas vous obliger à modifier les tests*, vous ne pouvez donc pas :

- Changer le comportement
- Changer les signatures des méthodes

...car vos tests sont couplés à ces deux éléments, mais vous pouvez :

- Introduire des méthodes privées, des champs et même de nouveaux types et interfaces
- Changer les mécanismes internes des méthodes publiques

Et si vous voulez changer la signature d'une méthode ?

```go
func (b BirthdayGreeter) WishHappyBirthday(age int, firstname, lastname string, email Email) {
	// un code fascinant d'envoi d'emails
}
```

Vous pourriez trouver que sa liste d'arguments est trop longue et vouloir apporter plus de cohésion et de sens au code.

```go
func (b BirthdayGreeter) WishHappyBirthday(person Person)
```

Eh bien, vous êtes maintenant dans une phase de **conception** et vous devez vous assurer de procéder avec prudence. Si vous ne le faites pas avec discipline, vous risquez de faire un gâchis de votre code, du test qui le soutient, *et* probablement des éléments qui en dépendent - rappelez-vous, ce n'est pas seulement vos tests qui utilisent `WishHappyBirthday`. Espérons qu'il est également utilisé par du code "réel" !

**Vous devriez toujours pouvoir piloter ce changement avec un test d'abord**. On peut ergoter sur la question de savoir s'il s'agit d'un changement de "comportement", mais vous voulez que votre méthode se comporte différemment.

Comme il s'agit d'un changement de comportement, appliquez également le processus TDD ici. L'un des avantages du TDD est qu'il vous offre un moyen simple, sûr et reproductible de piloter les changements de comportement dans votre système ; pourquoi l'abandonner dans ces situations simplement parce qu'il *semble* différent ?

Dans ce cas, vous allez modifier vos tests existants pour utiliser le nouveau type. Les étapes itératives et petites que vous faites habituellement avec le TDD pour réduire les risques et apporter discipline et clarté vous aideront également dans ces situations.

Il y a des chances que vous ayez plusieurs tests qui appellent `WishHappyBirthday` ; dans ces scénarios, je suggérerais de mettre en commentaire tous les tests sauf un, de réaliser le changement, puis de retravailler le reste des tests comme vous le jugez approprié.

### Grande conception

La conception peut nécessiter des changements plus importants et des conversations plus approfondies, et comporte généralement un niveau de subjectivité. Changer la conception de parties de votre système est généralement un processus plus long que la refactorisation ; néanmoins, vous devriez toujours vous efforcer de réduire les risques en réfléchissant à la façon de le faire par petites étapes.

### Voir la forêt plutôt que les arbres

> [Si quelqu'un ne peut pas **voir la forêt à cause des arbres** en anglais britannique, ou ne peut pas voir la forêt à cause des arbres en anglais américain, c'est qu'il est très impliqué dans les détails de quelque chose et ne remarque donc pas ce qui est important dans l'ensemble.](https://www.collinsdictionary.com/dictionary/english/cant-see-the-wood-for-the-trees)

Parler des questions de "grande" conception est plus facile lorsque **le code sous-jacent est bien factorisé**. Si vous et vos collègues devez passer un temps considérable à analyser mentalement un fouillis de code chaque fois que vous ouvrez un fichier, quelle chance avez-vous de réfléchir à la conception du code ?

C'est pourquoi **la refactorisation constante est si important dans le processus TDD**. Si nous ne résolvons pas les petits problèmes de conception, nous aurons du mal à concevoir la conception globale de notre système plus vaste.

Malheureusement, un code mal construit empire de façon exponentielle à mesure que les ingénieurs empilent de la complexité sur des fondations instables.

## Liste de contrôle mentale de départ

**Prenez l'habitude de passer en revue une liste de contrôle mentale à chaque cycle TDD.** Plus vous vous forcez à pratiquer, plus cela devient facile. **C'est une compétence qui nécessite de la pratique.** Rappelez-vous, chacun de ces changements ne devrait nécessiter aucun changement dans vos tests.

J'ai inclus des raccourcis pour IntelliJ/GoLand, que mes collègues et moi utilisons. Chaque fois que je forme un nouvel ingénieur, je l'encourage à essayer d'acquérir la mémoire musculaire et l'habitude d'utiliser ces outils pour refactoriser rapidement et en toute sécurité.

### Inliner les variables

Si vous créez une variable, uniquement pour la transmettre à une autre méthode/fonction :

```go
url := baseURL + "/user/" + id
res, err := client.Get(url)
```

Envisagez de l'inliner (`command+option+n`) *sauf si* le nom de la variable ajoute une signification importante.

```go
res, err := client.Get(baseURL + "/user/" + id)
```

Ne soyez pas _trop_ malin en inlinant ; l'objectif n'est pas d'avoir zéro variable et à la place d'avoir des one-liners ridicules que personne ne peut lire. Si vous pouvez ajouter un nom significatif à une valeur, il peut être préférable de la laisser telle quelle.

### DRY-ifier les valeurs avec des variables extraites

"Don't repeat yourself" (DRY - Ne vous répétez pas). Vous utilisez la même valeur plusieurs fois dans une fonction ? Envisagez d'extraire et de capturer une variable dans un nom de variable significatif (`command+option+v`).

Cela aide à la lisibilité et facilite la modification de la valeur à l'avenir, car vous n'aurez pas à vous souvenir de mettre à jour plusieurs occurrences de la même valeur.

### DRY-ifier les choses en général

[DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) a mauvaise réputation ces jours-ci, avec quelques justifications. DRY est l'un de ces concepts qui est *trop* facile à comprendre à un niveau superficiel et qui est ensuite mal appliqué.

Un ingénieur peut facilement pousser DRY trop loin, créant des abstractions déconcertantes et enchevêtrées pour économiser quelques lignes de code plutôt que l'_idée_ *réelle* de DRY, qui est de capturer une _idée_ en un seul endroit. Réduire le nombre de lignes de code est souvent un effet secondaire de DRY, **mais ce n'est pas l'objectif réel**.

Alors oui, DRY peut être mal appliqué, mais l'extrême opposé consistant à refuser de DRY-ifier quoi que ce soit est également mauvais. Le code répété ajoute du bruit et augmente les coûts de maintenance. Un refus de rassembler des concepts ou des valeurs connexes en une seule chose par peur d'une mauvaise utilisation de DRY cause des problèmes *différents*.

Donc, plutôt que d'être extrémiste d'un côté ou de l'autre entre "tout doit être DRY" ou "DRY est mauvais", engagez votre cerveau et réfléchissez au code que vous voyez devant vous. Qu'est-ce qui est répété ? Est-ce nécessaire ? La liste des paramètres semble-t-elle raisonnable si vous encapsulez du code répété dans une méthode ? Cela semble-t-il s'auto-documenter et encapsuler clairement l'"idée" ?

Neuf fois sur dix, vous pouvez regarder la liste d'arguments d'une fonction, et si elle semble désordonnée et confuse, c'est probablement une mauvaise application de DRY.

Si DRY-ifier du code semble difficile, vous rendez probablement les choses plus complexes ; envisagez de vous arrêter.

DRY-ifiez avec soin, **mais pratiquer cela fréquemment améliorera votre jugement**. J'encourage mes collègues à "simplement essayer" et à utiliser le contrôle de source pour revenir à la sécurité si c'est faux.

<u>**Essayer ces choses vous apprendra plus que d'en discuter**</u>, et le contrôle de source associé à de bons tests automatisés vous donne la configuration parfaite pour expérimenter et apprendre.

### Extraire les valeurs "magiques"

> [Valeurs uniques dont la signification n'est pas expliquée ou qui apparaissent plusieurs fois et qui pourraient (de préférence) être remplacées par des constantes nommées](https://en.wikipedia.org/wiki/Magic_number_(programming))

Utilisez la variable d'extraction (command+option+v) ou la constante (command+option+c) pour donner un sens aux valeurs magiques. Cela peut être considéré comme l'inverse du refactoring d'inlining. Je me retrouve souvent à "basculer" le code avec inline et extract pour m'aider à juger ce que je pense être plus lisible.

Rappelez-vous que l'extraction de valeurs répétées ajoute également un niveau de _couplage_. Tout ce qui utilise cette valeur est maintenant couplé. Considérez le code suivant :

```go
func main() {
	api1Client := http.Client{
		Timeout: 1 * time.Second,
	}
	api2Client := http.Client{
		Timeout: 1 * time.Second,
	}
	api3Client := http.Client{
		Timeout: 1 * time.Second,
	}
	//etc
}
```

Nous configurons des clients HTTP pour notre application. Il y a quelques _valeurs magiques_ ici, et nous pourrions DRY-ifier le `Timeout` en extrayant une variable et en lui donnant un nom significatif.

![Une capture d'écran de moi extrayant une variable](https://i.imgur.com/4sgUG7L.png)

Maintenant, le code ressemble à ceci

```go
func main() {
	timeout := 1 * time.Second
	api1Client := http.Client{
		Timeout: timeout,
	}
	api2Client := http.Client{
		Timeout: timeout,
	}
	api3Client := http.Client{
		Timeout: timeout,
	}
	// etc..
}
```

Nous n'avons plus de valeur magique ; nous lui avons donné un nom significatif, mais nous avons également fait en sorte que les trois clients **partagent le même timeout**. C'est _peut-être_ ce que vous voulez ; les refactorings sont assez spécifiques au contexte, mais c'est quelque chose dont il faut se méfier.

Si vous savez bien utiliser votre IDE, vous pouvez faire la refactorisation _inline_ pour permettre aux clients d'avoir à nouveau des valeurs `Timeout` séparées.

### Rendre les méthodes/fonctions publiques faciles à scanner

Votre code a-t-il des méthodes ou des fonctions publiques excessivement longues ?

Encapsulez les étapes dans des méthodes/fonctions privées avec la refactorisation de méthode d'extraction (`command+option+m`).

Le code ci-dessous contient des cérémonies ennuyeuses et distrayantes autour de la création d'une chaîne JSON et de sa transformation en `io.Reader` afin que nous puissions la `POST` dans une requête HTTP.

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	payload := []byte(`{"name": "` + name + `"}`)

	req, err := http.NewRequest(
		http.MethodPost,
		url,
		bytes.NewBuffer(payload),
	)
	//todo: gérer les codes, err etc
}
```

D'abord, utilisez la refactorisation de variable inline (command+option+n) pour mettre le `payload` dans la création du buffer.

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		bytes.NewBuffer([]byte(`{"name": "`+name+`"}`)),
	)
	// etc
}
```

Maintenant, nous pouvons extraire la création du payload JSON dans une fonction en utilisant la refactorisation de méthode d'extraction (`command+option+m`) pour supprimer le bruit de la méthode.

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		createWidgetPayload(name),
	)
	// etc
}
```

Les méthodes et fonctions publiques devraient décrire *ce qu'elles font* plutôt que *comment elles le font*.

> **Chaque fois que je dois réfléchir pour comprendre ce que fait le code, je me demande si je peux refactoriser le code pour rendre cette compréhension plus immédiatement évidente**

-- Martin Fowler

Cela vous aide à mieux comprendre la conception globale, et cela vous permet ensuite de poser des questions sur les responsabilités :

>  Pourquoi cette méthode fait-elle X ? Cela ne devrait-il pas être dans Y ?

> Pourquoi cette méthode fait-elle tant de tâches ? Pouvons-nous consolider cela ailleurs ?

Les fonctions et méthodes privées sont excellentes ; elles vous permettent d'envelopper des "comment" non pertinents en "quoi".

#### Mais maintenant je ne sais pas comment ça marche !

Une objection courante à ce refactoring, favorisant des fonctions et méthodes plus petites composées d'autres, est qu'il peut rendre difficile la compréhension du fonctionnement du code. Ma réponse directe à cela est

> Avez-vous appris à naviguer efficacement dans les bases de code à l'aide de vos outils ?

Assez délibérément, en tant que _rédacteur_ de `CreateWidget`, je ne veux pas que la création d'une chaîne spécifique soit un personnage essentiel dans la narration de la méthode. C'est un bruit distrayant et non pertinent pour le lecteur 99 % du temps.

Cependant, si quelqu'un _s'en soucie_, vous appuyez sur `command+b` (ou quel que soit le "naviguer vers le symbole" pour vous) sur `createWidgetPayload` ... et le lisez. Appuyez sur `command+flèche gauche` pour revenir en arrière.

### Déplacer la création de valeur au moment de la construction

Les méthodes doivent souvent créer des valeurs et les utiliser, comme l'`url` dans notre méthode `CreateWidget` précédente.

```go
type WidgetService struct {
	baseURL string
	client  *http.Client
}

func NewWidgetService(baseURL string) *WidgetService {
	client := http.Client{
		Timeout: 10 * time.Second,
	}
	return &WidgetService{baseURL: baseURL, client: &client}
}

func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		createWidgetPayload(name),
	)
	// etc
}
```

Une technique de refactoring que vous pourriez appliquer ici est, si une valeur est créée **qui ne dépend pas des arguments de la méthode**, vous pouvez plutôt créer un _champ_ dans votre type et le calculer dans votre fonction constructeur.

```go
type WidgetService struct {
	client          *http.Client
	createWidgetURL string
}

func NewWidgetService(baseURL string) *WidgetService {
	client := http.Client{
		Timeout: 10 * time.Second,
	}
	return &WidgetService{
		createWidgetURL: baseURL + "/widgets",
		client:          &client,
	}
}

func (ws *WidgetService) CreateWidget(name string) error {
	req, err := http.NewRequest(
		http.MethodPost,
		ws.createWidgetURL,
		createWidgetPayload(name),
	)
	// etc
}
```

En les déplaçant au moment de la construction, vous pouvez simplifier vos méthodes.

#### Comparaison de `CreateWidget`

En commençant par

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	payload := []byte(`{"name": "` + name + `"}`)
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		bytes.NewBuffer(payload),
	)
	// etc
}

```

Avec quelques refactorings de base, pilotés presque entièrement à l'aide d'outils automatisés, nous avons obtenu

```go
func (ws *WidgetService) CreateWidget(name string) error {
	req, err := http.NewRequest(
		http.MethodPost,
		ws.createWidgetURL,
		createWidgetPayload(name),
	)
	// etc
}
```

C'est une petite amélioration, mais elle se lit certainement mieux. Si vous êtes bien exercé, ce type d'amélioration vous prendra à peine une minute, et tant que vous avez bien appliqué le TDD, vous aurez le filet de sécurité des tests pour vous assurer que vous ne cassez rien. Ces améliorations mineures continues sont vitales pour la santé à long terme d'une base de code.

### Essayez de supprimer les commentaires

> Une heuristique que nous suivons est que chaque fois que nous ressentons le besoin de commenter quelque chose, nous écrivons plutôt une méthode.

-- Martin Fowler

Encore une fois, la refactorisation de méthode d'extraction peut être votre ami ici.

## Exceptions à la règle

Il existe des améliorations que vous pouvez apporter à votre code qui nécessitent un changement dans vos tests, que je serais quand même heureux de mettre dans le panier "refactoring", même si cela enfreint la règle.

Un exemple simple serait de renommer un symbole public (par exemple, une méthode, un type ou une fonction) avec `shift+F6`. Cela modifiera, bien sûr, les codes de production et de test.

Cependant, comme il s'agit d'un changement **automatisé et sûr**, le risque de tomber dans une spirale de tests et de code de production défaillants que tant de personnes connaissent avec d'autres types de changements de *conception* est minime.

Pour cette raison, tout changement que vous pouvez effectuer en toute sécurité avec votre IDE/éditeur, je l'appellerais volontiers du refactoring.

## Utilisez vos outils pour vous aider à pratiquer la refactorisation

- Vous devriez exécuter vos tests unitaires chaque fois que vous effectuez l'un de ces petits changements. Nous investissons du temps pour rendre notre code testable par unité, et la boucle de rétroaction de quelques millisecondes est l'un des avantages significatifs ; utilisez-la !
- Appuyez-vous sur le contrôle de source. Vous ne devriez pas avoir peur d'essayer des idées. Si vous êtes satisfait, faites un commit ; sinon, revenez en arrière. Cela devrait vous sembler confortable et facile, et ne pas être un gros problème.
- Plus vous tirez parti de vos tests unitaires et du contrôle de source, plus il est facile de *pratiquer* la refactorisation. Une fois que vous maîtrisez cette discipline, **vos compétences en conception augmentent rapidement** car vous disposez d'une boucle de rétroaction et d'un filet de sécurité fiables et efficaces.
- Trop souvent dans ma carrière, j'ai entendu des développeurs se plaindre de ne pas avoir le temps de refactoriser ; malheureusement, il est clair que cela leur prend tellement de temps parce qu'ils ne le font pas avec discipline - et ils ne l'ont pas assez pratiqué.
- Bien que votre vitesse de frappe ne soit jamais la cause principale d'une perte de temps, vous devriez être en mesure d'utiliser n'importe quel éditeur/IDE que vous utilisez pour refactoriser en toute sécurité et rapidement. Par exemple, si votre outil ne vous permet pas d'extraire des variables en une frappe de touche, vous le ferez moins car c'est plus laborieux et risqué.

## Ne demandez pas la permission de refactoriser

la refactorisation devrait être une occurrence fréquente dans votre travail, quelque chose que vous faites tout le temps. Cela ne devrait pas non plus être un gouffre temporel, surtout si c'est fait peu et souvent.

Si vous ne refactorisez pas, votre qualité interne en souffrira, la capacité de votre équipe diminuera et la pression augmentera.

Martin Fowler a une autre citation fantastique pour nous.

> Sauf lorsque vous êtes très proche d'une échéance, vous ne devriez pas reporter la refactorisation parce que vous n'avez pas le temps. L'expérience de plusieurs projets a montré qu'une séance de refactoring entraîne une augmentation de la productivité. Ne pas avoir assez de temps est généralement un signe que vous avez besoin de faire du refactoring.

## Conclusion

Ce n'est pas une liste exhaustive, juste un début. Lisez le livre Refactoring de Martin Fowler (2e éd.) pour devenir un pro.

la refactorisation devrait être extrêmement rapide et sûr lorsque vous êtes bien exercé, il y a donc peu d'excuses pour ne pas le faire. Trop de personnes considèrent la refactorisation comme une décision que d'autres doivent prendre plutôt que comme une compétence à apprendre jusqu'à ce qu'elle fasse partie régulière de votre travail.

Nous devrions toujours nous efforcer de laisser le code dans un état *exemplaire*.

Un bon refactoring conduit à un code plus facile à comprendre. Une compréhension du code signifie que de meilleures conceptions sont plus faciles à repérer. Il est beaucoup plus difficile de trouver des conceptions dans des systèmes avec des fonctions massives, du code inutilement dupliqué, une imbrication profonde, etc. **Un refactoring fréquent et petit est nécessaire pour une meilleure conception**.
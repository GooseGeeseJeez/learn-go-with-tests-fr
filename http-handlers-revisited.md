# Revisite des gestionnaires HTTP

**[Vous pouvez trouver tout le code ici](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/http-handlers-revisited)**

Ce livre contient déjà un chapitre sur [le test d'un gestionnaire HTTP](http-server.md), mais celui-ci proposera une discussion plus large sur leur conception, afin qu'ils soient simples à tester.

Nous examinerons un exemple réel et comment nous pouvons améliorer sa conception en appliquant des principes tels que le principe de responsabilité unique et la séparation des préoccupations. Ces principes peuvent être réalisés en utilisant des [interfaces](structs-methods-and-interfaces.md) et [l'injection de dépendances](dependency-injection.md). En faisant cela, nous montrerons que tester les gestionnaires est en réalité assez trivial.

![Question courante dans la communauté Go illustrée](amazing-art.png)

Tester les gestionnaires HTTP semble être une question récurrente dans la communauté Go, et je pense que cela révèle un problème plus large de malcompréhension de leur conception.

Très souvent, les difficultés que rencontrent les gens avec les tests proviennent de la conception de leur code plutôt que de l'écriture des tests elle-même. Comme je le souligne si souvent dans ce livre :

> Si vos tests vous causent de la douleur, écoutez ce signal et réfléchissez à la conception de votre code.

## Un exemple

[Santosh Kumar m'a tweeté](https://twitter.com/sntshk/status/1255559003339284481)

> Comment puis-je tester un gestionnaire http qui a une dépendance à mongodb ?

Voici le code

```go
func Registration(w http.ResponseWriter, r *http.Request) {
	var res model.ResponseResult
	var user model.User

	w.Header().Set("Content-Type", "application/json")

	jsonDecoder := json.NewDecoder(r.Body)
	jsonDecoder.DisallowUnknownFields()
	defer r.Body.Close()

	// vérifie s'il y a un corps JSON approprié ou une erreur
	if err := jsonDecoder.Decode(&user); err != nil {
		res.Error = err.Error()
		// retourne un code d'état 400
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	// Connexion à mongodb
	client, _ := mongo.NewClient(options.Client().ApplyURI("mongodb://127.0.0.1:27017"))
	ctx, _ := context.WithTimeout(context.Background(), 10*time.Second)
	err := client.Connect(ctx)
	if err != nil {
		panic(err)
	}
	defer client.Disconnect(ctx)
	// Vérifie si le nom d'utilisateur existe déjà dans le stockage de données des utilisateurs, si c'est le cas, 400
	// sinon insère l'utilisateur immédiatement
	collection := client.Database("test").Collection("users")
	filter := bson.D{{"username", user.Username}}
	var foundUser model.User
	err = collection.FindOne(context.TODO(), filter).Decode(&foundUser)
	if foundUser.Username == user.Username {
		res.Error = UserExists
		// retourne un code d'état 400
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	pass, err := bcrypt.GenerateFromPassword([]byte(user.Password), bcrypt.DefaultCost)
	if err != nil {
		res.Error = err.Error()
		// retourne un code d'état 400
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}
	user.Password = string(pass)

	insertResult, err := collection.InsertOne(context.TODO(), user)
	if err != nil {
		res.Error = err.Error()
		// retourne un code d'état 400
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	// retourne 200
	w.WriteHeader(http.StatusOK)
	res.Result = fmt.Sprintf("%s: %s", UserCreated, insertResult.InsertedID)
	json.NewEncoder(w).Encode(res)
	return
}
```

Énumérons toutes les choses que cette fonction unique doit faire :

1. Écrire des réponses HTTP, envoyer des en-têtes, des codes d'état, etc.
2. Décoder le corps de la requête en un `User`
3. Se connecter à une base de données (et tous les détails autour de cela)
4. Interroger la base de données et appliquer une logique métier en fonction du résultat
5. Générer un mot de passe
6. Insérer un enregistrement

C'est trop.

## Qu'est-ce qu'un gestionnaire HTTP et que devrait-il faire ?

En oubliant les détails spécifiques à Go pour un moment, quelle que soit la langue dans laquelle j'ai travaillé, ce qui m'a toujours bien servi, c'est de penser à la [séparation des préoccupations](https://en.wikipedia.org/wiki/Separation_of_concerns) et au [principe de responsabilité unique](https://en.wikipedia.org/wiki/Single-responsibility_principle).

Cela peut être assez délicat à appliquer selon le problème que vous résolvez. Qu'est-ce qu'une responsabilité _exactement_ ?

Les lignes peuvent devenir floues selon votre niveau d'abstraction de pensée, et parfois votre première intuition peut ne pas être la bonne.

Heureusement, avec les gestionnaires HTTP, j'ai une assez bonne idée de ce qu'ils devraient faire, quel que soit le projet sur lequel j'ai travaillé :

1. Accepter une requête HTTP, l'analyser et la valider.
2. Appeler un `ServiceChose` pour faire `LogiqueMétierImportante` avec les données que j'ai obtenues à l'étape 1.
3. Envoyer une réponse `HTTP` appropriée en fonction de ce que `ServiceChose` renvoie.

Je ne dis pas que chaque gestionnaire HTTP _jamais_ créé devrait avoir à peu près cette forme, mais 99 fois sur 100, c'est ce que je constate.

Lorsque vous séparez ces préoccupations :

 - Tester les gestionnaires devient un jeu d'enfant et se concentre sur un petit nombre de préoccupations.
 - Plus important encore, tester `LogiqueMétierImportante` n'a plus à se préoccuper de `HTTP`, vous pouvez tester la logique métier proprement.
 - Vous pouvez utiliser `LogiqueMétierImportante` dans d'autres contextes sans avoir à la modifier.
 - Si `LogiqueMétierImportante` change ce qu'elle fait, tant que l'interface reste la même, vous n'avez pas à changer vos gestionnaires.

## Les gestionnaires de Go

[`http.HandlerFunc`](https://golang.org/pkg/net/http/#HandlerFunc)

> Le type HandlerFunc est un adaptateur pour permettre l'utilisation de fonctions ordinaires comme gestionnaires HTTP.

`type HandlerFunc func(ResponseWriter, *Request)`

Lecteur, prenez une respiration et regardez le code ci-dessus. Qu'est-ce que vous remarquez ?

**C'est une fonction qui prend des arguments**

Il n'y a pas de magie de framework, pas d'annotations, pas de haricots magiques, rien.

C'est juste une fonction, _et nous savons comment tester des fonctions_.

Cela s'inscrit parfaitement dans le commentaire ci-dessus :

- Elle prend un [`http.Request`](https://golang.org/pkg/net/http/#Request) qui n'est qu'un ensemble de données que nous devons inspecter, analyser et valider.
- > [Une interface `http.ResponseWriter` est utilisée par un gestionnaire HTTP pour construire une réponse HTTP.](https://golang.org/pkg/net/http/#ResponseWriter)

### Exemple de test super basique

```go
func Teapot(res http.ResponseWriter, req *http.Request) {
	res.WriteHeader(http.StatusTeapot)
}

func TestTeapotHandler(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/", nil)
	res := httptest.NewRecorder()

	Teapot(res, req)

	if res.Code != http.StatusTeapot {
		t.Errorf("j'ai obtenu le statut %d mais je voulais %d", res.Code, http.StatusTeapot)
	}
}
```

Pour tester notre fonction, nous l'_appelons_.

Pour notre test, nous passons un `httptest.ResponseRecorder` comme argument `http.ResponseWriter`, et notre fonction l'utilisera pour écrire la réponse `HTTP`. L'enregistreur enregistrera (ou _espionnera_) ce qui a été envoyé, et ensuite nous pourrons faire nos assertions.

## Appeler un `ServiceChose` dans notre gestionnaire

Une plainte courante à propos des tutoriels TDD est qu'ils sont toujours "trop simples" et pas assez "réels". Ma réponse à cela est :

> Ne serait-il pas agréable que tout votre code soit simple à lire et à tester comme les exemples que vous mentionnez ?

C'est l'un des plus grands défis auxquels nous sommes confrontés, mais nous devons continuer à y travailler. Il _est possible_ (bien que pas nécessairement facile) de concevoir du code de manière à ce qu'il soit simple à lire et à tester si nous pratiquons et appliquons de bons principes d'ingénierie logicielle.

En récapitulant ce que fait le gestionnaire précédent :

1. Écrire des réponses HTTP, envoyer des en-têtes, des codes d'état, etc.
2. Décoder le corps de la requête en un `User`
3. Se connecter à une base de données (et tous les détails autour de cela)
4. Interroger la base de données et appliquer une logique métier en fonction du résultat
5. Générer un mot de passe
6. Insérer un enregistrement

En prenant l'idée d'une séparation des préoccupations plus idéale, je voudrais que ce soit plutôt comme :

1. Décoder le corps de la requête en un `User`
2. Appeler un `UserService.Register(user)` (c'est notre `ServiceChose`)
3. S'il y a une erreur, agir en conséquence (l'exemple envoie toujours un `400 BadRequest`, ce qui je pense n'est pas correct), je vais juste avoir un gestionnaire global de `500 Internal Server Error` _pour l'instant_. Je dois souligner que renvoyer `500` pour toutes les erreurs fait une API terrible ! Plus tard, nous pourrons rendre la gestion des erreurs plus sophistiquée, peut-être avec des [types d'erreur](error-types.md).
4. S'il n'y a pas d'erreur, `201 Created` avec l'ID comme corps de la réponse (à nouveau par souci de concision/paresse)

Pour des raisons de brièveté, je ne vais pas revoir le processus TDD habituel, consultez tous les autres chapitres pour des exemples.

### Nouvelle conception

```go
type UserService interface {
	Register(user User) (insertedID string, err error)
}

type UserServer struct {
	service UserService
}

func NewUserServer(service UserService) *UserServer {
	return &UserServer{service: service}
}

func (u *UserServer) RegisterUser(w http.ResponseWriter, r *http.Request) {
	defer r.Body.Close()

	// analyse et validation de la requête
	var newUser User
	err := json.NewDecoder(r.Body).Decode(&newUser)

	if err != nil {
		http.Error(w, fmt.Sprintf("impossible de décoder le payload utilisateur : %v", err), http.StatusBadRequest)
		return
	}

	// appeler un service pour s'occuper du travail difficile
	insertedID, err := u.service.Register(newUser)

	// selon ce que nous recevons, répondre en conséquence
	if err != nil {
		//todo: gérer différents types d'erreurs différemment
		http.Error(w, fmt.Sprintf("problème lors de l'enregistrement du nouvel utilisateur : %v", err), http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusCreated)
	fmt.Fprint(w, insertedID)
}
```

Notre méthode `RegisterUser` correspond à la forme de `http.HandlerFunc`, donc nous sommes prêts. Nous l'avons attachée comme méthode à un nouveau type `UserServer` qui contient une dépendance à un `UserService` qui est capturé comme une interface.

Les interfaces sont un moyen fantastique de s'assurer que nos préoccupations `HTTP` sont découplées de toute implémentation spécifique ; nous pouvons simplement appeler la méthode sur la dépendance, et nous n'avons pas à nous soucier de _comment_ un utilisateur est enregistré.

Si vous souhaitez explorer cette approche plus en détail en suivant le TDD, lisez le chapitre [Injection de Dépendances](dependency-injection.md) et le [chapitre Serveur HTTP de la section "Construire une application"](http-server.md).

Maintenant que nous nous sommes découplés de tout détail d'implémentation spécifique concernant l'enregistrement, l'écriture du code pour notre gestionnaire est simple et suit les responsabilités décrites précédemment.

### Les tests !

Cette simplicité se reflète dans nos tests.

```go
type MockUserService struct {
	RegisterFunc    func(user User) (string, error)
	UsersRegistered []User
}

func (m *MockUserService) Register(user User) (insertedID string, err error) {
	m.UsersRegistered = append(m.UsersRegistered, user)
	return m.RegisterFunc(user)
}

func TestRegisterUser(t *testing.T) {
	t.Run("peut enregistrer des utilisateurs valides", func(t *testing.T) {
		user := User{Name: "CJ"}
		expectedInsertedID := "whatever"

		service := &MockUserService{
			RegisterFunc: func(user User) (string, error) {
				return expectedInsertedID, nil
			},
		}
		server := NewUserServer(service)

		req := httptest.NewRequest(http.MethodGet, "/", userToJSON(user))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusCreated)

		if res.Body.String() != expectedInsertedID {
			t.Errorf("corps attendu de %q mais obtenu %q", res.Body.String(), expectedInsertedID)
		}

		if len(service.UsersRegistered) != 1 {
			t.Fatalf("attendu 1 utilisateur ajouté mais obtenu %d", len(service.UsersRegistered))
		}

		if !reflect.DeepEqual(service.UsersRegistered[0], user) {
			t.Errorf("l'utilisateur enregistré %+v n'était pas celui attendu %+v", service.UsersRegistered[0], user)
		}
	})

	t.Run("renvoie 400 bad request si le corps n'est pas un JSON utilisateur valide", func(t *testing.T) {
		server := NewUserServer(nil)

		req := httptest.NewRequest(http.MethodGet, "/", strings.NewReader("les problèmes me trouveront"))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusBadRequest)
	})

	t.Run("renvoie une erreur 500 internal server error si le service échoue", func(t *testing.T) {
		user := User{Name: "CJ"}

		service := &MockUserService{
			RegisterFunc: func(user User) (string, error) {
				return "", errors.New("impossible d'ajouter un nouvel utilisateur")
			},
		}
		server := NewUserServer(service)

		req := httptest.NewRequest(http.MethodGet, "/", userToJSON(user))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusInternalServerError)
	})
}
```

Maintenant que notre gestionnaire n'est pas couplé à une implémentation spécifique de stockage, il est trivial pour nous d'écrire un `MockUserService` pour nous aider à écrire des tests unitaires simples et rapides qui exercent les responsabilités spécifiques qu'il a.

### Et le code de la base de données ? Vous trichez !

Tout ceci est très délibéré. Nous ne voulons pas que les gestionnaires HTTP se préoccupent de notre logique métier, de nos bases de données, de nos connexions, etc.

En faisant cela, nous avons libéré le gestionnaire des détails compliqués, nous avons _aussi_ facilité le test de notre couche de persistance et de notre logique métier car elle n'est plus couplée à des détails HTTP non pertinents.

Tout ce que nous devons faire maintenant, c'est implémenter notre `UserService` en utilisant la base de données que nous voulons utiliser

```go
type MongoUserService struct {
}

func NewMongoUserService() *MongoUserService {
	//todo: passer l'URL de la BD en argument à cette fonction
	//todo: se connecter à la bdd, créer un pool de connexions
	return &MongoUserService{}
}

func (m MongoUserService) Register(user User) (insertedID string, err error) {
	// utiliser m.mongoConnection pour effectuer des requêtes
	panic("implémentez-moi")
}
```

Nous pouvons tester cela séparément et une fois que nous sommes satisfaits dans `main`, nous pouvons assembler ces deux unités pour notre application fonctionnelle.

```go
func main() {
	mongoService := NewMongoUserService()
	server := NewUserServer(mongoService)
	http.ListenAndServe(":8000", http.HandlerFunc(server.RegisterUser))
}
```

### Une conception plus robuste et extensible avec peu d'effort

Ces principes non seulement nous facilitent la vie à court terme, mais rendent également le système plus facile à étendre à l'avenir.

Il ne serait pas surprenant que dans les futures itérations de ce système, nous souhaitions envoyer à l'utilisateur un e-mail de confirmation d'inscription.

Avec l'ancienne conception, nous aurions dû modifier le gestionnaire _et_ les tests environnants. C'est souvent ainsi que des parties de code deviennent impossibles à maintenir, de plus en plus de fonctionnalités s'infiltrent car c'est déjà _conçu_ de cette façon ; pour que le "gestionnaire HTTP" gère... tout !

En séparant les préoccupations à l'aide d'une interface, nous n'avons pas besoin de modifier le gestionnaire _du tout_ car il ne se préoccupe pas de la logique métier autour de l'inscription.

## Conclusion

Tester les gestionnaires HTTP de Go n'est pas un défi, mais concevoir un bon logiciel peut l'être !

Les gens font l'erreur de penser que les gestionnaires HTTP sont spéciaux et abandonnent les bonnes pratiques d'ingénierie logicielle lorsqu'ils les écrivent, ce qui rend ensuite leur test difficile.

Répétons-le encore une fois ; **les gestionnaires http de Go sont juste des fonctions**. Si vous les écrivez comme vous écririez d'autres fonctions, avec des responsabilités claires et une bonne séparation des préoccupations, vous n'aurez aucun problème à les tester, et votre base de code sera plus saine.
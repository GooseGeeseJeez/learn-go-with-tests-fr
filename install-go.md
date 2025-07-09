# Installer Go, configurer l'environnement pour la productivité

Les instructions d'installation officielles de Go sont disponibles [ici](https://golang.org/doc/install).

## L'Environnement Go

### Les Modules Go

Go 1.11 a introduit les [Modules](https://go.dev/wiki/Modules). Cette approche est le mode de *build* par défaut depuis Go 1.16, par conséquent l'utilisation de `GOPATH` n'est pas recommandée.

Les modules visent à résoudre les problèmes liés à la gestion des dépendances, la sélection de version et les *builds* reproductibles ; ils permettent également aux utilisateurs d'exécuter du code Go en dehors de `GOPATH`.

L'utilisation des modules est assez simple. Sélectionnez un répertoire quelconque en dehors de `GOPATH` comme racine de votre projet, et créez un nouveau module avec la commande `go mod init`.

Un fichier `go.mod` sera généré, contenant le chemin du module, une version de Go, et ses exigences de dépendances, qui sont les autres modules nécessaires pour un *build* réussi.

Si aucun `<chemin-module>` n'est spécifié, `go mod init` essaiera de deviner le chemin du module à partir de la structure du répertoire. Il peut également être remplacé en fournissant un argument.

```sh
mkdir mon-projet
cd mon-projet
go mod init <chemin-module>
```

Un fichier `go.mod` pourrait ressembler à ceci :

```
module cmd

go 1.16

```

La documentation intégrée fournit un aperçu de toutes les commandes `go mod` disponibles.

```sh
go help mod
go help mod init
```

## Le Linting Go

Une amélioration par rapport au *linter* par défaut peut être configurée en utilisant [GolangCI-Lint](https://golangci-lint.run).

Cela peut être installé comme suit :

```sh
brew install golangci-lint
```

## Refactorisation et vos outils

Un sujet important de ce livre est l'importance de la refactorisation.

Vos outils peuvent vous aider à faire de plus gros refactorings sereinement.

Vous devriez être suffisamment à l'aise avec votre éditeur de code pour effectuer les opérations suivantes avec une simple combinaison de touches :

- **Extraire/Inliner une variable**. Prendre des valeurs magiques et leur donner un nom vous permet de simplifier votre code rapidement.
- **Extraire une méthode/fonction**. Il est vital de pouvoir prendre une section de code et extraire des fonctions/méthodes.
- **Renommer**. Vous devriez pouvoir renommer des symboles à travers les fichiers sereinement.
- **go fmt**. Go a un formateur opiniâtre appelé `go fmt`. Votre éditeur de code devrait l'exécuter sur chaque fichier venant d'être sauvegardé.
- **Exécuter les tests**. Vous devriez pouvoir faire n'importe lequel des points ci-dessus puis rapidement relancer vos tests pour vous assurer que votre refactorisation n'a rien cassé.

De plus, pour vous aider à travailler avec votre code, vous devriez pouvoir :

- **Voir la signature d'une fonction**. Vous ne devriez jamais douter sur la façon d'appeler une fonction que vous avez écrite en Go. Votre IDE devrait décrire une fonction par sa documentation, ses paramètres et ce qu'elle retourne.
- **Voir la définition d'une fonction**. Si ce qu'une fonction fait n'est toujours pas clair, vous devriez pouvoir aller directement au code source et essayer de le comprendre par vous-même.
- **Trouver les utilisations d'un symbole**. Comprendre le contexte d'une fonction peut vous aider à prendre des décisions lors de la refactorisation.

Maîtriser vos outils vous aidera à vous concentrer sur le code et à réduire le changement de contexte.

## Conclusion

À ce stade, vous devriez avoir Go installé, un éditeur de code prêt, et quelques outils de base en place. Go a un très large écosystème de produits tiers. Nous en avons identifié quelques uns utiles ici. Pour une liste plus complète, voir [https://awesome-go.com](https://awesome-go.com).
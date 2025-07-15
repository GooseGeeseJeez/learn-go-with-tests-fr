# Contribuer

Les contributions sont très bienvenues. J'espère que ce projet deviendra un excellent endroit pour les guides sur l'apprentissage de Go en écrivant des tests. N'hésitez pas à soumettre une PR (_Pull Request_) ou à créer une *issue*, ce que vous pouvez faire [ici](https://github.com/quii/learn-go-with-tests/issues).

**Note** : le lien ci-dessus renvoie à la version originale (en Anglais). Pour contribuer au repository qui contient le texte en Français, ça se passe [juste ici](https://github.com/GooseGeeseJeez/learn-go-with-tests-fr).

## Ce que nous recherchons

* Enseigner les fonctionnalités de Go \(par exemple, des choses comme `if`, `select`, les structs, les méthodes, etc.\).
* Mettre en valeur des fonctionnalités intéressantes de la bibliothèque standard. Montrer à quel point il est facile de faire du TDD pour un serveur HTTP par exemple.
* Montrer comment les outils de Go, comme le benchmarking, les détecteurs de course, etc., peuvent vous aider à créer d'excellents logiciels.

Si vous ne vous sentez pas suffisamment confiant pour soumettre votre propre guide, soumettre une *issue* pour quelque chose que vous voulez apprendre reste une contribution précieuse.

### ⚠️ Obtenez rapidement des retours pour les nouveaux contenus ⚠️

- Le TDD nous apprend à travailler de manière itérative et à obtenir des retours, et je vous suggère fortement de faire de même si vous souhaitez contribuer
    - Ouvrez une PR avec votre premier test et implémentation, discutez de votre approche afin que je puisse vous offrir des retours et corriger le cap
- C'est bien sûr open-source mais j'ai des opinions fortes sur le contenu. Plus tôt vous me parlerez, mieux ce sera.

## Guide de style

* Veillez à toujours renforcer le cycle TDD. Jetez un œil au [Modèle de Chapitre](template.md).
* Mettez l'accent sur l'itération des fonctionnalités guidées par les tests. L'exemple _Bonjour, Monde_ fonctionne bien car nous le rendons progressivement plus sophistiqué et apprenons de nouvelles techniques _guidées_ par les tests. Par exemple :
  * `Bonjour()` &lt;- comment écrire des fonctions, les types de retour.
  * `Bonjour(name string)` &lt;- arguments, constantes.
  * `Bonjour(name string)` &lt;- par défaut à "world" en utilisant `if`.
  * `Bonjour(name, language string)` &lt;- `switch`.
* Essayez de minimiser la surface de connaissances requises.
  * Il est important de trouver des exemples qui mettent en valeur ce que vous essayez d'enseigner sans confondre le lecteur avec d'autres fonctionnalités.
  * Par exemple, vous pouvez apprendre les `struct`s sans comprendre les pointeurs.
  * La concision est primordiale.
* Suivez le [guide de style Code Review Comments](https://go.dev/wiki/CodeReviewComments). C'est important pour un style cohérent dans toutes les sections.
* Votre section devrait avoir une application exécutable à la fin \(par exemple `package main` avec une fonction `main`\) afin que les utilisateurs puissent la voir en action et jouer avec.
* Tous les tests doivent passer.
* Exécutez `./build.sh` avant de soumettre une PR.
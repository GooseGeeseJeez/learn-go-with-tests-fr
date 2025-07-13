# Apprendre Go avec les tests

<p align="center">
  <img src="red-green-blue-gophers-smaller.png" />
</p>

[Art par Denise](https://twitter.com/deniseyu21)

[![Go Report Card](https://goreportcard.com/badge/github.com/quii/learn-go-with-tests)](https://goreportcard.com/report/github.com/quii/learn-go-with-tests)

## Formats

### En Français
- [Gitbook](https://goosegeesejeez.gitbook.io/apprendre-go-par-les-tests/)

### En Anglais
- [Gitbook original](https://quii.gitbook.io/learn-go-with-tests)
- [EPUB ou PDF](https://github.com/quii/learn-go-with-tests/releases)

## Soutenez-moi

Je suis fier d'offrir cette ressource gratuitement, mais si vous souhaitez montrer votre appréciation :

- [Tweetez-moi @quii](https://twitter.com/quii)
- <a rel="me" href="https://mastodon.cloud/@quii">Mastodon</a>
- [Offrez-moi un café :coffee:](https://www.buymeacoffee.com/quii)
- [Sponsorisez-moi sur GitHub](https://github.com/sponsors/quii)

## Pourquoi

* Explorer le langage Go en écrivant des tests
* **Acquérir une base solide avec le TDD**. Go est un bon langage pour apprendre le TDD car c'est un langage simple à apprendre et les tests sont intégrés
* Être confiant dans votre capacité à commencer à écrire des systèmes robustes et bien testés en Go
* [Regardez une vidéo, ou lisez pourquoi les tests unitaires et le TDD sont importants](why.md)

## Table des matières

### Les fondamentaux de Go

1. [Installer Go](install-go.md) - Configurer l'environnement pour la productivité.
2. [Bonjour, monde](hello-world.md) - Déclaration de variables, constantes, instructions if/else, switch, écrivez votre premier programme Go et votre premier test. Syntaxe des sous-tests et closures.
3. [Entiers](integers.md) - Explorer davantage la syntaxe de déclaration de fonction et apprendre de nouvelles façons d'améliorer la documentation de votre code.
4. [Itération](iteration.md) - Apprendre `for` et le benchmarking.
5. [Tableaux et slices](arrays-and-slices.md) - Apprendre les tableaux, slices, `len`, varargs, `range` et la couverture de tests.
6. [Structs, méthodes et interfaces](structs-methods-and-interfaces.md) - Apprendre `struct`, méthodes, `interface` et les tests dirigés par table.
7. [Pointeurs et erreurs](pointers-and-errors.md) - Apprendre les pointeurs et les erreurs.
8. [Maps](maps.md) - Apprendre le stockage de valeurs dans la structure de données map.
9. [Injection de dépendances](dependency-injection.md) - Apprendre l'injection de dépendances, comment elle se rapporte à l'utilisation d'interfaces et une introduction à io.
10. [Mocking](mocking.md) - Prendre du code existant non testé et utiliser l'ID avec le mocking pour le tester.
11. [Concurrence](concurrency.md) - Apprendre comment écrire du code concurrent pour rendre votre logiciel plus rapide.
12. [Select](select.md) - Apprendre comment synchroniser élégamment des processus asynchrones.
13. [Reflection](reflection.md) - Apprendre la reflection
14. [Sync](sync.md) - Apprendre certaines fonctionnalités du package sync incluant `WaitGroup` et `Mutex`
15. [Context](context.md) - Utiliser le package context pour gérer et annuler des processus longs
16. [Introduction aux tests basés sur les propriétés](roman-numerals.md) - Pratiquer le TDD avec le kata des chiffres romains et avoir une brève introduction aux tests basés sur les propriétés
17. [Mathématiques](math.md) - Utiliser le package `math` pour dessiner une horloge SVG
18. [Lire des fichiers](reading-files.md) - Lire des fichiers et les traiter
19. [Templating](html-templates.md) - Utiliser le package html/template de Go pour rendre du HTML à partir de données, et apprendre les tests d'approbation
20. [Generics](generics.md) - Apprendre comment écrire des fonctions qui prennent des arguments génériques et créer votre propre structure de données générique
21. [Revisiter les tableaux et slices avec les generics](revisiting-arrays-and-slices-with-generics.md) - Les generics sont très utiles quand on travaille avec des collections. Apprendre à écrire votre propre fonction `Reduce` et nettoyer quelques patterns communs.

### Construire une application

Maintenant que vous avez (espérons-le) digéré la section _Fondamentaux de Go_, vous avez une base solide de la majorité des fonctionnalités du langage Go et comment faire du TDD.

Cette section suivante impliquera de construire une application.

Chaque chapitre itérera sur le précédent, étendant les fonctionnalités de l'application selon les directives de notre product owner.

De nouveaux concepts seront introduits pour aider à faciliter l'écriture de bon code, mais la plupart du nouveau matériel sera d'apprendre ce qui peut être accompli à partir de la bibliothèque standard de Go.

À la fin de ceci, vous devriez avoir une compréhension solide de comment écrire itérativement une application en Go, soutenue par des tests.

* [Serveur HTTP](http-server.md) - Nous créerons une application qui écoute les requêtes HTTP et y répond.
* [JSON, routage et embedding](json.md) - Nous ferons en sorte que nos endpoints retournent du JSON et explorerons comment faire du routage.
* [IO et tri](io.md) - Nous persisterons et lirons nos données depuis le disque et couvrirons le tri des données.
* [Ligne de commande et structure de projet](command-line.md) - Supporter plusieurs applications depuis une base de code et lire l'entrée depuis la ligne de commande.
* [Time](time.md) - utiliser le package `time` pour programmer des activités.
* [WebSockets](websockets.md) - apprendre comment écrire et tester un serveur qui utilise les WebSockets.

### Fondamentaux des tests

Couvrir d'autres sujets autour des tests.

* [Introduction aux tests d'acceptation](intro-to-acceptance-tests.md) - Apprendre comment écrire des tests d'acceptation pour votre code, avec un exemple du monde réel pour arrêter gracieusement un serveur HTTP
* [Mettre à l'échelle les tests d'acceptation](scaling-acceptance-tests.md) - Apprendre des techniques pour gérer la complexité d'écrire des tests d'acceptation pour des systèmes non-triviaux.
* [Travailler sans mocks, stubs et spies](working-without-mocks.md) - Apprendre comment utiliser des fakes et des contrats pour créer des tests plus réalistes et maintenables.
* [Checklist de refactorisation](refactoring-checklist.md) - Quelques discussions sur ce qu'est la refactorisation, et quelques conseils de base sur comment la faire.

### Questions et réponses

Je rencontre souvent des questions sur internet comme

> Comment teste-t-on ma fonction incroyable qui fait x, y et z

Si vous avez une telle question, créez une issue sur github et j'essaierai de trouver le temps d'écrire un court chapitre pour traiter le problème. Je pense que du contenu comme celui-ci est précieux car il traite des questions _réelles_ des gens autour des tests.

* [OS exec](os-exec.md) - Un exemple de comment nous pouvons atteindre l'OS pour exécuter des commandes pour récupérer des données et garder notre logique métier testable.
* [Types d'erreur](error-types.md) - Exemple de création de vos propres types d'erreur pour améliorer vos tests et rendre votre code plus facile à utiliser.
* [Context-aware Reader](context-aware-reader.md) - Apprendre comment faire du TDD en augmentant `io.Reader` avec l'annulation. Basé sur [Context-aware io.Reader pour Go](https://pace.dev/blog/2020/02/03/context-aware-ioreader-for-golang-by-mat-ryer)
* [Revisiter les gestionnaires HTTP](http-handlers-revisited.md) - Tester les gestionnaires HTTP semble être le fléau de l'existence de nombreux développeurs. Ce chapitre explore les problèmes autour de la conception correcte des gestionnaires.

### Meta / Discussion

* [Pourquoi les tests unitaires et comment les faire fonctionner pour vous](why.md) - Regardez une vidéo, ou lisez pourquoi les tests unitaires et le TDD sont importants
* [Anti-patterns](anti-patterns.md) - Un court chapitre sur les anti-patterns TDD et tests unitaires

## Contribuer

* _Ce projet est un travail en cours_ Si vous souhaitez contribuer, n'hésitez pas à me contacter.
* Lisez [contributing.md](https://github.com/quii/learn-go-with-tests/tree/842f4f24d1f1c20ba3bb23cbc376c7ca6f7ca79a/contributing.md) pour les directives
* Des idées ? Créez une issue

## Contexte

J'ai de l'expérience dans l'introduction de Go aux équipes de développement et j'ai essayé différentes approches sur comment faire grandir une équipe de quelques personnes curieuses de Go vers des rédacteurs très efficaces de systèmes Go.

### Ce qui n'a pas marché

#### Lire _le_ livre

Une approche que nous avons essayée était de prendre [le livre bleu](https://www.amazon.co.uk/Programming-Language-Addison-Wesley-Professional-Computing/dp/0134190440) et chaque semaine discuter du chapitre suivant avec les exercices.

J'adore ce livre mais il requiert un haut niveau d'engagement. Le livre est très détaillé dans l'explication des concepts, ce qui est évidemment génial mais cela signifie que le progrès est lent et régulier - ce n'est pas pour tout le monde.

J'ai trouvé que bien qu'un petit nombre de personnes lisent le chapitre X et fassent les exercices, beaucoup de personnes ne le faisaient pas.

#### Résoudre quelques problèmes

Les katas sont amusants mais ils sont habituellement limités dans leur portée pour apprendre un langage ; vous êtes peu susceptible d'utiliser des goroutines pour résoudre un kata.

Un autre problème est quand vous avez différents niveaux d'enthousiasme. Quelques personnes apprennent juste beaucoup plus du langage que d'autres et quand elles démontrent ce qu'elles ont fait, elles finissent par confondre les personnes avec des fonctionnalités que les autres ne connaissent pas.

Cela finit par rendre l'apprentissage assez _non structuré_ et _ad hoc_.

### Ce qui a marché

De loin, le moyen le plus efficace était d'introduire lentement les fondamentaux du langage en lisant à travers [go by example](https://gobyexample.com/), les explorer avec des exemples et les discuter en groupe. C'était une approche plus interactive que "lisez le chapitre x comme devoir".

Au fil du temps, l'équipe a acquis une base solide de la _grammaire_ du langage pour que nous puissions ensuite commencer à construire des systèmes.

Ceci me semble analogue à pratiquer les gammes quand on essaie d'apprendre la guitare.

Peu importe à quel point vous pensez être artistique, vous êtes peu susceptible d'écrire de la bonne musique sans comprendre les fondamentaux et pratiquer les mécaniques.

### Ce qui marche pour moi

Quand _j'_ apprends un nouveau langage de programmation, je commence habituellement par jouer dans un REPL mais finalement, j'ai besoin de plus de structure.

Ce que j'aime faire, c'est explorer les concepts et ensuite solidifier les idées avec des tests. Les tests vérifient que le code que j'écris est correct et documentent la fonctionnalité que j'ai apprise.

En prenant mon expérience d'apprentissage avec un groupe et ma propre façon personnelle, je vais essayer de créer quelque chose qui j'espère s'avèrera utile pour d'autres équipes. Apprendre les fondamentaux en écrivant de petits tests pour que vous puissiez ensuite prendre vos compétences existantes de design de logiciel et livrer quelques grands systèmes.

## Pour qui c'est

* Les personnes qui sont intéressées à apprendre Go.
* Les personnes qui connaissent déjà un peu Go, mais veulent explorer les tests avec le TDD.

## Ce dont vous aurez besoin

* Un ordinateur !
* [Go installé](https://golang.org/)
* Un éditeur de texte
* Un peu d'expérience avec la programmation. Comprendre des concepts comme `if`, variables, fonctions etc.
* À l'aise avec l'utilisation du terminal

## Feedback

* Ajoutez des issues/soumettez des PRs [ici](https://github.com/quii/learn-go-with-tests) ou [tweetez-moi @quii](https://twitter.com/quii)

[Licence MIT](LICENSE.md)

[Le logo est par egonelbre](https://github.com/egonelbre) Quelle star !
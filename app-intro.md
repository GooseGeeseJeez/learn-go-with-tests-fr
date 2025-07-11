# Construire une application

Maintenant que vous avez, espérons-le, assimilé la section _Fondamentaux de Go_, vous possédez une solide connaissance de la majorité des fonctionnalités du langage Go et savez comment pratiquer le TDD.

Cette prochaine section consistera à construire une application.

Chaque chapitre s'appuiera sur le précédent, en élargissant les fonctionnalités de l'application selon les directives de notre product owner.

De nouveaux concepts seront introduits pour vous aider à écrire du code de qualité, mais la plupart des nouveaux éléments seront l'apprentissage de ce que permet la bibliothèque standard de Go.

À la fin de cette section, vous devriez avoir une bonne compréhension de la façon d'écrire itérativement une application en Go, soutenue par des tests.

- [Serveur HTTP](http-server.md) - Nous créerons une application qui écoute les requêtes HTTP et y répond.
- [JSON, routage et intégration](json.md) - Nous ferons en sorte que nos points de terminaison renvoient du JSON et explorerons comment faire du routage.
- [IO et tri](io.md) - Nous persisterons et lirons nos données depuis le disque et nous aborderons le tri des données.
- [Ligne de commande et structure de projet](command-line.md) - Prise en charge de plusieurs applications à partir d'une même base de code et lecture des entrées à partir de la ligne de commande.
- [Temps](time.md) - utilisation du package `time` pour planifier des activités.
- [WebSockets](websockets.md) - apprenez à écrire et à tester un serveur qui utilise les WebSockets.
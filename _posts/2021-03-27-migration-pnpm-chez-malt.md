---
layout: post
title:  "Migration PNPM chez Malt"
tags: ['tech', 'npm', 'pnpm', 'malt']
---

![PNPM](/static/blog/pnpm/pnpm.png)

Chez Malt, nous avons une codebase gérée dans un mono-repo (on ne reviendra pas sur les avantages et inconvénients de ce choix, ce n’est pas le but de ce post) et nous avons une stack composée de modules Maven pour la partie back et de modules NPM pour la partie front.
Au début de l’histoire de Malt, nous utilisions uniquementNPM pour gérer les dépendances, mais tout a commencé à se complexifier dès lors que l’on a voulu partager du code entre nos différentes applications.

En 2018, nous avions donc regardé les alternatives supportant les "workspaces" et nous avions fait le choix d’utiliser [Yarn](https://classic.yarnpkg.com/lang/en/) car cela répondait très bien à notre cas d’usage de l’époque (nous avions aussi évalué NPM + [Lerna](https://lerna.js.org/), mais c’est un autre sujet).

Début 2021, nous avons migré de [Yarn](https://classic.yarnpkg.com/lang/en/) vers [PNPM](https://pnpm.js.org/). En voici les raisons et ce que nous avons appris 🙂

### Pourquoi avoir choisi Yarn ?

Comme je l’ai expliqué, en 2018, nous avons fait le choix de remplacer NPM par [Yarn](https://classic.yarnpkg.com/lang/en/). Ce choix a principalement été guidé par le fait que Yarn gère des "workspaces", nous permettant de facilement gérer les dépendances "internes" au sein de notre mono-repo.

Très vite, cela a permis de nous faciliter la vie au quotidien :
- Le setup d’un nouvelle environnement de dev s’est retrouvé nettement simplifié, cela se résumait par deux commandes : `npm install -g yarn && yarn install`
- L’ajout d’un nouveau module dans notre codebase est devenu très simple aussi : on déclare le workspace yarn dans le fichier `package.json`, on fait un `yarn install`, et c’est parti !

Malheureusement, au fil du temps, cela est devenu de plus en plus douloureux au quotidien :
- Des soucis de performance, notamment sur notre CI, qui a commencé à vraiment devenir un problème au cours de l’année 2020.
- Des effets de bord malencontreux liés au "hoisting" des dépendances (je reviendrai dessus plus en détail dans la partie suivante).
- Une migration vers [Yarn v2](https://yarnpkg.com/) (a.k.a [Yarn Berry](https://yarnpkg.com/)) qui allait être inévitable pour ne pas garder un outil voué à être déprécié à l’avenir.

Mais avant de parler de la migration [PNPM](https://pnpm.js.org/), revenons plus en détail sur les problèmes que l’on a rencontré avec Yarn.

### Hoisting des dépendances

Afin de partager "intelligemment" les différentes dépendances communes au sein du workspace, Yarn utilise de manière astucieuse la résolution de dépendances définie dans Node.js (et implémentée dans tous les outils qu’on utilise comme [Webpack](https://webpack.js.org/), [Rollup](https://rollupjs.org/guide/en/), etc.).

Pour faire simple, cela fonctionne de la façon suivante :

1. Supposons qu’on ait besoin d’utiliser [lodash](https://lodash.com/), on va donc déclarer la dépendance dans notre fichier pour pouvoir l’utiliser : `import _ from 'lodash'` (ou `const _ = require('lodash')` si vous n’utilisez pas les imports ESM).
2. Lorsque Webpack construit notre bundle, il va donc essayer de résoudre la dépendance :
- Il va commencer par chercher le chemin vers `node_modules/lodash` à partir du fichier source.
- S’il n’existe pas, il va remonter au répertoire parent et donc chercher si `../node_modules/lodash` existe.
- Il remonte récursivement jusqu’à trouver la dépendance, ou échouera si elle n’existe nulle part.

Pour installer les dépendances, Yarn fait donc le choix de les placer à la racine du workspace pour éviter de dupliquer les dépendances dans chaque dossier `node_modules` de nos packages (économisant donc des copies de fichiers sur disque).

Si on résume avec un schéma, cela donne (à peu près) quelque chose comme ça :

![Hiérarchie des Node Modules](/static/blog/pnpm/node_modules.png)

J’ai vraiment simplifié ici pour bien comprendre :

- Notre mono-repo (identifié ici par `"@malt"`) contient trois modules : `"@malt/app1"`, `"@malt/app2"` et `"@malt/app3"`
- Nos trois modules dépendent de `"lodash@4.17.21"`
- Deux modules `"@malt/app2"` et `"@malt/app3"` dépendent de `"moment@2.29.1"`

Comme je l’expliquais précédemment, Yarn va "hoister" nos dépendances en les plaçant à la racine du repo, et c’est assez facile de comprendre pourquoi cela fonctionne une fois qu’on a défini l’algorithme de résolution.

Cela pose toutefois un problème : la dépendance `"moment@2.29.1"` est maintenant accessible au module `"@malt/app1"`, alors qu’il ne l’a pas déclarée dans son fichier `package.json` (rappelez vous, l’algorithme de résolution va chercher récursivement dans les répertoires parents) !

Imaginez maintenant que nos deux modules migrent de `"moment@2.29.1"` à `"date-fns"`, et vous avez un effet de bord sur `"@malt/app1"` qui n’aura plus accès à `"moment"`...

Encore pire : imaginez qu’on décide d’upgrade moment vers une nouvelle version majeure : vous risquez de casser l’application `"@malt/app1"` sans même vous en rendre compte 😱

A cela, on peut répondre deux choses :
- Au quotidien, il suffit "juste" d’être vigilant (dans les code reviews, etc.). Je trouve que le risque d’erreur reste élevé, et devient encore plus grand quand on déplace beaucoup de code, [comme cela nous est arrivé récemment](https://medium.com/nerds-malt/supporting-a-product-team-reorg-with-a-code-reorg-24639aae8ddf).
- On pourrait configurer notre [linter](https://www.npmjs.com/package/eslint-plugin-implicit-dependencies) (ou autre) pour vérifier qu’on ne fait pas n’importe quoi, mais je trouve que ça rajoute du tooling en plus là où l’outil de gestion des dépendances devrait, de base, nous éviter ce genre de difficultés.

Bref, ce fonctionnement a des avantages, mais nous cause aussi de nombreux soucis.

### Performances

Je l’expliquais au début du post, nous avons, au fil des mois, rencontré de plus en plus de problèmes de performance sur notre CI.

Un simple `yarn install` est devenu de plus en plus coûteux, comme vous pouvez le voir sur le graphe qui suit.
Le build front était devenu un vrai point de douleur :

![Temps de build](/static/blog/pnpm/build1.png)

On voit sur ce graphe que nos applications se "buildent" en moyenne en 10 à 12 minutes (pour être très précis : ce graphe inclut le build front et back), ce qui est assez long pour être pénible au quotidien.

*Remarque : je n’ai pas de graphes pour différencier temps de build back vs temps de build front, mais nos analyses ont montré que l’étape yarn install nous coûtait très cher… (petit spoiler : cela sera confirmé par le même graphe que je donne dans le bilan).*

On a donc commencé à chercher une alternative nous permettant de résoudre ces deux points.

### Make a choice

Fin 2020, nous avons commencé à réfléchir à une autre solution, et nous avons identifié trois successeurs :
- [NPM 7, avec le support des workspaces](https://docs.npmjs.com/cli/v7/using-npm/workspaces).
- Yarn v2 et son support par défaut de [PnP](https://yarnpkg.com/features/pnp) (pour être précis : PnP était [déjà utilisable avec yarn V1](https://classic.yarnpkg.com/en/docs/pnp/))
- PNPM

Pour ce genre de choix, ayant un impact sur l’ensemble de l’équipe de dev, nous avons pesé le pour et le contre de chaque outil et nous en avons discuté via l’ouverture d’une issue GitHub :

![GitHub Discussion](/static/blog/pnpm/gh-issue.png)

Après quelques jours de discussion, le choix s’est porté au final sur… PNPM !

### Pourquoi PNPM ?

Si je reprends la documentation de PNPM, on peut lire ceci :

> Files inside node_modules are linked from a single content-addressable storage

À l’inverse de Yarn ou NPM 7, PNPM a fait le choix de ne pas "hoister" (par défaut) les dépendances : il va plutôt gérer des [hard links](https://en.wikipedia.org/wiki/Hard_link) dans chaque dossier `node_modules`, faisant pointer chaque dépendance vers un emplacement unique.

Reprenons notre schéma précédent pour mieux comprendre son fonctionnement :

![Hiérarchie des Nodes Modules](/static/blog/pnpm/node_modules_pnpm.png)

Ce schéma peut paraître assez compliqué à première vue, mais il est en fait très simple à comprendre :
- Chaque package contient son propre dossier `node_modules` avec ses dépendances.
- Chaque dépendance, dans chaque package NPM est en fait un hard link vers `<root>/node_modules/.pnpm/<dépendance>`

Ce fonctionnement (très malin) a deux avantages principaux :
- Chaque application ne peut utiliser que les dépendances déclarées dans son fichier `package.json`.
- Les dépendances ne sont pas dupliquées sur le disque, on a à l’inverse des créations de hard links vers un emplacement unique.

On résout donc trois problèmes :
- Les problèmes amenés par le hoisting de dépendances sont résolus de facto, on n’a plus besoin de tooling supplémentaire pour vérifier nos imports dans le code.
- Les applications ne peuvent plus utiliser de dépendances transitives ([la hiérarchie des `node_modules` mise en place par PNPM ne permet plus ça](https://pnpm.js.org/blog/2020/05/27/flat-node-modules-is-not-the-only-way/)).
- Les performances sont très (très) bonnes, créer / supprimer des hard links n’est pas du tout un problème !

*Remarque en passant comme ça : le fonctionnement de Yarn PnP est aussi très intéressant, mais on l’a exclu de nos choix car cela impliquait aussi de changer le build de nos applications pour supporter cette approche (il y a un plugin Webpack pour ça). Bref, ça rajoutait du tooling, [encore](https://medium.com/@ericclemmons/javascript-fatigue-48d4011b6fc4)...*

Dans notre contexte, nous avons choisi PNPM au vu des trois problèmes résolus et du setup relativement simple.

La migration de Yarn vers PNPM n’a pas été si compliquée que ça, elle nous a en fait permis de corriger beaucoup (vraiment beaucoup) de problèmes de dépendances ayant été introduits dans notre codebase au fil de l’eau ces deux dernières années (au final, on a juste corrigé des soucis qui auraient pu casser nos applications un jour ou l’autre).

### Bilan

Nous utilisons maintenant PNPM au quotidien depuis environ deux mois. Cela a considérablement réduit nos temps de builds sur notre CI :

![Temps de build](/static/blog/pnpm/build2.png)

Nous sommes passés sur PNPM le 9 février, et comme on peut le constater sur ce graphe, nos temps de build ont considérablement réduit (de 11 minutes en moyenne à 7 minutes) !

On a aussi résolu le problème de dépendances : désormais il n’est plus possible d’utiliser une dépendance transitive ou non déclarée !
À titre personnel, je suis un utilisateur comblé de PNPM : l’outil est vraiment excellent. Au cours de la migration, j’ai progressivement eu ce sentiment étrange : "mais pourquoi NPM ne fonctionne pas comme ça depuis le départ ?" ou "pourquoi le support des workspace dans NPM 7 ne fonctionne pas comme ça ?".

Je ne dis pas que PNPM fonctionnera dans tous les contextes (et d’ailleurs, si vous n’avez pas besoin des workspaces, NPM ou Yarn fonctionneront probablement très bien), mais clairement, chez Malt, cela a été un vrai "game changer" sur la stack front 🔥

Et si vous n’êtes pas d’accord, ou que vous souhaitez participer à l’amélioration de notre stack, [n’hésitez pas à venir nous voir](https://careers.malt.com/careers.html) :)

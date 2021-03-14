---
layout: post
title:  "Prismic @ Malt"
tags: ['tech', 'prismic', 'malt']
---

*Cet article a initialement été publié sur [medium](https://medium.com/nerds-malt/prismic-malt-2f59b96c18a9).*

![Prismic @ Malt](/static/assets/prismic-malt.png)

Au mois d’octobre 2020, nous avons lancé la nouvelle identité de Malt, cela a été l’occasion de repenser la nouvelle charte graphique (les équipes Brand & Design de Malt pourront vous parler bien mieux que moi de tous les challenges associés), mais aussi le contenu.

Ce projet n’a pas seulement été l’occasion d’implémenter notre nouveau Design System, cela a été aussi l’occasion d’introduire dans notre produit un CMS Headless, à savoir [Prismic](https://prismic.io).

Voici donc un peu plus de détails sur la mise en place de cet outil et ce que nous avons appris.

### Why ?

Avant de parler technique, la question essentielle est : pourquoi avoir choisi de mettre en place un CMS Headless ?

Pour répondre à cette question, j’ai besoin de revenir un peu en arrière pour parler du contexte à l’époque de ce choix.
Si on revient au tout début de l’année 2020, nous avons sur le site de Malt plusieurs landing pages, dont le contenu est géré par l’équipe Marketing. À l’époque où toutes ces pages sont créées (en 2018–2019), nous n’avons pas de CMS, le choix est donc d’aller au plus simple et au plus vite : nous développons des pages (quasiment) statiques, traduites dans diverses langues correspondant aux déclinaisons internationales du site (Espagnol, Allemand, etc.).

Je dis "quasiment" statiques car le contenu étant internationalisé, celui-ci est géré via notre outil de gestion de traduction, à savoir [Phraseapp](https://phrase.com/fr/), auquel les équipes marketing ont accès.

Avoir des landing pages "statiques" a des avantages (rapidité de mise en place, gestion des évolutions côté équipe tech, etc.) mais le process de mise à jour du contenu est assez fastidieux :

1. L’équipe Marketing fait ses modifications dans [Phraseapp](https://phrase.com/fr/), pour ça ils ont besoin de connaître précisément la clé de traduction à modifier qui n’est pas toujours évidente à connaître.
2. Une fois les modifications faites, la mise à jour en production passe généralement par un ticket Jira pour demander l’update (récupération des traductions + tests sur notre environnement d’intégration).
3. Déploiement en production.

Entre la modification initiale et la mise à jour en production, il peut s’écouler entre une demi-journée et deux jours suivant la réactivité de l’équipe produit, [ce qui est bien mais pas top](https://youtu.be/b6ycU7Tsnzk?t=72) :)
Nous avons donc décidé de mettre en place un CMS Headless pour rendre nos équipes marketing autonomes sur le contenu et leur laisser la main sur la publication en production.

La mise en place de ce CMS va donc avoir les objectifs suivants :
- Rendre les équipes de Malt plus autonomes : la modification du contenu ne doit pas faire intervenir les équipes produits, et la mise à jour en production doit pouvoir se faire en quelques minutes.
- Créer du nouveau contenu doit être simple pour tout le monde, même sans connaissance technique.

### What ?

Afin de faire évoluer notre produit, nous avons introduit un nouveau [produit dans le produit](https://www.imdb.com/title/tt1375666/), à savoir [Prismic](https://prismic.io). Voyons un peu plus dans le détail pourquoi nous l’avons choisi.

Prismic est un CMS Headless : si vous ne savez pas ce qu’est un CMS Headless, on peut le résumer simplement comme étant un outil d’édition de contenu nous donnant accès à une API pour les documents que nous éditons. Cela se différencie d’un CMS type Wordpress, car dans le cas de Prismic, nous consommons uniquement l’API et le rendu se fait intégralement dans notre codebase (avec nos templates que nous écrivons).

Lorsque nous avons évalué les CMS Headless du marché, trois outils sont sortis du lot :
- [Prismic](https://prismic.io/)
- [Contentful](https://www.contentful.com/)
- [Strapi](https://strapi.io/)

Nous avons éliminé [Strapi](https://strapi.io/) quasiment d’emblée car nous souhaitions un outil SaaS (sans jugement négatif, ça ne correspondait juste pas à notre besoin). Le choix s’est donc concentré sur [Contentful](https://www.contentful.com/) vs [Prismic](https://prismic.io/) et pour cela nous avons élaboré quelques critères qui nous paraissaient primordiaux :
- Un pricing que l’on peut anticiper.
- Gestion de l’internationalisation et des localisations : la solution choisie a besoin d’être compatible avec tous nos sites localisés (www.malt.fr, en.malt.fr, www.malt.de, en.malt.de, etc.).
- Clients disponibles : idéalement un SDK "natif” ou au minimum une API Rest qu’on peut interroger facilement.

Ces trois critères ont guidé notre choix vers [Prismic](https://prismic.io/) :
- Le pricing est clair : un abonnement annuel, pas de facturation en fonction de la consommation.
- La gestion de l’internationalisation qui correspond exactement à ce que l’on recherche.
- Une API Rest, et même GraphQL, assez simple à comprendre et utiliser.

### How ?

Reste maintenant la partie tech, à savoir la mise en place de Prismic dans notre codebase.

#### Comment interroger l’API ?

Pour l’intégrer au mieux dans notre codebase, il nous apparaît clairement que nous devons l’adapter à notre stack Java. À l’inverse, adapter notre stack à la mise en place de Prismic nous parait contre-productif et risqué (sans rentrer dans les détails, cela demande de revoir certaines "briques” clés dans notre architecture).

Prismic met à disposition des SDK dans différents langages, mais malheureusement le SDK Java semble plus ou moins à l’abandon ; la priorité étant mise sur le SDK JS à utiliser dans une app React / VueJS / etc. Utiliser ce type de fonctionnement parait fun au premier abord (frameworks SPA + Server Side Rendering entre autres), mais cela nous demande pas mal de modifications pour l’intégrer proprement à nos applications Spring Boot.

On décide donc de ré-écrire notre propre client consommant l’API Rest de Prismic. Fort heureusement, écrire un client Rest est quelque chose qu’on sait plutôt bien faire chez Malt, et cela ne nous prend que très peu de temps :)

#### Mise en place

Afin de bien s’intégrer avec le fonctionnement de [Prismic](https://prismic.io) ([Slice](https://user-guides.prismic.io/en/articles/383933-slices), etc.), on décide de garder l’approche "Component”, mais pour cela nous n’allons pas utiliser de frameworks JS orientés Composants (Angular, React, VueJS, etc.).

Nous allons plutôt prendre cette approche :
- Pour chaque slice Prismic*, nous avons un template handlebars qui lui correspond.
- Pour chaque slice Prismic*, nous définissons le style qui lui correspond (un fichier sass pour chaque composant en gros), pour lequel nous "scopons” les styles en utilisation la notation [BEM](https://css-tricks.com/bem-101/).
- Éventuellement, nous avons la possibilité d’ajouter un peu de JS pour nos composants Prismic, mais cela s’avérera très (très) rare, chaque composant ayant un rendu purement statique ne nécessitant pas / peu de logique côté browser. En somme, on fait du "progressive enhancement” 🙂

_*Dans Prismic, un slice peut être vu comme composant de plus "haut niveau” (haut niveau, dans le sens "pas un champ de saisie”), typiquement, une section de contenu dans une landing page._

En faisant le choix de ne pas utiliser le [SDK Java de Prismic](https://github.com/prismicio/java-kit), nous devons ré-implémenter certaines choses (forcément, le SDK ne se résume pas seulement à un client HTTP). L’utilisation d’Handlebars nous permet de centraliser certaines parties sous forme de Helper de manière assez souple :
- Nous ré-implémentons le fonctionnement du ["RichText"](https://user-guides.prismic.io/en/articles/383762-rich-text) Prismic. Ce sera fait sous la forme d’un [helper Handlebars](https://jknack.github.io/handlebars.java/helpers.html) que l’ont pourra utiliser partout.
- Nous ré-implémentons également le fonctionnement des ["Links"](https://user-guides.prismic.io/en/articles/383950-link) Prismic sous forme de [Helper](https://jknack.github.io/handlebars.java/helpers.html).

Enfin, nous mettons en place from scratch le fonctionnement des Preview. Pour l’anecdote, cela m’amènera à remonter un problème de sécurité. Heureusement, l’équipe Support de Prismic est top, répond vite et m’a remercié par quelques Goodies ❤️

#### Résultat

Voici une vue générale de l’architecture que nous avons aujourd’hui en place :

![Prismic @ Malt](/static/assets/prismic-malt-stack.svg)

Nous avons donc :
- Une JSP incluant les templates Handlebars correspondant aux slices à afficher à l’écran (et déterminés dynamiquement par le document Prismic à afficher).
- Ces JSP sont servies par nos apps Spring Boot communiquant directement avec l’api Rest de Prismic.
- Un cache au milieu.

La mise en place du cache n’était pas obligatoire mais il y a ici deux objectifs principaux :
- Economiser les appels à Prismic (gain en perf pour afficher une page, [économie de bande passante](https://www.greenpeace.fr/la-pollution-numerique/), etc.)
- Avoir un fallback "dégradé” pour le cas où la communication avec Prismic tomberait. Bon, dans les faits ça n’est jamais arrivé : Prismic fonctionne terriblement bien (et super rapidement). Mais tout de même cela pourrait arriver, et [même aux meilleurs](https://www.theverge.com/2020/6/29/21306674/github-down-errors-outage-june-2020), et c’est aussi notre job de prévoir l’imprévisible.

Et grâce aux [webhooks](https://user-guides.prismic.io/en/articles/790505-webhooks) que l’on peut configurer dans Prismic, le cache est automatiquement invalidé à chaque publication d’un document, tout est donc automatisé 🔥

### Bilan

Cela va faire maintenant 4 mois que tout cela est en place, et on peut déjà en tirer un bilan :
- Globalement, les équipes Marketing sont devenues nettement plus autonomes. Il y a encore des améliorations à faire, tout n’est pas parfait non plus, mais aujourd’hui un changement de texte ne nécessite plus du tout l’intervention de l’équipe produit, et la mise à jour en prod se fait en quelques clics.
- Nous avons utilisé cette solution pour d’autres besoins que des landing pages marketing. Par exemple l’équipe Community publie régulièrement des super sélections de profils freelances en totale autonomie, par exemple la [sélection d’ingénieur devops](https://www.malt.fr/selection/devops) (aucune de ces pages n’a nécessité d’intervention de l’équipe produit).
- Nous avons aussi permis aux équipes Marketing de déployer [des pages entières sans aucune intervention de l’équipe produit](https://en.malt.de/c/malt-start), et ça c’était clairement une promesse difficile à croire au début :)

Personnellement, j’ai trouvé le produit Prismic vraiment excellent (et pourtant je faisais partie des sceptiques, probablement échaudé par les CMS type Wordpress & cie).

Et si vous voulez mettre les mains dans le cambouis pour améliorer notre solution, [n’hésitez pas à venir nous en parler](https://careers.malt.com/careers) !
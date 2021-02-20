---
layout: post
title:  "Encapsulation Everywhere"
---

### ➡️ Introduction

En tant que développeur, dès lors que l'on code une application / une librairie / un script / whatever, il nous arrive bien souvent d'utiliser des librairies externes, que ce soit un framework ou des librairies utilitaires.

Par exemple: supposons que vous développiez une application et que vous ayez besoin d'échapper les entrées utilisateurs pour se protéger de [failles XSS](https://en.wikipedia.org/wiki/Cross-site_scripting). Vous pourriez être tenté de ré-écrire votre propre fonction d'escaping mais finalement, pourquoi ré-écrire quelque chose déjà résolu ailleurs ?

En Java, par exemple, vous allez pouvoir utiliser:

- [Guava](https://github.com/google/guava) avec la classe [HtmlEscapers](https://guava.dev/releases/19.0/api/docs/com/google/common/html/HtmlEscapers.html).
- [Apache Commons Text](https://commons.apache.org/proper/commons-text/) avec la classe [StringEscapeUtils](https://commons.apache.org/proper/commons-text/javadocs/api-release/org/apache/commons/text/StringEscapeUtils.html)
- L'implémentation officielle de [l'Owasp](https://owasp.org/) [Owasp Java Encoder](https://owasp.org/www-project-java-encoder/)
- Et si vous utilisez [Spring Framework](https://spring.io/), vous avez déjà accès à la classe [HtmlUtils](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/util/HtmlUtils.html)
- Etc.

Du côté JavaScript, vous allez pouvoir utiliser:
- [Lodash](https://lodash.com/) et sa fonction [escape](https://lodash.com/docs/4.17.15#escape)
- [Underscore](https://underscorejs.org) et sa fonction [escape](https://underscorejs.org/#escape)
- Le package disponible sur [npm](https://www.npmjs.com/): [sanitize-html](https://www.npmjs.com/package/sanitize-html)
- Etc.

On voit que quelle que soit la plateforme / langage, les solutions ne manquent pas, l'idée maintenant sur votre codebase et de choisir la solution qui vous convient le mieux et de l'utiliser :)

### ➡️ Problèmes

Mais comment l'utiliser ? Simple : Suivant le langage choisi, on importe le package et on l'utilise !

Mais ceci présente un problème que vous ne voyez peut-être pas au premier abord : vous allez éparpiller un peu partout la solution choisie. Sur une codebase avec une taille raisonnable, cela pose peu de problème, mais quels sont les risques quand la codebase grossit ou bien que les équipes de devs augmentent en nombre ?

#### ⚠️ Inconsistences

En éparpillant l'utilisation de cette librairie, vous prenez le risque qu'un développeur arrivant dans votre équipe ne soit pas au fait de ce choix au départ et décide d'utiliser le premier utilitaire disponible dans votre codebase
- Parce que récupéré en tant que dépendance transitive,
- Ou présente pour une toute autre raison (rare sont les applications qui n'ont ni `lodash` ni `underscore`)

#### ⚠️ Et si on a besoin de changer de librairies ?

Il peut y avoir plusieurs raisons qui peuvent vous pousser à changer de librairie :
- Elle est dépréciée, voire n'est plus maintenue,
- Elle n'est plus compatible avec d'autres librairies / frameworks utilisées dans votre codebase,
- Elle n'est plus compatible avec votre environnement de production (version de jre / nodejs, OS, etc.),
- Changement de licence,
- Etc.

Bref, il peut y avoir beaucoup de raisons, et dans la plupart que je cite plus haut, rien ne vous empêche de :
- Forker la librairie et la maintenir / faire évoluer vous même, mais vous vous imposez de maintenir une librairie tierce en plus de votre code applicatif. Dans certains cas, ça marchera, dans d'autres vous n'aurez probablement pas assez de temps pour le faire correctement.
- Proposer une PR pour apporter les changements nécessaires, mais encore faut-il que la librairie soit open source, avec une licence compatible et que votre changement soit accepté.

#### ⚠️ Et s'il y a un breaking change ?

Autre cas qui peut devenir compliqué : et si la librairie que vous avez choisi décide d'introduire un breaking change ? Vous allez devoir re-passer partout où vous l'utilisez afin de faire les changements appropriés :
- Quand ils sont simple à gérer, ça passe généralement assez facilement.
- Sinon, vous avez devant vous quelques heures à passer sur cette migration (et en espérant que tout soit bien testé).

### ➡️ Encapsulation

Lorsqu'on on apprend la programmation orientée object, on nous avertit assez vite qu'il est important de bien respecter l'encapsulation sur nos classes (visibilité des attributs / méthodes, [Demeter Law](https://en.wikipedia.org/wiki/Law_of_Demeter), etc.).

Personnellement, je conseille d'appliquer cette même rigueur à vos dépendances en encapsulant le plus possible leur utilisation 🤓

Reprenons notre exemple précédent de fonction d'escaping, et supposons qu'on choisisse [Spring](https://spring.io) comme implémentation. Au début, ce choix est cohérent :
- On utilise déjà Spring comme framework un peu partout dans notre codebase
- Elle marche bien
- On n'a pas forcément envie d'ajouter encore une nouvelle dépendance

Et supposons qu'un an plus tard, vous vouliez utiliser cet utilitaire dans un autre contexte que [Spring](https://spring.io) (parce que vous aussi, vous avez découvert [Quarkus](https://quarkus.io/) et vous voulez tester ça en production 😄) : comme [Spring](https://spring.io) ne sera pas disponible dans votre nouvelle application, vous allez devoir utiliser une autre solution et vous vous retrouvez du coup avec deux librairies différentes pour faire la même chose... 😔

Et si vous aviez encapsulé votre dépendance avec ce genre d'implémentation :

```java
import org.springframework.web.util.HtmlUtils;

public final class Escapers {
  private Escapers() {
  }

  public static String escapeHtml(String input) {
    return HtmlUtils.escapeHtml(input);
  }
}
```

Si vous utilisez votre classe utilitaire `Escapers` dans votre codebase, vous avez **deux lignes à changer** pour faire évoluer votre implémentation vers [Apache Commons](https://commons.apache.org/) et votre fonction utilitaire devient utilisable partout dans votre codebase :

```diff
- import org.springframework.web.util.HtmlUtils;
+ import org.apache.commons.text.StringEscapeUtils;

public final class Escapers {
  private Escapers() {
  }

  public static String escapeHtml(String input) {
-     return HtmlUtils.escapeHtml(input);
+     return StringEscapeUtils.escapeHtml4(input);
  }
}
```

En bien sûr, vous ne ré-inventez pas la roue : vous continuez à déléguer l'implémentation à des librairies qui ont déjà résolu votre problème il y a longtemps et qui font ça très bien !

Je donne un exemple dans cet article avec une fonction d'escaping, mais vous pouvez faire le même type d'encapsultation pour presque tout utilitaire utilisé dans votre codebase, par exemple :

- [StringUtils](https://commons.apache.org/proper/commons-lang/apidocs/org/apache/commons/lang3/StringUtils.html)
- [NumberUtils](https://commons.apache.org/proper/commons-lang/apidocs/org/apache/commons/lang3/math/NumberUtils.html)
- Etc.

### ➡️ Conclusion

Cet article n'a pas pour but de vous dire d'encapsuler toutes vos dépendances, dans certains cas cela complexifiera beaucoup votre développement au quotidien. Mais il y a aussi une grande majorité de cas où mettre en place cette encapsulation au départ ne vous coûtera pas cher et vous en verrez les bénéfices plusieurs mois / années plus tard 😀

Donc, la prochaine fois que vous utilisez `lodash`, `underscore`, n'importe quel package utilitaire sur `npm`, `Apache Commons` ou `Guava`, pensez à ne pas faire leaké votre dépendance partout dans votre codebase 🙂

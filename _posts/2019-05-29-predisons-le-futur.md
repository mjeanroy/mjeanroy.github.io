---
layout: post
title:  "Prédisons le futur"
---

*Cet article a initialement été publié sur [medium](https://medium.com/nerds-malt/pr%C3%A9disons-le-futur-56c87ba1688b).*

Chez Malt, nous travaillons sur une stack côté back mélangeant plusieurs outils, avec entre autres une application backend en Java / SpringBoot et une base MongoDB (mais pour plus de détails sur les autres composants de la stack, n’hésitez pas à venir nous voir !).

Comme dans tout site public, nous avons eu à générer des tokens uniques et aléatoires pour certaines opérations sensibles.
Laissez moi vous raconter une histoire.

### Use Case

L’idée ici est de générer un token aléatoire et unique pour une action utilisateur, token devant expirer dans le temps (environ 1 heure de durée de vie).

Le développement de la feature s’est fait naturellement en utilisant les outils offerts à notre disposition (dans ce genre de cas, n’essayez pas de recoder ce genre de choses à la main, c’est un cas d’utilisation courant sans doute déjà résolu par des personnes très compétentes).

Nous sommes donc partis sur les fonctionnalités offertes par MongoDB, à savoir :
- Les tokens utilisateurs sont stockés dans MongoDB.
- MongoDB permettant l’utilisation d’un index TTL, la question de la durée de vie du token a été facilement résolue.
- Pour chaque document inséré, MongoDB va générer un identifiant de type `ObjectId` contenant une partie aléatoire. Nous sommes donc parti sur l’utilisation de cet `ObjectId` comme valeur de token (en faisant attention à ne jamais le rendre public d’une manière ou d’une autre).

Malheureusement, l’une de ces trois options a été un échec cuisant.

Ne laissons pas le suspense durer trop longtemps : un `ObjectId` utilisé comme token est une grossière erreur. Pour bien comprendre la raison, il nous faut rentrer un peu plus dans les détails de génération d’un `ObjectId` par MongoDB.

### Qu’est-ce qu’un `ObjectId` ?

Pour répondre à cette question, reprenons la documentation de MongoDB que vous pouvez retrouver [ici](https://docs.mongodb.com/manual/reference/method/ObjectId/) :

- Un `ObjectId` est un identifiant unique sur 12 octets.
- Les quatre premiers octets correspondent à un timestamp en secondes.
- Les cinq octets suivants représentent **une valeur aléatoire**.
- Les trois octets suivants correspondent à un incrément démarrant par **une valeur aléatoire**.

A priori, en lisant ça, on se dit qu’on est tranquille, non ?

Et bien pas tout à fait.

Commençons par le plus simple : la troisième partie d’un `ObjectId` est donc un incrément commençant par une valeur aléatoire.

Conséquence : si on génère plusieurs `ObjectId` d’affilée, on obtient une partie qui n’est plus vraiment aléatoire et on peut facilement prédire le prochain incrément.

Concernant la deuxième partie “aléatoire”, l’implémentation a changé depuis MongoDB 3.4.

Avec MongoDB 3.2, cette deuxième partie n’est en fait pas aléatoire et est liée à la machine sur laquelle l’ObjectId sera généré car, en réalité, il peut être décomposé de la façon suivante :
- Trois octets correspondant à un identifiant de la machine.
- Deux octets basés sur l’identifiant du processus.

Notre bien-aimé CTO Hugo l’explique aussi très bien dans un article [ici](https://eventuallycoding.com/2013/09/28/mongodb-utiliser-les-proprietes-de-vos-objectid-dans-vos-mapreduce/).

A partir de MongoDB 3.4, cette partie devient “vraiment” aléatoire, mais, en fait, pas tant que ça.

Si on regarde l’implémentation java : ces deux variables sont générées en static et ne changent ensuite plus (vous pouvez trouver les sources ici).
Dans le cas où l’`ObjectId` est généré par MongoDB directement (et pas par notre code Java), j’ai pu constater le même comportement (ces deux parties ne changent jamais). Par exemple en générant quatre `ObjectId` d’affilée dans mon client mongo préféré, j’obtiens cette suite logique (j’utilise mongo 3.6 ici) :

```
ObjectId("5cee4a91686504144c8ff6be")
ObjectId("5cee4a91686504144c8ff6bf")
ObjectId("5cee4a91686504144c8ff6c0")
ObjectId("5cee4a91686504144c8ff6c1")
```

On voit donc que l’on n’a pas vraiment d’aléatoire ici…

### Ok, mais comment on peut l’exploiter ?
L’exploitation de ce type d’aléatoire est théoriquement la suivante : si on arrive à générer plusieurs `ObjectId` d’affilée dans la même seconde, on peut donc prédire la suite qui va être générée à partir du premier `ObjectId` !

### Hmmm…

Essayons d’exploiter ça au travers de la fonctionnalité suivante.

Supposons un formulaire de reset de mot de passe :
- Lorsque je demande un reset de mon mot de passe, je reçois un mail me donnant le token (i.e l’ObjectId).
- J’utilise ce token pour valider mon nouveau mot de passe.
- Je me connecte avec mon nouveau mot de passe.

L’information importante ici est que je connais un “token” (celui qui m’est envoyé par mail), mais avec les faits énoncés au début, je peux donc prédire les tokens suivants !

Pour générer cette suite de token, une simple fonction javascript fera l’affaire (que je pourrai utiliser depuis les devtools de mon navigateur préféré) :

```js
// Emails que je connais
var emails = [
 'mickael.jeanroy+test1@gmail.com',
 'mickael.jeanroy+test2@gmail.com',
 'mickael.jeanroy+test3@gmail.com',
 'mickael.jeanroy+test4@gmail.com',
 'mickael.jeanroy+test5@gmail.com',
 'mickael.jeanroy+test6@gmail.com',
 'mickael.jeanroy+test7@gmail.com',
 'mickael.jeanroy+test8@gmail.com',
 'mickael.jeanroy+test9@gmail.com',
];
var queries = emails.map((x) => () => (
  $.ajax({
    url: '/url/reset/password',
    type: 'POST',
    contentType: 'application/x-www-form-urlencoded',
    data: 'email=${x}',
  }
));
function hack() {
 queries.forEach((query) => {
   query();
 });
}
```

Avec cette fonction JS, je vais appeler 9 fois mon URL de reset de mot de passe à la suite (i.e dans la même seconde). Dans la liste des emails connus, le premier mail m’appartient, mais pas forcément les suivants. :)

A partir du premier email reçu, je peux dès lors prédire les tokens suivants et prendre la main sur n’importe quel compte correspondant à la liste d’email que j’ai donné en paramètre. :)

### Conclusion

Evidemment, le souci n’est pas sur l’`ObjectId` en tant que tel mais plutôt sur ce que signifie “aléatoire”.

Vous l’aurez compris : un `ObjectId` ne doit jamais être utilisé en tant que token !

En java, vous pouvez par exemple utiliser `UUID.randomUUID()` qui utilise un `SecureRandom` générant une partie **vraiment aléatoire** (et si un `UUID` ne fait pas l’affaire pour vous, n’utilisez surtout pas un `Random` pseudo aléatoire).

Et j’oubliais : contrairement aux croyances, non **un timestamp n’est pas aléatoire** !

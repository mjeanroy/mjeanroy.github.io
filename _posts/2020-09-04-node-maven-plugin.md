---
layout: post
title:  "Maven, Node & NPM"
tags: ['maven', 'npm', 'tech']
---

Pour développer une application web, je pars généralement sur une stack qui consiste en un backend java, et une
application frontend dans un framework type Angular, React, VueJS, ou autre.

Avec cette stack, je vais avoir inévitablement deux outils de build aux concepts différents :

- Maven pour la partie back (à noter qu'il existe aussi Gradle, qui est un excellent outil mais Maven reste généralement mon premier choix).
- NodeJS et son package manager : `npm`, `yarn` ou `pnpm` (les trois sont très bien et le choix dépendra de votre contexte).

Se pose donc la question de faire cohabiter ces deux outils de build de manière simple.

Etant un peu éxigeant, je vais rajouter plusieurs critères pour dire que l'intégration est réussie :

- Un même cycle de vie.
- Une unique commande pour lancer mes différentes tâches (téléchargement des dépendances, build, tests, etc.).
- Je dois pouvoir travailler sur chaque projet de manière indépendante, mais l'ensemble doit pouvoir être buildé en une seule commande que ce soit sur mon environnement de dev ou sur mon outil de CI.

## Cycle de vie maven

Pour ceux qui ne débuteraient avec maven, il possède un cycle de vie qu'il me semble important de bien comprendre.
Lorsqu'on lance la commande `mvn verify`, l'application va être buildé suivant un cycle de vie bien précis. Je ne rentrerai pas
dans tous les détails, la [documentation maven](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html) étant très bien faite.

Pour rester simple, on peut résumer les grandes étapes du build :

1. Téléchargement des dépendances.
2. Validation.
3. Compilation / Build
4. Tests unitaires
5. Packaging
6. Tests d'intégration.

Personnellement, je trouve que ce cycle de vie pourrait correspondre assez bien au cycle de vie d'une application frontend :

1. Téléchagement des dépendances déclarées dans le fichier `package.json`
2. Validation syntaxique avec `ESLint`, `TSLint`, etc.
3. Build de l'application avec `webpack`, `rollup`, etc.
4. Tests (`jasmine`, `jest`, `karma`, etc.).
5. Packaging.
6. Tests end to end

C'est dans cette optique que j'ai développé il y a déjà un petit moment un plugin maven dont le seul rôle sera de lancer les scripts déclarés dans le fichier `package.json` associés à chaque étape de ce cyle de vie.

## Initialisation du projet

Pour la simplification, je présenterai uniquement les fichiers de build, les fameux `pom.xml` et `package.json` de chaque partie.

Afin d'avoir deux projets indépendants mais pouvant être buildé de manière unifiée, nous allons utiliser un projet multi-module de maven dont
voici l'arborescence :

```
<root>
  backend/
    src/
      main/
        java/
      test/
        java
    pom.xml
  frontend/
    src/
    package.json
    pom.xml
  pom.xml
```

Le `pom.xml` racine va être assez simple, on déclarera simplement nos sous-modules :

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.org</groupId>
  <artifactId>my-app</artifactId>
  <packaging>pom</packaging>
  <version>1.0-SNAPSHOT</version>

  <modules>
    <module>frontend</module>
    <module>backend</module>
  </modules>
</project>
```

### Backend

Le fichier `backend/pom.xml` est également assez simple, puisqu'il consistera juste à builder notre application java :

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

  <parent>
    <groupId>com.org</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0-SNAPSHOT</version>
  </parent>

  <modelVersion>4.0.0</modelVersion>
  <artifactId>my-app-backend</artifactId>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
      <version>2.2.0.RELEASE</version>
    </dependency>
  </dependencies>
</project>
```

### Frontend

Regardons maintenant le fichier `frontend/pom.xml` :

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

  <parent>
    <groupId>com.org</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0-SNAPSHOT</version>
  </parent>

  <modelVersion>4.0.0</modelVersion>
  <artifactId>my-app-frontend</artifactId>

  <build>
    <plugins>
      <plugin>
        <groupId>com.github.mjeanroy</groupId>
        <artifactId>node-maven-plugin</artifactId>
        <version>0.4.0</version>
        <executions>
          <execution>
            <id>npm</id>
            <goals>
              <goal>check</goal>
              <goal>install</goal>
              <goal>lint</goal>
              <goal>test</goal>
              <goal>build</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

Ici, nous utilisons le plugin `node-maven-plugin` pour intégrer nos commandes npm au cycle de vie maven.
Chaque goal maven précisé plus haut est bindé sur une phase du cycle de vie maven (plus d'infos sur le [readme](https://github.com/mjeanroy/node-maven-plugin) du projet).

Regardons à quoi ressemble le fichier `package.json` associé :

```json
{
  "name": "my-app",
  "version": "0.0.0",
  "scripts": {
    "ng": "ng",
    "start": "ng serve",
    "build": "ng build --aot=true --optimization=true --prod=true --watch=false",
    "test": "ng test -- --no-watch --no-progress --browsers=ChromeHeadlessCI",
    "tdd": "ng test",
    "lint": "ng lint",
    "e2e": "ng e2e"
  },
  "private": true,
  "dependencies": {
  },
  "devDependencies": {
  }
}
```

Chaque script va donc être appelé automatiquement par `node-maven-plugin`. Le plugin va également s'intégrer aux paramètres maven que vous utilisez peut-être comme `-DskipTests`, `-Dmaven.test.skip`, etc.

Evidemment, travailler sur la partie front ne nécessite absolument pas d'installer java ou maven, il est tout à fait possible de lancer les scripts npm de manière totalement indépendante.

## Intégration sur la CI

Le build du projet avec travis-ci va donc être assez simple car une seule commande sera suffisante pour builder / tester la partie back et front :

```yml
dist: trusty
sudo: false
language: java

jdk:
  - openjdk8

env:
  - NODE_VERSION="12.13.0"

addons:
  apt:
    sources:
      - google-chrome
    packages:
      - google-chrome-stable

before_install:
  - nvm install $NODE_VERSION
```

Je l'utilise depuis maintenant quelques années, mais n'hésitez pas à aller lire le README du projet, et à ouvrir les issues appropriées si vous pensez que quelque chose manque !

Notez que vous pouvez aussi utiliser github actions, c'est tout aussi simple :

```yml
name: CI
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - name: Set up JDK
        uses: actions/setup-java@v1
        with:
          java-version: 8
      - uses: actions/setup-node@v1
        name: Set up NodeJS
        with:
          node-version: '12.16.0'
      - name: Build
        run: mvn -B package -Dmaven.test.skip.exec=true --file pom.xml
      - name: Test
        run: mvn -B test --file pom.xml
```

C'est tout pour cet article !


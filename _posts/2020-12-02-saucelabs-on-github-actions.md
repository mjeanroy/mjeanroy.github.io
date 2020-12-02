---
layout: post
title:  "Utiliser SauceLabs avec GitHub Actions"
tags: SauceLabs "GitHub Actions" Travis TDD
---

### ➡️ Point de départ

J'ai été pendant très longtemps un utilisateur content (et même très très content) de [travis CI](https://travis-ci.com).

Il faut bien le dire :
- Ca marche bien (un peu moins maintenant, j'en parlerai plus loin).
- C'est hyper simple d'utilisation.
- Ca se branche avec GitHub en trois clics.

Sur quasiment tous mes (petits) projets open source dispo sur GitHub, travis était bien souvent l'une des premières choses que je mettais en place (un petit fichier `.travis.yml` à la racine du repo, et hop c'est parti). Ca marche avec Java, JavaScript et n'importe quel language en fait.

Sur certains projets JS, j'ai eu besoin de m'assurer que le code restait compatible avec de vieux navigateurs (Safari, IE pour ne pas les citer) et pour cela, deux choix se démarquent encore aujourd'hui :

- [SauceLabs](https://saucelabs.com/)
- [Browserstack](https://www.browserstack.com/)

Je ne parlerai pas des avantages et inconvénients des deux, je n'ai jamais vraiment testé browserstack, et pour cause : quand j'ai voulu brancher SauceLabs sur mes builds travis, ça m'a pris... deux lignes !

### ➡️ Et puis...

Puis, courant 2019, GitHub a annoncé [GitHub Actions](https://github.com/features/actions) : un outil de CI/CD directement intégré dans GitHub ! Etant très satisfait de travis, j'y ai surtout jeté un coup d'oeil par curiosité sans vraiment ressentir le besoin de tout migrer...

Et puis, il faut bien le dire, depuis quelques mois, travis-ci a quelques problèmes :
- Des jobs en queue pendant plusieurs jours, voire plusieurs jours.
- Une migration un peu fastidieuse de [travis-ci.org](https://travis-ci.org) vers [travis-ci.com](https://travis-ci.com)
- Et surtout : un crédit plus que limité pour les projets open source après migration vers [travis-ci.com](https://travis-ci.com).

Loin de moi l'envie de critique le service, je trouve toujours que c'est un outil extraordinaire, mais je pense qu'ils n'ont aujourd'hui plus la force de frappe et le budget pour rivaliser (vraiment un ressenti de très loin) et, même si j'aurais aimé laisser mes projets Open Source dessus, j'ai dû me résigner à migrer sur GitHub Actions.

S'est donc posé inévitablement la question d'utiliser SauceLabs sur GitHub Actions !

### ➡️ Talk is cheap, Show me the code

**1-** Pour commencer, il vous faut créer un compte sur [SauceLabs](https://saucelabs.com/) qui vous donnera accès aux credentials (username et access key) qui vous seront utiles un peu plus loin.

**2-** Ensuite, il vous faut ajouter les credentials SauceLabs en tant que secret sur votre repo GitHub : pour cela rendez-vous sur https://github.com/[username]/[project]/settings/secrets/actions et ajouter deux secrets :
- `SAUCE_USERNAME` correspondant à votre username sur SauceLabs
- `SAUCE_ACCESS_KEY` correspondant à votre access key sur SauceLabs.

**3-** Configurez votre build : pour cela créer le fichier `/.github/workflows/build.yml` à la racine du repo avec un contenu qui devrait ressember à :

```yml
name: CI
on: [push]
jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ ubuntu-latest ]
        node: [ 14 ]
    steps:
      - uses: actions/checkout@v1
      - uses: actions/setup-node@v1
        name: Set up NodeJS
        with:
          node-version: ${{ matrix.node }}
      - name: Install
        run: npm install
      - uses: saucelabs/sauce-connect-action@master
        with:
          username: ${{ secrets.SAUCE_USERNAME }}
          accessKey: ${{ secrets.SAUCE_ACCESS_KEY }}
          scVersion: 4.6.2
          tunnelIdentifier: github-action-tunnel
      - name: Test
        env:
          SAUCE_USERNAME: ${{ secrets.SAUCE_USERNAME }}
          SAUCE_ACCESS_KEY: ${{ secrets.SAUCE_ACCESS_KEY }}
        run: npm test
```

N'hésitez pas à remplacer la version de node et l'OS à utiliser par ce qui correspond à votre cas d'usage.

La partie importante est celle-ci :

```yml
- uses: saucelabs/sauce-connect-action@master
  with:
    username: ${{ secrets.SAUCE_USERNAME }}
    accessKey: ${{ secrets.SAUCE_ACCESS_KEY }}
    scVersion: 4.6.2
    tunnelIdentifier: github-action-tunnel
```

- Les deux clés `secrets.SAUCE_USERNAME` et `secrets.SAUCE_ACCESS_KEY` font référence aux secrets configurés juste avant.
- La valeur de `tunnelIdentifier` est importante, on la ré-utilisera plus tard.

**4-** Enfin, vous pouvez configurer votre suite de test. Pour ma part, j'utilise [Karma](https://karma-runner.github.io/latest/index.html), la configuration est très simple et nécessite simplement le package [`karma-sauce-launcher`](http://npmjs.com/package/karma-sauce-launcher).

Voici à quoi doit ressember votre fichier `karma.conf.js` :

```js
const browsers = {
  SL_safari_8: {
    base: 'SauceLabs',
    browserName: 'safari',
    version: '8.0',
    platform: 'OS X 10.10',
  },

  SL_safari_9: {
    base: 'SauceLabs',
    browserName: 'safari',
    version: '9.0',
    platform: 'OS X 10.11',
  },

  SL_safari_10: {
    base: 'SauceLabs',
    browserName: 'safari',
    version: '10.0',
    platform: 'OS X 10.11',
  },

  SL_Win10_edge: {
    base: 'SauceLabs',
    browserName: 'microsoftedge',
    version: 'latest',
    platform: 'Windows 10',
  },

  SL_Win10_ie_11: {
    base: 'SauceLabs',
    browserName: 'internet explorer',
    version: '11.285',
    platform: 'Windows 10',
  },

  SL_Win81_ie_11: {
    base: 'SauceLabs',
    browserName: 'internet explorer',
    version: '11.0',
    platform: 'Windows 8.1',
  },

  SL_ie_10: {
    base: 'SauceLabs',
    browserName: 'internet explorer',
    version: '10',
    platform: 'Windows 8',
  },

  SL_chrome: {
    base: 'SauceLabs',
    browserName: 'chrome',
    version: 'latest',
    platform: 'Windows 10',
  },

  SL_firefox: {
    base: 'SauceLabs',
    browserName: 'firefox',
    version: 'latest',
    platform: 'Windows 10',
  },
};

module.exports = (config) => {
  config.set({
    runnerPort: 9100,
    colors: true,
    logLevel: config.LOG_INFO,
    port: 9876,
    frameworks: [
      'jasmine',
    ],

    autoWatch: false,
    singleRun: true,

    reporters: [
      'dots',
      'saucelabs',
    ],

    browsers: Object.keys(browsers),

    // Adjuste these ones with your own settings.
    concurrency: 1,
    captureTimeout: 120000,
    browserNoActivityTimeout: 60000,
    browserDisconnectTimeout: 20000,
    browserDisconnectTolerance: 1,

    // Add SauceLabs browsers
    customLaunchers: browsers,

    // SauceLabs Configuration
    sauceLabs: {
      build: `GITHUB #${process.env.GITHUB_RUN_ID} (${process.env.GITHUB_RUN_NUMBER})`,
      startConnect: false,
      tunnelIdentifier: 'github-action-tunnel',
    },
  });
};
```

La partie importante concerne :
- La liste des browsers à démarrer côté SauceLabs (pour être sûr de la configuration, n'hésitez pas à utiliser leur outil dédié : https://wiki.saucelabs.com/display/DOCS/Platform+Configurator#/).
- La configuration des browsers dans Karma (voir `browsers` et `customLaunchers`).
- La configuration SauceLabs, reprenant le `tunnelIdentifier` vu plus haut !

Et voila, normalement, vous aurez votre build qui tournera avec SauceLabs, si vous voulez vous inspirer, n'hésitez pas à regarder comment ça tourne sur [ce projet](https://github.com/mjeanroy/jasmine-utils) avec :

- La configuration karma [ici](https://github.com/mjeanroy/jasmine-utils/blob/master/scripts/test/karma.saucelab.conf.js)
- La configuration du build GitHub Actions [ici](https://github.com/mjeanroy/jasmine-utils/blob/master/.github/workflows/build.yml)
- Et un exemple de build [ici](https://github.com/mjeanroy/jasmine-utils/runs/1472623252?check_suite_focus=true)

Good luck 👋 !

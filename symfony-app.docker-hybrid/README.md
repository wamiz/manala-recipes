# Symfony App: Docker Hybrid

A [Manala recipe](https://github.com/manala/manala-recipes) for projects using the Symfony CLI, PHP, Node.js, MySQL and Redis.

---

## Requirements

* [manala](https://manala.github.io/manala/)
* Docker Engine: 
    * [Debian](https://hub.docker.com/editions/community/docker-ce-server-debian)
    * [macOS](https://hub.docker.com/editions/community/docker-ce-desktop-mac)
    * [Windows](https://hub.docker.com/editions/community/docker-ce-desktop-windows)
* [Symfony CLI](https://symfony.com/doc/current/setup/symfony_server.html) (with [local proxy support](https://symfony.com/doc/current/setup/symfony_server.html#setting-up-the-local-proxy))
* PHP and Node.js must be installed by yourself on your machine

## Init

```
$ cd /path/to/my/app
$ manala init -i symfony-app.docker-hybrid --repository https://github.com/Wamiz/manala-recipes.git
```

## Configure PHP and Node.js versions

Since this recipe relies on having PHP and Node.js by yourself, it's important to create two files `.php-version` and `.nvmrc` 
which will contains the PHP and Node.js versions to use for your project.

```shell
cd /path/to/my/app
echo 8.0 > .php-version # Use PHP 8.0
echo 14 > .nvmrc # Use Node.js 14
```

Those files will be used by:
- The Symfony CLI when using `symfony php` and `symfony composer` (eg: `symfony console cache:clear, `symfony composer install)
- NVM when using `nvm use`
- GitHub Actions, thanks to [the action `setup-environment`](#github-actions)

**It is important to use `symfony php` and not `php` directly for running commands, thanks to its [Docker integration](https://symfony.com/doc/current/setup/symfony_server.html#docker-integration) 
it automatically exposes environments variables from Docker (eg: `DATABASE_URL`, `REDIS_URL`, ...) to PHP.**

## Quick start

In a shell terminal, change directory to your app, and run the following commands:

```shell
$ cd /path/to/my/app
$ manala init -i symfony-app.docker-hybrid --repository https://github.com/Wamiz/manala-recipes.git
```

Edit the `Makefile` at the root directory of your project and add the following lines at the beginning of the file:

```makefile
-include .manala/Makefile

# This function will be called at the end of "make setup"
define setup
	# For example:
	# $(MAKE) install-app
	# $(MAKE) init-db@test
endef

# This function will be called at the end of "make setup@integration"
define setup_integration
	# For example:
	# $(MAKE) install-app@integration
endef
```

Update the `.manala.yaml` file according your needs, then run:

```shell
manala up
```

**Don't forget to run the `manala up` command each time you update the `.manala.yaml` file to actually apply your changes.**

From now on, if you execute the `make help` command in your console, you should obtain the following output:

```shell
Usage: make [target] 	

Help:   
  help This help 	

Environment:   
  setup              Setup the development environment   
  setup@integration  Setup the integration environment   
  up                 Start the development environment   
  halt               Stop the development environment   
  destroy            Destroy the development environment 	

Project:
  install-app:             Install application
  install-app@integration: Install application in integration environment
```

## Docker interaction

Initialise Docker Compose containers and your app:
```bash
make setup
```

Start Docker Compose containers:
```bash
make up
```

Stop Docker Compose containers:
```bash
make halt
```

Stop and remove Docker Compose containers:
```shell
make destroy
```

## System

Here is an example of a system configuration in `.manala.yaml`:

```yaml
##########
# System #
##########

system:
    app_name: your-app
    mysql:
        version: 5.7
    redis:
        version: '*'
```

## Integration

### GitHub Actions

Since this recipe generates a `docker-compose.yaml` file, it can be used to provide a 
fully-fledged environnement according to your project needs on GitHub Actions.

```yaml
name: CI

on:
    pull_request:
        types: [opened, synchronize, reopened, ready_for_review]

env:
    TZ: UTC

jobs:
    php:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v2
            
            # The code of this local action can be found below
            - uses: ./.github/actions/setup-environment

            - uses: shivammathur/setup-php@v2
              with:
                  php-version: ${{ env.PHP_VERSION }} # PHP_VERSION comes from setup-environment local action
                  coverage: none
                  extensions: iconv, intl
                  ini-values: date.timezone=${{ env.TZ }}
                  tools: symfony

            - uses: actions/setup-node@v2
              with:
                  node-version: ${{ env.NODE_VERSION }} # NODE_VERSION comes from setup-environment local action

            - uses: actions/cache@v2
              with:
                  path: ${{ env.COMPOSER_CACHE_DIR }}
                  key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
                  restore-keys: ${{ runner.os }}-composer-

            - uses: actions/cache@v2
              with:
                  path: ${{ env.YARN_CACHE_DIR }}
                  key: ${{ runner.os }}-yarn-${{ hashFiles('**/yarn.lock') }}
                  restore-keys: ${{ runner.os }}-yarn-

            # Will setup the Symfony CLI and build Docker Compose containers
            # No need to create DATABASE_URL or REDIS_URL environment variables, they will be
            # automatically injected to PHP/Symfony thanks to the Symfony CLI's Docker Integration
            - run: make setup@integration

            # Check versions
            - run: symfony php -v # PHP 8.0.3
            - run: node -v # Node.js 14.16.0

            # Run some tests... remember to use "symfony php" and not "php"
            - run: symfony console cache:clear
            - run: symfony console lint:twig templates
            - run: symfony console lint:yaml config --parse-tags
            - run: symfony console lint:xliff translations

```

This is the code of local action `setup-environment`:
```yaml
# .github/actions/setup-environment/action.yml
name: Setup environment
description: Setup environment
runs:
    using: 'composite'
    steps:
        - run: echo "PHP_VERSION=$(cat .php-version | xargs)" >> $GITHUB_ENV
          shell: bash

        - run: echo "NODE_VERSION=$(cat .nvmrc | xargs)" >> $GITHUB_ENV
          shell: bash

        # Composer cache
        - id: composer-cache
          run: echo "::set-output name=dir::$(composer global config cache-files-dir)"
          shell: bash

        - run: echo "COMPOSER_CACHE_DIR=${{ steps.composer-cache.outputs.dir }}" >> $GITHUB_ENV
          shell: bash

        # Yarn cache
        - id: yarn-cache-dir
          run: echo "::set-output name=dir::$(yarn cache dir)"
          shell: bash

        - run: echo "YARN_CACHE_DIR=${{ steps.yarn-cache-dir.outputs.dir }}" >> $GITHUB_ENV
          shell: bash
```

### Common integration tasks

Add in your `Makefile`:

```makefile
# ...

# This function will be called during "make setup"
define setup
    $(MAKE) install-app
    $(MAKE) init-db@test
endef

# This function will be called during "make setup@integration"
define setup_integration
    $(MAKE) install-app@integration
endef

###########
# Install #
###########

## Install application
install-app: composer-install init-db
install-app:
	$(symfony) console cache:clear
	yarn install
	yarn dev

## Install application in integration environment
install-app@integration: export APP_ENV=test
install-app@integration:
	$(composer) install --ansi --no-interaction --no-progress --prefer-dist --optimize-autoloader
	yarn install --color=always --no-progress --frozen-lockfile
	yarn dev
	$(MAKE) init-db@integration

################
# Common tasks #
################

composer-install:
	$(composer) install --ansi --no-interaction

init-db:
	$(symfony) console doctrine:database:drop --force --if-exists --no-interaction
	$(symfony) console doctrine:database:create --no-interaction
	$(symfony) console doctrine:schema:update --force --no-interaction # to remove when we will use migrations
	# $(symfony) console doctrine:migrations:migrate --no-interaction
	$(symfony) console hautelook:fixtures:load --no-interaction

init-db@test: export APP_ENV=test
init-db@test: init-db

init-db@integration: export APP_ENV=test
init-db@integration:
	$(symfony) console doctrine:database:create --if-not-exists --no-interaction
	$(symfony) console doctrine:schema:update --force --no-interaction # to remove when we will use migrations
	# $(symfony) console doctrine:migrations:migrate --no-interaction
	$(symfony) console hautelook:fixtures:load --no-interaction

reload-db@test: export APP_ENV=test
reload-db@test:
	$(symfony) console hautelook:fixtures:load --purge-with-truncate --no-interaction
```

### Tools

#### Admin UI for MySQL

Run `make run-phpmyadmin` to run a local [PhpMyAdmin](https://github.com/phpmyadmin/phpmyadmin) instance

#### Admin UI for Redis

Run `make run-phpredisadmin` to run a local [PhpRedisAdmin](https://github.com/erikdubbelboer/phpRedisAdmin) instance.

### Installing PHP on your machine

Sur un OS Debian ou Ubuntu, il est recommandé d'utiliser le [PPA ondrej/php](https://launchpad.net/~ondrej/+archive/ubuntu/php/)
qui permet d'installer facilement plusieurs versions de PHP. Il est également possible d'utiliser [phpenv](https://github.com/phpenv/phpenv-installer)
ou [brew](https://formulae.brew.sh/formula/php).

Pour le PPA ondrej/php :

```shell
sudo add-apt-repository ppa:ondrej/php
sudo apt-get update

install_php() {
  VERSION=$1
  sudo apt install php$VERSION \
     php$VERSION-{zip,opcache,acpu,xdebug,fpm} \
     php$VERSION-{pgsql,mysql} \
     php$VERSION-{intl,curl,mbstring,xml,gd}
}

install_php 7.4
install_php 8.0
```

### Activer/désactiver XDebug

L'extension XDebug est activée par défaut après son installation. Il n'est pas souhaitable que XDebug soit tout le temps activé pour des raisons de performances.
Pour désactiver XDebug pour toutes les versions de PHP et pour toutes les SAPI (CLI, FPM, ...), lancer `sudo phpdismod -v ALL -s ALL xdebug`.

Pour activer XDebug sur un projet, il faut modifier le `php.ini` (**du projet**) et dé-commenter les lignes suivantes :

```ini
zend_extension=xdebug.so
xdebug.remote_enable=1
xdebug.remote_autostart=1
```

Pour vérifier si XDebug est bien activé, lancer `symfony php-fpm -i | grep xdebug` et plusieurs lignes de configuration devraient s'afficher.

## Installer Node.js sur sa machine

Sur un OS Debian, Ubuntu ou macOS, il est recommandé d'utiliser [nvm](https://github.com/nvm-sh/nvm) qui permet d'installer et utiliser
très simplement plusieurs versions de Node.js.
Suivre les étapes d'installation via https://github.com/nvm-sh/nvm#installing-and-updating.

Il faudra ensuite lancer `nvm use` dans le dossier du projet pour automatiquement utiliser la bonne version de Node.js (renseignéee dans
le fichier `.nvmrc`).

L'installation de `yarn` se fait via un `npm intall --global yarn`.

Pour ceux utilisant zsh, il est possible d'utiliser le plugin [](https://github.com/aspirewit/zsh-nvm-auto-switch)
qui permet d'automatiquement lancer `nvm use` lorsqu'un fichier `.nvmrc` est détecté.

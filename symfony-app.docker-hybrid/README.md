# Symfony App: Docker Hybrid

A [Manala recipe](https://github.com/manala/manala-recipes) for projects using the Symfony CLI, PHP, Node.js, MariaDB and Redis.

---

## Requirements

* [manala](https://manala.github.io/manala/)
* Docker Engine: 
    * [Debian](https://hub.docker.com/editions/community/docker-ce-server-debian)
    * [macOS](https://hub.docker.com/editions/community/docker-ce-desktop-mac)
    * [Windows](https://hub.docker.com/editions/community/docker-ce-desktop-windows)
* [Docker Compose](https://docs.docker.com/compose/install/)
* [Symfony CLI](https://symfony.com/doc/current/setup/symfony_server.html) (with [local proxy support](https://symfony.com/doc/current/setup/symfony_server.html#setting-up-the-local-proxy))
* PHP and Node.js must be installed by yourself on your machine, see:
    * [Installing PHP on your machine](#installing-php-on-your-machine)
    * [Installing Node.js on your machine](#installing-nodejs-on-your-machine)

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
- The Symfony CLI when using `symfony php` and `symfony composer` (eg: `symfony console cache:clear`, symfony composer install)
- NVM when using `nvm use`
- GitHub Actions, thanks to [the action `setup-environment`](#github-actions)

**It is important to use `symfony php` and not `php` directly, thanks to [Symfony CLI's Docker integration](https://symfony.com/doc/current/setup/symfony_server.html#docker-integration) 
it automatically exposes environment variables from Docker (eg: `DATABASE_URL`, `REDIS_URL`, ...) to PHP.**

## Quick start

In a shell terminal, change directory to your app and run the following commands:

```shell
$ cd /path/to/my/app
$ manala init -i symfony-app.docker-hybrid --repository https://github.com/Wamiz/manala-recipes.git
```

Edit the `Makefile` at the root directory of your project and add the following lines at the beginning of the file:

```makefile
-include .manala/Makefile

# This function will be called at the end of "make install"
define install
	# For example:
	# $(MAKE) install-app
	# $(MAKE) init-db@test
endef

# This function will be called at the end of "make install@integration"
define install_integration
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
  install              Install the environment   
  install@integration  Install the environment (integration)   
  up                   Update and start the environment   
  start                Start the environment   
  stop                 Stop the environment   
  down                 Stop and remove the environment 	

Development tools:   
  run-phpmyadmin     Start a web interface for PhpMyAdmin   
  run-phpredisadmin  Start a web interface for PhpRedisAdmin   
  open-mailcatcher   Open the web interface for MailCatcher   
  open-rabbitmq      Open the web interface for RabbitMQ 	

Project:
```

## Docker interaction

Initialise Docker Compose containers and your app:
```bash
make install
```

Update and start Docker Compose containers:
```bash
make up
```

Start Docker Compose containers:
```bash
make start
```

Stop Docker Compose containers:
```bash
make stop
```

Stop and remove Docker Compose containers:
```shell
make down
```

## System

Here is an example of a system configuration in `.manala.yaml`:

```yaml
##########
# System #
##########

system:
    app_name: your-app
    mariadb:
        version: 10.4
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

# This function will be called during "make install"
define install
    $(MAKE) install-app
    $(MAKE) init-db@test
endef

# This function will be called during "make install@integration"
define install_integration
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

- run `make run-phpmyadmin` to start a local [PhpMyAdmin](https://github.com/phpmyadmin/phpmyadmin) instance
- run `make run-phpredisadmin` to start a local [PhpRedisAdmin](https://github.com/erikdubbelboer/phpRedisAdmin) instance.
- run `make open-mailcatcher` to open a local [MailCatcher Web UI](https://mailcatcher.me).
- run `make open-rabbitmq` to open a local [RabbitMQ Management Web UI](https://www.rabbitmq.com/management.html).

### Installing PHP on your machine

#### Debian/Ubuntu 

On Debian or Ubuntu, it's recommended to use [deb.sury.org](https://deb.sury.org/#php-packages) to install multiple PHP versions. 
You can also use [phpenv](https://github.com/phpenv/phpenv-installer) or [brew](https://formulae.brew.sh/formula/php).

If using deb.sury.org, you can run the following commands:
```shell
# install PHP 7.4
sudo apt install php7.4 php7.4-{zip,opcache,acpu,xdebug,fpm} php7.4-{pgsql,mysql} php7.4-{json,intl,curl,mbstring,xml,gd}

# install PHP 8.0
sudo apt install php8.0 php8.0-{zip,opcache,acpu,xdebug,fpm} php8.0-{pgsql,mysql} php8.0-{intl,curl,mbstring,xml,gd}
```

#### MacOS

You can use [brew](https://formulae.brew.sh/formula/php) to instal multiple PHP versions.

```
# install PHP 7.4
brew install php@7.4

# install PHP 8.0
brew install php@8.0 
```

### Enable/disable XDebug

XDebug is enabled by default after its installation, but you don't want it to be always enabled for performance issues.
To globally disable XDebug, you can run `sudo phpdismod -v ALL -s ALL xdebug` (Debian).

To enable XDebug for your project, you can update your `.manala.yml` uncomment the following lines and then run `manala up`:

```ini
zend_extension=xdebug.so
xdebug.remote_enable=1
xdebug.remote_autostart=1
```

To check if XDebug is enabled, run `symfony php-fpm -i | grep xdebug`.

## Installing Node.js on your machine

On Debian, Ubuntu or macOS, it's recommended to use [nvm](https://github.com/nvm-sh/nvm) to easily install and manage multiple Node.js versions.

It's required to run `nvm use` in the project folder to automatically switch to the good Node.js version (from the `.nvmrc` file).

:information_source: For zsh and [Oh My Zsh](https://ohmyz.sh/) users, it's possible to use the [dedicated plugin](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/nvm) by using the following configuration:
```shell
# .zshrc

# Will autoload NVM
NVM_AUTOLOAD=1

plugins=(... nvm)

# The following commands are not required anymore, since the Oh My Zsh nvm plugin already loads nvm. 
#export NVM_DIR="$HOME/.nvm"
#[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
#[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```

### Yarn

The [package manager `yarn`](https://yarnpkg.com/) can be installed with `npm install --global yarn`.

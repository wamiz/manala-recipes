# Symfony App: Docker Hybrid

A [Manala recipe](https://github.com/manala/manala-recipes) for projects using the Symfony CLI, PHP, Node.js, MariaDB and Redis.

---

## Prerequisites

### 1. Manala

The [manala](https://manala.github.io/manala/) binary is required to use this recipe

### 2. Docker

We need Docker to ease the creating and management of our database (MariaDB) and Redis.

You can install Docker Desktop (3.4+) which constains Docker and the Docker compose plugin:

-   [Install Docker Desktop on MacOS](https://hub.docker.com/editions/community/docker-ce-desktop-mac)
-   [Install Docker Desktop on Linux](https://docs.docker.com/desktop/install/linux-install/), **and follow** the [post-installation steps](https://docs.docker.com/engine/install/linux-postinstall/) for the `docker group`, **you don't need to use Sudo to run Docker commands**.

### 3. Symfony CLI

The [Symfony CLI](https://symfony.com/doc/current/setup/symfony_server.html) is the key-tool of our project, it allows us to run a HTTPS webserver,
configure domain names, integrate Docker Compose to our application, etc...

You will need to install it, and configure the [local proxy support](https://symfony.com/doc/current/setup/symfony_server.html#setting-up-the-local-proxy).

### 4. PHP

In this recipe, PHP is not managed by Docker, so you need to install it on your machine.

> [!NOTE]
> If needed, you will need to adapt the version of PHP to install, depending on the version you want to use.
> The next instructions will use PHP 8.2, but you can use any version you want.

#### MacOS

You can use Homebrew to install PHP:

```shell
# install PHP 8.2
brew install php@8.2
```

And install the following extensions:

```shell
brew install pkg-config imagemagick pcre2

symfony pecl install {redis,apcu,imagick}
```

> [!IMPORTANT]
> You may get an error regarding `pcre2.h`. To solve this issue, you have to create a symbolic link:
>
> ```shell
> ln -s /opt/homebrew/Cellar/pcre2/$(brew list --versions pcre2 | cut -d ' ' -f2)/include/pcre2.h /opt/homebrew/Cellar/php/$(brew list --versions php@8.2 | cut -d ' ' -f2)/include/php/ext/pcre/pcre2.h
> ```

Finally, you can add PHP binaries to your path by adding the following line in your `.zshrc` or `.bashrc`:

```shell
export PATH="/opt/homebrew/opt/php@8.2/bin:$PATH"
export PATH="/opt/homebrew/opt/php@8.2/sbin:$PATH"
```

#### Debian/Ubuntu

On Debian or Ubuntu, it's recommended to use [deb.sury.org](https://deb.sury.org/#php-packages) to install multiple PHP versions.
You can also use [phpenv](https://github.com/phpenv/phpenv-installer) or [brew](https://formulae.brew.sh/formula/php).

If using deb.sury.org, you can run the following commands:

```shell
# install PHP 8.2
sudo apt install php8.2 php8.2-{zip,opcache,apcu,xdebug,fpm} php8.2-{pgsql,mysql} php8.2-{intl,curl,mbstring,xml,gd,imagick,redis}
```

### 5. Installing Node.js on your machine

On Debian, Ubuntu or macOS, you **must use** [nvm](https://github.com/nvm-sh/nvm) to easily install and manage multiple Node.js versions,
and to avoid permission issues when installing global packages.

It's required to run `nvm use` in the project folder to automatically switch to the good Node.js version (from the `.nvmrc` file).

> [!NOTE]
> For zsh and [Oh My Zsh](https://ohmyz.sh/) users, it's possible to use the [dedicated plugin](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/nvm) by using the following configuration:

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

## Using the recipe 

```
$ cd /path/to/my/app
$ manala init -i symfony-app.docker-hybrid --repository https://github.com/Kocal/manala-recipes.git
```

### 1. Configure PHP and Node.js versions

Since this recipe relies on having PHP and Node.js by yourself, it's important to create two files `.php-version` and `.nvmrc`
which will contains the PHP and Node.js versions to use for your project.

```shell
cd /path/to/my/app
echo 8.2 > .php-version # Use PHP 8.2
echo 16 > .nvmrc # Use Node.js 16
```

Those files will be used by:
- The Symfony CLI when using `symfony php` and `symfony composer` (eg: `symfony console cache:clear`, symfony composer install)
- NVM when using `nvm use`
- GitHub Actions, thanks to [the action `setup-environment`](#github-actions)

> [!IMPORTANT]
> It is important to use `symfony php` and not `php` directly, thanks to [Symfony CLI's Docker integration](https://symfony.com/doc/current/setup/symfony_server.html#docker-integration)
> which automatically exposes environment variables from Docker (eg: `DATABASE_URL`, `REDIS_URL`, ...) to PHP.

### 2. Update your `Makefile`

Edit the `Makefile` at the root directory of your project and add the following lines at the beginning of the file:

```makefile
include .manala/Makefile

# This function will be called at the end of "make install"
define install
	# For example:
	# $(MAKE) app.install
	# $(MAKE) database.init@integration
endef

# This function will be called at the end of "make install@integration"
define install_integration
	# For example:
	# $(MAKE) app.install@integration
endef
```

### 3. Update your Manala configuration file

Update the `.manala.yaml` file according your needs, then run the following command to update your environment configuration:

```shell
manala up
```

> [!IMPORTANT]
> Don't forget to run the `manala up` command each time you update the `.manala.yaml` file to actually apply your changes.

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

Project:
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
            - run: make install@integration

            # Check versions
            - run: symfony php -v # PHP 8.2.x
            - run: node -v # Node.js 16.x

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

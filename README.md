# moodle-docker: Docker Containers for Moodle Developers
[![moodle-docker CI](https://github.com/moodlehq/moodle-docker/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/moodlehq/moodle-docker/actions/workflows/ci.yml)

This repository contains Docker configuration aimed at Moodle developers and testers to easily deploy a testing environment for Moodle using Nginx and PHP-FPM.

## Features:
* All supported database servers (PostgreSQL, MySQL, Microsoft SQL Server, Oracle XE)
* Nginx webserver with PHP-FPM
* Behat configuration (browserless by default)
* Catch-all smtp server and web interface to messages using [Mailpit](https://github.com/axllent/mailpit)
* All PHP Extensions enabled configured for external services (e.g. solr, ldap)
* All supported PHP versions
* Zero-configuration approach
* Backed by [automated tests](https://travis-ci.com/moodlehq/moodle-docker/branches)

## Prerequisites
* [Docker](https://docs.docker.com) and [Docker Compose](https://docs.docker.com/compose/cli-command/#installing-compose-v2) installed if your Docker CLI version does not support `docker compose` command.
* It's recommended to always run the latest versions of each, but at the minimum Docker v20.10.15 and Docker Compose v2.5.0 should be used.
* 3.25GB of RAM (if you choose [Microsoft SQL Server](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-setup#prerequisites) as db server)
* A directory for Moodle data (moodledata).

## Quick start

```bash
# Change ./moodle to your /path/to/moodle if you already have it checked out
export MOODLE_DOCKER_WWWROOT=./moodle

# Define the path to your Moodle data directory
export MOODLE_DOCKER_DATAROOT=./moodledata # Ensure this directory exists and is writable

# Choose a db server (Currently supported: pgsql, mariadb, mysql, mssql, oracle)
export MOODLE_DOCKER_DB=pgsql # or mariadb

# Get Moodle code, you could select another version branch (skip this if you already got the code)
git clone -b MOODLE_403_STABLE git://git.moodle.org/moodle.git $MOODLE_DOCKER_WWWROOT

# Ensure customized config.php for the Docker containers is in place
cp config.docker-template.php $MOODLE_DOCKER_WWWROOT/config.php

# Start up containers
bin/moodle-docker-compose up -d

# Wait for DB to come up (important for oracle/mssql)
bin/moodle-docker-wait-for-db

# Work with the containers (see below)
# [..]

# Shut down and destroy containers
bin/moodle-docker-compose down
```
## Run several Moodle instances

By default, the script will load a single instance. If you want to run two
or more different versions of Moodle at the same time, you have to add this
environment variable prior running any of the steps at `Quick start`:

```bash
# Define a project name; it will appear as a prefix on container names.
export COMPOSE_PROJECT_NAME=moodle34

# Use a different public web port from those already taken
export MOODLE_DOCKER_WEB_PORT=1234

# [..] run all "Quick steps" now
```

Having set up several Moodle instances, you need to have set up
the environment variable `COMPOSE_PROJECT_NAME` to just refer
to the instance you expect to. See
[envvars](https://docs.docker.com/compose/reference/envvars/)
to see more about `docker-compose` environment variables.

## Use containers for running behat tests

Behat tests run in a browserless environment by default using the Nginx and PHP-FPM services.

```bash
# Initialize behat environment
bin/moodle-docker-compose exec webserver php admin/tool/behat/cli/init.php # 'webserver' is the PHP-FPM service
# [..]

# Run behat tests (e.g., non-Javascript tests)
bin/moodle-docker-compose exec -u www-data webserver php admin/tool/behat/cli/run.php --tags=@auth_manual # 'webserver' is the PHP-FPM service
# Example output:
# Running single behat site:
# Moodle X.Ydev (Build: YYYYMMDD), ...
# Php: X.Y.Z, pgsql: X.Y.Z, OS: Linux ...
# Server OS "Linux"
# Started at DD-MM-YYYY, HH:MM
# ...............
#
# 2 scenarios (2 passed)
# 15 steps (15 passed)
# Xs (XX.XXMb)
```

Notes:

* The behat faildump directory is exposed at http://localhost:8000/_/faildumps/ (or your configured web port).
* Browser-based Behat tests are no longer supported by default with this setup as Selenium has been removed.
  If you require browser-based testing, you would need to customize the Docker Compose setup to include a Selenium service and configure Moodle's Behat accordingly.

## Use containers for running phpunit tests

```bash
# Initialize phpunit environment
bin/moodle-docker-compose exec webserver php admin/tool/phpunit/cli/init.php
# [..]

# Run phpunit tests

bin/moodle-docker-compose exec webserver vendor/bin/phpunit auth/manual/tests/manual_test.php # 'webserver' is the PHP-FPM service
Moodle 4.0.4 (Build: 20220912), ef7a51dcb8e805a6889974b04d3154ba8bd874f2
Php: 7.3.33, pgsql: 11.15 (Debian 11.15-1.pgdg90+1), OS: Linux 5.10.0-11-amd64 x86_64
PHPUnit 9.5.13 by Sebastian Bergmann and contributors.

..                                                                  2 / 2 (100%)

Time: 00:00.304, Memory: 72.50 MB

OK (2 tests, 7 assertions)
```

Notes:
* If you want to run tests with code coverage reports:
```
# Build component configuration
bin/moodle-docker-compose exec webserver php admin/tool/phpunit/cli/util.php --buildcomponentconfigs # 'webserver' is the PHP-FPM service
# Execute tests for component
bin/moodle-docker-compose exec webserver php -d pcov.enabled=1 -d pcov.directory=. vendor/bin/phpunit --configuration reportbuilder --coverage-text # 'webserver' is the PHP-FPM service
```
* See available [Command-Line Options](https://phpunit.readthedocs.io/en/9.5/textui.html#textui-clioptions) for further info

## Use containers for manual testing

```bash
# Initialize Moodle database for manual testing
bin/moodle-docker-compose exec webserver php admin/cli/install_database.php --agree-license --fullname="Docker moodle" --shortname="docker_moodle" --summary="Docker moodle site" --adminpass="test" --adminemail="admin@example.com" # 'webserver' is the PHP-FPM service
```

Notes:
* Moodle is configured to listen on `http://localhost:8000/` (or your configured `MOODLE_DOCKER_WEB_PORT`).
* Mailpit is listening on `http://localhost:8000/_/mail` to view emails which Moodle has sent out.
* The admin `username` you need to use for logging in is `admin` by default. You can customize it by passing `--adminuser='myusername'`
* During manual testing, if you are facing that your Moodle site is logging
 you off continuously, putting the correct credentials, clean all cookies
 for your Moodle site URL (usually `localhost`) from your browser.
 [More info](https://github.com/moodlehq/moodle-docker/issues/256).

## Use containers for running behat tests for the Moodle App
This section is no longer applicable as Selenium and default Moodle App testing support have been removed.

## Using VNC to view behat tests
This section is no longer applicable as Selenium has been removed.

## Stop and restart containers
`bin/moodle-docker-compose down` which was used above after using the containers stops and destroys the containers. If you want to use your containers continuously for manual testing or development without starting them up from scratch everytime you use them, you can also just stop without destroying them. With this approach, you can restart your containers sometime later, they will keep their data and won't be destroyed completely until you run `bin/moodle-docker-compose down`.

```bash
# Stop containers
bin/moodle-docker-compose stop

# Restart containers
bin/moodle-docker-compose start
```

## Environment variables

You can change the configuration of the docker images by setting various environment variables **before** calling `bin/moodle-docker-compose up`.
When you change them, use `bin/moodle-docker-compose down && bin/moodle-docker-compose up -d` to recreate your environment.

| Environment Variable                      | Mandatory | Allowed values                        | Default value | Notes                                                                        |
|-------------------------------------------|-----------|---------------------------------------|---------------|------------------------------------------------------------------------------|
| `MOODLE_DOCKER_DB`                        | yes       | pgsql, mariadb, mysql, mssql, oracle  | none          | The database server to run against                                           |
| `MOODLE_DOCKER_WWWROOT`                   | yes       | path on your file system              | none          | The path to the Moodle codebase (`$CFG->dirroot`) you intend to test. Mounted into PHP-FPM and Nginx. |
| `MOODLE_DOCKER_DATAROOT`                  | yes       | path on your file system              | none          | The path to the Moodle data directory (`$CFG->dataroot`). Mounted into PHP-FPM and Nginx. Ensure this directory exists and is writable by the container user. |
| `MOODLE_DOCKER_DB_VERSION`                | no        | Docker tag - see relevant database page on docker-hub | mysql: 8.0 <br/>pgsql: 13 <br/>mariadb: 10.7 <br/>mssql: 2017-latest <br/>oracle: 21| The database server docker image tag |
| `MOODLE_DOCKER_PHP_VERSION`               | no        | 8.4, 8.3, 8.2, 8.1, 8.0, 7.4, 7.3, 7.2, 7.1, 7.0, 5.6| 8.2           | The php version to use for the PHP-FPM service                                                       |
| `MOODLE_DOCKER_PHPUNIT_EXTERNAL_SERVICES` | no        | any value                             | not set       | If set, dependencies for memcached, redis, solr, and openldap are added      |
| `MOODLE_DOCKER_BBB_MOCK`                  | no        | any value                             | not set       | If set the BigBlueButton mock image is started and configured                |
| `MOODLE_DOCKER_MATRIX_MOCK`               | no        | any value                             | not set       | If set the Matrix mock image is started and configured                       |
| `MOODLE_DOCKER_MLBACKEND`                 | no        | any value                             | not set       | If set the Python Machine Learning image is started and configured |
| `MOODLE_DOCKER_BEHAT_FAILDUMP`            | no        | Path on your file system              | not set       | Behat faildumps are available at http://localhost:8000/_/faildumps/ by default (served by Nginx), this allows for mapping a specific filesystem folder to retrieve the faildumps in bulk / automated ways |
| `MOODLE_DOCKER_DB_PORT`                   | no        | any integer value                     | none          | If you want to bind to any host IP different from the default 127.0.0.1, you can specify it with the bind_ip:port format (0.0.0.0 means bind to all). Username is "moodle" (or "sa" for mssql) and password is "m@0dl3ing". |
| `MOODLE_DOCKER_WEB_HOST`                  | no        | any valid hostname                    | localhost     | The hostname for web (used in Moodle's $CFG->wwwroot)                                |
| `MOODLE_DOCKER_WEB_PORT`                  | no        | any integer value (or bind_ip:integer)| 127.0.0.1:8000| The port number for Nginx. If set to 0, no port is used.<br/>If you want to bind to any host IP different from the default 127.0.0.1, you can specify it with the bind_ip:port format (0.0.0.0 means bind to all) |
| `MOODLE_DOCKER_TIMEOUT_FACTOR`            | no        | any integer value                     | 1             | If set the timeouts in behat will be multiplied by the factor                |
<!-- Removed MOODLE_DOCKER_BROWSER, MOODLE_DOCKER_SELENIUM_VNC_PORT, MOODLE_DOCKER_APP_PATH, MOODLE_DOCKER_APP_VERSION -->

In addition to that, `MOODLE_DOCKER_RUNNING=1` env variable is defined and available
in the webserver (PHP-FPM) container to flag being run by `moodle-docker`. Developer
can use this to conditionally make changes in `config.php`. The common case is
to load test-specific configuration:
```php
// Load moodle-docker config file if we are in moodle-docker environment
if (getenv('MOODLE_DOCKER_RUNNING')) {
    // Ensure this path matches the one in $MOODLE_DOCKER_WWWROOT/config.php
    // if you are using a custom config.php setup.
    // By default, cp config.docker-template.php $MOODLE_DOCKER_WWWROOT/config.php handles this.
    // require_once(__DIR__ . '/config.docker-template.php'); // This line might be redundant if config.php *is* the template
}

require_once(__DIR__ . '/lib/setup.php'); // Do not edit.
```

## Local customisations

In some situations you may wish to add local customisations, such as including additional containers, or changing existing containers.

This can be accomplished by specifying a `local.yml`, which will be added in and loaded with the existing yml configuration files automatically. For example:

``` file="local.yml"
version: "2"
services:

  # Add the adminer image at the latest tag on port 8080:8080
  adminer:
    image: adminer:latest
    restart: always
    ports:
      - 8080:8080
    depends_on:
      - "db"

  # Modify the webserver (PHP-FPM) image to add another volume:
  webserver:
    volumes:
      - "/opt/myphpextras:/usr/local/etc/php/conf.d/myextras.ini:ro"

  # Modify the nginx image to add another volume or custom config:
  nginx:
    volumes:
      - "./mynginx-custom.conf:/etc/nginx/conf.d/my-custom.conf:ro"
```

## Using XDebug for live debugging

The XDebug PHP Extension is not included in this setup and there are reasons not to include it by default.

However, if you want to work with XDebug, especially for live debugging, you can add XDebug to a running webserver container easily:

```
# Install XDebug extension with PECL
moodle-docker-compose exec webserver pecl install xdebug # 'webserver' is the PHP-FPM service

# Set some wise setting for live debugging - change this as needed
read -r -d '' conf <<'EOF'
; Settings for Xdebug Docker configuration
xdebug.mode = debug
xdebug.client_host = host.docker.internal
; For PHP-FPM, Xdebug needs to know which port to connect back to. Default is 9003.
xdebug.client_port = 9003
; Some IDEs (eg PHPSTORM, VSCODE) may require configuring an IDE key, uncomment if needed
; xdebug.idekey=MY_FAV_IDE_KEY
EOF
moodle-docker-compose exec webserver bash -c "echo '$conf' >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini" # 'webserver' is the PHP-FPM service

# Enable XDebug extension and restart the PHP-FPM container (webserver)
# The moodlehq/moodle-php-fpm image should handle enabling the extension if present.
# If not, you might need a command like: docker-php-ext-enable xdebug
# moodle-docker-compose exec webserver docker-php-ext-enable xdebug # Uncomment if needed for your PHP-FPM image
moodle-docker-compose restart webserver
```

While setting these XDebug settings depending on your local need, please take special care of the value of `xdebug.client_host` which is needed to connect from the container to the host. The given value `host.docker.internal` is a special DNS name for this purpose within Docker for Windows and Docker for Mac. If you are running on another Docker environment, you might want to try the value `localhost` instead or even set the hostname/IP of the host directly. Please turn off the firewall or open the port used in the `xdebug.client_port` (default 9003).

Open the port (9003 is the default one) by using the example command for Linux Ubuntu:
```
sudo ufw allow 9003
```


After these commands, XDebug ist enabled and ready to be used in the webserver container.
If you want to disable and re-enable XDebug during the lifetime of the webserver container, you can achieve this with these additional commands:

```
# Disable XDebug extension and restart the PHP-FPM container
moodle-docker-compose exec webserver bash -c "mv /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini.disabled || true" # 'webserver' is the PHP-FPM service
moodle-docker-compose restart webserver

# Enable XDebug extension and restart the PHP-FPM container
moodle-docker-compose exec webserver bash -c "mv /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini.disabled /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini || true" # 'webserver' is the PHP-FPM service
moodle-docker-compose restart webserver
```

## Advanced usage

As can be seen in [bin/moodle-docker-compose](https://github.com/moodlehq/moodle-docker/blob/main/bin/moodle-docker-compose),
this repo is just a series of Docker Compose configurations and a light wrapper which make use of companion docker images. Each part
is designed to be reusable and you are encouraged to use the docker `compose` commands as needed. The primary services are now `nginx` (web server), `webserver` (PHP-FPM), and `db` (database).

## Quick start with Gitpod

Gitpod is a free, cloud-based, development environment providing VS Code and a suitable development environment right in your browser.

When launching a workspace in Gitpod, it will automatically:

* Clone the Moodle repo into the `<workspace>/moodle` folder.
* Initialise the Moodle database.
* Start the Moodle webserver.

<p>
    <a href="https://gitpod.io/#https://github.com/moodlehq/moodle-docker" target="_blank" rel="noopener noreferrer">
        <img loading="lazy" src="https://gitpod.io/button/open-in-gitpod.svg" alt="Open in Gitpod" class="img_ev3q">
    </a>
</p>

> **IMPORTANT**: Gitpod is an alternative to local development and completely optional. We recommend setting up a local development environment if you plan to contribute regularly.

The Moodle Gitpod template supports the following environment variables:

* `MOODLE_REPOSITORY`. The Moodle repository to be cloned. The value should be URL encoded. If left undefined, the default repository `https://github.com/moodle/moodle.git` is used.
* `MOODLE_BRANCH`. The Moodle branch to be cloned. If left undefined, the default branch `main` is employed.
* `DATAFILE`. When specified, this feature URL initializes the Moodle site with essential data. The value should be URL encoded. The content of this file adheres to the [Behat generators format](https://moodledev.io/general/development/tools/generator#create-a-testing-scenario-using-behat-generators) for creating testing scenarios.
* `INSTALLADMINER`. Add this variable, set to any value, to install [adminer](https://www.adminer.org/).
* `CLONEALL`. If not set, a shallow clone is created, truncating the history to reduce the clone size. Set to `1` for a full clone.

For a practical demonstration, launch a Gitpod workspace with the 'main' branch patch for [MDL-79912](https://tracker.moodle.org/browse/MDL-79912). Simply open the following URL in your web browser (note that MOODLE_REPOSITORY should be URL encoded). The password for the **admin** user is **test**:

```
https://gitpod.io/#MOODLE_REPOSITORY=https%3A%2F%2Fgithub.com%2Fsarjona%2Fmoodle.git,MOODLE_BRANCH=MDL-79912-main/https://github.com/moodlehq/moodle-docker
```

To optimize your browsing experience, consider integrating the [Tampermonkey extension](https://www.tampermonkey.net/) into your preferred web browser for added benefits. Afterward, install the Gitpod script, which can be accessed via the following URL: [Gitpod script](https://gist.githubusercontent.com/sarjona/9fc728eb2d2b41a783ea03afd6a6161e/raw/gitpod.js). This script efficiently incorporates a button adjacent to each branch within the Moodle tracker, facilitating the effortless initiation of a Gitpod workspace tailored to the corresponding patch for the issue you're currently viewing.

## Companion docker images

The following Moodle customised docker images are close companions of this project:

* [moodle-php-apache](https://github.com/moodlehq/moodle-php-apache): Apache/PHP Environment preconfigured for all Moodle environments. (Note: This project now uses a PHP-FPM variant, e.g., `moodlehq/moodle-php-fpm` or similar, instead of Apache directly for the Moodle application.)
* [moodle-db-mssql](https://github.com/moodlehq/moodle-db-mssql): Microsoft SQL Server for Linux configured for Moodle
* [moodle-db-oracle](https://github.com/moodlehq/moodle-db-oracle): Oracle XE configured for Moodle

## Contributions

Are extremely welcome!

The provided Dockerfile is currently set up for PHP 8.2 and includes extensions for PostgreSQL. To adapt it for a PHP 8.1 Moodle codebase using MariaDB, we'll need to make the following adjustments:

1.    **Update PHP Version**: Change the default `PHP_VERSION` argument to `8.1`.
2.    **MariaDB PHP Extensions**:
    *   Add the system dependency for MariaDB/MySQL client libraries (e.g., `default-libmysqlclient-dev`).
    *   Remove PostgreSQL-specific system dependencies (`libpq-dev`).
    *   Install the `pdo_mysql` and `mysqli` PHP extensions.
    *   Remove the `pdo_pgsql` and `pgsql` PHP extensions.

I'll apply these changes to the Dockerfile.
````
ARG PHP_VERSION=8.1
// ...existing code...
RUN apt-get update && apt-get install -y --no-install-recommends \
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libpng-dev \
    libzip-dev \
    libxml2-dev \
    libicu-dev \
    libxslt1-dev \
    default-libmysqlclient-dev \
    libonig-dev \
    libsodium-dev \
// ...existing code...
RUN docker-php-ext-configure gd --with-freetype --with-jpeg && \
    docker-php-ext-install -j$(nproc) \
    gd \
    intl \
    opcache \
    pdo \
    pdo_mysql \
    mysqli \
    soap \
    zip \
    xsl \
    exif \
    bcmath \
    sodium \
    mbstring
// ...existing code...
````


Made changes.

I've updated the Dockerfile to support PHP 8.1 and MariaDB. Specifically, I've:
*   Changed the base PHP version to 8.1.
*   Ensured the necessary MariaDB/MySQL development libraries (`default-libmysqlclient-dev`) are installed instead of PostgreSQL ones.
*   Included the `pdo_mysql` and `mysqli` PHP extensions, and removed `pdo_pgsql`.

Now, to make your setup ready for a PHP 8.1 Moodle codebase with MariaDB:

1.  **Environment Variable for PHP Version**:
    In your base.yml, the `webserver` (PHP-FPM) service has an argument for `PHP_VERSION`:
    ```yaml
    # In base.yml
    services:
      webserver:
        build:
          context: ./assets/php-fpm
          args:
            PHP_VERSION: ${MOODLE_DOCKER_PHP_VERSION:-8.2} # Default to 8.2 if not set
    ```
    To use PHP 8.1, you need to set the `MOODLE_DOCKER_PHP_VERSION` environment variable to `8.1` in your `.env` file:
    ```env
    MOODLE_DOCKER_PHP_VERSION=8.1
    # ... other environment variables ...
    ```
    If this variable is not set, it will default to PHP 8.2 as per the Dockerfile and base.yml.

2.  **MariaDB Service (`db.mariadb.yml`)**:
    This file defines your MariaDB service. It typically includes the MariaDB image, port mappings (if you use db.mariadb.port.yml), volume for persistent data, and environment variables for the database setup (like `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`).
    The `config.php` I modified earlier will use environment variables like `MOODLE_DBHOST`, `MOODLE_DBNAME`, `MOODLE_DBUSER`, and `MOODLE_DBPASS`. Ensure these correspond to how the MariaDB service is configured or set them in your `.env` file to match the MariaDB container's settings.
    For example, db.mariadb.yml might have:
    ```yaml
    # Example snippet from a typical db.mariadb.yml
    services:
      db:
        image: mariadb:10.11 # Or your desired MariaDB version
        environment:
          MYSQL_ROOT_PASSWORD: your_root_password
          MYSQL_DATABASE: moodle_database_name_from_mariadb_yml
          MYSQL_USER: moodle_user_from_mariadb_yml
          MYSQL_PASSWORD: moodle_password_from_mariadb_yml
        # ... volumes, etc.
    ```
    Then, in your `.env` file for `config.php`:
    ```env
    MOODLE_DBTYPE=mariadb
    MOODLE_DBHOST=db # This is the service name in docker-compose
    MOODLE_DBNAME=moodle_database_name_from_mariadb_yml # Must match MYSQL_DATABASE
    MOODLE_DBUSER=moodle_user_from_mariadb_yml     # Must match MYSQL_USER
    MOODLE_DBPASS=moodle_password_from_mariadb_yml # Must match MYSQL_PASSWORD
    ```

3.  **`config.php` for MariaDB**:
    Ensure your `.env` file sets `MOODLE_DBTYPE` correctly:
    ```env
    MOODLE_DBTYPE=mariadb
    ```
    The rest of the database settings (`MOODLE_DBHOST`, `MOODLE_DBNAME`, etc.) will also be picked up from the `.env` file as configured previously.

4.  **Docker Compose Command**:
    When you run your setup, make sure to include db.mariadb.yml:
    ```bash
    docker-compose -f base.yml -f db.mariadb.yml -f webserver.port.yml up -d
    ```
    (Add other `.yml` files like service.mail.yml as needed).

With these changes and configurations, your environment should be ready to work with a Moodle codebase compatible with PHP 8.1 and a MariaDB database. Remember to restore your MariaDB database backup into the volume used by the MariaDB container, as discussed previously.
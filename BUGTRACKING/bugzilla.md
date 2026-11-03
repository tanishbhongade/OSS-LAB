# Bugzilla 5.2 with Docker Compose, MariaDB and Mailpit

This guide documents the complete Docker setup for Bugzilla 5.2 using:

- Debian Bookworm as the Bugzilla web container base image.
- Apache and mod_perl to serve Bugzilla.
- MariaDB 10.6 as the database.
- Mailpit as a local SMTP testing server.
- Docker Compose to manage all three services.

It also explains the changes made to the original Docker configuration and why they were necessary.

---

## 1. Download Bugzilla

Download the Bugzilla source code from the official Bugzilla website:

**Official download page:** https://www.bugzilla.org/download/

1. Open the [Bugzilla Download page](https://www.bugzilla.org/download/).
2. Select the Bugzilla release you want to use. For this setup, use **Bugzilla 5.2**.
3. Download the source archive.
4. Extract the archive into your project directory.
5. Ensure that the extracted Bugzilla source files are in the same directory as your `Dockerfile` and `docker-compose.yml`.

This Docker configuration is specifically prepared for Bugzilla 5.2. Use the corresponding Bugzilla 5.2 source code.

---

## 2. Project structure

The project should have a structure similar to this:

```text
bugzilla/
├── Dockerfile
├── Dockerfile.mariadb
├── docker-compose.yml
├── docker/
│   ├── 000-default.conf
│   ├── startup.sh
│   ├── my.cnf
│   └── mysql/
│       └── ...
├── Bugzilla source files...
└── ...
```

The `docker/` directory contains the Apache configuration, startup script, and MariaDB configuration files.

---

## 3. Bugzilla web Dockerfile

The web container uses Debian Bookworm.

### Final `Dockerfile`

```dockerfile
FROM debian:bookworm

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
    && apt-get -y dist-upgrade \
    && apt-get -y install \
        apache2 \
        mariadb-client \
        netcat-traditional \
        libappconfig-perl \
        libdate-calc-perl \
        libtemplate-perl \
        build-essential \
        libdatetime-timezone-perl \
        libdatetime-perl \
        libemail-address-perl \
        libemail-sender-perl \
        libemail-mime-perl \
        libemail-mime-modifier-perl \
        libdbi-perl \
        libdbix-connector-perl \
        libcgi-pm-perl \
        liblocale-codes-perl \
        libapache2-mod-perl2 \
        libapache2-mod-perl2-dev \
        libchart-perl \
        libxml-perl \
        libxml-twig-perl \
        perlmagick \
        libgd-graph-perl \
        libtemplate-plugin-gd-perl \
        libsoap-lite-perl \
        libhtml-scrubber-perl \
        libjson-rpc-perl \
        libdaemon-generic-perl \
        libtheschwartz-perl \
        libtest-taint-perl \
        libauthen-radius-perl \
        libfile-slurp-perl \
        libencode-detect-perl \
        libmodule-build-perl \
        libnet-ldap-perl \
        libauthen-sasl-perl \
        libfile-mimeinfo-perl \
        libhtml-formattext-withlinks-perl \
        libgd-dev \
        libmariadb-dev \
        libmariadb-dev-compat \
        graphviz \
        vim-common \
    && rm -rf /var/lib/apt/lists/*

RUN cpan -i Math::Random::ISAAC Math::Random::ISAAC::XS

# Install Perl modules through CPAN.
# Some required module versions were not available
# through the Debian/Ubuntu package repositories.
RUN cpan install Template::Toolkit Email::Address::XS Email::Sender DBD::MariaDB

WORKDIR /var/www/html

COPY --chown=root:www-data . /var/www/html

COPY ./docker/000-default.conf /etc/apache2/sites-available/000-default.conf

COPY ./docker /root/docker

# Remove Docker-related files and sensitive setup files
# from the web-accessible directory.
RUN rm -rf /var/www/html/docker* /var/www/html/Dockerfile*

RUN rm -rf /var/www/html/data \
    /var/www/html/localconfig \
    /var/www/html/index.html \
    && mkdir /var/www/html/data

# Enable the required Apache modules.
RUN a2enmod expires \
    && a2enmod headers \
    && a2enmod rewrite \
    && a2dismod mpm_event \
    && a2enmod mpm_prefork

EXPOSE 80/tcp

CMD ["/bin/bash", "/root/docker/startup.sh"]
```

### Why these changes were made

**1. Debian Bookworm**

```dockerfile
FROM debian:bookworm
```

We use Debian Bookworm as the base image for the Bugzilla web container.

**2. MariaDB development packages**

The original dependency setup used `libmysqlclient-dev`.

That package was unavailable in the Debian Bookworm repositories used for this build. We replaced it with:

```dockerfile
libmariadb-dev
libmariadb-dev-compat
```

These provide the MariaDB development headers and compatibility files needed to build database-related Perl modules.

**3. CPAN installation**

Two modules that were blocked or unavailable through the package repository were installed directly through CPAN:

```dockerfile
RUN cpan -i Math::Random::ISAAC Math::Random::ISAAC::XS
```

Additional Perl modules were installed using:

```dockerfile
RUN cpan install Template::Toolkit Email::Address::XS Email::Sender DBD::MariaDB
```

This was necessary because the college network's HTTP filtering proxy returned HTTP 403 errors for certain package downloads.

Using CPAN allowed us to install the required modules through an alternative distribution source.

**4. APT commands combined**

The original Dockerfile used separate commands:

```dockerfile
RUN apt-get update && apt-get -y dist-upgrade
RUN apt-get -y install ...
```

The updated Dockerfile combines updating, upgrading, installing, and cleaning the APT package lists in one layer.

This reduces unnecessary image layers and avoids retaining package-list files in the resulting image.

**5. Apache modules**

We retain the required Apache module configuration:

```dockerfile
a2enmod expires
a2enmod headers
a2enmod rewrite
a2dismod mpm_event
a2enmod mpm_prefork
```

The prefork MPM is used with the mod_perl configuration.

**6. Startup script**

The container starts using:

```dockerfile
CMD ["/bin/bash", "/root/docker/startup.sh"]
```

This explicitly invokes Bash to execute the startup script.

The script is responsible for waiting for the database, preparing the database if necessary, running Bugzilla's `checksetup.pl`, and keeping the container running.

---

## 4. MariaDB Dockerfile

Create or retain the following `Dockerfile.mariadb`:

```dockerfile
FROM mariadb:10.6

COPY docker/my.cnf /etc/my.cnf

COPY docker/mysql/ /etc/mysql/conf.d/
```

### Why this Dockerfile is different

The original MariaDB Dockerfile included an APT update and distribution upgrade.

We removed those commands because the official `mariadb:10.6` image already provides the database server and its required dependencies.

There is no need to install or upgrade the operating system packages again just to copy the MariaDB configuration files.

We retain:

- `docker/my.cnf`
- `docker/mysql/`

These contain the existing MariaDB configuration used by the project.

---

## 5. Docker Compose configuration

The Compose file manages the three containers:

1. `bugzilla5.web`
2. `bugzilla5.db`
3. `mailpit`

### Final `docker-compose.yml`

```yaml
# This Source Code Form is subject to the terms of the Mozilla Public
# License, v. 2.0. If a copy of the MPL was not distributed with this
# file, You can obtain one at http://mozilla.org/MPL/2.0/.

services:
  bugzilla5.web:
    build:
      context: .
      dockerfile: Dockerfile

    command: ["/bin/bash", "/root/docker/startup.sh"]

    volumes:
      - bugzilla5-data-dir:/var/www/html/data
      - .:/mnt/sync

    tmpfs:
      - /tmp
      - /run/lock

    ports:
      - "8080:80"

    depends_on:
      - bugzilla5.db

    environment:
      BZ_ADMIN_EMAIL: admin@bugzilla.test
      BZ_ADMIN_PASSWORD: "password01!"
      BZ_ADMIN_REALNAME: Test Admin
      BZ_URLBASE: http://127.0.0.1:8080/

      BZ_DB_HOST: bugzilla5.db
      BZ_DB_PORT: "3306"
      BZ_DB_USER: bugs
      BZ_DB_NAME: bugs
      BZ_DB_PASS: bugzilla

      MARIADB_ROOT_PASSWORD: bugzilla

  bugzilla5.db:
    build:
      context: .
      dockerfile: Dockerfile.mariadb

    volumes:
      - bugzilla5-mysql-db:/var/lib/mysql

    tmpfs:
      - /tmp
      - /var/lock

    environment:
      MARIADB_ROOT_HOST: "%"
      MARIADB_ROOT_PASSWORD: bugzilla

  mailpit:
    image: axllent/mailpit:latest
    container_name: bugzilla-mailpit
    restart: unless-stopped

    ports:
      - "8025:8025"
      - "1025:1025"

volumes:
  bugzilla5-mysql-db:
  bugzilla5-data-dir:
```

### Why these Compose changes were made

**1. Separate environment variables**

Originally, an environment YAML anchor was used:

```yaml
environment: &bugzilla_env
```

The same list was reused by the database service:

```yaml
environment: *bugzilla_env
```

This caused the MariaDB container to receive Bugzilla-specific variables such as `BZ_ADMIN_EMAIL` and `BZ_DB_HOST`.

We separated the environment variables.

The web container now receives Bugzilla configuration and the MariaDB root password needed by the startup script.

The database container receives only:

```yaml
environment:
  MARIADB_ROOT_HOST: "%"
  MARIADB_ROOT_PASSWORD: bugzilla
```

This makes the configuration clearer and avoids passing irrelevant settings to MariaDB.

**2. Database hostname**

The web container connects to the database using:

```yaml
BZ_DB_HOST: bugzilla5.db
```

Docker Compose creates an internal network for the services. The database service name acts as its hostname.

Therefore, the web container can reach MariaDB at `bugzilla5.db:3306`.

**3. Persistent volumes**

Two named volumes are used:

```yaml
volumes:
  - bugzilla5-data-dir:/var/www/html/data
```

and:

```yaml
volumes:
  - bugzilla5-mysql-db:/var/lib/mysql
```

They preserve Bugzilla data and MariaDB database files across container recreation.

The additional bind mount:

```yaml
- .:/mnt/sync
```

retains the project-directory mount from the original Compose configuration.

**4. Database startup dependency**

The web service has:

```yaml
depends_on:
  - bugzilla5.db
```

This establishes startup order, but does not guarantee that MariaDB is ready to accept connections.

The startup script handles this by waiting for the database before proceeding.

**5. No `.env` file**

All configuration values are defined directly in `docker-compose.yml`.

There are no `${VARIABLE_NAME}` references that require values from a `.env` file.

Consequently, **a `.env` file is not required for this setup**.

**6. Mailpit service**

Mailpit was added as a third service:

```yaml
mailpit:
  image: axllent/mailpit:latest
  container_name: bugzilla-mailpit
  restart: unless-stopped
  ports:
    - "8025:8025"
    - "1025:1025"
```

It provides a local SMTP server for testing Bugzilla email notifications without sending real emails.

---

## 6. Database configuration and credentials

The current example configuration uses:

| Setting | Value |
|---|---|
| Database name | `bugs` |
| Database user | `bugs` |
| Database password | `bugzilla` |
| Database hostname | `bugzilla5.db` |
| Database port | `3306` |
| MariaDB root password | `bugzilla` |

The web container receives the root password because the startup script uses it during initial database setup.

The database container uses the same root password for MariaDB initialization.

**Important:** These credentials are suitable only as development examples. Use strong passwords before deploying the setup in a shared or production environment.

Also, changing `MARIADB_ROOT_PASSWORD` in Compose does not automatically change the password in an existing initialized MariaDB volume. Existing database credentials must be changed inside MariaDB itself.

---

## 7. Mailpit configuration in Bugzilla

Mailpit captures outgoing email locally.

It exposes:

| Service | Address |
|---|---|
| SMTP server | `mailpit:1025` |
| Web interface | `http://localhost:8025` |

The SMTP hostname is `mailpit`, not `localhost`, because Bugzilla and Mailpit communicate over the Docker Compose network.

### Configure Bugzilla

1. Open Bugzilla.
2. Navigate to **Administration → Parameters → Email**.
3. Configure the following settings:

| Bugzilla parameter | Value |
|---|---|
| `mail_delivery_method` | `SMTP` |
| `mailfrom` | `bugzilla@localhost` |
| `smtpserver` | `mailpit:1025` |
| `smtp_username` | Leave blank |
| `smtp_password` | Leave blank |
| `smtp_ssl` | Off |

Save the configuration.

### Access Mailpit

Open:

```text
http://localhost:8025
```

Emails generated by Bugzilla should appear in the Mailpit interface.

No real email delivery takes place through Mailpit.

---

## 8. Build the Docker images

From the project directory, run:

```bash
docker compose build
```

This builds:

- The Bugzilla web image from `Dockerfile`.
- The MariaDB image from `Dockerfile.mariadb`.

Mailpit is pulled from its published image when needed.

If the build fails, inspect the error carefully. In particular, check whether APT or CPAN downloads are being blocked by the college network proxy.

---

## 9. Validate the Compose configuration

Before starting the containers, run:

```bash
docker compose config
```

This checks the Compose YAML and displays the resolved configuration.

The expected configuration should show:

- `bugzilla5.web` with the Bugzilla environment variables.
- `bugzilla5.db` with only `MARIADB_ROOT_HOST` and `MARIADB_ROOT_PASSWORD`.
- `mailpit` with ports `8025` and `1025`.
- Both named volumes.
- The internal Compose network.

No `.env` file is necessary.

---

## 10. Start the containers

Run:

```bash
docker compose up -d
```

The `-d` option starts the containers in detached mode.

Check their status:

```bash
docker compose ps
```

You should see the three services:

```text
bugzilla5.web
bugzilla5.db
mailpit
```

---

## 11. Inspect logs

To inspect all services:

```bash
docker compose logs
```

To follow the Bugzilla web container:

```bash
docker compose logs -f bugzilla5.web
```

To inspect MariaDB:

```bash
docker compose logs -f bugzilla5.db
```

To inspect Mailpit:

```bash
docker compose logs -f mailpit
```

The Bugzilla startup script should wait for MariaDB, prepare the database if necessary, and run `checksetup.pl`.

---

## 12. Access the services

Once the containers are running:

| Service | URL |
|---|---|
| Bugzilla | `http://localhost:8080` |
| Mailpit | `http://localhost:8025` |

Bugzilla's configured URL base is:

```text
http://127.0.0.1:8080/
```

This is suitable when accessing Bugzilla from the Docker host itself.

If other computers on the network need to access Bugzilla, change `BZ_URLBASE` to the host machine's reachable LAN address, for example:

```text
http://192.168.1.100:8080/
```

Replace the example address with the actual address of the Docker host.

---

## 13. Useful Docker Compose commands

### Start services

```bash
docker compose up -d
```

### Stop services

```bash
docker compose stop
```

### Stop and remove containers and the Compose network

```bash
docker compose down
```

This does not remove named volumes by default.

### Rebuild images

```bash
docker compose build
```

### Rebuild and recreate containers

```bash
docker compose up -d --build
```

### Follow all logs

```bash
docker compose logs -f
```

### Restart a particular service

```bash
docker compose restart bugzilla5.web
```

### Open a shell in the web container

```bash
docker compose exec bugzilla5.web bash
```

### Open a MariaDB shell

```bash
docker compose exec bugzilla5.db mariadb -u root -p
```

Enter the configured MariaDB root password when prompted.

---

## 14. Important notes about persistent data

The Compose configuration uses named volumes:

```yaml
volumes:
  bugzilla5-mysql-db:
  bugzilla5-data-dir:
```

These volumes are important.

- `bugzilla5-mysql-db` stores the MariaDB database.
- `bugzilla5-data-dir` stores Bugzilla's data directory.

Running:

```bash
docker compose down
```

does not normally delete these volumes.

However, running:

```bash
docker compose down -v
```

removes the Compose-managed named volumes as well.

**Be careful:** Removing the database volume can permanently delete your Bugzilla database.

Before removing volumes, back up your database and important Bugzilla files.

---

## 15. Backup the database

A logical database backup can be created with:

```bash
docker compose exec -T bugzilla5.db \
  mariadb-dump -u root -p bugs > bugzilla-backup.sql
```

The command prompts for the root password. Enter the configured MariaDB root password.

Restore a backup with:

```bash
docker compose exec -T bugzilla5.db \
  mariadb -u root -p bugs < bugzilla-backup.sql
```

Again, enter the root password when prompted.

Also back up the Bugzilla `data` directory and any project-specific configuration files that need to be retained.

---

## 16. Summary of changes from the original setup

| Original configuration | Updated configuration | Reason |
|---|---|---|
| `libmysqlclient-dev` | `libmariadb-dev` and `libmariadb-dev-compat` | The original package was unavailable in Debian Bookworm |
| Some Perl modules unavailable through APT | Install selected modules through CPAN | Work around repository availability and HTTP 403 restrictions |
| Separate APT update and install layers | Combined APT update, upgrade, install and cleanup | Reduce unnecessary image layers and remove package lists |
| MariaDB Dockerfile included APT upgrade | Use the official MariaDB image and copy configuration files | Avoid unnecessary operating-system package upgrades |
| Shared Compose environment anchor | Separate web and database environment settings | Avoid passing Bugzilla settings to MariaDB |
| No Mailpit service | Add Mailpit | Test Bugzilla email notifications locally |
| Shell-form or ambiguous startup invocation | Explicit Bash JSON-form `CMD` | Ensure the startup script runs under Bash |
| Environment values potentially externalized | Values defined directly in Compose | No `.env` file required |

---

## 17. Final architecture

The resulting Docker setup looks like this:

```text
                    Host machine
                         |
             -------------------------
             |                       |
        Port 8080                Port 8025
             |                       |
       Bugzilla Web                Mailpit
       Apache + Perl              SMTP + UI
       bugzilla5.web               mailpit
             |
             | Internal Docker network
             |
       MariaDB 10.6
       bugzilla5.db
             |
             |
       Persistent DB volume
       bugzilla5-mysql-db
```

Bugzilla communicates with MariaDB using the internal hostname `bugzilla5.db` and sends test emails to Mailpit using `mailpit:1025`.

The two named volumes preserve the database and Bugzilla data independently of container recreation.

**Final result:** Bugzilla 5.2, MariaDB 10.6 and Mailpit are managed together through Docker Compose, with the configuration kept directly in `docker-compose.yml` and no `.env` file required.


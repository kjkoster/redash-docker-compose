# Redash Docker Compose

This repository contains the files necessary to get started with Redash using
Docker Compose. The intention for this repository is to act as a starting
point for your own setup. 

The description assumes you will be running this on a clean machine, without
Docker installed. Feel free to skip steps that are superfluous for your
environment.

In particular, if you already have a running Postgres instance, you can just
point to that in your `.env` file, instead of setting up a new one.

See also:

* [sample Docker Compose file by Redash](https://github.com/getredash/setup/blob/master/data/docker-compose.yml)
  This demo is based 99% on that example. I just ironed out some of the kinks
  and improved the first-start experience.
* [Redash Environment Variables Settings](https://redash.io/help/open-source/admin-guide/env-vars-settings)
* [Redash Postgres Password Guide](https://www.restack.io/docs/redash-knowledge-redash-postgres-password-guide)

## Creating a `.env` File

**Note**: do not commit your `.env` files to Git. Instead, store it in a safe
place for passwords and secrets, such as a password manager or a vault of some
sort.

In order to keep passwords out of the Git repository, Docker Compose allows us
to use a `.env` file with sensitive environment variables. You have to create
your own, but here is an example:

```shell
REDASH_SECRET_KEY=34545ef55
REDASH_COOKIE_SECRET=99222acaef00
REDASH_REDIS_URL=redis://redis:6379/0

# Note that the password in the URL needs to match `POSTGRES_PASSWORD`
POSTGRES_PASSWORD=998643cabb11
REDASH_DATABASE_URL=postgresql://redash:998643cabb11@postgres:5432
```

Best replace all secrets with new ones.

## Installing Prerequisites

Since we are assuming an empty machine, you probably need to install Docker and
Docker Compose before you can use this demo.

Since we started with a clean machine, Docker may not be installed. Here is
[how to install Docker on Debian 11](https://docs.docker.com/engine/install/debian/).
We'll use the convenience script. That will work on most Linux-likes.

```shell
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh
$ sudo groupadd docker
$ sudo usermod -aG docker $USER
$ exit
```

Note the `exit` at the end. You have to log out and back in again for the group
changes to take effect on your login session. Once logged back in, run
`docker run hello-world` to verify everything works.

## Priming Postgres for Redash

If you just want a playground, you have to create the initial database schema in
Postgres before you can run Redash. If you already have a primed database, you
may skip this step.

First, start **only** the Postgres and Redash instances. The latter will
generate errors and complain, but that is OK for now.

```shell
$ docker compose up -d postgres redis redash
```

Keep the secrets in your `.env` file handy, as you will need them for this step.

Then open a shell inside the running Postgres instance. Here you need to create
the database and the user using the `psql` commands. Note that the password in
the example should be replaced with the one that you set for `POSTGRES_PASSWORD`
in your `.env` file.

```shell
$ docker exec -it redash-docker-compose-postgres-1 /bin/sh
/ # su - postgres
703c84e78120:~$ psql
psql (9.6.24)
Type "help" for help.

postgres=# CREATE USER redash WITH PASSWORD '998643cabb11';
CREATE ROLE
postgres=# CREATE DATABASE redash OWNER redash;
CREATE DATABASE
postgres=# GRANT ALL PRIVILEGES ON DATABASE redash TO redash;
GRANT
postgres=# ^D\q
703c84e78120:~$
/ #
```

With that done we can now open a shell on the Redash instance to set up the initial database schema.

```shell
$ docker exec -it redash-docker-compose-redash-1 /bin/sh
$ echo "REDASH_DATABASE_URL=postgresql://redash:998643cabb11@postgres:5432" > .env
$ bin/run ./manage.py database create_tables
[2024-01-24 10:49:00,099][PID:55][INFO][alembic.runtime.migration] Context impl PostgresqlImpl.
[2024-01-24 10:49:00,099][PID:55][INFO][alembic.runtime.migration] Will assume transactional DDL.
[2024-01-24 10:49:00,131][PID:55][INFO][alembic.runtime.migration] Running stamp_revision  -> e5c7a4e2df4d
$ rm .env
$ exit
```

Finally, clean up and remove the containers. Though sometimes inescapable,
manual operations on containers are a really bad practice and you should always
remove the containers afterwards, to avoid contaminating them.

```shell
$ docker compose down
```

## Starting and Stopping Redash

With the preparations done, you should now have a runnable version of Redash.

Starting is done as:

```shell
$ docker compose up
```

If you want to run everything in the background, add the daemon flag:

```shell
$ docker compose up -d
```

To check all is running:

```shell
$ docker compose ps
NAME                             IMAGE                        COMMAND                  SERVICE            CREATED          STATUS          PORTS
redash-demo_adhoc_worker_1       redash/redash:8.0.0.b32245   "/app/bin/docker-ent…"   adhoc_worker       22 minutes ago   Up 13 minutes   5000/tcp
redash-demo_nginx_1              redash/nginx:latest          "nginx -g 'daemon of…"   nginx              22 minutes ago   Up 13 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp, 443/tcp
redash-demo_postgres_1           postgres:9.6-alpine          "docker-entrypoint.s…"   postgres           23 minutes ago   Up 13 minutes   5432/tcp
redash-demo_redash_1             redash/redash:8.0.0.b32245   "/app/bin/docker-ent…"   redash             22 minutes ago   Up 13 minutes   0.0.0.0:5000->5000/tcp, :::5000->5000/tcp
redash-demo_redis_1              redis:5.0-alpine             "docker-entrypoint.s…"   redis              23 minutes ago   Up 13 minutes   6379/tcp
redash-demo_scheduled_worker_1   redash/redash:8.0.0.b32245   "/app/bin/docker-ent…"   scheduled_worker   22 minutes ago   Up 13 minutes   5000/tcp
redash-demo_scheduler_1          redash/redash:8.0.0.b32245   "/app/bin/docker-ent…"   scheduler          22 minutes ago   Up 13 minutes   5000/tcp
```

To stop Redash:

```shell
$ docker compose stop
```


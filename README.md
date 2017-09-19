
borges [![Build Status](https://travis-ci.org/src-d/borges.svg?branch=master)](https://travis-ci.org/src-d/borges) [![codecov.io](https://codecov.io/gh/src-d/borges/branch/master/graph/badge.svg?token=ETL49e3u1L)](https://codecov.io/gh/src-d/borges) [![GitHub version](https://badge.fury.io/gh/src-d%2Fborges.svg)](https://github.com/src-d/borges/releases)
======

**borges** collects and stores Git repositories.

> I have always imagined that Paradise will be a kind of library.

Borges is a set of tools for collection and storage of Git repositories at large scale.
It is a distributed system, similar to a search engine, that uses a custom
repository storage [file format](https://blog.sourced.tech/post/siva/) and is optimized
for saving storage space and keeping repositories up-to-date.

## Key concepts

 - **Borges producer**: a stand-alone process that reads repository URLs (from RabbitMQ or file) and schedules fetching this repository.

 - **Borges consumer**: a stand-alone process that takes URL from RabbitMQ, clones remote repository and pushes it to the appropriate *Rooted Repository* in the storage (local filesystem or HDFS).

 - **Rooted Repository**: a standard Git repository that stores all objects from all repositories that share common history, identified by same initial commit. It is stored in [Siva](https://github.com/src-d/go-siva) file format.

   ![Root Repository explanatory diagram](TODO)

Consumers and Producers run independently, communicating though a RabbitMQ instance
and storing repository meta-data in PostgeSQL.

Read the borges package godoc for further details on how does borges archive
the repositories.

## CLI

Both producer and consumer are shipped as a single binary,
see `borges --help` to get details about the main commands and their options.

## Setting up borges

Borges needs a database and a message broker to do its job. 
It works with a PostgeSQL database by default and uses RabbitMQ. 
To configuring those, you can use following environment variables:
* `CONFIG_DBUSER`, by default: `testing`
* `CONFIG_DBPASS`, by default: `testing`
* `CONFIG_DBHOST`, by default: `0.0.0.0`
* `CONFIG_DBPORT`, by default: `5432`
* `CONFIG_DBNAME`, by default: `testing`
* `CONFIG_DBSSLMODE`, by default: `disable`
* `CONFIG_DBAPPNAME`, by default: ``
* `CONFIG_DBTIMEOUT`, by default: `30s`

Other important settings are:
* `CONFIG_TEMP_DIR`: Local path to store temporal files needed by the Borges consumer, by default: `/tmp/sourced`
* `CONFIG_BROKER`: by default: `amqp://localhost:5672`
* `CONFIG_ROOT_REPOSITORIES_DIR`: .siva file storage. If no HDFS connection url is provided, this will be a path in local filesystem. Otherwise, it will be an HDFS directory, by default: `/tmp/root-repositories`
* `CONFIG_LOCKING`, by default: `local:`, other options: `etcd:`
* `CONFIG_HDFS`: (host:port) If this property is not provided, all root repositories will be stored into the local filesystem, by default: `""`

## Producer

The producer runs as a service. It determines which repositories should be
updated next and enqueues new jobs for them.

To launch the producer you just have to run it with the default configuration:

    borges producer

Producer reads [mentions](https://github.com/src-d/core-retrieval/blob/master/model/mention.go) from rovers rabbit queue by default, but it can
read urls directly from a file with the cli option:

    borges producer --source=file --file /path/to/file

The file must contain a url per line, it looks like:

```
https://github.com/a/repo1
https://github.com/b/repo2.git
http://github.com/c/repo3
http://github.com/d/repo4.git
```

When jobs fail they're sent to the buried queue. If you want to requeue them, you can pass the `--republish-buried` flag (this only works for the `mentions` source). For example:

```
borges producer --republish-buried
```

So a possible command to launch the producer could be:

```bash
$ CONFIG_DBUSER="user" \
CONFIG_DBPASS="pass" \
CONFIG_DBHOST="postgres" \
CONFIG_DBNAME="borges-db"  \
CONFIG_BROKER="amqp://guest:guest@rabbitmq:5672" \
borges producer --loglevel=debug
```

For more details, use `borges producer -h`.

## Consumer

The consumer runs as a service. It gets jobs from the queue and dispatches them
to a goroutine workers pool.

Each job is a request to update a repository. It can be a new or an existing
one. The remote repository is fetched (incrementally when possible) and each reference
is then pushed to a specific [Rooted Repository](#key-concepts), dedicated to storing all references from repositories that share the same *initial commit*.

Note that borges should be the only one creating and writing to the repository
storage.

To run a consumer instance from the command line with default configuration:

    borges consumer

You can select the number of workers to use, by default it uses 8:

    borges consumer --workers=20

A command you could use to run it could be:

```bash
$ CONFIG_TEMP_DIR="/borges/tmp"  \
CONFIG_ROOT_REPOSITORIES_DIR="/borges/root-repositories"  \
borges consumer --workers=20 --loglevel=debug
```

For more details, use `borges consumer -h`

## Administration Notes

Both the producer and consumer services will run even if they cannot connect to
the queue, or even if the queue is malfunctioning. If the queue does not work,
they will just retry until it does.

# Development

## Build

- `rm Makefile.main; rm -rf .ci` to make sure you will have the last Makefile changes.
- `make dependencies` to download vendor dependencies using Glide.
- `make packages` to generate binaries for several platforms.

You will find the built binaries in `borges_linux_amd64/borges` and `borges_darwin_amd64/borges`.

If you're running borges for the first time, make sure you initialize the schema of the database first. You can do so by running the following command:

```
borges init
```

## Test

`make test`

Borges has 2 runtime dependencies and has tests that depend on them:

  - RabbitMQ

    Consumers and Producers interact through a Queue. You can run one in Docker with the following command:
    ```
    docker run -d --hostname rabbit --name rabbit -p 8080:15672 -p 5672:5672 rabbitmq:3-management
    ```
    Note: a hostname needs to be provided, due to the fact that RabbitMQ stores data according to the host name


  - PostgreSQL

    Consumers creates [siva files] with *Rooted Repositories*, but all repository metadata is stored in PostgreSQL.
    You can run one in Docker with the following command:
    ```
    docker run --name postgres  -e POSTGRES_DB=testing -e POSTGRES_USER=testing -e POSTGRES_PASSWORD=testing  -p 5432:5432 -d postgres
    # to check it manually, use
    docker exec -ti some-postgres psql -U testing
    ```

Use `make test-coverage` to run all tests and produce a coverage report.

## License

GPLv3, see [LICENSE](LICENSE)


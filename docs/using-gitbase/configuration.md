# Configuration

## Environment variables

| Name                             | Description                                         |
|:---------------------------------|:----------------------------------------------------|
| `BBLFSH_ENDPOINT`                | bblfshd endpoint, default "127.0.0.1:9432"          |
| `PILOSA_ENDPOINT`                | pilosa endpoint, default "http://localhost:10101"   |
| `GITBASE_BLOBS_MAX_SIZE`         | maximum blob size to return in MiB, default 5 MiB   |
| `GITBASE_BLOBS_ALLOW_BINARY`     | enable retrieval of binary blobs, default `false`   |
| `GITBASE_UNSTABLE_SQUASH_ENABLE` | enable join squash rule to improve query performance **experimental**. This optimization collects inner joins between tables with a set of supported conditions and converts them into a single node that retrieves the data in chained steps (getting first the commits and then the blobs of every commit instead of joining all commits and all blobs, for example).|
| `GITBASE_SKIP_GIT_ERRORS`        | do not stop queries on git errors, default disabled |

## Command line arguments

```bash
Please specify one command of: server or version
Usage:
  gitbase [OPTIONS] <server | version>

Help Options:
  -h, --help  Show this help message

Available commands:
  server   Starts a gitbase server instance
  version  Show the version information
```

`server` command contains the following options:

```bash
Usage:
  gitbase [OPTIONS] server [server-OPTIONS]

Starts a gitbase server instance

The squashing tables and pushing down join conditions is still a
work in progress and unstable, disable by default. It can be enabled
using a not empty value at GITBASE_UNSTABLE_SQUASH_ENABLE env variable.

By default when gitbase encounters an error in a repository it
stops the query. With GITBASE_SKIP_GIT_ERRORS variable it won't
complain and just skip those rows or repositories.

Help Options:
  -h, --help          Show this help message

[server command options]
      -v              Activates the verbose mode
      -g, --git=      Path where the git repositories are located, multiple directories can be defined. Accepts globs.
          --siva=     Path where the siva repositories are located, multiple directories can be defined. Accepts globs.
      -h, --host=     Host where the server is going to listen (default: localhost)
      -p, --port=     Port where the server is going to listen (default: 3306)
      -u, --user=     User name used for connection (default: root)
      -P, --password= Password used for connection
          --pilosa=   URL to your pilosa server (default: http://localhost:10101)
      -i, --index=    Directory where the gitbase indexes information will be persisted. (default: /var/lib/gitbase/index)
```
AutoTag
=======

[![CI](https://github.com/pantheon-systems/autotag/actions/workflows/workflow.yml/badge.svg)](https://github.com/pantheon-systems/autotag/actions/workflows/workflow.yml)
[![Go Report Card](https://goreportcard.com/badge/github.com/pantheon-systems/autotag)](https://goreportcard.com/report/github.com/pantheon-systems/autotag)
[![Actively Maintained](https://img.shields.io/badge/Pantheon-Actively_Maintained-yellow?logo=pantheon&color=FFDC28)](https://pantheon.io/docs/oss-support-levels#actively-maintained-support)


Automatically increment version tags to a git repo based on commit messages.

- [AutoTag](#autotag)
  - [Dependencies](#dependencies)
  - [Installing](#installing)
    - [Pre-built binaries](#pre-built-binaries)
    - [Docker images](#docker-images)
    - [One-liner](#one-liner)
  - [Usage](#usage)
    - [Scheme: Autotag (default)](#scheme-autotag-default)
    - [Scheme: Conventional Commits](#scheme-conventional-commits)
    - [Pre-Release Tags](#pre-release-tags)
    - [Build metadata](#build-metadata)
  - [Examples](#examples)
    - [Goreleaser](#goreleaser)
  - [Troubleshooting](#troubleshooting)
    - [error getting head commit: object does not exist [id: refs/heads/master, rel_path: ]](#error-getting-head-commit-object-does-not-exist-id-refsheadsmaster-rel_path-)
  - [Build from Source](#build-from-source)
  - [Release information](#release-information)

Dependencies
------------

- [Git 2.x](https://git-scm.com/downloads) available in PATH

Version v1.0.0+ depends on the Git CLI, install Git with your distribution's package management
system.

Versions prior to v1.0.0 use cgo libgit or native golang Git, the binary will work standalone.

Installing
----------

### Pre-built binaries

| OS    | Arch  | binary              |
| ----- | ----- | ------------------- |
| macOS | amd64 | [autotag][releases] |
| Linux | amd64 | [autotag][releases] |

### Docker images

| Arch  | Images                                                           |
| ----- | ---------------------------------------------------------------- |
| amd64 | `quay.io/pantheon-public/autotag:latest`, `vX.Y.Z`, `vX.Y`, `vX` |

[releases]: https://github.com/pantheon-systems/autotag/releases/latest

### One-liner

An install script generated by [godownloader](https://github.com/goreleaser/godownloader) is
available for all supported platforms. This is often a convenient option for CI pipelines.

Examples:

Download and install latest version of `autotag` at `./bin/autotag`:

    curl -sL https://git.io/autotag-install | sh --

Install to a different location using `-b` flag:

    curl -sL https://git.io/autotag-install | sh -s -- -b /usr/bin

Install a specific version of `autotag`:

    curl -sL https://git.io/autotag-install | sh -- v1.2.1

> Only versions v1.2.0+ are supported by the install script.

Usage
-----

The `autotag` utility will use the current state of the git repository to determine what the next
tag should be and then creates the tag by executing `git tag`. The `-n` flag will print the next tag but not apply it.

`autotag` scans the `main` branch for commits by default. If no `main` branch is found, it will
fall back to the `master` branch.  Use `-b/--branch` to scan a different branch. The utility first
looks to find the most-recent reachable tag that matches a supported versioning scheme. If no tags
can be found the utility bails-out, so you do need to create a `v0.0.0` tag before using `autotag`.

Once the last reachable tag has been found, the `autotag` utility inspects each commit between the
tag and `HEAD` of the branch to determine how to increment the version.

Commit messages are parsed for keywords via schemes. Schemes influence the tag selection according
to a set of rules.

Schemes are specified using the `-s/--scheme` flag:

### Scheme: Autotag (default)

The autotag scheme implements SemVer style versioning `vMajor.Minor.Patch` (e.g., `v1.2.3`).

Before using autotag for the first time create an initial SemVer tag,
eg: `git tag v0.0.0 -m'initial tag'`

The next version tag is calculated based on the contents of commit message according to these
rules:

- Bump the **major** version by including `[major]` or `#major` in a commit message, eg:

```
[major] breaking change
```

- Bump the **minor** version by including `[minor]` or `#minor` in a commit message, eg:

```
[minor] new feature added
```

- Bump the **patch** version by including `[patch]` or `#patch` in a commit message, eg:

```
[patch] bug fixed
```

If no keywords are specified a **Patch** bump is applied.

### Scheme: Conventional Commits

Specify the [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/#examples) v1.0.0
scheme by passing `--scheme=conventional` to `autotag`.

Conventional Commits implements SemVer style versioning `vMajor.Minor.Patch` similar to the
autotag scheme, but with a different commit message format.

Examples of Conventional Commits:

- A commit message footer containing `BREAKING CHANGE:` will bump the **major** version:

```
feat: allow provided config object to extend other configs

BREAKING CHANGE: `extends` key in config file is now used for extending other config files
```

- A commit message header containing a _type_ of `feat` will bump the **minor** version:

```
feat(lang): add polish language
```

- A commit message header containg a `!` after the _type_ is considered a breaking change and will
  bump the **major** version:

```
refactor!: drop support for Node 6
```

If no keywords are specified a **Patch** bump is applied.

### Pre-Release Tags

`autotag` supports appending additional test to the calculated next version string:

- Use `-p/--pre-release-name=` to append a pre-release **name** to the version. Pre-release names are subject to the rules outlined in the [SemVer](https://semver.org/#spec-item-9)
  spec.

- Use `-T/--pre-release-timestmap=` to append **timestamp** to the version. Allowed timetstamp
  formats are `datetime` (YYYYMMDDHHMMSS) or `epoch` (UNIX epoch timestamp in seconds).

### Build metadata

Optional SemVer build metadata can be appended to the version string after a `+` character using the `-m/--build-metadata` flag. eg: `v1.2.3+foo`

Build metadata is subject to the rules outlined in the [SemVer](https://semver.org/#spec-item-10)
spec.

A common uses might be the current git reference: `git rev-parse --short HEAD`.

Multiple metadata items should be seperated by a `.`, eg: `foo.bar`

Examples
--------

```console
$ autotag
3.2.1
```

```console
$ autotag -p pre
3.2.1-pre

$ autotag -T epoch
3.2.1-1499320004

$ autotag -T datetime
3.2.1-20170706054703

$ autotag -p pre -T epoch
3.2.1-pre.1499319951

$ autotag -p rc -T datetime
3.2.1-rc.20170706054528

$ autotag -m g$(git rev-parse --short HEAD)
3.2.1+ge92b825

$ autotag -p dev -m g$(git rev-parse --short HEAD)
3.2.1-dev+ge92b825

$ autotag -m $(date +%Y%M%d)
3.2.1-dev+20200518

$ autotag  -m g$(git rev-parse --short HEAD).$(date +%s)
3.2.1+g11492a8.1589860151
```

For additional help information use the `-h/--help` flag:

```console
autotag -h
```

### Goreleaser

`autotag` works well with [goreleaser](https://goreleaser.com/) for automating the process of
creating new versions and releases from CI.

An example of a [Circle-CI](https://circleci.com/) job utilizing both `autotag` and `goreleaser`:

```yaml
jobs:
  release:
    steps:
      - run:
          name: install autotag binary
          command: |
            curl -sL https://git.io/autotag-install | sudo sh -s -- -b /usr/bin
      - run:
          name: increment version
          command: |
           autotag
      - run:
          name: build and push releases
          command: |
            curl -sL https://git.io/goreleaser | bash -s -- --parallelism=2 --rm-dist

workflows:
  version: 2
  build-test-release:
    jobs:
      - release
          requires:
            - build
          filters:
            branches:
              only:
                - master
```

Troubleshooting
---------------

### error getting head commit: object does not exist [id: refs/heads/master, rel_path: ]

```
error getting head commit: object does not exist [id: refs/heads/master, rel_path: ]
```

You may run into this error on certain CI platforms such as Github Actions or Azure DevOps
Pipelines. These platforms tend to make shallow clones of the git repo leaving out important data
that `autotag` expects to find. This can be solved by adding the following commands prior to
running `autotag`:

```sh
# fetch all tags and history:
git fetch --tags --unshallow --prune

if [ $(git rev-parse --abbrev-ref HEAD) != "master" ]; then
  # ensure a local 'master' branch exists at 'refs/heads/master'
  git branch --track master origin/master
fi
```

Build from Source
-----------------

Assuming you have Go 1.5+ installed you can checkout and run make deps build to compile the binary
at `./autotag/autotag`.

```console
git clone git@github.com:pantheon-systems/autotag.git

cd autotag

make test
make build
```

Release information
-------------------

Autotag itself uses `autotag` to increment releases. The default [autotag](#scheme-autotag-default)
scheme is used for version selection.

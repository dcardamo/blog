---
title: "Gobuffalo reflex Setup"
date: 2019-03-27T10:02:36-04:00
tags: ["gobuffalo"]
draft: false
---

I'm using [reflex](https://github.com/cespare/reflex) to automate my testing and linting as I'm working on changes with
[gobuffalo](https://gobuffalo.io).  I'm also using golangci-lint as my meta linter for golang.  Whenever I save a file
in my editor (vim or vscode) then my tests and linting is automatically performed.


## Installation

Install reflex:

```sh
go get -u github.com/cespare/reflex
```

Install golangci-lint using one of its releases.   This gets 1.15.0:

```sh
curl -sfL https://install.goreleaser.com/github.com/golangci/golangci-lint.sh | sh -s -- -b $(go env GOPATH)/bin v1.15.0
```

## golangci-lint setup

Create a file called `.golangci.yml` at the root of your gobuffalo project with the following contents:

{{<highlight yml>}}
linters-settings:
  govet:
    check-shadowing: true
  golint:
    min-confidence: 0.8
  gocyclo:
    min-complexity: 10
  maligned:
    suggest-new: true
  dupl:
    threshold: 100
  goconst:
    min-len: 2
    min-occurrences: 2
  depguard:
    list-type: blacklist
    packages:
      # logging is allowed only by logutils.Log, logrus
      # is allowed to use only in logutils package
      - github.com/sirupsen/logrus
  misspell:
    locale: US
  lll:
    line-length: 140
  goimports:
    local-prefixes: github.com/golangci/golangci-lint
  gocritic:
    enabled-tags:
      - performance
      - style
      - experimental
    disabled-checks:
      - wrapperFunc

linters:
  enable-all: true
  disable:
    - maligned
    - prealloc
    - gochecknoglobals
    - gochecknoinits
    - typecheck
    - ineffassign

run:
  skip-dirs:
    - node_modules

issues:
  exclude-rules:
    - text: "weak cryptographic primitive"
      linters:
        - gosec

# golangci.com configuration
# https://github.com/golangci/golangci/wiki/Configuration
service:
  golangci-lint-version: 1.15.x # use the fixed version to not introduce new linters unexpectedly
  prepare:
    - echo "here I can run custom commands, but no preparation needed for this repo"
{{</highlight>}}

Then try running it to make sure it works for you on the command line.   If this is your first time linting with the
enabled linters it likely you'll have a few findings to fix.

```sh
golangci-lint run
```

## Reflex Setup

Create a file at the root of your buffalo project called `.reflex.conf` with the following contents:
{{<highlight sh>}}
-R '^node_modules/' -r '\.go$' -- buffalo test
-R '^node_modules/' -r '\.go$' -- golangci-lint run
-R '^node_modules/' -r '\.fizz$' -- sh -c 'buffalo db drop; buffalo db create && buffalo db migrate'
{{</highlight>}}

Then you can start reflex in your shell by running:

```sh
reflex -s -R '^node_modules/' -g .reflex.conf -- reflex -d fancy -c .reflex.conf
```

What this setup does is it will start reflex and on any change to the .reflex.conf file it will restart reflex
automatically.   Then if you change any go file it will run your tests and lint in parallel.
If you change a migration file it will recreate your database.  This is helpful because the test database is populated
with a schema from your development database and if you don't do this your tests may fail if they are using the latest
schema.



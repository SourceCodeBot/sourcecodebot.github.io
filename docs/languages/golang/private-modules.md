# Working with private go modules

## publish go modules on github

`go mod init github.com/USER/REPO`

prepare github action

```yaml .github/workflows/main.yaml

name: Publish 

on:
  release:
    types: [published]

jobs:
    build:
        runs-on: ubuntu-latest
        steps:
            - name: Checkout
              uses: actions/checkout@v3
              with:
                token: ${{ secrets.GITHUB_TOKEN }}

            - name: Configure ENV
              run: |
                go env -w GOPRIVATE=github.com/USER/REPO 

            - name: install deps
              run: go mod download

            - name: build
              run: go build .

```


## use private go module in github actions

```yaml .github/workflows/main.yml 

name: Publish 

on:
  release:
    types: [published]

jobs:
    build:
        runs-on: ubuntu-latest
        steps:
            - name: Checkout
              uses: actions/checkout@v3
              with:
                token: ${{ secrets.GITHUB_TOKEN }}

            - name: setup ssh key
              uses: webfactory/ssh-agent@v0.9.0
              with:
                ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

            - name: Configure Env
              run: |
                git config --global url."git@github.com:".insteadOf "https://github.com/"
                go env -w GOPRIVATE=github.com/USER

            - name: install deps
              run: go mod download

            - name: build
              run: go build .


```

## build go app with dockerfile and private modules


create in your repo or home folder (what ever your scope is) a `.env` file with 

```env 
GITHUB_USERNAME=USER
GITHUB_TOKEN=TOKEN_FROM_GITHUB
```

check [Personal Access Tokens](https://github.com/settings/personal-access-tokens) for a token value.

> you need at least contents readonly permission and "all repositories", that the client is able to fetch your go module

```taskfile

version: '3'

vars:
    TAG:
        sh: git describe --tags --always

dotenv: ['.env', '{{.HOME}}/.env']

tasks:
    default:
        cmds:
            - task --list 

    build:
        desc: "build docker image"
        cmds:
            - docker build --build-arg "GITHUB_TOKEN=${GITHUB_TOKEN}" --build-arg "VERSION=${TAG}" -t app:${TAG} .


```

```Dockerfile 
FROM golang:1.23.4-alpine AS builder

WORKDIR /app 

LABEL maintainer="name it"

RUN apk add --no-cache git ca-certificates

ARG GITHUB_TOKEN
ENV CGO_ENABLED=0 GOOS=linux 

ENV GOPRIVATE=github.com/USER/*
RUN go env -w GOPRIVATE=github.com/USER/*

RUN git config --global url."https://${GITHUB_TOKEN}:x-oauth-basic@github.com/".insteadOf "https://github.com/"

COPY go.mod go.sum ./ 
RUN go mod download 

COPY . . 
RUN go build -o app -ldflags "-X main.version=${VERSION}" main.go 

FROM scratch

WORKDIR / 

COPY --form=builder /app/app /app 

ENTRYPOINT ["/app"]

```

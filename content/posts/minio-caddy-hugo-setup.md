---
title: "Minio Caddy Hugo Setup"
date: 2024-02-06T17:48:11+02:00
draft: false
toc: false
images:
tags:
  - tech
  - nixos
---

## Prerequisites

The actual website deployed in a such way. The system consists of:

1. NixOS host
2. Minio S3 server
3. Caddy server
4. Woodpecker pipeline to build and publish Hugo

## Hugo setup

First of all setup and test locally Hudo deploy, to publish statics directly to s3.

```toml
[deployment]
[[deployment.targets]]
name = "production"
URL = "s3://bucket-name?endpoint=https://s3.example.com&region=us-east-1&s3ForcePathStyle=true"
```

Put your minio credentials to `AWS_SECRET_ACCES_KEY` and `AWS_ACCESS_KEY_ID` variables or to `~/.aws/credentials` file and test deployment by `hugo deploy` command.

## Pipeline

The pipeline contains 2 steps, build static with Hugo with `hugomods/hugo` container and then deploy statics into an S3 bucket.
Such behaviour can be easily translated to drone CI.

```yaml
steps:
  build:
    image: hugomods/hugo
    when:
      - event: push
        branch: master
      - event: manual
    commands:
      - hugo
      - hugo deploy
    secrets: [ aws_secret_access_key, aws_access_key_id ] 
```

## Cady setup

First of all, you should set policies in Minio to share bucket content for anonymous users or local connections.

This example uses no extra caddy modules, but alternatively, you can use one of many s3 modules, to simplify setup.

Caddy setup is a little bit tricky because we have to add `index.html` suffix to all paths that have no file suffix to simulate `file_server` logic.
Also, we may handle proxy response with 400 and 403 status by rewriting requests path to `404.html`.

```Caddy
pltanton.dev {
  @r1 path_regexp static ^(.*)/$
  handle @r1 {
    rewrite * /bucket-name/{re.static.1}/index.html
  }

  @r2 path_regexp static ^(.*/[^./]+)$
  handle @r2 {
    rewrite * /bucket-name/{re.static.1}/index.html
  }

  reverse_proxy localhost:9200 {
    @not_found status 403 404
    handle_response @not_found {
      rewrite * /blog/404.html
      reverse_proxy localhost:9200
    }
  }
}
```
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

## Prerequisits

Actual website deployed such way. The system consists of:

1. NixOS host
2. Minio S3 server
3. Caddy server
4. Woodpecker pipeline to build and publish Hugo

## Pipeline

Pipeline contains 2 steps, build static with hugo with hugo container and then put results to s3 bucket via woodpecker pulugin.
Such behaviour can be easily translated to drone CI.

```yaml
steps:
  build:
    image: hugomods/hugo
    commands:
      - hugo
    when:
      - event: push
        branch: master
    
  upload:
    image: woodpeckerci/plugin-s3
    settings:
      bucket: blog
      access_key:
        from_secret: MINIO_ACCESS_KEY
      secret_key:
        from_secret: MINIO_SECRET_KEY
      source: public/**/*
      strip_prefix: public/
      path_style: true
      target: /
      endpoint: https://s3.example.com
    when:
      - event: push
        branch: master
```

## Cady setup

First of all you should setup policies in minio to share bucket content for anonymous user or for local connections.

This example use no extra caddy modules, but alternatively you can use on of many s3 modules, to simplify setup.

Caddy setup a little bit tricky, because we have to add `index.html` suffix to all paths that have no file suffix to simulate `file_server` logic.
Also we may handle proxy response with 400 and 403 status by rewriting request path to `404.html`.
Prefix `blog` here is bucket name with static content.

```Caddy
pltanton.dev {
  @r1 path_regexp static ^(.*)/$
  handle @r1 {
    rewrite * /blog/{re.static.1}/index.html
  }

  @r2 path_regexp static ^(.*/[^./]+)$
  handle @r2 {
    rewrite * /blog/{re.static.1}/index.html
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
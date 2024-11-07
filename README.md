# Overview

You can use this repository to build a Docker image which provides Tailscale ulitiies, including `derper`.
I publish the image irrgularly to [phbjdocker/tailscale](https://hub.docker.com/repository/docker/phbjdocker/tailscale).

This image exists because the upstream Docker image ([tailscale/tailscale](https://hub.docker.com/r/tailscale/tailscale))
does not include the utility `derper` that we can use to deploy our own DERP service. Basically, I am using upstream
facilities to build this image, but I have made two changes:

1. Make `build_docker.sh` build `derper`, see [the patch](https://github.com/bvlgah/tailscale-build/blob/main/patches/000-derper.patch).

2. Modify the Docker image tags created by `build_docker.sh`, which remains functionalities of this image unchanged,
see [the patch](https://github.com/bvlgah/tailscale-build/blob/main/patches/001-tag.patch).

I am using a Github workflow to build and publish this image, see [the workflow](https://github.com/bvlgah/tailscale-build/blob/main/.github/workflows/docker-build.yml).

Thanks Carson Yang for [his great article about running our own DERP servers][carson yang's blog].
This image is inspired by his work while using as many upstream tools as possible and supporting ARM and ARM64.

# Use

Note: I assume you already know how to use Docker and TLS certificates, so this is not a step-by-step tutorial.

I personally run a Tailscale client and a DERP server via Docker Compose on a server that has a domain and a TLS
certificate issued by [Let's Encrypt](https://letsencrypt.org).
The configuration is as follows (
see [tls.yml](https://github.com/bvlgah/tailscale-build/blob/main/compose/tls.yml)):

```
name: tailscale-derper-https

services:
  derper:
    image: ${IMAGE_NAME}:${IMAGE_TAG}
    volumes:
      - type: volume
        source: var-run-tailscale
        target: /var/run/tailscale
      - type: volume
        source: var-lib-tailscale
        target: /var/lib/tailscale
    secrets:
      - source: derper-tls-cert
        target: /run/secrets/${DERP_SERVER_HOSTNAME}.crt
      - source: derper-tls-key
        target: /run/secrets/${DERP_SERVER_HOSTNAME}.key
    ports:
      - 8080:443/tcp
      - 3478:3478/udp
    command:
      - "derper"
      - "--a=:443"
      - "--verify-clients=true"
      - "--hostname=${DERP_SERVER_HOSTNAME}"
      - "--http-port=-1"
      - "--certmode=manual"
      - "--certdir=/run/secrets"
    restart: always
  tailscale:
    image: ${IMAGE_NAME}:${IMAGE_TAG}
    volumes:
      - type: volume
        source: var-run-tailscale
        target: /var/run/tailscale
    environment:
      - "TS_HOSTNAME=${TAILSCALE_HOSTNAME}"
      - "TS_SOCKET=/var/run/tailscale/tailscaled.sock"
      - "TS_STATE_DIR=/var/lib/tailscale"
      - "TS_AUTH_ONCE=true"
    restart: always
volumes:
  var-run-tailscale:
  var-lib-tailscale:
secrets:
  derper-tls-cert:
    file: ${DERP_SERVER_CERT_SOURCE_PATH}
  derper-tls-key:
    file: ${DERP_SERVER_KEY_SOURCE_PATH}
```

You need a file (here is [a template](https://github.com/bvlgah/tailscale-build/blob/main/data/template.env))
which provides value to the following variables:

| Variable | Meaning |
| -------- | ------- |
| `IMAGE_NAME` | Docker image name, e.g. `phbjdocker/tailscale` |
| `IMAGE_TAG` | Docker image tag, e.g. `latest` |
| `TAILSCALE_HOSTNAME` | DERP server domain, e.g. `www.example.com` |
| `DERP_SERVER_CERT_SOURCE_PATH` | The path to your DERP server TLS certificate in the host environment, e.g. `$HOME/certs/www.example.com/cert` |
| `DERP_SERVER_KEY_SOURCE_PATH ` | The path to your DERP server TLS private key in the host environment, e.g. `$HOME/certs/www.example.com/key` |

# Known limitations

Limitations:

1. Do not support self-signed certificates as the upstream repo does, although I have
[another workflow](https://github.com/bvlgah/tailscale-build/blob/main/.github/workflows/docker-build-self-signed.yml)
supposed to build an image that supports self-signed certificates. For this use case, I recommend utilizing Carson Yang's
[image](https://hub.docker.com/r/yangchuansheng/ip_derper) and referring to [his blog][carson yang's blog].

2. If you want to deploy together your DERP server and Tailscale client together via Docker, you need to authenticate
your Tailscale client by either providing a Tailscale auth key or login with your Tailscale account (look your
Docker container log for the login url). For more details, please refer to [the overview of Tailscale's Docker image](https://hub.docker.com/r/tailscale/tailscale).

3. If you try to run a Tailscale client in a Docker container and authenticate it by login, you may encounter a failure.
Try log in again.

[carson yang's blog]: https://icloudnative.io/posts/custom-derp-servers

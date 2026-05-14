# Inception

Containerized infrastructure built from scratch with Docker. A small WordPress site running behind NGINX with TLS, backed by MariaDB — each service in its own container, on a custom network, with persistent volumes. Part of the 42 São Paulo curriculum.

![docker](https://img.shields.io/badge/docker-compose-2496ED?logo=docker&logoColor=white)
![nginx](https://img.shields.io/badge/nginx-TLS%201.2%2F1.3-009639?logo=nginx&logoColor=white)
![mariadb](https://img.shields.io/badge/mariadb-database-003545?logo=mariadb&logoColor=white)
![wordpress](https://img.shields.io/badge/wordpress-php--fpm-21759B?logo=wordpress&logoColor=white)

---

## About

The goal was to build a small production-like setup using only Docker and docker-compose, with one service per container, custom-built images (no pulling pre-baked WordPress / NGINX images from Docker Hub), persistent storage, and a private network between services. The constraints were strict on purpose — every choice had to be deliberate.

## Architecture

```
                ┌──────────────┐
   client ──▶  │  NGINX (443) │ ──┐
                └──────────────┘   │  inception_network
                                   ▼  (custom bridge)
                          ┌──────────────────┐
                          │ WordPress (PHP-FPM) │
                          └──────────────────┘
                                   │
                                   ▼
                          ┌──────────────┐
                          │   MariaDB    │
                          └──────────────┘

                   Volumes (host bind mounts):
                   /home/lmoraes/data/wordpress
                   /home/lmoraes/data/mariadb
```

Three services, each built from a custom Dockerfile based on Debian / Alpine. NGINX is the only entrypoint, exposed on `443`. WordPress and MariaDB are not reachable from outside the network.

## What I built

| Service | Role | Notes |
|---------|------|-------|
| **NGINX** | Reverse proxy + TLS termination | Self-signed cert, TLS 1.2/1.3 only, forwards PHP requests to `wordpress:9000` |
| **WordPress** | PHP-FPM application | Installed and configured via WP-CLI on first boot |
| **MariaDB** | Database | Bootstrapped with a custom init script, user and database created from env vars |

### Constraints that shaped the design

- No `:latest` tags — every base image is pinned to an explicit version
- No pre-built application images — each Dockerfile is written by hand
- Secrets are passed through environment variables, never committed
- All inter-service communication happens on a private custom bridge network
- Data persists across container restarts through bind-mounted host volumes
- Containers auto-restart on failure (`restart: unless-stopped`)

## Running locally

Requires Docker and docker-compose. The repo includes a `Makefile` for the common operations:

```bash
make            # builds and starts all services
make down       # stops the stack
make clean      # removes containers, images, and volumes
make re         # full rebuild
```

The site becomes reachable at `https://<your-host>/` after the containers boot. Credentials and host paths are configured through an `.env` file (template included).

## Project layout

```
.
├── Makefile
└── srcs/
    ├── docker-compose.yml
    ├── .env                # not committed
    └── requirements/
        ├── nginx/
        │   ├── Dockerfile
        │   ├── conf/nginx.conf
        │   └── tools/
        ├── wordpress/
        │   ├── Dockerfile
        │   ├── conf/
        │   └── tools/setup.sh
        └── mariadb/
            ├── Dockerfile
            ├── conf/
            └── tools/init.sh
```

## What I learned

- The difference between a container that **boots** and a container that is actually **ready** — and why entrypoint scripts and healthchecks matter more than the Dockerfile itself
- Why `:latest` is a footgun in any environment you care about reproducing
- How to design a private network so that the database is never reachable from outside, even by mistake
- The pain of bind mounts on permissions when the container user doesn't match the host user
- That secrets in `.env` are fine for dev but anything beyond that needs a real secret manager

## References

- [Docker docs — compose file reference](https://docs.docker.com/compose/compose-file/)
- [NGINX as TLS terminator](https://nginx.org/en/docs/http/configuring_https_servers.html)
- [WP-CLI documentation](https://wp-cli.org/)

# Development and testing environment

How the system runs as a whole on one machine: images, Compose, ports, secrets, the tailnet
sidecar that gives the gateway its address, and the order a stack comes up in. Everything about
the environment lives here; the other references describe the code and say only that a value
"comes from the environment".

## Contents

- Container build
- The suite Compose file
- Postgres and migrations
- Ports
- Secrets and the staged first start
- The gateway's address: a tailnet sidecar
- Log aggregation: Loki and Grafana
- Verifying what a service receives
- Platforms without Compose secrets

## Container build

Every service builds its own static Linux image with the same `Makefile`, pushed straight to the
registry — no Docker daemon involved:

```make
TAG ?= latest

build:
	swift package --swift-sdk aarch64-swift-linux-musl \
		--configuration release \
		--allow-network-connections all build-container-image \
		--product catalog \
		--repository ghcr.io/<organization>/<organization>-catalog \
		--tag $(TAG)

.PHONY: build
```

Replace only service, product, and repository names. The image's creation dates are the epoch, so
image age says nothing about freshness; probe a feature (`--help` listing a subcommand) instead.

## The suite Compose file

One `compose.yml` at the workspace root runs the whole system from published images and never
builds them: `${REGISTRY:-ghcr.io/<organization>}/<image>:${IMAGE_TAG:-latest}` with
`pull_policy: ${PULL_POLICY:-always}`. Beside it, a `.env.example` names every variable with the
required ones left empty, copied to a git-ignored `.env`; a required secret is declared as
`${VAR:?message}` so Compose refuses to start with an actionable error rather than a guessable
default. Each service package also carries a standalone `compose.yaml` for running that service
alone.

Shared configuration is declared once as anchors and merged into every service that needs it:

```yaml
x-postgres-connection: &postgres-connection
  POSTGRES_HOST: postgres
  POSTGRES_PORT: "5432"
  POSTGRES_USER: ${POSTGRES_USER:-emberfilm}
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-emberfilm}

x-jwt-verification: &jwt-verification
  JWT_PUBLIC_KEY_PATH: /run/secrets/jwt-public
```

An explicit key in a mapping wins over `<<:`, which is how a service overrides what an anchor
merges in. Verify an anchor is actually referenced: an unreferenced one is silently ignored, so a
`${VAR:?message}` guard inside it never fires and the stack starts without a value the service
requires.

Consumers reach producers by service name: `GRPC_<SERVICE>_HOST=<service>`,
`GRPC_<SERVICE>_PORT=50051` — the container port, whatever the host publishes. Startup is gated
with `depends_on` conditions, which help ordering and do not replace runtime recovery: the
application must tolerate a dependency restarting.

## Postgres and migrations

One `postgres:18` container for the suite, dedicated volume mounted at `/var/lib/postgresql`
(PostgreSQL 18 changed the image's volume layout; do not set a custom `PGDATA`), a health check,
and no `ports:`. A `postgres-init` job creates one database per service — `emberfilm_users`,
`emberfilm_authentication`, … — if absent. Never share a database or its credentials between
services because they happen to run in one project.

Per service:

- `<service>-migrate`: the service image with `command: ["database", "migrate"]`, gated on
  `postgres-init`, connecting as the owner and carrying `POSTGRES_SERVICE_ROLE` /
  `POSTGRES_SERVICE_PASSWORD` for its first migration to create;
- `<service>`: `command: ["serve"]`, gated on the migration job, and connecting **as the service
  role** by overriding the anchor's credentials;
- `<service>-worker`: the same image with `command: ["worker"]` when the service uses Temporal,
  with the database, dependency, and Temporal configuration its Activities need.

```yaml
entitlements-migrate:
  environment:
    <<: *postgres-connection
    POSTGRES_DB: emberfilm_entitlements
    POSTGRES_SERVICE_ROLE: ${ENTITLEMENTS_SERVICE_ROLE:-entitlements_service}
    POSTGRES_SERVICE_PASSWORD: ${ENTITLEMENTS_SERVICE_PASSWORD:-emberfilm}

entitlements:
  environment:
    <<: [*postgres-connection, *jwt-verification]
    POSTGRES_DB: emberfilm_entitlements
    POSTGRES_USER: ${ENTITLEMENTS_SERVICE_ROLE:-entitlements_service}
    POSTGRES_PASSWORD: ${ENTITLEMENTS_SERVICE_PASSWORD:-emberfilm}
```

The migration library refuses a migration list whose order differs from what a database has
already applied — it throws, it does not revert. A change that inserts a migration before applied
ones (adopting the service-role-first standard on an existing database, say) therefore means
`docker compose down -v` and a fresh start, not an in-place `up`. Appending a migration applies in
place.

## Ports

Publish nothing that nothing outside the stack calls. Postgres, Valkey, Temporal, and the
gateway get no `ports:`; the gateway is reached through its sidecar (below), and internal
services by name. Where a gRPC port is published for development convenience, make it a variable
with a default — `${USERS_HOST_PORT:-50052}:50051` — so a collision is settled in `.env` rather
than by editing the file, and remember that two services defaulting to the same host port fail at
`up`, not at `config`. Reach an internal service with `docker compose exec` rather than opening a
port.

## Secrets and the staged first start

Key material is mounted as files and configured by path — never as an environment variable,
which is readable from `/proc/<pid>/environ`, reported by the runtime's inspect command, and
inherited by every child process:

```yaml
secrets:
  jwt-public:
    file: ./secrets/jwt-public.pem
  jwt-private:
    file: ./secrets/jwt-private.pem
  authentication-secret:
    file: ./secrets/authentication.secret
  authentication-worker-secret:
    file: ./secrets/authentication-worker.secret

services:
  authentication:
    environment:
      <<: [*postgres-connection, *jwt-verification]
      JWT_PRIVATE_KEY_PATH: /run/secrets/jwt-private
      SERVICE_CREDENTIAL_ID: authentication
      SERVICE_CREDENTIAL_SECRET_PATH: /run/secrets/authentication-secret
    secrets:
      - jwt-public
      - jwt-private
      - authentication-secret
```

Only the authenticating service mounts the private key. Every service that verifies tokens
merges `*jwt-verification` and mounts `jwt-public`. Compose refuses to start a service whose
secret's source file is missing, so a stack without keys fails at `up` rather than at the first
request — the same guarantee a `${VAR:?message}` guard gives a variable. `docker compose config`
validates a file whose secret source is missing; only `up` refuses.

A process's service credential is a mounted secret like a key, and it is the one secret that
cannot exist before the stack has run: it is issued against the issuer's migrated database. The
first start is therefore staged, and `.env.example` says so:

```sh
docker compose up -d postgres postgres-init authentication-migrate
docker compose run --rm --no-deps -T authentication service-credentials create --id authentication \
    > secrets/authentication.secret < /dev/null
docker compose run --rm --no-deps -T authentication service-credentials create --id authentication-worker \
    > secrets/authentication-worker.secret < /dev/null
chmod 600 secrets/*.secret
docker compose up -d
```

`--no-deps` because the issuing service itself cannot start until its own secret exists;
`-T … < /dev/null` because `docker compose run` otherwise attaches stdin and, in a script piped
over `ssh`, consumes the rest of the script. Every service that verifies tokens must already be
the build that decodes the process's role before that process starts, or its every call is
refused.

Rotating the signing keys means generating a new pair and restarting every service; access tokens
signed by the old key stop verifying and clients recover on their next refresh, provided refresh
tokens are database rows rather than signed tokens. Rotating a service credential is deleting its
row and issuing another; the old token stops working within its lifetime.

## The gateway's address: a tailnet sidecar

The gateway publishes no host port. A Tailscale sidecar owns its address — node identity,
certificate, and optionally Funnel for the public internet — and the gateway runs inside the
sidecar's network namespace, reached by the sidecar at localhost and by nothing else:

```yaml
api-tailscale:
  image: tailscale/tailscale:latest
  hostname: emberfilm-api                      # the node, and so the DNS name, on the tailnet
  environment:
    TS_AUTHKEY: ${TS_AUTHKEY:-}                # first registration only; state re-registers itself
    TS_SERVE_CONFIG: /config/serve-api.json
    TS_STATE_DIR: /var/lib/tailscale
  volumes:
    - ./state/api:/var/lib/tailscale           # the node's identity and certificate (root-owned)
    - ./config:/config
    - /dev/net/tun:/dev/net/tun
  cap_add:
    - net_admin
    - sys_module
  restart: unless-stopped

api:
  network_mode: service:api-tailscale          # same namespace: the sidecar proxies to localhost
  depends_on:
    api-tailscale:
      condition: service_started
  # no ports
```

`config/serve-api.json` terminates HTTPS on 443 for the node's certificate domain and proxies to
`http://127.0.0.1:8080`; `AllowFunnel` for that domain makes it public. The gateway still resolves
upstreams by service name from inside the shared namespace. Nothing listens on a host port, so
nothing collides with whatever else the machine runs and nothing is reachable by IP.

Two facts that matter at a cutover. The public address *is* the node's identity in `state/api`:
moving a service behind that hostname means moving the state directory into the new stack —
it is root-owned, so copy it through a container (`docker run --rm -v old:/src:ro -v new:/dst
alpine cp -a /src /dst/api`) — not registering a new node. And the serve config routes by path
prefix, longest match first, so a strangler cutover — the new gateway for most paths, the old one
for the few it does not serve yet — is a handler per prefix in one config rather than two
hostnames.

Where no such sidecar exists, publish the gateway's HTTP port as a variable, and still nothing
else.

## Log aggregation: Loki and Grafana

Every process ships its own logs. There is no promtail or log-scraping agent: the in-process
`LokiLogProcessor` batches and pushes to Loki directly (see [composition.md](composition.md)), so
a service that runs anywhere with a route to Loki is aggregated, container or not. Two long-lived
services back this, and one env block points every process at them.

```yaml
# Log store. Single-binary filesystem mode on the image's default config — schema v13, which keeps
# the per-line structured metadata each service attaches. No host port; only Grafana reads it.
loki:
  image: grafana/loki:3.0.0
  command: ["-config.file=/etc/loki/local-config.yaml"]
  volumes:
    - loki-data:/loki
  restart: unless-stopped

# Dashboards. Behind its own tailnet sidecar, exactly like the gateway — a private node, no host
# port — and pointed at Loki out of the box by a provisioned datasource.
grafana:
  image: grafana/grafana:latest
  network_mode: service:grafana-tailscale
  environment:
    GF_SECURITY_ADMIN_USER: ${GRAFANA_ADMIN_USER:-admin}
    GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD:-admin}
  volumes:
    - grafana-data:/var/lib/grafana
    - ./config/grafana/provisioning:/etc/grafana/provisioning   # datasources/loki.yaml
  depends_on:
    loki:
      condition: service_started
    grafana-tailscale:
      condition: service_started
  restart: unless-stopped
```

The `grafana-tailscale` sidecar is the gateway's sidecar with a different `hostname` and
`serve-grafana.json` (proxying `http://127.0.0.1:3000`). `config/grafana/provisioning/datasources/loki.yaml`
declares the Loki datasource (`type: loki`, `url: http://loki:3100`, `isDefault: true`) so Grafana
comes up ready to query with no manual step.

One anchor carries the observability wiring into every service — merged the same way as the
Postgres and JWT anchors, so serve, worker, and migrate alike log with a consistent identity:

```yaml
x-observability: &observability
  LOKI_URL: ${LOKI_URL:-http://loki:3100}   # unset it to log to stdout alone
  ENVIRONMENT: ${ENVIRONMENT:-production}   # names the deployment on every label
  SERVICE_VERSION: ${IMAGE_TAG:-latest}     # the running build, on every label
```

Reuse across a cutover works like the gateway's node: Grafana's identity and its stored dashboards
live in `state/grafana` and `grafana-data`; moving the dashboards behind the same hostname means
moving that state, not registering a new node.

## Verifying what a service receives

`docker compose config <service>` renders exactly the environment, secrets, and ports a service
will get, anchors merged and variables substituted. Use it before `up` whenever an anchor, a
`${VAR:?}` guard, or an override is touched. A live check after `up`: which role each service
connected as (`pg_stat_activity`), which node the sidecar registered (`tailscale status --self`),
and a request through the public hostname from a machine on the tailnet.

## Platforms without Compose secrets

The equivalent of a secret is a file mount plus the path variable. Most such platforms deploy a
container whose mount is missing rather than refusing, so the failure appears in the logs as an
unreadable key instead of a failed deploy — check them after the first rollout rather than reading
a green deploy as proof the mount landed.

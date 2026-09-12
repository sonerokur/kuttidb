# Docker deployment

KuttiDB ships a minimal, non-root Linux image built from a multi-stage
`Dockerfile`: a `build-base` Alpine stage compiles `make TLS=0 kuttidb`,
and the runtime stage is `alpine:3.21` with a dedicated `kuttidb` user
(uid/gid 10001), the binary, and an entrypoint that stages secrets.

- **TLS is disabled in the baseline image** so it carries no OpenSSL runtime.
  Terminate TLS at an ingress, or build a site-specific image with
  `make TLS=1` (then pass `--tls-cert`/`--tls-key`).
- The runtime user is non-root. Auth and metrics tokens arrive through
  mounted secrets, which are typically root-owned; the entrypoint copies
  them to a `0600` private path under the data directory before exec, because
  the server intentionally rejects group/other-readable auth files.

## Published image (GHCR)

Official multi-arch images (`linux/amd64`, `linux/arm64`) are published to
GitHub Container Registry on the same `v*` tags as binary releases by
[`.github/workflows/release-docker.yml`](../../.github/workflows/release-docker.yml).
The workflow always targets `ghcr.io/<owner>/<repo>` for the repository that
runs it (lowercase `GITHUB_REPOSITORY`), so a fork publishes under its own
GHCR namespace. Upstream `kuttidb/kuttidb` publishes:

| Tag | Meaning |
|---|---|
| `ghcr.io/kuttidb/kuttidb:<version>` | Every release tag (leading `v` stripped), e.g. `0.1.0`, `0.1.0-beta.1` |
| `ghcr.io/kuttidb/kuttidb:latest` | Updated only for stable tags (no hyphen in the git tag) |

```sh
docker pull ghcr.io/kuttidb/kuttidb:0.1.0
docker run --rm -p 127.0.0.1:7379:7379 \
  ghcr.io/kuttidb/kuttidb:0.1.0 \
  7379 /var/lib/kuttidb/kuttidb.wal 100
```

Re-publishing an existing version tag is unsupported — cut a new patch tag
instead (same policy as binaries; see [RELEASE.md](RELEASE.md)). After the
first publish, set the GHCR package visibility to **public** if anonymous
pulls are required (one-time GitHub UI setting on the package for that
repository, e.g. `kuttidb/kuttidb` upstream).

Local contributor workflows may keep building `kuttidb:local` via Compose;
use the GHCR image for deployed / released runs.

## Atomic job completion in containers

The image supports the full completion surface, including `TLS=0` builds (no
weak randomness fallback: proofs come from urandom). With Compose, opt in by
setting `KUTTIDB_JOB_COMPLETION=1`; the entrypoint adds `--job-completion`
exactly once. For a custom `docker run` command, pass `--job-completion`
directly (and optionally the `--job-*` budget flags — defaults in
[DEPLOYMENT.md](DEPLOYMENT.md#atomic-job-completion-sizing)):

```yaml
    command:
      # ... existing flags ...
      - "--queue-wal"
      - "/var/lib/kuttidb/queue.wal"
      - "--job-completion"
```

Keep the Queue WAL on the mounted volume (it is the commit authority for
durable state and receipts), preserve non-root ownership, and back the
volume up before first use. The container-recovery test
(`src/test_container_recovery.py`) covers restart-on-volume semantics; the
feature adds no HA claim.

## Image layout

| Path | Purpose |
|---|---|
| `/usr/local/bin/kuttidb` | the single KuttiDB binary |
| `/usr/local/bin/docker-entrypoint` | secret staging, then `exec` the server |
| `/var/lib/kuttidb` | data directory (cache WAL, queue WAL, stream WAL, snapshot), volume-mount this |

## Running

```sh
docker run --rm -p 127.0.0.1:7379:7379 kuttidb:local 7379 /var/lib/kuttidb/kuttidb.wal 100
```

The entrypoint passes arguments straight to the server; see
[PROTOCOL.md](../design/PROTOCOL.md) for the full flag list. A durable container
typically uses:

```sh
docker run -d --name kuttidb \
  -v kuttidb-data:/var/lib/kuttidb \
  -p 127.0.0.1:7379:7379 \
  kuttidb:local 7379 /var/lib/kuttidb/kuttidb.wal 100 - 64 \
    --durability always --queue-wal /var/lib/kuttidb/queue.wal
```

`--durability always` makes every acknowledged write fsync before the
acknowledgement; without a mounted volume the data does not survive the
container. `docker stop` triggers a graceful SIGTERM shutdown, which flushes
and closes all WALs cleanly.

## Healthcheck

The image defines a `HEALTHCHECK` (`nc -z 127.0.0.1 7379`): it passes while
the process accepts client connections. That is a **liveness** signal only.
The durability-aware readiness check is the authenticated HTTP `GET /ready`
on the optional metrics listener (it returns 503 while cache, queue, or
stream persistence is fail-closed); a container-level healthcheck cannot
carry the bearer token, so use it from Kubernetes probes or an authenticated
scraper instead. See [KUBERNETES.md](KUBERNETES.md).

## Multi-architecture

Release builds publish `linux/amd64` and `linux/arm64` as one manifest list
via Buildx/QEMU in `release-docker.yml`. There is no separate PR-time
`ci.yml` multiarch job in this tree; use `workflow_dispatch` on
`release-docker.yml` to exercise the multi-arch build without pushing tags
to GHCR.

## Compose example

[`compose.yaml`](../../compose.yaml) runs one durable instance with a named
volume, an auth token from a file (`KUTTIDB_AUTH_TOKEN_FILE` must point at a
1–1024-byte token file), loopback-only client and metrics ports, and
`--durability always`:

```sh
KUTTIDB_AUTH_TOKEN_FILE=./token docker compose up --build
```

Enable atomic job completion in that stack with:

```sh
KUTTIDB_AUTH_TOKEN_FILE=./token KUTTIDB_JOB_COMPLETION=1 docker compose up --build
```

The metrics scrape is bound to `127.0.0.1:9099` inside the container and
published loopback-only; loopback metric binds require no bearer token, which
keeps the example simple. For anything non-loopback, mount a token file and
pass `--metrics-token-file` as the Kubernetes manifest does.

The Compose file deliberately runs **one instance**: KuttiDB is a single-node
durable store. There is no replication yet, so a volume does not protect
against node loss — see [DURABILITY.md](../design/DURABILITY.md).

## Verifying recovery

`src/test_container_recovery.py` (CI job `container-recovery`) seeds durable
cache, queue, and stream state into a Docker volume, SIGKILLs the container,
restarts on the same volume, and verifies every acknowledged durable item
survived. Run it locally with:

```sh
docker build -t kuttidb:ci .
KUTTIDB_TEST_IMAGE=kuttidb:ci python3 src/test_container_recovery.py
```

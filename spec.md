# Movies App — Specification (Draft)

**Status:** Draft v0.3
**Date:** 2026-05-05

## 1. Overview

Build a small, self-contained, Kubernetes-native HTTP API that serves a catalog
of movies and actors from local data files. The service exposes a read-only
REST API, Prometheus metrics, and structured JSON logs. Prometheus and Grafana
run alongside the service in the same cluster, with a pre-provisioned Grafana
dashboard.

The implementation is greenfield. The implementation language and HTTP
framework are **deliberately unspecified** — any stack (Go + chi, Rust + axum,
Python + FastAPI, TypeScript + Fastify, .NET, etc.) is acceptable
as long as every requirement in this document is met.

## 2. Goals

- Implement the public REST contract (paths, query params, status codes, JSON shapes) defined in §6.
- Run on a single-node **k3s** cluster (and any conformant Kubernetes: k3d, kind, minikube, AKS, EKS, GKE).
- Serve data from versioned local JSON files baked into the container image at `/data` during `docker build`.
- First-class observability: Prometheus metrics, structured JSON logs, Grafana dashboards.
- Reproducible local dev loop: bringing up the full stack on a local k3s/k3d cluster is documented step-by-step in the implementation README (see §9.1).
- Provide a Web Validate-compatible end-to-end test suite that runs against the in-cluster service as part of the inner-loop dev process (§12).

## 3. Non-Goals

- No multi-tenant auth, RBAC, or user management.
- No write APIs — the service is read-only.
- No external/cloud secrets manager integration. If a runtime secret is ever needed, it is supplied via a native Kubernetes `Secret` (env or projected volume) — no Vault, Key Vault, AWS Secrets Manager, etc.
- No horizontal data sharding — the dataset is small and fits in memory.

## 4. Architecture

```
                ┌────────────────────────────────────────────┐
                │              Kubernetes Cluster            │
                │                                            │
   client ──►   │  Ingress ──► movies-api Deployment         │
                │                 │                          │
                │                 ├─ /metrics  (Prometheus)  │
                │                 ├─ /healthz  (liveness)    │
                │                 └─ /readyz   (readiness)   │
                │                                            │
                │  Prometheus Deployment ──► scrapes /metrics│
                │  Grafana Deployment    ──► reads Prom DS   │
                │                                            │
                │  ConfigMap: grafana-dashboards             │
                └────────────────────────────────────────────┘
```

### 4.1 Components

| Component   | Purpose                                     | Image                              |
|-------------|---------------------------------------------|------------------------------------|
| movies-api  | The Web API (language/framework of choice)  | `movies-api:<tag>` (built locally) |
| Prometheus  | Scrapes `/metrics` every 15s                | `prom/prometheus:latest`           |
| Grafana     | Dashboards + Prometheus datasource          | `grafana/grafana:latest`           |

## 5. Data Layer

### 5.1 Storage format

- Source of truth: JSON files committed to the repo under `data/`:
  - `data/movies.json`
  - `data/actors.json`
  - `data/ratings.json`
- Files are UTF-8, pretty-printed for diff-friendliness, with stable key ordering.
- Schemas are defined in §5.5 and must match the API response shapes in §6.

### 5.2 Packaging

- The JSON files under `data/` are copied into the container image at `/data` as part of `docker build` (e.g. `COPY data/ /data/`).

### 5.3 Loading

- On startup the service loads all files into in-memory immutable collections.

### 5.4 Seed data

- A representative seed dataset (a few hundred movies / actors) is committed under `data/`.
- A one-time `tools/` script (any language) may be provided to generate or transform seed data, but is not required at runtime.

### 5.5 Schema

Implementers must determine the JSON schema by inspecting the files under `data/`.

## 6. API Surface

All endpoints are read-only `GET`. JSON responses use `application/json; charset=utf-8`.

| Method | Path                          | Notes                                                                 |
|--------|-------------------------------|-----------------------------------------------------------------------|
| GET    | `/api/movies`                 | Query: `q`, `genre`, `year`, `rating`, `actorId`, `pageNumber`, `pageSize` |
| GET    | `/api/movies/{id}`            | 404 on miss; id format `tt########`                                   |
| GET    | `/api/actors`                 | Query: `q`, `pageNumber`, `pageSize`                                  |
| GET    | `/api/actors/{id}`            | 404 on miss; id format `nm########`                                   |
| GET    | `/api/genres`                 | Returns array of strings                                              |
| GET    | `/healthz`                    | Plaintext (`pass` / `warn` / `fail`)                                  |
| GET    | `/version`                    | Plaintext semver only, e.g. `1.2.3`                                   |
| GET    | `/metrics`                    | Prometheus exposition                                                 |
| GET    | `/readyz`                     | 200 only after dataset loaded                                         |
| GET    | `/`                           | Redirects to `/swagger`                                               |
| GET    | `/swagger`                    | Swagger UI for the OpenAPI spec                                       |
| GET    | `/swagger/v1/swagger.json`    | OpenAPI 3 document                                                    |

Validation rules (must return HTTP 400 on violation):

- `pageNumber` ∈ [1, 10000] (default: `1` when omitted)
- `pageSize` ∈ [1, 1000] (default: `25` when omitted)
- `year` - examine data
- `rating` - examine data
- `q` length ∈ [2, 20] when present
- `actorId` matches `^nm\d{5,9}$`; `movieId` matches `^tt\d{5,9}$`

### 6.1 `/version` response

`GET /version` returns `200 OK` with `Content-Type: text/plain; charset=utf-8` and a body containing **only** the semver string of the running build, with no surrounding whitespace, JSON, or trailing newline beyond a single `\n`. Example body:

```
1.2.3
```

- The endpoint must respond `200` even before the dataset has finished loading (it does not depend on `/readyz`).

## 7. Observability

### 7.1 Metrics (Prometheus)

Exposed at `/metrics` in Prometheus text exposition format. Implementers should use the idiomatic Prometheus client library for their language.

### 7.2 Structured logging

- Logs written to **stdout** as one JSON object per line.
- Levels: `debug`, `info`, `warn`, `error`. Configurable via `MOVIES_LOG_LEVEL` (default `info`).
- No PII; query strings are logged but request bodies are not (service is GET-only).
- Log schema is language-agnostic — any library is acceptable as long as the field names match.

### 7.3 Grafana dashboard

- Provisioned automatically
- A `prometheus` datasource is provisioned automatically
- Anonymous viewer access enabled for local dev; admin password set via env in dev overlay only.

## 8. Kubernetes Manifests

Every deployable component (movies-api, Prometheus, Grafana, and any future addition) is delivered exclusively as Kubernetes manifests managed by **Kustomize** (`base/` + `overlays/`). Helm charts are out of scope and must not be added.

### 8.1 movies-api Deployment requirements

- `livenessProbe`: `GET /healthz`
- `readinessProbe`: `GET /readyz`
- `resources.requests`: 100m CPU, 128Mi memory
- `resources.limits`: 500m CPU, 512Mi memory
- `securityContext`: non-root (uid 1000), read-only root FS, drop ALL caps, no privilege escalation
- Container listens on port 8080 (api + `/metrics`); split to 9090 if the implementation prefers a separate metrics port.
- Prometheus is deployed and managed by the **Prometheus Operator**; metrics are scraped via a `ServiceMonitor` (`movies-servicemonitor.yaml`). Plain `prometheus.io/scrape` pod/service annotations are not used and must not be relied on.

## 9. Build & Packaging

- Single multi-stage `Dockerfile` producing a minimal runtime image (distroless or Alpine equivalent for the chosen language).
- Image runs as a **non-root** user.
- Build/test commands are exposed via the language's idiomatic tooling
- A `Makefile` is optional
- full automation of the cluster bring-up is **not** required.
- A `devcontainer.json` is encouraged but not required.

### 9.1 Local dev loop (documented in implementation README)

The implementation README must walk a new contributor through the following steps. They may be wrapped in scripts or `make` targets, but each step must be runnable on its own from the command line.

## 10. Testing & Benchmarks

### 10.1 Unit tests

- Idiomatic unit-test framework for the chosen language. Cover:
- Coverage target: ≥ 80% line coverage on data and HTTP layers.

### 10.2 Integration tests

- In-process HTTP tests against a known dataset

### 10.3 End-to-end / contract tests

- A validation suite executed against the in-cluster service as part of the inner-loop dev process (§12).
- Suite must cover every endpoint in §6 plus negative cases for each validation rule.

### 10.4 Benchmarks

- Performance targets:
  - p95 `/api/movies` < 50 ms in-cluster
  - p95 `/api/movies/{id}` < 10 ms
  - Sustained 500 RPS on a single 500m-CPU pod with < 1% error rate

## 11. Configuration

Config is sourced from three layers, listed in **increasing** precedence (later layers override earlier ones):

1. **Built-in defaults** (column below).
2. **Environment variables** (12-factor); no on-disk config files required at runtime.
3. **Command-line flags** passed to the binary.

Each setting has all three forms. Flag names are the kebab-case equivalent of the env var with the `MOVIES_` prefix dropped.

| Env var                       | CLI flag                  | Default       | Purpose                                |
|-------------------------------|---------------------------|---------------|----------------------------------------|
| `MOVIES_DATA_DIR`             | `--movies-data-dir`       | `/data`       | Where data files are mounted           |
| `MOVIES_LOG_LEVEL`            | `--movies-log-level`      | `info`        | Minimum log level                      |
| `MOVIES_PORT`                 | `--movies-port`           | `8080`        | HTTP listen port                       |

Additional rules:

- Boolean flags accept `true`/`false`, `1`/`0`, `yes`/`no` (case-insensitive).
- Unknown flags or invalid values cause the process to exit non-zero before the HTTP listener starts.
- `--help` / `-h` prints all flags, their env-var equivalents, defaults, and current effective values, then exits 0.
- The effective configuration (with secret values redacted) is logged once at `info` level on startup.

## 12. Inner-Loop Dev Process

The contract is a **repeatable, fully local inner loop** that any contributor can execute on their workstation against the local k3s/k3d cluster from §9.1. The loop must be runnable end-to-end in a few minutes and produce reproducible results.

For each iteration:

1. **Make a change** to source, manifests, or data files.
2. **Bump the version**
3. **Build the image**
4. **Deploy the new version**
5. **Verify the version is live**
6. **Run validation tests**
7. **Inspect metrics on the Grafana dashboard**
8. **Iterate**

Requirements:

- The implementation README documents every command in this loop verbatim. A fresh clone on a clean machine must reach a successful step 7 by following only that README.
- Validation suites and the dashboard JSON live in the repo and are versioned alongside the code.
- The loop has no external network dependencies beyond pulling base images and language toolchains.

## 13. Security

- Non-root container, read-only root FS, no Linux capabilities, no privilege escalation.
- NetworkPolicy: movies-api ingress only from the Ingress controller and Prometheus pod.
- No secrets are expected at runtime (data files are public catalog data). If a secret becomes necessary (e.g. Grafana admin password, OTLP auth token), it must be delivered via a native Kubernetes `Secret` referenced by `envFrom`/`valueFrom.secretKeyRef` or a projected volume — never baked into images, ConfigMaps, or repo files.
- Dependency scanning via the language's standard auditor (run locally as part of the inner loop).

## 14. Acceptance Criteria

- [ ] Following the documented dev-loop steps in §9.1 brings up movies-api + Prometheus + Grafana on a fresh local k3s cluster.
- [ ] All endpoints in §6 respond per the contract; baseline + benchmark Web Validate suites pass.
- [ ] `/metrics` exposes all metrics in §7.1 with the specified names and labels.
- [ ] Logs are valid JSON (one object per line) with all required fields in §7.2.
- [ ] Grafana dashboard auto-provisions and shows live data from Prometheus.
- [ ] Container image runs as non-root with a read-only root FS.
- [ ] The inner-loop dev process in §12 runs end-to-end on a clean machine: build new version → deploy to local k3s → `/version` returns the new semver → validation tests pass → the test run is visible on the Grafana dashboard.

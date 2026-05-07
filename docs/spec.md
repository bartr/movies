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

### 1.1 A note on what this spec deliberately does *not* tell you

This spec is intentionally incomplete. That is not an oversight — it is the
experiment.

The baseline being replicated (a 4-person team, 26 weeks, ~2020) started with
*less* information than this document gives you. They had no schema, no
canonical data shape, no list of metric names, no panel layout, no
pre-negotiated query semantics, and no agreement on what "observable" meant
in practice. They figured all of it out as they went — and a meaningful share
of those 26 weeks was spent on exactly that figuring-out.

Several things in this spec are therefore left to your judgment on purpose:

- **Data schema.** §5 tells you the files exist and where they live. The
  shapes of the JSON objects are for you to read and decide how to model.
- **Query semantics.** §6 lists the query parameters. What `q` matches on,
  whether filters combine as AND or OR, default sort order, and tie-breaking
  are design decisions, not spec gaps.
- **Metric names, labels, and histogram buckets.** §7.1 requires Prometheus
  metrics. *Which* metrics, with *what* labels and *what* bucket boundaries,
  is part of the work. The choices you make are part of the evidence.
- **Dashboard panel set.** §7.3 requires a provisioned Grafana dashboard. The
  panels, queries, and layout are yours to design against the metrics you
  chose to emit.
- **Inner-loop ergonomics.** §12 defines *what* the inner loop must do.
  *How* it is wrapped (scripts, `make`, `just`, `task`, language-native
  tooling) is your call.
- **The HTTP replay / load-test tool.** §10.3 and §10.4 require an
  in-repo tool that drives the API end-to-end and produces the numbers
  for §10.4. You build that tool. Off-the-shelf tools (k6, Locust,
  Vegeta, hey, wrk, JMeter, Gatling) are **out of scope**. The 2020
  baseline built theirs; building yours is part of the work being
  measured.

If you find yourself wishing the spec answered one of these for you, that is
the methodology working as intended. Research it, make a defensible call,
record the decision in the session log, and move on. The hypothesis under
test is that **sessions + an AI inner loop close the discovery gap that used
to take a team a quarter** — not that you can implement a fully-specified
contract quickly. Specifying the answers would measure typing speed, not
methodology.

What this spec *does* fix — the public contract surface (§6 paths and status
codes), the deployment shape (§4, §8), the performance bar (§10.4), the
acceptance checklist (§14) — is what makes runs comparable across
participants. Everything else is the experiment.

## 2. Goals

- Implement the public REST contract (paths, query params, status codes, JSON shapes) defined in §6.
- Run on a single-node **k3s** cluster (and any conformant Kubernetes: k3d, kind, minikube, AKS, EKS, GKE).
- Serve data from versioned local JSON files baked into the container image at `/data` during `docker build`.
- First-class observability: Prometheus metrics, structured JSON logs, Grafana dashboards.
- Reproducible local dev loop: bringing up the full stack on a local k3s/k3d cluster is documented step-by-step in the implementation README (see §12).
- Provide an HTTP replay tool end-to-end test suite that runs against the in-cluster service as part of the inner-loop dev process (§12).

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

- Source of truth: JSON files committed to the repo under `src/data/`:
  - `src/data/movies.json`
  - `src/data/actors.json`
  - `src/data/ratings.json`
- Files are UTF-8, pretty-printed for diff-friendliness, with stable key ordering.
- Schemas are defined in §5.5 and must match the API response shapes in §6.

### 5.2 Packaging

- The JSON files under `src/data/` are copied into the container image at `/data` as part of `docker build` (e.g. `COPY src/data/ /data/`).

### 5.3 Loading

- On startup the service loads all files into in-memory immutable collections.

### 5.4 Seed data

- A representative seed dataset (a few hundred movies / actors) is committed under `src/data/`.
- A one-time `tools/` script (any language) may be provided to generate or transform seed data, but is not required at runtime.

### 5.5 Schema

Implementers must determine the JSON schema by inspecting the files under `src/data/`.

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

## 10. Testing & Benchmarks

### 10.1 Unit tests

- Idiomatic unit-test framework for the chosen language. Cover:
- Coverage target: ≥ 80% line coverage on data and HTTP layers.

### 10.2 Integration tests

- In-process HTTP tests against a known dataset

### 10.3 End-to-end / contract tests

- A validation suite executed against the in-cluster service as part of the inner-loop dev process (§12).
- Suite must cover every endpoint in §6 plus negative cases for each validation rule.
- The validation suite is driven by an **in-repo HTTP replay tool that you build as part of this project.** Off-the-shelf load-test or HTTP-replay tools (k6, Locust, Vegeta, hey, wrk, JMeter, Gatling, etc.) are **out of scope** — do not use them. The replay tool, its scenario/case format, and its assertion model are part of the deliverable. The same tool drives the §10.4 benchmark numbers; one tool, two modes (functional contract assertions and sustained load).
- Language and design of the replay tool are your call — it does not need to be the same language as the API. A small CLI that reads scenario files, issues requests, asserts status + JSON, and reports pass/fail and latency is sufficient.

### 10.4 Benchmarks

- Performance targets:
  - p95 `/api/movies` < 50 ms in-cluster
  - p95 `/api/movies/{id}` < 10 ms
  - Sustained 500 RPS on a single 500m-CPU pod with < 1% error rate
- Numbers are produced by the same in-repo replay tool from §10.3 driven in a sustained-load mode. Off-the-shelf load generators are out of scope (see §10.3).

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

The contract is a **repeatable, fully local inner loop** that any contributor can execute on their workstation against a local k3s/k3d cluster. The loop must be runnable end-to-end in a few minutes and produce reproducible results.

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

- [ ] Following the documented dev-loop steps in §12 brings up movies-api + Prometheus + Grafana on a fresh local k3s cluster.
- [ ] All endpoints in §6 respond per the contract; the in-repo replay tool's baseline (functional) and benchmark (sustained-load) modes both pass against a freshly-deployed cluster.
- [ ] `/metrics` exposes all metrics in §7.1 with the specified names and labels.
- [ ] Logs are valid JSON (one object per line) with all required fields in §7.2.
- [ ] Grafana dashboard auto-provisions and shows live data from Prometheus.
- [ ] Container image runs as non-root with a read-only root FS.
- [ ] The inner-loop dev process in §12 runs end-to-end on a clean machine: build new version → deploy to local k3s → `/version` returns the new semver → validation tests pass → the test run is visible on the Grafana dashboard.

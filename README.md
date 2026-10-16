<p align="center">
  <img src="assets/logo.svg" alt="Bifröst" width="96" height="96">
</p>

<h1 align="center">Bifröst</h1>

<p align="center"><b>A K8s based PaaS for developers by developers.</b></p>

Bifröst turns a Kubernetes cluster into a platform you can ship to. Create an **App**, get an isolated namespace with quotas, a closed
network and the right permissions, plug in only the data services it needs, and deploy from any CI with short-lived credentials. No
long-lived kubeconfigs, no cluster-admin tokens in your pipelines.

It ships as a single Go binary (API, embedded web portal, token issuer) plus the `bifrost` command-line client. **This repository
distributes the CLI releases.**

## What you get

- **Apps** – one namespace per App with CPU/memory/pod quotas, default-deny network policy, Pod Security `restricted`, and RBAC scoped to
  that App only.
- **Passwordless CI** – federate your CI's ID token (GitLab `id_tokens` today) or use a service-account key file. Tokens last about an
  hour; there is nothing to leak in your pipeline.
- **`kubectl` that just works** – `bifrost get-credentials` wires a kubectl context through a short-lived SSH-certificate tunnel, so the
  Kubernetes API does not have to be exposed.
- **Optional building blocks per App** – PostgreSQL, Redis, NATS, MongoDB and S3-compatible storage, each created as an isolated tenant
  with its connection Secret in your namespace. Add them only when you need them.
- **Queues, backups, runners, replication** – a simplified view over BullMQ/JetStream queues, scheduled backups, CI runner pools and
  database replication, linking out to the native tool for anything advanced.
- **Everything is optional** – components are switched on and off from a YAML configuration file; Bifröst shows only what you enable.
- **Portal, API and CLI** – the same operations through the web portal, the HTTP API (`/v1`) and the CLI.

Bifröst grew inside gravity, a self-hosted infrastructure stack, but is **not tied to it**: it is meant to run on any Kubernetes
cluster, on any server or cloud (developed and tested on K3s).

> **Status:** under active development. Expect rough edges and breaking changes.

> **No release has been published yet** — the install commands below will return 404 until the first one (v0.1.0) is out.

## Requirements (server side)

- Kubernetes 1.31+ (structured authentication configuration is used to trust Bifröst-issued tokens).
- An OIDC provider for people (Auth0 is the one tested) and, for CI federation, your CI's OIDC issuer.
- HTTPS ingress for the token endpoint (`plane`) and a way to protect the web portal (today it verifies Cloudflare Access JWTs).

## Install the CLI

Linux and macOS (the installer verifies the SHA-256 checksum before installing):

```sh
curl -fsSL https://github.com/acalvoa/bifrost/releases/latest/download/install.sh | sh
```

Windows: download `bifrost_windows_amd64.exe` (or `arm64` / `386`) from the
[latest release](https://github.com/acalvoa/bifrost/releases/latest) and put it on your `PATH`.

| OS | Architectures |
|---|---|
| macOS | Apple Silicon (arm64), Intel (amd64) |
| Linux | amd64, arm64, 386 |
| Windows | amd64, arm64, 386 |

Every release includes `SHA256SUMS`. Verify by hand with `sha256sum -c SHA256SUMS --ignore-missing`. Shell completion:
`bifrost completion bash|zsh|fish`.

## Using the CLI

### Accounts: point at a Bifröst and sign in

An *account* is a name + the address of one Bifröst + an identity. You can keep several (for example one per environment or customer) and
pick one per command with `-a/--account` or `BIFROST_ACCOUNT`.

```sh
bifrost auth login https://plane.example.com --account prod     # people: Auth0 device flow, opens your browser
bifrost auth login https://plane.staging.example.com --account staging
bifrost account list                                            # accounts, addresses, active one
bifrost account use staging                                     # change the default
bifrost apps list -a prod                                       # or just for this command
```

The address is never assumed: if you leave it out the CLI asks for it (or fails with a hint when there is no terminal).

### Apps

```sh
bifrost apps create shop --description "Online store" --cpu 2 --memory 4Gi --pods 20
bifrost apps list
bifrost apps get shop
bifrost apps stats shop                    # CPU, memory, restarts
bifrost apps history shop                  # who changed what
bifrost apps delete shop                   # archives it (--purge removes it after the retention period)
```

### Services (optional, per App)

```sh
bifrost services list                      # catalogue: postgres, redis, nats, mongo, silo (S3)
bifrost services describe postgres         # options and operations
bifrost services add shop postgres         # isolated database + role; connection Secret lands in the App's namespace
bifrost services usage shop postgres
bifrost services remove shop postgres      # deletes its data, asks for confirmation
```

### kubectl

```sh
bifrost get-credentials gravity --app shop      # creates the context bifrost-<account>-shop
kubectl --context bifrost-prod-shop get pods
bifrost logs shop -f                            # logs of the App's pods
```

Each account gets its own local tunnel port, so several clusters can be open at the same time (`bifrost tunnel status --all`).

### CI/CD without secrets (GitLab)

Link the repository to an App once:

```sh
bifrost sa bindings add shop --project-path acme/shop --issuer https://gitlab.com --role deploy --ref-protected
```

Then in `.gitlab-ci.yml`:

```yaml
deploy:
  image: alpine
  id_tokens:
    BIFROST_ID_TOKEN:
      aud: https://plane.example.com          # your Bifröst address
  script:
    - curl -fsSL https://github.com/acalvoa/bifrost/releases/latest/download/install.sh | BIFROST_BIN_DIR=. sh
    - ./bifrost auth federate --api https://plane.example.com --provider gitlab --id-token-env BIFROST_ID_TOKEN --app shop
    - ./bifrost get-credentials gravity --app shop
    - kubectl apply -f k8s/
```

Any other CI can use a **service-account key file** instead:

```sh
bifrost sa create shop deploy-bot --role deploy --out key.json    # shown once; store it as a CI secret
bifrost auth activate-service-account key.json                    # in the pipeline
```

### Queues, runners, replication

```sh
bifrost queues list
bifrost runners list                       # orchestrator state, origins, pools, machines
bifrost replication list                   # replication links and their health
bifrost replication lag <link>
bifrost open portal                        # open the web portal (also: ratatoskr, orchestrator)
```

### Scripting

Every command accepts `-o json`. Exit codes: `0` ok, `1` failure, `2` usage error. Destructive commands ask you to type the name unless you
pass `--yes`.

## How it stays secure

- Tokens are short-lived (about 1 hour); SSH certificates for the tunnel last 5 minutes and can only forward to the Kubernetes API.
- A repository can only deploy to the App it is linked to; Apps cannot create RBAC or touch other namespaces. Admission policies stop
  privileged pods, host mounts and routes to hostnames that do not belong to the App.
- Service-account keys are shown once, only the public key is stored, and they can be rotated or revoked.

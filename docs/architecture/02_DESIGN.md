# Design — the Go CLI

This document covers the `homeserverctl` Go CLI: how it talks to Kubernetes,
how config/manifests are organized so the same commands work identically on
the Debian prod box and a MacBook dev cluster, and how it replaces the pieces
of `auto-homelab`'s CLI that shelled out to Docker. See
[ARCHITECTURE.md](01_ARCHITECTURE.md) for the system it's operating on.

## What we're keeping from auto-homelab

The old Go CLI's *shape* is worth reusing even though its *transport* isn't:

- Cobra command tree, `slog`-based logging, `--log-level` flag.
- Heavy interface/dependency-injection style — `Runner`, `Env`,
  `FilesHandler`-type interfaces with real + test-mock implementations. This
  made the old code genuinely well-tested and should carry over directly.
- The `configure` command's idea of a pluggable strategy registry for
  acquiring config values (constant / generated-random / IP / string / path) —
  same idea, initially writing into plain Kubernetes `Secret`s instead of a
  `.env` file, later into Vault once Phase 2 (see below) happens.

What's being replaced:

- Every `docker compose` / `docker exec` shell-out → talking to the
  Kubernetes API directly via [`client-go`](https://github.com/kubernetes/client-go),
  the official Go client library. 📘 This is the same API `kubectl` itself
  uses — no shelling out to `kubectl` as a subprocess, no scraping its text
  output.
- `--docker-context` (a recurring pain point in the old code, threaded through
  every command) → `--kubeconfig` / `--context`, the standard Kubernetes way
  of pointing a client at a specific cluster.
- `.env` file → Vault + generated Kubernetes `Secret`/`ConfigMap` objects.

## Command tree (proposed)

```
homeserverctl cluster up [--env=dev|prod]     # bootstrap kubeadm (prod) or kind (dev)
homeserverctl cluster down [--env=dev|prod]   # tear the cluster down
homeserverctl cluster status                  # node/pod health summary

homeserverctl secrets generate [service]      # generate + apply Secret(s) via strategy registry
homeserverctl secrets rotate <service> <key>  # regenerate one GENERATED value
homeserverctl secrets sync                    # ensure Secrets (or ExternalSecrets, post-Vault) are in sync

# Phase 2 additions, once Vault is introduced:
homeserverctl secrets init                    # first-time Vault init + unseal-key output
homeserverctl secrets unseal                  # prompt for unseal key shares
homeserverctl secrets set <path> <key>=<val>  # write a secret into Vault

homeserverctl apply [service...]              # apply manifests for one/all services
homeserverctl service add <name>              # scaffold a new service's manifest set

homeserverctl backup run [service]            # trigger a CronJob's Job immediately
homeserverctl backup list [service]           # list restic snapshots
homeserverctl backup restore <service> <snap> # scale down, restore, scale up

homeserverctl up                              # cluster up + secrets sync + apply everything
homeserverctl down                            # apply's inverse / cluster down
homeserverctl status                          # top-level health across everything
```

📘 **Why `up`/`down` at both the cluster and top level:** `cluster up` only
gets you an empty Kubernetes cluster (like a bare OS install). The top-level
`up` is the "one command to bring up the whole homeserver" you asked for —
cluster, secrets, and every service, in dependency order.

## Config & manifest layout — sharing values between dev and prod

You asked for either a repo-committed YAML/Helm-values tree, or a clear
convention for sharing values between dev and prod. The standard Kubernetes
answer to exactly this is **Kustomize**.

📘 **Kustomize:** a way to describe one set of "base" manifests plus small,
explicit *overlays* that patch them per environment — without templating
syntax (no `{{ }}`) and without duplicating the whole manifest per
environment. It ships built into `kubectl` (`kubectl apply -k`), and
`client-go` can consume the same directory tree.

Proposed layout, following the pattern already prototyped in auto-homelab's
`kubernetes/grimmory/` testbed:

```
manifests/
  base/
    system/
      istio/
      cert-manager/
      vault/
      external-secrets/
    apps/
      immich/
        deployment.yaml
        statefulset-db.yaml
        service.yaml
        gateway.yaml
        virtualservice.yaml
        pvc.yaml
        cronjob-backup.yaml
      calibre/
      paperless/
      firefly/
      navidrome/
  overlays/
    dev/
      kustomization.yaml     # smaller resource requests, staging Let's Encrypt, dev Vault
    prod/
      kustomization.yaml     # real resource limits, prod Let's Encrypt, real Vault
```

- Third-party components (Istio — its `base`, `istiod`, and `gateway` charts
  —, cert-manager, Vault, Cilium, External Secrets Operator) are installed via
  their official Helm charts — Kustomize isn't a good fit for other people's
  complex charts, and Helm is the ecosystem standard for consuming them.
  First-party service manifests
  (Immich, Calibre, etc.) are plain Kustomize — no templating needed since,
  per auto-homelab's own Grimmory findings, the dev/prod divergence per
  service is small (replica count, resource limits, hostnames, cert issuer).
- The Go CLI shells out to neither Helm nor `kubectl` — it uses `client-go`
  to apply the rendered Kustomize output, and the
  [Helm Go SDK](https://helm.sh/docs/topics/advanced/#go-sdk) for chart
  installs, so `homeserverctl apply` is one consistent code path regardless of
  which mechanism produced the manifest.

## Secrets workflow

### Phase 1 — plain Secrets (build this first)

Ported directly from auto-homelab's `configure` command and its
strategy-registry pattern (`internal/config/registry.go`): a pluggable
`Acquirer` interface with implementations for `CONSTANT`, `GENERATED`
(`crypto/rand`), `IP`, `STRING`, and `PATH` values, driven by a per-service
schema.

1. `homeserverctl secrets generate immich` walks that service's schema,
   resolving each field via its strategy (prompting interactively for
   anything that needs human input, generating randomly for anything that
   doesn't).
2. The resolved values are written two places: (a) directly into the cluster
   as a `Secret` object, via `client-go`'s `CoreV1().Secrets(ns).Apply(...)`
   — never as a committed manifest; (b) a **git-ignored** local file (e.g.
   `secrets/<env>/immich.local.yaml`) as a human-readable backup, exactly
   mirroring `.env`'s role in auto-homelab. The `.gitignore` entry for this
   path is one of the first things committed to this repo, before any real
   secret is ever generated.
3. Service manifests reference the `Secret` only by name (`secretKeyRef`) —
   identical to how they'll reference it after the Phase 2 migration, so no
   manifest changes are needed later.
4. `homeserverctl secrets rotate immich db_password` re-runs just that one
   field's strategy and re-applies the `Secret` — the rotation story auto-
   homelab never had (its `.env` values were effectively static once set).

### Phase 2 — Vault (later)

1. `homeserverctl secrets set homeserver/cloudflare api_token=<value>` writes
   directly into Vault via its HTTP API (using Vault's official Go client,
   not a curl/CLI shell-out).
2. An `ExternalSecret` resource (part of the base manifests for each service
   that needs one) declares which Vault path/keys a service needs.
3. External Secrets Operator (running in-cluster, see ARCHITECTURE.md)
   continuously syncs those into real `Secret` objects.
4. Pods reference those `Secret` objects normally via `envFrom`/`env` — no
   Vault-awareness needed in the app manifests themselves, and no change from
   how they referenced Phase 1's `Secret`s.

For local dev, the same flow (whichever phase we're in) runs against
throwaway/test credentials — Phase 1 generates dev-only random values just
like prod; Phase 2's dev Vault is seeded with fake Cloudflare/B2/DB
credentials. `--env` is the only thing that changes.

## Backup integration

The CronJobs described in ARCHITECTURE.md are the source of truth and run
independent of the CLI. The CLI commands are a convenience layer on top:

- `backup run <service>` creates a one-off `Job` from the same `CronJob`'s pod
  template (`client-go`'s `BatchV1().Jobs().Create`, copying the CronJob's
  `spec.jobTemplate`) — equivalent to `kubectl create job --from=cronjob/...`.
- `backup restore <service> <snapshot>` orchestrates: scale the
  Deployment/StatefulSet to 0 → run a restore Job that pulls the restic
  snapshot into the PVC (and replays the DB dump if present) → scale back up.
  This directly replaces the manual, destructive `scripts-shell/restore_*.sh`
  scripts from the old project with a single idempotent, scriptable command.

## Environment detection / `--env`

- `--env=prod` targets the kubeconfig for the Debian kubeadm cluster (either
  run directly on the box, or via a kubeconfig copied to the Mac — see open
  question below).
- `--env=dev` targets the local `kind` cluster's kubeconfig.
- Every command accepts `--env`, defaulting to whatever the current
  `kubectl`-style context is, so day-to-day usage on each machine doesn't
  require typing it repeatedly.

> ❓ **Open question:** You leaned towards running the CLI directly on the
> Debian box for simplicity. Confirming: does that mean day-to-day you'll SSH
> or sit at the physical GNOME desktop to run `homeserverctl` commands against
> prod, while the Mac is used purely for `--env=dev`? Or would occasional
> remote control from the Mac (via a copied-out kubeconfig) also be useful —
> e.g. `homeserverctl apply --env=prod` from your laptop without touching the
> box at all? This affects whether we need to think about kubeconfig
> distribution/security at all right now.

## Testing strategy

- Unit tests continue the old project's interface-mocking style, now mocking
  `client-go`'s clientset interfaces (which are themselves designed to be
  fake-able — `k8s.io/client-go/kubernetes/fake`) instead of a `Runner`
  wrapping shell commands.
- Integration/end-to-end testing happens against the real `kind` dev cluster —
  `homeserverctl cluster up --env=dev && homeserverctl up --env=dev` should be
  a fast, disposable, real-cluster smoke test.

## Open questions (design-specific)

1. CLI execution location: on-box only, or also remotely from the Mac against
   prod?
2. Any preference on Helm chart pinning/versioning strategy (exact chart
   versions vendored in-repo vs. always-latest)?

# Homeserver

## Overview

`homeserver` is a Go CLI that stands up and tears down a personal Kubernetes-based
homeserver with a single command. Production runs on one Debian machine (`kubeadm`,
no high availability by design). Istio's ingress gateway handles ingress routing to
per-service hostnames/paths (e.g. `https://homeserver.example.com/immich`), with HTTPS
certificates issued dynamically via the Cloudflare API (the domain lives at
Cloudflare). Credentials need to be encrypted somewhere — exact mechanism TBD
(Bitwarden API vs. self-hosted Vault vs. lighter alternatives). The same CLI should
also be able to spin up a local version of the stack on a MacBook, using a separate
set of test credentials, for development. The user is newer to Kubernetes, so
concepts will be explained inline as they come up in later design/implementation
conversations — this doc doesn't try to teach k8s up front.

## Proposed architecture (strawman — for discussion)

- **Go CLI** (`homeserverctl` or similar) — the single entrypoint. Expected
  subcommands roughly: `up`, `down`, `status`, `apply`, `secrets sync`. It owns
  the full lifecycle: cluster bootstrap, secrets setup, networking, and
  (eventually) databases per service.
- **Cluster layer** — `kubeadm` on the Debian production box; a local equivalent
  on macOS (k3d / kind / minikube — TBD), selected via something like
  `--env=prod` vs `--env=local`.
- **Networking layer** — Istio's ingress gateway as the ingress controller /
  reverse proxy (Envoy under the hood). One `Gateway` + `VirtualService` pair
  per service, routed by hostname and/or path.
- **TLS** — cert-manager with a Cloudflare DNS-01 ACME issuer, since the domain
  and API access already live at Cloudflare.
- **DNS** — likely no extra in-cluster DNS service is needed: Cloudflare DNS
  points the domain at the home public IP, Istio's ingress gateway does
  host/path routing internally, and Kubernetes' built-in CoreDNS already
  handles service discovery inside the cluster. Called out as an open question below in case the home
  network setup needs something more (e.g. dynamic DNS).
- **Secrets** — open question. Candidates: Bitwarden API (already paid for/
  managed), self-hosted Vault, or lighter options like SOPS+age or Kubernetes
  Sealed Secrets. Needs a decision on the trade-off between reusing an existing
  managed service vs. running another self-hosted component.
- **App/service layer** — one pod per service, starting with Immich. Each
  service gets its own Deployment + Service + Ingress, applied/templated by the
  Go CLI.
- **Future** — per-service databases, backups, update/upgrade flow.

## Open questions

**Secrets management**
- Bitwarden API integration vs. self-hosted Vault vs. SOPS/age vs. Kubernetes
  Sealed Secrets? We can start with self-hosted Vault (or another service), as long as it's easy to backup to backblaze (see comments about backup below).
- How much tolerance is there for running "yet another service" (Vault) versus
  reusing something already paid for and managed (Bitwarden)? I don't mind running another service, I've got plenty of computing power available.

**DNS**
- Is the Debian box reachable via a static public IP, or does it need dynamic
  DNS? All of this is going to be local. The Debian box will be available at 192.168.1.250.
- Is `example.com` already proxied through Cloudflare? I don't know what this means.
- Is the routing model single-host + path-based (as sketched:
  `homeserver.example.com/immich`), or would per-service subdomains
  (`immich.example.com`) be preferred? The services will be called e.g. `immich.homeserver.example.com`. All `homeserver.example.com` will be reachable ONLY locally from within my home network. Other URLs, e.g. `blog.example.com`, will be reachable through the internet.

**Local dev cluster**
- Preference among k3d / kind / minikube / Docker Desktop's built-in Kubernetes
  for the Mac? No preference, whatever is easier and fits better overall with this app. As long as the Go CLI app manages the context switch (meaning the exact same CLI commands will be run here on my laptop and on the server), then I'm good.
- Is there an existing Docker or Colima setup on the Mac already? Assume Docker Desktop is installed.

**Kubernetes distro on Debian**
- Plain `kubeadm` + a CNI (Flannel / Calico / Cilium — which one?), or open to
  `k3s`/`k0s` to simplify single-node operations? I would prefer having the most industry standard setup, which at the moment involves Cilium if I'm not wrong.

**Service list & priority**
- Immich is first — what's the rest of the roadmap (media server, home
  automation, backups, etc.)? This affects how generic the "add a service"
  abstraction in the Go CLI needs to be from day one. I think Immich is the most complex service we are going to handle, as it has a database, file storage (we need to backup both), extra containers for machine learning features...

**State/config storage**
- Where should the Go CLI keep cluster state/config — a repo-committed
  YAML/Helm-values tree, or something more dynamic? YAML/Helm values are perfect as long as the same values can be used for both dev and prod deployments - or there is a system or set of conventions to clearly identify both types of values.

**GitOps vs. imperative**
- Should the CLI talk to the Kubernetes API directly (`kubectl apply`-style), or
  generate manifests for a GitOps tool like Argo CD/Flux later? Ideally we talk to Kubernetes directly.

**Debian box specifics**
- Is the OS already installed? What's the access model (SSH key, user)? Debian 13 is installed, with Gnome, I will use it like a normal desktop computer (with a monitor attached to it, logging in to Gnome).
- Should the Go CLI run *on* the box itself, or control it remotely over
  SSH/kubeconfig from the Mac? Whatever is simpler. My initial guess is that it will be simpler to run it on the box itself.

**Backups/persistence**
- What are the expectations for storage (local disk, NAS, cloud) for stateful
  services like Immich's photo library? I am going to back everything up to a BackBlaze B2 bucket I already own. At the moment I use CLI `restic` which works amazingly well.

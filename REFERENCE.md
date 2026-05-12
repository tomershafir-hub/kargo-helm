# Kargo Helm Example — Reference

A walkthrough of every moving part in this repo, what it does, and how the
pieces wire together. This is a fork of [akuity/kargo-helm](https://github.com/akuity/kargo-helm)
with the demo image swapped from a private `ghcr.io/<user>/guestbook` to
the public Docker Hub `nginx` image so it works without a custom registry.

---

## What this example demonstrates

A GitOps pipeline that promotes **two independent streams** of change
through four environments (`dev` → `staging` → `prod-us` / `prod-eu`):

1. **An image stream** — Kargo watches a container registry. When a new
   semver-tagged image appears, Kargo creates a `Freight` for it. Promoting
   that Freight bumps `image.tag` in `env/<stage>/values.yaml`, commits the
   change, and tells Argo CD to sync.

2. **A feature-flags stream** — Kargo watches a path in this Git repo
   (`base/feature-flags.yaml`). New commits touching that file produce
   Freight. Promoting that Freight copies the file into
   `env/<stage>/feature-flags.yaml`, commits, and syncs.

The two streams are independent: you can roll a new image without touching
flags, or roll a flag change without bumping the image.

---

## Repository layout

```
.
├── kargo/                  # Kargo control-plane manifests (Project, Warehouses, Stages, PromotionTask)
├── charts/guestbook/       # The Helm chart that gets deployed in every env
├── env/<stage>/            # Per-env Helm values (image tag, replicaCount, feature flags)
├── base/                   # Source of truth that the `features` Warehouse watches
├── argocd/                 # Argo CD Application manifests (one per stage)
├── docs/                   # Screenshots referenced from the README
├── personalize.sh          # Bootstrap script — rewrites `<yourgithubusername>` placeholders
├── download-cli.sh         # Helper to install the `kargo` CLI
└── README.md               # Upstream akuity quickstart
```

---

## `kargo/` — the control plane

This is the heart of the example. Four files, each a Kargo CRD.

### `kargo/project.yaml`

Defines the `kargo-helm` Project (i.e. the namespace that contains everything
else). Nothing interesting beyond grouping.

### `kargo/warehouse.yaml`

A **Warehouse** is a subscription to an external source of artifacts. New
artifacts become **Freight**.

Two warehouses here:

| Warehouse | Source | Strategy | Produces Freight when… |
|---|---|---|---|
| `guestbook` | container image `nginx` | `SemVer`, constrained to `^1.27` | a new nginx tag matching `1.27.x` appears on Docker Hub |
| `features` | git repo `main` branch, path `base/feature-flags.yaml` | `NewestFromBranch` | someone commits a change to `base/feature-flags.yaml` |

> **Note on the image repoURL.** Kargo normalizes Docker Hub library images.
> Use the canonical short form `nginx` (not `docker.io/library/nginx`) — the
> same string must appear in `vars.image` inside the PromotionTask, otherwise
> `imageFrom(vars.image, warehouse("guestbook"))` returns a zero value and the
> promotion errors with `reflect: call of reflect.Value.Field on zero Value`.

### `kargo/stages.yaml`

Defines the four **Stages**: `dev`, `staging`, `prod-us`, `prod-eu`. A Stage
is a *target* for promotion. Each Stage declares:

- **`requestedFreight`** — which warehouses it consumes, and where that
  Freight is allowed to come from. `dev` accepts Freight `direct` from each
  warehouse (i.e. fresh from the source). Downstream stages only accept
  Freight that has already been promoted to an upstream Stage:
  - `staging` ← from `dev`
  - `prod-us` ← from `staging`
  - `prod-eu` ← from `staging`

  This is what enforces "code must pass through dev before it can reach prod."

- **`promotionTemplate`** — how to actually perform the promotion. All four
  Stages reference the same shared `PromotionTask` named `promote` (see below).

- **annotations** — `kargo.akuity.io/argocd-context` tells the Kargo UI which
  Argo CD `Application` corresponds to this Stage so the UI can show sync
  status. `kargo.akuity.io/color` is purely cosmetic.

### `kargo/promotiontask.yaml`

The **PromotionTask** is the reusable recipe that every Stage runs when you
hit "Promote". One task, six steps:

```yaml
vars:
- name: image       # the image repoURL to look up in Freight
  value: nginx
- name: repoURL     # this git repo
  value: https://github.com/tomershafir-hub/kargo-helm.git
- name: branch
  value: main
```

| # | Step | What it does |
|---|---|---|
| 1 | `git-clone` | Clones `repoURL@branch` into `./out` so subsequent steps can edit files. |
| 2 | `yaml-update` (gated on guestbook Freight) | Sets `image.tag` in `./out/env/<stage>/values.yaml` to the new image tag pulled from the guestbook Freight via `imageFrom(vars.image, warehouse("guestbook")).Tag`. |
| 3 | `git-clone` (gated on features Freight) | Clones the repo a second time at the **exact commit** in the features Freight, into `./features`. This pins the feature-flags file to the promoted commit instead of `HEAD`. |
| 4 | `copy` (gated on features Freight) | Copies `./features/base/feature-flags.yaml` → `./out/env/<stage>/feature-flags.yaml`. |
| 5 | `git-commit` | Commits whatever the prior steps changed, with a message that switches text based on the Freight origin (image bump vs. flag change). |
| 6 | `git-push` | Pushes the commit to `repoURL@branch`. Argo CD picks this up on its next sync. |
| 7 | `argocd-update` | Tells Argo CD to sync the corresponding `kargo-helm-<stage>` Application immediately, instead of waiting for the next polling interval. |

The `if:` guards on steps 2/3/4 are what make a single PromotionTask handle
both streams: only the relevant step runs depending on which Warehouse the
Freight came from.

The `git-commit` message uses a YAML folded scalar (`>-`) so the inner double
quotes around `"guestbook"` and `"features"` don't collide with the outer
string delimiters.

---

## `charts/guestbook/` — the Helm chart

A plain Helm chart deployed by Argo CD in every Stage. Notable files:

- **`values.yaml`** — defaults. `image.repository: nginx` is the canonical
  short form; `image.tag` is overridden by each env folder.
- **`templates/deployment.yaml`** — a Deployment. The container's
  `envFrom.configMapRef: feature-flags` is what wires the feature-flag
  ConfigMap into the running container's env vars.
- **`templates/feature-flags.yaml`** — renders a `ConfigMap` from
  `.Values.featureFlags` (which each env supplies). This is the **target**
  of the features promotion stream.
- **`templates/service.yaml`**, **`ingress.yaml`** — standard.
- **`templates/sync-job.yaml`** — a PostSync Argo CD hook (`busybox:latest`,
  sleeps 10s). Demo only; in real life this would be a smoke test.
- **`templates/_helpers.tpl`** — shared label/selector templates.

---

## `env/<stage>/` — per-env values

Each Stage has its own values file that overrides the chart defaults:

```yaml
image:
  tag: 1.27.0       # ← updated by the PromotionTask's yaml-update step
replicaCount: 1     # 2 in prod-us and prod-eu
```

Plus a `feature-flags.yaml` (copied in by the PromotionTask).

`prod-us` and `prod-eu` have `replicaCount: 2` to demonstrate that per-env
differences live here, not in the chart or in Kargo.

---

## `base/` — source of truth for feature flags

The `base/feature-flags.yaml` is the file the `features` Warehouse watches.
Edit it on `main`, and Kargo will discover a new Freight on the next poll.

---

## `argocd/` — Argo CD Applications

One Application per Stage (`kargo-helm-dev`, `kargo-helm-staging`, etc.).
Each Application:

- Points at this Git repo
- Renders `charts/guestbook` with the values file at `env/<stage>/values.yaml`
- Targets a namespace per stage

Argo CD does the actual cluster apply. Kargo only ever talks to Argo CD via
the `argocd-update` step in the PromotionTask.

---

## How a promotion flows end-to-end

Image promotion to `dev`:

1. Docker Hub publishes `nginx:1.27.5`.
2. `guestbook` Warehouse's next poll discovers it → creates a new `Freight`.
3. User clicks "Promote to dev" in the UI for that Freight.
4. The `promote` PromotionTask runs:
   - clones the repo
   - `yaml-update` writes `image.tag: 1.27.5` into `env/dev/values.yaml`
   - commits + pushes
   - tells Argo CD app `kargo-helm-dev` to sync
5. Argo CD pulls the new commit, renders the chart with the new tag,
   applies to the cluster. New pods come up on `nginx:1.27.5`.
6. The Freight is now marked as having reached `dev` and becomes eligible
   for promotion to `staging`.

Feature-flag promotion follows the same flow, but it's the `git-clone`
(at commit) + `copy` steps that fire instead of `yaml-update`.

---

## Common pitfalls

- **`reflect.Value.Field on zero Value` on `imageFrom(...).Tag`** — the
  `vars.image` string in `promotiontask.yaml` doesn't match the repoURL
  stored inside the Freight. Check the Freight with
  `kargo get freight --project kargo-helm -o yaml` and copy the exact
  `repoURL` string into `vars.image`. Docker Hub library images
  canonicalize to `nginx`, not `docker.io/library/nginx`.

- **Old Freight after changing a Warehouse repoURL** — pre-existing Freight
  carries the *old* repoURL. After editing the Warehouse, refresh it in the
  UI so new Freight is created under the new URL; only promote the new ones.

- **Docker Hub anonymous pull rate limits** — the Warehouse poll counts
  against the limit. For light testing this is fine; under heavy use, add
  a Docker Hub `Credentials` resource in the project.

- **Outer double-quoted YAML strings containing `warehouse("...")` calls** —
  the inner `"` closes the outer string. Use a folded scalar (`>-`) or
  escape the inner quotes.

- **Argo CD Application not syncing after promotion** — verify the
  `argocd-update` step targets the right Application name (must be
  `kargo-helm-<stage>` in this repo). The annotation on the Stage
  (`kargo.akuity.io/argocd-context`) is only for the UI; the actual
  trigger lives in the PromotionTask.

---

## Useful commands

```shell
# Apply all Kargo manifests
kargo apply -f ./kargo

# List Freight in the project
kargo get freight --project kargo-helm

# Inspect a specific Freight (to see stored image repoURL)
kargo get freight <name> --project kargo-helm -o yaml

# Force a Warehouse to poll right now
kargo refresh warehouse guestbook --project kargo-helm

# Trigger a promotion from the CLI (UI is usually easier)
kargo promote --project kargo-helm --stage dev --freight <freight-name>
```

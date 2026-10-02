# GitLab Runner cache stack on K3s

A self-hosted caching layer for GitLab CI running on Kubernetes (K3s), built
to cut CI build times when running your own GitLab Runner instead of
GitLab.com's shared runners.

It solves three separate caching problems that all showed up once we moved
CI off shared runners and onto our own hardware:

1. **Base images get re-pulled from the internet on every build** (docker.io,
   gcr.io, ghcr.io) - slow, and repeatedly burns your egress bandwidth.
2. **Kaniko build layers aren't cached across builds** unless you give it a
   registry to push/pull cache layers to/from.
3. **GitLab Runner's own distributed cache** (dependencies, tool databases,
   etc.) needs an S3-compatible backend somewhere.

Net result in our setup: **~30% faster build times** after the cache warmed
up, mostly from no longer re-pulling the same base images and Trivy's
vulnerability DB on every run.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  K3s cluster                                                  │
│                                                                 │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐      │
│  │ registry-    │   │ registry-    │   │ registry-    │      │
│  │ mirror       │   │ mirror-gcr   │   │ mirror-ghcr  │      │
│  │ (docker.io)  │   │ (gcr.io)     │   │ (ghcr.io)    │      │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘      │
│         │ pull-through, proxy.ttl auto-eviction │             │
│         └───────────────────┴───────────────────┘             │
│                             │                                  │
│              containerd hosts.toml (cluster-wide)              │
│                             │                                  │
│  ┌──────────────┐          │           ┌──────────────┐       │
│  │ registry     │◄─────────┘           │ garage       │       │
│  │ (kaniko      │  kaniko --cache-repo  │ (S3 backend  │       │
│  │  build cache)│                       │  for runner  │       │
│  └──────────────┘                       │  cache)      │       │
│                                          └──────────────┘       │
│  ┌──────────────────────────┐                                  │
│  │ CronJob: weekly garbage-  │                                  │
│  │ collect on the kaniko    │                                  │
│  │ cache-repo                │                                  │
│  └──────────────────────────┘                                  │
└─────────────────────────────────────────────────────────────┘
```

Three independent pieces:

- **Pull-through mirrors** (`registry/registry-mirror*.yaml`) - plain
  `registry:2` in [proxy mode](https://distribution.github.io/distribution/recipes/mirror/)
  for docker.io, gcr.io and ghcr.io. Each one is wired into containerd
  cluster-wide via K3s's `registries.yaml` mechanism, so **every pod** in
  the cluster benefits automatically - not just CI jobs. They use `proxy.ttl`
  so images nobody has pulled in a while get evicted on their own; anything
  actively used stays cached indefinitely.
- **Kaniko cache-repo** (`registry/registry.yaml`) - a second, non-proxy
  `registry:2` that kaniko pushes/pulls build layer cache to via
  `--cache-repo`. Cleaned up weekly by a `CronJob` that runs the registry's
  own `garbage-collect --delete-untagged=true` against the *live* pod via
  `kubectl exec` - no downtime, only removes genuinely orphaned blobs.
- **Garage** (`garage/garage.yaml`) - S3-compatible backend for GitLab
  Runner's built-in [distributed cache](https://docs.gitlab.com/runner/configuration/autoscale/#distributed-runners-caching)
  (dependencies, Trivy's vulnerability DB, etc). Single-node Rust object
  store, chosen specifically for its tiny resource footprint - [Garage's own
  docs](https://garagehq.deuxfleurs.fr/) cite it running comfortably on a
  Raspberry Pi.

## Prerequisites

- A K3s cluster (this was built and tested on K3s specifically, not stock
  Kubernetes - the containerd mirror wiring step is K3s-specific)
- `kubectl` and `kustomize` (or `kubectl apply -k`)
- Enough disk for the registry PVCs (defaults below total ~90Gi, tune to
  your needs)

## Setup

### 1. Create the Garage config secret

Not included in the manifests on purpose - don't commit a real `rpc_secret`,
even for a single-node setup, to a repo:

```bash
kubectl create namespace garage
cp garage/garage.toml.template /tmp/garage.toml
sed -i "s/REPLACE_WITH_OUTPUT_OF_openssl_rand_-hex_32/$(openssl rand -hex 32)/" /tmp/garage.toml
kubectl create secret generic garage-config -n garage \
  --from-file=garage.toml=/tmp/garage.toml
rm /tmp/garage.toml
```

### 2. Apply the manifests

```bash
kubectl apply -k .
```

This creates the `registry` and `garage` namespaces and deploys everything.

### 3. Wire the pull-through mirrors into containerd

This is the part that makes the mirrors apply cluster-wide instead of just
to jobs that explicitly opt in. K3s reads `/etc/rancher/k3s/registries.yaml`
and generates the appropriate `hosts.toml` files for containerd:

```bash
DOCKER_MIRROR_IP=$(kubectl get svc registry-mirror -n registry -o jsonpath='{.spec.clusterIP}')
GCR_MIRROR_IP=$(kubectl get svc registry-mirror-gcr -n registry -o jsonpath='{.spec.clusterIP}')
GHCR_MIRROR_IP=$(kubectl get svc registry-mirror-ghcr -n registry -o jsonpath='{.spec.clusterIP}')

sudo tee /etc/rancher/k3s/registries.yaml > /dev/null <<EOF
mirrors:
  "docker.io":
    endpoint:
      - "http://${DOCKER_MIRROR_IP}:5000"
  "gcr.io":
    endpoint:
      - "http://${GCR_MIRROR_IP}:5000"
  "ghcr.io":
    endpoint:
      - "http://${GHCR_MIRROR_IP}:5000"
EOF

sudo systemctl restart k3s
```

Verify it took effect:

```bash
sudo find /var/lib/rancher/k3s/agent/etc/containerd/certs.d -type f -exec echo "--- {} ---" \; -exec cat {} \;
```

You should see a `hosts.toml` per registry pointing at the mirror's
ClusterIP with `capabilities = ["pull", "resolve"]`.

### 4. Bootstrap Garage: layout, bucket, and an access key

Garage needs a one-time cluster layout assignment before it'll serve
anything, even as a single node:

```bash
NODE_ID=$(kubectl exec -n garage deployment/garage -- /garage node id -q | cut -d'@' -f1)
kubectl exec -n garage deployment/garage -- /garage layout assign -z local -c 15G "$NODE_ID"
kubectl exec -n garage deployment/garage -- /garage layout apply --version 1

kubectl exec -n garage deployment/garage -- /garage bucket create gitlab-runner-cache
kubectl exec -n garage deployment/garage -- /garage key create gitlab-runner-key
kubectl exec -n garage deployment/garage -- /garage bucket allow \
  --read --write --owner --key gitlab-runner-key gitlab-runner-cache
```

The `key create` output prints a Key ID and Secret key once - save them,
you'll need both for the next step.

### 5. Point your GitLab Runner at the cache stack

In your `gitlab-runner` Helm values (or `config.toml`), reference the
in-cluster service DNS names - not ClusterIPs, which are reassigned on
every reinstall:

```toml
[[runners]]
  environment = [
    "CACHE_REPO=registry.registry.svc.cluster.local:5000/cache",
    "DOCKER_MIRROR=registry-mirror.registry.svc.cluster.local:5000",
  ]
  [runners.cache]
    Type = "s3"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "garage.garage.svc.cluster.local:3900"
      AccessKey = "<Key ID from step 4>"
      SecretKey = "<Secret key from step 4>"
      BucketName = "gitlab-runner-cache"
      Insecure = true
```

And in your `.gitlab-ci.yml`, point kaniko at the cache-repo:

```yaml
build:
  image:
    name: gcr.io/kaniko-project/executor:v1.24.0-debug
    entrypoint: [""]
  script:
    - /kaniko/executor
      --context "${CI_PROJECT_DIR}"
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
      --destination "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}"
      --cache=true
      --cache-repo="${CACHE_REPO}"
      --insecure
      --skip-tls-verify
```

No changes needed to pull base images through the mirrors - that part is
transparent once step 3 is done, since it's wired in at the containerd
level.

## A couple of things that bit us

- **This used to run MinIO.** `minio/minio` was pulled from Docker Hub
  entirely in September 2026 (MinIO Inc moved the community edition to
  source-only distribution), and the `quay.io/minio/minio` fallback started
  requiring auth for anonymous pulls just two weeks later. On top of that,
  the last free community image carries an unpatched auth-bypass CVE that
  MinIO has said will only ever be fixed in the paid product. Switched to
  Garage instead - smaller footprint, Apache-friendlier ecosystem, and one
  less dependency on a single vendor's goodwill.
- **Don't use the old `mirrors:` block format in containerd's `config.toml`**
  if you're checking whether the mirror config took effect - modern
  containerd (and what K3s generates) uses per-registry `hosts.toml` files
  under `certs.d/`, not the legacy inline mirrors block. Looking in the
  wrong place makes it look like the mirror isn't working when it actually is.

## License

MIT - do whatever you want with it.

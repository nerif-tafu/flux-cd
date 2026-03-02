# homelab cluster

- **infrastructure/** — cluster services (Traefik, cert-manager, MetalLB, storage, etc.)
- **apps/** — workloads
- **flux-image/** — Flux image automation (ImageRepository + ImagePolicy)
- **flux-system/** — Flux controllers and sync manifests

## Traefik layout

There are three Traefik instances, each with its own LoadBalancer IP:

| Instance    | IP            | Role |
|------------|---------------|------|
| **internal** | 192.168.3.62 | LAN-only HTTP(S). Serves Ingress/IngressRoutes that have label `traefik.tafu.casa/instance=internal`. |
| **public**   | 192.168.3.61 | Internet-facing HTTP(S). Serves Ingress/IngressRoutes with label `traefik.tafu.casa/instance=public`. |
| **gateway**  | 192.168.3.63 | Raw TCP/UDP only (WireGuard, game servers). Serves IngressRouteTCP/UDP with label `traefik.tafu.casa/instance=gateway`. No HTTP. |

**What the label is:** `traefik.tafu.casa/instance` is a **Kubernetes label** (the `traefik.tafu.casa` part is just a label key prefix, like a domain, to avoid clashes). It is **not** a hostname. You put this label on Ingress or IngressRoute resources so the right Traefik instance picks them up (internal, public, or gateway).

**Dashboard hostnames (internal only, on .62):**  
- **traefik-internal.cl.tafu.casa** — dashboard for the internal Traefik instance  
- **traefik-public.cl.tafu.casa** — dashboard for the public Traefik instance (still served via .62, not exposed on the internet)

Internal and public use **Ingress** (and optional **IngressRoute**) with TLS. The gateway uses **IngressRouteTCP** / **IngressRouteUDP** only.

---

# Deploying a new service

Below are the steps for each service type, including **image auto-updates** (Flux image automation) where applicable.

---

## 1. Internal web app (on cluster)

**Example:** wg-easy frontend (vpn.cl.tafu.casa)

**Meaning:** HTTP(S) app running in the cluster, reachable only on LAN (internal Traefik at .62).

### Steps

1. **App directory** under `apps/<name>/`:
   - **namespace.yaml** — namespace matching the app name.
   - **deployment.yaml** — image with optional policy ref for auto-update (see below).
   - **service.yaml** — ClusterIP, selector matching the deployment.
   - **certificate.yaml** — cert-manager Certificate for the hostname (e.g. `vpn.cl.tafu.casa`), issuer `letsencrypt-cloudflare`, secret name used in Ingress.
   - **ingress.yaml** — `ingressClassName: traefik-internal`, label `traefik.tafu.casa/instance: internal`, host, TLS with the certificate secret.
   - **kustomization.yaml** — list the above (add pvc.yaml etc. if needed).

2. **Register the app** in `apps/kustomization.yaml`: add `- ./<name>`.

3. **Optional – image auto-update** (e.g. semver):
   - In **flux-image/** add:
     - **`<name>-imagerepository.yaml`** — ImageRepository (image, interval, optional `secretRef` for private registries).
     - **`<name>-imagepolicy.yaml`** — ImagePolicy (e.g. semver `>=1.0.0`).
   - In deployment, set image to e.g. `ghcr.io/org/repo:1.0.0 # {"$imagepolicy": "flux-system:<name>"}`.
   - Add both files to **flux-image/kustomization.yaml**.

Internal Traefik watches Ingress with `traefik.tafu.casa/instance=internal` and serves them on 192.168.3.62.

---

## 2. Internal web app (outside cluster)

**Example:** pihole (pihole.cl.tafu.casa → 192.168.3.2:80)

**Meaning:** HTTP(S) app running on a fixed IP/host on the LAN; Traefik proxies to it.

### Steps

1. **Service + Endpoints** in the **traefik** namespace so Traefik can route to the external IP:
   - **Service** — ClusterIP, same name in `traefik` namespace, port(s) matching the backend.
   - **Endpoints** — same name/namespace, `subsets[].addresses[].ip` = external IP, `ports[].port` = backend port.

2. **IngressRoute** (or IngressRoute in a single file with Service/Endpoints) under **infrastructure/traefik/routes/lan/**:
   - Label: `traefik.tafu.casa/instance: internal`.
   - `entryPoints: [websecure]`, route `Host(`<name>.cl.tafu.casa`)` → service name in `traefik` namespace, port.
   - TLS: e.g. `secretName: wildcard-cl-tafu-casa-tls` (or a dedicated cert).

3. **Register** the new YAML in **infrastructure/traefik/routes/lan/kustomization.yaml** (resources list).

No app directory under `apps/` and no image automation; the backend is not in the cluster.

---

## 3. Internal game server (on cluster)

**Example:** none in repo (same pattern as external game, but do not port-forward the gateway from the internet).

**Meaning:** TCP/UDP game server in the cluster, reachable only on LAN via the gateway IP (.63).

### Steps

1. **App directory** under `apps/<name>/`:
   - **namespace.yaml**, **deployment.yaml**, **service.yaml** (ClusterIP, game port(s)), **pvc.yaml** if needed.
   - **kustomization.yaml** — list the above.

2. **Gateway entrypoint** in **infrastructure/traefik/helmrelease-gateway.yaml**:
   - Under `values.ports` add a new entry (e.g. `myserver`) with `port`, `expose: true`, `exposedPort`, and `protocol: TCP` or `UDP`.

3. **IngressRouteTCP or IngressRouteUDP** under **infrastructure/traefik/routes/gateway/**:
   - Label: `traefik.tafu.casa/instance: gateway`.
   - `entryPoints: [<entrypoint-name>]` (same as in the gateway).
   - Route with `match: HostSNI(\`*\`)` and service pointing to `<name>.<namespace>:<port>`.

4. **Register** the route file in **infrastructure/traefik/routes/gateway/kustomization.yaml**.

5. **Optional – image auto-update** — add ImageRepository + ImagePolicy in **flux-image/** and reference in the deployment (as in section 1).

Do **not** port-forward the gateway’s IP/port from the internet so the server stays internal-only.

---

## 4. External web app (on cluster)

**Example:** fitv2 (fit.tafu.casa)

**Meaning:** HTTP(S) app in the cluster, reachable from the internet via public Traefik (.61).

### Steps

1. **App directory** under `apps/<name>/`:
   - **namespace.yaml**, **deployment.yaml**, **service.yaml** (ClusterIP).
   - **ingress.yaml** — `ingressClassName: traefik-public`, label `traefik.tafu.casa/instance: public`, host (e.g. fit.tafu.casa), TLS (cert-manager or existing secret). Add annotation `traefik.ingress.kubernetes.io/router.middlewares: traefik-fail2ban-public@kubernetescrd` so the public fail2ban middleware applies.
   - **kustomization.yaml** — list the above. Add **ghcr-pull-secret.yaml** (and reference in deployment) if using a private registry.

2. **Certificate** — either:
   - In the app: **certificate.yaml** with ClusterIssuer (e.g. letsencrypt-cloudflare) and `ingress` annotation, or
   - Rely on cert-manager + Ingress annotation `cert-manager.io/cluster-issuer: letsencrypt-cloudflare` and `tls.secretName` in Ingress.

3. **Register** the app in **apps/kustomization.yaml**: `- ./<name>`.

4. **Optional – image auto-update** — add ImageRepository (with `secretRef` if private) and ImagePolicy in **flux-image/**, reference in deployment with `# {"$imagepolicy": "flux-system:<name>"}`.

Public Traefik watches Ingress with `traefik.tafu.casa/instance=public` and serves them on 192.168.3.61 (port-forward from internet as needed).

---

## 5. External web app (outside cluster)

**Example:** none in repo.

**Meaning:** HTTP(S) app running on an external host; public Traefik proxies to it.

### Steps

1. **Service + Endpoints** in the **traefik** namespace:
   - **Service** — ClusterIP, name e.g. `public-<name>`, port(s) matching the backend.
   - **Endpoints** — same name/namespace, `subsets[].addresses[].ip` = external IP/host, `ports[].port` = backend port.

2. **IngressRoute** under **infrastructure/traefik/routes/public/** (create a new file, e.g. `<name>.yaml`):
   - Label: `traefik.tafu.casa/instance: public`.
   - `entryPoints: [websecure]`, route `Host(`<public-hostname>`)` → the Service in `traefik` namespace, port.
   - TLS: cert-manager or existing secret for the public hostname.

3. **Register** the new file in **infrastructure/traefik/routes/public/kustomization.yaml** (add to resources).

4. Ensure a **Certificate** or ClusterIssuer exists for the public hostname (e.g. in infrastructure or a shared place).

No app under `apps/` and no image automation.

---

## 6. External game server (on cluster)

**Example:** minecraft (192.168.3.63:25565)

**Meaning:** TCP (and optionally UDP) game server in the cluster, exposed on the gateway IP for internet/LAN access.

### Steps

1. **App directory** under `apps/<name>/`:
   - **namespace.yaml**, **deployment.yaml**, **service.yaml** (ClusterIP, game port), **pvc.yaml** if needed.
   - **kustomization.yaml** — list the above.

2. **Gateway entrypoint** in **infrastructure/traefik/helmrelease-gateway.yaml**:
   - Under `values.ports` add an entry (e.g. `minecraft`) with `port`, `expose: true`, `exposedPort`, `protocol: TCP` (and a second entry for UDP if needed).

3. **IngressRouteTCP** (or **IngressRouteUDP**) under **infrastructure/traefik/routes/gateway/**:
   - Label: `traefik.tafu.casa/instance: gateway`.
   - `entryPoints: [<entrypoint-name>]`.
   - Route `match: HostSNI(\`*\`)`, service `<name>.<namespace>:<port>`.

4. **Register** the route in **infrastructure/traefik/routes/gateway/kustomization.yaml**.

5. **Register** the app in **apps/kustomization.yaml**: `- ./<name>`.

6. **Optional – image auto-update** — add ImageRepository + ImagePolicy in **flux-image/** and reference in the deployment (e.g. for Docker Hub or GHCR).

Port-forward the gateway IP and port from the internet if the server should be reachable from outside LAN.

---

## Image auto-update (Flux) summary

- **flux-image/imagerepository.yaml** — `image:`, `interval:`, optional `secretRef` for private registries.
- **flux-image/imagepolicy.yaml** — `imageRepositoryRef.name`, and either:
  - **semver** — `policy.semver.range: ">=1.0.0"`, or
  - **alphabetical** — `policy.alphabetical.order: asc`.
- **Deployment** — set image to a concrete tag and append `# {"$imagepolicy": "flux-system:<name>"}` so Flux rewrites it from the policy.
- Add both ImageRepository and ImagePolicy to **flux-image/kustomization.yaml**.

ImageUpdateAutomation (if configured) can open PRs when the policy selects a new tag; otherwise run `flux reconcile image policy <name> -n flux-system` and redeploy to pick the new image.

# homelab cluster

- **infrastructure/** — cluster services (Traefik, cert-manager, MetalLB, storage, etc.)
- **apps/** — workloads
- **flux-image/** — Flux image automation (ImageRepository + ImagePolicy)
- **flux-system/** — Flux controllers and sync manifests

## Storage layout

Two backends. Pick by access pattern, not by size.

| Class | Backing | Use for | Backed up |
|---|---|---|---|
| **longhorn** (default) | 3 replicas on the nodes' NVMe | Databases, config, state, anything small, hot, random-access or irreplaceable | Yes — nightly to the NAS |
| **nfs-bulk** | Synology NAS, `/volume1/k8s-bulk`, 4TB quota | Large sequential files: media libraries, game assets, map tiles | **No — treat as expendable** |
| local-path | Node-local, k3s built-in | Nothing. Present but unused. | No |

**Never put a database on nfs-bulk.** Embedded databases assume local-filesystem
semantics NFS does not reliably provide: SQLite relies on POSIX advisory locking
that NFS implements loosely, and BoltDB mmaps its whole file, which over NFS has
no coherency guarantee. The NAS is also RAID 6 on 5900rpm disks, paying a
read-modify-write penalty on every small write. filebrowser (BoltDB) and
convertx (SQLite, in WAL mode) both straddle the two classes for exactly this
reason: database on Longhorn, bulk directories on the NAS.

**Sizes on nfs-bulk are advisory.** NFS enforces nothing per volume — a pod
sees the whole share's free space in `df`. The only real limit is the btrfs
quota on the share, set on the NAS, not in this repo.

**PVCs cannot be shrunk.** Growing is online and easy; shrinking means delete
and recreate. On nfs-bulk that is survivable because the class sets
`reclaimPolicy: Retain`, `onDelete: retain`, and a `subDir` templated from
namespace and claim name, so a recreated claim finds its old directory. On
longhorn it means a real migration.

### Backups

| What | How | Where |
|---|---|---|
| Longhorn volume data | RecurringJob `daily-backup`, 02:00, retain 14 | `k8s-backup/longhorn` |
| k3s datastore, cluster CA, all Secrets | CronJob `cluster-backup`, 01:00, retain 14 | `k8s-backup/cluster` |
| nfs-bulk data | Nothing, deliberately | — |

This cluster's datastore is **SQLite/kine, not etcd** — `k3s etcd-snapshot`
does nothing here. The bundle is encrypted to a public certificate whose
private key is held offline; without that key the backups are unrecoverable,
and it is not in this repo or the cluster by design.

### Cluster state not in git

Like the Proxmox node labels, some storage config lives only on the machines:

- **NFS export rules on the NAS.** Keyed on node addresses. The black-* nodes
  are multi-homed and reach the NAS from their Proxmox `vmbr0` bridge address
  (.23/.25/.27), *not* their node IP — a new node needs both added.
- **The btrfs share quotas** (4TB bulk, 5.2TB backup).
- **The default StorageClass annotation.** k3s ships `local-path` as default and
  Longhorn also claims it, which leaves two defaults; Kubernetes resolves that
  by silently picking the newest rather than erroring. `local-path` was patched
  to `is-default-class: false` so `longhorn` is unambiguous.

  This **survives a k3s restart**, despite what you might expect from a
  packaged addon. k3s tracks each bundled manifest with an
  `addons.k3s.cattle.io` object carrying a checksum, and re-applies only when
  that checksum changes — a restart re-extracts an identical
  `local-storage.yaml`, the checksum matches, and it is skipped. What *does*
  revert it is a k3s **upgrade** that ships a changed manifest, so re-check
  after one:

  ```
  kubectl get sc   # longhorn should be the only (default)
  ```

  The permanent fix is `--disable local-storage` on the server, deliberately
  not done: it deletes a working provisioner to settle what is currently a
  cosmetic ambiguity, needs a k3s restart (an API outage — silver-01 is the
  only server node), and nothing in this repo relies on the default anyway
  because every PVC names its class.

## Disaster recovery

**Read this before you need it.** The whole plan rests on one thing that is not
on any machine here: the backup recovery **private key**, held offline. Without
it every `cluster-*.tar.gz.enc` is permanently unreadable and this runbook stops
at step 3. Check occasionally that you can still open it.

### Facts you will need

| | |
|---|---|
| k3s version | `v1.34.3+k3s3` |
| Server node | silver-01, 192.168.3.20, `enp2s0` |
| Agents | black-01 .22, black-02 .24, black-03 .26 |
| Flannel | `host-gw` backend |
| Datastore | **SQLite/kine — NOT etcd** |
| NAS | 192.168.3.10, shares `k8s-backup` and `k8s-bulk` |
| Repo | `https://github.com/nerif-tafu/flux-cd`, path `./clusters/homelab` |
| MetalLB pool | 192.168.3.50–192.168.3.100 |

---

### Scenario A — cluster lost, NAS intact

The case this design is built for: four bare nodes, the NAS untouched, the repo
on GitHub, the recovery key in your password manager. Everything needed is
outside the cluster.

Two things make this much less work than it looks:

- **Bulk data does not get restored — it reattaches.** `nfs-bulk` volumes live
  on the NAS and were never inside the cluster. The class uses
  `subDir: ${pvc.metadata.namespace}-${pvc.metadata.name}`, so a rebuilt cluster
  applying the same PVCs from git resolves to the same directories
  (`webplayer-webplayer-media`, `rust-api-rust-api-bulk`, …) and picks the data
  straight back up. Nothing to copy. This is not theoretical — it is the same
  mechanism used to resize webplayer's claim in place.
- **Only Longhorn volumes need restoring**, and they come from
  `k8s-backup/longhorn`.

**Expected data loss:** back to the last 02:00 Longhorn run and the last 01:00
control-plane run. Bulk data: none.

#### Step 0 — check the NAS first

Do this before touching the nodes, because the export ACL is keyed on node
addresses and a rebuild can change them.

```bash
ssh nerif@192.168.3.10 'sudo exportfs -v'
```

Every rebuilt node needs its address in the rules for **both** shares. If you
reuse 192.168.3.20/.22/.24/.26 the existing rules still apply. Note the black-*
nodes reach the NAS from their Proxmox `vmbr0` address (.23/.25/.27), *not*
their node IP — see "Cluster state not in git" below. Missing entries show up
later as a flat `mount.nfs: access denied by server`.

#### Step 1 — install the server on silver-01

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.34.3+k3s3" sh -s - server \
  --node-ip=192.168.3.20 \
  --flannel-iface=enp2s0 \
  --flannel-backend=host-gw \
  --disable traefik
```

`--disable traefik` matters: Traefik is managed by this repo as three separate
instances, and k3s's bundled one would fight them for :80/:443.

Then stop it, so the datastore can be swapped underneath:

```bash
sudo systemctl stop k3s
```

#### Step 2 — decrypt the newest bundle

On any machine with the offline key:

```bash
sudo mount -t nfs -o nfsvers=4.1 192.168.3.10:/volume1/k8s-backup /mnt/restore
ls /mnt/restore/cluster/

openssl smime -decrypt -binary -inform DER \
  -in /mnt/restore/cluster/cluster-YYYY-MM-DD.tar.gz.enc \
  -inkey cluster-backup-recovery.key | tar xzf -
```

Verify before trusting it — a bundle that extracts is not necessarily a bundle
that restores:

```bash
python3 -c "import sqlite3;print(sqlite3.connect('k3s/state.db').execute('pragma integrity_check').fetchone())"
python3 -c "import sqlite3;print(sqlite3.connect('k3s/state.db').execute('select count(*) from kine').fetchone())"
```

Expect `('ok',)` and a five-figure row count (~10,200 at time of writing).

#### Step 3 — restore the datastore and cluster identity

With k3s **stopped**:

```bash
sudo cp k3s/state.db          /var/lib/rancher/k3s/server/db/state.db
sudo rm -f /var/lib/rancher/k3s/server/db/state.db-wal \
           /var/lib/rancher/k3s/server/db/state.db-shm
sudo cp k3s/token k3s/node-token /var/lib/rancher/k3s/server/
sudo cp -a k3s/tls  /var/lib/rancher/k3s/server/
sudo cp -a k3s/cred /var/lib/rancher/k3s/server/
sudo chown -R root:root /var/lib/rancher/k3s/server
sudo systemctl start k3s
```

**Do not skip `tls/` and `token`.** Restoring `state.db` alone gives you a
cluster whose CA no longer matches the one every ServiceAccount token and every
agent was issued against — nodes will refuse to join and half the pods will fail
to authenticate. Deleting the stale `-wal`/`-shm` matters too: they belong to
the *old* database file and SQLite will try to replay them over the restored one.

Check it came up:

```bash
sudo k3s kubectl get nodes
```

silver-01 should be `Ready`. The three agents will show `NotReady` — they do not
exist yet.

#### Step 4 — rejoin the three agents

Get the token (it is the one you just restored):

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

On each black node, with its own IP:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.34.3+k3s3" \
  K3S_URL=https://192.168.3.20:6443 K3S_TOKEN="<node-token>" sh -s - agent \
  --node-ip=192.168.3.22 \
  --flannel-iface=<lan-nic>
```

On the black-* nodes `--flannel-iface` is not optional: they have **two NICs on
192.168.3.0/24** (`enp87s0` and `enp88s0`, the latter bridged to `vmbr0` by the
Proxmox profiles), and flannel picking the wrong one gives you a cluster whose
pod network silently fails between nodes. Pick the interface holding the node IP.

Because the CA and node names came back with the datastore, the nodes rejoin as
themselves. Wait for four `Ready`.

#### Step 5 — restore Longhorn volumes BEFORE Flux

Order matters here more than anywhere else in this runbook.

Longhorn is normally installed by Flux, but if you let Flux run first it also
creates every PVC — empty — and you will be fighting it while restoring. So
install Longhorn by hand, restore, and let Flux adopt the result.

```bash
helm repo add longhorn https://charts.longhorn.io
helm install longhorn longhorn/longhorn -n longhorn-system --create-namespace \
  --version 1.6.2 \
  --set defaultSettings.backupTarget=nfs://192.168.3.10:/volume1/k8s-backup/longhorn
```

Longhorn scans the target and every `BackupVolume` reappears within a minute:

```bash
sudo k3s kubectl -n longhorn-system get backupvolumes
```

For each one, restore it **to its original volume name** (the `pvc-<uuid>`), then
use Longhorn's *Create PV/PVC* to bind it back to the namespace and claim name it
had. Both are recorded in the bundle you already decrypted:

```bash
grep -A3 'claimRef' api/cluster_persistentvolumes.yaml | grep -E 'name|namespace'
```

Get the claim names right and the PVC manifests in git match what already
exists, so Flux adopts them instead of creating empty replacements.

**Skip anything on `nfs-bulk`** — those PVCs are not Longhorn volumes and have no
backups. They reattach on their own in the next step.

#### Step 6 — recreate the two bootstrap secrets

Flux cannot decrypt a single Secret in this repo without the age key, and cannot
clone without credentials. Both are in the bundle.

The age key, from `api/ns_secrets.yaml` (the `sops-age` Secret, key
`age.agekey`, base64 in `data`):

```bash
sudo k3s kubectl create ns flux-system
sudo k3s kubectl -n flux-system create secret generic sops-age \
  --from-file=age.agekey=./age.agekey
```

Confirm you have the right key before continuing — its public half must match
the recipient in `.sops.yaml`:

```
age1zxpvrp778wzdh73fq5k2yln24aechr8n69n66smtjvepd56q9g9sa0xw5g
```

And the git credentials (keys `username` and `password`, same file):

```bash
sudo k3s kubectl -n flux-system create secret generic git-credentials \
  --from-literal=username=<username> --from-literal=password=<token>
```

#### Step 7 — reinstall Flux and point it at the repo

```bash
flux install
sudo k3s kubectl apply -f clusters/homelab/flux-system/gotk-sync.yaml
```

From here it rebuilds itself: cert-manager, MetalLB, the three Traefik
instances, csi-driver-nfs, monitoring, logging, and every app. Watch it:

```bash
flux get kustomizations --watch
```

The `nfs-bulk` PVCs bind as csi-driver-nfs provisions them, each resolving to
its existing directory on the NAS. Confirm the data actually came back rather
than assuming it:

```bash
sudo k3s kubectl -n webplayer exec deploy/webplayer -- ls /data | head
```

#### Step 8 — replace the state that was never in git

```bash
sudo k3s kubectl label node black-01 homelab.tafu.casa/proxmox-capable=true
sudo k3s kubectl label node black-02 homelab.tafu.casa/proxmox-capable=true
sudo k3s kubectl label node black-03 homelab.tafu.casa/proxmox-capable=true

sudo k3s kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

#### Step 9 — verify, then prove backups run again

```bash
sudo k3s kubectl get nodes                          # 4 Ready
flux get kustomizations                             # Ready=True
sudo k3s kubectl get pvc -A | grep -c Bound         # every claim bound
sudo k3s kubectl -n longhorn-system get volumes.longhorn.io   # healthy
curl -sk -o /dev/null -w '%{http_code}\n' https://webplayer.cl.tafu.casa/ \
  --resolve webplayer.cl.tafu.casa:443:192.168.3.62
```

Then force one backup rather than waiting for 01:00 — a cluster you cannot back
up is not recovered:

```bash
sudo k3s kubectl -n cluster-backup create job --from=cronjob/cluster-backup restore-check
sudo k3s kubectl -n cluster-backup logs job/restore-check -c package
```

---

### Scenario B — NAS lost, cluster intact

The cluster keeps running: Longhorn volumes are on the nodes' NVMe and are not
touched. What you lose is every backup and everything on `nfs-bulk`.

Apps with NFS mounts will hang on I/O (the class mounts `hard`, deliberately —
`soft` would hand applications truncated files instead). Expect webplayer,
rust-api, filebrowser, convertx and sevtech's dynmap to block until the NAS
returns or their claims are removed.

To rebuild: recreate both shares, quotas and export rules on the replacement NAS
(see "Cluster state not in git"), and the nightly jobs repopulate the backups
from scratch. **The bulk data does not come back** — it was never copied
anywhere. That was the accepted trade in exchange for the capacity.

### Scenario C — both lost

Git still has every manifest, so the cluster rebuilds; all persistent data is
gone. Follow Scenario A steps 1, 4, 6, 7 and 8, skipping anything involving the
NAS, and accept empty volumes. The SOPS age key is the deciding factor: without
it the repo is undecryptable and there is no cluster to recover it from. This is
why it also lives in a password manager.

### Partial failures

- **One volume corrupted** — restore just that one from the Longhorn UI. No
  rebuild.
- **A node dies** — Longhorn rebuilds its replicas onto the survivors
  automatically. Re-add the `proxmox-capable` label if it was one of those, and
  add the replacement's addresses to the NAS export rules.
- **silver-01 dies** — it is the only server node, so this is a full Scenario A
  rebuild even though the three workers are healthy and their data is intact.

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

## Proxmox VE (internal, failover-capable)

**URL:** https://proxmox.cl.tafu.casa (internal Traefik at .62)

Proxmox runs as a privileged `hostNetwork` pod (dockurr/proxmox) with Longhorn PVCs for VM disk and cluster config. It is a **singleton** (one replica, RWO volumes): the goal is **failover to another node** when the preferred host is down, not zero-downtime HA.

### Node requirements

Label every node that may host Proxmox (requires `/dev/kvm` and `enp88s0` on the LAN):

```bash
kubectl label node black-01 homelab.tafu.casa/proxmox-capable=true
kubectl label node black-02 homelab.tafu.casa/proxmox-capable=true
kubectl label node black-03 homelab.tafu.casa/proxmox-capable=true
```

Labels are cluster state (not in git). Remove a label to take a node out of the pool:

```bash
kubectl label node black-03 homelab.tafu.casa/proxmox-capable-
```

### Scheduling

- **nodeSelector:** `homelab.tafu.casa/proxmox-capable=true`
- **Preferred order:** `black-02` → `black-01` → `black-03`
- When the preferred node is unavailable, Kubernetes reschedules elsewhere after Longhorn detaches the volume (minutes of downtime).

### Networking

Each candidate node has a profile in `apps/proxmox/network-profiles.yaml` that bridges `enp88s0` → `vmbr0` with a **reserved** LAN IP:

| Node     | Node IP       | vmbr0 (VM bridge) |
|----------|---------------|-------------------|
| black-01 | 192.168.3.22  | 192.168.3.23      |
| black-02 | 192.168.3.24  | 192.168.3.25      |
| black-03 | 192.168.3.26  | 192.168.3.27      |

An init container picks the profile from `spec.nodeName`. Adding a new failover node requires a new profile key and label on that node.

On failover, the startup script migrates VM configs under `/etc/pve/nodes/<old>/` to the new k8s hostname. VMs on DHCP usually recover; static LAN IPs may need updating if they targeted the old bridge address.

### Manifests (`apps/proxmox/`)

- **deployment.yaml** — `hostNetwork`, nodeSelector + affinity, init container for network profile
- **network-profiles.yaml** — per-node `/etc/network/interfaces`
- **startup-configmap.yaml** — pmxcfs `/etc/hosts` fix and node config migration
- **pvc.yaml** / **pvc-config.yaml** — Longhorn data volumes
- **ingress.yaml** — internal Traefik only (`traefik.tafu.casa/instance: internal`)
- **servers-transport.yaml** — Traefik skips verify on Proxmox :8006 self-signed cert

---

## Image auto-update (Flux) summary

- **flux-image/imagerepository.yaml** — `image:`, `interval:`, optional `secretRef` for private registries.
- **flux-image/imagepolicy.yaml** — `imageRepositoryRef.name`, and either:
  - **semver** — `policy.semver.range: ">=1.0.0"`, or
  - **alphabetical** — `policy.alphabetical.order: asc`.
- **Deployment** — set image to a concrete tag and append `# {"$imagepolicy": "flux-system:<name>"}` so Flux rewrites it from the policy.
- Add both ImageRepository and ImagePolicy to **flux-image/kustomization.yaml**.

ImageUpdateAutomation (if configured) can open PRs when the policy selects a new tag; otherwise run `flux reconcile image policy <name> -n flux-system` and redeploy to pick the new image.

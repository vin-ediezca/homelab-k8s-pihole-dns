# Pi-hole Homelab DNS — 2026

Network-wide ad blocking and DNS filtering for the entire home LAN,
running on Kubernetes with a stable LAN VIP via MetalLB — plus remote
admin access over Tailscale.

> **📌 Adapt the IPs to your network.** Every address in this repo is
> specific to *this* homelab and used as a working example. If you're
> following along, substitute your own values everywhere they appear
> (README, `metallb-config.yaml`, `pihole-values.yaml`):
>
> | Value in this repo | What it is | Use instead |
> | ------------------ | ---------- | ----------- |
> | `10.0.0.0/24` | LAN subnet | whatever your router uses (`192.168.1.0/24` is common) |
> | `10.0.0.1` | gateway/router | your router's IP |
> | `10.0.0.53` | Pi-hole VIP | any free IP in your subnet, **outside the DHCP range** |
> | `10.0.0.50–10.0.0.60` | MetalLB pool | a small reserved block, **outside the DHCP range** |
> | `10.0.0.100–10.0.0.200` | router DHCP range | check your router |
> | `worker-1` | example name of the node Pi-hole is pinned to | one of your nodes from `kubectl get nodes` — goes in `pihole-values.local.yaml` (gitignored), **never** in the committed values; see [Upgrading safely](#upgrading-safely) |
> | `Asia/Manila` | timezone (`TZ` in `pihole-values.yaml`) | your [TZ database name](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) |
> | `pihole-homelab` | Tailscale machine hostname | any name you like (edit `pihole-ts-ingress.yaml`) |
> | Omada gateway | this homelab's router (TP-Link Omada) | any router — the only requirement is a configurable DHCP DNS server |
>
> One universal rule: the MetalLB pool must not overlap your router's
> DHCP range **or** your cluster's pod/service CIDRs.

---

## Architecture

```
Home LAN 10.0.0.0/24
┌───────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Omada Gateway (10.0.0.1)              Devices (phones/PCs/IoT)   │
│  DHCP DNS → 10.0.0.53   ─────────────>  DNS queries :53           │
│                                              │                    │
│                                              ▼                    │
│  ┌──────────────────── Kubernetes Cluster ─────────────────────┐  │
│  │                                                             │  │
│  │  MetalLB (L2/ARP) announces VIP 10.0.0.53                   │  │
│  │        │                                                    │  │
│  │        ▼                                                    │  │
│  │  Service pihole-dns   LoadBalancer  TCP+UDP 53  (Local)     │  │
│  │  Service pihole-web   LoadBalancer  80/443  (shared VIP)    │  │
│  │        │                                                    │  │
│  │        ▼                                                    │  │
│  │  Deployment pihole (1 replica, Recreate, pinned to a node)  │  │
│  │    └── PVC pihole (local-path)                              │  │
│  │          upstream DNS ► 1.1.1.1 / 9.9.9.9                   │  │
│  │                                                             │  │
│  │  Ingress pihole-ingress (tailscale) ── admin UI over        │  │
│  │        the tailnet: pihole-homelab.<tailnet>.ts.net         │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

**How it works:**

- MetalLB in **Layer 2 (ARP) mode** answers ARP for `10.0.0.53` from the
  node running the Pi-hole pod, giving DNS a fixed LAN IP that survives
  pod restarts. L2 mode is used deliberately — MetalLB's BGP mode
  conflicts with Calico's BGP machinery on this cluster.
- The DNS Service uses `mixedService: true` so **TCP and UDP 53 share one
  Service**, and both the DNS and web Services share the same VIP via the
  `metallb.universe.tf/allow-shared-ip` annotation — the admin UI is
  reachable on the LAN at `http://10.0.0.53/admin` with no ingress
  controller involved.
- `externalTrafficPolicy: Local` preserves **real client source IPs**, so
  the dashboard shows which device made each query (and avoids FTL's
  per-client rate limit throttling the whole network as one client).
- One replica with `strategyType: Recreate` — FTL owns a SQLite database
  on the PVC, and two instances writing it concurrently would corrupt it.
  The pod always runs on the same node: `local-path` volumes are
  node-local, so the PV carries a hard node affinity.
- The Tailscale Kubernetes Operator provisions a proxy for
  `pihole-ingress`, exposing the admin UI at
  `https://pihole-homelab.<tailnet>.ts.net/admin` from anywhere on the
  tailnet.
- The Omada gateway's DHCP hands out `10.0.0.53` as the **only** DNS
  server — no public resolver as "backup", because clients treat DNS
  servers as a set and would silently bypass blocking.

---

## Access

| Endpoint | Address | Used by |
| -------- | ------- | ------- |
| DNS (TCP+UDP 53) | `10.0.0.53` | every DHCP client on the LAN |
| Admin UI (LAN) | `http://10.0.0.53/admin` | you (browser, on the LAN) |
| Admin UI (tailnet) | `https://pihole-homelab.<tailnet>.ts.net/admin` | you (browser, anywhere) |

---

## File Layout

```
pi-hole/
├── README.md
├── .gitignore                ← keeps *.local.yaml out of git
├── metallb-config.yaml       ← IPAddressPool + L2Advertisement (kubectl apply)
├── pihole-values.yaml        ← Helm values for mojo2600/pihole (generic, committed)
├── pihole-values.local.yaml  ← YOUR cluster's overrides, e.g. node name (gitignored; you create it)
└── pihole-ts-ingress.yaml    ← Tailscale Ingress for the admin UI (kubectl apply)
```

---

## Versions

| Component | Helm Chart | App Version |
| --------- | ---------- | ----------- |
| Pi-hole (mojo2600/pihole) | 2.37.0 | Pi-hole v6 (`pihole/pihole`, chart appVersion) |
| MetalLB | 0.15.x | — |
| Tailscale Operator | 1.98.4 | v1.98.4 (shared with the observability stack) |

> Check the live versions any time with `helm list -A`.

---

## MetalLB IP Address Plan

MetalLB pool: **`10.0.0.50 – 10.0.0.60`** — outside the Omada DHCP range
(`10.0.0.100 – 10.0.0.200`).

| IP | Assignment |
| -- | ---------- |
| `10.0.0.50 – 10.0.0.52` | free (previously auto-grabbed by Wazuh services — released) |
| `10.0.0.53` | **Pi-hole** (DNS + web, shared VIP) — never change after rollout |
| `10.0.0.54 – 10.0.0.60` | free |

---

## Prerequisites

- Kubernetes **1.26+** (mixed-protocol LoadBalancer Services; this cluster runs 1.33)
- Helm **3.14+**
- `kubectl` configured for your cluster
- Default StorageClass available for PVCs
  (e.g. [rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner))
- **Tailscale Kubernetes Operator** already installed — see
  [homelab-k8s-observability-stack](https://github.com/vin-ediezca/homelab-k8s-observability-stack)
  Step 3 for the ACL, OAuth client, and operator install
- A reserved LAN IP block for MetalLB, **outside the router's DHCP range**
- ⚠️ **Check your pod network CIDR before picking MetalLB addresses** —
  they must not overlap. On this cluster, Calico's pod network lives
  inside `192.168.0.0/16` (the Calico default), so MetalLB addresses
  must come from `10.0.0.0/24` and never 192.168.x.x. If your LAN *is*
  192.168.x.x, make sure your CNI's pod CIDR is something else

---

## Step 1 — Add Helm Repos

```
helm repo add metallb  https://metallb.github.io/metallb
helm repo add mojo2600 https://mojo2600.github.io/pihole-kubernetes/
helm repo update
```

---

## Step 2 — Install MetalLB

```
kubectl create namespace metallb-system
kubectl label namespace metallb-system \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged

helm upgrade --install metallb metallb/metallb \
  --namespace metallb-system \
  --wait --timeout 5m
```

> If kube-proxy runs in IPVS mode, `strictARP: true` must be set in the
> kube-proxy ConfigMap first. kubeadm defaults to iptables mode, where
> this is a no-op.

Apply the address pool and L2 advertisement:

```
kubectl apply -f metallb-config.yaml
```

Verify:

```
kubectl get pods -n metallb-system
# controller + 1 speaker per node, all Running

kubectl get ipaddresspool,l2advertisement -n metallb-system
# homelab-pool  ["10.0.0.50-10.0.0.60"]  /  homelab-l2
```

---

## Step 3 — Create Namespace and Admin Secret

The admin password lives in a Secret, not in `pihole-values.yaml`
(values end up in Helm release history).

```
kubectl create namespace pihole

kubectl -n pihole create secret generic pihole-admin \
  --from-literal=password='<STRONG-PASSWORD>'
```

---

## Step 4 — Deploy Pi-hole

Values specific to *your* cluster go in a local override file. It's
gitignored, so the committed `pihole-values.yaml` stays generic. At
minimum, pin the pod to one of your nodes (`kubectl get nodes`):

```
cat > pihole-values.local.yaml <<'EOF'
nodeSelector:
  kubernetes.io/hostname: worker-1   # ← replace with YOUR node name
EOF
```

Then install with **both** files (the later file wins):

```
helm upgrade --install pihole mojo2600/pihole \
  --namespace pihole \
  --values pihole-values.yaml \
  --values pihole-values.local.yaml \
  --wait --timeout 5m
```

> ⚠️ Pass both files on **every** future `helm upgrade` too. See
> [Upgrading safely](#upgrading-safely) for why, and for what breaks if
> you don't.

Verify both Services hold the VIP:

```
kubectl -n pihole get pods,svc
```

Expected output:

```
NAME                          READY   STATUS    RESTARTS   AGE
pod/pihole-xxxxxxxxxx-xxxxx   1/1     Running   0          2m

NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)
service/pihole-dns   LoadBalancer   10.x.x.x         10.0.0.53     53:xxxxx/TCP,53:xxxxx/UDP
service/pihole-web   LoadBalancer   10.x.x.x         10.0.0.53     80:xxxxx/TCP,443:xxxxx/TCP
```

Key choices baked into `pihole-values.yaml` (see the file for full
inline comments):

| Setting | Value | Why |
| ------- | ----- | --- |
| `strategyType` | `Recreate` | FTL's SQLite DB tolerates exactly one writer |
| `nodeSelector` | *(unset — set in `pihole-values.local.yaml`)* | local-path PVs are node-local; the override makes first placement deterministic |
| `serviceDns.mixedService` | `true` | TCP+UDP 53 on one Service/VIP |
| `externalTrafficPolicy` | `Local` | real client IPs in the query log |
| `serviceDhcp.enabled` | `false` | DHCP stays on the Omada gateway |
| `FTLCONF_dns_listeningMode` | `all` | required in k8s — queries arrive on virtual interfaces |
| `FTLCONF_database_maxDBdays` | `91` | caps FTL history so the 2Gi PVC never fills |
| `podDnsConfig` | `127.0.0.1` + `1.1.1.1` | the pod must never depend on itself for DNS |

> **Pi-hole v6 note:** anything set via `FTLCONF_*` env vars is
> **read-only in the web UI** — env vars are the single source of truth.
> To change one of those settings, edit `pihole-values.yaml` and
> `helm upgrade`.

---

## Step 5 — Apply Tailscale Ingress

Not managed by Helm, so it survives chart upgrades — same pattern as the
observability stack:

```
kubectl apply -f pihole-ts-ingress.yaml
```

Verify:

```
kubectl -n pihole get ingress
# pihole-ingress   tailscale   *   pihole-homelab.<tailnet>.ts.net   80, 443

kubectl get pods -n tailscale | grep pihole
# ts-pihole-ingress-xxxxx-0   1/1   Running
```

Also confirm `pihole-homelab` appears in **Tailscale admin → Machines**.

---

## Step 6 — Verify DNS Before Touching the Router

From a LAN machine (**not** a cluster node — see Design Decisions):

```
dig @10.0.0.53 example.com +short
# 23.x.x.x  (real answer)

dig @10.0.0.53 example.com +tcp +short
# same answer over TCP

dig @10.0.0.53 doubleclick.net +short
# 0.0.0.0   (blocked)
```

Then log in at `http://10.0.0.53/admin` with the Step 3 password.

---

## Step 7 — Point the Network at Pi-hole (Omada)

**Omada Controller → Settings → Wired Networks → LAN → edit → Advanced
Settings → DNS Server → Manual:** *(on any other router: the equivalent
setting is the DNS server handed out by the LAN's DHCP server)*

- **DNS 1:** `10.0.0.53`
- **DNS 2:** *(empty)*

> **Do not** add a public resolver (8.8.8.8 / 1.1.1.1) as DNS 2. Clients
> treat DNS servers as a set, not a fallback order — many devices query
> the public resolver directly and bypass blocking, producing
> inconsistent filtering that is maddening to debug.

Clients pick up the new DNS on **DHCP lease renewal** (lease time here:
120 min). To force it immediately: toggle Wi-Fi on the device, or reboot
the wireless APs to make every client reassociate at once.

Verify on a client (not a cluster node):

```
nslookup doubleclick.net
# Server:  10.0.0.53          ← must be the VIP
# Address: 0.0.0.0            ← blocked
```

Then watch **Pi-hole → Query Log** while browsing on that device —
its queries should stream in, blocked entries in red.

---

## Client-Side Bypass Checklist

If a client shows `10.0.0.53` as its DNS server but ads persist:

| Escape route | Where to check |
| ------------ | -------------- |
| Browser DoH ("Secure DNS") | Chrome/Edge → Privacy & Security → disable, or set to "with your current provider"; Firefox → DoH off |
| Android Private DNS | Set to Automatic/Off (a hostname like `dns.google` ignores DHCP DNS entirely) |
| IPv6 DNS | Disabled on this network (WAN IPv6 off, LAN IPv6 Interface Type: None) — if ever enabled, dual-stack clients will resolve over IPv6 and skip Pi-hole |
| Hardcoded-DNS devices | Smart TVs / IoT that ignore DHCP — needs a router firewall rule redirecting outbound :53, out of scope here |

**Expectation check:** DNS blocking cannot remove YouTube / Facebook /
Instagram / Twitch ads — they are served from the same domains as the
content. Score what *is* blocked at
[d3ward.github.io/toolz/adblock.html](https://d3ward.github.io/toolz/adblock.html)
(~60–70% on default lists is healthy).

---

## Operations

```
# Upgrade chart / image (expect a brief network-wide DNS blip — Recreate strategy)
# ALWAYS pass both values files — see "Upgrading safely" below
helm repo update
helm upgrade pihole mojo2600/pihole -n pihole \
  -f pihole-values.yaml -f pihole-values.local.yaml \
  --wait --timeout 5m

# Roll back a bad upgrade
helm rollback pihole -n pihole

# Rebuild blocklists (gravity)
kubectl -n pihole exec deploy/pihole -- pihole -g

# Tail the container log
kubectl -n pihole logs deploy/pihole -f

# Config backup (also: Settings → Teleporter → Export in the UI)
kubectl -n pihole exec deploy/pihole -- tar czf - /etc/pihole > pihole-backup-$(date +%F).tgz
```

### Upgrading safely

> ⚠️ **Read this before every `helm upgrade`.**
>
> `strategyType: Recreate` deletes the running Pi-hole pod **before** the
> new one starts, so the whole LAN has no DNS for that window. If the new
> pod can't start, the outage lasts until someone notices. The easiest
> way to cause that is a `nodeSelector` naming a node that doesn't exist:
> the new pod sits `Pending` indefinitely.

1. **Real node names live only in `pihole-values.local.yaml`.** The
   committed `pihole-values.yaml` has no active `nodeSelector`. Its
   `worker-1` is a commented-out example, so don't uncomment it as-is.
2. **Pass both files on every upgrade.** If the local file is forgotten,
   Helm drops the `nodeSelector`. The pod still lands on the right node
   (the PV's node affinity pins it), but the pod spec changes, so you get
   an unneeded restart now and another when the file is added back.
3. **Check the rendered node name before upgrading.** It must be the node
   that holds the PVC:

   ```
   helm template pihole mojo2600/pihole \
     -f pihole-values.yaml -f pihole-values.local.yaml \
     | grep -A1 nodeSelector
   # kubernetes.io/hostname: <a name listed by `kubectl get nodes`>

   kubectl get pv -o custom-columns=CLAIM:.spec.claimRef.name,NODE:.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0] \
     | grep pihole
   # pihole   <the same node name>
   ```

4. **If the pod is `Pending` after an upgrade**, restore DNS first and
   investigate second: `helm rollback pihole -n pihole` brings back the
   previous revision. Then fix the values and upgrade again. If the
   cluster itself is the problem, use the escape hatch below.

**Emergency escape hatch:** if the cluster will be down for a while,
set the Omada LAN's DHCP DNS back to **Auto** — the network gets working
(unfiltered) internet within a lease renewal. Flip back to Manual /
`10.0.0.53` when the cluster returns.

**Failure behavior:**

| Event | Effect |
| ----- | ------ |
| Pod crash / OOM | kubelet restarts it in seconds; clients retry and barely notice |
| `helm upgrade` | seconds-to-a-minute DNS blip (Recreate strategy) — do it at low-traffic times |
| The Pi-hole node reboots | DNS down until the node returns (PVC is node-local) — minutes for a planned reboot; use the escape hatch for anything longer |
| Control-plane down | Pi-hole keeps serving — the running pod doesn't need the API server |

---

## Design Decisions

- **L2 mode, not BGP.** MetalLB BGP conflicts with Calico's BGP on this
  cluster, and the Omada gateway isn't a BGP peer anyway.
- **The cluster nodes do NOT use Pi-hole for DNS.** If the nodes
  resolved through a Pi-hole running *on* them, a cold cluster boot would
  deadlock: nodes need DNS to pull images to start the pod that provides
  DNS. Node DNS stays on the router/public resolvers — which is also why
  `nslookup` on a node returns real answers for blocked domains
  (systemd-resolved at `127.0.0.53` → upstream, by design).
- **The VIP never changes after rollout.** Every DHCP client caches it;
  changing it strands devices until leases renew.
- **Single instance, no HA.** Accepted trade-off for this homelab: DNS
  availability is coupled to cluster health, mitigated by the escape
  hatch above and doing disruptive work at low-traffic times.
- **`tailscale set --accept-dns=false` before any tailnet-wide DNS
  override.** The cluster nodes run tailscaled; a tailnet nameserver
  override would rewrite *their* resolv.conf to point at the Pi-hole they
  host — the bootstrapping deadlock in disguise.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
| ------- | ------------ | --- |
| Service `EXTERNAL-IP` stuck `<pending>` | MetalLB missing, pool missing, or VIP outside pool | `kubectl describe svc pihole-dns -n pihole` — the Events section is definitive |
| `AllocationFailed ... address also in use by <ns>/<svc>` | Another LoadBalancer Service holds the VIP (this happened with `wazuh/wazuh-workers`) | Free the IP (convert the other Service to ClusterIP) or pick a different VIP; update the IP plan table |
| `dig` → `host unreachable` | No LoadBalancer holds the VIP → nothing answers ARP | Same as above — fix the allocation first |
| VIP assigned but `dig` times out | `FTLCONF_dns_listeningMode` not `all`, or only one protocol exposed | Confirm the env var and `mixedService: true` |
| Blocked domains resolve on a cluster node | Working as designed — nodes bypass Pi-hole | Test from a real client |
| Client has right DNS but ads persist | Browser DoH / Android Private DNS / platform ads | See Client-Side Bypass Checklist |
| Whole network slow, dashboard shows one mega-client | `externalTrafficPolicy` was switched to `Cluster` → SNAT + FTL rate limiting | Revert to `Local` |
| Web UI toggle greyed out | Setting is env-var-controlled (v6 makes `FTLCONF_*` read-only in UI) | Change in `pihole-values.yaml`, `helm upgrade` |
| Pod `Pending` | `nodeSelector` names a node that doesn't exist (e.g. the `worker-1` example left in) or isn't the one holding the PVC (`volume node affinity conflict`), or that node is down | Right after an upgrade, `helm rollback pihole -n pihole` first, then `kubectl -n pihole describe pod`. See [Upgrading safely](#upgrading-safely) |

---

## Related Repositories

- [homelab-k8s-observability-stack](https://github.com/vin-ediezca/homelab-k8s-observability-stack) — Tailscale Operator install, Grafana/Prometheus/Loki/Tempo
- [homelab-k8s-wazuh-siem](https://github.com/vin-ediezca/homelab-k8s-wazuh-siem) — Wazuh SIEM (services converted to ClusterIP to free the MetalLB pool)

---

## References

- Pi-hole v6 Docker configuration & `FTLCONF_*` reference — docs.pi-hole.net/docker
- mojo2600/pihole-kubernetes Helm chart — github.com/MoJo2600/pihole-kubernetes
- MetalLB Layer 2 concepts & configuration — metallb.universe.tf
- Tailscale Kubernetes Operator — tailscale.com/kb
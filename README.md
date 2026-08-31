# VM → Docker → Kubernetes: a practical that *proves* why each step matters

Most "VM vs Docker vs Kubernetes" tutorials just *describe* the differences.
This one **proves them**: there are runnable experiments in `experiments/` that
actually **break on a bare VM**, then get **fixed by Docker**, then get
**operated by Kubernetes** — and the real captured output is saved next to each
one so you can show it even without running anything.

The story in one line: **run it (VM) → package it (Docker) → orchestrate it (K8s).**

```
Problem on a bare VM            Who fixes it        Proof
────────────────────────────   ─────────────────   ─────────────────────────────
Two apps' dependencies clash    Docker (isolation)  experiments/01  &  03
A crashed process stays dead    Kubernetes          experiments/02  &  04
Can't scale / load-balance      Kubernetes          experiments/04
No self-healing / no rollout    Kubernetes          experiments/04
```

---

## Quick start — run the whole proof

Only `python3` is needed (plus internet for two tiny `pip install`s). **No
Docker or kubectl required** — the sandboxed experiments use virtualenvs and a
pure-Python "toy Kubernetes" as stand-ins, and ship the real Docker/K8s files
alongside for when you have a cluster.

```bash
bash experiments/run_all.sh
```

Or run them one at a time (each prints a clear 🔴 PROVEN / 🟢 FIXED verdict):

```bash
bash   experiments/01_vm_dependency_conflict/run.sh
bash   experiments/02_vm_no_selfheal/run.sh
bash   experiments/03_docker_fixes_it/run.sh
python3 experiments/04_k8s_superpowers/mini_orchestrator.py
```

Full captured logs live in each experiment's `OUTPUT.md`.

---

## Part 1 — Proving the bare-VM problems

A "bare VM" means: one machine, one shared OS, one shared set of system
libraries and one shared Python. Everything you deploy lives together. That
sharing is exactly where the pain comes from.

### Problem A — dependency hell (experiment 01)

Two apps land on the same VM. `app_a` (legacy) needs **Jinja2 3.0.3**; `app_b`
(new) needs **Jinja2 3.1.4**. A machine can hold only **one** version of a
package at a time, so deploying the second app silently changes the library out
from under the first.

```text
== STEP 1 — Team A deploys app_a onto the VM ==
[app_a] running on Jinja2 3.0.3
[app_a] OK  ✅   (legacy report generated)

== STEP 2 — Team B deploys app_b onto the SAME VM ==
Uninstalling Jinja2-3.0.3:
Successfully installed Jinja2-3.1.4
[app_b] OK  ✅   (analytics job ran)

== STEP 3 — Nobody touched app_a. Let's just run it again. ==
ImportError: cannot import name 'Markup' from 'jinja2'

🔴 PROVEN: deploying app_b BROKE the already-working app_a.
```

`app_a` was never edited. It broke because the shared machine could not satisfy
both dependency sets at once. On a real VM your options are all bad: pin
everything to the lowest common version, never upgrade, or juggle fragile
per-app virtualenvs by hand. → full log: `experiments/01_vm_dependency_conflict/OUTPUT.md`

### Problem B — no self-healing (experiment 02)

Start the app, confirm it serves traffic, then let the process die (a bug, an
out-of-memory kill, a bad deploy). Nothing brings it back.

```text
== STEP 2 — Confirm it's serving traffic ==
GET / -> ok host=vm pid=820 path=/      ✅ service is UP

== STEP 3 — The process crashes (we simulate it with 'kill') ==
== STEP 4 — Wait 3 seconds, then try again. ==
GET / -> <connection refused>
🔴 PROVEN: the app is DOWN and nothing restarted it.
```

`systemd Restart=on-failure` (used by `vm/myapp.service`) can restart a process
on *one* box — but it can't survive the machine dying, can't run multiple copies,
and can't load-balance or roll out a new version. Those need orchestration.

---

## Part 2 — Why Docker matters (experiment 03)

The conflict in experiment 01 existed **only** because both apps shared one
environment. Docker's core idea: **package each app with its own dependencies
into an isolated, portable image.** Then two incompatible versions coexist
happily, because they never touch the same libraries.

The sandbox has no Docker daemon, so `run.sh` proves the *principle* with one
isolated virtualenv per app (a venv is the on-one-machine version of what a
container image does). The **real, portable** version is in the Dockerfiles in
that folder — see `experiments/03_docker_fixes_it/DOCKER.md`.

```text
== STEP 3 — Run BOTH apps, each in its own box, back to back ==
[app_a] running on Jinja2 3.0.3   ... OK ✅
[app_b] running on Jinja2 3.1.4   ... OK ✅

🟢 FIXED: the SAME two apps that clashed on one VM now BOTH work.
```

What Docker buys you over a bare VM:

- **Isolation** — each container has its own dependencies and filesystem view.
- **"Works on my machine" → works everywhere** — the image bundles the runtime,
  the OS libraries and the code, so the artifact you test is the artifact you run.
- **Speed** — start/stop in seconds, not minutes of provisioning.
- **Reproducibility** — the `Dockerfile` *is* the setup, versioned in git,
  instead of a pile of manual `apt-get` steps someone ran once.

For real: `cd experiments/03_docker_fixes_it && docker compose up --build`.

---

## Part 3 — Why Kubernetes matters (experiment 04)

Docker packages and can even restart **one** container on **one** host. It does
**not**, on its own: run many replicas, load-balance across them, reschedule
them when they die, or scale on command. That operational layer is Kubernetes.

`experiments/04_k8s_superpowers/mini_orchestrator.py` is a ~150-line pure-Python
"toy Kubernetes" that actually does these things so you can watch them happen:

```text
== kubectl apply — desired: 3 replicas ==
   pods: {'pod-01': 974, 'pod-02': 975, 'pod-03': 976}

== Load balancing: 6 requests through the Service ==
   client GET / -> served by pod=pod-01 ... pod-02 ... pod-03 ... (cycles)

== Self-healing: one pod crashes ==
   killed pod-01 (simulated crash)
   pods 1.5s later: {'pod-02', 'pod-03', 'pod-04'}
   🟢 the ReplicaSet noticed and recreated a pod back to 3/3

== kubectl scale --replicas=5 (one command) ==
   requests now spread across 5 pods

🟢 replicas + load balancing + self-healing + scaling — that's Kubernetes.
```

Every toy behaviour maps directly to a real Kubernetes object, and the real
manifests to reproduce it on `minikube`/`kind` are in `k8s/` with a walkthrough
in `experiments/04_k8s_superpowers/K8S.md`:

| Toy orchestrator | Real Kubernetes | File |
|---|---|---|
| keep N pods alive (reconcile loop) | Deployment / ReplicaSet | `k8s/deployment.yaml` |
| round-robin front end | Service | `k8s/service.yaml` |
| env from a shared config | ConfigMap | `k8s/configmap.yaml` |
| `scale(n)` | `kubectl scale` / `replicas:` | `k8s/deployment.yaml` |
| respawn a killed pod | controller self-healing | (automatic) |

---

## The demo app (used by all three real deployments)

`app/app.py` is a small Flask service that reports its **hostname**, **deploy
mode**, **version** and a **visit counter**. That makes the deployment method
visible in a browser:

- on a **VM** the hostname never changes (one machine),
- in **Docker** it's the container id,
- on **Kubernetes** (3 pods) it *changes between refreshes* as the Service
  load-balances — the same effect the toy orchestrator prints in experiment 04.

| Route | Purpose |
|---|---|
| `/` | HTML page: hostname / mode / version / visits |
| `/api/info` | same data as JSON |
| `/healthz` | health check for Docker & K8s probes |

Run it locally with no infra:

```bash
cd app && python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt && python app.py   # http://localhost:8080
```

---

## Deploy the app for real, three ways

These are the production-style files (unchanged classics), one folder each.

### Method 1 — bare VM (`vm/`)
System packages + a virtualenv + a dedicated user + a `systemd` unit.
```bash
cd vm && sudo bash setup.sh          # provisions the machine and starts the service
systemctl status myapp ; journalctl -u myapp -f
```
The classic way — and the one whose problems Part 1 proves.

### Method 2 — Docker (`docker/`)
```bash
docker build -t devops-demo:1.0.0 -f docker/Dockerfile .
docker run -d --name demo -p 8080:8080 devops-demo:1.0.0
# or: cd docker && docker compose up --build
```

### Method 3 — Kubernetes (`k8s/`)
```bash
minikube start
docker build -t devops-demo:1.0.0 -f docker/Dockerfile .
minikube image load devops-demo:1.0.0
kubectl apply -f k8s/configmap.yaml -f k8s/deployment.yaml -f k8s/service.yaml
kubectl get pods -w
minikube service devops-demo --url      # refresh a few times, watch the hostname change
```
Then try the superpowers live:
```bash
kubectl scale deployment devops-demo --replicas=5     # scale
kubectl delete pod <pod-name>                         # self-heal (it comes back)
kubectl set image deployment/devops-demo web=devops-demo:1.1.0   # rolling update
kubectl rollout undo deployment/devops-demo           # instant rollback
```

---

## Side-by-side

| Aspect | VM | Docker | Kubernetes |
|---|---|---|---|
| What you ship | source + manual setup | a self-contained image | images + declarative YAML |
| Dependency isolation | ❌ shared, apps clash (exp 01) | ✅ per-image | ✅ per-pod |
| "Works on my machine" | common problem | solved | solved |
| Startup | minutes | seconds | seconds per pod |
| Self-healing | ❌ manual (exp 02) | restart one container | ✅ across the cluster (exp 04) |
| Scaling + load balancing | new VM + manual LB | `docker run` more, manual LB | ✅ `kubectl scale`, built-in LB (exp 04) |
| Rolling update / rollback | manual, risky | manual | ✅ one command |
| Config management | files/env on host | env vars / mounts | ConfigMaps & Secrets |
| Best for | one simple, stable service | consistent packaging & local dev | many services, scale, HA |

**Mental model:** VM = *a whole computer you babysit*. Docker = *the app in a
box that runs the same everywhere*. Kubernetes = *a robot ops team that runs,
heals, scales and updates your boxes for you.* They stack: a real K8s cluster is
itself a bunch of VMs running Docker containers.

---

## Project structure

```
devops-deployment-demo/
├── README.md                     # this file
├── app/                          # the shared Flask demo app
│   ├── app.py · requirements.txt · templates/index.html
├── experiments/                  # ⭐ the runnable proof
│   ├── run_all.sh                # run all four in order
│   ├── 01_vm_dependency_conflict/  (run.sh · app_a/ · app_b/ · OUTPUT.md)
│   ├── 02_vm_no_selfheal/          (run.sh · service.py · OUTPUT.md)
│   ├── 03_docker_fixes_it/         (run.sh · Dockerfile.app_a/_b · docker-compose.yml · DOCKER.md · OUTPUT.md)
│   └── 04_k8s_superpowers/         (mini_orchestrator.py · worker.py · K8S.md · OUTPUT.md)
├── vm/                           # Method 1: setup.sh · myapp.service
├── docker/                       # Method 2: Dockerfile · docker-compose.yml · .dockerignore
└── k8s/                          # Method 3: configmap · deployment · service · ingress
```

---

## Troubleshooting

- **Experiment 1/3 `pip` errors** → they need internet to fetch two tiny
  Jinja2 wheels; re-run once you're online. Both scripts clean up their temp
  virtualenvs automatically.
- **Experiment 4 prints `no pods available`** → a pod was still starting; it's
  transient, the reconcile loop fills it in. Re-run if you see it repeatedly.
- **`docker compose` build context errors (exp 03)** → run it from inside
  `experiments/03_docker_fixes_it/` so the relative build contexts resolve.
- **k8s pods stuck in `ImagePullBackOff`** → load the image into the cluster
  (`minikube image load devops-demo:1.0.0`) and keep `imagePullPolicy: IfNotPresent`.
- **Port already in use** → change the published port (`-p 9090:8080`) or the
  experiment's port variable near the top of the script.

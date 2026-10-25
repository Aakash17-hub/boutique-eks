# ShopSphere

A hands-on Kubernetes project: Google's **Online Boutique** (11 microservices) running on **Minikube on an AWS EC2 instance**, with hand-written manifests, per-service RBAC, default-deny NetworkPolicies and a **GitLab CI/CD** pipeline that validates and deploys from git.

The goal was not just to get the demo running, but to write every manifest myself, break things on purpose, and document what I learned.

> The application itself is [GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo) (Apache 2.0), image version `v0.10.7`. The original release manifest is kept unchanged in `upstream/` for reference. Everything in `k8s/`, `bootstrap/` and `.gitlab-ci.yml` is my own work.

---

## Architecture

```mermaid
flowchart TB
    laptop["Developer laptop"]
    bastion["Bastion EC2<br/>Public subnet"]
    subgraph AWS["AWS environment"]
        subgraph PUBLIC["Public subnet"]
            mini["ShopSphere Minikube EC2<br/>Ubuntu 24.04 · Docker driver<br/>GitLab Runner · shell executor"]
        end
        subgraph PRIVATE["Private subnet"]
            gitlab["GitLab EC2<br/>Repository · CI/CD pipelines"]
        end
    end

    laptop -->|"SSH ProxyJump"| bastion
    bastion -->|"SSH"| mini
    mini -->|"Runner polls GitLab for jobs"| gitlab
    gitlab -.->|"Pipeline triggers job execution on runner"| mini
    mini --> workloads["Kubernetes namespace: shopsphere<br/>11 application workloads"]

    classDef host fill:#eaf3ff,stroke:#3777b8,color:#17365d
    classDef service fill:#eaf8ef,stroke:#27834a,color:#174d2a
    class laptop,bastion,mini,gitlab host
    class workloads service
```

- The Minikube host has its own security group. Inbound is **SSH only, from the bastion's security group**. Nothing else is exposed.
- The frontend Service is `ClusterIP`. I reach it with `kubectl port-forward` through an SSH tunnel, so there is no public load balancer.
- The Minikube host sits in a public subnet so it can pull images without a NAT gateway (a NAT gateway is the most expensive idle resource in this setup).
- Host: Ubuntu 24.04 on `m7i-flex.large` (2 vCPU, 8 GiB), 40 GB gp3, 4 GB swap. Minikube runs with `--driver=docker --cpus=2 --memory=6144`.

### Service call graph

```mermaid
flowchart LR
  user([Browser via SSH tunnel]) --> frontend
  frontend --> productcatalogservice
  frontend --> currencyservice
  frontend --> cartservice
  frontend --> shippingservice
  frontend --> recommendationservice
  frontend --> checkoutservice
  frontend --> adservice
  recommendationservice --> productcatalogservice
  checkoutservice --> productcatalogservice
  checkoutservice --> currencyservice
  checkoutservice --> cartservice
  checkoutservice --> shippingservice
  checkoutservice --> paymentservice
  checkoutservice --> emailservice
  cartservice --> redis-cart
```

### Ports

| Service | Container port | Service port |
|---|---|---|
| frontend | 8080 | 80 |
| productcatalogservice | 3550 | 3550 |
| currencyservice | 7000 | 7000 |
| cartservice | 7070 | 7070 |
| shippingservice | 50051 | 50051 |
| paymentservice | 50051 | 50051 |
| emailservice | **8080** | **5000** |
| recommendationservice | 8080 | 8080 |
| checkoutservice | 5050 | 5050 |
| adservice | 9555 | 9555 |
| redis-cart | 6379 | 6379 |

The load generator from the upstream demo is intentionally not deployed (the node only has 2 vCPUs).

---

## Repository layout

```
.
├── upstream/                 # original Online Boutique release manifest (unchanged)
├── k8s/                      # my manifests, applied by CI
│   ├── 00-namespace.yaml
│   ├── 01-rbac.yaml          # ServiceAccounts, read-only Role
│   ├── 02-networkpolicies.yaml
│   └── <service>.yaml        # one Deployment + Service per service (11 files)
├── bootstrap/
│   └── ci-deployer.yaml      # least-privilege identity for CI (applied by hand, NOT by CI)
├── docs/                     # screenshots and notes
└── .gitlab-ci.yml
```

---

## What is implemented

### 1. Manifests (`k8s/`)

- One namespace, `shopsphere`. Every object carries `app` and `app.kubernetes.io/part-of` labels.
- **Probes** on every workload: gRPC probes for the gRPC services, an HTTP probe on `/_healthz` for the frontend, and a TCP readiness plus `redis-cli ping` liveness probe for Redis.
- **Requests and limits** chosen from real usage (`kubectl top`), not copied blindly:

| Workload | CPU request / limit | Memory request / limit |
|---|---|---|
| most gRPC services (currency, productcatalog, shipping, payment, email, checkout) | 20m / 200m | 32Mi / 128Mi |
| cartservice | 30m / 200m | 48Mi / 128Mi |
| frontend | 20m / 200m | 64Mi / 128Mi |
| recommendationservice | 30m / 200m | 128Mi / 450Mi |
| adservice (JVM, slower start) | 20m / 300m | 128Mi / 300Mi |
| redis-cart | 20m / 100m | 32Mi / 128Mi |

### 2. RBAC

- Every service has its **own ServiceAccount** with `automountServiceAccountToken: false`. None of these services talks to the Kubernetes API, so no token is mounted into the pods.
- `ops-viewer`: a namespaced, read-only Role (get/list/watch on pods, pod logs, services, deployments) with a RoleBinding. It cannot delete anything or read secrets.

### 3. NetworkPolicies

- `default-deny-ingress` applies to pods across the namespace. The file defines 11 destination-specific allow policies: 10 service-to-service rules and one frontend ingress rule. The frontend rule allows traffic to pod port 8080 without a source-selector restriction.
- Rules use container ports, not Service ports (this matters for `emailservice`).
- Ingress is restricted by policy rules; egress is not restricted yet (see Limitations).
- Enforcement was verified on the original Minikube profile (default `kindnet` CNI, image `kindest/kindnetd:v20260820-69b56db7`). In a throwaway namespace, a deny-all policy turned a working request into a timeout. In `shopsphere`, a test pod without a permitted label could not reach `paymentservice`, while a pod labelled `app=checkoutservice` could. This covers the tested paths only, not every rule or egress behaviour.

### 4. CI/CD (GitLab CI)

A shell runner on the Minikube host runs two stages:

| Stage | What it does | When |
|---|---|---|
| `validate` | `kubectl apply --dry-run=server` on all manifests | every push |
| `deploy` | `kubectl apply`, then `kubectl rollout status` for every deployment | `main` branch only |

The pipeline does **not** use an admin kubeconfig. It uses a dedicated `ci-deployer` ServiceAccount whose Role can create/update/patch deployments, services, service accounts and network policies in `shopsphere`, and nothing else. It cannot delete, cannot read secrets, cannot touch RBAC objects and cannot reach other namespaces. For that reason the pipeline skips `00-namespace.yaml` and `01-rbac.yaml`; those are applied by an admin.

**Demo:** changing `replicas: 1` to `replicas: 2` in `k8s/frontend.yaml` and pushing created a second frontend pod with no manual `kubectl apply`. Pushing `replicas: 1` scaled it back. Rollback is a git commit.

### CI/CD evidence

The screenshots below are historical examples from pipeline #12, showing the validation and frontend scale-up demonstration. They are not screenshots of the latest pipeline.

![Pipeline history](docs/screenshots/ci-pipelines-history.png)
*Historical pipeline list: pipelines #9–#11 failed due to the YAML parsing issue documented below; #12 and #13 passed at the time of capture.*

![Validate job](docs/screenshots/ci-validate-12.png)
*Historical validation job: server-side dry run reports the planned frontend change.*

![Deploy job](docs/screenshots/ci-deploy-12-scale-up.png)
*Historical deploy job: the frontend rollout waits for the second replica and succeeds.*

**Latest recorded verification:** commit `cbac4b1` (`docs: improve NetworkPolicy verification and setup guide`) passed the GitLab validation and deployment stages. The deploy job applied the manifests and all 11 application deployments rolled out successfully. Capture a fresh GitLab pipeline screenshot if you want visual evidence of this latest run.

---

## Verification

| Check | Evidence / command | Result |
|---|---|---|
| Application health | `kubectl get pods -n shopsphere` on the original `minikube` profile | All 11 application pods were `1/1 Running` in the latest health check; historical synchronized restarts were observed and the exact root cause was not confirmed |
| Deployment rollout | `kubectl rollout status` for each deployment | All 11 deployments successfully rolled out |
| Frontend HTTP response | `curl -I http://127.0.0.1:8080` while forwarding `svc/frontend` port 80 to local port 8080 | `HTTP/1.1 200 OK` |
| End to end | Add an item to the cart and place a test order through the browser | Successfully completed in the latest smoke test; order flow reached completion |
| `ops-viewer` rights | `kubectl auth can-i list pods -n shopsphere --as=system:serviceaccount:shopsphere:ops-viewer` | `yes` (previously tested) |
| `ops-viewer` limits | Same check with `delete pods`, `get secrets`, or namespace `default` | `no` (previously tested) |
| `ci-deployer` rights and limits | `kubectl auth can-i` checks using its kubeconfig | Previously tested; the pipeline also successfully completed validate and deploy stages |
| NetworkPolicy enforcement | Separate `netpol-test` Minikube profile with Calico; temporary test pods used source labels `app=checkoutservice`, `app=cartservice`, and `app=netpol-test` | Three allowed paths succeeded and three denied paths timed out. Calico version was not captured. |

### NetworkPolicy test details

The Calico test was performed on the separate `netpol-test` profile, not the original `minikube` profile. The test pods were deleted afterward and the `netpol-test` profile was stopped. The Calico version was not recorded.

| Destination | Permitted test pod | Denied test pod | Result |
|---|---|---|---|
| `paymentservice:50051` | `netpol-allow-checkout` (`app=checkoutservice`) — connection open | `netpol-deny` (`app=netpol-test`) — connection timed out | Pass |
| `emailservice:5000` | `netpol-allow-checkout` (`app=checkoutservice`) — connection open | `netpol-deny` (`app=netpol-test`) — connection timed out | Pass |
| `redis-cart:6379` | `netpol-allow-cart` (`app=cartservice`) — connection open | `netpol-deny` (`app=netpol-test`) — connection timed out | Pass |

These tests verify the three listed traffic paths only. They do not prove every NetworkPolicy rule or egress behavior. The original cluster and the Calico test cluster are separate; do not infer Calico enforcement on the original profile from these tests.

---

## Lessons and troubleshooting log

**Pods stuck in `Pending` while the node was almost idle.** The events said `Insufficient cpu`. The scheduler works with CPU *requests*, which were at 97% of the node, while real usage was around 7%. Fix: set requests from `kubectl top` measurements.

**Service port vs container port (emailservice).** `checkoutservice` calls `emailservice:5000`, which is the Kubernetes Service port; the emailservice container listens on port 8080. Keep the distinction clear when debugging connectivity: clients address the Service port, while NetworkPolicy rules are evaluated against traffic to the destination pod port. The Calico test connected through the `emailservice:5000` Service port; NetworkPolicy evaluates traffic at the destination pod port, 8080.

**A passing readiness probe does not mean healthy dependencies.** I set `REDIS_ADDR=wrong-host:6379` on `cartservice`. The new pod still became Ready, because its probe only checks that the process is alive, and the rolling update replaced the good pod with the broken one. Mitigations: dependency-aware probes, and monitoring/alerting (planned).

**NetworkPolicy selects pods by labels, not by workload identity.** On the Calico test profile, a test pod labelled `app=checkoutservice` was permitted to reach `paymentservice`. A principal allowed to create pods in the namespace may be able to assign a permitted label, so RBAC must restrict pod creation as well as protect privileged resources. The `ops-viewer` role cannot create pods. This test does not establish that the original kindnet-backed profile enforces NetworkPolicies.

**CI kubeconfig and the Minikube CA.** Minikube's kubeconfig points to the CA *file* in the admin user's home directory instead of embedding it. The runner user cannot read it, so kubectl failed with `x509: certificate signed by unknown authority`. Fix: base64-embed the CA into the runner's kubeconfig.

**YAML trap in `.gitlab-ci.yml`.** `- echo "Manifests: $FILES"` is valid YAML, but it parses as a mapping (`{'echo "Manifests': '$FILES"'}`) instead of a string, so GitLab rejected the file and created no jobs (pipelines #9 to #11). Replacing the colon with `=` fixed it. A local `yaml.safe_load` passed the broken file, so it is not a reliable check: use GitLab's Pipeline editor validation instead.

**Configuration drift.** `kubectl set env` changes live state that is not in git. `kubectl rollout undo` plus `kubectl apply -f` brought the cluster back in sync. Git should be the source of truth, and the pipeline enforces that.

**`kubectl port-forward` pins to one pod.** After a rollout replaces the pod, the forward dies. Run it with `nohup` and restart it after deploys.

**SSH through a bastion.** The security-group rule "SSH from the bastion's security group" only matches traffic from the bastion's *private* address, so the jump target must be the instance's private IP.

---

## Limitations

- Single node, single replica per service. This is a learning environment, not high availability.
- Redis uses emptyDir, so carts are lost when its pod restarts.
- Egress is unrestricted. Three allowed and three denied traffic attempts were tested on a separate Calico-backed profile; enforcement of every policy and the original kindnet-backed profile remains unverified.
- Images are pinned by tag (v0.10.7), not by digest.
- No Ingress, TLS or public endpoint by design.
- The ci-deployer token is a long-lived ServiceAccount token stored in the runner user's kubeconfig. Better: short-lived tokens or a hardened runner.
- The GitLab Runner is a shell executor, so jobs run directly on the Minikube host.

## Roadmap

- [ ] Jenkins deploy pipeline (SSH to the Minikube host), alongside GitLab CI
- [ ] Monitoring and alerting: Prometheus, Grafana, Alertmanager (to catch the failure the readiness probe missed)
- [ ] Centralised logging
- [ ] Egress NetworkPolicies
- [ ] Migrate the main Minikube profile to Calico and repeat NetworkPolicy tests for the required traffic paths
- [ ] Short EKS proof of concept (deferred to a later phase)
- [ ] Terraform/Ansible for the host setup

---

## Running it

Prerequisites: an Ubuntu host with Docker, `kubectl` and `minikube` installed.

```bash
# For NetworkPolicy enforcement, create a separate Calico-backed test profile.
# --cni takes effect when the cluster is created.
minikube start -p netpol-test --driver=docker --cni=calico --cpus=2 --memory=6144

# For the main learning cluster (kindnet by default):
minikube start --driver=docker --cpus=2 --memory=6144

# namespace, RBAC, policies and all workloads (files are applied in order)
kubectl apply -f k8s/

# optional: identity used by CI
kubectl apply -f bootstrap/ci-deployer.yaml

kubectl get pods -n shopsphere

# open the shop at http://localhost:8080
kubectl port-forward -n shopsphere --address 127.0.0.1 svc/frontend 8080:80
```

On a remote host, add an SSH tunnel from your laptop: `ssh -N -L 8080:localhost:8080 <host>`.

## Cost notes

Instances are stopped between sessions (`minikube stop`, then stop the EC2 instances). No NAT gateway is needed to run this project, because the Minikube host is in a public subnet and GitLab traffic stays on private IPs.

## Credits

Application: [GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo), licensed under Apache 2.0.

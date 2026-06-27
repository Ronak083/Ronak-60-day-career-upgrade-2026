# Day 2 — Crossplane: Control Plane for Cloud Infrastructure

> **One-liner:** Crossplane turns Kubernetes into a control plane for cloud resources. You declare infrastructure as K8s objects; Crossplane *continuously reconciles* them to real AWS state — no `terraform apply`, no drift windows, no pipeline.

**What I actually built today:** a real **EKS** cluster with **IRSA**, ran **Crossplane v2.3.3**, provisioned a real **S3 bucket** and **RDS MySQL** instance with **zero static credentials**, connected a real **Spring Boot app** to the Crossplane-provisioned DB end-to-end, and started an `AppDatabase` self-service abstraction.

---

## 1. The core mental model (lead with this in interviews)

| | Terraform / Ansible (my SS&C past) | Crossplane |
|---|---|---|
| Execution | One-shot `apply` / push | **Continuous reconciliation loop** |
| Drift | Detected only on next `plan` | Detected & **healed within ~60s** |
| State | State file (S3/locking) | **etcd + live AWS = the state** |
| Credentials | CI holds cloud creds (push) | Cluster **pulls**, uses IRSA (no static keys) |
| Interface | HCL modules | **CRDs** — `kubectl get managed` |

**The single repeating pattern of the week:** ArgoCD (Day 1) and Crossplane (Day 2) are *both control loops*. Git/CRD = desired state, a controller continuously makes reality match. Recognising they're the same pattern is the mental-model upgrade.

---

## 2. The object model (know these cold)

- **Managed Resource (MR)** — lowest level. **1 MR = 1 cloud resource.** An `S3Bucket` MR = one bucket. *Always your first debugging surface — `kubectl describe` the MR.*
- **ProviderConfig** — *which* AWS account + *how* to auth (IRSA, in my case). Cluster-scoped. Referenced by name from MRs. Like Terraform's `provider` block.
- **Provider** — the plugin (`provider-aws-s3`) that installs the CRDs and runs a controller pod.
- **Composite Resource (XR)** — a custom abstraction *you* define (e.g. `AppDatabase`) that fans out into many MRs (RDS + subnet group + SG…). Developers ask for one thing, get many.
- **XRD (CompositeResourceDefinition)** — the *schema* of your XR (the fields a developer sets).
- **Composition** — the *implementation*: what cloud resources the XR expands into. (XRD + Composition ≈ a Terraform module that lives in K8s and reconciles forever.)
- **Claim / namespaced XR** — the developer-facing, namespace-scoped request. *(See §6 — this changed in Crossplane v2.)*

```
Developer:  AppDatabase (namespaced XR)  ──►  Composition + Function  ──►  RDS Instance (MR)  ──►  AWS
Platform team owns:        XRD + Composition
```

---

## 2b. What we actually installed — every component explained

This is the concrete inventory of what ran in the cluster, top to bottom.

### Crossplane core (Helm, namespace `crossplane-system`) — **v2.3.3**
Two pods:
- **`crossplane`** — the core controller. Runs the package manager (installs Providers/Functions) and the reconcilers for XRs/Compositions.
- **`crossplane-rbac-manager`** — auto-grants each Provider the RBAC it needs to manage its CRDs, so you don't hand-write ClusterRoles per provider.

Installing core alone gives you ~25 CRDs (Provider, ProviderConfig, Composition, XRD, Function…) but **no AWS knowledge yet** — that comes from providers.

### Providers — the plugins that teach Crossplane about AWS
A **Provider** = a package that (a) installs CRDs for one AWS service and (b) runs a controller pod that calls the AWS API. We installed:

| Provider | Version | What it gave us |
|---|---|---|
| **`provider-aws-s3`** | v1.21.0 | CRDs in group `s3.aws.upbound.io` — `Bucket`, `BucketPolicy`, `BucketVersioning`, … |
| **`provider-aws-rds`** | v1.21.0 | CRDs in group `rds.aws.upbound.io` — `Instance`, `SubnetGroup`, … |
| **`provider-family-aws`** | v2.6.1 | Pulled in *automatically* as a dependency. Ships the shared **`ProviderConfig`** CRDs and the AWS auth machinery. |

Why a "family" provider? Upbound split the giant monolithic AWS provider into small per-service providers (s3, rds, ec2…) that all share one **family** provider for auth/config. You install only the services you need → far fewer CRDs and a lighter cluster. Each service provider runs its **own controller pod** under its **own ServiceAccount**.

> Counting CRDs before/after a provider install is the quickest way to *see* this: core ≈ 25 CRDs; after `provider-aws-s3` it jumps by the whole `s3.aws.upbound.io` group.

### ProviderConfig — *which account, how to auth* (`providerconfig.yaml`)
```yaml
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default            # cluster-scoped; referenced by name from every MR
spec:
  credentials:
    source: IRSA           # "don't use a Secret — use the injected web-identity token"
```
One `ProviderConfig` named `default`, shared by **both** providers (it lives in the shared family group). `source: IRSA` is the line that says "authenticate via the pod's IAM-role token, no static keys." Think of it as Terraform's `provider "aws" {}` block — but the credentials come from IRSA at runtime.

### DeploymentRuntimeConfig — pins the SA + IRSA annotation (`runtimeconfig*.yaml`)
A provider normally generates a **random** ServiceAccount name, which makes IRSA impossible (the IAM trust policy must name an exact SA). The `DeploymentRuntimeConfig` fixes that:
```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: aws-irsa-config
spec:
  serviceAccountTemplate:
    metadata:
      name: provider-aws-s3                                   # fixed, predictable SA name
      annotations:
        eks.amazonaws.com/role-arn: arn:aws:iam::<acct>:role/crossplane-provider-aws-s3
```
The provider references it via `runtimeConfigRef`. We made **one per provider** (`aws-irsa-config` for s3, `aws-irsa-rds-config` for rds) because each provider's pod runs under a different SA and assumes a different IAM role. *(This is the modern replacement for the old, deprecated `ControllerConfig`.)*

### Managed Resources (MRs) — the actual cloud things (`MR/`)
- **`MR/bucket.yaml`** — an `S3Bucket`. `providerConfigRef.name: default` → authed via IRSA. Name is globally-unique (`...-300545976125-aps1`) because S3 names are global.
- **`MR/rds.yaml`** — an RDS `Instance` (`db.t3.micro`, MySQL 8.0.46). Reads its admin password from the `rds-master-password` Secret (`passwordSecretRef`) and **publishes a connection Secret** (`quorumify-db-conn`) via `writeConnectionSecretToRef`.

### Two Secrets in play
- **`rds-master-password`** — *input* you create; the admin password Crossplane feeds to RDS.
- **`quorumify-db-conn`** — *output* Crossplane creates after the DB is ready, merging your `username`/`password` with AWS's `host`/`port`/`endpoint`. A developer mounts this one Secret and gets a working DSN.

### How it all chains together
```
DeploymentRuntimeConfig ──► Provider pod's ServiceAccount  (fixed name + role-arn annotation)
            │                         │
            │            EKS pod-identity webhook injects AWS_ROLE_ARN + token file
            ▼                         ▼
       ProviderConfig (source: IRSA) ──► provider calls AssumeRoleWithWebIdentity
            ▲                         │
   MR (Bucket / RDS Instance) ────────┘  → short-lived creds → resource created in AWS
   (providerConfigRef: default)
```

---

## 3. IRSA — the deepest interview topic of the day

**Problem it solves:** a pod needs to call AWS without baking long-lived access keys into a Secret (leak-forever, manual rotation, shared blast radius).

**IRSA = IAM Roles for Service Accounts:** a K8s ServiceAccount assumes an IAM role and gets **short-lived, auto-rotated** creds. The trust bridge is **OIDC**.

**The five pieces and the flow:**
1. **OIDC provider** — every EKS cluster publishes an OIDC issuer; `eksctl --with-oidc` registers it in IAM as a trusted identity provider. *Without this, nothing works.*
2. **ServiceAccount** — the pod's K8s identity; EKS mints a signed JWT for it.
3. **IAM role + trust policy** — the trust policy says *"assumable via this OIDC provider, but only for `sub = system:serviceaccount:<ns>:<sa>`"*. That `sub` condition scopes the role to **one** ServiceAccount.
4. **Pod Identity webhook** — sees the SA annotation `eks.amazonaws.com/role-arn` and injects `AWS_ROLE_ARN` + `AWS_WEB_IDENTITY_TOKEN_FILE` into the pod.
5. **`AssumeRoleWithWebIdentity`** — the AWS SDK trades the JWT for temp creds; STS verifies the signature against the OIDC provider and checks the `sub`.

**Crossplane-specific wrinkle:** the provider pod runs under a SA Crossplane generates. To wire IRSA you use a **`DeploymentRuntimeConfig`** to give the provider a *fixed* SA name + the role annotation, then a **`ProviderConfig` with `source: IRSA`**. Each provider (s3, rds) = its own SA = its own IAM role.

**vs EKS Pod Identity (newer):** does the same job without a per-cluster OIDC provider or trust-policy juggling — you create a "pod identity association" (SA→role). IRSA is still the most common in the wild and the thing interviews ask about; knowing Pod Identity *exists and why* is a bonus point.

**Proof IRSA works** (what I verified): the provider pod had `AWS_ROLE_ARN` + `AWS_WEB_IDENTITY_TOKEN_FILE` injected, and S3/RDS API calls succeeded with no `AccessDenied`.

---

## 4. The debugging playbook (trace down the chain)

`XR → (Composition + Function) → MR → AWS`. Failures travel down it.

```bash
kubectl get managed                     # are the MRs even created? SYNCED/READY?
kubectl describe <mr>                    # Status.Conditions — the real AWS error lives here
kubectl get appdatabase -n <ns>         # the XR — Synced/Ready?
kubectl describe appdatabase ... -n <ns># Composition / function errors
kubectl describe instance.rds... <name> # MR's own conditions
```
**Rule:** `SYNCED=False` → Crossplane couldn't talk to AWS / bad params (read the condition). `READY=False` while `SYNCED=True` → AWS is still building it (just wait).

---

## 5. Real gotchas I hit today (these make great "tell me about a time you debugged…" stories)

1. **S3 `301 MovedPermanently` on HeadBucket** → the bucket name already exists **in another AWS account**. **S3 bucket names are globally unique.** `crossplane-bucket-example` was taken. Fix: a globally unique name (account-id + region suffix).
2. **RDS `InvalidParameterCombination: Cannot find version 8.4.3`** → the engine error surfaced at the **AWS API**, shown in `Status.Conditions`, *after* IRSA auth succeeded. Lesson: a 400-param error ≠ an auth error. Fix: `aws rds describe-db-engine-versions` → used `8.0.46`.
3. **Pod couldn't reach RDS** → RDS landed in the **default VPC** (no `dbSubnetGroupName`), EKS was in the eksctl VPC. **Different VPCs, no peering → timeout.** The #1 real-world "app can't reach DB" cause. Prod fix: put RDS in the app's VPC via a `DBSubnetGroup` + SG allowing the cluster. Test fix I used: `publiclyAccessible: true` + open SG 3306 (test-only, torn down after).
4. **`dbName` can't be added post-creation** → had to `CREATE DATABASE quorumify` manually (via a throwaway `mysql:8` pod, which also proved pod→RDS connectivity) because the app's JDBC URL needs the schema to exist.
5. **Connection Secret merges two sources** → Crossplane's `quorumify-db-conn` Secret combined the `username`/`password` *I* supplied with the `host`/`port`/`endpoint` *AWS* generated. That's the platform-eng payoff: a dev mounts one Secret and gets a working DSN, never touching the console.

**End-to-end win:** the Spring app booted in EKS, HikariCP connected to the Crossplane-provisioned RDS, and Hibernate created its tables in the DB. Full stack proven: **app → IRSA → Crossplane → RDS.**

---

## 6. Crossplane **v2** differences (don't get caught using deprecated syntax)

I'm on **Crossplane v2.3.3**. Two things changed vs older tutorials/the 60-day plan:
- **Compositions require Pipeline mode + a Function.** Inline `spec.resources` patch-and-transform was **removed**. You must install e.g. `function-patch-and-transform` and write `spec.mode: Pipeline`.
- **Claims are gone.** Instead the XRD has **`scope: Namespaced`**, so the **Composite Resource itself is namespaced** — a developer creates the XR directly in their namespace. The namespaced XR *is* the self-service object Claims used to be. XRD API is `apiextensions.crossplane.io/v2`.

---

## 7. Task 6 — Crossplane + ArgoCD (GitOps for infra) — *concept only, skipped hands-on*

- Crossplane resources are **just K8s manifests**, so ArgoCD manages XRD/Composition/XR like any Deployment → **GitOps for cloud infrastructure**.
- Result: **two stacked reconcile loops** — ArgoCD (Git→cluster) + Crossplane (cluster→AWS). Change a tag in Git → ArgoCD syncs → Crossplane reconciles → AWS updates.
- **Gotcha:** ArgoCD marks a Crossplane resource Healthy as soon as the object exists, *before AWS finishes*. Add **custom health checks** (Lua in `argocd-cm`) that wait for the MR's `Ready=True`. Also watch Helm-hook vs ArgoCD-sync-hook collisions (Day 3 topic).

---

## 8. Interview Q&A (rehearse out loud)

- **MR vs XR?** MR = one cloud resource, provider-defined. XR = your custom abstraction that composes many MRs.
- **Someone deletes an S3 bucket in the console — what happens?** Crossplane's reconcile loop detects the drift and **recreates it within ~60s**. No pipeline trigger.
- **Dev needs a DB but has no cluster-admin?** Platform team publishes an XRD + Composition; dev creates a **namespaced XR** (v2) in their own namespace. Self-service, least privilege.
- **Crossplane drift detection vs Terraform's?** Crossplane is **continuous** (always reconciling); Terraform only sees drift on the next `plan`.
- **Composition isn't creating the RDS — first commands?** `kubectl get managed` (did the MR appear?) → `kubectl describe` the XR (Composition/function errors) → `kubectl describe` the MR (AWS error in conditions).
- **How does IRSA avoid static keys?** OIDC trust + `AssumeRoleWithWebIdentity` → short-lived, auto-rotated, per-ServiceAccount-scoped creds. Walk the 5 pieces (§3).
- **Why did my new ProviderConfig/MR fail with a 301?** S3 global name collision — bucket names are unique across all of AWS.

---

## 9. Cleanup checklist (cost discipline)

- [x] `kubectl delete` all MRs / XRs (RDS especially — bills by the second)
- [x] Verify in AWS: `aws rds describe-db-instances`, `aws s3 ls`
- [x] Revoke any test SG rules (closed 3306 → 0.0.0.0/0)
- [x] **`eksctl delete cluster --name crossplane-lab --region ap-south-1`** ← EKS control plane is $0.10/hr even when idle

**Golden rule learned:** never leave RDS or an EKS cluster running overnight. Provision → verify → **delete immediately**.

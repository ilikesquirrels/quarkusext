# Quarkus Extensions Guide: Jib, Kubernetes & Helm

> **Purpose:** This document explains what the `quarkus-jib`, `quarkus-kubernetes`, and `quarkus-helm` extensions do, their trade-offs, and which `application.properties` entries are safe to remove if you are not using the full build-time deployment pipeline.

---

## Table of Contents

1. [How the three extensions work together](#1-how-the-three-extensions-work-together)
2. [quarkus-jib](#2-quarkus-jib)
3. [quarkus-kubernetes](#3-quarkus-kubernetes)
4. [quarkus-helm](#4-quarkus-helm)
5. [Properties that are safe to remove](#5-properties-that-are-safe-to-remove)
6. [Properties you should keep](#6-properties-you-should-keep)
7. [Decision guide: should we keep all three extensions?](#7-decision-guide-should-we-keep-all-three-extensions)

---

## 1. How the three extensions work together

These three extensions form a **build-time pipeline**. None of them affect your application's runtime behavior — they only influence what gets generated or pushed when you run `mvn package` or `./mvnw package`.

```
mvn package
     │
     ├─► quarkus-kubernetes  ──► generates kubernetes.yml / openshift.yml in target/kubernetes/
     │
     ├─► quarkus-helm        ──► wraps those manifests into a Helm chart in target/helm/
     │
     └─► quarkus-jib         ──► builds & (optionally) pushes a container image, no Docker daemon needed
```

The key insight: **all configuration for these extensions is build-time only**. If a property is only consumed during `mvn package`, it has zero effect at runtime and can safely live in a build profile or be removed if unused.

---

## 2. quarkus-jib

### What it does

[Jib](https://github.com/GoogleContainerTools/jib) is a Google library that builds optimized OCI/Docker container images **without requiring a Docker daemon**. The Quarkus extension integrates Jib into the Maven build so that `mvn package -Dquarkus.container-image.build=true` produces a ready-to-push image.

### Pros

| Pro | Detail |
|-----|--------|
| **No Docker daemon needed** | Works in CI environments (GitHub Actions, GitLab CI, etc.) where Docker-in-Docker is unavailable or slow |
| **Reproducible, layered images** | Jib separates dependencies, resources, and classes into distinct layers, so rebuilds only push changed layers |
| **Faster pushes** | Because layers are cached and reused, image pushes are significantly faster than a full `docker build` |
| **Secure by default** | Uses `distroless` or `ubi-minimal` base images with no shell, reducing attack surface |
| **No Dockerfile to maintain** | Image structure is derived from the build, not a hand-written Dockerfile |

### Cons

| Con | Detail |
|-----|--------|
| **Less control over image structure** | Advanced Dockerfile tricks (multi-stage builds, custom entrypoints, etc.) require workarounds |
| **Base image choice is limited** | You configure a base image via properties, but it is not as flexible as a Dockerfile |
| **Image is not built unless you opt in** | You must pass `-Dquarkus.container-image.build=true` or set it in properties — easy to forget |
| **Push credentials in properties** | Registry username/password are often placed in `application.properties`, which is a security risk if the file is committed |
| **Adds build time** | Even without pushing, Jib adds noticeable time to the Maven build |

### Key properties

```properties
# Whether to build the image at all during mvn package
quarkus.container-image.build=true

# Whether to push after building
quarkus.container-image.push=false

# Image coordinates
quarkus.container-image.registry=registry.example.com
quarkus.container-image.group=my-team
quarkus.container-image.name=my-app
quarkus.container-image.tag=1.0.0

# Credentials (prefer env vars or CI secrets instead)
quarkus.container-image.username=myuser
quarkus.container-image.password=secret

# Base image (Jib-specific)
quarkus.jib.base-jvm-image=registry.access.redhat.com/ubi8/openjdk-21
```

---

## 3. quarkus-kubernetes

### What it does

This extension generates Kubernetes manifests (Deployment, Service, ConfigMap, Ingress, etc.) at build time, derived from your `application.properties`. The output lands in `target/kubernetes/kubernetes.yml`. You can apply it with `kubectl apply -f target/kubernetes/kubernetes.yml`.

### Pros

| Pro | Detail |
|-----|--------|
| **Manifests stay in sync with code** | Image name, environment variables, ports, and resource limits are derived from the same build that produces the artifact |
| **No separate YAML files to maintain** | Reduces drift between what the app expects and what is deployed |
| **Supports many resource types** | Deployments, Services, Ingresses, ConfigMaps, Secrets, HPA, ServiceAccounts, RBAC, etc. |
| **OpenShift support built-in** | Switch `quarkus.kubernetes.deployment-target=openshift` to get DeploymentConfig and Route instead |
| **Integrates with Helm** | The generated YAML is what `quarkus-helm` wraps into a chart |

### Cons

| Con | Detail |
|-----|--------|
| **Property explosion** | Nearly every field in a Kubernetes manifest has a corresponding property. It is easy to accumulate dozens of properties that duplicate cluster-level configuration |
| **Opinionated defaults** | Default values are injected into the generated YAML even when not set, which can surprise teams expecting minimal manifests |
| **Not a replacement for GitOps** | The manifests are generated into `target/` which is gitignored by default — teams need a strategy to capture and version them |
| **Limited templating** | Unlike Helm, there are no conditionals or loops; advanced manifest patterns require post-processing |
| **Namespace/environment drift** | If different environments need different manifests, you need multiple build profiles, which gets complicated |

### Key properties

```properties
# Target platform (kubernetes, openshift, knative, etc.)
quarkus.kubernetes.deployment-target=kubernetes

# Replica count
quarkus.kubernetes.replicas=1

# Service type
quarkus.kubernetes.service-type=ClusterIP

# Resource limits and requests
quarkus.kubernetes.resources.limits.cpu=500m
quarkus.kubernetes.resources.limits.memory=256Mi
quarkus.kubernetes.resources.requests.cpu=250m
quarkus.kubernetes.resources.requests.memory=128Mi

# Liveness / readiness probes
quarkus.kubernetes.liveness-probe.http-action-path=/q/health/live
quarkus.kubernetes.readiness-probe.http-action-path=/q/health/ready

# Labels and annotations
quarkus.kubernetes.labels."app.kubernetes.io/part-of"=my-platform
quarkus.kubernetes.annotations."prometheus.io/scrape"=true

# Environment variables injected into the Pod
quarkus.kubernetes.env.vars.JAVA_OPTS=-Xmx200m
quarkus.kubernetes.env.secrets=my-app-secret

# Ingress
quarkus.kubernetes.ingress.expose=true
quarkus.kubernetes.ingress.host=my-app.example.com
```

---

## 4. quarkus-helm

### What it does

This extension wraps the Kubernetes manifests generated by `quarkus-kubernetes` into a [Helm](https://helm.sh/) chart, output to `target/helm/`. The chart can be packaged, versioned, and deployed with `helm install` / `helm upgrade`.

### Pros

| Pro | Detail |
|-----|--------|
| **Helm-native deployment** | Teams already using Helm can treat the app as a first-class chart without writing one by hand |
| **Parameterisation via `values.yaml`** | Jib image tag, replica count, resource limits, etc. become Helm values — easy to override per environment |
| **Chart versioning** | Charts can be published to a Helm registry (OCI or classic) for reproducible deploys |
| **Works with ArgoCD / Flux** | GitOps tools that understand Helm can pick up the generated chart |
| **Reduces boilerplate** | No need to hand-write `templates/deployment.yaml`, `templates/service.yaml`, etc. |

### Cons

| Con | Detail |
|-----|--------|
| **Two layers of abstraction** | `application.properties` → Kubernetes manifests → Helm chart. Debugging requires understanding all three layers |
| **Generated charts are hard to customise** | Helm's full power (named templates, helpers, complex conditionals) is not available in the generated chart |
| **values.yaml is auto-generated** | If you edit it manually and then rebuild, your changes are overwritten |
| **Not useful without Helm** | If your cluster uses raw `kubectl apply` or Kustomize, this extension adds complexity with no benefit |
| **Output in `target/`** | The chart is regenerated on every build; teams need a CI step to extract and publish it |

### Key properties

```properties
# Chart name and version
quarkus.helm.name=my-app
quarkus.helm.version=1.0.0
quarkus.helm.description=My application Helm chart

# Repository for pushing the chart
quarkus.helm.repository.push=true
quarkus.helm.repository.url=oci://registry.example.com/charts

# Which values to expose in values.yaml
quarkus.helm.values."replicas".value=1
quarkus.helm.values."image.tag".value=latest
```

---

## 5. Properties that are safe to remove

The following categories of properties are **build-time only** and have **no runtime effect**. They are safe to remove (or move to a dedicated build profile) if you are not actively using the corresponding feature.

### 5.1 Image build & push (quarkus-jib)

These properties do nothing unless `-Dquarkus.container-image.build=true` is passed or `build=true` is set. If your CI pipeline passes this flag via command line, you do not need these in `application.properties` at all.

```properties
# Safe to remove if image is not built during development
quarkus.container-image.build=false
quarkus.container-image.push=false

# Safe to move to CI secrets / pipeline variables
quarkus.container-image.username=...
quarkus.container-image.password=...

# Safe to remove if using default (project artifactId)
quarkus.container-image.name=my-app

# Safe to remove if tag is set via CLI (-Dquarkus.container-image.tag=...)
quarkus.container-image.tag=latest
```

> ⚠️ **Security note:** `username` and `password` should never be in a committed `application.properties`. Move them to environment variables (`QUARKUS_CONTAINER_IMAGE_USERNAME`, `QUARKUS_CONTAINER_IMAGE_PASSWORD`) or CI secrets.

### 5.2 Kubernetes manifest generation (quarkus-kubernetes)

These only affect the YAML written to `target/kubernetes/`. They are completely ignored at runtime.

```properties
# Safe to remove if you don't use the generated manifests
quarkus.kubernetes.replicas=1           # default is already 1
quarkus.kubernetes.service-type=ClusterIP  # default is already ClusterIP
quarkus.kubernetes.deployment-target=kubernetes  # default is already kubernetes

# Safe to remove if no Ingress is needed
quarkus.kubernetes.ingress.expose=false
quarkus.kubernetes.ingress.host=...

# Safe to remove if managed at the cluster level (e.g. via LimitRange)
quarkus.kubernetes.resources.limits.cpu=...
quarkus.kubernetes.resources.limits.memory=...
quarkus.kubernetes.resources.requests.cpu=...
quarkus.kubernetes.resources.requests.memory=...

# Safe to remove if probes are handled by the cluster or a service mesh
quarkus.kubernetes.liveness-probe.http-action-path=...
quarkus.kubernetes.readiness-probe.http-action-path=...
```

### 5.3 Helm chart generation (quarkus-helm)

Everything under `quarkus.helm.*` is build-time only. If you are not publishing or using Helm charts, this entire block can be removed.

```properties
# Entire block safe to remove if not using Helm
quarkus.helm.name=...
quarkus.helm.version=...
quarkus.helm.description=...
quarkus.helm.repository.push=...
quarkus.helm.repository.url=...
quarkus.helm.values.*=...
```

---

## 6. Properties you should keep

These properties look similar but do have runtime or important build implications.

| Property | Why keep it |
|----------|-------------|
| `quarkus.container-image.registry` | Affects image resolution at runtime if you use image pull policies |
| `quarkus.kubernetes.env.vars.*` | Environment variables injected into the Pod — runtime config |
| `quarkus.kubernetes.env.secrets.*` | Secrets mounted into the Pod — runtime config |
| `quarkus.kubernetes.service-account` | Affects Pod RBAC at runtime |
| `quarkus.kubernetes.namespace` | Needed if your tooling reads this for `kubectl apply` targeting |
| `quarkus.container-image.group` | Used to construct the full image reference; needed for consistency |

---

## 7. Decision guide: should we keep all three extensions?

Use this table to decide whether each extension is earning its complexity:

| Question | If YES | If NO |
|----------|--------|-------|
| Does CI build the container image using `mvn package`? | Keep `quarkus-jib` | Consider removing it; use a plain Dockerfile instead |
| Do you apply `target/kubernetes/*.yml` directly to the cluster? | Keep `quarkus-kubernetes` | Consider removing it; maintain hand-crafted manifests or use Kustomize |
| Do you deploy via `helm install` / `helm upgrade`? | Keep `quarkus-helm` | Remove it; it adds a layer of indirection with no benefit |
| Do different environments need different manifests? | Use build profiles, not `application.properties` entries | — |
| Is anyone on the team familiar with how the chain works? | Document it (this file!) | Consider simplifying to fewer extensions |

### Recommended minimal `application.properties` (build-time section)

If you keep all three extensions but want a clean properties file, scope build-time config to a Maven profile in `pom.xml` or pass values via the CLI. Your `application.properties` should then contain **only** the properties that differ between environments or that have actual runtime significance:

```properties
# application.properties — runtime + meaningful build config only

# Image coordinates (needed for consistency across build and deploy)
quarkus.container-image.registry=registry.example.com
quarkus.container-image.group=my-team
quarkus.container-image.name=my-app

# Pod environment (runtime effect)
quarkus.kubernetes.env.secrets=my-app-secret

# Helm chart identity
quarkus.helm.name=my-app
quarkus.helm.version=${project.version}
```

Everything else should be either a CLI flag, a CI/CD variable, or simply left at its default.

---

*Generated for internal team reference. Share freely within the team.*

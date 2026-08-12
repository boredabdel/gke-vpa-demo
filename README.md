# GKE VPA CPU Startup Boost Demo

This repository demonstrates how to use **GKE Vertical Pod Autoscaler (VPA) CPU Startup Boost** to accelerate the startup time of a Java Spring Boot application without over-provisioning steady-state resources.

## Overview

Resource-intensive workloads (such as JVM-based applications) often require significant CPU during initialization for tasks like class loading, JIT compilation, and dependency injection. 

Using **GKE CPU Startup Boost**, GKE temporarily increases CPU requests during the Pod's startup phase and then automatically scales the CPU requests back to baseline levels in-place (without restarting the container) using Kubernetes **In-Place Pod Resizing (IPPR)**.

## Prerequisites

* **GKE Cluster**: Version `1.36.0-gke.4447000` or later (Autopilot or Standard).
  * If using **GKE Standard**, ensure VPA is enabled on the cluster.
  * If using **GKE Autopilot**, VPA is enabled by default.
* `kubectl` connected to your cluster.

## Repository Structure

```text
.
├── application.yaml                 # Spring Boot configuration (injected via ConfigMap)
├── kustomization.yaml               # Kustomize manifest to deploy all demo resources
├── vpa.yaml                         # VerticalPodAutoscaler with CPU Startup Boost
└── examples/
    ├── spring-demo-app.yaml         # Deployment, ServiceAccount, and Service
    └── spring-demo-app-gmp.yaml     # (Optional) Google Managed Prometheus PodMonitoring
```

## How It Works

1. **Base Configuration** ([examples/spring-demo-app.yaml](examples/spring-demo-app.yaml)):
   * CPU request: `250m`
   * Memory request: `1Gi`
2. **Startup Boost Policy** ([vpa.yaml](vpa.yaml)):
   * Multiplier `factor: 4` boosts the CPU request to `1000m` (1 vCPU) during startup.
   * `durationSeconds: 120` keeps the boost active for 2 minutes before scaling back.
   * `updateMode: "Off"` ensures VPA is used exclusively for startup boost without changing steady-state recommendations.

> [!NOTE]
> **Autopilot Memory Ratio**: GKE Autopilot enforces a maximum ratio of 1 vCPU per 1 GiB memory for general-purpose workloads. Setting the base memory request to `1Gi` allows the full 4x CPU boost (1 vCPU) without ratio capping.

## Quickstart

### 1. Deploy the Application

Deploy the manifests using Kustomize:

```sh
kubectl apply -k .
```

### 2. Monitor CPU In-Place Resizing

Watch the Pod resource requests and allocated resources in real time:

```sh
kubectl get pod -l app.kubernetes.io/name=spring-demo-app -w \
  -o custom-columns='TIME:.metadata.resourceVersion,NAME:.metadata.name,CPU_REQ:.spec.containers[*].resources.requests.cpu,ALLOC_CPU:.status.containerStatuses[*].allocatedResources.cpu,STATUS:.status.phase'
```

* **During Startup**: `CPU_REQ` and `ALLOC_CPU` will be `1` (1000m).
* **After ~120s**: `CPU_REQ` and `ALLOC_CPU` will automatically drop to `250m` without restarting the Pod.

### 3. Verify Boost Annotations & Events

Check the startup boost annotation injected by the VPA admission webhook:

```sh
kubectl get pod -l app.kubernetes.io/name=spring-demo-app -o jsonpath='{.items[0].metadata.annotations.vpaCpuStartupBoost/spring-demo-app}'
```

Check for the downscaling event once the boost period finishes:

```sh
kubectl get events --field-selector reason=InPlaceResizedByVPA
```

## Cleanup

To remove all demo resources:

```sh
kubectl delete -k .
```

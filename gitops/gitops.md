## Overview
Welcome to the OpenShift GitOps Foundation Enablement Session. This module provides a comprehensive introduction to OpenShift container concepts, GitOps principles powered by ArgoCD, and a practical application of these skills by deploying the Red Hat Compliance Operator declaratively.


## Table of Contents
1. [Module 1: OpenShift 101](#module-1-openshift-101)
2. [Module 2: Introduction to GitOps and ArgoCD](#module-2-introduction-to-gitops-and-argocd)
3. [Module 3: Declarative Installation of the Compliance Operator](#module-3-declarative-installation-of-the-compliance-operator)
4. [Laboratory Exercise: Deploying Your First GitOps Application](#laboratory-exercise-deploying-your-first-gitops-application)
5. [Appendix: References & Resources](#appendix-references--resources)

---

## Module 1: OpenShift 101

### 1.1 What is Red Hat OpenShift?
Red Hat OpenShift is an enterprise-grade Kubernetes container platform. While Kubernetes excels at container orchestration, OpenShift provides a full-stack automated operation system managing everything from the underlying operating system (Red Hat Enterprise Linux CoreOS) to developer tooling, security integrations, and routing networks.

### 1.2 Core Architectural Concepts
To effectively manage infrastructure via GitOps, you must understand the foundational components of an OpenShift cluster:

* **Namespaces and Projects:** A Kubernetes `Namespace` provides isolation for resources. OpenShift wraps this in a `Project` custom resource, adding metadata like display names, descriptions, and integration with OpenShift's Role-Based Access Control (RBAC).
* **Pods:** The smallest deployable unit in Kubernetes, representing a single instance of a running process in your cluster.
* **Deployments & DeploymentConfigs:** Declarative configurations defining the desired state for application pods (e.g., number of replicas, container images, update strategies).
* **Services:** An abstract way to expose an application running on a set of Pods as a network service, providing internal load balancing.
* **Routes:** An OpenShift-specific resource that exposes a `Service` externally by giving it an externally reachable hostname (e.g., `myapp.apps.cluster.example.com`).
* **Operators:** A method of packaging, deploying, and managing a Kubernetes application. An Operator takes human operational knowledge and encodes it into software, extending the Kubernetes API using Custom Resource Definitions (CRDs).

---

## Module 2: Introduction to GitOps and ArgoCD

### 2.1 The Philosophy of GitOps
GitOps is an operational framework that takes DevOps best practices used for application development—such as version control, collaboration, compliance, and CI/CD—and applies them to infrastructure automation.

The four core principles of GitOps are:
1.  **Declarative System Definition:** The entire system (infrastructure, apps, configuration) is described declaratively (usually via YAML).
2.  **Git as the Single Source of Truth:** The desired state is version-controlled in a Git repository.
3.  **Automated State Approval & Pull:** Approved changes in Git are automatically pulled and applied to the cluster.
4.  **Continuous Reconciliation & Feedback Loop:** A software agent constantly compares the desired state (Git) with the actual state (Cluster) and automatically corrects any drift.

### 2.2 Red Hat OpenShift GitOps & ArgoCD
Red Hat OpenShift GitOps is an operator-backed solution utilizing **ArgoCD** as its declarative, GitOps continuous delivery engine. 

Key ArgoCD abstractions you need to know:
* **ArgoCD Application:** A Custom Resource (CR) that defines the relationship between a source Git repository (where the YAML files live) and a destination target Kubernetes cluster/namespace (where they should be deployed).
* **ArgoCD AppProject:** A logical grouping of Applications, providing a security boundary where you can restrict what repositories can be used, what clusters can be targeted, and what resource kinds can be deployed.
* **Synchronization (Sync):** The process of making the actual state match the desired state. This can be manual or automated.
* **Out of Sync / Drift:** When someone manually edits a resource in the cluster via the CLI or Web Console, it deviates from Git. ArgoCD flags this as `OutOfSync` and can be configured to self-heal (overwrite the manual change).

---

## Module 3: Declarative Installation of the Compliance Operator

Now we will apply our GitOps knowledge to deploy infrastructure software. The **Red Hat Compliance Operator** is an OpenShift operator that runs vulnerability and compliance scans (using OpenSCAP) against your cluster nodes and platform configuration to ensure compliance with standards like CIS Profiles, NIST SP 800-53, or PCI-DSS.

Instead of navigating the OpenShift Web Console or running manual CLI commands (as described in the [Red Hat Compliance Operator Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.9/html/security_and_compliance/compliance-operator#compliance-operator-installation)), we will structure this installation strictly as a GitOps pipeline.

### 3.1 Structure of the GitOps Repository
To implement this, your Git repository should mimic the following directory structure within your foundation workspace:

```text
gitops-foundation/
├── clusters/
│   └── management/
│       └── apps/
│           └── compliance-operator-app.yaml
└── infrastructure/
    └── compliance-operator/
        ├── namespace.yaml
        ├── operator-group.yaml
        └── subscription.yaml
```

### 3.2 Component Manifests

1. Namespace Manifest (infrastructure/compliance-operator/namespace.yaml)
Creates the dedicated space where the operator and its scanning pods will execute.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-compliance
  labels:
    openshift.io/cluster-monitoring: "true"
```

2. OperatorGroup Manifest (infrastructure/compliance-operator/operator-group.yaml)
Scopes the multi-tenant operator to monitor the dedicated compliance namespace.

```yaml
apiVersion: [operators.coreos.com/v1](https://operators.coreos.com/v1)
kind: OperatorGroup
metadata:
  name: compliance-operator
  namespace: openshift-compliance
spec:
  targetNamespaces:
  - openshift-compliance
```

3. Subscription Manifest (infrastructure/compliance-operator/subscription.yaml)
Subscribes to the Red Hat catalog to fetch and maintain updates for the operator.


```yaml
apiVersion: [operators.coreos.com/v1alpha1](https://operators.coreos.com/v1alpha1)
kind: Subscription
metadata:
  name: compliance-operator
  namespace: openshift-compliance
spec:
  channel: "stable"
  installPlanApproval: Automatic
  name: compliance-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

3.3 The ArgoCD Application Manifest
To glue these pieces together, we create the ArgoCD Application resource. This resource tells OpenShift GitOps to look at your repository and spin up the Compliance Operator.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: compliance-operator
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: '[https://github.com/](https://github.com/)<your-org>/gitops-foundation.git'
    targetRevision: HEAD
    path: infrastructure/compliance-operator
  destination:
    server: '[https://kubernetes.default.svc](https://kubernetes.default.svc)'
    namespace: openshift-compliance
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```


## Laboratory Exercise: Deploying Your First GitOps Application

**Prerequisites**
* An active OpenShift Cluster (version 4.9+ recommended).

* The Red Hat OpenShift GitOps operator installed on the cluster.

* Cluster Administrator (cluster-admin) privileges.

* A personal or organizational Git repository containing the manifests detailed in Module 3.

1. Commit Manifests to Git
Clone your Git repository, create the folder structure under your foundation directory, paste the manifests above (updating the repoURL in the Application manifest), and push to your main branch.

2. Apply the ArgoCD Application
Log into your OpenShift cluster via CLI and apply the root application manifest:


```bash
oc apply -f clusters/management/apps/compliance-operator-app.yaml
```

3. Monitor in the ArgoCD Dashboard
Extract your ArgoCD admin password:

```bash
oc extract secret/openshift-gitops-cluster-cluster --to=- -n openshift-gitops
```

Retrieve the ArgoCD URL:

```bash
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'
```

Navigate to the dashboard. You should see the compliance-operator application moving from OutOfSync to Synced as it builds out the Namespace, OperatorGroup, and Subscription.

4. Verify Compliance Operator Health
Once synchronized, verify that the Custom Resource Definitions (CRDs) have been populated and that the manager pods are operational:

```bash
# Check custom resource availability
oc get crd | grep compliance

# Check operator pod health
oc get pods -n openshift-compliance
```

You should see pods matching compliance-operator-XXXXX and ocp-profile-parser-XXXXX in a Running status.

# Appendix: References & Resources
* [Red Hat Compliance Operator Official Installation Guide](https://docs.redhat.com/en/documentation/openshift_container_platform/4.9/html/security_and_compliance/compliance-operator#compliance-operator-installation) - Reference for manual parameters, catalog names, and scan definitions.
* [OpenShift GitOps Documentation](https://www.google.com/search?q=https://docs.redhat.com/en/documentation/openshift_gitops/) - Continuous delivery engine architecture.



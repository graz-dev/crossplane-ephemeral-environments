# Ephemeral PostgreSQL Clusters with Crossplane and kube-green

Welcome to this hands-on tutorial! Together, we will explore how to dynamically provision and manage the lifecycle of a PostgreSQL cluster using the power of [Crossplane](https://www.crossplane.io/), [kube-green](https://kube-green.dev/) and [CloudNativePG](https://cloudnative-pg.io/).

The main goal is to create a reusable, on-demand PostgreSQL cluster definition that can be provisioned whenever needed and automatically scaled down or "put to sleep" during inactive hours to save resources and costs. This is a common pattern for creating ephemeral development or testing environments.

We will cover:
- Setting up a local control plane with **Kind**.
- Installing and configuring **Crossplane** and its Kubernetes providers.
- Defining our own cloud infrastructure abstractions using **Composition**.
- Provisioning a complete PostgreSQL cluster.
- Integrating **kube-green** to schedule hibernation for our cluster, scaling down the pods during off-hours.

Let's get started!

---

## Phase 1: Prepare the Workspace

First, we need to set up our local environment with all the necessary tools. We'll use a local Kubernetes cluster created with `kind` as our control plane where Crossplane will run.

1.  **Install Kind**
    [Kind](https://kind.sigs.k8s.io/) lets you run local Kubernetes clusters using Docker container "nodes". It's perfect for development and testing. If you're on macOS and use Homebrew, you can run:

    ```bash
    brew install kind
    ```

    *For other installation options, please refer to the [official Kind documentation](https://kind.sigs.k8s.io/docs/user/quick-start/#installing-with-a-package-manager).*

2.  **Install Helm**
    [Helm](https://helm.sh/) is the package manager for Kubernetes, which helps you manage complex applications. We'll use it to install Crossplane.

    ```bash
    brew install helm
    ```

    *For other installation options, see the [Helm installation guide](https://helm.sh/docs/intro/install/).*

3.  **Install kubectl**
    `kubectl` is the command-line tool for interacting with Kubernetes clusters.

    ```bash
    brew install kubectl
    ```
    
    *For other installation options, see the [kubectl installation guide](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/).*

4.  **Create a Local Cluster**
    Now, let's create our control plane cluster using Kind.

    ```bash
    kind create cluster --name ephemeral-environments-demo
    ```

5.  **Verify the Cluster**
    Check that your local cluster is up and running.

    ```bash
    kubectl get ns
    ```

    You should see a similar output:

    ```console
    NAME                 STATUS   AGE
    default              Active   4m52s
    kube-node-lease      Active   4m52s
    kube-public          Active   4m52s
    kube-system          Active   4m53s
    local-path-storage   Active   4m47s
    ```

---

## Phase 2: Install Crossplane and kube-green

With our local cluster ready, it's time to install the core components: Crossplane for infrastructure provisioning and kube-green for lifecycle management.

1.  **Add Crossplane Helm Repository**
    First, we add the official Helm chart repository for Crossplane.

    ```bash
    helm repo add crossplane-stable https://charts.crossplane.io/stable
    helm repo update
    ```

2.  **Install Crossplane**
    Now, we install Crossplane into its own namespace, `crossplane-system`.

    ```bash
    helm install crossplane --namespace crossplane-system --create-namespace crossplane-stable/crossplane
    ```

3.  **Verify Crossplane Installation**
    Let's make sure the Crossplane pods are running correctly.

    ```bash
    kubectl get pods -n crossplane-system
    ```

    The output should look like this (pod names may vary):

    ```console
    NAME                                       READY   STATUS    RESTARTS   AGE
    crossplane-67b976bbf4-hn9kk                1/1     Running   0          81s
    crossplane-rbac-manager-594757659d-dhr97   1/1     Running   0          81s
    ```

4.  **Install Cert-Manager**
    Cert-manager is a dependency for kube-green. It's used to manage certificates for webhooks.

    ```bash
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml
    ```

5.  **Install kube-green**
    Finally, we install kube-green, which will manage the sleep schedule for our cluster.

    ```bash
    kubectl apply -f https://github.com/kube-green/kube-green/releases/latest/download/kube-green.yaml
    ```

6.  **Verify kube-green Installation**
    Check that the kube-green controller is running.

    ```bash
    kubectl get pods -n kube-green
    ```
    
    You should see a running pod:
    
    ```console
    NAME                                             READY   STATUS    RESTARTS   AGE
    kube-green-controller-manager-6c677846bb-vxs64   1/1     Running   0          85s
    ```

---

## Phase 3: Configure Kubernetes Providers for Crossplane

Crossplane uses **Providers** to interact with external APIs like Kubernetes. We need to install and configure the specific providers for Kubernetes.

1.  **Install the Kubernetes Provider **
    This provider manages Kubernetes resources. You can find it here: [manifests/providers/provider-kubernetes.yaml](manifests/providers/provider-kubernetes.yaml)
    <details>
    <summary><code>manifests/providers/provider-kubernetes.yaml</code></summary>

    ```yaml
    apiVersion: pkg.crossplane.io/v1
    kind: Provider
    metadata:
      name: provider-kubernetes
    spec:
      package: xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.18.0
    ```
    </details>

    ```bash
    kubectl apply -f manifests/providers/provider-kubernetes.yaml
    ```

2.  **Create a ProviderConfig**
    The `ProviderConfig` tells the Kubernetes providers how to authenticate. It references the secret we just created. You can find it here: [manifests/providers/provider-kubernetes-config.yaml](manifests/providers/provider-kubernetes-config.yaml)
    <details>
    <summary><code>manifests/providers/provider-kubernetes-config.yaml</code></summary>

    ```yaml
    apiVersion: kubernetes.crossplane.io/v1alpha1
    kind: ProviderConfig
    metadata:
      name: kubernetes-in-cluster
    spec:
      credentials:
        source: InjectedIdentity
    ```
    </details>

    ```bash
    kubectl apply -f manifests/providers/provider-kubernetes-config.yaml
    ```

3.  **Install the CloudNativePG Operator**
    This operator manages PostgreSQL clusters.

    ```bash
    kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.26/releases/cnpg-1.26.0.yaml
    ```

4.  **Check the operator installation**

    ```bash
    kubectl rollout status deployment -n cnpg-system cnpg-controller-manager
    ```

    You should get this output:

    ```console
    deployment "cnpg-controller-manager" successfully rolled out
    ```

---

## Phase 4: Define a Reusable Infrastructure Abstraction

This is where the magic of Crossplane shines. We will define our own custom API for provisioning a "PostgresCluster". This involves creating a few key components:
- **CompositeResourceDefinition (XRD):** This defines the schema for our custom API—what inputs it accepts and what outputs it returns.
- **Composition:** This maps our custom API to the actual cloud resources that need to be created.

1.  **Create the PostgresCluster XRD (`xrd-postgrescluster.yaml`)**
    This defines an API to request a standard networking stack. You can find it here: [manifests/apis/xrd-postgrescluster.yaml](manifests/apis/xrd-postgrescluster.yaml)
    <details>
    <summary><code>manifests/apis/xrd-postgrescluster.yaml</code></summary>

    ```yaml
    apiVersion: apiextensions.crossplane.io/v1
    kind: CompositeResourceDefinition
    metadata:
      name: xpostgresclusters.k8s.crossplane.grazdev.io
    spec:
      group: k8s.crossplane.grazdev.io
      names:
        kind: XPostgresCluster
        plural: xpostgresclusters
      versions:
      - name: v1alpha1
        served: true
        referenceable: true
        schema:
          openAPIV3Schema:
            type: object
            properties:
              spec:
                type: object
                properties:
                  instances:
                    type: integer
                    description: "Number of PostgreSQL instances in the cluster."
                    default: 1
                  hibernation:
                    type: string
                    description: "Set to 'on' to hibernate the cluster, 'off' to wake it up."
                    default: "off"
                required:
                - instances
    ```
    </details>
    
    ```bash
    kubectl apply -f manifests/apis/xrd-postgrescluster.yaml
    ```
    
2.  **Create the PostgresCluster Composition (`composition-postgrescluster.yaml`)**
    This `Composition` map the abstract API to an `Object` resource of the crossplane kubernetes-provider that defines a CloudNativePG `Cluster`. You can find it here: [manifests/apis/composition-postgrescluster.yaml](manifests/apis/composition-postgrescluster.yaml)
    <details>
    <summary><code>manifests/apis/composition-postgrescluster.yaml</code></summary>

    ```yaml
    apiVersion: apiextensions.crossplane.io/v1
    kind: Composition
    metadata:
      name: postgrescluster.k8s.crossplane.grazdev.io
    spec:
      compositeTypeRef:
        apiVersion: k8s.crossplane.grazdev.io/v1alpha1
        kind: XPostgresCluster
      resources:
      - name: postgres-cluster-object
        base:
          apiVersion: kubernetes.crossplane.io/v1alpha2
          kind: Object
          spec:
            managementPolicies: ["*"]
            forProvider:
              manifest:
                apiVersion: postgresql.cnpg.io/v1
                kind: Cluster
                metadata:
                  namespace: default
                spec:
                  storage:
                    size: 1Gi
                  bootstrap:
                    initdb:
                      database: appdb
                      owner: appuser
            providerConfigRef:
              name: kubernetes-in-cluster
        patches:
        - fromFieldPath: "metadata.name"
          toFieldPath: "spec.forProvider.manifest.metadata.name"
        - fromFieldPath: "spec.instances"
          toFieldPath: "spec.forProvider.manifest.spec.instances"
        - type: FromCompositeFieldPath
          fromFieldPath: spec.hibernation
          toFieldPath: spec.forProvider.manifest.metadata.annotations[cnpg.io/hibernation]
          policy:
            fromFieldPath: Optional
    ```
    </details>
    
    ```bash
    kubectl apply -f manifests/apis/composition-postgrescluster.yaml
    ```

3.  **Give the permission to the kubernetes-provider service account to act as a cluster-admin**

    ```bash
    SA=$(kubectl -n crossplane-system get sa -o name | grep provider-kubernetes | sed -e 's|serviceaccount\/|crossplane-system:|g')
    ```

    Then run:

    ```bash
    kubectl create clusterrolebinding provider-kubernetes-admin-binding --clusterrole cluster-admin --serviceaccount="${SA}"
    ```

---

## Phase 5: Provision and Manage the PostgreSQL Cluster

Now that we have defined our custom `PostgresCluster` API, let's use it to provision a real cluster.

1.  **Create a Claim**
    A `Claim` is a request for a resource defined by our XRD. This simple YAML is all a user needs to provision a complete PostgreSQL cluster. You can find it here: [manifests/claims/claim-postgrescluster.yaml](manifests/claims/claim-postgrescluster.yaml)
    <details>
    <summary><code>manifests/claims/claim-postgrescluster.yaml</code></summary>

    ```yaml
    apiVersion: k8s.crossplane.grazdev.io/v1alpha1
    kind: XPostgresCluster
    metadata:
      name: my-production-db
      namespace: default
    spec:
      instances: 1
      hibernation: "off"
    ```
    </details>
    
    ```bash
    kubectl apply -f manifests/claims/claim-postgrescluster.yaml
    ```

2.  **Monitor Provisioning**
    Provisioning a PostgreSQL cluster is fast. You can monitor the status with the following command:
    
    ```bash
    kubectl get xpostgrescluster my-production-db
    ```
    
    Wait until `SYNCED` and `READY` are both `True`.

    ```console
    NAME               SYNCED   READY   COMPOSITION                                 AGE
    my-production-db   True     True    postgrescluster.k8s.crossplane.grazdev.io   3m50s
    ```

3.  **Access the New PostgreSQL Cluster**
    Once ready, you can check the status of the CloudNativePG cluster:
    
    ```bash
    kubectl get clusters.postgresql.cnpg.io -n default my-production-db
    ```

    You should see the cluster in a healthy state:

    ```console
    NAME               AGE   INSTANCES   READY   STATUS                     PRIMARY
    my-production-db   43s   1           1       Cluster in healthy state   my-production-db-1
    ```

    And the database pods:

    ```bash
    kubectl get pods -n default -l cnpg.io/cluster=my-production-db
    ```

    You should see the running pod:

    ```console
    NAME                 READY   STATUS    RESTARTS   AGE
    my-production-db-1   1/1     Running   0          65s
    ```

---

## Phase 6: Implement Hibernation with kube-green

Now, let's configure kube-green to automatically scale down our cluster's node pool to save costs during inactive hours.

1.  **Grant kube-green Permissions**
    We need to give kube-green permission to modify our custom `XPostgresCluster` resources. You can find it here: [manifests/kube-green/kube-green-postgrescluster.yaml](manifests/kube-green/kube-green-postgrescluster.yaml)
    <details>
    <summary><code>manifests/kube-green/kube-green-postgrescluster.yaml</code></summary>

    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: kube-green-xpostgrescluster-patcher
    rules:
    - apiGroups:
      - "k8s.crossplane.grazdev.io"
      resources:
      - "xpostgresclusters"
      verbs:
      - "get"
      - "list"
      - "watch"
      - "patch"
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: kube-green-patch-xpostgrescluster
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: kube-green-xpostgrescluster-patcher
    subjects:
    - kind: ServiceAccount
      name: kube-green-controller-manager
      namespace: kube-green
    ```
    </details>
    
    ```bash
    kubectl apply -f manifests/kube-green/kube-green-postgrescluster.yaml
    ```

2.  **Create a SleepInfo Resource**
    The `SleepInfo` resource tells kube-green when to sleep and wake up, and what to patch. In this case, we patch the `hibernation` parameter of our `XPostgresCluster` to scale it down. You can find it here: [manifests/kube-green/sleepinfo.yaml](manifests/kube-green/sleepinfo.yaml)
    <details>
    <summary><code>manifests/kube-green/sleepinfo.yaml</code></summary>

    ```yaml
    apiVersion: kube-green.com/v1alpha1
    kind: SleepInfo
    metadata:
      name: sleep-schedule-for-postgres
      namespace: default
    spec:
      weekdays: "*"
      timeZone: "Europe/Rome"
      sleepAt: "18:47" # Adjust to a few minutes from now for testing
      wakeUpAt: "18:49" # Adjust to a few minutes from now for testing
      patches:
      - target:
          group: k8s.crossplane.grazdev.io
          kind: XPostgresCluster
        patch: |
          - op: replace
            path: /spec/hibernation
            value: "on"
    ```
    </details>
    
    ```bash
    kubectl apply -f manifests/kube-green/sleepinfo.yaml
    ```

3.  **Verify Hibernation (Sleep)**
    At the `sleepAt` time, kube-green will patch our `XPostgresCluster` resource, and Crossplane will hibernate the PostgreSQL cluster.
    
    You can watch the `XPostgresCluster` resource to see the change:

    ```bash
    watch "kubectl get xpostgrescluster my-production-db -n default -o yaml"
    ```
    
    You should see the `hibernation` property change to `on`.

    Then, check the pods:

    ```bash
    watch kubectl get pods -n default -l cnpg.io/cluster=my-production-db
    ```

    You should see no pods running:

    ```console
    No resources found in default namespace.
    ```

4.  **Verify Hibernation (Wake Up)**
    At the `wakeUpAt` time, kube-green will revert the patch, and Crossplane will wake up the PostgreSQL cluster.

    You will see the pod being recreated:
    
    ```bash
    watch kubectl get pods -n default -l cnpg.io/cluster=my-production-db
    ```
    
    And the cluster will return to a healthy state.

Congratulations! You have successfully created a self-service, ephemeral PostgreSQL cluster with scheduled hibernation.
# Ephemeral EKS Clusters with Crossplane and kube-green

Welcome to this hands-on tutorial! Together, we will explore how to dynamically provision and manage the lifecycle of an Amazon EKS (Elastic Kubernetes Service) cluster using the power of [Crossplane](https://www.crossplane.io/) and [kube-green](https://kube-green.dev/).

The main goal is to create a reusable, on-demand EKS cluster definition that can be provisioned whenever needed and automatically scaled down or "put to sleep" during inactive hours to save resources and costs. This is a common pattern for creating ephemeral development or testing environments.

We will cover:
- Setting up a local control plane with **Kind**.
- Installing and configuring **Crossplane** and its AWS providers.
- Defining our own cloud infrastructure abstractions using **Composition**.
- Provisioning a complete EKS cluster with its networking stack.
- Integrating **kube-green** to schedule hibernation for our cluster, scaling down the nodes during off-hours.

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

## Phase 3: Configure AWS Providers for Crossplane

Crossplane uses **Providers** to interact with external APIs like AWS. We need to install and configure the specific providers for EKS, EC2, and IAM.

1.  **Install the AWS EKS Provider**
    This provider manages EKS resources.
    <details>
    <summary><code>provider-aws-eks.yaml</code></summary>

    ```yaml
    # provider-aws-eks.yaml
    apiVersion: pkg.crossplane.io/v1
    kind: Provider
    metadata:
        name: provider-aws-eks
    spec:
        package: xpkg.upbound.io/upbound/provider-aws-eks:v1.2.1
    ```
    </details>

    ```bash
    kubectl apply -f provider-aws-eks.yaml
    ```

2.  **Install the AWS EC2 Provider**
    This provider manages networking resources like VPCs and Subnets.
    <details>
    <summary><code>provider-aws-ec2.yaml</code></summary>

    ```yaml
    # provider-aws-ec2.yaml
    apiVersion: pkg.crossplane.io/v1
    kind: Provider
    metadata:
        name: provider-aws-ec2
    spec:
        package: xpkg.upbound.io/upbound/provider-aws-ec2:v1.2.1
    ```
    </details>

    ```bash
    kubectl apply -f provider-aws-ec2.yaml
    ```

3.  **Install the AWS IAM Provider**
    This provider manages IAM roles and policies required by the EKS cluster.
    <details>
    <summary><code>provider-aws-iam.yaml</code></summary>

    ```yaml
    # provider-aws-iam.yaml
    apiVersion: pkg.crossplane.io/v1
    kind: Provider
    metadata:
        name: provider-aws-iam
    spec:
        package: xpkg.upbound.io/upbound/provider-aws-iam:v1.2.1
    ```
    </details>
    
    ```bash
    kubectl apply -f provider-aws-iam.yaml
    ```
    
4.  **Verify Provider Installation**
    Check that all providers are installed and healthy.

    ```bash
    kubectl get providers.pkg.crossplane.io
    ```

    The output should show all providers as `INSTALLED` and `HEALTHY`.

    ```console
    NAME                          INSTALLED   HEALTHY   PACKAGE                                               AGE
    provider-aws-ec2              True        True      xpkg.upbound.io/upbound/provider-aws-ec2:v1.2.1       3m3s
    provider-aws-eks              True        True      xpkg.upbound.io/upbound/provider-aws-eks:v1.2.1       6m25s
    provider-aws-iam              True        True      xpkg.upbound.io/upbound/provider-aws-iam:v1.2.1       19s
    ```

5.  **Configure AWS Credentials**
    Crossplane needs AWS credentials to create resources on your behalf. Create a file named `aws-credentials.txt` with your credentials.

    ```ini
    [default]
    aws_access_key_id = <your-access-key-id>
    aws_secret_access_key = <your-secret-access-key>
    ```
    
    Now, create a Kubernetes secret from this file.

    ```bash
    kubectl create secret generic aws-secret -n crossplane-system --from-file=creds=./aws-credentials.txt
    ```

6.  **Create a ProviderConfig**
    The `ProviderConfig` tells the AWS providers how to authenticate. It references the secret we just created.
    <details>
    <summary><code>provider-aws-config.yaml</code></summary>

    ```yaml
    # provider-aws-config.yaml
    apiVersion: aws.upbound.io/v1beta1
    kind: ProviderConfig
    metadata:
        name: default
    spec:
        credentials:
        source: Secret
        secretRef:
            namespace: crossplane-system
            name: aws-secret
            key: creds
    ```
    </details>

    ```bash
    kubectl apply -f provider-aws-config.yaml
    ```

---

## Phase 4: Define a Reusable Infrastructure Abstraction

This is where the magic of Crossplane shines. We will define our own custom API for provisioning a "KubernetesCluster". This involves creating a few key components:
- **CompositeResourceDefinition (XRD):** This defines the schema for our custom API—what inputs it accepts and what outputs it returns.
- **Composition:** This maps our custom API to the actual cloud resources that need to be created.

We will create two levels of abstraction:
1.  `XNetworking`: A composition for all the necessary AWS networking resources (VPC, Subnets, etc.).
2.  `XEKSCluster`: A composition for the EKS cluster itself, which uses the networking resources.
3.  `XKubernetesCluster`: A top-level composition that brings the networking and EKS cluster compositions together into a single, simplified API for our users.

### Step 4.1: The Networking Layer

1.  **Create the Networking XRD (`xrd-networking.yaml`)**
    This defines an API to request a standard networking stack.
    <details>
    <summary><code>xrd-networking.yaml</code></summary>

    ```yaml
    # xrd-networking.yaml
    apiVersion: apiextensions.crossplane.io/v1
    kind: CompositeResourceDefinition
    metadata:
      name: xnetworkings.net.aws.crossplane.grazdev.io
    spec:
      group: net.aws.crossplane.grazdev.io
      names:
        kind: XNetworking
        plural: xnetworkings
      claimNames:
        kind: Networking
        plural: networkings
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
                    id:
                      type: string
                      description: ID of this Network that other objects will use to refer to it.
                    parameters:
                      type: object
                      description: Network configuration parameters.
                      properties:
                        region:
                          type: string
                      required:
                        - region
                  required:
                    - id
                    - parameters
                status:
                  type: object
                  properties:
                    subnetIds:
                      type: array
                      items:
                        type: string
                    securityGroupClusterIds:
                      type: array
                      items:
                        type: string
    ```
    </details>
    
    ```bash
    kubectl apply -f xrd-networking.yaml
    ```
    
2.  **Create the Networking Composition (`composition-networking.yaml`)**
    This Composition implements the `XNetworking` API by defining all the underlying AWS resources (VPC, Subnets, Internet Gateway, etc.). For more details on how this works, see the [Crossplane Composition documentation](https://docs.crossplane.io/latest/concepts/composition/).
    <details>
    <summary><code>composition-networking.yaml</code></summary>

    ```yaml
    # composition-networking.yaml
    apiVersion: apiextensions.crossplane.io/v1
    kind: Composition
    metadata:
      name: networking
      labels:
        provider: aws
    spec:
      compositeTypeRef:
        apiVersion: net.aws.crossplane.grazdev.io/v1alpha1
        kind: XNetworking

      writeConnectionSecretsToNamespace: crossplane-system

      patchSets:
      - name: networkconfig
        patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.id
          toFieldPath: metadata.labels[net.aws.crossplane.grazdev.io/network-id] # the network-id other Composition MRs (like EKSCluster) will use
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.region

      resources:
        ### VPC and InternetGateway
        - name: platform-vcp
          base:
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: VPC
            spec:
              forProvider:
                cidrBlock: 10.0.0.0/16
                enableDnsSupport: true
                enableDnsHostnames: true
                tags:
                  Owner: Platform Team
                  Name: platform-vpc
          patches:
            - type: PatchSet
              patchSetName: networkconfig
            - fromFieldPath: spec.id
              toFieldPath: metadata.name
        
        - name: gateway
          base:
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: InternetGateway
            spec:
              forProvider:
                vpcIdSelector:
                  matchControllerRef: true
          patches:
            - type: PatchSet
              patchSetName: networkconfig


        ### Subnet Configuration
        - name: subnet-public-eu-central-1a
          base:
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: Subnet
            metadata:
              labels:
                access: public
            spec:
              forProvider:
                mapPublicIpOnLaunch: true
                cidrBlock: 10.0.0.0/24
                vpcIdSelector:
                  matchControllerRef: true
                tags:
                  kubernetes.io/role/elb: "1"
          patches:
            - type: PatchSet
              patchSetName: networkconfig
            # define eu-central-1a as zone & availabilityZone
            - type: FromCompositeFieldPath
              fromFieldPath: spec.parameters.region
              toFieldPath: metadata.labels.zone
              transforms:
                - type: string
                  string:
                    fmt: "%sa"
            - type: FromCompositeFieldPath
              fromFieldPath: spec.parameters.region
              toFieldPath: spec.forProvider.availabilityZone
              transforms:
                - type: string
                  string:
                    fmt: "%sa"
            # provide the subnetId for later use as status.subnetIds entry
            - type: ToCompositeFieldPath
              fromFieldPath: metadata.annotations[crossplane.io/external-name]
              toFieldPath: status.subnetIds[0]
        
        - name: subnet-public-eu-central-1b
          base:
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: Subnet
            metadata:
              labels:
                access: public
            spec:
              forProvider:
                mapPublicIpOnLaunch: true
                cidrBlock: 10.0.1.0/24
                vpcIdSelector:
                  matchControllerRef: true
                tags:
                  kubernetes.io/role/elb: "1"
          patches:
            - type: PatchSet
              patchSetName: networkconfig
              # define eu-central-1b as zone & availabilityZone
            - type: FromCompositeFieldPath
              fromFieldPath: spec.parameters.region
              toFieldPath: metadata.labels.zone
              transforms:
                - type: string
                  string:
                    fmt: "%sb"
            - type: FromCompositeFieldPath
              fromFieldPath: spec.parameters.region
              toFieldPath: spec.forProvider.availabilityZone
              transforms:
                - type: string
                  string:
                    fmt: "%sb"
              # provide the subnetId for later use as status.subnetIds entry
            - type: ToCompositeFieldPath
              fromFieldPath: metadata.annotations[crossplane.io/external-name]
              toFieldPath: status.subnetIds[1]

        - name: subnet-public-eu-central-1c
          base:
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: Subnet
            metadata:
              labels:
                access: public
            spec:
              forProvider:
                mapPublicIpOnLaunch: true
                cidrBlock: 10.0.2.0/24
                vpcIdSelector:
                  matchControllerRef: true
                tags:
                  kubernetes.io/role/elb: "1"
          patches:
            - type: PatchSet
              patchSetName: networkconfig
              # define eu-central-1c as zone & availabilityZone
            - type: FromCompositeFieldPath
              fromFieldPath: spec.parameters.region
              toFieldPath: metadata.labels.zone
              transforms:
                - type: string
                  string:
                    fmt: "%sc"
            - type: FromCompositeFieldPath
              fromFieldPath: spec.parameters.region
              toFieldPath: spec.forProvider.availabilityZone
              transforms:
                - type: string
                  string:
                    fmt: "%sc"
              # provide the subnetId for later use as status.subnetIds entry
            - type: ToCompositeFieldPath
              fromFieldPath: metadata.annotations[crossplane.io/external-name]
              toFieldPath: status.subnetIds[2]  

        ### SecurityGroup & SecurityGroupRules Cluster API server
        - name: securitygroup-cluster
          base:
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: SecurityGroup
            metadata:
              labels:
                net.aws.crossplane.grazdev.io: securitygroup-cluster
            spec:
              forProvider:
                description: cluster API server access
                name: securitygroup-cluster
                vpcIdSelector:
                  matchControllerRef: true
          patches:
            - type: PatchSet
              patchSetName: networkconfig
            - fromFieldPath: spec.id
              toFieldPath: metadata.name
              # provide the securityGroupId for later use as status.securityGroupClusterIds entry
            - type: ToCompositeFieldPath
              fromFieldPath: metadata.annotations[crossplane.io/external-name]
              toFieldPath: status.securityGroupClusterIds[0]

        - name: securitygrouprule-cluster-inbound
          base:
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: SecurityGroupRule
            spec:
              forProvider:
                #description: Allow pods to communicate with the cluster API server & access API server from kubectl clients
                type: ingress
                cidrBlocks:
                  - 0.0.0.0/0
                fromPort: 443
                toPort: 443
                protocol: tcp
                securityGroupIdSelector:
                  matchLabels:
                    net.aws.crossplane.grazdev.io: securitygroup-cluster
          patches:
            - type: PatchSet
              patchSetName: networkconfig

        - name: securitygrouprule-cluster-outbound
          base:
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: SecurityGroupRule
            spec:
              forProvider:
                description: Allow internet access from the cluster API server
                type: egress
                cidrBlocks: # Destination
                  - 0.0.0.0/0
                fromPort: 0
                toPort: 0
                protocol: tcp
                securityGroupIdSelector:
                  matchLabels:
                    net.aws.crossplane.grazdev.io: securitygroup-cluster
          patches:
            - type: PatchSet
              patchSetName: networkconfig

        ### Route, RouteTable & RouteTableAssociations
        - name: route
          base:
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: Route
            spec:
              forProvider:
                destinationCidrBlock: 0.0.0.0/0
                gatewayIdSelector:
                  matchControllerRef: true
                routeTableIdSelector:
                  matchControllerRef: true
          patches:
            - type: PatchSet
              patchSetName: networkconfig

        - name: routeTable
          base:
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: RouteTable
            spec:
              forProvider:
                vpcIdSelector:
                  matchControllerRef: true
          patches:
          - type: PatchSet
            patchSetName: networkconfig

        - name: mainRouteTableAssociation
          base:
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: MainRouteTableAssociation
            spec:
              forProvider:
                routeTableIdSelector:
                  matchControllerRef: true
                vpcIdSelector:
                  matchControllerRef: true
          patches:
            - type: PatchSet
              patchSetName: networkconfig

        - name: RouteTableAssociation-public-a
          base:
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: RouteTableAssociation
            spec:
              forProvider:
                routeTableIdSelector:
                  matchControllerRef: true
                subnetIdSelector:
                  matchControllerRef: true
                  matchLabels:
                    access: public
          patches:
            - type: PatchSet
              patchSetName: networkconfig
            # define eu-central-1a as subnetIdSelector.matchLabels.zone
            - type: FromCompositeFieldPath
              fromFieldPath: spec.parameters.region
              toFieldPath: spec.forProvider.subnetIdSelector.matchLabels.zone
              transforms:
                - type: string
                  string:
                    fmt: "%sa"

        - name: RouteTableAssociation-public-b
          base:
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: RouteTableAssociation
            spec:
              forProvider:
                routeTableIdSelector:
                  matchControllerRef: true
                subnetIdSelector:
                  matchControllerRef: true
                  matchLabels:
                    access: public
          patches:
            - type: PatchSet
              patchSetName: networkconfig
            # define eu-central-1b as subnetIdSelector.matchLabels.zone
            - type: FromCompositeFieldPath
              fromFieldPath: spec.parameters.region
              toFieldPath: spec.forProvider.subnetIdSelector.matchLabels.zone
              transforms:
                - type: string
                  string:
                    fmt: "%sb"

        - name: RouteTableAssociation-public-c
          base:
            apiVersion: ec2.aws.upbound.io/v1beta1
            kind: RouteTableAssociation
            spec:
              forProvider:
                routeTableIdSelector:
                  matchControllerRef: true
                subnetIdSelector:
                  matchControllerRef: true
                  matchLabels:
                    access: public
          patches:
            - type: PatchSet
              patchSetName: networkconfig
            # define eu-central-1c as subnetIdSelector.matchLabels.zone
            - type: FromCompositeFieldPath
              fromFieldPath: spec.parameters.region
              toFieldPath: spec.forProvider.subnetIdSelector.matchLabels.zone
              transforms:
                - type: string
                  string:
                    fmt: "%sc"
    ```
    </details>
    
    ```bash
    kubectl apply -f composition-networking.yaml
    ```
    
### Step 4.2: The EKS Cluster Layer

1.  **Create the EKS Cluster XRD (`xrd-ekscluster.yaml`)**
    This defines an API for an EKS cluster, which requires networking information (like subnet IDs) as input.
    <details>
    <summary><code>xrd-ekscluster.yaml</code></summary>

    ```yaml
    # xrd-ekscluster.yaml
    apiVersion: apiextensions.crossplane.io/v1
    kind: CompositeResourceDefinition
    metadata:
      name: xeksclusters.eks.aws.crossplane.grazdev.io
    spec:
      group: eks.aws.crossplane.grazdev.io
      names:
        kind: XEKSCluster
        plural: xeksclusters
      claimNames:
        kind: EKSCluster
        plural: eksclusters
      defaultCompositionRef:
        name: aws-eks
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
                  id:
                    type: string
                    description: ID of this Cluster that other objects will use to refer to it.
                  parameters:
                    type: object
                    description: EKS configuration parameters.
                    properties:
                      subnetIds:
                        type: array
                        items:
                          type: string
                      securityGroupClusterIds:
                        type: array
                        items:
                          type: string
                      region:
                        type: string
                      nodes:
                        type: object
                        description: EKS node configuration parameters.
                        properties:
                          count:
                            type: integer
                            description: Desired node count, from 1 to 10.
                        required:
                        - count
                    required:
                    - subnetIds
                    - securityGroupClusterIds
                    - region
                    - nodes
                required:
                - id
                - parameters
              status:
                type: object
                properties:
                  clusterStatus:
                    description: The status of the control plane
                    type: string
                  nodePoolStatus:
                    description: The status of the node pool
                    type: string
    ```
    </details>

    ```bash
    kubectl apply -f xrd-ekscluster.yaml
    ```

2.  **Create the EKS Cluster Composition (`composition-ekscluster.yaml`)**
    This Composition creates the EKS control plane, node groups, and associated IAM roles. It gets the network details from the `XNetworking` resource we defined earlier.
    <details>
    <summary><code>composition-ekscluster.yaml</code></summary>

    ```yaml
    # composition-ekscluster.yaml
    apiVersion: apiextensions.crossplane.io/v1
    kind: Composition
    metadata:
      name: aws-eks
      labels:
        provider: aws
    spec:
      compositeTypeRef:
        apiVersion: eks.aws.crossplane.grazdev.io/v1alpha1
        kind: XEKSCluster
      
      writeConnectionSecretsToNamespace: crossplane-system

      patchSets:
      - name: clusterconfig
        patches:
        - fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.region

      resources:
        ### Cluster Configuration
        - name: eksCluster
          base:
            apiVersion: eks.aws.upbound.io/v1beta1
            kind: Cluster
            metadata:
              annotations:
                meta.upbound.io/example-id: eks/v1beta1/cluster
                uptest.upbound.io/timeout: "2400"
            spec:
              forProvider:
                roleArnSelector:
                  matchControllerRef: true
                  matchLabels:
                    role: clusterRole
                vpcConfig:
                  - endpointPrivateAccess: true
                    endpointPublicAccess: true
          patches:
            - type: PatchSet
              patchSetName: clusterconfig
            - fromFieldPath: spec.id
              toFieldPath: metadata.name
            # Using the XNetworking defined securityGroupClusterIds & subnetIds for the vpcConfig
            - fromFieldPath: spec.parameters.securityGroupClusterIds
              toFieldPath: spec.forProvider.vpcConfig[0].securityGroupIds
            - fromFieldPath: spec.parameters.subnetIds
              toFieldPath: spec.forProvider.vpcConfig[0].subnetIds

            - type: ToCompositeFieldPath
              fromFieldPath: status.atProvider.status
              toFieldPath: status.clusterStatus    
          readinessChecks:
            - type: MatchString
              fieldPath: status.atProvider.status
              matchString: ACTIVE

        - name: kubernetesClusterAuth
          base:
            apiVersion: eks.aws.upbound.io/v1beta1
            kind: ClusterAuth
            spec:
              forProvider:
                clusterNameSelector:
                  matchControllerRef: true
          patches:
            - type: PatchSet
              patchSetName: clusterconfig
            - fromFieldPath: spec.writeConnectionSecretToRef.namespace
              toFieldPath: spec.writeConnectionSecretToRef.namespace
            - fromFieldPath: spec.id
              toFieldPath: spec.writeConnectionSecretToRef.name
              transforms:
                - type: string
                  string:
                    fmt: "%s-access"
          connectionDetails:
            - fromConnectionSecretKey: kubeconfig

        ### Cluster Role and Policies
        - name: clusterRole
          base:
            apiVersion: iam.aws.upbound.io/v1beta1
            kind: Role
            metadata:
              labels:
                role: clusterRole
            spec:
              forProvider:
                assumeRolePolicy: |
                  {
                    "Version": "2012-10-17",
                    "Statement": [
                        {
                            "Effect": "Allow",
                            "Principal": {
                                "Service": [
                                    "eks.amazonaws.com"
                                ]
                            },
                            "Action": [
                                "sts:AssumeRole"
                            ]
                        }
                    ]
                  }
          
        
        - name: clusterRolePolicyAttachment
          base:
            apiVersion: iam.aws.upbound.io/v1beta1
            kind: RolePolicyAttachment
            spec:
              forProvider:
                policyArn: arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
                roleSelector:
                  matchControllerRef: true
                  matchLabels:
                    role: clusterRole


        ### NodeGroup Configuration
        - name: nodeGroupPublic
          base:
            apiVersion: eks.aws.upbound.io/v1beta1
            kind: NodeGroup
            spec:
              forProvider:
                clusterNameSelector:
                  matchControllerRef: true
                nodeRoleArnSelector:
                  matchControllerRef: true
                  matchLabels:
                    role: nodegroup
                subnetIdSelector:
                  matchLabels:
                    access: public
                scalingConfig:
                  - minSize: 1
                    maxSize: 10
                    desiredSize: 1
                instanceTypes:
                  - t3.small
          patches:
            - type: PatchSet
              patchSetName: clusterconfig
            - fromFieldPath: spec.parameters.nodes.count
              toFieldPath: spec.forProvider.scalingConfig[0].desiredSize
            - fromFieldPath: spec.id
              toFieldPath: spec.forProvider.subnetIdSelector.matchLabels[net.aws.crossplane.grazdev.io/network-id]
            - type: ToCompositeFieldPath
              fromFieldPath: status.atProvider.status
              toFieldPath: status.nodePoolStatus  
          readinessChecks:
          - type: MatchString
            fieldPath: status.atProvider.status
            matchString: ACTIVE

        ### Node Role and Policies
        - name: nodegroupRole
          base:
            apiVersion: iam.aws.upbound.io/v1beta1
            kind: Role
            metadata:
              labels:
                role: nodegroup
            spec:
              forProvider:
                assumeRolePolicy: |
                  {
                    "Version": "2012-10-17",
                    "Statement": [
                        {
                            "Effect": "Allow",
                            "Principal": {
                                "Service": [
                                    "ec2.amazonaws.com"
                                ]
                            },
                            "Action": [
                                "sts:AssumeRole"
                            ]
                        }
                    ]
                  }
          

        - name: workerNodeRolePolicyAttachment
          base:
            apiVersion: iam.aws.upbound.io/v1beta1
            kind: RolePolicyAttachment
            spec:
              forProvider:
                policyArn: arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
                roleSelector:
                  matchControllerRef: true
                  matchLabels:
                    role: nodegroup
          

        - name: cniRolePolicyAttachment
          base:
            apiVersion: iam.aws.upbound.io/v1beta1
            kind: RolePolicyAttachment
            spec:
              forProvider:
                policyArn: arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
                roleSelector:
                  matchControllerRef: true
                  matchLabels:
                    role: nodegroup
          
        - name: containerRegistryRolePolicyAttachment
          base:
            apiVersion: iam.aws.upbound.io/v1beta1
            kind: RolePolicyAttachment
            spec:
              forProvider:
                policyArn: arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
                roleSelector:
                  matchControllerRef: true
                  matchLabels:
                    role: nodegroup
    ```
    </details>
    
    ```bash
    kubectl apply -f composition-ekscluster.yaml
    ```
    
### Step 4.3: The Top-Level Kubernetes Cluster Abstraction

Now we create the final, user-facing abstraction that combines networking and the EKS cluster into one simple API.

1.  **Create the Top-Level XRD (`xrd-kubernetescluster.yaml`)**
    This is the simplified API we will expose to users. It only requires a region and node count.
    <details>
    <summary><code>xrd-kubernetescluster.yaml</code></summary>

    ```yaml
    # xrd-kubernetescluster.yaml
    apiVersion: apiextensions.crossplane.io/v1
    kind: CompositeResourceDefinition
    metadata:
      name: xkubernetesclusters.k8s.crossplane.grazdev.io
    spec:
      group: k8s.crossplane.grazdev.io
      names:
        kind: XKubernetesCluster
        plural: xkubernetesclusters
      claimNames:
        kind: KubernetesCluster
        plural: kubernetesclusters
      connectionSecretKeys:
      - kubeconfig
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
                  id:
                    type: string
                    description: ID of this Cluster that other objects will use to refer to it.
                  parameters:
                    type: object
                    description: Cluster configuration parameters.
                    properties:
                      region:
                        type: string
                      nodes:
                        type: object
                        description: Cluster node configuration parameters.
                        properties:
                          count:
                            type: integer
                            description: Desired node count, from 1 to 100.
                        required:
                        - count
                    required:
                    - region
                    - nodes
                required:
                - id
                - parameters
              status:
                type: object
                properties:
                  subnetIds:
                    type: array
                    items:
                      type: string
                  securityGroupClusterIds:
                    type: array
                    items:
                      type: string
    ```
    </details>
    
    ```bash
    kubectl apply -f xrd-kubernetescluster.yaml
    ```
    
2.  **Create the Top-Level Composition (`composition-kubernetescluster.yaml`)**
    This Composition nests the `XNetworking` and `XEKSCluster` resources, creating the entire stack from a single user request.
    <details>
    <summary><code>composition-kubernetescluster.yaml</code></summary>

    ```yaml
    # composition-kubernetescluster.yaml
    apiVersion: apiextensions.crossplane.io/v1
    kind: Composition
    metadata:
      name: kubernetes-cluster
    spec:
      compositeTypeRef:
        apiVersion: k8s.crossplane.grazdev.io/v1alpha1
        kind: XKubernetesCluster
      
      writeConnectionSecretsToNamespace: crossplane-system

      resources:
        ### Nested use of XNetworking XR
        - name: compositeNetworkEKS
          base:
            apiVersion: net.aws.crossplane.grazdev.io/v1alpha1
            kind: XNetworking
          patches:
            - fromFieldPath: spec.id
              toFieldPath: spec.id
            - fromFieldPath: spec.parameters.region
              toFieldPath: spec.parameters.region
            # provide the subnetIds & securityGroupClusterIds for later use
            - type: ToCompositeFieldPath
              fromFieldPath: status.subnetIds
              toFieldPath: status.subnetIds
              policy:
                fromFieldPath: Required
            - type: ToCompositeFieldPath
              fromFieldPath: status.securityGroupClusterIds
              toFieldPath: status.securityGroupClusterIds
              policy:
                fromFieldPath: Required
        
        ### Nested use of XEKSCluster XR
        - name: compositeClusterEKS
          base:
            apiVersion: eks.aws.crossplane.grazdev.io/v1alpha1
            kind: XEKSCluster
          connectionDetails:
            - fromConnectionSecretKey: kubeconfig
          patches:
            - fromFieldPath: spec.id
              toFieldPath: spec.id
            - fromFieldPath: spec.id
              toFieldPath: metadata.annotations[crossplane.io/external-name]
            - fromFieldPath: metadata.uid
              toFieldPath: spec.writeConnectionSecretToRef.name
              transforms:
                - type: string
                  string:
                    fmt: "%s-eks"
            - fromFieldPath: spec.writeConnectionSecretToRef.namespace
              toFieldPath: spec.writeConnectionSecretToRef.namespace
            - fromFieldPath: spec.parameters.region
              toFieldPath: spec.parameters.region
            - fromFieldPath: spec.parameters.nodes.count
              toFieldPath: spec.parameters.nodes.count
            - fromFieldPath: status.subnetIds
              toFieldPath: spec.parameters.subnetIds
              policy:
                fromFieldPath: Required
            - fromFieldPath: status.securityGroupClusterIds
              toFieldPath: spec.parameters.securityGroupClusterIds
              policy:
                fromFieldPath: Required
    ```
    </details>

    ```bash
    kubectl apply -f composition-kubernetescluster.yaml
    ```

---

## Phase 5: Provision and Manage the EKS Cluster

Now that we have defined our custom `KubernetesCluster` API, let's use it to provision a real cluster.

1.  **Create a Claim**
    A `Claim` is a request for a resource defined by our XRD. This simple YAML is all a user needs to provision a complete EKS cluster.
    <details>
    <summary><code>claim-kubernetescluster.yaml</code></summary>

    ```yaml
    # claim-kubernetescluster.yaml
    apiVersion: k8s.crossplane.grazdev.io/v1alpha1
    kind: KubernetesCluster
    metadata:
      namespace: default
      name: deploy-target-eks
    spec:
      id: deploy-target-eks
      parameters:
        region: eu-central-1
        nodes:
          count: 3
      writeConnectionSecretToRef:
        name: eks-cluster-kubeconfig
    ```
    </details>
    <details>
    <summary>Apply the Claim</summary>

    ```bash
    kubectl apply -f claim-kubernetescluster.yaml
    ```
    </details>

2.  **Monitor Provisioning**
    Provisioning an EKS cluster can take around 20 minutes. You can monitor the status with the following command:
    <details>
    <summary>Check Cluster Status</summary>

    ```bash
    kubectl get kubernetescluster
    ```
    </details>
    Wait until `SYNCED` and `READY` are both `True`.
    ```console
    NAME                SYNCED   READY   CONNECTION-SECRET        AGE
    deploy-target-eks   True     True    eks-cluster-kubeconfig   59m
    ```

3.  **Access the New EKS Cluster**
    Once ready, Crossplane creates a secret containing the `kubeconfig` for the new cluster. Let's extract it and verify access.
    <details>
    <summary>Get Kubeconfig and Verify Nodes</summary>

    ```bash
    # Extract the kubeconfig
    kubectl get secret eks-cluster-kubeconfig -o jsonpath='{.data.kubeconfig}' | base64 --decode > ekskubeconfig

    # Use the kubeconfig to get the nodes of the new cluster
    KUBECONFIG=ekskubeconfig kubectl get nodes
    ```
    </details>
    You should see the three nodes you requested:
    ```console
    NAME                                          STATUS   ROLES    AGE     VERSION
    ip-10-0-0-23.eu-central-1.compute.internal    Ready    <none>   9m19s   v1.33.0-eks-802817d
    ip-10-0-1-173.eu-central-1.compute.internal   Ready    <none>   9m16s   v1.33.0-eks-802817d
    ip-10-0-2-197.eu-central-1.compute.internal   Ready    <none>   46m     v1.33.0-eks-802817d
    ```

---

## Phase 6: Implement Hibernation with kube-green

Now, let's configure kube-green to automatically scale down our cluster's node pool to save costs during inactive hours.

1.  **Grant kube-green Permissions**
    We need to give kube-green permission to modify our custom `KubernetesCluster` resources.
    <details>
    <summary><code>kube-green-kubernetescluster.yaml</code></summary>

    ```yaml
    # kube-green-kubernetescluster.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: kube-green-kubernetescluster-patcher
    rules:
    - apiGroups: ["k8s.crossplane.grazdev.io"]
      resources: ["kubernetesclusters"]
      verbs: ["get", "list", "watch", "patch"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: kube-green-kubernetescluster-patcher
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: kube-green-kubernetescluster-patcher
    subjects:
    - kind: ServiceAccount
      name: kube-green-controller-manager 
      namespace: kube-green
    ```
    </details>
    <details>
    <summary>Apply the RBAC rules</summary>

    ```bash
    kubectl apply -f kube-green-kubernetescluster.yaml
    ```
    </details>

2.  **Create a SleepInfo Resource**
    The `SleepInfo` resource tells kube-green when to sleep and wake up, and what to patch. In this case, we patch the `nodes.count` parameter of our `KubernetesCluster` to scale it down.
    <details>
    <summary><code>sleepinfo.yaml</code></summary>

    ```yaml
    # sleepinfo.yaml
    apiVersion: kube-green.com/v1alpha1
    kind: SleepInfo
    metadata:
      name: sleep-schedule-for-kubernetescluster
      namespace: default
    spec:
      weekdays: "*"
      timeZone: "Europe/Rome"
      sleepAt: "21:51" # Adjust to a few minutes from now for testing
      wakeUpAt: "22:10" # Adjust to a few minutes after sleepAt
      patches:
      - target:
          group: k8s.crossplane.grazdev.io
          kind: KubernetesCluster
        patch: |
          - op: replace
            path: /spec/parameters/nodes/count
            value: 1
    ```
    </details>
    <details>
    <summary>Apply the SleepInfo</summary>

    ```bash
    kubectl apply -f sleepinfo.yaml
    ```
    </details>

3.  **Verify Hibernation (Sleep)**
    At the `sleepAt` time, kube-green will patch our `KubernetesCluster` resource, and Crossplane will scale down the EKS node group. This can take a few minutes.
    <details>
    <summary>Check Nodes After Sleep</summary>

    ```bash
    KUBECONFIG=ekskubeconfig kubectl get nodes
    ```
    </details>
    You should see the node count decrease to 1:
    ```console
    NAME                                         STATUS   ROLES    AGE   VERSION
    ip-10-0-2-96.eu-central-1.compute.internal   Ready    <none>   14m   v1.33.3-eks-3abbec1
    ```
    *Note: EC2 instances are terminated one at a time, so this process is not instantaneous.*

4.  **Verify Hibernation (Wake Up)**
    At the `wakeUpAt` time, kube-green will revert the patch, and Crossplane will scale the node group back up to its original count.
    <details>
    <summary>Check Nodes After Wake Up</summary>

    ```bash
    KUBECONFIG=ekskubeconfig kubectl get nodes
    ```
    </details>
    You should see the node count return to 3:
    ```console
    NAME                                          STATUS   ROLES    AGE     VERSION
    ip-10-0-0-23.eu-central-1.compute.internal    Ready    <none>   9m19s   v1.33.0-eks-802817d
    ip-10-0-1-173.eu-central-1.compute.internal   Ready    <none>   9m16s   v1.33.0-eks-802817d
    ip-10-0-2-197.eu-central-1.compute.internal   Ready    <none>   46m     v1.33.0-eks-802817d
    ```

Congratulations! You have successfully created a self-service, ephemeral EKS cluster with scheduled hibernation.
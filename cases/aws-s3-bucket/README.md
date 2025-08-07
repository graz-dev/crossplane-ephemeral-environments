# Ephemeral S3 Buckets with Crossplane and kube-green

Welcome to this hands-on tutorial! Together, we will explore how to dynamically provision and manage the lifecycle of an Amazon S3 (Simple Storage Service) bucket using the power of [Crossplane](https://www.crossplane.io/) and [kube-green](https://kube-green.dev/).

The main goal is to create a reusable, on-demand S3 bucket definition that can be provisioned whenever needed and automatically have its policies updated during inactive hours to enhance security. This is a common pattern for managing access to resources in ephemeral development or testing environments.

We will cover:
- Setting up a local control plane with **Kind**.
- Installing and configuring **Crossplane** and its AWS providers.
- Defining our own cloud infrastructure abstractions using **Composition**.
- Provisioning a secure S3 bucket.
- Integrating **kube-green** to schedule policy changes for our bucket, restricting access during off-hours.

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

## Phase 3: Configure AWS Provider for Crossplane

Crossplane uses **Providers** to interact with external APIs like AWS. We need to install and configure the specific provider for S3.

1.  **Install the AWS S3 Provider**
    This provider manages S3 resources. You can find it here: [manifests/providers/provider-aws-s3.yaml](manifests/providers/provider-aws-s3.yaml)
    <details>
    <summary><code>manifests/providers/provider-aws-s3.yaml</code></summary>

    ```yaml
    apiVersion: pkg.crossplane.io/v1
    kind: Provider
    metadata:
      name: provider-aws-s3
    spec:
      package: xpkg.upbound.io/upbound/provider-aws-s3:v1.23.1
    ```
    </details>

    ```bash
    kubectl apply -f manifests/providers/provider-aws-s3.yaml
    ```

2.  **Verify Provider Installation**
    Check that the provider is installed and healthy.

    ```bash
    kubectl get providers.pkg.crossplane.io
    ```

    The output should show the provider as `INSTALLED` and `HEALTHY`.

3.  **Configure AWS Credentials**
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

4.  **Create a ProviderConfig**
    The `ProviderConfig` tells the AWS provider how to authenticate. It references the secret we just created. You can find it here: [manifests/providers/provider-aws-config.yaml](manifests/providers/provider-aws-config.yaml)
    <details>
    <summary><code>manifests/providers/provider-aws-config.yaml</code></summary>

    ```yaml
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
    kubectl apply -f manifests/providers/provider-aws-config.yaml
    ```

---

## Phase 4: Define a Reusable Infrastructure Abstraction

This is where the magic of Crossplane shines. We will define our own custom API for provisioning a "SecureBucket". This involves creating a few key components:
- **CompositeResourceDefinition (XRD):** This defines the schema for our custom API—what inputs it accepts.
- **Composition:** This maps our custom API to the actual AWS resources that need to be created (Bucket, BucketPolicy, and BucketPublicAccessBlock).

1.  **Create the SecureBucket XRD (`xrd-securebucket.yaml`)**
    This defines an API to request a secure S3 bucket. You can find it here: [manifests/apis/xrd-securebucket.yaml](manifests/apis/xrd-securebucket.yaml)
    <details>
    <summary><code>manifests/apis/xrd-securebucket.yaml</code></summary>

    ```yaml
    apiVersion: apiextensions.crossplane.io/v1
    kind: CompositeResourceDefinition
    metadata:
      name: xsecurebuckets.aws.crossplane.grazdev.io
    spec:
      group: aws.crossplane.grazdev.io
      names:
        kind: XSecureBucket
        plural: xsecurebuckets
      claimNames:
        kind: SecureBucket
        plural: securebuckets
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
                  parameters:
                    type: object
                    properties:
                      bucketName:
                        description: "Bucket name."
                        type: string
                      region:
                        description: "Bucket region."
                        type: string
                      policy:
                        description: "Bucket policy in JSON format."
                        type: string
                      blockPublicPolicy:
                        description: "Block public access policy."
                        type: boolean
                        default: true
                    required:
                    - bucketName
                    - region
                    - policy
    ```
    </details>
    
    ```bash
    kubectl apply -f manifests/apis/xrd-securebucket.yaml
    ```
    
2.  **Create the SecureBucket Composition (`composition-securebucket.yaml`)**
    This Composition implements the `XSecureBucket` API by defining the underlying AWS S3 resources. For more details on how this works, see the [Crossplane Composition documentation](https://docs.crossplane.io/latest/concepts/composition/). You can find it here: [manifests/apis/composition-securebucket.yaml](manifests/apis/composition-securebucket.yaml)
    <details>
    <summary><code>manifests/apis/composition-securebucket.yaml</code></summary>

    ```yaml
    apiVersion: apiextensions.crossplane.io/v1
    kind: Composition
    metadata:
      name: s3.securebucket.aws.crossplane.grazdev.io
      labels:
        type: secure-s3-from-manifests
    spec:
      compositeTypeRef:
        apiVersion: aws.crossplane.grazdev.io/v1alpha1
        kind: XSecureBucket
      resources:
        - name: bucket
          base:
            apiVersion: s3.aws.upbound.io/v1beta1
            kind: Bucket
            spec:
              forProvider: {}
              providerConfigRef:
                name: default
          patches:
            - fromFieldPath: "spec.parameters.bucketName"
              toFieldPath: "metadata.name"
            - fromFieldPath: "spec.parameters.region"
              toFieldPath: "spec.forProvider.region"
        - name: bucket-policy
          base:
            apiVersion: s3.aws.upbound.io/v1beta1
            kind: BucketPolicy
            spec:
              forProvider: {}
          patches:
            - fromFieldPath: "spec.parameters.bucketName"
              toFieldPath: "metadata.name"
              transforms:
                - type: string
                  string:
                    fmt: "%s-policy"
            - fromFieldPath: "spec.parameters.bucketName"
              toFieldPath: "spec.forProvider.bucket"
            - fromFieldPath: "spec.parameters.region"
              toFieldPath: "spec.forProvider.region"
            - fromFieldPath: "spec.parameters.policy"
              toFieldPath: "spec.forProvider.policy"
        - name: bucket-public-access-block
          base:
            apiVersion: s3.aws.upbound.io/v1beta1
            kind: BucketPublicAccessBlock
            spec:
              forProvider:
                blockPublicAcls: false
                restrictPublicBuckets: false
          patches:
            - fromFieldPath: "spec.parameters.bucketName"
              toFieldPath: "metadata.name"
              transforms:
                - type: string
                  string:
                    fmt: "%s-public-access-block"
            - fromFieldPath: "spec.parameters.bucketName"
              toFieldPath: "spec.forProvider.bucket"
            - fromFieldPath: "spec.parameters.region"
              toFieldPath: "spec.forProvider.region"
            - fromFieldPath: "spec.parameters.blockPublicPolicy"
              toFieldPath: "spec.forProvider.blockPublicPolicy"
    ```
    </details>
    
    ```bash
    kubectl apply -f manifests/apis/composition-securebucket.yaml
    ```

---

## Phase 5: Provision and Manage the S3 Bucket

Now that we have defined our custom `SecureBucket` API, let's use it to provision a real bucket.

1.  **Create a Claim**
    A `Claim` is a request for a resource defined by our XRD. This simple YAML is all a user needs to provision a complete S3 bucket with a policy. You can find it here: [manifests/claims/claim-securebucket.yaml](manifests/claims/claim-securebucket.yaml)
    <details>
    <summary><code>manifests/claims/claim-securebucket.yaml</code></summary>

    ```yaml
    apiVersion: aws.crossplane.grazdev.io/v1alpha1
    kind: SecureBucket
    metadata:
      name: my-final-bucket-for-production
    spec:
      compositionSelector:
        matchLabels:
          type: secure-s3-from-manifests
      parameters:
        bucketName: "my-final-bucket-for-production-2025"
        region: "us-central-1"
        blockPublicPolicy: false
        # change with your aws account id and username below
        policy: |
          {
            "Version": "2012-10-17",
            "Statement": [
              {
                "Sid": "AllowAdminAccess",
                "Effect": "Allow",
                "Principal": {
                  "AWS": "arn:aws:iam::<your-aws-account-id>:user/<your-aws-username>"
                },
                "Action": "s3:*",
                "Resource": [
                  "arn:aws:s3:::my-final-bucket-for-production-2025",
                  "arn:aws:s3:::my-final-bucket-for-production-2025/*"
                ]
              }
            ]
          }
    ```
    </details>
    
    ```bash
    kubectl apply -f manifests/claims/claim-securebucket.yaml
    ```

2.  **Monitor Provisioning**
    Provisioning an S3 bucket is usually very fast. You can monitor the status with the following command:
    
    ```bash
    kubectl get securebuckets
    ```
    
    Wait until `SYNCED` and `READY` are both `True`.

    ```console
    NAME                             SYNCED   READY   CONNECTION-SECRET   AGE
    my-final-bucket-for-production   True     True                        38s
    ```
    
    You can also verify that the bucket was created in the AWS Console.

    ![](/cases/aws-s3-bucket/imgs/s3-1.png)

---

## Phase 6: Implement Hibernation with kube-green

Now, let's configure kube-green to automatically update our bucket's policy to restrict access during inactive hours.

1.  **Grant kube-green Permissions**
    We need to give kube-green permission to modify our custom `SecureBucket` resources. You can find it here: [manifests/kube-green/kube-green-securebucket.yaml](manifests/kube-green/kube-green-securebucket.yaml)
    <details>
    <summary><code>manifests/kube-green/kube-green-securebucket.yaml</code></summary>

    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: kube-green-s3-patcher
    rules:
    - apiGroups: ["aws.crossplane.grazdev.io"]
      resources: ["securebuckets"]
      verbs: ["get", "list", "watch", "patch"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: kube-green-s3-patcher-binding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: kube-green-s3-patcher
    subjects:
    - kind: ServiceAccount
      name: kube-green-controller-manager 
      namespace: kube-green
    ```
    </details>
    
    ```bash
    kubectl apply -f manifests/kube-green/kube-green-securebucket.yaml
    ```

2.  **Create a SleepInfo Resource**
    The `SleepInfo` resource tells kube-green when to sleep and wake up, and what to patch. In this case, we patch the `policy` parameter of our `SecureBucket` to deny all access except for a specific IAM user. You can find it here: [manifests/kube-green/sleepinfo.yaml](manifests/kube-green/sleepinfo.yaml)
    <details>
    <summary><code>manifests/kube-green/sleepinfo.yaml</code></summary>

    ```yaml
    apiVersion: kube-green.com/v1alpha1
    kind: SleepInfo
    metadata:
      name: lock-s3-bucket
    spec:
      weekdays: "*"
      timeZone: "Europe/Rome"
      sleepAt: "19:05" # Adjust to a few minutes from now for testing
      wakeUpAt: "19:07" # Adjust to a few minutes from now for testing
      patches:
      - target:
          group: aws.crossplane.grazdev.io
          kind: SecureBucket
        # change with your aws account id and username below
        patch: |
          - op: replace
            path: /spec/parameters/policy
            value: |
              {
                "Version": "2012-10-17",
                "Statement": [
                  {
                    "Sid": "AllowAdminAccess",
                    "Effect": "Allow",
                    "Principal": {
                      "AWS": "arn:aws:iam::<your-account-id>:user/<your-username>"
                    },
                    "Action": "s3:*",
                    "Resource": [
                      "arn:aws:s3:::my-final-bucket-for-production-2025",
                      "arn:aws:s3:::my-final-bucket-for-production-2025/*"
                    ]
                  },
                  {
                    "Sid": "DenyEveryoneElse",
                    "Effect": "Deny",
                    "Principal": "*",
                    "Action": "s3:*",
                    "Resource": [
                      "arn:aws:s3:::my-final-bucket-for-production-2025",
                      "arn:aws:s3:::my-final-bucket-for-production-2025/*"
                    ],
                    "Condition": {
                      "StringNotLike": {
                        "aws:PrincipalArn": "arn:aws:iam::<your-account-id>:user/<your-username>"
                      }
                    }
                  }
                ]
              }
    ```
    </details>
    
    ```bash
    kubectl apply -f manifests/kube-green/sleepinfo.yaml
    ```

3.  **Verify Hibernation (Sleep)**
    At the `sleepAt` time, kube-green will patch our `SecureBucket` resource, and Crossplane will update the S3 bucket policy.
    
    You can check the resource:
    ```bash
    kubectl describe securebuckets my-final-bucket-for-production
    ```
    
    And your bucket on the AWS console should show the new, more restrictive policy:

    ![](/cases/aws-s3-bucket/imgs/s3-2.png)

4.  **Verify Hibernation (Wake Up)**
    At the `wakeUpAt` time, kube-green will revert the patch, and Crossplane will restore the original bucket policy.
    
    The `SecureBucket` should return to its original state, and the bucket too:
    
    ![](/cases/aws-s3-bucket/imgs/s3-3.png)

Congratulations! You have successfully created a self-service S3 bucket with scheduled policy updates.

---
## Considerations: Security Benefits

This approach of using `kube-green` to manage S3 bucket policies provides significant security benefits, especially for non-production environments.

By automatically applying a restrictive policy during off-hours, you can:

-   **Minimize Exposure:** Reduce the time window during which the bucket is accessible, minimizing the risk of unauthorized access or data leakage.
-   **Enforce Least Privilege:** Ensure that even if initial permissions are broad for development or testing, they are automatically tightened to a "deny-all" state when not in active use.
-   **Automate Security Best Practices:** Instead of relying on manual changes, the security posture is automated, reducing the chance of human error.

This simple example demonstrates how combining Crossplane's infrastructure management with kube-green's lifecycle automation can create more secure, on-demand environments. While the cost savings for S3 storage itself might be negligible in this scenario, the security value of time-based access control is substantial.
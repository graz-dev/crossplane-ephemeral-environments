# Ephemeral Environments with Crossplane and kube-green

![Crossplane Badge](https://img.shields.io/badge/tool-crossplane-blue) ![kube-green Badge](https://img.shields.io/badge/tool-kube_green-blue) </br>
![AWS EKS](https://img.shields.io/badge/resource-AWS_EKS_Cluster-blue) ![AWS S3](https://img.shields.io/badge/resource-AWS_S3_Bucket-blue) ![Cloudnative PG](https://img.shields.io/badge/resource-CloudNativePG_Cluster-blue)


This repository contains a collection of hands-on tutorials demonstrating how to create and manage ephemeral environments using [Crossplane](https://www.crossplane.io/) for infrastructure provisioning and [kube-green](https://kube-green.dev/) for lifecycle management (hibernation).

The main goal is to showcase how to build self-service, on-demand infrastructure that can be automatically scaled down or put to sleep during inactive hours to save resources and costs. This is a powerful pattern for development, testing, and staging environments.

## Available Tutorials

The tutorials are organized into `cases`, each focusing on a different scenario. I recommend following them in the order below, as they progressively increase in complexity.

> **Note on AWS Usage**
> The `aws-s3-bucket` and `aws-eks-cluster` tutorials require an active AWS subscription. However, the resources provisioned are designed to be compatible with the [AWS Free Tier](https://aws.amazon.com/free/) to minimize costs.

### 1. CloudNativePG Operator: Managing Kubernetes Operator Resources

*   **Complexity:** Low
*   **Focus:** Learn how to manage Kubernetes operators and their custom resources with Crossplane. This case shows how to provision a PostgreSQL cluster using the [CloudNativePG operator](https://cloudnative-pg.io/) and then hibernate it with kube-green by scaling its pods to zero.
*   **Good for:** Understanding the basics of Crossplane's `provider-kubernetes` and managing resources within your Kubernetes cluster.

[**➡️ Start with the CloudNativePG tutorial**](./cases/cloudnativepg-operator/README.md)

### 2. AWS S3 Bucket: Provisioning Simple Cloud Resources

*   **Complexity:** Medium
*   **Focus:** This tutorial demonstrates how to provision a simple cloud resource, an AWS S3 bucket. You will learn how to use Crossplane to create the bucket and then use kube-green to apply a more restrictive security policy during off-hours, enhancing security.
*   **Good for:** A first step into provisioning external cloud resources with Crossplane and managing their configuration over time.

[**➡️ Continue with the AWS S3 Bucket tutorial**](./cases/aws-s3-bucket/README.md)

### 3. AWS EKS Cluster: Provisioning Complex Infrastructure

*   **Complexity:** High
*   **Focus:** This is the most advanced tutorial, guiding you through provisioning a complete Amazon EKS (Elastic Kubernetes Service) cluster with its networking stack. You will define a high-level abstraction for a Kubernetes cluster and use kube-green to hibernate it by scaling down the node pool to save significant costs.
*   **Good for:** Learning how to compose complex infrastructure stacks and manage their lifecycle for cost optimization.

[**➡️ Finish with the AWS EKS Cluster tutorial**](./cases/aws-eks-cluster/README.md)

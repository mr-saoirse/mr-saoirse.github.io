---
layout: post
title:  "EKS K8s with Terraform"
date:   2020-12-07 17:49:28 -0500
categories: IaC
---

# Overview

I am using this EKS tutorial as an exercise in terraform and AWS tweaking. I may or not use EKS in this way or perhaps use a Fargate centric method. The [linked tutorial][tutorial] provides the meat. It is an excellent tutorial and I am still keeping track of my own experience here. The tutorial is good for showing something like back best practices for a dev setup such as multiple environments.  EKS is a managed service that costs a few dollars a day to keep running. Fargate price (TODO)

Im using a Dev application user on my AWS account.

Start by listing clusters
```bash
aws eks list-clusters
```

### Note
```bash
kubectl config current-context
```

To check the config or kubectl
```bash
 kubectl config view --minify
```

## Basics

Create the EKS role

The easiest way to create an EKS cluster is

```bash
eksctl create cluster --name test-cluster  --node-type t2.micro --nodes 3 --nodes-min 3  --nodes-max 5  --region  us-east-1
```

Or better with yaml using the `-f file.yaml`

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: learnk8s
  region: eu-central-1
nodeGroups:
  - name: worker-group
    instanceType: t2.micro
    desiredCapacity: 3
    minSize: 3
    maxSize: 5
```

Delete with

```bash
eksctl delete cluster --name [name] --region us-east-1
```

## Terraform

I updated to a recent version of the aws eks terraform module and ran it in about 20 minutes. It creates a kubec config in the output and you can do your thing with

```bash
KUBECONFIG=./kubeconfig_my-cluster kubectl get pods --all-namespaces
```
Or for convenience set up for this session
```bash
export KUBECONFIG="${PWD}/kubeconfig_my-cluster"
```

The EKS is created in a VPC and uses the private subsets. Apart from the VPC construction and the ability to hook into other things, the EKS bit is not that much more complicated than the yaml. WE could do things like add sections for GPUs

```yaml
gpu = {
      desired_capacity = 1
      max_capacity     = 10
      min_capacity     = 1

      instance_type = "p3.2xlarge"
    }
```

TODO: I should create a simple docker image for hello world type kubernetes deployments. The tutorial also discusses some snippets for ingress ALP and load balancer stuff which is not in scope today. The Helm configuration inline in the terraform is a good recipe though so think about this at some point.

## Multiple environments using modules
Package up with `main.tf` in a folder such as `cluster` and define some variables such as `cluster_name`. then...

```yaml
module "dev_cluster" {
  source        = "./cluster"
  cluster_name  = "dev"
}
```
More variables could easily be refactored out in the `main.tf` e.g. things like machine sizes.


There are some additional details for configuring these e.g. setting up roles and Fargate [here][more-eks-terraform]

## Notes
- IAM Role attachment was needed - for more on [IAM on EKS][IAM-EKS] and how to [trouleshoot][troubleshoot-iam] and check the [policy examples]
- I am not sure what the [aws authenticator][aws-auth-k8s] does exactly. I mean I have read what it does nit i dont know what is required when
-some help to [troubleshoot][auth-issues-eks] identity on eks

[tutorial]: https://learnk8s.io/terraform-eks
[eks-admin-policy-SO]: https://stackoverflow.com/questions/56011492/accessdeniedexception-creating-eks-cluster-user-is-not-authorized-to-perform
[role-create-guide]: https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html#role-create
[aws-auth-k8s]: https://github.com/kubernetes-sigs/aws-iam-authenticator
[auth-issues-aws]: https://aws.amazon.com/premiumsupport/knowledge-center/eks-api-server-unauthorized-error/
[IAM-EKS]: https://docs.aws.amazon.com/eks/latest/userguide/security-iam.html
[trouleshoot-iam]: [https://docs.aws.amazon.com/eks/latest/userguide/troubleshooting_iam.html#security-iam-troubleshoot-cannot-view-nodes-or-workloads]
[policy-examples]:https://docs.aws.amazon.com/eks/latest/userguide/security_iam_id-based-policy-examples.html
[more-eks-terraform]: https://engineering.finleap.com/posts/2020-02-27-eks-fargate-terraform/
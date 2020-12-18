---
layout: post
title:  "eksctl Gitops"
date:   2020-12-17 17:49:28 -0500
categories: DevOps
---

# Overview
As part of my automation platform research, going to play with using `eksctl` gitops. This [tutorial][tutorial] is something to start with.

I use a yaml file to create a demo cluster - it takes 20 minutes so get it going

```yaml
#cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: learn-gitops
  region: us-east-1
nodeGroups:
  - name: worker-group
    instanceType: t2.micro
    desiredCapacity: 3
    minSize: 3
    maxSize: 3
```

```bash
eksctl create cluster -f cluster.yaml
```

_**Note**: I was unable to have max size 5 in us east 1 due to an avail. zone error. I think it should be possible to be explicit with the zones but was not sure about the yaml schema for this to use the file flag route._

The tutorial uses [Flux][flux-home] ([Docs][flux-docs]) to manage cluster sync with gut. Fun. It avoids needing to kubectl things in practice and manages pulling docker images and the like via syncing code to git single source of truth.

I update my config to add the gitops elements and then `eksctl enable repo -f cluster.yaml`

```yaml
#other stuff above
git:
  repo:
    url: "git@github.com:mr-saoirse/cloud-stacks"
    branch: master
    fluxPath: "flux/"
    user: "mr-saoirse"
    email: "amartey@gmail.com"
  operator:
    namespace: "flux"
    withHelm: true
  bootstrapProfile:
  #goodies in the quick start
    source: app-dev
    revision: master
```

Now you may be wondering about auth here and the install process provides a way to do this. It will make an SSH key that can be added to your repo.

From the notes
_Copy the lines starting with ssh-rsa and give it read/write access to your repository. For example, in GitHub, by adding it as a deploy key. There you can easily do this in the Settings > Deploy keys > Add deploy key. Just make sure you check Allow write access as well
_


Quick check what we have on the cluster for flux or else use `--all-namespaces` to see quick start bits we added using our config above under `bootstrapProfile`
```bash
kubectl get pods --namespace flux
```
and if you take a lok at the git repo we can see flux goodies.

Flux polls but you can force the sync action - run on a cluster with flux.
```bash
fluxctl sync --k8s-fwd-ns flux
```

## The Superneat quick-start profile

The config above already has everything we need but we can enable the profile with

```bash
eksctl enable profile app-dev -f cluster.yaml
```
This adds `base` to your repo which you can play around with. Also you can look at kubernetes pods to see everything (in my case i did not see any monitoring pods i.e. prometheus and my cluster scaler was  in a `CrashLoopBackOff state - these are probably due to my scaling limits I put on my cluster though)

```bash
kubectl get pods --all-namespaces
```
I can update my node group policy to something like this

```yaml
nodeGroups:
  - name: ng-1
    instanceType: m5.large
    desiredCapacity: 1
    iam:
      withAddonPolicies:
        albIngress: true
        autoScaler: true
        cloudWatch: true
```

Now i want to delete my cluster
```
eksctl delete cluster --name gitops-1
```

GO again with a new setup
```bash
eksctl create cluster -f cluster.yaml
#eksctl enable repo -f cluster.yaml
eksctl enable profile app-dev -f cluster.yaml
kubectl get pods --all-namespaces
```
## Final thoughts

But what is the best practice to do this properly with git. Is this command smart enough about flux things? Also, in general for example if I built my own profile and commited it, how do I generally bootstrap the process. Im going to leave those questions for another day though because I have a lot to dive into around the basic demo profile. Luckily, the readme on the quick start is deployed to git repo with lots of friendly links to explore!

It wa fun to see what Flux is doing at the moment and if anything the quick start was neat. Now before going to deeply into Flux i want to keep on track with my ArgoCD and Terraform adventures. Useful to see how [Argo is/if-is][argo-flux] better than Flux for doing the same thing. Of course that post is out of date so its important to look at version 2 and more to the point, study roadmaps because what is true today probably will not be true in a month or two.

[tutorial]: https://eksctl.io/gitops-quickstart/
[flux-home]: https://github.com/fluxcd/flux
[flux-docs]: https://toolkit.fluxcd.io/
[argo-flux]: https://luktom.net/en/e1683-argocd-vs-flux
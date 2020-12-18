---
layout: post
title:  "Circle and Argo CD setup"
date:   2020-12-17 17:49:28 -0500
categories: DevOps
summary: Brief
---

# Overview
I review a sample project for setting up Circle CI and Argo for a python application for mrsaoirse. I ha my github, docker hub and circle ci accounts ready to go. I generated a test k8s cluster. I did this loop... create a new repo at github.com, clone it locally, create a dev off the main branch and push back. I added a `python_module` module and put a test folder beside it. At the root of the repo beside the python folder i created `./circleci/config.yml` which is [this project]. Under steps there is a run section for tests (below) and also some stuff that I took from the sample for now to setup the docker context for running in circle CI. I set the working directory to `~/repo` but not sure if this is a cricle thing yet?

## Circle

_**Note** In Visual studio code setup pytests with cmd/ctl + shift + P and `python: configure tests`. I use the root directory as the folder context. Then check tests run locally._

```yaml
  - run:
      name: run tests
      command: |
        python -m pytest test/test_main.py
```
I added a build and push section after the test section (with its steps) - this uses lots of variables for docker creds and other things. Circle CI will allow for these things to be configured.

All the stuff above sits in a jobs section in the yaml. We also add the top-level `workflows` element with ordered sub sections for test build and push.

Add all the changes to the dev branch and push up to the dev branch

```python
git add . & git commit -m "added circle config and tests with minimal module structure"
git push origin dev
```

Now in settings section of Circle CI web UI I added a new context called `DOCKERHUB`. There is a section for env variables. I added
- `DOCKERHUB_USERNAME`
- `DOCKERHUB_PASSWORD`

These were referenced (along with the context in the config). now in Circle CI I can setup my project, use existing config and start building. At first Circle did not see my new dev branch?

_Ok so my circle project file was wrong but as soon as I pushed it, Circle noticed it and errors in it - 'All Branches' were selected so it seemed to know about the dev branch and the config.yml had entries to say build on the dev branch specifically - i.e. it was controlled by config. Well thats nice, thats basically what Circle is supposed to do for us_

-I noted the basic file structure. There are jobs and workflows. Jobs contained elements such as `machine`, `working_directory`, `steps`. Steps contain sequences with things like checkout and run commands
-I needed to not use docker layer caching on my basic Circle CI plan. Im guessing you pay for build spaces here. Thats fine.

# Argo
Setup k8s should follow the official [getting started guide][argo-start].

```bash
kubectl create namespace argocd
#
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v0.9.2/manifests/install.yaml
#
kubectl get pod -n argocd
#port forward the service @ kubectl get svc -n argocd argocd-server <- note the IP>
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

The server takes a moment to be ready but all good! Now setup local tools by downloading like so, and chmod the executable stored in local bin.

```bash
brew install argocd
```

```bash
#get the pod
#kubectl get pods -n argocd -l app=argocd-server -o name | cut -d'/' -f 2
argocd login localhost:8080
#enter admin for password and the server pod for password e.g. argocd-server-bfb8cc86b-s8tn8
#password can be changed with:> argocd account update-password
```

We can add a repo and a project to the ArgoCD tool. A folder in our sample python project has a K8s folder with a sample yaml that will get synced and deployed. I think the url to use in the application is `https://kubernetes.default.svc` - using the `default` namespace choose an app name (not sure it matters but would be expected to match deployment in practice)

I created the application for the repo in Argo and applied a sync. Keep an eye with `kubectl get pods -w` and the browser page says `Progressing`
Setup auto sync of the app with optional `--auto-prune`

```bash
argocd app set APP_NAME --sync-policy automated
```

## TODO when project gets fleshed out more

- Docker image is built and use by Argo
- Play around more with my K8s cluster with load balancing etc. Currently im setup and tearing down which is time consuming but want to make a full prod ready cluster - [inspiration example][weave-prod-guide]
- Work out the proper development cycle for branches, PRs, etc.
- Spend more time with credentials with git

[tutorial]: https://www.digitalocean.com/community/tutorials/webinar-series-gitops-tool-sets-on-kubernetes-with-circleci-and-argo-cd
[python-circle]: https://circleci.com/docs/2.0/language-python/
[gitops-weave]: https://www.weave.works/blog/gitops-operations-by-pull-request
[argo-start]: https://argoproj.github.io/argo-cd/getting_started/
[weave-prod-guide]: https://www.weave.works/blog/provisioning-lifecycle-production-ready-kubernetes-cluster/
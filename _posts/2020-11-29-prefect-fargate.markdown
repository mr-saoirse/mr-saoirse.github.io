---
layout: post
title:  "Prefect with Fargate"
date:   2020-11-25 17:49:28 -0500
categories: overview
---

Assuming a User with permissions and a configured AWS terminal environment follow below. Note that the environment variables on a deve machine for the access key details should match expected user.

Create and log into a Fargate cluster

The [instructions to use AWS CLI to connect ECR][aws-ecr-guide] require a container uri. If you create a repository the registry is part of the URI.

```bash
kubectl config current-context
```

With a user having permission `AWSCloudFormationFullAccess` you can create a Fargate k8s cluster with (eksctl must be installed)
```bash
eksctl create cluster --name fargate-eks --region  <YOUR_AWS_REGION> --fargate
#e.g. eksctl create cluster --name fargate-eks --region  us-east-1 --fargate
```

Create an account at prefect.io and create an access token from the user or team menu item in the side bar. Pip install prefect and set the backend as cloud

```bash
prefect backend cloud
```

Then using the token hash, login

```bash
prefect auth login -t <TOKEN_HASH>
```

create a Fargate runner(scope) token

```bash
prefect auth create-token -n EKS_Fargate_K8s_Token -s RUNNER
```

Use the token to setup Fargate as the runner context

```bash
#https://docs.prefect.io/orchestration/agents/kubernetes.html
prefect agent kubernetes install -t <RUNNER_TOKEN_HASH_FROM_LAST_COMMAND> --rbac | kubectl apply -f -
```
Note that this just generates standard yaml for K8s so this where you can jump in and set up things like ENV or whatever but not obvious to me yet what context flows run in with respect to the agent i.e. flows dont inherit these env variables as far as I know.

Check the Prefect UI e.g. in the cloud account to see the agent AND check the pod is created on K8s.



## Code and project

```bash
prefect create project <PROJECT_NAME>
```

```python
from prefect.environments.storage import Dockerpip install
from prefect import Flow, task
import pandas as pd

#fill in some task functions

@task(log_stdout=True)
def load(y):
    pass

#THE FLOW NAME SHOULD MATCH SOME REGISTERED REGISTRY ON FARGATE
#NOTE - it’s easiest to give your ECR repository the same name as your Prefect flow to avoid confusion.
#aws ecr create-repository --repository-name REGISTRY_FLOW_NAME
with Flow("REGISTRY_FLOW_NAME",
   #if the storage info is excluded the flow will still be scheduled but won't run
          storage=Docker(registry_url="<YOUR_ECR_REGISTRY_ID>.dkr.ecr.eu-central-1.amazonaws.com",
                         python_dependencies=["pandas==1.1.0"],
                         #add secrets - read from cloud or maybe we can set this on the agent PREFECT__CONTEXT__SECRETS__AWS_CREDENTIALS=${AWS_CREDENTIALS}
                         secrets=["AWS_CREDENTIALS"],
                         image_tag='latest')) as flow:
    extracted_df = extract()
    transformed_df = transform(extracted_df)
    load(transformed_df)


if __name__ == '__main__':
    # flow.run()
    flow.register(project_name='Medium_AWS_Prefect')
```

run the flow - make sure to re-run the login of expired

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 065921881757.dkr.ecr.us-east-1.amazonaws.com
```

For more background and to run with a dask cluster, [see the nice post here][prefect-tutorial]

Basically if we import the prefect environment for Dask Kubernetes we can run a new flow on Fargate using map semantics to run parallel...

```python
# We can add a custom executor for DASK if we have a cluster already setup
with Flow("dask-k8-REGISTRY-FLOW") as flow:
    random.seed(123)
    incs = inc.map(x=range(100))
    decs = dec.map(x=range(100))
    adds = add.map(x=incs, y=decs)
    total = list_sum(adds)

if __name__ == '__main__':
    flow.storage = Docker(registry_url="<YOUR_ECR_REGISTRY_ID>.dkr.ecr.eu-central-1.amazonaws.com", image_tag='latest')
    flow.environment = DaskKubernetesEnvironment(min_workers=3, max_workers=5)
    flow.register(project_name="emotif")
```

## Example: Writing data from BBC news to S3

To setup AWS credentials (for S3) such that Prefect can run the flow on Fargate,

Add packages to the Flow registry...
```python
python_dependencies=[
                "pandas==1.1.0",
                "feedparser",
                "boto3",
                "beautifulsoup4"
            ],
```

## Cleanup

```bash
eksctl delete cluster -n fargate-eks --wait
aws ecr delete-repository --repository-name dask-k8
aws ecr delete-repository --repository-name basic-etl-prefect-flow
```

# Final

Check out the amazing Prefect UI to manage projects, check ogs, schedule flows, check agents, manage cloud hooks, manage secrets and keys, look at flow versions and much more...

Questions I have after this trial are

- What is the best dev lifecycle for something that feels more REPL when developing flows. Do secrets need to be added to Docker storage, do they need to be added to cloud UI, do they need to be parsed out to env vars?
- Related to this there is a delay in scheduling flows from the cloud and it would be good to understand why
- What is the best practice for secrets, env variables and parameters for flows and how much can be managed in exec envs directly like Fargate
- How to manage parallel flows in Fargate context with or without thinking about Dask.
- TODO update the pipeline to do things with NLP tasks and knowledge graphs.
 - Can we manage local storage/cache using the agent or otherwise e.g. for storing transform models and not fetching them each time from some store.
 - Try mapping over the stories in a feed
 - Aside: how to use Github hosted code windows in Wordpress and Gitpages

[aws-ecr-guide]: https://docs.aws.amazon.com/AmazonECR/latest/userguide/getting-started-cli.html
[prefect-tutorial]: https://towardsdatascience.com/distributed-data-pipelines-made-easy-with-aws-eks-and-prefect-106984923b30


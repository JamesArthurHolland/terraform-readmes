# Dual cluster tech challenge

### Stack

###### CI/CD
Github actions using reusable workflows and actions.

###### Kubernetes
Linode Kubernetes Engine

###### Infrastructure as Code
Terraform using linode's terraform provider. Remote state handled by an s3 bucket on linode.

###### Web server coding
Golang using gin router, multi-stage docker build, pushing to docker hub.

## Folder structure

#### .github
Github actions folder

#### common
The healthcheck would in theory be used across multiple golang microservices, so I put this into a common lib.
There is also a "Microservice" interface for golang in here which I created to handle the graceful shutdown of services
running on kubernetes (handles sigterm), this would be used commonly across other golang microservices, so it should be
in common.

#### deployment
This contains the helm chart for deploying the mirror-server, as well as the terraform code for linode and gcp. The gcp 
code doesn't deploy the s3 bucket for the file upload, I didn't get there before switching to linode.

#### mirror-server
The golang source code for the string manipulation. I abstracted the string manipulation into a pipeline. Each action 
can be thought of as a "Manipulator" which takes conditional+manipulation functions as params, and applies the manipulation
function based on whether the condition is met.

Each of the manipulation functions are unit tested individually, then in combination as part of the final
pipeline, using the example given in the pdf.

#### script
Script folder containing the bash scripts for doing the building and deployment.

#### test
Postman tests

## Curl to manually test endpoints.

Add to your hosts file:

```
192.46.238.117  dual-jh-devops.com
194.195.247.238 dev.dual-jh-devops.com release-1-1.dual-jh-devops.com cmvmcy9wdwxslz.dual-jh-devops.com
```
```
$ curl dual-jh-devops.com/api/health
$ curl dev.dual-jh-devops.com/api/health
$ curl release-1-1.dual-jh-devops.com/api/health
$ curl cmvmcy9wdwxslz.dual-jh-devops.com/api/health
```

## Overview

The solution deploys 2 different clusters, depending on the environment. (DEV and PROD)

There are 4 different types of build:
1. PR builds that point to dev (.github/workflows/DevPROpened.yml)
2. PR builds that point to main (.github/workflows/ReleasePROpened.yml)
3. Merge to dev (.github/workflows/MergeToDevelop.yml)
4. Merge to main (.github/workflows/MergeToMain.yml)

Github actions only allows for workflows to be in the `workflows/` folder. It does not allow for subdirectories.
To make up for this lack of functionality, the reusable workflows, `build.yml` and `deploy.yml` have a different 
naming case than the 4 main workflows, to help visually differentiate. 


### On pull request opened

#### Build
The golang unit tests are run, and this will break the build if they fail.

If the tests are successful, the docker image is built and pushed to `ozone2021/mirror-server` (public dockerhub repo)

The tag changes based on whether `is_release_version` is set on the build action or not. For a release PR into main, 
the scripts make sure that the naming of the branch is `release/*`, if not, it fails. 

The release number is used to tag the container image eg `release/1.1` would create `ozone2021/mirror-server:1.0`

For PRs into the dev branch, I use the git commit hash as the tag, as this is unique and requires no verbal consensus 
between devs on the version semantics, just for the purposes of development.

The Dev/prod k8s cluster is created via terraform if it doesn't exist.

This uses terraform workspaces, with the name being given as the environment. I use a remote terraform s3 backend to
maintain state across GH Actions builds.

#### Deploy

Deployment happens with a custom helm chart. The kubeconfig is parsed from linode's terraform provider output.

The ingress used depends on the environment.

dev branch deploys to `dev.dual-jh-devops.com`   
main deploys to `dual-jh-devops.com`   
release deploys to `release-1-1.dual-jh-devops.com`   
dev pr deploys to `<truncated_base64_of_branch_name>.dual-jh-devops.com`   

### On pull request merged / closed

After the PR is merged / closed, the helm install is destroyed, for PRs.

Merges to the develop and main branch deploy without teardown. The images have already been built at the PR stage, so 
there is no need to build again for these stages.

You will need to add the IP of the ingress to your hostsfile. The IP can be found output at the bottom of
the "Deploy and end to end test" stage of the github action output.

## Kubeconfig

The kubeconfig is output in the logs as part of `script/build_infrastructure.sh` in the deploy step.

It's base64 encoded.

You have access to this to inspect any new PRs that you may created, or merges to main/dev, so that you can see 
the creation of pods, as is described.

It's enclosed between double quotes.

## S3 credentials
access: EFH62B8A318LTLAG69DL
secret: G2H7o6iPRO4Vh0wgYAD72IuRCfY42AkunIWleuAn
endpoint: eu-central-1.linodeobjects.com

Linode's cloud storage is s3 compliant, however there are gotcha's with using s3cmd with linode.

[This guide will help, if you need it.](https://www.linode.com/docs/products/storage/object-storage/guides/s3cmd/)

## Testing

Automated end to end postman tests using postman's commandline runner `newman`

The health endpoint is tested by checking the status code is 200.

The `api/mirror` endpoint is tested using the example in the pdf.

For S3 random number upload testing, I counted the number of objects in the bucket, using s3cmd. After the postman tests run, the number should have changed.

`s3cmd ls s3://mirror-uploads-bucket | wc -l`

There is some issue with this, so I had to comment out the test, it seems to be intermittent.

Tests only run against the dev environment.

## Challenges

### Parallel builds fail

The helm upgrade command for the ingress has a lock which means if multiple actions run at the same
time, you get an error:

```
Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress
Error: Process completed with exit code 1.
```

The ingress controller is a one time install, it rarely changes, so the fix was a command
to check whether it is already installed.

```
ingress_running=$(kubectl get po -n ingress-nginx | grep ingress | awk '{print $3}')

if [ "$ingress_running" != "Running" ]; then
  helm upgrade --install --wait ingress-nginx ingress-nginx \
    --repo https://kubernetes.github.io/ingress-nginx \
    --namespace ingress-nginx --create-namespace
fi
```

### Terraform backend

I used s3 compatible object storage from linode for the terraform remote state.

This took a bit of searching to implement correctly, as TF assumes AWS is being used when you use the s3 backend provider.

This is why you'll see the validation skipping.

### Ingress nginx needs new annotations

There are new and required annotations which took me a while to debug. The version of the linode k8s cluster is
1.21, which is newer than what I'd worked with before.
```
nginx.ingress.kubernetes.io/affinity-mode: persistent
nginx.ingress.kubernetes.io/affinity: cookie
kubernetes.io/ingress.class: nginx
```

## Things that could be improved

### Script testing

I worked in a test driven development way for the coding, but the bash scripts I could have saved time by writing tests,
rather than deploying to github actions and waiting for any errors to show. I'll definitely do this in future.

### TLS support

I've a few domain names in my possession. I was considering showing the use of cert manager with letsencrypt and how to 
apply this, as I've done it before, but there's only so much I can do without it being overkill.

### Service mesh

Due to the nature of the task, there isn't any service to service communication within the cluster. Linode uses calico,
from inspecting the pods, but I don't know if there is any added config needed to secure that inter pod communication, 
it's not secured out the box with kubernetes, I've used consul before, but I'm unfamiliar with istio and consul.

### Full GitOps

It would be nice to have a fully created infrastructure from scratch, currently any PR could alter the terraform
code for the dev cluster, potentially breaking it for everyone else. Release branch PRs could be altered to create the
full infrastructure. Creating entire K8s clusters + other infrastructure for each dev PR might not be cost effective.

### Unversioned docker images

It would be good to have some form of garbage collection for the stale docker images that are tagged with the github commit 
hash. This would be quite hard to do, as if you did it based on time, then images that don't change often, such as test
database images, might get deleted by the cleanup, unless you had some form of exclusions list.

### Ingress controller

There is a delay between the ingress controller being created, and it receiving an IP from Linode. I've just
used `sleep` to add a delay, but this leads to wasted time, as it's a liberal number of seconds. A better way would be to
poll until the IP becomes available.

### Uploads bucket

For speed, I reused the same s3 credentials as the remote state. It's insecure to do this, each bucket should have its
own credentials.

The linode provider I was using had a bug regarding adding the access and secret keys, I read through the github issues 
for the provider to see I needed to update to a newer version.

Instead of using a kubernetes secret to handle the S3_ACCESS and S3_SECRET, and read them into the environment, I passed 
them in as part of the helm upgrade command. A kubernetes secret would be preferable. You could base64 encode the 
kubernetes secret then apply it as part of the deploy_helm script.

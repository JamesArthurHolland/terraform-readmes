# Company1 tech challenge 

My solution uses github actions, terraform and kubernetes to deploy clusters on demand, using linode's platform.

Github actions work the best with repos hosted on git, in my opinion. They provide hooks for pull requests out the box.
 Their caching has also recently gotten a lot better.

The solution deploys a cluster when a pull request is opened. I've signposted where the test action would run against the cluster.

After the PR is merged, the cluster is destroyed.

Merges to the develop and main branch also build clusters, without teardown.

The PR workflow uses the base64 encoded hash of the branch name as the subdomain for the solution. Except 
for develop, which uses "develop", and main, which doesn't have a subdomain.

You will need to add the IP of the ingress to your hostsfile. The IP can be found output at the bottom of 
the "Build deploy" stage of the github action output.

`<INGRESS IP> base64.companynamewordpress.com`

## Services

### Wordpress

I used the bitnami wordpress helm chart to spin up 2 wordpress pods. It comes with the built-in option of 
adding a memcached pod to act as a caching layer.

`script/deploy_wordpress.sh`

Sticky sessions are achieved through nginx annotations.


### Mariadb

There is a master and 2 slaves. Wordpress can spin up its own, but I think it's preferential to have more access to the parameters 
of mariadb without having to edit the helm chart of wordpress.

`script/deploy_maria_db.sh`

## Challenges

### Terraform backend

I needed to add a backend to terraform for running the automation, otherwise the terraform state would be lost 
between git checkouts of the source.

I initially tried to use kubernetes secrets, but this relies on the cluster already existing. A chicken/egg 
scenario.

I switched to using an s3 compatible object storage from linode.

This took a bit of searching to implement correctly, as TF assumes AWS is being used.

### Disk Volume garbage collection

The volumes which are specified in the helm charts for wordpress and mariadb are bound by the linode CSI driver.

This means they are not torn down by the terraform destroy command.

I had initially overlooked this, thinking that `terraform destroy` would take care of removing all the kubernetes clusters
etc. I realised I needed to delete the helm installs so that the driver could delete the PVCs.

### Ingress nginx needs new annotations

There is a new and required annotation which took me a while to debug. The version of the linode k8s cluster is
1.21, which is newer than what I'd worked with before.

### Wordpress login fails, volume mounting fails

I couldn't work out why the mounting fails was. The mariadb works fine. I'm not sure whether there is a compatibility 
issue with this new k8s version. I had to turn off persistence for the pod to run.

The wordpress page shows, but user login fails. I think this could be related to the lack of disk space.

## Things that could be improved

### Ingress controller

There is a delay between the ingress controller being created, and it receiving an IP from Linode. I've just 
used `sleep` to add a delay, but this leads to wasted time, as it's a liberal number of seconds. A better way would be to
poll until the IP becomes available.

### Secret management

To save time, I hardcoded the passwords into the bash files.

Secrets are how it should be done, I've demonstrated github action secret usage for S3_ACCESS and S3_SECRET.

Hashicorp vault provides better secret management, from what I hear.

Github actions only allows per environment secrets on the highest pricing tier.

On lower pricing tiers, any dev can change the github workflow to output the base64 of a prod secret.

### Terraform backend

The initial bucket for state was setup manually.

### Metrics

I would have liked to have added prometheus + grafana.

### Testing

For the sake of time, I've not done the testing that I would intend to do.

I would normally run health checks and end to end tests using newman.

The health checks would be a good initial indicator of any failures. The end to end tests of the actual application 
would show any inter-service issues, and whether or not the infrastructure change broke the application.

Health checks rely on external endpoints. Not all services have external endpoints. The mariadb for instance.
A health-check container can be run on the cluster, which can then feed back the internal health-check data.

Alternatively you could use `kubectl exec` but that doesn't give a great indication of whether each service is reachable.

I would also run a script to check whether the required wordpress database tables had been instantiated OK.
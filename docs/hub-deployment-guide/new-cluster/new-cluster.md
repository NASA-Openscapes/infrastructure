(new-cluster:new-cluster)=
# New Kubernetes cluster on GCP, Azure or AWS

This guide will walk through the process of adding a new cluster to our [terraform configuration](https://github.com/2i2c-org/infrastructure/tree/HEAD/terraform).

You can find out more about terraform in [](/topic/infrastructure/terraform) and their [documentation](https://www.terraform.io/docs/index.html).

```{attention}
Currently, we do not deploy clusters to AWS _solely_ using terraform.
We use [eksctl](https://eksctl.io/) to provision our k8s clusters on AWS and
terraform to provision supporting infrastructure, such as storage buckets.
```

## Cluster Design

This guide will assume you have already followed the guidance in [](/topic/infrastructure/cluster-design) to select the appropriate infrastructure.

## Prerequisites

`````{tab-set}
````{tab-item} AWS
:sync: aws-key
1. Install `kubectl`, `helm`, `sops`, etc.

   In [](tutorials:setup) you find instructions on how to setup `sops` to
   encrypt and decrypt files.

2. Install [`aws`](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#getting-started-install-instructions)

   Verify install and version with `aws --version`. You should have at least
   version 2.

3. Install or upgrade [eksctl](https://eksctl.io/introduction/#installation)

   Mac users with homebrew can run `brew install eksctl`.

   Verify install and version with `eksctl version`. You typically need a very
   recent version of this CLI.

4. Install [`jsonnet`](https://github.com/google/jsonnet)

   Mac users with homebrew can run `brew install jsonnet`.

   Verify install and version with `jsonnet --version`.
````

````{tab-item} Google Cloud
:sync: gcp-key
1. Install `kubectl`, `helm`, `sops`, etc.

   In [](tutorials:setup) you find instructions on how to setup `sops` to
   encrypt and decrypt files.
````

````{tab-item} Azure
:sync: azure-key
1. Install `kubectl`, `helm`, `sops`, etc.

   In [](tutorials:setup) you find instructions on how to setup `sops` to
   encrypt and decrypt files.
````
`````

## Create a new cluster

### Setup credentials

`````{tab-set}

````{tab-item} AWS
:sync: aws-key
Depending on whether this project is using AWS SSO or not, you can use the following
links to figure out how to authenticate to this project from your terminal.

- [For accounts setup with AWS SSO](cloud-access:aws-sso:terminal)
- [For accounts without AWS SSO](cloud-access:aws-iam:terminal)
````

````{tab-item} Google Cloud
:sync: gcp-key
N/A
````

````{tab-item} Azure
:sync: azure-key
N/A
````
`````

### Generate cluster files

We automatically generate the files required to setup a new cluster:

`````{tab-set}

````{tab-item} AWS
:sync: aws-key
- A `.jsonnet` file for use with `eksctl`
- A `sops` encrypted [ssh key](https://eksctl.io/introduction/#ssh-access) that can be used to ssh into the kubernetes nodes.
- A ssh public key used by `eksctl` to grant access to the private key.
- A `.tfvars` terraform variables file that will setup most of the non EKS infrastructure.
- The cluster config directory in `./config/cluster/<new-cluster>`
- The support values file `support.values.yaml`
- The the support credentials encrypted file `enc-support.values.yaml` 
````

````{tab-item} Google Cloud
:sync: gcp-key
- A `.tfvars` file for use with `terraform`
- The cluster config directory in `./config/cluster/<new-cluster>`
- A sample `cluster.yaml` config file
- The support values file `support.values.yaml`
- The the support credentials encrypted file `enc-support.values.yaml` 
````

````{tab-item} Azure
:sync: azure-key
```{warning}
An automated deployer command doesn't exist yet, these files need to be manually generated!
```
````
`````

You can generate these with:

`````{tab-set}
````{tab-item} AWS
:sync: aws-key

```bash
export CLUSTER_NAME=<cluster-name>
export CLUSTER_REGION=<cluster-region-like ca-central-1>
```

```bash
deployer generate dedicated-cluster aws --cluster-name=$CLUSTER_NAME --cluster-region=$CLUSTER_REGION
```

After running this command, you will be asked to provide the type of hub that will be deployed in the cluster, i.e. `basehub` or `daskhub`.

- If you already know that the there will be daskhubs running in this cluster, then type in `daskhub` and hit ENTER.

  This will generate a specific node pool for dask workers to run on, in the appropriate `.jsonnet` file that will be used with `eksctl`.
- Otherwise, just hit ENTER and it will default to a basehub infrastructure that you can later amend if daskhubs will be needed by following the guide on [how to add support for daskhubs in an existing cluster](howto:features:daskhub).

This will generate the following files:

1. `eksctl/$CLUSTER_NAME.jsonnet` with a default cluster configuration, deployed to `us-west-2`
2. `eksctl/ssh-keys/secret/$CLUSTER_NAME.key`, a `sops` encrypted ssh private key that can be
   used to ssh into the kubernetes nodes.
3. `eksctl/ssh-keys/$CLUSTER_NAME.pub`, an ssh public key used by `eksctl` to grant access to
   the private key.
4. `terraform/aws/projects/$CLUSTER_NAME.tfvars`, a terraform variables file that will setup
   most of the non EKS infrastructure.

### Create and render an eksctl config file

We use an eksctl [config file](https://eksctl.io/usage/schema/) in YAML to specify
how our cluster should be built. Since it can get repetitive, we use
[jsonnet](https://jsonnet.org) to declaratively specify this config. You can
find the `.jsonnet` files for the current clusters in the `eksctl/` directory.

The previous step should've created a baseline `.jsonnet` file you can modify as
you like. The eksctl docs have a [reference](https://eksctl.io/usage/schema/)
for all the possible options. You'd want to make sure to change at least the following:

- Region / Zone - make sure you are creating your cluster in the correct region
  and verify the suggested zones 1a, 1b, and 1c actually are available in that
  region.

  ```bash
  # a command to list availability zones, for example
  # ca-central-1 doesn't have 1c, but 1d instead
  aws ec2 describe-availability-zones --region=$CLUSTER_REGION
  ```
- Size of nodes in instancegroups, for both notebook nodes and dask nodes. In particular,
  make sure you have enough quota to launch these instances in your selected regions.
- Kubernetes version - older `.jsonnet` files might be on older versions, but you should
  pick a newer version when you create a new cluster.

Once you have a `.jsonnet` file, you can render it into a config file that eksctl
can read.

```{tip}
Make sure to run this command inside the `eksctl` directory.
```

```bash
jsonnet $CLUSTER_NAME.jsonnet > $CLUSTER_NAME.eksctl.yaml
```

```{tip}
The `*.eksctl.yaml` files are git ignored as we can regenerate it, so work
against the `*.jsonnet` file and regenerate the YAML file when needed by a
`eksctl` command.
```

### Create the cluster

Now you're ready to create the cluster!

```{tip}
Make sure to run this command **inside** the `eksctl` directory, otherwise it cannot discover the `ssh-keys` subfolder.
```

```bash
eksctl create cluster --config-file=$CLUSTER_NAME.eksctl.yaml
```

This might take a few minutes.

If any errors are reported in the config (there is a schema validation step),
fix it in the `.jsonnet` file, re-render the config, and try again.

Once it is done, you can test access to the new cluster with `kubectl`, after
getting credentials via:

```bash
aws eks update-kubeconfig --name=$CLUSTER_NAME --region=$CLUSTER_REGION
```

`kubectl` should be able to find your cluster now! `kubectl get node` should show
you at least one core node running.

````

````{tab-item} Google Cloud
:sync: gcp-key
```bash
export CLUSTER_NAME=<cluster-name>
export CLUSTER_REGION=<cluster-region-like ca-central-1>
export PROJECT_ID=<gcp-project-id>
```

```bash
deployer generate dedicated-cluster gcp --cluster-name=$CLUSTER_NAME --project-id=$PROJECT_ID --cluster-region=$CLUSTER_REGION
```

After running this command, you will be asked to provide the type of hub that will be deployed in the cluster, i.e. `basehub` or `daskhub`.

- If you already know that the there will be daskhubs running in this cluster, then type in `daskhub` and hit ENTER.

  This will generate a specific node pool for dask workers to run on, in the appropriate `.jsonnet` file that will be used with `eksctl`.
- Otherwise, just hit ENTER and it will default to a basehub infrastructure that you can later amend if daskhubs will be needed by following the guide on [how to add support for daskhubs in an existing cluster](howto:features:daskhub).

This will generate the following files:

Generating the terraform infrastructure file...
1. `terraform/gcp/projects/$CLUSTER_NAME.tfvars`
2. `config/clusters/$CLUSTER_NAME`
3. `config/clusters/$CLUSTER_NAME/cluster.yaml`
4. `config/clusters/$CLUSTER_NAME/support.values.yaml`
5. `config/clusters/$CLUSTER_NAME/enc-support.values.yaml`
````

````{tab-item} Azure
:sync: azure-key

An automated deployer command doesn't exist yet, these files need to be manually generated. The _minimum_ inputs this file requires are:

- `subscription_id`: Azure subscription ID to create resources in.
  Should be the id, rather than display name of the project.
- `resourcegroup_name`: The name of the Resource Group to be created by terraform, where the cluster and other resources will be deployed into.
- `global_container_registry_name`: The name of an Azure Container Registry to be created by terraform to use for our image.
  This must be unique across all of Azure.
  You can use the following [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/) command to check your desired name is available:

  ```bash
  az acr check-name --name ACR_NAME --output table
  ```

- `global_storage_account_name`: The name of a storage account to be created by terraform to use for Azure File Storage.
  This must be unique across all of Azure.
  You can use the following [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/) command to check your desired name is available:

  ```bash
  az storage account check-name --name STORAGE_ACCOUNT_NAME --output table
  ```

- `ssh_pub_key`: The public half of an SSH key that will be authorised to login to nodes.

See the [variables file](https://github.com/2i2c-org/infrastructure/tree/HEAD/terraform/azure/variables.tf) for other inputs this file can take and their descriptions.

```{admonition} Naming Convention Guidelines for Container Registries and Storage Accounts

Names for Azure container registries and storage accounts **must** conform to the following guidelines:

- alphanumeric strings between 5 and 50 characters for [container registries](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/resource-name-rules#microsoftcontainerregistry), e.g., `myContainerRegistry007`
- lowercase letters and numbers strings between 2 and 24 characters for [storage accounts](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/resource-name-rules#microsoftstorage), e.g., `mystorageaccount314`
```

```{note}
A failure will occur if you try to create a storage account whose name is not entirely lowercase.
```

We recommend the following conventions using `lowercase`:

- `{CLUSTER_NAME}hubregistry` for container registries
- `{CLUSTER_NAME}hubstorage` for storage accounts

```{note}
Changes in Azure's own requirements might break our recommended convention. If any such failure occurs, please signal it.
```

This increases the probability that we won't take up a namespace that may be required by the Hub Community, for example, in cases where we are deploying to Azure subscriptions not owned/managed by 2i2c.

Example `.tfvars` file:

```
subscription_id                = "my-awesome-subscription-id"
resourcegroup_name             = "my-awesome-resource-group"
global_container_registry_name = "myawesomehubregistry"
global_storage_account_name    = "myawesomestorageaccount"
ssh_pub_key                    = "ssh-rsa my-public-ssh-key"
```
````
`````

## Add GPU nodegroup if needed

If this cluster is going to have GPUs, you should edit the generated jsonnet file
to [include a GPU nodegroups](howto:features:gpu).

## Initialising Terraform

Our default terraform state is located centrally in our `two-eye-two-see-org` GCP project, therefore you must authenticate `gcloud` to your `@2i2c.org` account before initialising terraform.
The terraform state includes **all** cloud providers, not just GCP.

```bash
gcloud auth application-default login
```

Then you can change into the terraform subdirectory for the appropriate cloud provider and initialise terraform.

`````{tab-set}
````{tab-item} AWS
:sync: aws-key

Our AWS *terraform* code is now used to deploy supporting infrastructure for the EKS cluster, including:

- An IAM identity account for use with our CI/CD system
- Appropriately networked EFS storage to serve as an NFS server for hub home directories
- Optionally, setup a [shared database](features:shared-db:aws)
- Optionally, setup [user buckets](howto:features:storage-buckets)

The steps above will have created a default `.tfvars` file. This file can either be used as-is or edited to enable the optional features listed above.

Initialise terraform for use with AWS:

```bash
cd terraform/aws
terraform init
```
````

````{tab-item} Google Cloud
:sync: gcp-key
```bash
cd terraform/gcp
terraform init -backend-config=backends/default-backend.hcl -reconfigure
```
````

````{tab-item} Azure
:sync: azure-key
```bash
cd terraform/azure
terraform init
```
````
`````

````{note}
There are other backend config files stored in `terraform/backends` that will configure a different storage bucket to read/write the remote terraform state for projects which we cannot access from GCP with our `@2i2c.org` email accounts.
This saves us the pain of having to handle multiple authentications as these storage buckets are within the project we are trying to deploy to.

For example, to work with Pangeo you would initialise terraform like so:

```bash
terraform init -backend-config=pangeo-backend.hcl -reconfigure
```

<!-- TODO: add instructions on how/when to create other backends -->
````

## Creating a new terraform workspace

We use terraform workspaces so that the state of one `.tfvars` file does not influence another.
Create a new workspace with the below command, and again give it the same name as the `.tfvars` filename.

```bash
terraform workspace new WORKSPACE_NAME
```

```{note}
Workspaces are defined **per backend**.
If you can't find the workspace you're looking for, double check you've enabled the correct backend.
```

## Plan and Apply Changes

```{important}
When deploying to Google Cloud, make sure the [Compute Engine](https://console.cloud.google.com/apis/library/compute.googleapis.com), [Kubernetes Engine](https://console.cloud.google.com/apis/library/container.googleapis.com), [Artifact Registry](https://console.cloud.google.com/apis/library/artifactregistry.googleapis.com), and [Cloud Logging](https://console.cloud.google.com/apis/library/logging.googleapis.com) APIs are enabled on the project before deploying!
```

First, make sure you are in the new workspace that you just created.

```bash
terraform workspace show
```

Plan your changes with the `terraform plan` command, passing the `.tfvars` file as a variable file.

```bash
terraform plan -var-file=projects/CLUSTER.tfvars
```

Check over the output of this command to ensure nothing is being created/deleted that you didn't expect.
Copy-paste the plan into your open Pull Request so a fellow 2i2c engineer can double check it too.

If you're both satisfied with the plan, merge the Pull Request and apply the changes to deploy the cluster.

```bash
terraform apply -var-file=projects/CLUSTER.tfvars
```

Congratulations, you've just deployed a new cluster!

## Exporting and Encrypting the Cluster Access Credentials

In the previous step, we will have created an IAM user with just enough permissions for automatic deployment of hubs from CI/CD. Since these credentials are checked-in to our git repository and made public, they should have least amount of permissions possible.

To begin deploying and operating hubs on your new cluster, we need to export these credentials, encrypt them using `sops`, and store them in the `secrets` directory of the `infrastructure` repo.


1. First, make sure you are in the right terraform directory:

    `````{tab-set}
    ````{tab-item} AWS
    :sync: aws-key
      ```bash
      cd terraform/aws
      ```
    ````

    ````{tab-item} Google Cloud
    :sync: gcp-key
      ```bash
      cd terraform/gcp
      ```
    ````

    ````{tab-item} Azure
    :sync: azure-key
      ```bash
      cd terraform/azure
      ```
    ````
    `````

1. Check you are still in the correct terraform workspace

    ```bash
    terraform workspace show
    ```

    If you need to change, you can do so as follows

    ```bash
    terraform workspace list  # List all available workspaces
    terraform workspace select WORKSPACE_NAME
    ```

1. Fetch credentials for automatic deployment

    Create the directory if it doesn't exist already:
    ```bash
    mkdir -p ../../config/clusters/$CLUSTER_NAME
    ```

    `````{tab-set}
    ````{tab-item} AWS
    :sync: aws-key
      ```bash
      terraform output -raw continuous_deployer_creds > ../../config/clusters/$CLUSTER_NAME/deployer-credentials.secret.json
      ```
    ````

    ````{tab-item} Google Cloud
    :sync: gcp-key
      ```bash
      terraform output -raw ci_deployer_key > ../../config/clusters/$CLUSTER_NAME/deployer-credentials.secret.json
      ```
    ````

    ````{tab-item} Azure
    :sync: azure-key
      ```bash
      terraform output -raw kubeconfig > ../../config/clusters/$CLUSTER_NAME/deployer-credentials.secret.yaml
      ```
    ````
    `````

1. Then encrypt the key using `sops`.

    ```{note}
    You must be logged into Google with your `@2i2c.org` account at this point so `sops` can read the encryption key from the `two-eye-two-see` project.
    ```

    ```bash
    sops --output ../../config/clusters/$CLUSTER_NAME/enc-deployer-credentials.secret.json --encrypt ../../config/clusters/$CLUSTER_NAME/deployer-credentials.secret.json
    ```

    This key can now be committed to the `infrastructure` repo and used to deploy and manage hubs hosted on that cluster.

1. Double check to make sure that the `config/clusters/$CLUSTER_NAME/enc-deployer-credentials.secret.json` file is actually encrypted by `sops` before checking it in to the git repo. Otherwise this can be a serious security leak!

    ```bash
    cat ../../config/clusters/$CLUSTER_NAME/enc-deployer-credentials.secret.json
    ```

## Create a `cluster.yaml` file

```{seealso}
We use `cluster.yaml` files to describe a specific cluster and all the hubs deployed onto it.
See [](config:structure) for more information.
```

Create a `cluster.yaml` file under the `config/cluster/$CLUSTER_NAME>` folder and populate it with the following info:

`````{tab-set}

````{tab-item} AWS
:sync: aws-key
```yaml
name: <your-cluster-name>
provider: aws # <copy paste link to sign in url here>
aws:
  key: enc-deployer-credentials.secret.json
  clusterType: eks
  clusterName: <your-cluster-name>
  region: <your-region>
hubs: []
```

```{note}
The `aws.key` file is defined _relative_ to the location of the `cluster.yaml` file.
```
````

````{tab-item} Google Cloud
:sync: gcp-key
```yaml
name: <cluster-name>  # This should also match the name of the folder: config/clusters/$CLUSTER_NAME>
provider: gcp
gcp:
  # The location of the *encrypted* key we exported from terraform
  key: enc-deployer-credentials.secret.json
  # The name of the GCP project the cluster is deployed in
  project: <gcp-project-name>
  # The name of the cluster *as it appears in the GCP console*! Sometimes our
  # terraform code appends '-cluster' to the 'name' field, so double check this.
  cluster: <cluster-name-in-gcp>
  # The GCP zone the cluster in deployed in. For multi-regional clusters, you
  # may have to strip the last identifier, i.e., 'us-central1-a' becomes 'us-central1'
  zone: <gcp-zone>
  billing:
   # Set to true if billing for this cluster is paid for by the 2i2c card
   paid_by_us: true
   bigquery:
    # contains information about bigquery billing export (https://cloud.google.com/billing/docs/how-to/export-data-bigquery)
    # for calculating how much this cluster costs us. Required if `paid_by_us` is
    # set to true.
    project: <id-of-gcp-project-where-bigquery-dataset-lives>
    dataset: <id-of-bigquery-dataset>
    billing_id: <id-of-billing-account-associated-with-this-project>
```

### Billing information

For projects where we are paying the cloud bill & then passing costs through, you need to fill
in information under` gcp.billing.bigquery` and set `gcp.billing.paid_by_us` to `true`. Partnerships
should be able to tell you if we are doing cloud costs pass through or not, and eventually this should
be provided by a single source of truth for all contracts.

1. Going to the [Billing Tab](https://console.cloud.google.com/billing/linkedaccount) on Google Cloud Console
2. Make sure the correct project is selected in the top bar. You might have to select the 'All' tab in the
   project chooser if you do not see the project right away.
3. Click 'Go to billing account'
4. In the default view (Overview) that opens, you can find the value for `billing_id` in the right sidebar,
   under "Billing Account". It should be of the form `XXXXXX-XXXXXX-XXXXXX`.
5. Select "Billing export" on the left navigation bar, and you will find the values for `project` and
   `dataset` under "Detailed cost usage".
6. If "Detailed cost usage" is not set up, you should [enable it](new-gcp-project:billing-export)
````
````{tab-item} Azure (kubeconfig)
:sync: azure-key
```{warning}
We use this config only when we do not have permissions on the Azure subscription
to create a [Service Principal](https://learn.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals)
with terraform.
```
```yaml
name: <cluster-name>  # This should also match the name of the folder: config/clusters/$CLUSTER_NAME
provider: kubeconfig
kubeconfig:
  # The location of the *encrypted* key we exported from terraform
  file: enc-deployer-credentials.secret.yaml
```
````
````{tab-item} Azure (Service Principal)
```yaml
name: <cluster-name>  # This should also match the name of the folder: config/clusters/$CLUSTER_NAME
provider: azure
azure:
  # The location of the *encrypted* key we exported from terraform
  key: enc-deployer-credentials.secret.json
  # The name of the cluster *as it appears in the Azure Portal*! Sometimes our
  # terraform code adjusts the contents of the 'name' field, so double check this.
  cluster: <cluster-name>
  # The name of the resource group the cluster has been deployed into. This is
  # the same as the resourcegroup_name variable in the .tfvars file.
  resource_group: <resource-group-name>
```
````
`````

Commit this file to the repo.

## Access

`````{tab-set}
````{tab-item} AWS
:sync: aws-key

### Grant additional access

First, we need to grant the freshly created deployer IAM user access to the kubernetes cluster.

1. As this requires passing in some parameters that match the created cluster,
  we have a `terraform output` that can give you the exact command to run.

  ```bash
  terraform output -raw eksctl_iam_command
  ```

2. Run the `eksctl create iamidentitymapping` command returned by `terraform output`.
  That should give the continuous deployer user access.

  The command should look like this:

  ```bash
  eksctl create iamidentitymapping \
      --cluster $CLUSTER_NAME \
      --region $CLUSTER_REGION \
      --arn arn:aws:iam::<aws-account-id>:user/hub-continuous-deployer \
      --username hub-continuous-deployer \
      --group system:masters
  ```

  Test the access by running:

  ```bash
  deployer use-cluster-credentials $CLUSTER_NAME
  ```

  and running:

  ```bash
  kubectl get node
  ```

  It should show you the provisioned node on the cluster if everything works out ok.

### Grant `eksctl` access to other users

```{note}
This section is still required even if the account is managed by SSO. Though a
user could run `deployer use-cluster-credentials $CLUSTER_NAME` to gain access
as well.
```

AWS EKS has a strange access control problem, where the IAM user who creates
the cluster has [full access without any visible settings
changes](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html),
and nobody else does. You need to explicitly grant access to other users. Find
the usernames of the 2i2c engineers on this particular AWS account, and run the
following command to give them access:

```{note}
You can modify the command output by running `terraform output -raw eksctl_iam_command` as described in [](new-cluster:aws:terraform:cicd).
```

```bash
eksctl create iamidentitymapping \
   --cluster $CLUSTER_NAME \
   --region $CLUSTER_REGION \
   --arn arn:aws:iam::<aws-account-id>:user/<iam-user-name> \
   --username <iam-user-name> \
   --group system:masters
```

This gives all the users full access to the entire kubernetes cluster.
After this step is done, they can fetch local config with:

```bash
aws eks update-kubeconfig --name=$CLUSTER_NAME --region=$CLUSTER_REGION
```

This should eventually be converted to use an [IAM Role] instead, so we need not
give each individual user access, but just grant access to the role - and users
can modify them as they wish.

[iam role]: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html
````

````{tab-item} Google Cloud
:sync: gcp-key
Test deployer access by running:

```bash
deployer use-cluster-credentials $CLUSTER_NAME
```

and running:

```bash
kubectl get node
```

It should show you the provisioned node on the cluster if everything works out ok.
````

````{tab-item} Azure
:sync: azure-key
Test deployer access by running:

```bash
deployer use-cluster-credentials $CLUSTER_NAME
```

and running:

```bash
kubectl get node
```

It should show you the provisioned node on the cluster if everything works out ok.
````
`````

## Adding the new cluster to CI/CD

To ensure the new cluster is appropriately handled by our CI/CD system, please add it as an entry in the following places:

- The [`deploy-hubs.yaml`](https://github.com/2i2c-org/infrastructure/blob/008ae2c1deb3f5b97d0c334ed124fa090df1f0c6/.github/workflows/deploy-hubs.yaml#L121) GitHub workflow has a job named []`upgrade-support-and-staging`] that need to list of clusters being
automatically deployed by our CI/CD system. Add an entry for the new cluster
here.

  [`upgrade-support-and-staging`]: https://github.com/2i2c-org/infrastructure/blob/18f5a4f8f39ed98c2f5c99091ae9f19a1075c988/.github/workflows/deploy-hubs.yaml#L128-L166

## Cluster is now ready, what are the next steps

```{important}
Cluster is now ready to perform the next steps:
1. [](support-components).
2. [](new-hub)
```
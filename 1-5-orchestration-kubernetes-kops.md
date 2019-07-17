# Part 5: Creating a Kubernetes cluster with `kops` 

### Kubernetes and Kops

If you've done any sort of DevOps work lately, or probably software-development in general, you have likely heard of [Kubernetes](https://kubernetes.io/).
Kubernetes is not the only [container orchestration system](https://blog.newrelic.com/engineering/container-orchestration-explained/), but it's become the most important.
Here are some [fun K8s facts](https://dzone.com/articles/10-basic-facts-about-kubernetes-that-you-didnt-kno).

I'm not going to concentrate on teaching you about Kubernetes, but rather demonstrate how to deploy our application to a Kubernetes cluster in the cloud.

Before we can deploy our applicaiton to a K8s cluster, we have to deploy said K8s cluster.
There are lots of ways to do this.
We could use a hosted service like [Amazon's EKS](https://aws.amazon.com/eks/) or [Google's GKE](https://cloud.google.com/kubernetes-engine/).

I might create another tutorial using a hosted service later on, but it's easy to do, and would preclude the rest of this tutorial.
Here, though, I want to show how to generate Terraform code to build a self-managed cluster in AWS.

So in this tutorial we are going to use [kops](https://kubernetes.io/docs/setup/production-environment/tools/kops/).
More specifically, we are going to use `kops` to create [Terraform](https://www.terraform.io/) code which we will then use to create our cluster, and check into source control.
This is far from the only way to solve this problem, but it is a useful one for a number of reasons:

- This method works well if you only need a few clusters, and we only need one.
- kops is easy to use, yet makes it easy to rebuild a cluster from scratch if needed.
- The code that defines our cluster can be checked into source control, but we don't have to build it by hand.
- This method works well for learning a bit about Kubernetes, because you can look through the Terraform code to see what goes into building a K8s cluster.

There are a couple of downsides to this approach:

- The `kops create` command is not [idempotent](https://en.m.wikipedia.org/wiki/Idempotence) and therefore is difficult to integrate into a CI pipeline. This tutorial only partial solves the problem of CI-managed orchestration infrastructure.
- Managing more than a handful of clusters this way quickly becomes difficult and error-prone.

To solve these problems, we could use the powerful set of tools from [Rancher](https://rancher.com/).
These seem very promising, and I plan to build a Rancher-based tutorial in the future.
For now I want to focus at a slightly lower level.

You can read a bit more about this approach [here](https://medium.com/bench-engineering/deploying-kubernetes-clusters-with-kops-and-terraform-832b89250e8e).

### Complete Parts 1-4

Make sure you've completed the parts of the tutorial up to this point.

I'll assume that you still have all the files in place from the previous exercises, contained in your `source` directory at the top level of your local clone of the `devops-toolkit` [repository](https://github.com/sloanahrens/devops-toolkit).

Your `source` directory should have at least the following structure:

```
devops-toolkit/
    source/
        .circleci/
            config.yml
        container_environments/
            test-stack.yaml
        django/
            stockpicker/
                ...
            requirements.txt
        docker/
            baseimage/
                Dockerfile
            celery/
                Dockerfile
            scripts/ 
                initialize-webapp-prod.sh
                wait-for-it.sh
            stacktest/
                Dockerfile
                integration-tests.sh
            webapp/
                Dockerfile
            docker-compose-local-dev-django.yaml
            docker-compose-local-dev-django-celery.yaml
            docker-compose-local-dev-django.html-celery-beat.yaml
            docker-compose-local-image-stack.yaml
            docker-compose-tagged-image-stack.yaml
            docker-compose-unit-test.yaml
        .dockerignore
        .gitignore
```

### `devops` Development Environment

In this exercise we'll use the `devops` development environment, so make sure you can run it as described [here](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/0-local-dev-env-devops.md).

Make sure you have built the `devops` Docker image, by running the following command from your `devops-toolkit` directory:

```bash
docker build -t devops -f docker/devops/Dockerfile .
```

Go to your `source` directory now, and and start the development environment Docker container with:

```bash
docker run -it \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v $PWD:/src \
    --rm \
    --name \
    local_devops \
    devops \
    /bin/bash
```

Remember that in this command, the argument `-v $PWD:/src` connects the directory on the host OS from which you _run_ the command, to the `/src` directory inside the container.

Run `ls` from inside the container and should see the files you created in the `source` directory:

```
root@1bc94b877039:/src# ls
container_environments	django	docker
```

### Environment variables

The `devops` container environment already has `kops` installed, so we can run the `kops create` command once we set up a few environment variables.

In [Part 4](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-4-ci-circleci-aws.md) you will have created an AWS IAM user, for which you should have credentials.
We need to add those credentials to the environment with (replace "REPLACE" with your user's key values):

```bash
export AWS_ACCESS_KEY_ID=REPLACE
export AWS_SECRET_ACCESS_KEY=REPLACE
```

We also need a few more variables.
For the purposes of this exercise, I'm going to use the AWS `us-west-2` (Oregon) region.
The code already in the `devops-toolkit` repo uses `us-east-2`.

If the AWS region you are using is different than `us-west-2`, you'll need to replace it in what follows.

```bash
export REGION=us-west-2
export ZONES=${REGION}b
export CLUSTER_NAME=devops-toolkit-${REGION}.k8s.local
export BUCKET_NAME=devops-toolkit-k8s-state-${REGION}
```

### Create directories

We need to create a series of directories for what follows, and we may as well do it all at once.
Make sure you are working in your `source` directory, from the `devops` container environment, and run:

```bash
mkdir -p kubernetes/specs && \
mkdir -p kubernetes/$REGION/specs && \
mkdir -p kubernetes/$REGION/cluster && \
mkdir -p kubernetes/$REGION/remote_state
```

### Terraform remote state

We're going to set up [remote state](https://www.terraform.io/docs/providers/terraform/d/remote_state.html) for our Terraform-based deployment of a Kubernetes cluster.
This solves the problem of carrying sensitive [terraform state files](https://stackoverflow.com/questions/38486335/should-i-commit-tfstate-files-to-git) around in a public GitHub repository, for one thing.

If you want to learn more about Terraform, check out [the docs](https://www.terraform.io/intro/index.html).

If you want to lear more about how I set up remote state, check out [this article](https://blog.itp-inc.com/how-to-terraform-locking-state-in-s3-amazon-web-services/).

Terraform has already been installed in the `devops` container, so we won't need to install it.

Let's go ahead and create our first Terraform code file.
My value of `$REGION` is `us-west-2`, so I'm going to use `us-west-2` in what follows.
Using the region-name hard-coded in the directory structure makes it easier to adapt this method to handle multiple clusters.
If you are using a different region, you will need to update the directory and all the file paths that follow, accordingly.
The AWS region is also used in the terraform code that follows, so you'll need to update that with your region as well (it's used in several places).

Create `source/kubernetes/us-east-2/remote_state/remote_state_resources.tf`:

```hcl-terraform
provider "aws" {
  region = "us-west-2"
}

resource "aws_s3_bucket" "terraform-state-storage-us-west-2" {
    bucket = "terraform-state-storage-us-west-2"

    versioning {
      enabled = true
    }

    lifecycle {
      prevent_destroy = true
    }

    tags {
      Name = "S3 Remote Terraform State Store us-west-2"
    }
}

# create a dynamodb table for locking the state file
resource "aws_dynamodb_table" "dynamodb-terraform-state-lock-us-west-2" {
  name = "terraform-state-lock-dynamo-us-west-2"
  hash_key = "LockID"
  read_capacity = 20
  write_capacity = 20

  attribute {
    name = "LockID"
    type = "S"
  }

  tags {
    Name = "DynamoDB Terraform State Lock Table for us-west-2"
  }
}

```

Now we need to go to the `remote_state` directory in the `devops` environment, so run:

```bash
cd /src/kubernetes/$REGION/remote_state
```

Now we will initialize terraform, and run `terraform plan` to see what it's going to build for us:

```bash
terraform init && \
terraform plan
```

You should see output similar to:

```
root@8655a7cde2c0:/src/kubernetes/us-west-2/remote_state# terraform init && \
> terraform plan

Initializing provider plugins...
- Checking for available provider plugins on https://releases.hashicorp.com...
- Downloading plugin for provider "aws" (2.19.0)...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.aws: version = "~> 2.19"

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.


------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + aws_dynamodb_table.dynamodb-terraform-state-lock-us-west-2
      id:                          <computed>
      arn:                         <computed>
      attribute.#:                 "1"
      attribute.2068930648.name:   "LockID"
      attribute.2068930648.type:   "S"
      billing_mode:                "PROVISIONED"
      hash_key:                    "LockID"
      name:                        "terraform-state-lock-dynamo-us-west-2"
      point_in_time_recovery.#:    <computed>
      read_capacity:               "20"
      server_side_encryption.#:    <computed>
      stream_arn:                  <computed>
      stream_label:                <computed>
      stream_view_type:            <computed>
      tags.%:                      "1"
      tags.Name:                   "DynamoDB Terraform State Lock Table for us-west-2"
      write_capacity:              "20"

  + aws_s3_bucket.terraform-state-storage-us-west-2
      id:                          <computed>
      acceleration_status:         <computed>
      acl:                         "private"
      arn:                         <computed>
      bucket:                      "terraform-state-storage-us-west-2"
      bucket_domain_name:          <computed>
      bucket_regional_domain_name: <computed>
      force_destroy:               "false"
      hosted_zone_id:              <computed>
      region:                      <computed>
      request_payer:               <computed>
      tags.%:                      "1"
      tags.Name:                   "S3 Remote Terraform State Store us-west-2"
      versioning.#:                "1"
      versioning.0.enabled:        "true"
      versioning.0.mfa_delete:     "false"
      website_domain:              <computed>
      website_endpoint:            <computed>


Plan: 2 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.

```

We haven't created anything yet.
Now let's do that, by running

```bash
terraform apply --auto-approve
```

You should see output similar to:

```
root@8655a7cde2c0:/src/kubernetes/us-west-2/remote_state# terraform apply --auto-approve
aws_s3_bucket.terraform-state-storage-us-west-2: Creating...
  acceleration_status:         "" => "<computed>"
  acl:                         "" => "private"
  arn:                         "" => "<computed>"
  bucket:                      "" => "terraform-state-storage-us-west-2"
  bucket_domain_name:          "" => "<computed>"
  bucket_regional_domain_name: "" => "<computed>"
  force_destroy:               "" => "false"
  hosted_zone_id:              "" => "<computed>"
  region:                      "" => "<computed>"
  request_payer:               "" => "<computed>"
  tags.%:                      "" => "1"
  tags.Name:                   "" => "S3 Remote Terraform State Store us-west-2"
  versioning.#:                "" => "1"
  versioning.0.enabled:        "" => "true"
  versioning.0.mfa_delete:     "" => "false"
  website_domain:              "" => "<computed>"
  website_endpoint:            "" => "<computed>"
aws_dynamodb_table.dynamodb-terraform-state-lock-us-west-2: Creating...
  arn:                       "" => "<computed>"
  attribute.#:               "" => "1"
  attribute.2068930648.name: "" => "LockID"
  attribute.2068930648.type: "" => "S"
  billing_mode:              "" => "PROVISIONED"
  hash_key:                  "" => "LockID"
  name:                      "" => "terraform-state-lock-dynamo-us-west-2"
  point_in_time_recovery.#:  "" => "<computed>"
  read_capacity:             "" => "20"
  server_side_encryption.#:  "" => "<computed>"
  stream_arn:                "" => "<computed>"
  stream_label:              "" => "<computed>"
  stream_view_type:          "" => "<computed>"
  tags.%:                    "" => "1"
  tags.Name:                 "" => "DynamoDB Terraform State Lock Table for us-west-2"
  write_capacity:            "" => "20"
aws_s3_bucket.terraform-state-storage-us-west-2: Creation complete after 8s (ID: terraform-state-storage-us-west-2)
aws_dynamodb_table.dynamodb-terraform-state-lock-us-west-2: Still creating... (10s elapsed)
aws_dynamodb_table.dynamodb-terraform-state-lock-us-west-2: Creation complete after 10s (ID: terraform-state-lock-dynamo-us-west-2)

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

Congrats!
You have now programmatically allocated cloud infrastructure (if you haven't before).

We will use it in the next section.

### Pre-reqs for `kops create`

We will now need to be working in the `cluster` directory, so (from the running `devops` container) run:

```bash
cd /src/kubernetes/$REGION/cluster
```

In order to use the Terraform remote state infrastructure we just created in the last section, we need to create another file.

Create `source/kubernetes/cluster/remote_state.tf` (replace `us-west-2` if needed):

```hcl-terraform
terraform {
  backend "s3" {
    encrypt = true
    bucket = "terraform-state-storage-us-west-2"
    dynamodb_table = "terraform-state-lock-dynamo-us-west-2"
    region = "us-west-2"
    key = "terraform.tfstate"
  }
}
```

Before we can create a cluster we need to [create a key pair with the AWS CLI](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-key-pair.html).

So from the `/src/kubernetes/$REGION/cluster` directory, from the `devops` container, run:

```bash
aws ec2 create-key-pair --key-name $REGION --region $REGION | jq -r '.KeyMaterial' >$REGION.pem
chmod 400 $REGION.pem
ssh-keygen -y -f $REGION.pem >$REGION.pub
```

(Be sure your `.gitignore` file contains the line `*.pem` so your key doesn't get compromised by being accidentally pushed to a public GitHub repository.)

Next we need to create an [AWS S3 Bucket](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingBucket.html) to store our [kops state file]().

Run this command:

```bash
aws s3api create-bucket \
    --region=$REGION \
    --bucket $BUCKET_NAME \
    --create-bucket-configuration \
    LocationConstraint=$REGION
```

You should see output similar to:

```
root@8655a7cde2c0:/src/kubernetes/us-west-2/cluster# aws s3api create-bucket \
>     --region=$REGION \
>     --bucket $BUCKET_NAME \
>     --create-bucket-configuration \
>     LocationConstraint=$REGION
{
    "Location": "http://devops-toolkit-k8s-state-us-west-2.s3.amazonaws.com/"
}
```

### `kops create`

We're finally ready to run the `kops create` command.
The way we're going to run it, it won't actually create the infrastructure, it will just create terraform code for us.

So run:

```bash
kops create cluster \
    --cloud=aws \
    --name $CLUSTER_NAME \
    --state s3://$BUCKET_NAME \
    --master-count 1 \
    --node-count 2 \
    --node-size t2.medium \
    --master-size c4.large \
    --zones $ZONES \
    --master-zones $ZONES \
    --ssh-public-key $REGION.pub \
    --authorization RBAC \
    --kubernetes-version v1.11.9 \
    --topology private \
    --networking calico \
    --out=. \
    --target=terraform
```

```
...
kops has set your kubectl context to devops-toolkit-us-west-2.k8s.local

Terraform output has been placed into .
Run these commands to apply the configuration:
   cd .
   terraform plan
   terraform apply

Suggestions:
 * validate cluster: kops validate cluster
 * list nodes: kubectl get nodes --show-labels
 * ssh to the master: ssh -i ~/.ssh/id_rsa admin@api.devops-toolkit-us-west-2.k8s.local
 * the admin user is specific to Debian. If not using Debian please use the appropriate user based on your OS.
 * read about installing addons at: https://github.com/kubernetes/kops/blob/master/docs/addons.md.
```

This will have created a file called `kubernetes.tf` and a folder called `data`.
We will update one of the files in the `data` folder shortly, but first we need to do a few more things.

Take a look though `source/kubernetes/us-west-2/kubernetes.tf`

Next we need to initialize Terraform in the new directory (`source/kubernetes/$REGION/cluster`), with:

```bash
terraform init
```

You should see:

```
root@8655a7cde2c0:/src/kubernetes/us-west-2/cluster# terraform init

Initializing the backend...

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Checking for available provider plugins on https://releases.hashicorp.com...
- Downloading plugin for provider "aws" (2.19.0)...

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, it is recommended to add version = "..." constraints to the
corresponding provider blocks in configuration, with the constraint strings
suggested below.

* provider.aws: version = "~> 2.19"

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Now we can run `terraform plan` to see what all will be created for us.

You should see output that ends with:

```
...
Plan: 48 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.

Releasing state lock. This may take a few moments...

```

Now we can finally actually create our infrastructure, with `terraform apply`.

Run:

```bash
terraform apply --auto-approve
```

It will take a little while for this command to finish.

Your output should end with:

```
...

```

### Update, add some resources

Next we are going to need to finish allocating our cluster by running a "rolling update", for reasons explained [here]().

First run these three commands:

```bash
export KUBECONFIG=$PWD/kubecfg.yaml

kops export kubecfg --name $CLUSTER_NAME --state s3://$BUCKET_NAME

kops update cluster $CLUSTER_NAME --state s3://$BUCKET_NAME --target=terraform --out=.

```

Next we need to add a resource that will help provide persistence for our application stacks in Kubernetes.
There are a number of ways to handle this problem, and I have found Amazon [EFS]() to be a handy way to solve the problem.
It works well as long as you don't have a large amount of data, and we won't.

Create the following file, replacing `us-west-2` with your region as needed.

`source/kubernetes/$REGION/cluster/efs.tf`:

```hcl-terraform
resource "aws_efs_file_system" "efs-us-west-2-k8s-local" {
}

resource "aws_efs_mount_target" "us-west-2b-us-west-2-k8s-local" {
  file_system_id = "${aws_efs_file_system.efs-us-west-2-k8s-local.id}"
  subnet_id      = "${aws_subnet.us-west-2b-devops-toolkit-us-west-2-k8s-local.id}"

  security_groups = [
    "${aws_security_group.nodes-devops-toolkit-us-west-2-k8s-local.id}",
  ]
}

output "efs_id" {
  value = "${aws_efs_file_system.efs-us-west-2-k8s-local.id}"
}
```

Finally we need to update the file that sets the IAM permissions for our Kubernetes user.

Edit `source/kubernetes/$REGION/cluster/data/aws_iam_role_policy_nodes.devops-toolkit-us-west-2.k8s.local_policy` to match:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeRegions"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:GetEncryptionConfiguration",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::devops-toolkit-k8s-state-us-west-2"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:Get*"
      ],
      "Resource": [
        "arn:aws:s3:::devops-toolkit-k8s-state-us-west-2/devops-toolkit-us-west-2.k8s.local/addons/*",
        "arn:aws:s3:::devops-toolkit-k8s-state-us-west-2/devops-toolkit-us-west-2.k8s.local/cluster.spec",
        "arn:aws:s3:::devops-toolkit-k8s-state-us-west-2/devops-toolkit-us-west-2.k8s.local/config",
        "arn:aws:s3:::devops-toolkit-k8s-state-us-west-2/devops-toolkit-us-west-2.k8s.local/instancegroup/*",
        "arn:aws:s3:::devops-toolkit-k8s-state-us-west-2/devops-toolkit-us-west-2.k8s.local/pki/issued/*",
        "arn:aws:s3:::devops-toolkit-k8s-state-us-west-2/devops-toolkit-us-west-2.k8s.local/pki/private/kube-proxy/*",
        "arn:aws:s3:::devops-toolkit-k8s-state-us-west-2/devops-toolkit-us-west-2.k8s.local/pki/private/kubelet/*",
        "arn:aws:s3:::devops-toolkit-k8s-state-us-west-2/devops-toolkit-us-west-2.k8s.local/pki/ssh/*",
        "arn:aws:s3:::devops-toolkit-k8s-state-us-west-2/devops-toolkit-us-west-2.k8s.local/secrets/dockerconfig"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:Get*"
      ],
      "Resource": "arn:aws:s3:::devops-toolkit-k8s-state-us-west-2/devops-toolkit-us-west-2.k8s.local/pki/private/calico-client/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:GetRepositoryPolicy",
        "ecr:DescribeRepositories",
        "ecr:ListImages",
        "ecr:BatchGetImage"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListResourceRecordSets",
        "route53:GetHostedZone"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/Z1CDZE44WDSMXZ"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:GetChange"
      ],
      "Resource": [
        "arn:aws:route53:::change/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "elasticfilesystem:DescribeFileSystems",
        "elasticfilesystem:CreateFileSystem",
        "elasticfilesystem:CreateTags",
        "elasticfilesystem:DescribeMountTargets",
        "elasticfilesystem:CreateMountTarget",
        "ec2:DescribeSubnets",
        "ec2:DescribeNetworkInterfaces",
        "ec2:CreateNetworkInterface"
      ],
     "Resource": "*"
    }
  ]
}
```

### Rolling kops update

Now we need to run our terraform commands again.

Run:

```bash
terraform init \
  && terraform plan
```

Your output should end with:

```
...
Plan: 2 to add, 1 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.

Releasing state lock. This may take a few moments...
```

Now update the cloud infrastructure with:

```bash
terraform apply --auto-approve
```

Now we need to run [rolling update]() with the following command:

```bash
kops rolling-update cluster --name $CLUSTER_NAME --state s3://$BUCKET_NAME --cloudonly --force --yes
```

This command will take awhile to finish, especially if you added more masters or nodes to the `kops create` command earlier.

Once it finally finishes, run the following:

```bash
kops validate cluster --name $CLUSTER_NAME --state s3://$BUCKET_NAME
```

Your output should look similar to:

```
root@8655a7cde2c0:/src/kubernetes/us-west-2/cluster# kops validate cluster --name $CLUSTER_NAME --state s3://$BUCKET_NAME
Validating cluster devops-toolkit-us-west-2.k8s.local

INSTANCE GROUPS
NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
master-us-west-2b	Master	c4.large	1	1	us-west-2b
nodes			Node	t2.medium	2	2	us-west-2b

NODE STATUS
NAME						ROLE	READY
ip-172-20-41-131.us-west-2.compute.internal	node	True
ip-172-20-49-53.us-west-2.compute.internal	master	True
ip-172-20-58-98.us-west-2.compute.internal	node	True

Your cluster devops-toolkit-us-west-2.k8s.local is ready
```

If it does not, something went wrong.
What I do in _that_ situation is delete the cluster and start over, and make sure I followed all the steps exactly and in order.

If things went well, contrats!
You're now the proud owner of a Kubernetes cluster.
Here are [some things you can do with it](https://kubernetes.io/docs/reference/kubectl/cheatsheet/).

### Deleting the cluster when you are finished with it

When you are finished with your K8s cluster in AWS, you should deleted it, because it will cost you money.

To delete the cluster, run:

```bash
terraform destroy --auto-approve && \
kops delete cluster --name $CLUSTER_NAME --state s3://$BUCKET_NAME --yes
```

You can find all of these instructions (for region `us-east-2`) in the `devops-toolkit` repo, [here](https://github.com/sloanahrens/devops-toolkit/blob/master/kubernetes/us-east-2/cluster/kops_instructions.txt).

In the next exercise we will integrate the maintenance of our running cluster into CI, and install some K8s dependencies our app stacks will need.

[Prev: Part 4](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-4-ci-circleci-aws.md)
[Next: Part 6](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-6-ci-cluster-maintenance.md)

# The Polybot Service: Terraform Project

## Background and goals

In the previous Kubernetes, you've provisioned a scalable, highly available and well-architected Polybot service using cloud resources. 

This project focuses on Infrastructure as Code (IaC) techniques in order to automate the provisioning of the cloud resources for your Polybot service **in different AWS regions**. 

Why is this good? Here are just three reasons among many...

- Your infrastructure configurations would be versioned as part of your git repo, allows you to track changes and revert to previous versions if needed.
- Let's say compliance regulations require to provision your service across 6 different regions on the globe. You'll be able to do it in a click of a mouse, while ensuring consistency between regions. 
- You'll significantly improve your readiness to [Disaster Recovery (RD)](https://cloud.google.com/learn/what-is-disaster-recovery).

## Preliminaries

No need to fork this repo as it doesn't contain code resources. 
You can work on the same repo used in the previous Kubernetes project.

> [!WARNING]
> This project involves working with multiple AWS services. 
> Please proceed with caution and ensure that you understand the implications of provisioning resources using Terraform. 
> Always double-check your Terraform configurations before applying changes, and use `terraform destroy` when necessary to tear down resources and avoid additional charges.

Let's get started...


## Configuration files - structure and usage

In your **Polybot Infra** repo, here is a general structure for the Terraform root module directory:

```text
tf/
├── modules/
├───── k8s-cluster/                         # Module for k8s cluster related resources (instances, sg, iam, asg, lb, etc..)
│      ├── main.tf
│      ├── variables.tf
│      └── outputs.tf
├───── polybot/                           # Module for polybot and yolo5 related resources (sqs, s3, etc, secret manager...)
│      ├── main.tf
│      ├── variables.tf
│      └── outputs.tf
├── main.tf                         # Main configuration file
├── outputs.tf
├── variables.tf
├── region.us-east-1.tfvars         # Values related for us-east-1 region
└── region.eu-central-1.tfvars      # Values related for eu-central-1 region
```

Once your work is completed, you should be able to provision (and destroy) the Polybot service as follows:


1. Select (or create of doesn't exist) the workspace in which you want to provision the infrastructure:

```bash
terraform workspace select us-east-1 || terraform workspace new us-east-1
```

2. Provision the service with region-specific values file and variables:

```bash
# Terraform will use the value from the environment variable TF_VAR_botToken for the botToken variable.
export TF_VAR_botToken="123456789:ABCDefghIJKLmnopQRStuvWXYZ1234567890"
terraform apply -var-file=region.us-east-1.tfvars
```

Wait for the service until it's operational...

3. When done, **destroy all resources to avoid unnecessary costs**.

```bash
terraform destroy -var-file=region.us-east-1.tfvars
```

## General notes

- As the above configuration files structure suggests, the `k8s-cluster` and `polybot` should be configured as separate **Terraform Modules**. 
- Since Terraform can provision and destroy the same infrastructure consistently, avoid additional charges and don't keep your AWS resources existing for long time. **Destroy them when you don't work on your project**. 
- Don't store sensitive data in the `.tf` or `.tfvars` files.
- Feel free to use other Terraform modules (e.g. for VPC, ASG, etc...).
- If you work properly, you should be able to provision your service in a new fresh AWS account **that you have never seen before, in any arbitrary region**!
  Thus, make sure that all of your service resources are managed by Terraform **without exceptions**. You shouldn't do any manual step to make it work.
  
  For example, you cannot rely on a dedicated AMI that you created beforehand which contains the application code and dependencies, as on a new AWS account, those AMI would not be available.

## Manage state across multiple regions

The below image demonstrates the Polybot Service deployed in 6 different regions.
Each region has its own Telegram bot token and app URL, but resource-wise, the regions are identical, they are deployed using the same stack of resources in AWS.
The different `.tfstate` files of all regions are stored in the same S3 bucket, located in one of the regions (`us-east-1` in this case). 

![][tf_project_regions]

Choose one of your regions to store the `.tfstate` files for all workspaces in a dedicated [S3 bucket backend](https://developer.hashicorp.com/terraform/language/settings/backends/s3). 

## Applying region-specific values per region 

The `region.REGION_CODE.tfvars` files should contain region-related values.
As mentioned above, once you want to apply the configuration for a given region, provide the corresponding `.tfvars` in the `terraform apply` command:

```bash
terraform apply -var-file=region.us-east-1.tfvars 
```

All your `.tf` configuration files should be region-agnostic, which means, you have to avoid hardcoding specific region-related settings, such as AMIs, availability zones, etc...

To avoid large charges in AWS, test the service provisioning in **2 different regions only** (and once it works, destroy it).
Generalizing it to more regions should be as simple as creating the corresponding `.tfvars` file for the region, as well as generating a new Telegram token. 

## Deploy the Polybot and the Yolo5 services in the cluster

As you may know, in order for the service to be up and running, it's necessary not only to provision the underlying infrastructure, but also to deploy the Polybot and Yolo5 apps within the provisioned Kubernetes cluster. 

Feel free to design your own solution, but you are highly encouraged to leverage the CI/CD pipeline developed in the previous project for this purpose.

The workflow is as follows: 

1. Terraform is used to provision the infrastructure (leaving the k8s cluster empty):

```bash
terraform apply ....
```

2. (Optional) Trigger a CI/CD pipeline to bootstrap the cluster (`kubeadm init`, install Calico CNI, install nginx Ingress controller, install ArgoCD with its apps). This is an optional step, you are free to implement the cluster bootstrapping in any other way (e.g. by trigger a Lambda function immediately after the cluster infrastructure provisioned).
3. Then, trigger the CI/CD pipeline from previous project (with necessary adjustments) to deploy the initial app version onto the cluster.

## Integrate a simple CI/CD pipeline for Terraform 

Similarly to the CI/CD pipelines used to deploy a new version of the service, we also want to automate the provisioning process of AWS resources done by Terraform. 

When you change some of your `.tf` configuration files locally, then commit and push them, a **semi-automated** pipeline would provision the new infrastructure changes in AWS. 
By semi-automated we mean that the actual infrastructure provisioning would be triggered **manually**, since it's important to have a human approval step before making changes to the product infrastructure.

You're provided with three GitHub Actions workflows:

- `.github/workflows/infra-provisioning-main.yaml` - This is the main pipeline. It should be **manually triggered** from the GitHub Actions tab, while selecting the regions where you want to provision the infrastructure.
- `.github/workflows/infra-provisioning-region.yaml` - This workflow is automatically called by the main pipeline and provisions the infrastructure for a specific region.
- `.github/workflows/infra-destroying.yaml` - To be used to destroy your infrastructure.

Feel free to use these pipelines as a starting point for your work. 

Test the pipeline by making changes to your Terraform configurations and observing the provisioning process in AWS.

## Improvements to the Polybot service in the cluster level

#### Make the MongoDB persistent with EBS-CSI plugin 

The EBS [Container Storage Interface (CSI)](https://github.com/container-storage-interface/spec) driver create and attach EBS volumes as storage for your Pods. 

Install the addon from the [official project repo](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/install.md).

Add the aws-ebs-csi-driver Helm repository.
 
```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
```

Create a values file as follows:

```yaml
# ebs-csi-values.yaml

storageClasses:
  - name: ebs-sc
    annotations:
      storageclass.kubernetes.io/is-default-class: "true"
    provisioner: ebs.csi.aws.com
    volumeBindingMode: WaitForFirstConsumer
    parameters:
      csi.storage.k8s.io/fstype: xfs
      type: gp2
      encrypted: "true"
```

Install the latest release of the driver.

```bash
helm upgrade --install aws-ebs-csi-driver -f ebs-csi-values.yaml -n kube-system aws-ebs-csi-driver/aws-ebs-csi-driver
```

#### Dev and Prod environments 

Your service should be deployed for both **Development** and **Production** environments, **on the same** Kubernetes cluster.

As a principal, the resources related to each environment should be **completely independent**:

- Two different Telegram tokens for 2 bots.
- Different resources in AWS: S3, SQS, Secret Manager, etc...
- Different namespaces in Kubernetes (create the `dev` and `prod` namespaces and deploy each env in its own namespace).

> [!NOTE]
> While it's possible to separate compute resources in Kubernetes, for this project, we will share EC2 instances and VPC to save costs. 
> This means both environments will use the same underlying infrastructure for compute, but remain logically separated within the cluster.

Make sure to align your CI/CD pipelines to reflect different environments as done in class. 

## Getting started template

Here is a good starting point for `tf/main.tf :

```terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">=5.0"
    }
  }
  
  backend "s3" {
    bucket = "YOUR_TFSTATE_BUCKET"
    key    = "path/to/my/key"
    region = "us-east-1"
  }

  required_version = ">= 1.7.0"
}

provider "aws" {
  region  = var.region
}
```

# Good Luck


[PolybotServiceAWS]: https://github.com/alonitac/INTPolybotServiceAWS
[tf_project_regions]: https://alonitac.github.io/DevOpsTheHardWay/img/tf_project_regions.png

## Managing Infrastructure with Terraform and Terragrunt

### Objective
Terraform and Terragrunt are powerful devops tools to manage cloud infrastructure as code. With this tutorial you'll learn
how to manage resources for an example AWS project.

### Terraform Intro
Terraform allows us to define the infra resources we need for an AWS project as `.tf` files with HashiCorp Language (HCL).
One or mode terraform files within a folder is called a terraform module. When you use terraform to create or update the
infra resources for your project, you generally run the following commands within your module:
```shell
terraform init
terraform plan
terraform apply
```

You can use the code below to create and configure an example module. The module would be how you might start a Virtual
Private Cloud (VPC) module. To use it for real projects you would need to add more resources like subnets, route tables,
the internet gateway, nat gateways, and more.
- https://gist.github.com/rkhullar/de91fa49db1dbf346ad1830e1b3e2dbb?file=interface.tf
- https://gist.github.com/rkhullar/de91fa49db1dbf346ad1830e1b3e2dbb?file=default.tf
- https://gist.github.com/rkhullar/de91fa49db1dbf346ad1830e1b3e2dbb?file=provider.tf
- https://gist.github.com/rkhullar/de91fa49db1dbf346ad1830e1b3e2dbb?file=terraform.tfvars

After you run the `terraform apply` command you should see a new file called `terraform.tfstate`. This file is used to keep
track of resources that are managed by terraform. As you add more resources to your module or update your configuration,
the state file is used to generate a change plan during subsequent applies. The change plan details which resources are being
created, updated, or deleted.

### Terragrunt Intro
"Terragrunt is a thin wrapper that provides extra tools for keeping your configurations DRY, working with multiple
Terraform modules, and managing remote state." Continuing from the terraform example, there are some limits around managing
the tfstate and reusing code that are solved with Terragrunt. It's essential to store the terraform state files in a shared
and secure location. You should not store them locally, and you should not store them in version control either. For AWS
projects it's common to store the state inside S3 buckets.





[terraform]: https://www.terraform.io
[terragrunt]: https://terragrunt.gruntwork.io
[hashicorp]: https://www.hashicorp.com
[aws-provider]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs

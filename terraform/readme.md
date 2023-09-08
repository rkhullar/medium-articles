## Managing Infrastructure with Terraform and Terragrunt

### Objective
Terraform and Terragrunt are powerful devops tools to manage cloud infrastructure as code. With this tutorial you'll learn
how to manage resource for an example AWS project.

### Terraform Intro
Terraform allows us to define the infra resources we need for an AWS project as `.tf` files with Hashicorp Language (HCL).
One or mode terraform files within a folder is called a terraform module. When you use terraform to create or update the
infra resources for your project, you generally run the following commands within your module:
```shell
terraform init
terraform plan
terraform apply
```

You can use the following code to create an example module:
terraform.tfstate

### Terragrunt Intro





[terraform]: https://www.terraform.io
[terragrunt]: https://terragrunt.gruntwork.io
[hashicorp]: https://www.hashicorp.com
[aws-provider]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs

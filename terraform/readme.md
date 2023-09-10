## Managing AWS Infrastructure with Terraform and Terragrunt

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

You configure Terraform to access your AWS accounts the same way you would configure the AWS cli. You should set your
`AWS_PROFILE` environment variable. And if you're working with an AWS organization you likely need to run `aws sso login`
each day to refresh your profile's session credentials.

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
Terraform modules, and managing remote state." There are limits with Terraform around managing the tfstate and reusing code
that are solved with terragrunt.

#### Remote State
It's essential to store the tfstate files in a shared and secure location. You should not store them locally,
and you should not store them in version control either. For AWS projects it's common to store the tfstate files inside
S3 buckets and to use dynamodb table for state locking. With the remote state bucket, you and your teammates have shared
access to your project's tfstate files. And with the lock table we ensure that only one `terraform apply` command against
the tfstate can run at a time.

Terragrunt can automatically create the remote state resources for you during the first apply. You define the naming pattern
and tags you want, and terragrunt would create the s3 bucket and dynamodb lock table with the proper configuration. The
s3 bucket would have versioning and encryption enabled. And it should block all public access.

#### Module Reuse and Versioning
When we create modules for our projects, the goal is to use those modules to provision multiple environments. For example,
you might call your environments `dev`, `stage`, and `prod`. And you may want to deploy your project in multiple regions
like AWS `us-east-1`, and `us-west-1`. Another key goal for managing infrastructure and software in general is the ability
to promote changes upwards across your environments. We should deploy to lower environments like `dev` and `stage` before 
deploying to `prod`. Terragrunt helps us easily achieve both of these goals, enabling us to efficiently use Terraform
without code repetition or custom scripts.

### Project Structure
We will use two Git repositories for managing our project infrastructure: `example-infra-live` and `example-infra-modules`.
I recommend keeping at least the `live` repo private since it has hard coded configuration and details like your AWS account
numbers and team email address. We are also going to reference [rkhullar/terraform-modules][common-modules]. That public repo
has AWS modules and constructs that can be used across multiple projects.

For our example project we'll create three modules that can managed independently of one another. From here on out we will
refer to those types of modules as "constructs." The `iam` construct will define the iam roles for our project. And the
`lambdas` and `api` constructs will define our lambda function and HTTP API Gateway respectively.

The project structure for the `live` uses the following pattern:
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=live-template.txt

So if you have two AWS accounts in your organization for `prod` and `non-prod` the complete structure would look like this:
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=live.txt

The root `terragrunt.hcl` has logic for managing the remote state and for providing general context to each context like 
the region, environment, account id and more. The root `common.hcl` file defines common configuration values like the
project name, your organization name, and team email. The nested `terragrunt.hcl` defines specific for your constructs
and the url reference to the `modules` repo. 

For the `modules` repo we'll have a folder for each construct. And as a good practice we should organize the terraform
files based on responsibility. Here's the complete project structure for the `modules` repo:
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=modules.txt

### Module Code
There is some code repetition within our constructs since we need to define common input variables and common tags.
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-common-interface.tf
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=modules-common-locals.tf
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=modules-common-provider.tf

Starting with the `iam` construct let's add terraform code to create a basic lambda function role.
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-iam-default.tf
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-iam-interface.tf

Next let's add code to manage the example lambda function. The `remotes.tf` code reads the remote state from the `iam`
construct so that we have access to the full iam role name. We could have inferred the name instead via terraform
interpolation, but I prefer this method since I think it's more practical. For my own projects I would need the construct
to read other remote states for referencing parameter store values or security groups.
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-lambdas-default.tf
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-lambdas-remotes.tf
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-lambdas-interface.tf

Finally, let's implement the `api` construct. Note that before you run the terragrunt commands in the next session you'll
need to have access to modify DNS records for a public domain name, and you'll need to make sure that ACM certificates 
exist for the subdomain you're using.
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-api-default.tf
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-api-remotes.tf
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-lambdas-interface.tf

### Live Code
For the live repo we don't need to cover each file. We'll add the root `terragrunt.hcl` and `common.hcl`, the `non-prod`
`account.hcl`, and the `dev` `terragrunt.hcl` files.

- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=live-terragrunt.hcl
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=live-common.hcl
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=live-account-template.hcl
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=live-dev-iam.hcl
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=live-dev-lambdas.hcl
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=live-dev-api.hcl



[terraform]: https://www.terraform.io
[terragrunt]: https://terragrunt.gruntwork.io
[hashicorp]: https://www.hashicorp.com
[aws-provider]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs
[common-modules]: https://github.com/rkhullar/terraform-modules

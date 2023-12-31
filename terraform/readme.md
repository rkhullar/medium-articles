## Managing AWS Infrastructure with Terraform and Terragrunt

### Objective
Terraform and Terragrunt are powerful devops tools to manage cloud infrastructure as code. With this tutorial you'll learn
how to manage resources for an example AWS project like from my previous tutorial:
[FastAPI on AWS with MongoDB Atlas and Okta][medium-fastpi]

### Terraform Intro
Terraform allows us to define the infrastructure resources we need for an AWS project as `.tf` files with HashiCorp Language (HCL).
One or more terraform files within a folder is called a terraform module. When you use terraform to create or update the
infrastructure resources for your project, you generally run the following commands within your module:
```shell
terraform init
terraform plan
terraform apply
```

You configure Terraform to access your AWS accounts the same way you would configure the AWS CLI. You should set your
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
_"Terragrunt is a thin wrapper that provides extra tools for keeping your configurations DRY, working with multiple
Terraform modules, and managing remote state."_

There are limits with Terraform around managing the tfstate and reusing code that are solved with terragrunt.

#### Remote State
It's essential to store the tfstate files in a shared and secure location. You should not store them locally,
and you should not store them in version control either. For AWS projects it's common to store the tfstate files inside
S3 buckets and to use dynamodb tables for state locking. With the remote state bucket, you and your teammates have shared
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
numbers and team email address. We are also going to reference [rkhullar/terraform-modules][common-modules]. That repo has
AWS modules and constructs that can be used across multiple projects.

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
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-common-locals.tf
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-common-provider.tf

Starting with the `iam` construct let's add terraform code to create a basic lambda function role.

- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-iam-default.tf
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-iam-interface.tf

Next let's add code to manage the lambda function itself. The underlying module uses hello world code to provision the
python lambda function. This is key since we want only to manage the configuration for the lambda function in terraform,
like the runtime, memory, timeout, and environment variables. We don't want to actually manage the source code since that
should live in another codebase and managed through a CI/CD pipeline. After the first provision, the module should ignore
changes to the lamda function's source code.

- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-lambdas-default.tf
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-lambdas-interface.tf
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-lambdas-remotes.tf

The `remotes.tf` code reads the remote state from the `iam` construct so that we have access to the full iam role name
We could have inferred the name instead via terraform interpolation, but I prefer this method since I think it's more
practical. For my own projects I would need the construct to read other remote states for referencing things like parameter
store entries, subnets, and security groups.

Finally, let's implement the `api` construct. Note that before you run the terragrunt commands in the next session you'll
need to have access to modify DNS records for a public domain name, and you'll need to make sure that ACM certificates 
exist for the subdomain you're using.

- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-api-default.tf
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-api-interface.tf
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-api-remotes.tf
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=module-api-domain.tf

### Live Code
For the live repo we don't need to cover each file. We'll add the root `terragrunt.hcl` and `common.hcl`, the `non-prod`
`account.hcl`, and the `dev` `terragrunt.hcl` files.

- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=live-root-terragrunt.hcl
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=live-root-common.hcl
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=live-account-template.hcl
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=live-dev-iam.hcl
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=live-dev-lambdas.hcl
- https://gist.github.com/rkhullar/a244ec2fd1bc958fffc4ce3a44ed613e?file=live-dev-api.hcl

In order to provision the `dev` environment you would navigate to the corresponding `iam`, `lambdas`, and `api` `hcl` files
and run the following commands: The `init` command is actually optional since [auto-init][auto-init] is enabled by default.
```shell
terragrunt init
terragrunt plan
terragrunt apply
```

Notice in the construct hcl files that the `terraform` module sources have version numbers in the urls. As you make changes
within your `modules` repo you should create tags for the next semantic version. I suggest starting with `0.1.0` and incrementing
the patch version if there are no breaking changes. Increment the minor or major version if there are changes that require
more complex commands like terraform state migration. And increment the major version if you are doing a major launch,
or are upgrading the underlying system like the terraform version or the AWS provider version. After you've pushed the tag,
you should update the references from within the `live` repo.

### Additional Reading
- [Terraform AWS Provider Documentation][aws-provider]
- [Terraform Provider Plugin Cache][provider-plugin-cache]
- [Terraform Tutorials][terraform-tutorials]
- [Terraform Version Compatibility][terraform-version-compat]
- [Terraform Module Registry][terraform-module-registry]

[terraform]: https://www.terraform.io
[terragrunt]: https://terragrunt.gruntwork.io
[hashicorp]: https://www.hashicorp.com
[aws-provider]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs
[common-modules]: https://github.com/rkhullar/terraform-modules
[auto-init]: https://terragrunt.gruntwork.io/docs/features/auto-init
[provider-plugin-cache]: https://developer.hashicorp.com/terraform/cli/config/config-file#provider-plugin-cache
[terraform-tutorials]: https://developer.hashicorp.com/terraform/tutorials
[terraform-version-compat]: https://terragrunt.gruntwork.io/docs/getting-started/supported-terraform-versions
[medium-fastpi]: https://medium.com/@rajan-khullar/fastapi-on-aws-with-mongodb-atlas-and-okta-6e37c1d9069
[terraform-module-registry]: https://registry.terraform.io/browse/modules
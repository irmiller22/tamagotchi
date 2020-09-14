# Intro - Getting Started with Terraform

> What is Terraform?

In engineering speak, Terraform is an Infrastructure as Code (IaC) provisioning tool. Hashicorp, the compoany that developed Terraform, describes the technology as "a tool for building, changing, and versioning infrastructure safely and efficiently.

Infrastructure is a word that has started to take on additional meaning beyond the hardware components layer. The cloud is an abstraction of hardware provided by tech giants such as Google, Amazon, and Microsoft. These abstractions provide on-demand compute, storage, and networking services via APIs.

Terraform is an IaC technology that automates infrastructure provisioning in the cloud. Since provisioning infrastructure in the cloud is a tedious process, Terraform simplifies this process by allowing you to write machine-readable definition files.

IaC in the past was accomplished via other configuration management tools such as Chef, Ansible, and Puppet.

> Why Terraform?

Terraform enjoys several advantages that make it easy to work with:

1. Easy to use
1. Free and open source
1. Declarative
1. Cloud agnostic
1. Expressive and extendable

In relation to the competition, Terraform behaves in a declarative manner (and not in an imperative manner). This means that you describe the end result that you want, and not the logic that gets you to the end result. Other IaC tools such as Chef and Puppet were designed to install and manage software on existing servers. They basically work by performing push-pull updates, and restoring software to a desired state. As a result, they tend to favor _mutable_ infrastructure. Terraform favors _immutable_ infrastructure. For example, Terraform prefers to treat infrastructure like cattle instead of like a pet. If a server gets into a bad state, blow the server away and provision a new one instead of trying to nurse it back to health.

Immutable infrastructure is much easier to reason about versus mutable infrastructure. There's no need to worry about leaving artifacts behind in between deployments. THe less you have to worry about the previous state of software, the better.

> Hello, Terraform!

This section will focus on a simple use case where we deploy an AWS EC2 virtual machine.

Here are the steps that will be taken:

- Writing configuration files
- Configuring the AWS provider
- Initializing Terraform (`terraform init`)
- Deploying to AWS (`terraform apply`)
- Cleaning up AWS (`terraform destroy`)

The first step is setting up the configuration file:

```
# main.tf

resource "aws_instance" "helloworld" {
    ami           = "ami-09dd2e08d601bff67"
    instance_type = "t2.micro"
    tags          = {
        Name = "HelloWorld"
    }
}
```

This declarative configuration specifies that a Terraform resource named "helloworld" will be provisioned, and will have a specific Amazon Machine Image (AMI), instance type, and a "HelloWorld" tag.

A `resource` is the central element of Terraform - it is how Terraform deploys infrastructure like gateways, load balancers, VMs, databases, etc.

To summarize the resource definition syntax:

```
resource "resource_type" "resource_name" { ... }

# resource is the AWS element
# resource_type.resource_name is the Terraform resource identifier
```

Next step is to configure the AWS provider.

```
# main.tf

provider "aws" {
    version = 2.65.0
    region  = "us-west-2"
}
```

At this point, we're ready to initialize Terraform via `terraform init`.

# Lifecycle of a Terraform Resource

This document will do a deep dive into the internals of Terraform by walking through the lifecycle of a single resource. In order to keep things simple, we'll use a resource that doesn't call any remote network API. These resources are called _local-only_ resources, and only exist within the confines of Terraform. They service a marginal purpose, such as to glue together infrastructure objects. Examples of _local-only_ resources include private keys, self-signed TLS certificates, and random IDs.

## Process Overview

`local_file` is a resource from the local provider, and it allows us to create and manage text files with Terraform. We'll create a file called `art_of_war.txt` containing the first two stanzas of Art of War.

The inputs will be the `main.tf` file, and the output will be the `art_of_war.txt` file. In addition, we'll be leveraging several Terraform lifecycle function hooks: `Create`, `Read`, `Update`, and `Delete`. Generally, resource creation calls `Create`, plan generation calls `Read`, resource updating calls `Update`, and resource deletion calls `Delete`.

```
# main.tf

terraform {
    required_version = "~> 0.13"
    required_providers {
        local = "~> 1.4"
    }
}

resource "local_file" "literature" {
    filename = "art_of_war.txt"
    content = <<-EOT
        Sun Tzu said: The art of war is of vital importance to the State.

        It is a matter of life and death, a road either to safety or to ruin. Hence it is a subject of inquiry which can on no account be neglected.
    EOT
}
```

`terraform {...}` is a special configuration block responsible for configuring terraform itself. It's primarly used to lock the version of Terraform, but can also configure where your state file is stored, and where providers are downloaded.

In order to set up your Terraform workspace, you can run `terraform init`.

## Resource Plan

Right now, we've created our Terraform workspace, but we haven't actually created the file that we declared in the "local_file" resource type. In order to do that, we can see what would be generated via `terraform plan`. The plan outputs the following:

```
Terraform will perform the following actions:

  # local_file.literature will be created
  + resource "local_file" "literature" {
      + content              = <<~EOT
            Sun Tzu said: The art of war is of vital importance to the State.

            It is a matter of life and death, a road either to safety or to ruin. Hence it is a subject of inquiry which can on no account be neglected.
        EOT
      + directory_permission = "0777"
      + file_permission      = "0777"
      + filename             = "art_of_war.txt"
      + id                   = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

Sometimes the plan can fail. In order to debug more closely, you can set the environment variable `TF_LOG` to a non-zero value in order to be more verbose.

THere are three stages of a `terraform plan` invocation:

- Reading configuration and state
- Determining actions to take
- Outputting plan

We can also inspect the plan by reading the output in JSON format. We can save the output of the plan by setting the optional `-out` flag:

`terraform plan -out plan.out`
`terraform show -json plan.out > plan.json`

## Creating the Resource

Now we can create the resource via `terraform apply`.

## Updating the Resource

What happens if we add some changes to the Terraform configuration? No problem, we can apply those changes.

We'll get this plan once we add some more sentences:

```
  # local_file.literature must be replaced
-/+ resource "local_file" "literature" {
      ~ content              = <<~EOT # forces replacement
            Sun Tzu said: The art of war is of vital importance to the State.

            It is a matter of life and death, a road either to safety or to ruin. Hence it is a subject of inquiry which can on no account be neglected.
          +
          + The art of war, then, is governed by five constant factors, to be taken into account in one's deliberations, when seeking to determine conditions obtaining in the field.
          +
          + These are: 1, the moral law; 2, heaven; 3, earth; 4, the commander, 5, method and discipline.
        EOT
        directory_permission = "0777"
        file_permission      = "0777"
        filename             = "art_of_war.txt"
      ~ id                   = "de0e4d570e67f3a8102c83d933edb651a91e42db" -> (known after apply)
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

Once we're good with the plan, we can run `terraform apply`.

## Configuration Drift

What happens if we experience configuration drift? Let's say that `Sun Tzu` was changed to `Napoleon` in the `art_of_war.txt` file. When we run `terraform plan`, we'll get a plan back as if the file didn't exist. It actually does exist, just run `terraform show`. Terraform has an interesting implementation where if the contents of the file don't exactly match up with what's specified in the tfstate file, then the resource doesn't exist anymore.

So how do we fix this? Let's try to reconcile the state that Terraform knows about with what's currently deployed via `terraform refresh`:

```bash
irmiller22@personal ../ch_2_resource_lifecycle ❯ terraform refresh
local_file.literature: Refreshing state... [id=64dcb3f0eb2c90aa8c3d5d3c668b64891b2a9bd9]
```

What this ends up doing is just clearing out the tfstate file, which is the correct behavior since Terraform believes the file doesn't exist anymore. In order to recreate the file and add it to the tfstate file, we can run `terraform apply -auto-approve`.

# Terraform in provider Google Cloud

## Setup & Requirements

1. Activation cloud shell
2. list the active account name with command

   `gcloud auth list`

3. Click **Authorize** on dialog shell
4. (Optional) You can list the project ID with this command

   `gcloud config list project`

## Build infrastructure

Start by creating your example configuration to a file named `main.tf`.

Terraform recognizes files ending in `.tf` or `.tf.json` as configuration files and will load them when it runs.

1. Create file `main.tf` file with command

   `touch main.tf`

2. You can use **Open Editor** button on the toolbar of Cloud Shell.
3. In the Shell/Editor, add the following content to the `main.tf` file. Make sure to replace `<PROJECT_ID>` with the your Project ID:
   ```terraform
   terraform {
     required_providers {
       google = {
         source = "hashicorp/google"
         version = "3.5.0"
       }
     }
   }
   provider "google" {
     project = "<PROJECT_ID>"
     region  = "us-central1"
     zone    = "us-central1-c"
   }
   resource "google_compute_network" "vpc_network" {
     name = "terraform-network"
   }
   ```

### Initialize the directory

Initialize your new Terraform configuration by running the `terraform init` command in the same directory as your `main.tf` file:

`terraform init`

### Format and validate the configuration

Format your configuration. Terraform will print out the names of the files it modified, if any. In this case, your configuration file was already formatted correctly, so Terraform won't return any file names.

`terraform fmt`

You can also make sure your configuration is syntactically valid and internally consistent by using the `terraform validate` command.

`terraform validate`

## Create infrastructure

Apply the configuration now with the `terraform apply` command.

Terraform will print output similar to what is shown below. We have truncated some of the output for brevity.

`terraform apply`

Terraform will indicate what infrastructure changes it plans to make, and prompt for your approval before it makes those changes.

In this case the plan looks acceptable, so type `yes` at the confirmation prompt to proceed. It may take a few minutes for Terraform to provision the network.

In Cloud Shell run the `terraform show` command to inspect the current state

## Change infrastructure

you will modify your configuration, and learn how to apply changes to your Terraform projects.

### Create a new resource

You can create new resources by adding them to your Terraform configuration and running `terraform apply` to provision them.

```terraform
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "f1-micro"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network = google_compute_network.vpc_network.name
    access_config {
    }
  }
}
```

Now run `terraform apply` to create the compute instance. Terraform will prompt you to confirm the operation.

`terraform apply`

### Change Configuration

In addition to creating resources, Terraform can also make changes to those resources.

```
resource "google_compute_instance" "vm_instance" {
   name         = "terraform-instance"
   machine_type = "f1-micro"
+  tags         = ["web", "dev"]
   ## ...
 }

```

Run `terraform apply` again.

`terraform apply`

The prefix `~` means that Terraform will update the resource in-place. You can apply this change now by responding yes, and Terraform will add the tags to your instance.

### Destructive changes

A destructive change is a change that requires the provider to replace the existing resource rather than updating it. This usually happens because the cloud provider does not support updating the resource in the way described by your configuration.

```
boot_disk {
     initialize_params {
-      image = "debian-cloud/debian-11"
+      image = "cos-cloud/cos-stable"
     }
   }
```

This modification changes the boot disk from a Debian 11 image to Google's Container-Optimized OS.

Now run `terraform apply` again. Terraform will replace the instance because Google Cloud Platform does not support replacing the boot disk image on a running instance.

`terraform apply`

The prefix `-/+` means that Terraform will destroy and recreate the resource, rather than updating it in-place. Terraform and the GCP provider handle these details for you, and the execution plan reports what Terraform will do.

Once again, Terraform prompts for approval of the execution plan before proceeding. Answer `yes` to execute the planned steps:

As indicated by the execution plan, Terraform first destroyed the existing instance and then created a new one in its place. You can use `terraform show` again to see the new values associated with this instance.

### Destroy infrastructure

You have now seen how to build and change infrastructure. Before moving on to creating multiple resources and showing resource dependencies, you will see how to completely destroy your Terraform-managed infrastructure.

Destroying your infrastructure is a rare event in production environments. But if you're using Terraform to spin up multiple environments such as development, testing, and staging, then destroying is often a useful action.

Resources can be destroyed using the terraform destroy command, which is similar to terraform apply but it behaves as if all of the resources have been removed from the configuration.

`terraform destroy`

The `-` prefix indicates that the instance and the network will be destroyed. As with apply, Terraform shows its execution plan and waits for approval before making any changes.

## Create Resources Depedencies

Real-world infrastructure has a diverse set of resources and resource types. Terraform configurations can contain multiple resources, multiple resource types, and these types can even span multiple providers.

## Assigning a static IP address

Now add to your configuration by assigning a static IP to the VM instance in `main.tf` file:

```terraform
resource "google_compute_address" "vm_static_ip" {
  name = "terraform-static-ip"
}
```

run `terraform plan` to show what would be changed.

Update the `network_interface` configuration for your instance like so

```terraform
  network_interface {
    network = google_compute_network.vpc_network.self_link
    access_config {
      nat_ip = google_compute_address.vm_static_ip.address
    }
  }
```

The `access_config` block has several optional arguments, and in this case you'll set `nat_ip` to be the static IP address. When Terraform reads this configuration, it will

- Ensure that `vm_static_ip` is created before `vm_instance`
- Save the properties of `vm_static_ip` in the state
- Set `nat_ip` to the value of the `vm_static_ip.address` property

Run `terraform plan` with save then plan

`terraform plan -out static_ip`

Saving the plan this way ensures that you can apply exactly the same plan in the future. If you try to apply the file created by the plan, Terraform will first check to make sure the exact same set of changes will be made before applying the plan.

### Implicit and explicit dependencies

By studying the resource attributes used in interpolation expressions, Terraform can automatically infer when one resource depends on another. In the example above, the reference to `google_compute_address.vm_static_ip.address` creates an implicit dependency on the `google_compute_address` named `vm_static_ip`.

Terraform uses this dependency information to determine the correct order in which to create and update different resources. In the example above, Terraform knows that the `vm_static_ip` must be created before the `vm_instance` is updated to use it.

Implicit dependencies via interpolation expressions are the primary way to inform Terraform about these relationships, and should be used whenever possible.

Sometimes there are dependencies between resources that are not visible to Terraform. The `depends_on` argument can be added to any resource and accepts a list of resources to create explicit dependencies for

1. Add a Cloud Storage bucket and an instance with an explicit dependency on the bucket by adding the following to `main.tf` file

```terraform
# New resource for the storage bucket our application will use.
resource "google_storage_bucket" "example_bucket" {
  name     = "<UNIQUE-BUCKET-NAME>"
  location = "US"
  website {
    main_page_suffix = "index.html"
    not_found_page   = "404.html"
  }
}
# Create a new instance that uses the bucket
resource "google_compute_instance" "another_instance" {
  # Tells Terraform that this VM instance must be created only after the
  # storage bucket has been created.
  depends_on = [google_storage_bucket.example_bucket]
  name         = "terraform-instance-2"
  machine_type = "f1-micro"
  boot_disk {
    initialize_params {
      image = "cos-cloud/cos-stable"
    }
  }
  network_interface {
    network = google_compute_network.vpc_network.self_link
    access_config {
    }
  }
}

```

Now run `terraform plan` and `terraform apply` to see these changes in action

## Provision infrastructure

Terraform uses provisioners to upload files, run shell scripts, or install and trigger other software like configuration management tools.

### Defining a provisioner

To define a provisioner, modify the resource block defining the first `vm_instance` in your configuration to look like the following:

```terraform
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "f1-micro"
  tags         = ["web", "dev"]
  provisioner "local-exec" {
    command = "echo ${google_compute_instance.vm_instance.name}:  ${google_compute_instance.vm_instance.network_interface[0].access_config[0].nat_ip} >> ip_address.txt"
  }
  # ...
}

```

Run `terraform apply`

Terraform treats provisioners differently from other arguments. Provisioners only run when a resource is created, but adding a provisioner does not force that resource to be destroyed and recreated

Use `terraform taint` to tell Terraform to recreate the instance

`terraform taint google_compute_instance.vm_instance`

A tainted resource will be destroyed and recreated during the next apply.

Run `terraform apply`

Verify everything worked by looking at the contents of the `ip_address.txt` file.

---
tags:
  - Terraform
---

# More than Certified in Terraform - Derek Morgan

Course projects: <https://github.com/liftconfig/terraform-mtc-projects>

## Section 1: Introduction

### 12. Terraform Terminology

- Declarative language
- Walks through a dependency graph
- Requires state
- Idempotent
- Not always declarative. For example; provisioners (local/remote-exec) may operate procedurally

## Section 2: Terraform Basics and Docker

### 15. Init Deep Dive

<https://terraform.io/docs/cli/commands/init.html>

### 16. Terraform Dependency Lock

<https://terraform.io/docs/language/dependency-lock.html>

- Locks the provider version
- Won't let you change the provider version with the lock in place unless you use `terraform init -upgrade`. This means you don't necessarily have to manually set the version in terraform:required_providers config.
- If you use `~>` you can lock it so only minor version upgrades are permitted x.x.y `version = "~> 2.12.0"` or x.y `version = "~> 2.12"`. Whatever is the latest version of the furthest right number will be used.

### 17. Your first terraform apply

- Make sure AWS subnet has allocated public IPs enabled
- Spin up Cloud9 Instance with access option selected as SSH
- Allow SSH from local public IP into EC2 C9 instance inbound security group
- Use ssh-keygen to generate pub/private keys on local Linux container. Copy into `/home/vscode/.ssh/`
- In C9 append pub key to `/home/ubuntu/.ssh/authorized_keys`
- `ssh ubuntu@x.x.x.x` from local Linux container
- You can then set the host in the docker provider to `host = "ssh://ubuntu@x.x.x.x"` and plan/apply terraform

### 18. Terraform plan & apply deeper dive

- `terraform plan -out=plan1` creates an encoded plan file. The plan file is not encrypted so it needs to be stored somewhere safe
- `terraform apply plan1` no confirmation dialogue, just creates resources
- `terraform plan -destroy` simulate a destroy

### 19. Referencing other resources

<https://terraform.io/docs/configuration/expressions.html>

### 20. View docker image in browser

- To find public IP of EC2 instance `curl http://169.254.169.254/latest/meta-data/public-ipv4`

### 21. Terraform State Deeper Dive

- Lineage: generated when your state file is originally created. Verify that it's the correct state
- terraform.tfstate.backup = previous state file before latest apply
- View the state file without accessing the file directly: `terraform show -json | jq`. jq prettifys the JSON output.
- `terraform state list` lists all resources in the state file (1 line per resource). Quick way to check what resources have been provisioned without too much detail.
- `terraform state show xxxx` shows the attributes or a resource in the state file

### 22. Terraform Console & Outputs

- Use outputs to display state information which can then be used by other apps or terraform modules
- Use `terraform console` to query the attributes without having to look through the state file

```terraform
  vscode ➜ /workspaces/terraform-associate/  terraform-docker $ terraform console
  > docker_container.nodered_container.name
  "nodered"
  > docker_container.nodered_container.id
  "ee6cc21182468fb448213865632303360d99db6aaf56be5  fa1bbd15038ae7001"
  > docker_container.nodered_container. network_data[0].ip_address
  "172.17.0.2"
```

- Add an output resource
  
```terraform
  output "ip-address" {
  value = docker_container.nodered_container.network_data[0].ip_address
  description = "The IP Address of the container"
  }
```

- After output resources have been created, you can use `terraform output` to show the value of all output resources

```terraform
  vscode ➜ /workspaces/terraform-associate/terraform-docker $ terraform output
  IP-Address = "172.17.0.2"
  container-name = "nodered"
```

### 23. Terraform Functions

<https://terraform.io/docs/configuration/functions/join.html>

- `join(separator, list)`
- `join ("+", ["thing", 1]) = "thing+1"`

```terraform
  > docker_container.nodered_container.network_data[0].ip_address
  "172.17.0.3"
  > docker_container.nodered_container.ports[0].external
  1880
  > join(":", [docker_container.nodered_container.network_data[0].ip_address, docker_container.nodered_container.ports[0].external])
  "172.17.0.3:1880"
```

### 24. The Random Resource

<https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/string>

- Can be used for unique resources names

```terraform
  resource "random_string" "random" {
    length = 4
    special = false
    upper = false
  }

  resource "docker_container" "nodered_container" {
    name  = join("-",["nodered", random_string.random.result])
    image = docker_image.nodered_image.image_id
    ports {
      internal = 1880
      # external = 1880
    }
  }
```

### 25. Multiple Resources and Count

<https://developer.hashicorp.com/terraform/language/meta-arguments/count>

- Count.index will reference the current resource count #. Combine with random_string to reference a matching random_string index.

```terraform
  resource "random_string" "random" {
    count = 2
    length = 4
    special = false
    upper = false
  }

  resource "docker_container" "nodered_container" {
    count = 2
    name  = join("-",["nodered", random_string.random[count.index].result])
    image = docker_image.nodered_image.image_id
    ports {
      internal = 1880
      # external = 1880
    }
  }
```

### 26. The splat expression

<https://developer.hashicorp.com/terraform/language/expressions/splat>

- Used like a for loop, but allows you to more concisely reference all the resources created by count

```terraform
  vscode ➜ /workspaces/terraform-associate/terraform-docker $ terraform console
  > docker_container.nodered_container[*].name
  [
    "nodered-s061",
    "nodered-9pu5",
  ]

  output "container-name" {
    value = docker_container.nodered_container[*].name
    description = "The name of the container"
  }
```

### 27. For Expression

```terraform
> [for i in [1, 2, 3]: i + 1]
[
  2,
  3,
  4,
]
> [for i in docker_container.nodered_container[*]: i.name]
[
  "nodered-s061",
  "nodered-9pu5",
]
> [for i in docker_container.nodered_container[*]: join(":",[i.network_data[0].ip_address, i.ports[0]["external"]])] 
[
  "172.17.0.3:32776",
  "172.17.0.2:32775",
]

output "container-name" {
  value = docker_container.nodered_container[*].name
  description = "The name of the container"
}

output "ip-address" {
  value = [for i in docker_container.nodered_container[*]: join(":",[i.network_data[0].ip_address, i.ports[0]["external"]])] 
  description = "The IP Address and external port of the container"
}
```

### 28. Tainting Resources

- Deprecated in favour of `-replace=...` option when using plan/apply
- Way to force a resource to be destroyed and re-created. Example use case is if you need to re-apply a configuration from a volume. Similar to rebooting / reloading a daemon
- Be very careful with taint. For example if you taint the random_string resource, the resource that relies on it for its name would also get replaced on next apply

```bash
vscode ➜ /workspaces/terraform-associate/terraform-docker $ terraform taint random_string.random[0]
Resource instance random_string.random[0] has been marked as tainted.
vscode ➜ /workspaces/terraform-associate/terraform-docker $ terraform plan
```

- Untaint a resource `terraform untaint random_string.random[0]`

### 29. State locking & breaking state

- `terraform apply -lock=true` is the default. If you don't lock the state there's a potential for corruption if multiple updates are done simultaneously.

### 30. Terraform import

<https://developer.hashicorp.com/terraform/cli/import>

`terraform import docker_container.nodered_container2 $(docker inspect --format="{{.ID}}" nodered-3pfl)`

- You can find the above import statement in the import section of the terraform docker provider.

### 31. Terraform Refresh and State rm

- `terraform refresh` manually refresh state. If an object name has changed this will update the state file without having to apply.
- `terraform state rm random_string.random[1]` Manually remove a resource from the state file. Only should be done when you don't want to run a terraform apply immediately (if for example, you're deleting a random_string resources that is also used for the name of another resource).

### 32. Adding Variables

<https://developer.hashicorp.com/terraform/language/values/variables>

- If you don't define a default value or a value, the console will ask you for a value on every plan/apply/destroy
- You can specify the value on the CLI `terraform plan -var ext_port=1880`
- You can specify the value using ENV variables `export TF_VAR_ext_port=1880`. Unset with `unset TF_VAR_ext_port`

### 33. Variable Validation

- See variables link above. Allows you to put conditions and error messages within the variable block to validate the values

### 35. Sensitive Variables

- `terraform.tfvars` is your variable definition file. This file should always be added to your git ignore file

### 36. Variable Definition Procedure

- Specify different tfvar file for different environments `terraform plan --var-file west.tfvars`
- Any variables specified in the CLI will override the default terraform.tfvars file
- If you specify both a `--var-file` and a `-var` on the command line then order matters. The last item in the command will win e.g. `terraform plan -var-file=west.tfvars -var ext_port=2000` then the -var value of 2000 will be used. `terraform plan  -var ext_port=2000 -var-file=west.tfvars` then the value of ext_port from west.tfvars will be used.

### 37. Hiding sensitive values from CLI

- Set `sensitive = true` in the variable
- If you try to use a sensitive variable in a non-sensitive output you will get the following error:

```terraform
  Error: Output refers to sensitive values

  on outputs.tf line 6:
  6: output "ip-address"
```

- `terraform show | grep external` will also show as sensitive
- Only place sensitive variables are not hidden is in the terraform.tfstate file

### 38. The bind mount and local-exec

<https://developer.hashicorp.com/terraform/language/resources/provisioners/local-exec>

- Invokes a local executable on the machine running terraform after a resource has been created.
- Remote exec will do the same but on a remote machine
- Not recommended to use local-exec as it's not idempotent, will run on every apply. Use a configuration management tool like Ansible instead.
- Local-exec provisioner needs to run in a resource block. You can use null_resource for this
- Example - create a directory for a container's persistent volume using remote-exec:
  
```terraform
  resource "null_resource" "dockevol" {
    connection {
      type = "ssh"
      user = "ubuntu"
      host = "x.x.x.x"
      private_key = "${file("../id_rsa")}"
    }

    provisioner "remote-exec" {
      inline = [
        "mkdir /home/ubuntu/environment/noderedvol/ || true && sudo chown -R 1000:1000 /home/ubuntu/environment/noderedvol/"
      ]
    }

  resource "docker_container" "nodered_container" {
    count = var.container_count
    name  = join("-",["nodered", random_string.random[count.index].result])
    image = docker_image.nodered_image.image_id
    ports {
      internal = var.int_port
      external = var.ext_port
    }
    volumes {
      container_path = "/data"
      host_path = "/home/ubuntu/environment/noderedvol"
    }
    depends_on = [ null_resource.dockervol ]
  }
```

### 39. Utilising Local Values

<https://developer.hashicorp.com/terraform/language/values/locals>

- Local values are like a traditional functions temporary local variables
- Example - creating a local var for counting the number of items in a list:

```terraform
  ext_port = [1880, 1881, 1882, 1883]

  locals {
    container_count = length(var.ext_port)
  }
```

### 40. Min & Max functions. The expand expression

- The expand expression can be used to expand a list of values so that they can be evaluated by the likes of min/max. These functions don't allow lists natively
- `max([10, 20, 30]...), max is 30`

```terraform
  variable "ext_port" {
    type = list
    
    validation {
      condition = max(var.ext_port...) <= 65535 && min(var.ext_port...) > 0
      error_message = "The external port must be between 0 and 65535"
    }
  }
```

### 41. Path references and string interpolation

<https://developer.hashicorp.com/terraform/language/expressions/references>

- Evaluates expression inside ${..}. Allows you to insert a variable in a string `"Hello, ${var.name}!"`

### 42. Maps and Lookups: the Image Variable

<https://stackoverflow.com/questions/11489428/how-to-make-vim-paste-from-and-copy-to-systems-clipboard>
<https://developer.hashicorp.com/terraform/language/functions/lookup>

- Use case is for setting the value of identifiers depending on the environment (prod, dev, test)

```terraform
  variable "env" {
    type = string
    description = "Env to deploy to"
    default = "dev"
  }

  variable "image" {
    type = map
    description = "Image to container"
    default = {
      dev = "nodered/node-red:latest"
      prod = "nodered/node-red:latest-minimal"
    }
  }

  resource "docker_image" "nodered_image" {
    name = lookup(var.image, var.env)
  }

  $ terraform plan | grep name
      + hostname                                    = (known after apply)
      + name                                        = (known after apply)
      + name        = "nodered/node-red:latest"
  $ terraform plan -var="env=prod" | grep name
      + hostname                                    = (known after apply)
      + name                                        = (known after apply)
      + name        = "nodered/node-red:latest-minimal"
```

- Example - setting ports for docker container based on environment:

```terraform
  ext_port = {
    dev = [1980, 1981]
    prod = [1880, 1881]
  } 

  variable "ext_port" {
    type = map
    
    validation {
      condition = max(var.ext_port["dev"]...) <= 65535 && min(var.ext_port["dev"]...) >= 1980
      error_message = "The external port must be between 0 and 65535"
    }

    validation {
      condition = max(var.ext_port["prod"]...) <= 1980 && min(var.ext_port["prod"]...) >= 1880
      error_message = "The external port must be between 0 and 65535"
    }
  }

  locals {
    container_count = length(lookup(var.ext_port, var.env))
  }

  resource "docker_container" "nodered_container" {
    count = local.container_count
    name  = join("-",["nodered", random_string.random[count.index].result])
    image = docker_image.nodered_image.image_id
    ports {
      internal = var.int_port
      external = lookup(var.ext_port,var.env)[count.index]
    }
    volumes {
      container_path = "/data"
      host_path = "/home/ubuntu/environment/noderedvol"
    }
    depends_on = [ null_resource.dockervol ]
  }
```

> Note you can't reference another variable in a variable definition so you can't use var.env in the ext_port validation

### 44. Terraform Workspaces

<https://developer.hashicorp.com/terraform/cloud-docs/workspaces>

- Isolated versions of the terraform state that allows you to deploy multiple versions of the same environment with different variables, counts etc.
- Typically, workspaces are tied to branches in git. Work better in small well-defined environments. Could become hard to manage if there are many differences between environments
- Only particular backends support multiple workspaces (most popular ones do)
- New folders for workspace state will be stored in `terraform.tfstat.d/"workspace name"`

```terraform
terraform workspace new dev
terraform workspace new prod
terraform workspace show
terraform workspace list
  default
  dev
* prod

terraform workspace select dev
terraform apply --auto-approve -var="env=dev"
Apply complete! Resources: 6 added, 0 changed, 0 destroyed.

Outputs:

container-name = [
  "nodered-q0tu",
  "nodered-gwtn",
]
ip-address = [
  "172.17.0.3:1980",
  "172.17.0.2:1981",
]

terraform workspace select prod
terraform apply --auto-approve -var="env=prod"

Outputs:

container-name = [
  "nodered-swel",
  "nodered-25uo",
]
ip-address = [
  "172.17.0.5:1880",
  "172.17.0.4:1881",
]

~/environment $ docker image ls
REPOSITORY         TAG              IMAGE ID       CREATED      SIZE
nodered/node-red   latest-minimal   7b324af6f086   7 days ago   272MB
nodered/node-red   latest           4eb056ef6be0   7 days ago   558MB
~/environment $ docker ps
CONTAINER ID   IMAGE          COMMAND             CREATED          STATUS                             PORTS                    NAMES
909aad51f8ca   7b324af6f086   "./entrypoint.sh"   14 seconds ago   Up 11 seconds (health: starting)   0.0.0.0:1880->1880/tcp   nodered-swel
f9da0351780d   7b324af6f086   "./entrypoint.sh"   14 seconds ago   Up 11 seconds (health: starting)   0.0.0.0:1881->1880/tcp   nodered-25uo
bc7f1018bff3   4eb056ef6be0   "./entrypoint.sh"   2 minutes ago    Up 2 minutes (healthy)             0.0.0.0:1981->1880/tcp   nodered-gwtn
c6412f9c4ce4   4eb056ef6be0   "./entrypoint.sh"   2 minutes ago    Up 2 minutes (healthy)             0.0.0.0:1980->1880/tcp   nodered-q0tu
```

### 45. Referencing your Workspaces

- Use `terraform.workspace` instead of creating a var.env
- This means you don't have to specify the environment variable when doing a terraform apply (less chance of an error)

```terraform
vscode ➜ /workspaces/terraform-associate/terraform-docker $ terraform workspace select prod
vscode ➜ /workspaces/terraform-associate/terraform-docker $ terraform console
> terraform.workspace
"prod"

resource "docker_image" "nodered_image" {
  name = lookup(var.image, terraform.workspace)
}
```

### 46. Using Map Keys instead of Lookups

- Map keys = new way of accessing map values without a lookup. Preferred way of using map in most scenarios
- Only reason to use a lookup over a map key is the default value you can specify `lookup(map, key, default)`

```terraform
ports {
  internal = var.int_port
  external = lookup(var.ext_port,terraform.workspace)[count.index]
}

ports {
  internal = var.int_port
  external = var.ext_port[terraform.workspace][count.index]
}
```

## Section 3: Modular Deployments

### 47. Modules Intro

![Module Overview Diagram](../assets/images/terraform/image.png)

### 48. First Module!

- Move Terraform required providers and provider attributes into their own `providers.tf` file
- Run `terraform init` whenever a new module is used to create a resource
- Create a new module folder and base files `mkdir image/ && cd image/ && touch variables.tf main.tf outputs.tf providers.tf`
- Passing outputs from a module:

```terraform
# --- root/main.tf ---

module "image" {
  source = "./image"
}

resource "docker_container" "nodered_container" {
  count = local.container_count
  name  = join("-",["nodered", terraform.workspace, random_string.random[count.index].result])
  image = module.image.image_out

# --- image/main.tf ---

resource "docker_image" "nodered_image" {
  name = nodred/node-red:latest
}

# --- image/outputs.tf ---

output "image_out" {
  value = docker_image.nodered_image.image_id
}
```

### 49. Module Variables

- Modules shouldn't have any hardcoded values and should be rarely changed. Only specific people should have access to edit modules
- Passing variables into modules:

```terraform
# --- main.tf ---

module "image" {
  source = "./image"
  image_in = var.image[terraform.workspace]
}

# --- variables.tf ---

variable "image" {
  type = map
  description = "Image to container"
  default = {
    dev = "nodered/node-red:latest"
    prod = "nodered/node-red:latest-minimal"
  }
}

# --- image/main.tf ---

resource "docker_image" "nodered_image" {
  name = var.image_in
}

# --- image/variables.tf ---

variable "image_in" {
  description = "name of image"
}
```

### 50. Terraform Graph

- Terraform Graph combined with graphViz allows you to visually see your deployment dependencies
- `Terraform validate` validate your configuration code without doing a plan/apply
- Install graphviz: `sudo apt install graphviz`
- Create a PDF: `terraform graph | dot -Tpdf > graph-plan.pdf`
- Create a PNG: `terraform graph | dot -Tpng > graph-plan.png`
- Create a PNG of plan to destroy: `terraform graph -type=plan-destroy| dot -Tpng > graph-destroy.png
- Very useful for finding dependency issues

### 51. Troubleshooting Dependencies

- Can create two kinds of manual dependencies - implicit and explicit
  - Implicit: You reference one of the resources attributes within your calling resource
  - Explicit: Use `depends_on` within a resource

### 52. The Container Module + module outputs

- Reference the outputs of the module in the main outputs file

```terraform
# --- container/main.tf ---

resource "docker_container" "nodered_container" {
  name  = var.name_in 
  image = var.image_in
  ports {
    internal = var.int_port_in
    external = var.ext_port_in
  }
  volumes {
    container_path = var.container_path_in
    host_path = var.host_path_in
  }
}

# --- container/outputs.tf ---

output "container-name" {
  value = docker_container.nodered_container.name
  description = "The name of the container"
}

output "ip-address" {
  value = [for i in docker_container.nodered_container[*]: join(":",[i.network_data[0].ip_address, i.ports[0]["external"]])] 
  description = "The IP Address and external port of the container"
}

# alternative:
output "ip-address" {
  value = flatten(module.container[*].ip-address)
  description = "The IP Address and external port of the container"
}
```

### 54. Module information flow

![Variables flowing between modules](../assets/images/terraform/image-1.png)

- Information doesn't flow directly between modules. Variables are passed between modules as outputs and inputs via the root module.

### 55. Docker Volume

- Consider if you need a separate module for certain resources or if they should be included within another module
- In this example each container will always have a volume so it makes sense to include it in the container module
- You can use the `volumes` block in the docker_container resource to create a volume but the only problem is these won't get removed with `terraform destroy`
- Correct way is to use a docker_volume resource

```terraform
resource "docker_container" "nodered_container" {
  name  = var.name_in 
  image = var.image_in
  ports {
    internal = var.int_port_in
    external = var.ext_port_in
  }
  volumes {
    container_path = var.container_path_in
    volume_name = docker_volume.container_volume.name
  }
}

resource "docker_volume" "container_volume" {
  name = "${var.name_in}-volume"
}
```

```bash
root@ip-172-31-0-155:/home/ubuntu/environment# docker volume ls
DRIVER    VOLUME NAME
local     nodered-dev-jtar-volume
local     nodered-dev-zkyc-volume
root@ip-172-31-0-155:/home/ubuntu/environment# ls /var/lib/docker/volumes/nodered-dev-jtar-volume/_data/
flows.json  lib  node_modules  package.json  settings.js
```

### 56. Lifecycle Customisation and Targeting

<https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle>

- Lifecycle block arguments `create_before_destroy`, `prevent_destroy`, `ignore_changes`
- Target a particular resource for destruction

```terraform
$ terraform state list
module.container[0].docker_container.nodered_container
module.container[0].docker_volume.container_volume
module.container[1].docker_container.nodered_container
module.container[1].docker_volume.container_volume
module.image.docker_image.nodered_image
$ terraform destroy -target=module.container[0].docker_container.nodered_container
```

### 58. Using for_each

<https://developer.hashicorp.com/terraform/language/meta-arguments/for_each>

- One type of design pattern, may not work for everyone
- Cycles through a map of objects
- Resources that use count use a numerical index, whereas for_each has a key value. A lot easier to troubleshoot and manage keys.

```terraform
module.container[0].docker_container.nodered_container
module.container[0].docker_volume.container_volume

vs

module.image["influxdb"].docker_image.container_image
module.image["nodered"].docker_image.container_image
```

```terraform
# --- root/main.tf ---

locals {
  deployment = {
    nodered = {
      container_count = length(var.ext_port["nodered"][terraform.workspace])
      image = var.image["nodered"][terraform.workspace]
      int = 1880
      ext = var.ext_port["nodered"][terraform.workspace]
      container_path = "/data"
    }
    influxdb = {
      container_count = length(var.ext_port["influxdb"][terraform.workspace])
      image = var.image["influxdb"][terraform.workspace]
      int = 8086 
      ext = var.ext_port["influxdb"][terraform.workspace]
      container_path = "/var/lib/influxdb"
    }
  }
}

module "image" {
  source = "./image"
  for_each = local.deployment
  image_in = each.value.image
}

module "container" {
  source = "./container"
  for_each = local.deployment
  count_in = each.value.container_count
  name_in  = each.key
  image_in = module.image[each.key].image_out
  int_port_in = each.value.int
  ext_port_in = each.value.ext
  container_path_in = each.value.container_path
}

# --- root/terraform.tfvars ---

ext_port = {
  nodered = {
    dev = [1980]
    prod = [1880]
  }
  influxdb = {
    dev = [8186, 8187]
    prod = [8086]
  }
} 

# --- container/main.tf ---

resource "random_string" "random" {
  count = var.count_in
  length = 4
  special = false
  upper = false
}

resource "docker_container" "app_container" {
  count = var.count_in
  name  =  join("-",[var.name_in, terraform.workspace, random_string.random[count.index].result])
  image = var.image_in
  ports {
    internal = var.int_port_in
    external = var.ext_port_in[count.index]
  }
  volumes {
    container_path = var.container_path_in
    volume_name = docker_volume.container_volume[count.index].name
  }
}

resource "docker_volume" "container_volume" {
  count = var.count_in
  name = "${var.name_in}-${random_string.random[count.index].result}-volume"
  lifecycle {
    prevent_destroy = false
  }
}

$ terraform console
> local.deployment
{
  "influxdb" = {
    "image" = "influxdb:latest"
  }
  "nodered" = {
    "image" = "nodered/node-red:latest"
  }
}
> keys(local.deployment)
[
  "influxdb",
  "nodered",
]
> values(local.deployment)
[
  {
    "image" = "influxdb:latest"
  },
  {
    "image" = "nodered/node-red:latest"
  },
]
> values(local.deployment["nodered"])
[
  "nodered/node-red:latest",
]
```

### 61. New Outputs and Map Transformations

- Quickly build a map using a for loop

```terraform
$ terraform console
> {for x in [1, 2, 3, "blue"]: x => "fish"}
{
  "1" = "fish"
  "2" = "fish"
  "3" = "fish"
  "blue" = "fish"
}
```

- Used to build structured maps for module output

```terraform
# --- container/outputs.tf ---

output "application_access" {
  value = {for x in docker_container.app_container[*]: x.name => join(":", [x.network_data[0].ip_address], x.ports[*]["external"])}
}

##For loop technically not necessary as the outputs aren't being manipulated. Can just use module.container for simplicity
output "application_access" {
  value = [for x in module.container[*]: x]
  description = "The name and socket for each application."
}

Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:

application_access = [
  {
    "influxdb" = {
      "application_access" = {
        "influxdb-dev-7tuy" = "172.17.0.3:8186"
        "influxdb-dev-s70o" = "172.17.0.4:8187"
      }
    }
    "nodered" = {
      "application_access" = {
        "nodered-dev-bz3i" = "172.17.0.2:1980"
      }
    }
  },
]
```

### 62. Grafana apply yourself

<https://grafana.com/docs/grafana/latest/installation/docker>

### 64. Self Referencing and Provisioner Failure Modes

<https://developer.hashicorp.com/terraform/language/resources/provisioners/syntax#the-self-object>

- Allows the provisioner to reference the parent resource's attributes
- Example of backing up docker volume before deletion

```terraform
resource "docker_volume" "container_volume" {
  count = var.count_in
  name = "${var.name_in}-${random_string.random[count.index].result}-volume"
  lifecycle {
    prevent_destroy = false
  }
  provisioner "remote-exec" {
    when = destroy
    on_failure = continue
    connection {
      host = "x.x.x.x"
      type = "ssh"
      user = "ubuntu"
      private_key = "${file("../../id_rsa")}"
    }
    inline = [
      "mkdir /home/ubuntu/environment/backup/"
    ]
  }
  provisioner "remote-exec" {
    when = destroy
    on_failure = fail
    connection {
      host = "x.x.x.x"
      type = "ssh"
      user = "ubuntu"
      private_key = "${file("../../id_rsa")}"
    }
    inline = [
      "sudo tar -czvf /home/ubuntu/environment/backup/${self.name}.tar.gz ${self.mountpoint}/"
    ]
  }
}
```

### 65. Terraform apply yourself - self referencing

```terraform
resource "docker_container" "app_container" {
  count = var.count_in
  name  =  join("-",[var.name_in, terraform.workspace, random_string.random[count.index].result])
  image = var.image_in
  ports {
    internal = var.int_port_in
    external = var.ext_port_in[count.index]
  }
  volumes {
    container_path = var.container_path_in
    volume_name = docker_volume.container_volume[count.index].name
  }
  provisioner remote-exec {
    connection {
      host = "x.x.x.x"
      type = "ssh"
      user = "ubuntu"
      private_key = "${file("../../id_rsa")}"
    }
    inline = [
      "echo ${self.name}: ${self.network_data[0].ip_address}:${join("", [for x in self.ports[*]["external"]: x])} >> /home/ubuntu/environment/containers.txt"
    ]
  }
  provisioner "remote-exec" {
    when = destroy
    on_failure = continue
    connection {
      host = "x.x.x.x"
      type = "ssh"
      user = "ubuntu"
      private_key = "${file("../../id_rsa")}"
    }
    inline = [
      "rm -f /home/ubuntu/environment/containers.txt"
    ]
  }
}
```

### 66. Dynamic Blocks

<https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks>

- Allows you to create repeatable nested blocks within a resource using for_each
- Overuse can make configuration hard to read and maintain

```terraform

# --- root/locals.tf ---

locals {
  deployment = {
    grafana = {
      container_count = length(var.ext_port["grafana"][terraform.workspace])
      image           = var.image["grafana"][terraform.workspace]
      int             = 3000
      ext             = var.ext_port["grafana"][terraform.workspace]
      volumes = [
        { container_path_each = "/var/lib/grafana" },
        { container_path_each = "/etc/grafana" }
      ]
    }
  }
}

# --- root/main.tf ---

module "container" {
  source      = "./container"
  for_each    = local.deployment
  count_in    = each.value.container_count
  name_in     = each.key
  image_in    = module.image[each.key].image_out
  int_port_in = each.value.int
  ext_port_in = each.value.ext
  volumes_in  = each.value.volumes
}

# --- container/main.tf ---

resource "docker_container" "app_container" {
  count = var.count_in
  name  = join("-", [var.name_in, terraform.workspace, random_string.random[count.index].result])
  image = var.image_in
  ports {
    internal = var.int_port_in
    external = var.ext_port_in[count.index]
  }
  dynamic "volumes" {
    for_each = var.volumes_in
    content {
      container_path = volumes.value["container_path_each"]
      volume_name = docker_volume.container_volume[volumes.key].name
    }
  }
}

resource "docker_volume" "container_volume" {
  count = length(var.volumes_in)
  name  = "${var.name_in}-${count.index}-volume"
  lifecycle {
    prevent_destroy = false
  }
}
```

```bash
$ docker inspect $(docker ps -a -q) | grep volume
 "grafana-1-volume:/etc/grafana:rw",
 "grafana-0-volume:/var/lib/grafana:rw"
 "Type": "volume",
 "Name": "grafana-1-volume",
 "Source": "/var/lib/docker/volumes/grafana-1-volume/_data",
 "Type": "volume",
 "Name": "grafana-0-volume",
 "Source": "/var/lib/docker/volumes/grafana-0-volume/_data",
```

### 67. Nesting the Volume module

- When there are multiple containers (multiple external ports), a volume module will allow the correct number of modules to be created ()

```terraform

# --- root/locals.tf ---

locals {
  deployment = {
    nodered = {
      container_count = length(var.ext_port["nodered"][terraform.workspace])
      image           = var.image["nodered"][terraform.workspace]
      int             = 1880
      ext             = var.ext_port["nodered"][terraform.workspace]
      container_path  = "/data"
      volumes = [
        { container_path_each = "/data" }
      ]
    }
    influxdb = {
      container_count = length(var.ext_port["influxdb"][terraform.workspace])
      image           = var.image["influxdb"][terraform.workspace]
      int             = 8086
      ext             = var.ext_port["influxdb"][terraform.workspace]
      container_path  = "/var/lib/influxdb"
      volumes = [
        { container_path_each = "/var/lib/influxdb" }
      ]
    }
    grafana = {
      container_count = length(var.ext_port["grafana"][terraform.workspace])
      image           = var.image["grafana"][terraform.workspace]
      int             = 3000
      ext             = var.ext_port["grafana"][terraform.workspace]
      volumes = [
        { container_path_each = "/var/lib/grafana" },
        { container_path_each = "/etc/grafana" }
      ]
    }
  }
}

# --- root/variables.tf ---

variable "int_port" {
  type    = number
  default = 1880

  validation {
    condition     = var.int_port == 1880
    error_message = "The internal port must be 1880"
  }
}

# --- root/terraform.tfvars ---

ext_port = {
  nodered = {
    dev  = [1980]
    prod = [1880]
  }
  influxdb = {
    dev  = [8186]
    prod = [8086]
  }
  grafana = {
    dev  = [3100, 3101]
    prod = [3000]
  }
} 

# --- root/main.tf ---

module "container" {
  source      = "./container"
  for_each    = local.deployment
  count_in    = each.value.container_count
  name_in     = each.key
  image_in    = module.image[each.key].image_out
  int_port_in = each.value.int
  ext_port_in = each.value.ext
  volumes_in  = each.value.volumes
}

# --- root/outputs.tf ---

output "application_access" {
  value       = [for x in module.container[*] : x]
  description = "The name and socket for each application."
}

# --- container/variables.tf ---

variable "name_in" {}
variable "image_in" {}
variable "int_port_in" {}
variable "ext_port_in" {}
variable "count_in" {}
variable "volumes_in" {}

# --- container/main.tf ---

resource "docker_container" "app_container" {
  count = var.count_in
  name  = join("-", [var.name_in, terraform.workspace, random_string.random[count.index].result])
  image = var.image_in
  ports {
    internal = var.int_port_in
    external = var.ext_port_in[count.index]
  }
  dynamic "volumes" {
    for_each = var.volumes_in
    content {
      container_path = volumes.value["container_path_each"]
      volume_name    = module.volume[count.index].volume_output[volumes.key]
    }
  }

module "volume" {
  source       = "./volume"
  count        = var.count_in
  volume_count = length(var.volumes_in)
  volume_name  = "${var.name_in}-${terraform.workspace}-${random_string.random[count.index].result}-volume"
}

# --- container/outputs.tf ---

output "application_access" {
  value = { for x in docker_container.app_container[*] : x.name => join(":", [x.network_data[0].ip_address], x.ports[*]["external"]) }
}

# --- container/volume/variables.tf ---

variable "volume_count" {}
variable "volume_name" {}

# --- container/volume/main.tf ---
resource "docker_volume" "container_volume" {
  count = var.volume_count
  name  = "${var.volume_name}-${count.index}"
  lifecycle {
    prevent_destroy = false
  }
  provisioner "remote-exec" {
    when       = destroy
    on_failure = continue
    connection {
      host        = "x.x.x.x"
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("../../id_rsa")
    }
    inline = [
      "mkdir /home/ubuntu/environment/backup/"
    ]
  }
  provisioner "remote-exec" {
    when       = destroy
    on_failure = fail
    connection {
      host        = "x.x.x.x"
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("../../id_rsa")
    }
    inline = [
      "sudo tar -czvf /home/ubuntu/environment/backup/${self.name}.tar.gz ${self.mountpoint}/"
    ]
  }
}

#container/volume/outputs.tf

output "volume_output" {
  value = docker_volume.container_volume[*].name
}
```

```bash
$ sudo tree /var/lib/docker/volumes
/var/lib/docker/volumes
├── 1d3f5154b76b183837c385fe30b34060eb6aff9e75560ac4c84631112c650904
│   └── _data
├── backingFsBlockDev
├── be650d694e83103874d9e889bd697080216be219f4a15b0c6bbf93619391def3
│   └── _data
├── grafana-dev-qt6e-volume-0
│   └── _data
│       ├── alerting
│       │   └── 1
│       │       └── __default__.tmpl
│       ├── csv
│       ├── grafana.db
│       ├── plugins
│       └── png
├── grafana-dev-qt6e-volume-1
│   └── _data
│       ├── grafana.ini
│       ├── ldap.toml
│       └── provisioning
│           ├── access-control
│           ├── alerting
│           ├── dashboards
│           ├── datasources
│           ├── notifiers
│           └── plugins
├── grafana-dev-zo9d-volume-0
│   └── _data
│       ├── alerting
│       │   └── 1
│       │       └── __default__.tmpl
│       ├── csv
│       ├── grafana.db
│       ├── plugins
│       └── png
├── grafana-dev-zo9d-volume-1
│   └── _data
│       ├── grafana.ini
│       ├── ldap.toml
│       └── provisioning
│           ├── access-control
│           ├── alerting
│           ├── dashboards
│           ├── datasources
│           ├── notifiers
│           └── plugins
└── metadata.db

36 directories, 10 files
```

### 68. Weatherboard Dashboard Project

- Build weather monitoring app to monitor weather for rocket launches in Houston and Kennedy Space Center
- Use nodered to pull from the openweather map api, insert the data into influxdb, and then use grafana to graph the data

1. Signup and get API key from openweathermap.org
2. Setup default bucket in influxdb called "weather". Copy API key.
3. Use nodered template found in resources. Add API key for http request to openweathermap.org, add influxdb API key and bucket name
4. Add influxdb as a data source in grafana and import the dashboard found in resources

## Section 4: Deploy AWS Resources with Terraform

### 69. What we're going to build

![AWS VPC](../assets/images/terraform/image-2.png)

- Standard 3 tier app with a K3 control plane and worker node using free tier (T2 micro for K3s)
- K3s doesn't use etcd but instead can use MySQL or PostgreSQL (Amazon RDS)
- Use provisioners to copy kubeconfig settings files from control plane to the Cloud9 control node. This will allow k8s commands to be run from the node to the cluster automatically using terraform.

### 70a. Configuring Terraform Cloud

- CLI driven workflow = Terraform workflow commands are run remotely via CLI (see execution modes below)
- API driven workflow = Use TFC APIs to initiate workflow
- Git driven workflow = changes in git trigger workflow
- Execution modes for a workspace:
  - Remote (default) = Plan and apply occur on TFC infrastructure and reviews are done in the GUI
  - Local = Plan and apply occur locally and TFC is only used for state synchronisation
- After specifying terraform cloud org and workspace in backends.tf use `terraform login` to request an API token to allow local TF to use TFC as state backend

### 70b. Using Dev Container (Ubuntu) to deploy to AWS with TFC backend

- [Assuming roles using terraform (Reddit)](https://www.reddit.com/r/Terraform/comments/tkkn04/best_authentication_method_for_aws/)
- [Authenticating to AWS Provider with Terraform (Reddit)](https://www.reddit.com/r/Terraform/comments/x0o3av/authenticating_to_aws_provider/)
- [aws-vault env vars](https://github.com/99designs/aws-vault/blob/master/USAGE.md#environment-variables)

1. Install [Password Store](https://www.passwordstore.org/) and [aws-vault](https://github.com/99designs/aws-vault/blob/master/README.md) in dockerfile

```dockerfile
RUN curl -L -o /usr/local/bin/aws-vault https://github.com/99designs/aws-vault/releases/latest/download/aws-vault-linux-$(dpkg --print-architecture)
RUN chmod 755 /usr/local/bin/aws-vault
RUN apt-get update && apt-get install -y \
    pass 
```

2. Create GPG key to use with aws-vault, init pass, set env vars, add AWS account, login to TFC

```bash
gpg --full-generate-key #Follow prompts
gpg --list-secret-keys --keyid-format LONG #Only needed if key-id isn't recorded when output
pass init <gpg key-id>
export AWS_VAULT_PASS_PREFIX=aws-vault
export AWS_VAULT_BACKEND=pass
export AWS_DEFAULT_REGION=ap-southeast-2
export AWS_MFA_SERIAL=arn:aws:iam::x:mfa/xxxxxxxxx
export GPG_TTY=$(tty)
aws-vault add aws-profile #Enter AWS Key ID & Secret Access Key
aws-vault exec aws-profile #Enter AWS OTP
terraform login #Enter TFC token
```

Example of AWS key and STS token in password-store:

```bash
$ tree ~/.password-store/
/home/vscode/.password-store/
└── aws-vault
    ├── aws-profile.gpg
    └── sts.GetSessionToken.gpg

$ pass
Password Store
└── aws-vault
    ├── aws-profile
    └── sts.GetSessionToken
```

After MFA Prompt, aws-vault opens a subshell with temp STS credentials added as env vars. Terraform will pick up the access key, secret access key, and session token from here. It will assume the specified role in the provider config and then execute the commands.

```terraform
provider "aws" {
  region = var.aws_region

  assume_role {
    duration = "1h"
    role_arn = "arn:aws:iam:::role/mtc-terraform"
  }
}
```

### 71. The AWS Provider

<https://registry.terraform.io/providers/hashicorp/aws/latest/docs>

### 72. Deploy a VPC

<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc>

### 73. Remote State in Terraform Cloud

- TFC workspace settings:
  - Add SSH keys for downloading TF modules from Git
  - Manually lock state
  - Deny destroy plans

### 76. The cidrsubnet() Function

<https://developer.hashicorp.com/terraform/language/functions/cidrsubnet>

- `range (a,b,c)` a = min, b = max, c = step

```terraform
terraform console
> cidrsubnet("10.123.0.0/16", 8, 20)
"10.123.20.0/24"
> range (1, 6, 2)
tolist([
  1,
  3,
  5,
])

# --- root/main.tf ---

module "networking" {
  source = "./networking"
  vpc_cidr = "10.123.0.0/16"
  public_sn_count = 2
  private_sn_count = 3
  public_cidrs = [for i in range(2, 255, 2) : cidrsubnet("10.123.0.0/16", 8, i)]
  private_cidrs = [for i in range(1, 255, 2) : cidrsubnet("10.123.0.0/16", 8, i)]
}

# --- networking/main.tf ---

resource "aws_subnet" "mtc_public_subnet" {
  count = var.public_sn_count
  vpc_id = aws_vpc.mtc_vpc.id
  cidr_block = var.public_cidrs[count.index]
  map_public_ip_on_launch = true #defaults to false
  availability_zone = ["ap-southeast-2a","ap-southeast-2b","ap-southeast-2c"][count.index]

  tags = {
    Name = "mtc_public_${count.index + 1}"
  }
}

$ terraform state list | grep public
module.networking.aws_subnet.mtc_public_subnet[0]
module.networking.aws_subnet.mtc_public_subnet[1]
```

### 77. The aws_availability_zones Data Source + 78. The random_shuffle Resource

<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/availability_zones>

```terraform
# --- networking/main.tf ---

data "aws_availability_zones" "available" {}

resource "random_shuffle" "az_list" {
  input = data.aws_availability_zones.available.names
  result_count = var.max_subnets
}

resource "aws_subnet" "mtc_public_subnet" {
  ...
  availability_zone = random_shuffle.az_list.result[count.index]
  ...
}
```

### 80. Route tables & Internet Gateway

<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/route_table>
<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/default_route_table>
<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet_gateway>

- One internet gateway per VPC (no count etc. required)
- Caveat around the default route table - it's an existing resource that terraform "adopts". All routes will be removed/replaced

```terraform
resource "aws_default_route_table" "mtc_private_rt" {
  default_route_table_id = aws_vpc.mtc_vpc.default_route_table_id
}

#This sets the default route table in the VPC to the existing default route table that is created during VPC creation rather than assigning a new one
```

### 81. The create_before_destroy Lifecycle Meta Argument

- When making a change to a VPC that requires recreating the VPC (e.g. changing the CIDR block) terraform will not delete the internet gateway. Instead, it will try and associate it to the new VPC which doesn't exist.

```bash
  # module.networking.aws_internet_gateway.mtc_internet_gateway will be updated in-place
  ~ resource "aws_internet_gateway" "mtc_internet_gateway" {
        id       = "igw-0196283b5867f33f8"
        tags     = {
            "Name" = "mtc_igw"
        }
      ~ vpc_id   = "vpc-0b39532bb43b7b98c" -> (known after apply)
        # (3 unchanged attributes hidden)
  }
```

- Need to use create_before_destroy lifecycle argument on the VPC to tell Terraform to create a new VPC before destroying the old one

```terraform
resource "aws_vpc" "mtc_vpc" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "mtc_vpc_${random_integer.random.id}"
  }
  lifecycle {
    create_before_destroy = true
  }
}
```

### 82. Security Groups

<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group>

- Always a chance when modifying security groups of causing a very short interruption to traffic
- Can use nested `for_each` to access inner maps within locals and then use dynamic blocks to iterate through them.

```terraform
# --- root/locals.tf ---

locals {
  security_groups = {
    public = {
      name        = "public_sg"
      description = "Security Group for Public Access"
      ingress = {
        ssh = {
          from        = 22
          to          = 22
          protocol    = "tcp"
          cidr_blocks = [var.access_ip]
        }
        http = {
          from        = 80
          to          = 80
          protocol    = "tcp"
          cidr_blocks = ["0.0.0.0/0"]
        }
      }
    }
    rds = {
      name        = "rds_sg"
      description = "RDS Access"
      ingress = {
        mysql = {
          from        = 3306
          to          = 3306
          protocol    = "tcp"
          cidr_blocks = [local.vpc_cidr]
        }
      }
    }
  }
}

# --- root/main.tf ---

module "networking" {
  source           = "./networking"
  vpc_cidr         = local.vpc_cidr
  access_ip        = ""
  security_groups  = local.security_groups
  public_sn_count  = 2
  private_sn_count = 4
  max_subnets      = 20
  public_cidrs     = [for i in range(2, 255, 2) : cidrsubnet(local.vpc_cidr, 8, i)]
  private_cidrs    = [for i in range(1, 255, 2) : cidrsubnet(local.vpc_cidr, 8, i)]
}

# --- networking/main.tf ---

resource "aws_security_group" "mtc_sg" {
  for_each    = var.security_groups
  name        = each.value.name
  description = each.value.description
  vpc_id      = aws_vpc.mtc_vpc.id
  
  dynamic "ingress" {
    for_each = each.value.ingress
    content {
      from_port   = ingress.value.from
      to_port     = ingress.value.to
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### 85. VPC RDS Subnet Group and Conditionals

<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_subnet_group>

- Need a db_subnet to create an RDS
- Conditionals allow you to control how many instances are deployed. Uses logical "if variable == value, then x, else y". Example:

```terraform
> "us-east-1" == "us-east-1" ? 3: 1
3
> "us-west-1" == "us-east-1" ? 3: 1
1
> true == true ? 1 : 0
1
> false == true ? 1 : 0
0
> true ? 1 : 0
1
> false ? 1 : 0
0
```

- Conditional bool is useful to control whether a resource is deployed based on a flag. If `db_subnet_group = false` then subnet_group does not deploy
- `count = var.db_subnet_group == true ? 1 : 0` can be shortened to `count = var.db_subnet_group ? 1 : 0`

```terraform
# --- root/main.tf ---

module "networking" {
  source           = "./networking"
  ...
  db_subnet_group  = true
}

resource "aws_db_subnet_group" "mtc_rds_subnetgroup" {
  count = var.db_subnet_group == true ? 1 : 0
  name = "mtc_rds_subnetgroup"
  subnet_ids = aws_subnet.mtc_private_subnet.*.id
  tags = {
    "Name" = "mtc_rds_sng"
  }
}
```

### 86. Basic RDS Setup

<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance>

- Most K8s deployments you would use etcd but in this case using Rancher K3s it allows you to use a standard RDS (MySQL or Postgres) to store the data

### 89. The terraform providers schema command

- Shows the entire AWS schema. 65 Mbps file, 890000 lines
- example of where password is marked as sensitive in aws_db_instance

```json
terraform providers schema -json | jq '.' > schema.json

        "aws_db_instance": {
          "version": 2,
          "block": {
            "attributes": {
              "password": {
                "type": "string",
                "description_kind": "plain",
                "optional": true,
                "sensitive": true
              }
```

### 90. ALB setup

<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb>

- Load balances K3s in public subnet. Usually these would be in private subnets, but there's no bastion host or NAT gateway in this scenario
- Subnets and security groups similar to `aws_db_instance`

### 92. ALB Target Group and the UUID and substr functions

<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group>
<https://developer.hashicorp.com/terraform/language/functions/uuid>
<https://developer.hashicorp.com/terraform/language/functions/substr>

- use instead of the random function for naming

```terraform
> uuid()
b5ee72a3-54dd-c4b8-551c-4bdc0204cedb

substr(string, offset, length)

> substr (uuid(), 0, 4)
"b5ee"

resource "aws_lb_target_group" "mtc_tg" {
  name = "mtc-lb-tg-${substr(uuid(), 0, 3)}"
}
```

- Note, each time terraform apply is ran it will generate a new UUID. If this is used in a resource name it may try and destroy/re-create the resource each time (as is the case with ALB target group)

- [create before destroy when changing lb_target_group port](https://stackoverflow.com/questions/57183814/error-deleting-target-group-resourceinuse-when-changing-target-ports-in-aws-thr)

### 93. ALB listener

<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener>

- Receives the external traffic and forwards to target group

### 94. ALB lifecycle Policies: ignore_changes and create_before_destroy

<https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle>

- To prevent the alb target group from being replaced on each run using the UUID function for generating the name, use the ignore_changes lifecycle policy
- If replacing the target group (for example after the name is updated) the listener needs to reference the new target group before the old one is destroyed
- Solution: Use the create_before_destroy lifecycle policy on the target group

```terraform
resource "aws_lb_target_group" "mtc_tg" {
  name     = "mtc-lb-tg-${substr(uuid(), 0, 3)}"
  port     = var.tg_port
  protocol = var.tg_protocol
  vpc_id   = var.vpc_id
  lifecycle {
    ignore_changes = [name]
    create_before_destroy = true
  }
  health_check {
    healthy_threshold   = var.lb_healthy_threshold
    unhealthy_threshold = var.lb_unhealthy_threshold
    timeout             = var.lb_timeout
    interval            = var.lb_interval
  }
}
```

### 96. The aws_ami datasource

<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ami>

- Use awscli to search and sort Ubuntu images

```bash
aws ec2 describe-images --owners 099720109477 --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" --query 'Images[*].[ImageId, Name]' --output table | awk '{print $4,$2}' | sort

Latest image: ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-20240207.1 ami-0d6f74b9139d26bf1
```

- How to always get the latest version in terraform

```terraform
# --- compute/main.tf ---

data "aws_ami" "server_ami" {
  most_recent = true
  owners = ["099720109477"]

  filter {
    name = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}
```

### 97. EC2 instances

<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance>

### 98. SSH Key for Instance

<https://www.udemy.com/course/terraform-certified/learn/lecture/24789250#notes>

- Pass public key as a file rather than a string

```terraform
# --- compute/main.tf ---

resource "aws_key_pair" "mtc_auth" {
  key_name = var.key_name
  public_key = file(var.public_key_path)
}

# --- root/main.tf ---

module "compute" {
  source          = "./compute"
  ...
  public_key_path = "../../id_rsa.pub"
}
```

### 100. Controlling random_id Changes with Keepers

<https://registry.terraform.io/providers/hashicorp/random/latest/docs#resource-keepers>

- Use the random_id resource in some cases as the UUID method `name = "mtc-lb-tg-${substr(uuid(), 0, 3)}"` may not always generate what's required. UUID also changes on each apply which is usually not wanted.
- Keepers trigger a new random ID resource when the consumer of that resource is destroyed
- The resources all provide a map argument called keepers that can be populated with arbitrary key/value pairs that when one of the values is modified, a new random ID should be generated. Otherwise, random_id only generates randomness when it is first created.
- The key_name in the keepers block determines when a new random_id should be created (when the key_name is changed)

```terraform
resource "random_id" "mtc_node_id" {
  byte_length = 2
  count       = var.instance_count
  keepers = {
    #Generate a new id each time a new key_name is changes. A instance is replaced by terraform whenever the key changes.
    key_name = var.key_name
  }
}
```

### 102. EC2 User Data and Template Files

<https://docs.k3s.io/installation>
<https://developer.hashicorp.com/terraform/language/functions/templatefile>

- Template files: Pass template file path and variables from terraform deployment `templatefile(path, vars)`, and it will generate the required output (can do JSON & YAML)
- Similar to Ansible Jinja template

```terraform
# --- root/main.tf ---

module "compute" {
  source          = "./compute"
  ...
  user_data_path  = "${path.root}/userdata.tpl"
  db_endpoint     = module.database.db_endpoint
  dbname          = var.dbname
  dbuser          = var.dbuser
  dbpassword      = var.dbpassword
}

# --- compute/main.tf ---

resource "aws_instance" "mtc_node" {
  ...
  user_data = templatefile(var.user_data_path,
    {
      nodename    = "mtc-${random_id.mtc_node_id[count.index].dec}"
      db_endpoint = var.db_endpoint
      dbname      = var.dbname
      dbuser      = var.dbuser
      dbpass      = var.dbpassword
    }
  )
}

# --- userdata.tpl ---

#!/bin/bash
sudo hostnamectl set-hostname ${nodename} &&
curl -sfL https://get.k3s.io | sh -s - server \
--datastore-endpoint="mysql://${dbuser}:${dbpass}@tcp(${db_endpoint})/${dbname}" \
--write-kubeconfig-mode 644 \
--tls-san=$(curl http://169.254.169.254/latest/meta-data/public-ipv4) \
--token="th1s1sat0k3n!"
```

### 104. Deploying NGINX on Kubernetes Cluster!

```yaml
# --- deployment.yaml ---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      # manage pods with the label app: nginx
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          hostPort: 8000
```

```bash
ubuntu@mtc-28454:~$ kubectl get nodes
NAME        STATUS   ROLES                  AGE     VERSION
mtc-28454   Ready    control-plane,master   3m40s   v1.28.6+k3s2
ubuntu@mtc-28454:~$ vim deployment.yaml
ubuntu@mtc-28454:~$ kubectl apply -f deployment.yaml 
deployment.apps/nginx created
ubuntu@mtc-28454:~$ kubectl get pods
NAME                     READY   STATUS              RESTARTS   AGE
nginx-77d6466568-77dmw   0/1     Pending             0          8s
nginx-77d6466568-j7bxx   0/1     ContainerCreating   0          8s
ubuntu@mtc-28454:~$ curl localhost:8000
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
```

- Only 1x K3s node so 1/2 NGINX replicas is Pending
- Using RDS to share the state of the cluster, so as soon as the 2nd K3s instance is deployed it will join the cluster

```bash
ubuntu@mtc-28454:~$ kubectl get nodes
NAME        STATUS   ROLES                  AGE   VERSION
mtc-28454   Ready    control-plane,master   14m   v1.28.6+k3s2
mtc-8953    Ready    control-plane,master   46s   v1.28.6+k3s2
ubuntu@mtc-28454:~$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-77d6466568-j7bxx   1/1     Running   0          7m38s
nginx-77d6466568-77dmw   1/1     Running   0          7m38s
```

### 105. Add K3 Instances to ALB target group

<https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group_attachment>

- Configure target group attachment which assigns instances (targets) to an ALB target group

```terraform
# --- loadbalancing/outputs.tf ---

output "lb_target_group_arn" {
  value = aws_lb_target_group.mtc_tg.arn
}

# --- root/main.tf ---

module "compute" {
  source              = "./compute"
  instance_count      = 2
  ...
  lb_target_group_arn = module.loadbalancing.lb_target_group_arn
}

# --- compute/main.tf ---

resource "aws_lb_target_group_attachment" "mtc_tg_attach" {
  count            = var.instance_count
  target_group_arn = var.lb_target_group_arn
  target_id        = aws_instance.mtc_node[count.index].id
  port             = 8000
}
```

### 106. Adding Outputs for our Resources

- Output to show ALB DNS name

```terraform
# --- loadbalancing/outputs.tf ---

output "lb_endpoint" {
  value = aws_lb.mtc_lb.dns_name
}

# --- root/outputs.tf ---

output "load_balancer_endpoint" {
  value = module.loadbalancing.lb_endpoint
}
```

```bash
 $ terraform refresh
module.networking.random_integer.random: Refreshing state... [id=41]
module.compute.random_id.mtc_node_id[1]: Refreshing state... [id=Ivk]
...
Outputs:

load_balancer_endpoint = "mtc-loadbalancer-1200784660.ap-southeast-2.elb.amazonaws.com"
```

- Terraform refresh will refresh state and show the outputs without having to do an apply. Can also do a `terraform output` once the outputs have been created the first time.

### 107. Sensitive Outputs and how to access them

- Create output for EC2 instances name and IP

```terraform
# --- compute/outputs.tf ---

output "instance" {
  value = aws_instance.mtc_node[*]
}

# --- root/outputs.tf ---

output "instances" {
  value = {for i in module.compute.instance : i.tags.Name => i.public_ip}
}
```

```bash
terraform refresh
Outputs:

instances = {
  "mtc_node-28454" = "3.106.116.78"
  "mtc_node-8953" = "13.210.137.182"
}
load_balancer_endpoint = "mtc-loadbalancer-1200784660.ap-southeast-2.elb.amazonaws.com"
```

- In older versions of Terraform when you send all attributes of the instances in compute/outputs.tf output you would need to add `sensitive = true` to the outputs in the compute and root modules. Then you would need to output in JSON to get the values. With later versions you don't need this workaround because you aren't outputting the sensitive values in the root module.

```bash
terraform output -json | jq '."instances"."value"'
{
  "mtc_node-28454": "3.106.116.78",
  "mtc_node-8953": "13.210.137.182"
}
```

### 109. Utilising the remote and local-exec provisioner to SCP the Kubeconfig file

- Install Kubectl to run K8s commands locally in devcontainer. Add to dockerfile

```docker
RUN curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
RUN echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list

RUN apt-get update && apt-get install -y \
    kubectl
```

- Remote provisioner will run delay.sh to check for the presence of k3s.yaml on the remote node, so it knows when the node is operational.
- Local provisioner copies k3s YAML config locally from k3s node and replaces local IP with the public IP. Then you can run kubectl commands locally and access the remote node.
- Local provisioner will remove the k3s config file upon destroy

```terraform
# --- compute/main.tf ---

resource "aws_instance" "mtc_node" {
    ...
    provisioner "remote-exec" {
    connection {
      type        = "ssh"
      host        = self.public_ip
      user        = "ubuntu"
      private_key = file("${path.cwd}/../../id_rsa")
    }
    script = "${path.cwd}/delay.sh"
    }
    provisioner "local-exec" {
    command = templatefile("${path.cwd}/scp_script.tpl",
      {
        nodeip          = self.public_ip
        k3s_path        = "${path.cwd}/../../"
        nodename        = self.tags.Name
        privatekey_path = "${path.cwd}/../../id_rsa"
      }
    )
    }
    provisioner "local-exec" {
    when    = destroy
    command = "rm -f ${path.cwd}/../../k3s-${self.tags.Name}"
  }
}
```

```bash
# --- delay.sh ---

#!/bin/bash
while [ ! -f /etc/rancher/k3s/k3s.yaml ]; do
    echo -e "Waiting for k3s to bootstrap..."
    sleep 3
done

# --- scp_script.tpl ---

scp -i ${privatekey_path} \
-o StrictHostKeyChecking=no \
-o UserKnownHostsFile=/dev/null \
-q ubuntu@${nodeip}:/etc/rancher/k3s/k3s.yaml ${k3s_path}/k3s-${nodename}.yaml && 
sed -i 's/127.0.0.1/${nodeip}/' ${k3s_path}/k3s-${nodename}.yaml
```

```yaml
# --- k3s-mtc_node-50332.yaml ---
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ...
    server: https://x.x.x.x:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: ...
    client-key-data: ...
```

```bash
$ export KUBECONFIG=/workspaces/terraform-associate/k3s-mtc_node-50332.yaml
$ kubectl get nodes
NAME        STATUS   ROLES                  AGE     VERSION
mtc-50332   Ready    control-plane,master   4m41s   v1.28.6+k3s2
```

### 112. Deploying K8s Resources from our Cloud9 Instance

```yaml
# --- nginx-dep.yaml ---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      # manage pods with the label app: nginx
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          hostPort: 8000
```

```bash
$ kubectl get nodes
NAME        STATUS   ROLES                  AGE   VERSION
mtc-50332   Ready    control-plane,master   51m   v1.28.6+k3s2
$ kubectl create -f nginx-dep.yaml
deployment.apps/nginx created
$ kubectl get pods
NAME                     READY   STATUS              RESTARTS   AGE
mtc-50332   Ready    control-plane,master   52m   v1.28.6+k3s2
$ terraform output -json | jq '."load_balancer_endpoint"."value"'
"mtc-loadbalancer-574255458.ap-southeast-2.elb.amazonaws.com"
$ curl mtc-loadbalancer-574255458.ap-southeast-2.elb.amazonaws.com
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

## Section 5: Deploying Kubernetes Resources with Terraform  

### 113. Configuring K8s provider

<https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs>

### 114. Our First K8s Deployment

- For large K8s deployment, Terraform is not the right tool. Don't always play well together. Probably best to use native k8s tools or Helm/ArgoCD
- This deployment is node-red containers (behind ALB) with their own local volume storage (not shared EFS, EBS etc.)

```terraform
# --- root/main.tf ---

resource "kubernetes_deployment" "iotdep" {
  metadata {
    name = "iotdep"
    labels = {
      app = "iotapp"
    }
  }

  spec {
    replicas = 1

    selector {
      match_labels = {
        app = "iotapp"
      }
    }

    template {
      metadata {
        labels = {
          app = "iotapp"
        }
      }

      spec {
        container {
          image = "nodered/node-red:latest"
          name  = "nodered-container"
          volume_mount {
            name       = "nodered-vol"
            mount_path = "/data"
          }
          port {
            container_port = 1880
            host_port      = 8000
          }
        }
        volume {
          name = "nodered-vol"
          empty_dir {
            medium = ""
          }
        }
      }
    }
  }
}
```

### 116. Terraform Apply Yourself - 3 k8s deployments using for_each

- Deploy nodered, influxdb, and grafana using the one deployment resource

```terraform
# --- root/locals.tf ---

locals {
  deployment = {
    nodered = {
      image      = "nodered/node-red:latest"
      int        = 1880
      ext        = 1880
      volumepath = "/data"
    }
    influxdb = {
      image      = "influxdb"
      int        = 8086
      ext        = 8086
      volumepath = "var/lib/influxdb"
    }
    grafana = {
      image      = "grafana/grafana"
      int        = 3000
      ext        = 3000
      volumepath = "var/lib/grafana"
    }
  }
}

# --- root/mains.tf ---

resource "kubernetes_deployment" "iotdep" {
  for_each = local.deployment
  metadata {
    name = "${each.key}-dep"
    labels = {
      app = "iotapp"
    }
  }

  spec {
    replicas = 1

    selector {
      match_labels = {
        app = "iotapp"
      }
    }

    template {
      metadata {
        labels = {
          app = "iotapp"
        }
      }

      spec {
        container {
          image = each.value.image
          name  = "${each.key}-container"
          volume_mount {
            name       = "${each.key}-vol"
            mount_path = each.value.volumepath
          }
          port {
            container_port = each.value.int
            host_port      = each.value.ext
          }
        }
        volume {
          name = "${each.key}-vol"
          empty_dir {
            medium = ""
          }
        }
      }
    }
  }
}
```

```bash
$ kubectl get deployments
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nodered-dep    0/1     1            0           3m16s
grafana-dep    1/1     1            1           3m16s
influxdb-dep   1/1     1            1           3m16s

$ kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
grafana-dep-66f59cdd49-xlz7n    1/1     Running   0          4m8s
influxdb-dep-69bb978bfd-hjrxk   1/1     Running   0          4m8s
nodered-dep-5867bf8c7c-5str7    1/1     Running   0          4m8s
```

![nodered and influxdb running](../assets/images/terraform/image-3.png)

### 117. Terraform_remote_state datasource and the split Function

<https://developer.hashicorp.com/terraform/language/state/remote-state-data>

- Pull values from a remote state file. Recommendation is to not use this if possible. You need full access to the workspace state. Better to use tfe_outputs data source.
- Use the output from the terraform-aws state file which shows the env var with the location of the K8s config file. Then use split to just get the file location.

```terraform
$ terraform console
> split("=", "export KUBECONFIG=/workspaces/terraform-associate/terraform-aws/terraform-aws/../../k3s-mtc_node-59231.yaml")[1]
"/workspaces/terraform-associate/terraform-aws/terraform-aws/../../k3s-mtc_node-59231.yaml"
> split("=", "export KUBECONFIG=/workspaces/terraform-associate/terraform-aws/terraform-aws/../../k3s-mtc_node-59231.yaml")
tolist([
  "export KUBECONFIG",
  "/workspaces/terraform-associate/terraform-aws/terraform-aws/../../k3s-mtc_node-59231.yaml",
])
```

- Accessing tfstate from terraform-aws in TFC. `data.terraform_remote_state.kubeconfig.outputs.kubeconfig` has the location of the K8s config file.
  
```terraform
# --- root/datasources.tf ---

data "terraform_remote_state" "kubeconfig" {
  backend = "remote"

  config = {
    organization = "mtc-terraform-th"
    workspaces = {
      name = "mtc-dev"
    }
  }
}

# --- root/providers.tf ---

locals {
  config = data.terraform_remote_state.kubeconfig.outputs.kubeconfig
}
terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.26.0"
    }
  }
}

provider "kubernetes" {
  config_path = split("=",local.config[0])[1]
}
```

## Section 6: Continuous Deployment using Terraform Cloud

### 118. What We're Going to Build

![Deployment overview](../assets/images/terraform/image-4.png)

- CI/CD pipeline
- GitHub Networking & Compute Repos hosting modules
- GitHub hosting Deployment repo (main.tf)
- Oauth GitHub with Terraform Cloud so that when a change is made in the module repos it will be reflected in the modules in TFC. Also, so that a change in the deployment repo will deploy in TFC using a deployment workspace.
- Deployment envars in TFC will contain AWS credentials

### 120. Setting up GitHub

- Pull repo <https://github.com/derekops/terraform-aws-networking>
- GitHub > Settings > Developer settings > PAT
- When creating a module repo, the repo name must start with terraform-aws or TFC will not pick it up as a module
- Output below: clone repo, remove git, initialise a new repo, create new repo in GitHub GUI, then add the GitHub repo as origin source

```bash
git clone https://github.com/derekops/terraform-aws-networking.git
rm -rf .git 
git init && git add .
git commit -m "Git initial"
git branch -M main
git remote add origin https://github.com/liftconfig/terraform-aws-networking.git
git status
```

### 121. Configuring Terraform Cloud

Authorise TFC to access GitHub account. Create an Application in GitHub

1. In terraform Cloud Org settings configure a VSC provider
2. Link to GitHub to register a new OAuth Application <https://github.com/settings/applications/new> TFC provides all the steps to do this.
3. Create a new workspace and chose Version Control Workflow. You will be able to select GitHub as a VCS provider.
4. Choose a repository from GitHub

### 122. Our First Cloud Deployment

- TFC will pick up any variable file `*.auto.tfvars` and automatically use the variables in the configuration.
- When creating variables if you are using anything other than a string (list, map etc.) make sure to tick the HCL box or it will be inside quotes like a string.

### 123. TFC Repo-based modules

- Publish private modules in TFC under registry > modules
- Select VSC Provider and then a repo for the module (same as a VSC workflow workspace)
- "Module source repository has no tags" - A module requires Git tagging before it can be utilised.

```bash
$ git tag -a v1.0.0 -m "first version"
$ git tag
v1.0.0
$ git push origin v1.0.0
To https://github.com/liftconfig/terraform-aws-networking.git
 * [new tag]         v1.0.0 -> v1.0.0
```

![TFC Module inputs](../assets/images/terraform/image-5.png)
![TFC Module resources](../assets/images/terraform/image-7.png)

- Modules won't pick up the `*.auto.tfvars` file, so this needs to be specified outside the repository.

### 124. Utilising the Configuration Designer to Create a Deployment

- Registry (where you can see your modules) > Design configuration > Select modules
- Add variables for each of the modules. You can use the outputs of other modules as inputs

![TFC Deployment](../assets/images/terraform/image-8.png)

- You can then copy the code output and publish to a new VSC repo

![TFC Publish config to VSC](../assets/images/terraform/image-9.png)

```terraform
# --- mtc-control/deployments/mtc-dev/main.tf ---

//--------------------------------------------------------------------
// Variables

//--------------------------------------------------------------------
// Modules
module "compute" {
  source  = "app.terraform.io/mtc-terraform-th/compute/aws"
  version = "1.0.1"

  aws_region = "ap-southeast-2"
  public_key_material = "ssh-rsa "
  public_sg = "${module.networking.public_sg}"
  public_subnets = "${module.networking.public_subnets}"
}

module "networking" {
  source  = "app.terraform.io/mtc-terraform-th/networking/aws"
  version = "1.0.0"

  access_ip = "x.x.x.x"
  aws_region = "ap-southeast-2"
}
```

### 125. Settings up Providers for our New Deployment

- Here we're using terraform code (outside of Git/TFC) to deploy code to Git/TFC
- Deploy the above main.tf code to GitHub, then run using TFC
- Create a new GitHub repository and attach repository to TFC workspace all using Terraform code
- Modules are scoped to the organisation in TFC so the above module source will all work

```terraform
# --- root/providers.tf ---

terraform {
  required_providers {
    github = {
      source = "integrations/github"
      version = "4.13.0"
    }

    tfe = {
      source = "hashicorp/tfe"
    }
  }
}

provider "github" {
  token = var.github_token
  owner = var.github_owner
}

provider "tfe" {
  token = var.tfe_token
}
```

### 127. Configuring our GitHub Resources in Terraform

- In local TF - Create repository, default branch, and upload main.tf

```terraform
# --- root/main.tf ---

resource "github_repository" "mtc_repo" {
  name = "mtc-dev-repo"
  description = "VPC and Compute Resources"
  auto_init  = true
  license_template = "mit"

  visibility = "private"
}

resource "github_branch_default" "default" {
  repository = github_repository.mtc_repo.name
  branch = "main"
}

resource "github_repository_file" "maintf" {
  repository = github_repository.mtc_repo.name
  branch = "main"
  file = "main.tf"
  content = file ("./deployments/mtc-dev/main.tf")
  #not uploading file, you are creating a new file and extracting content from main.tf
  commit_message = "Managed by Terraform"
  commit_author = "tim"
  commit_email = "tim@tim.com"
  overwrite_on_create = true
}
```

### 128. Configuring our Terraform Cloud Resources in Terraform

- In local TF create TFC resources: mtc_oauth client to github, workspace using repo from GitHub, workspace environment variables for AWS

```terraform
# --- root/main.tf ---
resource "tfe_oauth_client" "mtc_oauth" {
  organization     = local.organization
  api_url          = "https://api.github.com"
  http_url         = "https://github.com"
  oauth_token      = var.github_token
  service_provider = "github"
}

resource "tfe_workspace" "mtc_workspace" {
  name         = github_repository.mtc_repo.name
  organization = local.organization
  vcs_repo {
    identifier     = "${var.github_owner}/${github_repository.mtc_repo.name}"
    oauth_token_id = tfe_oauth_client.mtc_oauth.oauth_token_id
  }
}

resource "tfe_variable" "aws_creds" {
  for_each     = local.aws_creds
  key          = each.key
  value        = each.value
  category     = "env"
  sensitive    = true
  workspace_id = tfe_workspace.mtc_workspace.id
  description  = "AWS creds"
}
```

- After the workspace is created a plan will automatically run using the main.tf in the mtc_repo

> DO NOT run destroy from local TF (destroying TFC workspace, GitHub repo etc) without running a destroy from TFC workspace. Otherwise, you are left with all the AWS resources created from the workspace

### 130. Pushing and pulling remote state

- Locally pull the state down from the mtc-dev-repo, modify and push back up. Should be an extremely rare scenario that would call for this.

```terraform
# --- mtc-control/deployments/mtc-dev/main.tf ---

terraform {
  backend "remote" {
    organization = "mtc-terraform-th"
    
    workspaces {
      name = "mtc-dev-repo"
    }
  }
}
```

```bash
terraform init
terraform state pull >> terraform.tfstate
terraform state push terraform.tfstate
Failed to write state: cannot overwrite existing state with serial 1 with a different state that has the same serial
# need to increment the serial # in the state, then you can push back up to TFC.
terraform state push terraform.tfstate
Acquiring state lock. This may take a few moments...
Releasing state lock. This may take a few moments...
```

### 131. Updating our Deployment Modules and Destroying it

- Updated variable in terraform-aws-compute module repo and then publish as new module version

```bash
git add .
git commit -m "updated key name"
[main 20b9a08] updated key name
 1 file changed, 1 insertion(+), 1 deletion(-)
git tag -a v1.0.2 -m "updated keyname"
git push origin v1.0.2

Enumerating objects: 6, done.
Counting objects: 100% (6/6), done.
Delta compression using up to 12 threads
Compressing objects: 100% (4/4), done.
Writing objects: 100% (4/4), 438 bytes | 438.00 KiB/s, done.
Total 4 (delta 2), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To https://github.com/liftconfig/terraform-aws-compute.git
 * [new tag]         v1.0.2 -> v1.0.2
```

- Update module version in main.tf to use new version

```terraform
# --- mtc-control/deployments/mtc-dev/main.tf ---

//--------------------------------------------------------------------
// Variables

//--------------------------------------------------------------------
// Modules
module "compute" {
  source  = "app.terraform.io/mtc-terraform-th/compute/aws"
  version = "1.0.2"

  aws_region = "ap-southeast-2"
  public_key_material = "ssh-rsa "
  public_sg = "${module.networking.public_sg}"
  public_subnets = "${module.networking.public_subnets}"
}

module "networking" {
  source  = "app.terraform.io/mtc-terraform-th/networking/aws"
  version = "1.0.0"

  access_ip = "x.x.x.x"
  aws_region = "ap-southeast-2"
}
```

### Cleaning up

- Run destroy in TFC mtc-dev-repo workspace
- Destroy mtc-dev-repo GitHub repo and TFC workspace
- Destroy resources from aws-networking TFC workspace (created prior to turning into module)
- Delete TFC token, GitHub PAT, AWS Access Key

### Time to complete course

| Date     | Day | Time |
|----------|-----|------|
| 15/01/24 | MON | 3H   |
| 16/01/24 | TUE | 1H   |
| 17/01/24 | WED | 2H   |
| 18/01/24 | THU | 1.5H |
| 19/01/24 | FRI | 2H   |
| 20/01/24 | SAT | 1.5H |
| 21/01/24 | SUN | 1H   |
| 22/01/24 | MON | 2H   |
| 23/01/24 | TUE | 1H   |
| 24/01/24 | WED | 1H   |
| 25/01/24 | THU | 2H   |
| 26/01/24 | FRI | 1H   |
| 28/01/24 | SUN | 2H   |
| 29/01/24 | MON | 3H   |
| 30/01/24 | TUE | 1.5H |
| 31/01/24 | WED | 1H   |
| 03/02/24 | SAT | 1H   |
| 04/02/24 | SUN | 2.5H |
| 05/02/24 | MON | 1H   |
| 06/02/24 | TUE | 1H   |
| 11/02/24 | SUN | 2H   |
| 12/02/24 | MON | 2H   |
| 13/02/24 | TUE | 2H   |
| 14/02/24 | THU | 1.5H |
| 18/02/24 | FRI | 0.5H |
| 19/02/24 | MON | 1H   |
| 20/02/24 | TUE | 0.5H |
| 26/02/24 | MON | 1.5H |
| 27/02/24 | TUE | 1.5H |
| 28/02/24 | WED | 2H   |

- Total = 46.5.
- Course video length = 12.5 = 4x to complete
- Average ~1 hour a day / 7.75 hours a week

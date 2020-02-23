## Setup Docker Swarm on Digital Ocean with Terraform

Containers have really changed the way we deploy and package software making orchestrating them an interesting point of study. Some solutions I've found particularly interesting are Kubernetes, Docker Swarm and AWS ECS. Each with it's own pros and cons but in this article we would be exploring Docker Swarm. Cloud services play a big part in the way containers are used today because they provide and manage compute and storage infrastructure and this help businesses and developers focus on the product. We would set up a Docker Swarm on Digital Ocean and to make the setup easily reproducible we would use Terraform to configure the infrastructure as code.

> 
Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently. Terraform can manage existing and popular service providers as well as custom in-house solutions.

### Prerequisites

- A Digital Ocean account with a generated API token

The architecture we would be configuring with Terraform is a Docker swarm with two Digital Ocean droplet nodes and a Digital Ocean load balancer to distribute traffic between the nodes. The illustration below explains it better.

![Load Balanced](https://favorite-things.s3.us-east-2.amazonaws.com/load-balancing.png)

The first step is to install Terraform on your machine. You can follow the installation guide on the  [Terraform website](https://learn.hashicorp.com/terraform/getting-started/install.html)  to get that done.

Once that's setup, create a new project folder. For this article, I'll name it `terraform-DO-docker-swarm` (DO here is an abbreviation for Digital Ocean). Our folder is ready and we are ready to begin configuring our infrastructure. Create a new file and name it `do-resources.tf`. This is the Terraform configuration file that would house our Digital Ocean resource configurations. To begin we need a few variables to help our configuration be flexible and help avoid committing credentials to version control. We do this declaring the variables in the `do-resources.tf` file.

```
variable "do_token" {
  type = string
}

variable "public_ssh_key_location" {
  type = string
}

variable "private_ssh_key_location" {
  type = string
}
```

The variables should be set in a new file called `terraform.tfvars` this file should not be committed to version control. It's content should look like this:

```
public_ssh_key_location = ""
private_ssh_key_location = ""
do_token = "" 
```

The values should be updated with your Digital Ocean key, path to your public ssh key and private ssh key.

We need to inform Terraform that Digital Ocean would be our cloud provider for this setup and pass the token variable. We do this by specifying a `provider` in the `do-resources.tf` file.

```
# ...variable definition

provider "digitalocean" {
  token = var.do_token
}
```

Next, we configure an ssh key resource and specify the public key to use to connect to our Digital Ocean droplets.

```
# ...

resource "digitalocean_ssh_key" "default" {
  name       = "swarm-ssh"
  public_key = file(var.public_ssh_key_location)
}

```

Our ssh key is configured. We need a Droplet (node) to serve as the Swarm manager so we define a droplet resource for the manager.

```
resource "digitalocean_droplet" "swarm_manager" {
  image    = "ubuntu-18-04-x64"
  name     = "swarm-manager-1"
  region   = "nyc3"
  size     = "s-1vcpu-1gb"
  private_networking = true
  ssh_keys = [digitalocean_ssh_key.default.fingerprint]
}
```

For the sake of simplicity, we would run a replicated nginx container in this swarm. To configure the deployment we need to create a `docker-compose.yml` file. 

```
version: "3.7"

services:
  web:
    image: nginx
    deploy:
      replicas: 6
      restart_policy:
        condition: on-failure
    ports:
    - "8080:80"
```
The `docker-compose.yml` exposes the port `8080` for this service and specifies that 6 replicas should be deployed to the swarm for more redundancy. Feel free to modify it to use any image or configuration you want. We need to deploy the `docker-compose.yml` file on the manager node therefore we have to copy it to the server. This is done by using Terraform's `file` provisioner and specifying a connection configuration to be used by the resource provisioners.

```
resource "digitalocean_droplet" "swarm_manager" {
  # ...
  connection {
    user        = "root"
    type        = "ssh"
    host = self.ipv4_address
    private_key = file(var.private_ssh_key_location)
  }

  provisioner "file" {
    source      = "docker-compose.yml"
    destination = "/srv/docker-compose.yml"
  }
}
```

This copies file or files from the source on the local machine to a destination on the provisioned server.

After copying the `docker-compose.yml` file, we need to install Docker and run some commands when the droplet is provisioned. This can be done using Terraform's `remote-exec` provisioner.

```
resource "digitalocean_droplet" "swarm_manager" {
  # ...
  provisioner "remote-exec" {
    scripts = [
      "scripts/docker-install.sh",
      "scripts/start-swarm.sh"
    ]
  }
}
```

Here, I defined two scripts to be ran sequentially on creation of a droplet. The first script installs docker on the machine and setups firewall rules to allow Docker Swarm ports. It is saved as `scripts/docker-install.sh` and contains the following commands.

```
#!/bin/bash

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Configure firewall to enable Docker Swarm ports
yes | ufw enable

ufw allow 22/tcp
ufw allow 2376/tcp
ufw allow 2377/tcp
ufw allow 7946/tcp
ufw allow 7946/udp
ufw allow 4789/udp
ufw allow 8080/tcp

ufw reload
```

The next script initiates the Docker Swarm on the manager node (droplet) and is saved as `scripts/start-swarm.sh`

```
#!/bin/bash

# Get Droplet's private IP
private_ip=$(hostname -I | cut -d " " -f 3)

# Start Swarm
docker swarm init \
--listen-addr $private_ip \
--advertise-addr $private_ip
```

After the `remote-exec` scripts are run, we need a way to store the Swarm token so we can use it to connect worker nodes to the Swam. To do this, we would use the the `local-exec` provisioner to connect to the droplet via ssh, run the command to get the token and store it in a `token.txt` file on the local machine (this file should be ignored by version control). We do that by adding a `local-exec` provisioner to the manager resource configuration

```
resource "digitalocean_droplet" "swarm_manager" {
  # ...
  provisioner "local-exec" {
    command = "ssh -i ${var.private_ssh_key_location} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@${self.ipv4_address} 'docker swarm join-token -q worker' > token.txt"
  }
}
```

Our manager node is ready. Next, we create a similar resource for our worker node.

```
resource "digitalocean_droplet" "swarm_worker" {
  image    = "ubuntu-18-04-x64"
  name     = "swarm-worker-1"
  region   = "nyc3"
  size     = "s-1vcpu-1gb"
  private_networking = true
  ssh_keys = [digitalocean_ssh_key.default.fingerprint]
}
```

We need to install docker and join the preconfigured Docker Swarm on creation of the worker node. To do this we add a `remote-exec` provisioner to the *swarm_worker* node resource configuration.

```
resource "digitalocean_droplet" "swarm_worker" {
  # ...
  connection {
    user        = "root"
    type        = "ssh"
    host = self.ipv4_address
    private_key = file(var.private_ssh_key_location)
  }

  provisioner "remote-exec" {
    inline = [
      "sh $(${file("scripts/docker-install.sh")})",
      "docker swarm join --token ${trimspace(file("token.txt"))} ${digitalocean_droplet.swarm_manager.ipv4_address_private}:2377"
    ]
  }
}
```

There's a slight difference in the script here as you can see. `inline` is used instead of `scripts`. This is done to allow the configuration interpolate variables and files into the command. The `token.txt` created after the* manager* node creation is used by the worker to join the Docker Swarm via the manager's private ip address.

After our worker is connected, we can proceed to deploy our Nginx service specified in the `docker-compose.yml` file. A Terraform `local-exec` provisioner  is added to the worker node configuration to make this happen. The `local-exec` connects to the manager via ssh and runs a  `docker stack deploy` command on the *manager* node to deploy the service using the saved `docker-compose` file. 

```
resource "digitalocean_droplet" "swarm_worker" {
  # ...
  provisioner "local-exec" {
    command = "ssh -i ${var.private_ssh_key_location} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@${digitalocean_droplet.swarm_manager.ipv4_address} 'cd /srv && docker stack deploy --compose-file docker-compose.yml testapp'"
  }
}
```

At this point, both nodes in the docker swarm should be properly configured and our containers should be running fine on them. We need one more thing, a load balancer. A load balancer is needed to distribute traffic between the nodes in the Docker Swarm. We configure this by adding a `digitalocean_loadbalancer` resource to the `do-resources.tf` file, specify entry and target ports, a healthcheck port and link droplets to the load balancer.

```
# ...
resource "digitalocean_loadbalancer" "public" {
  name   = "swarm-load-balancer"
  region = "nyc3"

  forwarding_rule {
    entry_port     = 80
    entry_protocol = "http"

    target_port     = 8080
    target_protocol = "http"
  }

  healthcheck {
    port     = 8080
    protocol = "tcp"
  }

  droplet_ids = [digitalocean_droplet.swarm_manager.id, digitalocean_droplet.swarm_worker.id]
}
```

With the final piece in place, we can proceed to add outputs to our configuration to display the IP addresses of the resources we just configured. 

```
# ...
output "swarm_manager_ip" {
  value = digitalocean_droplet.swarm_manager.ipv4_address
}

output "swarm_worker_ip" {
  value = digitalocean_droplet.swarm_worker.ipv4_address
}

output "swarm_loadbalancer_ip" {
  value = digitalocean_loadbalancer.public.ip
}
```

The final `do-resources.tf` file should look like this:

```
variable "do_token" {
  type = string
}

variable "public_ssh_key_location" {
  type = string
}

variable "private_ssh_key_location" {
  type = string
}


provider "digitalocean" {
  token = var.do_token
}

resource "digitalocean_ssh_key" "default" {
  name       = "swarm-ssh"
  public_key = file(var.public_ssh_key_location)
}


resource "digitalocean_droplet" "swarm_manager" {
  image    = "ubuntu-18-04-x64"
  name     = "swarm-manager-1"
  region   = "nyc3"
  size     = "s-1vcpu-1gb"
  private_networking = true
  ssh_keys = [digitalocean_ssh_key.default.fingerprint]

  connection {
    user        = "root"
    type        = "ssh"
    host = self.ipv4_address
    private_key = file(var.private_ssh_key_location)
  }

  provisioner "file" {
    source      = "docker-compose.yml"
    destination = "/srv/docker-compose.yml"
  }

  provisioner "remote-exec" {
    scripts = [
      "scripts/docker-install.sh",
      "scripts/start-swarm.sh"
    ]
  }

  provisioner "local-exec" {
    command = "ssh -i ${var.private_ssh_key_location} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@${self.ipv4_address} 'docker swarm join-token -q worker' > token.txt"
  }
}

resource "digitalocean_droplet" "swarm_worker" {
  image    = "ubuntu-18-04-x64"
  name     = "swarm-worker-1"
  region   = "nyc3"
  size     = "s-1vcpu-1gb"
  private_networking = true
  ssh_keys = [digitalocean_ssh_key.default.fingerprint]

  connection {
    user        = "root"
    type        = "ssh"
    host = self.ipv4_address
    private_key = file(var.private_ssh_key_location)
  }

  provisioner "remote-exec" {
    inline = [
      "sh $(${file("scripts/docker-install.sh")})",
      "docker swarm join --token ${trimspace(file("token.txt"))} ${digitalocean_droplet.swarm_manager.ipv4_address_private}:2377"
    ]
  }

  provisioner "local-exec" {
    command = "ssh -i ${var.private_ssh_key_location} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@${digitalocean_droplet.swarm_manager.ipv4_address} 'cd /srv && docker stack deploy --compose-file docker-compose.yml testapp'"
  }
}

resource "digitalocean_loadbalancer" "public" {
  name   = "swarm-load-balancer"
  region = "nyc3"

  forwarding_rule {
    entry_port     = 80
    entry_protocol = "http"

    target_port     = 8080
    target_protocol = "http"
  }

  healthcheck {
    port     = 8080
    protocol = "tcp"
  }

  droplet_ids = [digitalocean_droplet.swarm_manager.id, digitalocean_droplet.swarm_worker.id]
}

output "swarm_manager_ip" {
  value = digitalocean_droplet.swarm_manager.ipv4_address
}

output "swarm_worker_ip" {
  value = digitalocean_droplet.swarm_worker.ipv4_address
}

output "swarm_loadbalancer_ip" {
  value = digitalocean_loadbalancer.public.ip
}

```

Now we need to apply these configurations to create resources on our Digital Ocean account. If you haven't initiated the Terraform project yet you can do that by running

```bash
terraform init
```

This saves the provider configurations for the setup.

To apply our configurations, run this on your terminal

```bash
terraform apply -var-file=terraform.tfvars --auto-approve
```
It's output should look like:

![Screenshot 2020-02-23 at 17.29.17.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1582475453293/ZG_N0GP5N.png)

Visiting the load balancer IP address on your browser should display the Nginx homepage if everything goes as planned.


![nginx-loadbalanced.png](https://favorite-things.s3.us-east-2.amazonaws.com/nginx-loadbalanced.png)

If you wish to terminate the resources you can run the following command to destroy the setup

```bash
terraform destroy -var-file=terraform.tfvars --auto-approve
```

Source files for these configurations can be found in this [Github repository](https://github.com/otseobande/terraform-DO-docker-swarm).

### References
- [Terraform Digital Ocean Provider Documentation](https://www.terraform.io/docs/providers/do/index.html)
- [Docker Swarm Documentation](https://docs.docker.com/engine/swarm/)
- [SoftLayer's Docker Swarm Setup on Github](https://github.com/softlayer/terraform-provider-softlayer/tree/master/cookbooks/docker-swarm)
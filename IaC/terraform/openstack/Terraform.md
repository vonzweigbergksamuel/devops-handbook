# Terraform Setup Guide - Creating Infrastructure Files

This guide walks you through creating Terraform files for OpenStack infrastructure. We organize files and use variables to make the infrastructure **easy to maintain and modify over time**.

To setup Terraform to deploy the infrastructure to your OpenStack Cumulus using pipeline you can view this repository: [Terraform-Gitlab-LNU-Pipeline](https://github.com/LudwigWittenberg/2dv013-infrastructure-template)

## Table of contents

- [Why We Organize This Way](#why-we-organize-this-way)
- [Prerequisites](#prerequisites)
  - [Step 1: Gather Your OpenStack Details](#step-1-gather-your-openstack-details)
  - [Step 2: Create the `variables.tf` File](#step-2-create-the-variablestf-file)
  - [Step 3: Create the `provider.tf` File](#step-3-create-the-providertf-file)
  - [Step 4: Create the `network.tf` File](#step-4-create-the-networktf-file)
  - [Step 5: Create the `ports.tf` File](#step-5-create-the-portstf-file)
  - [Step 6: Create the `instances.tf` File](#step-6-create-the-instancestf-file)
  - [Step 7: Create the `floating-ip.tf` File](#step-7-create-the-floating-iptf-file)
  - [Step 8: Create the `inventory.tf` File](#step-8-create-the-inventorytf-file)
  - [Step 9: Create the `config.tf` File](#step-9-create-the-configtf-file)
  - [Step 10: Create the `output.tf` File](#step-10-create-the-outputtf-file)
- [Now Deploy](#now-deploy)
- [Key Benefits of This Structure](#key-benefits-of-this-structure)
- [Common Changes (Only Edit variables.tf)](#common-changes-only-edit-variablestf)
- [Cleanup](#cleanup)

## Why We Organize This Way

Instead of one giant file, we split into smaller files because:

- **Easier to maintain** - Each file has one purpose (network, security, instances, etc.)
- **Easier to modify** - Change variables once, applies everywhere
- **Easier to understand** - Clear separation of concerns
- **Easier to reuse** - Use same files for different projects

**The Variables File** is the key - modify only `variables.tf` to change your entire infrastructure. Don't touch code files if you just want different settings.

---

## Prerequisites

Before starting, install:

**Terraform:**

Follow the instructions in the [Terraform Installation Guide](https://learn.hashicorp.com/tutorials/terraform/install-cli) to install Terraform.

**OpenStack CLI:**

Follow the instructions in the [OpenStack CLI Installation Guide](https://docs.openstack.org/newton/user-guide/common/cli-install-openstack-command-line-clients.html) to install the OpenStack CLI.

---

## Step 1: Gather Your OpenStack Details

Before creating files, collect your OpenStack information:

```bash
openstack network list --external
openstack image list
openstack flavor list
openstack keypair list
```

Write down:

- **External Network ID**:
- **Image Name**:
- **Flavor**:
- **SSH Key Name**:
- **Floating IP Pool**:

---

## Step 2: Create the `variables.tf` File

This is the **main configuration file**. Every setting goes here. You only modify THIS file to change your infrastructure.

**Create:** `Terraform/variables.tf`

```hcl
# ========================================
# Terraform Variables Configuration
# ========================================
# Edit ONLY this file to change your infrastructure settings
# All other files reference these variables

# ========================================
# NETWORK CONFIGURATION
# ========================================

variable "subnetwork_cidr" {
  type    = string
  default = "192.168.4.0/24"
  # Change this if you want a different internal network range
}

variable "external_network_id" {
  type        = string
  description = "The ID of the external network from OpenStack"
  default     = "your-external-network-id"
  # Replace with your external network ID from: openstack network list --external
}

# ========================================
# PORT CONFIGURATION
# ========================================

variable "ssh_port" {
  type    = string
  default = "22"
  # Standard SSH port - change only if your OpenStack blocks port 22
}

variable "http_port" {
  type    = string
  default = "80"
  # Web traffic port for webservers - change to 8080 if needed
}

# ========================================
# WEBSERVER CONFIGURATION
# ========================================

variable "webservers_instances" {
  type    = set(string)
  default = ["webserver1"]
  # Scale up: ["webserver1", "webserver2", "webserver3", "webserver4"]
  # Scale down: ["webserver1"]
}

variable "webservers_image" {
  type    = string
  default = "Ubuntu server 24.04.3 autoupgrade"
  # From: openstack image list
  # Example: "Ubuntu 22.04"
}

# ========================================
# FLOATING IP POOL (Public IPs)
# ========================================

variable "floating_ip_pool" {
  type    = string
  default = "campus"
  # From: openstack network list --external
  # Example: "public", "campus", "external"
}

# ========================================
# AUTHENTICATION & GENERAL
# ========================================

variable "key_name" {
  type    = string
  default = "placeholder"
  # Set via environment: export TF_VAR_key_name="your-key-name"
  # From: openstack keypair list
}

variable "identity_file" {
  type    = string
  default = "placeholder"
  # Set via environment: export TF_VAR_identity_file="~/.ssh/your-key.pem"
}

variable "ansible_user" {
  type    = string
  default = "ubuntu"
  # Change to "cloud-user" or "admin" if using different image
}

variable "flavor_id" {
  type    = string
  default = "c1-r1-d10"
  # From: openstack flavor list
  # Example: "m1.small", "m1.medium"
}

variable "project_name" {
  type    = string
  default = "<your project name here>"
  # Used to prefix all resource names
  # Example: "myapp", "production", "testing"
}
```

---

## Step 3: Create the `provider.tf` File

This file tells Terraform "Use OpenStack" and sets required versions.

**Create:** `Terraform/provider.tf`

```hcl
# ========================================
# Terraform Provider Configuration
# ========================================
# Tells Terraform to use OpenStack provider and required versions

terraform {
  required_version = ">= 0.14.0"
  required_providers {
    openstack = {
      source  = "terraform-provider-openstack/openstack"
      version = "~> 1.53.0"
    }
  }
}

# Configure OpenStack Provider
# Authentication via environment variables:
# export OS_AUTH_URL, OS_USERNAME, OS_PASSWORD, OS_PROJECT_NAME, etc.
provider "openstack" {
  auth_url = "https://cumulus.lnu.se:5000/v3"
  # Get from your OpenStack dashboard or from the cumulus.sh file
}
```

---

## Step 4: Create the `network.tf` File

This creates the virtual network, subnet, and router. These are the foundation that all instances sit on.

**Create:** `Terraform/network.tf`

```hcl
# ========================================
# Virtual Network Setup
# ========================================
# Creates the private network where all instances live

# Private network (internal, cannot reach internet without router)
resource "openstack_networking_network_v2" "network" {
  name           = "${var.project_name}-network"
  admin_state_up = "true"
}

# Subnet defines IP range for the network
# All instances get IPs from this range
resource "openstack_networking_subnet_v2" "subnet" {
  network_id = openstack_networking_network_v2.network.id
  cidr       = var.subnetwork_cidr  # From variables.tf (10.0.0.0/24)
  ip_version = 4
}

# Router connects internal network to external network (internet)
resource "openstack_networking_router_v2" "router" {
  name                = "${var.project_name}-router"
  admin_state_up      = true
  external_network_id = var.external_network_id  # From variables.tf
}

# Attach subnet to router so instances can reach internet
resource "openstack_networking_router_interface_v2" "router_interface" {
  router_id = openstack_networking_router_v2.router.id
  subnet_id = openstack_networking_subnet_v2.subnet.id
}
```

---

## Step 5: Create the `ports.tf` File

This creates security groups (firewalls) and network ports. Security groups control which traffic is allowed.

**Create:** `Terraform/ports.tf`

```hcl
# ========================================
# Security Groups & Network Ports
# ========================================
# Security Groups = Firewalls for instances
# Network Ports = Connection points for instances

# ========== SSH SECURITY GROUP ==========
# Allows SSH (port 22) for remote management

resource "openstack_networking_secgroup_v2" "ssh_security_group" {
  name        = "${var.project_name}-ssh-security-group"
  description = "Allows SSH access"
}

resource "openstack_networking_secgroup_rule_v2" "secgroup_rule_ssh" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = var.ssh_port  # From variables.tf (22)
  port_range_max    = var.ssh_port
  remote_ip_prefix  = "0.0.0.0/0"   # Open to everyone
  security_group_id = openstack_networking_secgroup_v2.ssh_security_group.id
}

# ========== HTTP SECURITY GROUP ==========
# Allows HTTP (port 80) for web traffic

resource "openstack_networking_secgroup_v2" "http_security_group" {
  name        = "${var.project_name}-http-security-group"
  description = "Allows HTTP web traffic"
}

resource "openstack_networking_secgroup_rule_v2" "secgroup_rule_http" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = var.http_port  # From variables.tf (80)
  port_range_max    = var.http_port
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = openstack_networking_secgroup_v2.http_security_group.id
}

# ========== NETWORK PORTS - WEBSERVERS ==========
# Each webserver gets a port with HTTP + SSH security groups

resource "openstack_networking_port_v2" "webservers_port" {
  for_each       = var.webservers_instances  # From variables.tf
  name           = "${var.project_name}-port-${each.value}"
  network_id     = openstack_networking_network_v2.network.id
  admin_state_up = "true"
  depends_on     = [openstack_networking_subnet_v2.subnet]
  security_group_ids = [
    openstack_networking_secgroup_v2.http_security_group.id,
    openstack_networking_secgroup_v2.ssh_security_group.id
  ]
}

```

---

## Step 6: Create the `instances.tf` File

This creates the virtual servers (Load Balancer, Webservers, Staging Server).

**Create:** `Terraform/instances.tf`

```hcl
# ========================================
# Compute Instances (Virtual Servers)
# ========================================

# ========== WEBSERVER INSTANCES ==========
# Creates multiple webservers based on var.webservers_instances
# If you have ["webserver1", "webserver2"], creates 2 servers

resource "openstack_compute_instance_v2" "webservers" {
  for_each        = var.webservers_instances  # From variables.tf
  name            = "${var.project_name}-${each.value}"
  image_name      = var.webservers_image      # From variables.tf
  flavor_id       = var.flavor_id             # From variables.tf
  key_pair        = var.key_name              # From variables.tf
  security_groups = [
    openstack_networking_secgroup_v2.http_security_group.name,
    openstack_networking_secgroup_v2.ssh_security_group.name
  ]

  network {
    port = openstack_networking_port_v2.webservers_port[each.key].id
  }

  depends_on = [openstack_networking_port_v2.webservers_port]
}

```

---

## Step 7: Create the `floating-ip.tf` File

This assigns public IPs (Floating IPs) to instances so they can be reached from the internet.

**Create:** `Terraform/floating-ip.tf`

```hcl
# ========================================
# Floating IPs (Public IP Addresses)
# ========================================

# ========== WEBSERVER PUBLIC IP ==========
# Webserver gets its own public IP for testing

resource "openstack_networking_floatingip_v2" "webserver_floating_ip" {
  for_each = var.webservers_instances  # From variables.tf
  pool = var.floating_ip_pool  # From variables.tf
}

resource "openstack_networking_floatingip_associate_v2" "floating_ip_webserver" {
  for_each = var.webservers_instances  # From variables.tf
  floating_ip = openstack_networking_floatingip_v2.webserver_floating_ip[each.key].address
  port_id     = openstack_networking_port_v2.webservers_port[each.key].id
}
```

---

## Step 8: Create the `inventory.tf` File

This generates an Ansible inventory file so you can manage instances with Ansible.

**Create:** `Terraform/inventory.tf`

```hcl
# ========================================
# Ansible Inventory Generation
# ========================================
# Creates a file for Ansible to know how to connect to each instance

resource "local_file" "inventory" {
  filename = "${path.module}/../Ansible/inventory.ini"
  content = <<EOF

# Webservers Group
[webservers]
${join("\n", [for name, _ in openstack_compute_instance_v2.webservers : "${name} ansible_host=${openstack_networking_port_v2.webservers_port[name].all_fixed_ips[0]} ansible_user=${var.ansible_user} private_ip=${openstack_networking_port_v2.webservers_port[name].all_fixed_ips[0]} server_label=${name}"])}

[webservers:vars]
ansible_ssh_common_args='-o ProxyCommand="ssh -i ${var.identity_file} -o IdentitiesOnly=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -W %h:%p ${var.ansible_user}@${openstack_networking_floatingip_v2.webserver_floating_ip[each.key].address}" -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'

# Global variables for all hosts
[all:vars]
ansible_user=${var.ansible_user}
ansible_ssh_private_key_file=${var.identity_file}
webserver_port=${var.http_port}EOF
}
```

---

## Step 9: Create the `config.tf` File

This waits for instances to be ready before deployment continues.

**Create:** `Terraform/config.tf`

```hcl
# ========================================
# Configuration Bootstrap
# ========================================
# Waits for SSH to be available before continuing

resource "null_resource" "wait_ssh" {
  # Connect to webserver to verify it's ready
  for_each = var.webservers_instances  # From variables.tf
  connection {
    type        = "ssh"
    user        = var.ansible_user  # From variables.tf
    private_key = file(var.identity_file)
    host        = openstack_networking_floatingip_v2.webserver_floating_ip[each.key].address
  }

  # Simple command to verify connection works
  provisioner "remote-exec" {
    inline = [
      "echo 'Infrastructure is ready'"
    ]
  }

  # Always check (new check each time)
  triggers = {
    always_run = timestamp()
  }

  depends_on = [
    local_file.inventory,
    openstack_networking_floatingip_associate_v2.floating_ip_load_balancer,
    openstack_networking_router_interface_v2.router_interface
  ]
}
```

---

## Step 10: Create the `output.tf` File

This displays important information after deployment (public IPs, private IPs).

**Create:** `Terraform/output.tf`

```hcl
# ========================================
# Terraform Output Values
# ========================================
# Shows important information after deployment

output "webserver_public_ips" {
  value = {
    for name, _ in openstack_compute_instance_v2.webservers :
    name => openstack_networking_floatingip_v2.webserver_floating_ip[name].address
  }
  description = "Public IPs of webservers"
}
```

---

## Now Deploy

With all files created:

### Step 1: Set Environment Variables if you haven't already

```bash
export OS_AUTH_URL="https://cumulus.lnu.se:5000/v3"
export OS_USERNAME="your-username"
export OS_PASSWORD="your-password"
export OS_PROJECT_NAME="your-project-name"
export OS_REGION_NAME="RegionOne"
export OS_IDENTITY_API_VERSION=3
export TF_VAR_key_name="your-ssh-key-name"
export TF_VAR_identity_file="~/.ssh/your-key.pem"
```

### Step 2: Initialize

```bash
cd Terraform
terraform init
```

### Step 3: Plan

```bash
terraform plan
```

### Step 4: Apply

```bash
terraform apply
```

Type `yes` when prompted.

---

## Key Benefits of This Structure

- ✅ **Easy to Modify** - Change `variables.tf`, everything updates
- ✅ **Easy to Maintain** - Each file has clear purpose
- ✅ **Easy to Scale** - Add more webservers by editing one line in variables
- ✅ **Easy to Understand** - New developers see which file does what
- ✅ **Easy to Reuse** - Use same files for multiple environments

---

## Common Changes (Only Edit variables.tf)

**Add more webservers:**

```hcl
variable "webservers_instances" {
  default = ["webserver1", "webserver2", "webserver3"]
}
```

**Change ports:**

```hcl
variable "load_balancer_port" {
  default = "9000"
}
```

**Use different image:**

```hcl
variable "webservers_image" {
  default = "CentOS 7"
}
```

**Bigger machines:**

```hcl
variable "flavor_id" {
  default = "m1.large"
}
```

Then: `terraform plan && terraform apply`

---

## Cleanup

```bash
terraform destroy
```

Type `yes` to delete everything.

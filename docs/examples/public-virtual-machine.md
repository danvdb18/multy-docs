---
sidebar_position: 1
---

# Public Virtual Machine

Place virtual machine in public subnet with httpd web server.

```hcl
terraform {
  required_providers {
    multy = {
      source = "multycloud/multy"
    }
  }
}

provider "multy" {
  api_key = "xxx"
  aws     = {}
  azure   = {}
}

variable "clouds" {
  type    = set(string)
  default = ["aws", "azure"]
}

resource "multy_virtual_network" "vn" {
  for_each = var.clouds
  cloud    = each.key

  name       = "multy_vn"
  cidr_block = "10.0.0.0/16"
  location   = "eu_west_1"
}

resource "multy_subnet" "subnet" {
  for_each = var.clouds

  name               = "multy_subnet"
  cidr_block         = "10.0.10.0/24"
  virtual_network_id = multy_virtual_network.vn[each.key].id
}

resource "multy_network_security_group" "nsg" {
  for_each = var.clouds
  cloud    = each.key

  name               = "multy_nsg"
  virtual_network_id = multy_virtual_network.vn[each.key].id
  location           = "eu_west_1"
  rule {
    protocol   = "tcp"
    priority   = 132
    from_port  = 80
    to_port    = 80
    cidr_block = "0.0.0.0/0"
    direction  = "both"
  }
}

resource "multy_route_table" "rt" {
  for_each           = var.clouds
  name               = "web_app_rt"
  virtual_network_id = multy_virtual_network.vn[each.key].id
  route {
    cidr_block  = "0.0.0.0/0"
    destination = "internet"
  }
}

resource "multy_route_table_association" "rta" {
  for_each       = var.clouds
  route_table_id = multy_route_table.rt[each.key].id
  subnet_id      = multy_subnet.subnet[each.key].id
}

resource "multy_virtual_machine" "vm" {
  for_each = var.clouds
  cloud    = each.key
  location = "eu_west_1"

  name               = "multy_vm"
  size               = "micro"
  image_reference    = {
    os      = "cent_os"
    version = "8.2"
  }
  subnet_id          = multy_subnet.subnet[each.key].id
  generate_public_ip = true
  user_data_base64   = base64encode(<<-EOF
      #!/bin/bash -xe
      sudo su;
      yum update -y && yum install -y httpd.x86_64;
      systemctl start httpd.service && systemctl enable httpd.service;
      touch /var/www/html/index.html;
      echo "<h1>Hello from Multy on ${each.key}</h1>" > /var/www/html/index.html
    EOF
  )

  depends_on = [multy_network_security_group.nsg]
}

output "aws_endpoint" {
  value = multy_virtual_machine.vm["aws"].public_ip
}

output "azure_endpoint" {
  value = multy_virtual_machine.vm["azure"].public_ip
}
```

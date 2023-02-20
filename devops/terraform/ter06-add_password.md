Terraform 06: Provisioning созданных ресурсов через remote-exec
===

<p>Задание на использование провижинера remote-exec для установки пароля пользователю</p>

<details>
<summary>Содержимое файла <b>main.tf</b> не изменилось</summary>

```yml
terraform {
  required_providers {
    digitalocean = {
      source = "digitalocean/digitalocean"
      version = "~> 2.0"
    }
    aws = {
      source = "hashicorp/aws"
      version = "4.52.0"
    }
  }
}

provider "digitalocean" {
  token = var.do_token
}

provider "aws" {
  access_key = "${var.aws_access_key}"
  secret_key = "${var.aws_secret_key}"
  region     = "us-east-1"
}

```

</details>

<br><p>В файле локальных ресурсах следующие изменения:

* теперь в переменных мы указываем именно приватный ключ, а путь до публичного формируется в переменной локальных ресурсов
* захардкожен пароль (условия задания были именно такие) для пользователя

</p>

<details>
<summary>Содержимое файла <b>locals.tf</b></summary>

```yml
locals {
  # Увидел такое решение по пренеймингу размеров VPC
  # Использование:
  # size = local.sizes.nano

  do_vm_sizes = {
    nano      = "s-1vcpu-1gb"
    micro     = "s-2vcpu-2gb"
    small     = "s-2vcpu-4gb"
    medium    = "s-4vcpu-8gb"
    large     = "s-6vcpu-16gb"
    x-large   = "s-8vcpu-32gb"
    xx-large  = "s-16vcpu-64gb"
    xxx-large = "s-24vcpu-128gb"
    maximum   = "s-32vcpu-192gb"
  }

  # Увидел такое решение по пренеймингу регионов
  # Использование:
  # size = local.regions.frankfurt

  do_regions = {
    new_york_1    = "nyc1"
    new_york_3    = "nyc3"
    san_francisco = "sfo3"
    amsterdam     = "ams3"
    singapore     = "sgp1"
    london        = "lon1"
    frankfurt     = "fra1"
    toronto       = "tor1"
    india         = "blr1"
  }

  do_tags = [var.tag_cources, var.tag_task, var.tag_user_email]

  do_vps_user_name = split("_", split(":", var.tag_user_email)[1])[0]
  do_vm_ip = digitalocean_droplet.srv[*].ipv4_address

  do_public_ssh_key_path = join("", [var.ssh_private_key_user_path, ".pub"])
  do_root_password = "PasswordString"
}

```

</details>

<br><p>В создаваемых ресурсах изменений достаточно, но они не сложные к пониманию. Описывать смысла особого не вижу.</p>

<details>
<summary>Содержимое файла <b>resouces.tf</b></summary>

```yml
resource "digitalocean_droplet" "srv" {
  count    = var.do_vm_count
  image    = var.do_vm_img
  name     = join("", [var.do_vm_name, count.index])
  region   = local.do_regions.frankfurt
  size     = local.do_vm_sizes.nano
  tags     = local.do_tags
  ssh_keys = [data.digitalocean_ssh_key.shared.id, digitalocean_ssh_key.user.id]

  connection {
      host        = self.ipv4_address
      type        = "ssh"
      user        = "root"
      private_key = file(var.ssh_private_key_user_path)
    }

  provisioner "remote-exec" {
    inline = [
      "sleep 10s",
      "echo root:${local.do_root_password} | chpasswd",
      "sed -i '/^PasswordAuthentication/c PasswordAuthentication yes' /etc/ssh/sshd_config",
      "cat /etc/ssh/sshd_config | grep -i ^PasswordAuthentication",
      "/usr/bin/systemctl restart sshd.service"
    ]
  }
}

# первый вариант был такой, а затем выполнен переход на chpasswd
#"/bin/echo -e '${local.do_root_password}\n${local.do_root_password}' | /usr/bin/passwd root",

data "digitalocean_ssh_key" "shared" {
  name = var.ssh_pub_key_shared
}

resource "digitalocean_ssh_key" "user" {
  name       = "Terraform user key"
  public_key = file(local.do_public_ssh_key_path)
}

data "aws_route53_zone" "aws_zone" {
  name = var.aws_zone
}

resource "aws_route53_record" "srv_rec" {
  count = var.do_vm_count
  zone_id = data.aws_route53_zone.aws_zone.zone_id
  name    = "${local.do_vps_user_name}-${count.index}"
  type    = "A"
  ttl     = 300
  records = [local.do_vm_ip[count.index]]
}

```

</details>

<br><p>Здесь мы, как и ранее, убеждаемся что возвращаемые значения именно те, что необходимы нам.</p>

<details>
<summary>Содержимое файла <b>outputs.tf</b></summary>

```yml
output "srv_public_info" {
  description = "DigitalOcean output IPv4 public address"
  value = {
    for index, droplet in digitalocean_droplet.srv :
      droplet.name => {
        dns_record = aws_route53_record.srv_rec[index].name
        ip = droplet.ipv4_address
      }
  }
}

```

</details>

<br><p></p>

<details>
<summary>Содержимое файла <b>variables.tf</b></summary>

```yml
variable "do_token" {
  type = string
  sensitive = true
}

variable "ssh_pub_key_shared" {
  type = string
  sensitive = true
}

variable "ssh_private_key_user_path" {
  type = string
  sensitive = true
}

variable "tag_task" {
  type = string
}

variable "tag_user_email" {
  type = string
}

variable "tag_cources" {
  type = string
}

variable "do_vm_name" {
  type = string
  default = "test"
}

variable "do_vm_img" {
  type = string
  default = "ubuntu-20-04-x64"
}

variable "do_vm_count" {
  type = number
  default = 1
}

variable "aws_zone" {
  type = string
}

variable "aws_access_key" {
  type = string
  sensitive = true
}

variable "aws_secret_key" {
  type = string
  sensitive = true
}

```

</details>

<br><p></p>

<details>
<summary>Содержимое файла <b>terraform.tfvars.sample</b></summary>

```yml
tag_cources = "cources:your_cources"
tag_task = "task_name:your_task"
tag_user_email = "user_email:name_at_1_com"

ssh_pub_key_shared = "SHARED_SSH_KEY"
ssh_private_key_user_path = "PATH_to_YOUR_SSH_KEY"

do_token = "your_token"
do_vm_name = "do_vm-name"
do_vm_img = "ubuntu-20-04-x64"
do_vm_count = number_vm_for_create

aws_access_key = "your_aws_access_key"
aws_secret_key = "your_aws_secret_key"
aws_zone = "example.net"

```

</details>

Навигация
---

* [Вернуться в основное меню](../../README.md)
* [Вернуться в меню обучения по DevOps](../README.md)
* [Вернуться в меню обучения использования Terraform](./README.md)

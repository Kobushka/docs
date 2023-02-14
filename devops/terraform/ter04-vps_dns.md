Выполнение задания
===

<p>Опишите в конфигурации локальную переменную, хранящую IP-адрес созданной VPS, чтобы передать ее в ресурс создания DNS. Дополнительно, опишите ресурс aws_route53_record для создания DNS-записи. DNS запись должна соответствовать шаблону: ВАШ_ЛОГИН.cource.corp.net (логином может являться имя-фамилия или логин почты) и разрешаться в IP создаваемой VM.
</p>
<p>В провайдеры добавлена возможность работать с ресурсами AWS</p>

<details>
<summary>Содержимое файла <b>main.tf</b></summary>

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

<br><p>В локальные ресурсы добавлено получение требуемого имени для формирования наименования виртуальной машины. А также в локальных ресурсах мы получаем список с публичными адресами созданных виртуальных машин. Это позволило получить более читаемый файл с ресурсами</p>

<details>
<summary>Содержимое файла <b>locals.tf</b></summary>

```yml
locals {
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
  
  do_vps_public_ipv4 = digitalocean_droplet.srv.ipv4_address
  do_vps_user_name = split("_", split(":", var.tag_user_email)[1])[0]
}

```

</details>

<br><p>В ресурсах мы создаем кроме вирутальных машин в Digital Ocean, еще и записи в DNS-сервисе AWS Route53</p>

<details>
<summary>Содержимое файла <b>resouces.tf</b></summary>

```yml
resource "digitalocean_droplet" "srv" {
  image  = var.do_vm_img
  name   = var.do_vm_name
  region = local.do_regions.frankfurt
  size   = local.do_vm_sizes.nano
  tags   = [var.tag_cources, var.tag_task, var.tag_user_email]
  ssh_keys = [data.digitalocean_ssh_key.shared.id, digitalocean_ssh_key.user.id]
}

data "digitalocean_ssh_key" "shared" {
  name = var.ssh_pub_key_shared
}

resource "digitalocean_ssh_key" "user" {
  name       = "Terraform user key"
  public_key = file(var.ssh_pub_key_user)
}

data "aws_route53_zone" "corp" {
  name = var.aws_corp_zone
}

resource "aws_route53_record" "srv_rec" {
  zone_id = data.aws_route53_zone.corp.zone_id
  name    = local.do_vps_user_name
  type    = "A"
  ttl     = 300
  records = [local.do_vps_public_ipv4]
}

```

</details>

<br><p>Здесь мы убеждаемся что возвращаемые значения именно те, что необходимы нам.</p>

<details>
<summary>Содержимое файла <b>outputs.tf</b></summary>

```yml
output "ipv4_public" {
  description = "DigitalOcean output IPv4 public address"
  value       = local.do_vps_public_ipv4
}

output "aws_corp_zone_id" {
  description = "value"
  value = aws_route53_record.srv_rec.zone_id
}

output "aws_srv_zone_name" {
  description = "value"
  value = aws_route53_record.srv_rec.name
}

```

</details>

<br><p>Описываем необходимые для создания ресурсов переменные</p>

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

variable "ssh_pub_key_user" {
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

variable "aws_corp_zone" {
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

<br><p>Ну и получаем значения этих переменных в файле.</p>

<details>
<summary>Содержимое файла <b>terraform.tfvars.sample</b></summary>

```yml
tag_cources = "cources:your_cources"
tag_task = "task_name:your_task"
tag_user_email = "user_email:name_at_1_com"

ssh_pub_key_shared = "SHARED_SSH_KEY"
ssh_pub_key_user = "PATH_to_YOUR_SSH_KEY"

do_token = "your_token"
do_vm_name = "do_vm-name"
do_vm_img = "ubuntu-20-04-x64"

aws_access_key = "your_aws_access_key"
aws_secret_key = "your_aws_secret_key"
aws_corp_zone = "example.net"

```

</details>

Навигация
---

* [Вернуться в основное меню](../../README.md)
* [Вернуться в меню обучения по DevOps](../README.md)
* [Вернуться в меню обучения использования Terraform](./README.md)

Выполнение задания
===

<p> Некоторое продолжение развития предыдущего задания. Больше целенаправленно с Gitlab мы больше работать не будем, но приветствуется создание проектов с помощью этого провайдера.</p>
<p>В этом задании мы будем создавать ресурсы на платформе облачного провайдера Digital Ocean</p>
<p>Так как провайдеров использовано немного, то было принято решение не выделять провайдеры в отдельный файл и они остаются в файле main.tf</p>

<details>
<summary>Содержимое файла <b>main.tf</b></summary>

```yml
terraform {
  required_providers {
    digitalocean = {
      source = "digitalocean/digitalocean"
      version = "~> 2.0"
    }
  }
}

provider "digitalocean" {
  token = var.do_token
}

```

</details>

<br>Локальные переменные были выделены в отдельный файл. По условия задания это не требовалось, но для большей читаемости кода было применено. Выделены следующие локальные ресурсы:</p>

<details>
<summary>Содержимое файла <b>locals.tf</b></summary>

```yml
locals {
  sizes = {
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

  regions = {
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
}

```

</details>

<br>По условия задачи нам необходимо создать минимальную виртуальную машину и добавить в нее два ssh-ключа: общий (для проверки кураторами) и свой для дальнейшей работы и проверок.

<details>
<summary>Содержимое файла <b>resouces.tf</b></summary>

```yml
resource "digitalocean_droplet" "srv" {
  image  = var.vm_img
  name   = var.vm_name
  region = local.regions.frankfurt
  size   = local.sizes.nano
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

```

</details>

<br>В выводе в консоль было принято решение получить два адреса созданного ресурса: приватный адрес и публичный.

<details>
<summary>Содержимое файла <b>outputs.tf</b></summary>

```yml
output "ipv4_public" {
  description = "DigitalOcean output IPv4 public address"
  value       = digitalocean_droplet.srv.ipv4_address
}

output "ipv4_private" {
  description = "DigitalOcean output IPv4 private address"
  value       = digitalocean_droplet.srv.ipv4_address_private
}

```

</details>

<br>Описываем необходимые переменные в файле:

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

variable "vm_name" {
  type = string
  default = "test"
}

variable "vm_img" {
  type = string
  default = "ubuntu-20-04-x64"
}

```

</details>

<details>
<summary>Содержимое файла <b>terraform.tfvars.sample</b></summary>

```yml
do_token = "your_do_token"
tag_task = "task_name:your_task"
tag_user_email = "user_email:your_email"
tag_cources = "cources:your_cources"
vm_name = "do_vm-name"
vm_img = "ubuntu-20-04-x64"
ssh_pub_key_shared = "SHARED_SSH_KEY"
ssh_pub_key_user = "PATH_to_YOUR_SSH_KEY"

```

</details>

Навигация
---

* [Вернуться в основное меню](../../README.md)
* [Вернуться в меню обучения по DevOps](../README.md)
* [Вернуться в меню обучения использования Terraform](./README.md)

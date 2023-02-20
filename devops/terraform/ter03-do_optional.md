Terraform 03: [Optional] Получение существующих ресурсов косвенными способами
===

<p>В этом задании необходимо доработать проект из предыдущего задания. Цель научиться работаь с API запросами и после получения обрабатывать их с помощью скриптов.</p>
<p>Итого что мы имеем.</p>

1. По условия задания у нас каким-то образом остался ключ только в виде его содержимого, без ID/Name/Fingerprint
2. Разрешена только либо переменная содержащая тело ключа, либо чтение тела из файла
3. Провиженеры никакие не доспускаются. Необходимо воспользоваться только "общими" функциями и провайдерами
4. фильтрацию JSON предлагается делать с помощью консольной утилиты jq

<br><p>В провайдеры добавлен провайдер <b>external</b>, что отражено в файле main.tf</p>

<details>
<summary>Содержимое файла <b>main.tf</b></summary>

```yml
terraform {
  required_providers {
    digitalocean = {
      source = "digitalocean/digitalocean"
      version = "~> 2.0"
    }
    external = {
      source = "hashicorp/external"
      version = "2.2.3"
    }
  }
}

provider "digitalocean" {
  token = var.do_token
}

provider "external" {
  # Configuration options
}

```

</details>

<br><p>Файл локальных ресурсов без изменений</p>

<details>
<summary>Содержимое файла <b>locals.tf</b></summary>

```yml
locals {
  # Увидел такое решение по пренеймингу размеров VPC
  # Использование:
  # size = local.sizes.nano

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

  # Увидел такое решение по пренеймингу регионов
  # Использование:
  # size = local.regions.frankfurt

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

<br><p>Как видно из решения, нам в задании показали разницу ресурсов DATA и RESOURCE. Пришлось много разбираться с API и запросными линками.</p>
<p>В частности, пришлось искать и удалять ключ в ресурсах провайдера, после сбоя создания ресурсов.</p>

<details>
<summary>Содержимое файла <b>resouces.tf</b></summary>

```yml
resource "digitalocean_droplet" "srv" {
  image  = var.vm_img
  name   = var.vm_name
  region = local.regions.frankfurt
  size   = local.sizes.nano
  tags   = [var.tag_cources, var.tag_task, var.tag_user_email]
  ssh_keys = [data.external.export_shared_key.result.id, digitalocean_ssh_key.user.id]
}

data "external" "export_shared_key" {
  program = ["bash", "${path.root}/scripts/script.sh"]

  query = {
    token = var.do_token
    pub_key = var.ssh_pub_key_shared
  }
}

resource "digitalocean_ssh_key" "user" {
  name       = "Terraform user key"
  public_key = file(var.ssh_pub_key_user)

```

</details>

<br><p>Ну и главный пожалуй ресурс этого задания, скрипт, который получает по API-запросу все ключи из ресурсов провайдера. Затем фильтрует до искомого ключа.</p>
<p>Потом, разбирает полученный массив данных и отправляет обратно строковые значения</p>

<details>
<summary>Содержимое файла <b>scripts/script.sh</b></summary>

```bash
#!/bin/bash
# это промежуточный вариант, до полного понимания как это должно работать
# token=`grep do_token ./terraform.tfvars | cut -d ' ' -f 3 | cut -d '"' -f 2`
# pub_key=`grep pub_key ./terraform.tfvars | cut -d ' ' -f 3,4 | cut -d '"' -f 2`

eval "$(jq -r '@sh "token=\(.token) pub_key=\(.pub_key)"')"

curl_result=$(curl --silent -X GET -H "Content-Type: application/json" -H "Authorization: Bearer $token" "https://api.digitalocean.com/v2/account/keys?per_page=200" | jq -r '.ssh_keys[] | select(.public_key=="'"$pub_key"'")')

id=`echo $curl_result | jq -r '.id'`
name=`echo $curl_result | jq -r '.name'`
fingerprint=`echo $curl_result | jq -r '.fingerprint'`

echo `jq -n --arg id "$id" \
            --arg name "$name" \
            --arg fingerprint "$fingerprint" \
            '{"id":$id, "name":$name, "fingerprint":$fingerprint}'`

```

> Ответ от куратора на вопрос про возврат значений
> "... TF здесь подсказывает, что ожидает получить в JSON только строковые значения, так как идентификатор число и хочется вернуть его как число, но это ограничение именно провайдера."

</details>

<br><p>Здесь все понятно, фактически мы хотим убедиться что возвращаемые значения именно те что необходимы нам.</p>

<details>
<summary>Содержимое файла <b>outputs.tf</b></summary>

```yml
output "shared_ssh-key_id" {
  description = "Export ID Shared key from external provider"
  value       = data.external.export_shared_key.result.id
}

output "shared_ssh-key_name" {
  description = "Export NAME Shared key from external provider"
  value       = data.external.export_shared_key.result.name
}

output "shared_ssh-key_fingerprint" {
  description = "Export FINGERPRINT Shared key from external provider"
  value       = data.external.export_shared_key.result.fingerprint
}

output "ipv4_public" {
  description = "DigitalOcean output IPv4 public address"
  value       = digitalocean_droplet.srv.ipv4_address
}

```

</details>

<br><p>Пременные тоже без изменений</p>

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

<br><p>Ну и файл с примерами заполнений переменных для проекта.</p>

<details>
<summary>Содержимое файла <b>terraform.tfvars.sample</b></summary>

```yml
do_token = "your_token"
tag_task = "task_name:your_task"
tag_user_email = "user_email:your_email"
tag_cources = "cources:your_cources"
vm_name = "do_vm-name"
vm_img = "ubuntu-20-04-x64"
ssh_pub_key_shared = "SHARED_BODY_SSH_KEY"
ssh_pub_key_user = "PATH_to_YOUR_SSH_KEY"

```

</details>

Навигация
---

* [Вернуться в основное меню](../../README.md)
* [Вернуться в меню обучения по DevOps](../README.md)
* [Вернуться в меню обучения использования Terraform](./README.md)

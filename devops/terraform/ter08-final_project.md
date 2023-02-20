Terraform 08: Итоговая работа по terraform
===

<p>В итоговом проекте мы объединим все изученные методы и подготовим готовую конфигурацию.</p>

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

<br><p></p>

<details>
<summary>Содержимое файла <b>locals.tf</b> не изменилось</summary>

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

  do_vps_user_name = split("_", split(":", var.tag_task[2])[1])[0]
  do_public_ssh_key_path = join("", [var.ssh_private_key_user_path, ".pub"])
  do_vm_names = digitalocean_droplet.srv[*].name
  do_vm_ip = digitalocean_droplet.srv[*].ipv4_address
}

```

</details>

<br><p>Все изменения как обычно в этом файле. Самая непонятная часть была в части использования шаблонов и создание файлов на их основе.</p>
<p>Но, как оказалось там все просто. Нужно было только почитать и попробовать различные примеры.</p>

<details>
<summary>Содержимое файла <b>resouces.tf</b></summary>

```yml
# Create a VPS into Digital Ocean
resource "digitalocean_droplet" "srv" {
  count    = length(var.devs)
  image    = var.do_vm_img
  name     = join("-", [var.devs[count.index], local.do_vps_user_name])
  region   = local.do_regions.frankfurt
  size     = local.do_vm_sizes.nano
  tags     = var.tag_task
  ssh_keys = [data.digitalocean_ssh_key.shared.id, digitalocean_ssh_key.user.id]

  # connection to the created VPS
  connection {
      host        = self.ipv4_address
      type        = "ssh"
      user        = "root"
      private_key = file(var.ssh_private_key_user_path)
    }

  # Change root password and enable password authentication
  provisioner "remote-exec" {
    inline = [
      "sleep 10s",
      "echo root:'${random_string.password[count.index].result}' | chpasswd",
      "sed -i '/^PasswordAuthentication/c PasswordAuthentication yes' /etc/ssh/sshd_config",
      "cat /etc/ssh/sshd_config | grep -i ^PasswordAuthentication",
      "/usr/bin/systemctl restart sshd.service"
    ]
  }
}

# Generate random password string
resource "random_string" "password" {
  count   = length(var.devs)
  length  = 16
  special = true
  # override_special = "~!@%*"
}

# Get SSH key (REBRAIN) from variables
data "digitalocean_ssh_key" "shared" {
  name = var.ssh_pub_key_shared
}

# Get SSH key (user) from local variables
resource "digitalocean_ssh_key" "user" {
  name       = "Terraform user key"
  public_key = file(local.do_public_ssh_key_path)
}

# get value from variables
data "aws_route53_zone" "aws_zone" {
  name = var.aws_aws_zone
}

# Create records Route53
resource "aws_route53_record" "srv_rec" {
  count   = length(var.devs)
  zone_id = data.aws_route53_zone.aws_zone.zone_id
  name    = local.do_vm_names[count.index]
  type    = "A"
  ttl     = 300
  records = [local.do_vm_ip[count.index]]
}

# Upload the required data to a file
resource "local_file" "out" {
  filename = "${path.module}/inventory.txt"
  # используется для повторения создания этого ресурса
  # если раскомментирвоать и добавить индекс в имя файла, то будет создано столько файлов, сколько насчитает индекс
  # count = length(var.devs)
  content = templatefile("${path.module}/templates/inventory.txt.tftpl", {
    names = local.do_vm_names,
    domain = var.aws_zone,
    public_ip = local.do_vm_ip,
    root_password = random_string.password[*].result
  })
}

```

</details>

<br><p>Это содержимое шаблона, которое заполняется из ресурсов и создается файл в указанном месте.</p>

<details>
<summary>Содержимое файла <b>templates/inventory.txt.tftpl</b></summary>

```yml
%{ for index, name in names ~}
${index+1} : ${name}.${domain} : ${public_ip[index]} : root : ${root_password[index]}
%{ endfor ~}

```

</details>

<br><p></p>

<details>
<summary>Содержимое файла <b>outputs.tf</b></summary>

```yml
# В консоль мы по условию ничего не выводим
```

</details>

<br><p>В переменных по условию были введены определенные лстовые занчения. В частности теги и назначение создаваемых виртуальных машин.</p>

<details>
<summary>Содержимое файла <b>variables.tf</b></summary>

```yml
# Эта часть переменных изменяется здесь и в переменных не определяется
variable "tag_task" {
  type = list
  default = ["cources:devops", "task_name:terraform-final", "user_email:user_at_yandex_ru"]
}

variable "devs" {
  type    = list
  default = ["lb", "app1", "app2"]
}

# Эта часть переменных определяется в terraform.tfvars
variable "do_token" {
  type = string
  sensitive = true
}

variable "do_vm_img" {
  type = string
  default = "ubuntu-20-04-x64"
}

variable "ssh_pub_key_shared" {
  type = string
  sensitive = true
}

variable "ssh_private_key_user_path" {
  type = string
  sensitive = true
}

variable "aws_rebrain_zone" {
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
ssh_pub_key_shared = "SHARED_SSH_KEY"
ssh_private_key_user_path = "PATH_to_YOUR_PRIVATE_SSH_KEY"

do_token = "your_token"
do_vm_img = "ubuntu-20-04-x64"

aws_access_key = "your_aws_access_key"
aws_secret_key = "your_aws_secret_key"
aws_rebrain_zone = "example.net"

```

</details><br><br>

Навигация
---

* [Вернуться в основное меню](../../README.md)
* [Вернуться в меню обучения по DevOps](../README.md)
* [Вернуться в меню обучения использования Terraform](./README.md)

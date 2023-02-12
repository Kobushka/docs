Terraform 01: Знакомство с Terraform
===

<p>Знакомиться с инструментом будем на уже знакомом материале - использование платформы <b>Gitlab</b>.<br>В качестве задания необходимо создать репозиторий на платформе в своей группе.

<details>
<summary>Содержимое файла <b>main.tf</b></summary>

```yml
terraform {
  required_providers {
    gitlab = {
      source = "gitlabhq/gitlab"
    }
  }
  required_version = ">= 0.13"
}

provider "gitlab" {
    token = var.gitlab_token
    base_url = var.gitlab_url
}

```

</details>

<details>
<summary>Содержимое файла <b>resources.tf</b></summary>

```yml
resource "gitlab_project" "ter01" {
  name         = var.repository_name
  description  = var.repository_description
  namespace_id = data.gitlab_group.user_devops.id
}

resource "gitlab_deploy_key" "ter01_deploy_key" {
  project = gitlab_project.ter01.id
  title   = var.repository_deploy_sshkey_title
  key     = var.repository_deploy_sshkey
  can_push = true
}

data "gitlab_group" "user_devops" {
  full_path = join("/",[var.gitlab_cource_group, var.gitlab_user_group])
}

```

</details>

<details>
<summary>Содержимое файла <b>outputs.tf</b></summary>

```yml
output "project_id" {
  description = "ID repository"
  value       = gitlab_project.ter01.id
}

```

</details>

<details>
<summary>Содержимое файла <b>variables.tf</b></summary>

```yml
variable "gitlab_token" {
  type = string
  default = ""
  sensitive = true
}

variable "gitlab_url" {
  type = string
  default = ""
}

variable "repository_name" {
  type = string
  default = ""
}

variable "repository_description" {
  type = string
  default = "Test repo for TERRAFORM"
}

variable "repository_deploy_sshkey" {
  type = string
  default = ""
  sensitive = true
}

variable "repository_deploy_sshkey_title" {
  type = string
  default = ""  
}

variable "gitlab_user_group" {
  type = string
}

```

</details>

<details>
<summary>Содержимое файла <b>terraform.tfvars.sample</b></summary>

```yml
gitlab_token = "****YOUR_TOKEN****"
gitlab_url = "https://gitlab.url/"
repository_name = "your-repository-name"
repository_description = "YOUR commentary"
repository_deploy_sshkey = "ssh-rsa AAAA..."
repository_deploy_sshkey_title = "YOUR SSH key title"
gitlab_cource_group = "YOUR cources group name"
gitlab_user_group = "YOUR user group name"

```

</details><br>

<p><b>Запись некоторых моментов, чтобы не забылось.</b>

<details>
<summary>Terraform и санкционное давление</summary>

<p>Так как компания HashiCorp участвует в давлении на нашу страну, то пришлось искать варианты обхода данных ограничений.
<p>Решений нашлось далеко не одно:

* Во-первых - это и использование VPN, как платных, так и бесплатных.
* Во-вторых - использование проксирования запросов на скачивание провайдеров из ресурсов Terraform

<p>Второй вариант (на сегодняшний день пока не закрыт) и проще и быстрее, если конечно, нет уже купленного подключения VPN.
<p>Заключается в настройке конфигурационного файла.

```bash
# Windows
notepad $env:APPDATA/terraform.rc

# Linux
nano ~/.terraformrc
```

В него надо внести следующее содержимое:

```bash
provider_installation {
  network_mirror {
    url = "https://terraform-mirror.yandexcloud.net/"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
}
```

Если по каким-то причинам не нравиться Яндекс, то можно использовать любой другой доступный альтернативный репозиторий. Никакие перезагрузки не нужны. Все начинает работать сразу после внесения изменений.

</details>
<details>
<summary>Замечание от куратора</summary>

Группа пользователя специально была задана цифрами, чтобы потом указать на это и отправить на доработку.

> Идентификатор пространства имен берете из переменной, но с цифровыми идентификаторами часто случается путаница (в том числе и в этом задании), поэтому давайте получим его с помощью источника данных data провайдера гитлаб на основе имени/пути группы (не забудьте параметризовать через переменную)

```yml
# Добавлена возможность получения идентификатора из API провайдера
resource "gitlab_project" "ter01" {
   ...
   namespace_id = data.gitlab_group.user_devops.id
}

# Вот здесь необходимо добавить для возможности пушить потом с этим ключом
resource "gitlab_deploy_key" "ter01_deploy_key" {
  ...
  can_push = true
}

# получение группы пользователя как ресурсов из провайдера
data "gitlab_group" "user_devops" {
  # оказалось я отстал немного и то что ранее делалось подстановками значений через "${}"
  # в новых версиях осталось (и настоятельно рекомендуется) только как подстановка
  # функции теперь используются напрямую для улучшения читаемости кода
  full_path = join("/",[var.gitlab_cource_group, var.gitlab_user_group])
}
```

После доработки выполнение задание было принято на <b>отлично</b>

</details><br>

Навигация
---

* [Вернуться в основное меню](../../README.md)
* [Вернуться в меню обучения по DevOps](../README.md)
* [Вернуться в меню обучения использования Terraform](./README.md)

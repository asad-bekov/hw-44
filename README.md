# Домашнее задание «Вычислительные мощности. Балансировщики нагрузки"

> Репозиторий: hw-44\
> Выполнил: Асадбек Асадбеков\
> Дата: октябрь 2025

## Выполнение задания в Yandex Cloud с использованием Terraform

---

## Задание 1. Yandex Cloud

### 1.1. Создание бакета Object Storage и загрузка файла

**Цель:**

* Создать бакет в Object Storage
* Разместить файл с картинкой
* Сделать файл доступным из интернета

<details>
<summary>Посмотреть provider.tf</summary>

```hcl
terraform {
  required_providers {
    yandex = {
      source  = "yandex-cloud/yandex"
      version = "~> 0.127"
    }
  }
  required_version = ">= 1.3.0"
}

provider "yandex" {
#  token     = var.yc_token
  cloud_id  = var.cloud_id
  folder_id = var.folder_id
  zone      = var.zone
}
```

</details>

<details>
<summary>Посмотреть variables.tf</summary>

```hcl
# variable "yc_token" {
#  description = "Yandex Cloud OAuth or IAM token"
#  type        = string
# }

variable "cloud_id" {
  description = "Yandex Cloud cloud-id"
  type        = string
}

variable "folder_id" {
  description = "Yandex Cloud folder-id"
  type        = string
}

variable "yc_folder_id" {
  description = "Yandex Cloud folder-id (alias)"
  type        = string
  default     = ""
}

variable "zone" {
  description = "Compute default zone"
  type        = string
}
```
</details>

<details>
<summary>Посмотреть terraform.tfvars</summary>

```hcl
# yc_token  = "Удален из-за безопасности"
cloud_id  = "b1gsj7sfde79kl5qkpbl"
folder_id = "b1gm0hnoge59gnkmh3dl"
zone      = "ru-central1-a"
```
</details>

<details>
<summary>Посмотреть main.tf</summary>

```hcl
resource "yandex_vpc_network" "lamp_network" {
  name = "lamp-network"
}

resource "yandex_vpc_subnet" "lamp_subnet" {
  name           = "lamp-subnet"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.lamp_network.id
  v4_cidr_blocks = ["192.168.100.0/24"]
}

data "yandex_iam_service_account" "terraform_sa" {
  name = "terraform-sa"
}

resource "yandex_compute_instance_group" "lamp_group" {
  name                = "lamp-group"
  folder_id           = var.folder_id
  service_account_id  = data.yandex_iam_service_account.terraform_sa.id
  
  instance_template {
    platform_id = "standard-v3"
    resources {
      cores  = 2
      memory = 2
    }

    boot_disk {
      initialize_params {
        image_id = "fd827b91d99psvq5fjit"
        size     = 10
      }
    }

    network_interface {
      subnet_ids = [yandex_vpc_subnet.lamp_subnet.id]  # используем новую подсеть
      nat       = true
    }

    metadata = {
      user-data = <<-EOF
        #!/bin/bash
        apt-get update
        apt-get install -y apache2
        echo "<html><body><h1>Welcome to Netology LAMP Group</h1><p><img src='https://storage.yandexcloud.net/netology-asad-20251027/netology-image.jpg' width='500'></p></body></html>" > /var/www/html/index.html
        systemctl enable apache2
        systemctl start apache2
      EOF
    }
  }

  scale_policy {
    fixed_scale {
      size = 0
    }
  }

  allocation_policy {
    zones = ["ru-central1-a"]
  }

  deploy_policy {
    max_unavailable = 1
    max_creating    = 2
    max_expansion   = 1
    max_deleting    = 1
  }

  health_check {
    interval = 10
    timeout  = 5
    unhealthy_threshold = 3
    healthy_threshold   = 2
    tcp_options {
      port = 80
    }
  }

  load_balancer {
    target_group_name = "lamp-target-group"
  }
}

resource "yandex_lb_network_load_balancer" "lamp_nlb" {
  name = "lamp-network-lb"

  listener {
    name = "lamp-listener"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_compute_instance_group.lamp_group.load_balancer.0.target_group_id

    healthcheck {
      name = "http-healthcheck"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}

output "load_balancer_ip" {
  value = yandex_lb_network_load_balancer.lamp_nlb.listener[*].external_address_spec[*].address
}

output "instance_group_info" {
  value = {
    id      = yandex_compute_instance_group.lamp_group.id
    status  = yandex_compute_instance_group.lamp_group.status
  }
}

resource "yandex_alb_load_balancer" "lamp_alb" {
  name        = "lamp-app-lb"
  network_id  = yandex_vpc_network.lamp_network.id

  allocation_policy {
    location {
      zone_id   = "ru-central1-a"
      subnet_id = yandex_vpc_subnet.lamp_subnet.id
    }
  }

  listener {
    name = "lamp-http-listener"
    endpoint {
      address {
        external_ipv4_address {
        }
      }
      ports = [80]
    }
    http {
      handler {
        http_router_id = yandex_alb_http_router.lamp_router.id
      }
    }
  }
}

resource "yandex_alb_http_router" "lamp_router" {
  name = "lamp-router"
}

resource "yandex_alb_virtual_host" "lamp_host" {
  name           = "lamp-host"
  http_router_id = yandex_alb_http_router.lamp_router.id
  
  route {
    name = "lamp-route"
    http_route {
      http_route_action {
        backend_group_id = yandex_alb_backend_group.lamp_backend_group.id
      }
    }
  }
}

resource "yandex_alb_target_group" "lamp_target_group" {
  name = "lamp-alb-target-group"

  target {
    subnet_id  = yandex_vpc_subnet.lamp_subnet.id
    ip_address = yandex_compute_instance_group.lamp_group.instances[0].network_interface[0].ip_address
  }
  target {
    subnet_id  = yandex_vpc_subnet.lamp_subnet.id
    ip_address = yandex_compute_instance_group.lamp_group.instances[1].network_interface[0].ip_address
  }
  target {
    subnet_id  = yandex_vpc_subnet.lamp_subnet.id
    ip_address = yandex_compute_instance_group.lamp_group.instances[2].network_interface[0].ip_address
  }
}

resource "yandex_alb_backend_group" "lamp_backend_group" {
  name = "lamp-backend-group"

  http_backend {
    name             = "lamp-http-backend"
    weight           = 1
    port             = 80
    target_group_ids = [yandex_alb_target_group.lamp_target_group.id]
    
    healthcheck {
      timeout          = "10s"
      interval         = "2s"
      healthcheck_port = 80
      http_healthcheck {
        path = "/"
      }
    }
  }
}

output "alb_ip" {
  value = yandex_alb_load_balancer.lamp_alb.listener[*].endpoint[*].address[*].external_ipv4_address[*].address
}
```
</details>

### Скриншот 1 — Создание bucket

**Команда:**  
```bash
terraform apply -auto-approve
```

**Описание:** Создание bucket в Object Storage.  

![Создание bucket](https://github.com/asad-bekov/hw-44/blob/main/img/1.png)

---

### Скриншот 2 — Подтверждение создания bucket

**Команда:**  
```bash
terraform apply -auto-approve
```

**Описание:** Подтверждение создания ресурса `yandex_storage_bucket.image_bucket` с публичным доступом и настройками версионирования.  

![Подтверждение создания bucket](https://github.com/asad-bekov/hw-44/blob/main/img/2.png)

---

### Скриншот 3 — Загрузка файла в bucket

**Команда:**  
```bash
yc storage s3 cp ./netology-image.jpg s3://netology-asad-20251027/netology-image.jpg && yc storage s3api put-object-acl   --bucket netology-asad-20251027   --key netology-image.jpg   --acl public-read
```

**Описание:** Загрузка файла `netology-image.jpg` в созданный bucket и установка публичного доступа к объекту.  

![Загрузка файла в bucket](https://github.com/asad-bekov/hw-44/blob/main/img/3.png)

---

### Скриншот 4 — Проверка публичного доступа к файлу

**Команда:** Открытие URL в браузере.  
**Описание:** Проверка публичного доступа к загруженному изображению — отображение картинки с логотипами DevOps-инструментов по прямой ссылке в Object Storage.  

![Проверка публичного доступа к файлу](https://github.com/asad-bekov/hw-44/blob/main/img/4.png)

---

### 1.2. Создание Instance Group с LAMP

**Цель:**

* Создать группу из 3 ВМ
* Установить LAMP stack
* Разместить веб-страницу с картинкой из bucket
* Настроить health checks


### Скриншот 5 — Создание Instance Group

**Команда:**  
```bash
terraform apply -auto-approve
```

**Описание:** Создание Instance Group (группы виртуальных машин) с именем `lamp_group`. В процессе развертывания создается несколько экземпляров VM, что видно по прогрессу создания и финальному сообщению об успешном завершении.  

![Создание Instance Group](https://github.com/asad-bekov/hw-44/blob/main/img/5.png)

---

### Скриншот 6 — Проверка Instance Group в YC

**Команда:** Просмотр в веб-консоли YC — раздел *Compute Cloud → Виртуальные машины*.  
**Описание:** Проверка созданных виртуальных машин в группе. На скриншоте видны три запущенных экземпляра (в статусе *Running*) с ОС LAMP, их IP-адреса и зона доступности `ru-central1-a`.  

![Проверка Instance Group в YC](https://github.com/asad-bekov/hw-44/blob/main/img/6.png)

---

### Скриншот 7 — Проверка веб-страницы на Instance

**Команда:** Открытие публичного IP-адреса в браузере (например, `http://158.160.44.66`).  
**Описание:** Проверка работы веб-страницы, развернутой на одной из ВМ группы. Страница содержит заголовок **"Welcome to Netology LAMP Group"** и изображение, загруженное ранее в Object Storage.  

![Проверка веб-страницы на Instance](https://github.com/asad-bekov/hw-44/blob/main/img/7.png)

---

### 1.3. Подключение Instance Group к Network Load Balancer

### Скриншот 8 — Настройка сетевого балансировщика

**Команда:**  
```bash
terraform apply -auto-approve
```

**Описание:** Создание Network Load Balancer (`lamp_nlb`) для распределения трафика между экземплярами Instance Group. После успешного создания Terraform выводит ID группы и публичный IP балансировщика — например, `84.201.149.106`.  

![Настройка сетевого балансировщика](https://github.com/asad-bekov/hw-44/blob/main/img/8.png)

---

### Скриншот 9 — Проверка работы Load Balancer

**Команда:** Открытие публичного IP-адреса балансировщика (например, `http://84.201.149.106`).  
**Описание:** Проверка работы Network Load Balancer — страница **"Welcome to Netology LAMP Group"** отображается успешно, что подтверждает корректное распределение трафика между ВМ.  

![Проверка работы Load Balancer](https://github.com/asad-bekov/hw-44/blob/main/img/9.png)

---

### Скриншот 10 — Проверка VM после настройки балансировщика

**Команда:** Просмотр в веб-консоли YC — *Compute Cloud → Виртуальные машины*.  
**Описание:** Повторная проверка состояния виртуальных машин в группе. Все экземпляры находятся в статусе *Running*, что подтверждает их готовность к работе и подключение к балансировщику.  

![Проверка VM после настройки балансировщика](https://github.com/asad-bekov/hw-44/blob/main/img/10.png)

---

### Скриншот 11 — Проверка отказоустойчивости через CLI

**Команды:**  
```bash
yc compute instance-group list-instances lamp-group
yc compute instance delete fhmg7kja8hll19l03mpd
curl http://84.201.149.106
```

**Описание:** Проверка отказоустойчивости балансировщика. Один из экземпляров удаляется вручную, после чего `curl` подтверждает, что страница продолжает отвечать — трафик перераспределяется между оставшимися ВМ.  

![Проверка отказоустойчивости через CLI](https://github.com/asad-bekov/hw-44/blob/main/img/11.png)

---

### Скриншот 12 — Проверка состояния Load Balancer в консоли

**Команда:** Просмотр в YC — *Network Load Balancer → lamp-network-lb → Обзор*.  
**Описание:** Проверка состояния балансировщика и целевых групп. После удаления одной ВМ балансировщик корректно обновил состояние — 2 экземпляра остаются *Healthy*.  

![Проверка состояния Load Balancer в консоли](https://github.com/asad-bekov/hw-44/blob/main/img/12.png)

---

### 1.4. Создание Application Load Balancer

**Команда:**  
```bash
terraform apply -auto-approve
```

**Описание:** Создание Application Load Balancer (`lamp_host`) с виртуальным хостом. Terraform развертывает ALB, привязывает его к целевой группе и выводит публичный IP (например, `158.160.170.14`).  

![Создание Application Load Balancer](https://github.com/asad-bekov/hw-44/blob/main/img/13.png)

---

### Скриншот 14 — Общий обзор инфраструктуры в YC

**Команда:** Просмотр в YC — *Marketplace → Сервисы каталога*.  
**Описание:** На скриншоте видны все созданные компоненты: 3 ВМ в Compute Cloud, 1 bucket в Object Storage, 1 Network Load Balancer и 1 Application Load Balancer. Также отображается информация о целевых группах (в выпадающем меню), что подтверждает корректное развертывание всей инфраструктуры.  

![Общий обзор инфраструктуры в консоли YC](https://github.com/asad-bekov/hw-44/blob/main/img/14.png)

---

### Скриншот 15 — Проверка NLB

**Команда:** Открытие IP-адреса NLB (`http://84.201.149.106`).  
**Описание:** Проверка работы сайта через Network Load Balancer (L4). Страница успешно загружается, балансировщик корректно распределяет TCP/UDP-трафик.  

![Проверка NLB](https://github.com/asad-bekov/hw-44/blob/main/img/15.1.png)

---

### Скриншот 16 — Проверка ALB

**Команда:** Открытие IP-адреса ALB (`http://158.160.170.14`).  
**Описание:** Проверка работы сайта через Application Load Balancer (L7). Страница отображается корректно — ALB обрабатывает HTTP-запросы и маршрутизирует трафик по приложениям.  

![Проверка ALB](https://github.com/asad-bekov/hw-44/blob/main/img/15.2.png)

**Итог:**

* Все ресурсы созданы успешно
* Instance Group работает, LB корректно распределяет нагрузку
* Настроены health checks и отказоустойчивость
* Веб-страница доступна через оба балансировщика

---

**Все задачи выполнены успешно с использованием Terraform!**


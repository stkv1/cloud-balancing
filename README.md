# Домашнее задание к занятию «Отказоустойчивость в облаке»
# Козлов Станислав

## Задание 1
Возьмите за основу решение к заданию 1 из занятия «Подъём инфраструктуры в Яндекс Облаке».

Теперь вместо одной виртуальной машины сделайте terraform playbook, который:
создаст 2 идентичные виртуальные машины. Используйте аргумент count для создания таких ресурсов;
создаст таргет-группу. Поместите в неё созданные на шаге 1 виртуальные машины;
создаст сетевой балансировщик нагрузки, который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.
Рекомендуем изучить документацию сетевого балансировщика нагрузки для того, чтобы было понятно, что вы сделали.

Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.

Перейдите в веб-консоль Yandex Cloud и убедитесь, что:

созданный балансировщик находится в статусе Active,
обе виртуальные машины в целевой группе находятся в состоянии healthy.
Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.
В качестве результата пришлите:

1. Terraform Playbook.

[Uploading main.tf…]()terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"
}

#set variable string for token
variable "yandex_cloud_token" {
  type = string
  description = "Введите секретный токен от yandex_cloud"
}

provider "yandex" {
  token                    = var.yandex_cloud_token
  cloud_id                 = "b1gb2jo6jnscm01lbu8i"
  folder_id                = "b1gbs69rm17l1ks91fee"
  zone                     = "ru-cetral1-b"
}

resource "yandex_compute_instance" "vm" {

  count                     = 2
  name                      = "balancing-vm${count.index}"
  allow_stopping_for_update = true
  platform_id               = "standard-v1"
  zone                      = "ru-central1-b"

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      # debian10
      image_id = "fd8n0gp0i9d7u9a5ejjk"
    }
  }

  network_interface {
    subnet_id = "${yandex_vpc_subnet.subnet-1.id}"
    nat       = true
  }
  
  metadata = {
    user-data = "${file("./meta.yml")}"
  }
}

resource "yandex_lb_target_group" "target-1" {
  name      = "target-1"
  target {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    address   = yandex_compute_instance.vm[0].network_interface.0.ip_address
  }
  target {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    address   = yandex_compute_instance.vm[1].network_interface.0.ip_address
  }
}

resource "yandex_lb_network_load_balancer" "lb-1" {
  name = "lb-1"
  listener {
    name = "my-list"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }
  attached_target_group {
    target_group_id = yandex_lb_target_group.target-1.id
    healthcheck {
      name = "http"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}

resource "yandex_vpc_network" "network-1" {
  name = "network1"
}
  
resource "yandex_vpc_subnet" "subnet-1" {
  name           = "subnet1"
  zone           = "ru-central1-b"
  v4_cidr_blocks = ["192.168.10.0/24"]
  network_id     = "${yandex_vpc_network.network-1.id}"
}


2. Скриншот статуса балансировщика и целевой группы.

  ![image](https://github.com/stkv1/cloud-balancing/assets/145263196/01a73f33-ed30-401d-9408-0879c4533fa7)


3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.

   ![03](https://github.com/stkv1/cloud-balancing/assets/145263196/3cb56cef-0aab-4370-8954-c548650785bc)


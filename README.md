
# Домашнее задание к занятию «Основы Terraform. Yandex Cloud» - Храмов Александр


### Задание 0

1. Ознакомьтесь с [документацией к security-groups в Yandex Cloud](https://cloud.yandex.ru/docs/vpc/concepts/security-groups?from=int-console-help-center-or-nav). 
Этот функционал понадобится к следующей лекции.



### Решение 0



------

### Задание 1
В качестве ответа всегда полностью прикладывайте ваш terraform-код в git.
Убедитесь что ваша версия **Terraform** ~>1.8.4

1. Изучите проект. В файле variables.tf объявлены переменные для Yandex provider.
2. Создайте сервисный аккаунт и ключ. [service_account_key_file](https://terraform-provider.yandexcloud.net).
4. Сгенерируйте новый или используйте свой текущий ssh-ключ. Запишите его открытую(public) часть в переменную **vms_ssh_public_root_key**.
5. Инициализируйте проект, выполните код. Исправьте намеренно допущенные синтаксические ошибки. Ищите внимательно, посимвольно. Ответьте, в чём заключается их суть.
6. Подключитесь к консоли ВМ через ssh и выполните команду ``` curl ifconfig.me```.
Примечание: К OS ubuntu "out of a box, те из коробки" необходимо подключаться под пользователем ubuntu: ```"ssh ubuntu@vm_ip_address"```. Предварительно убедитесь, что ваш ключ добавлен в ssh-агент: ```eval $(ssh-agent) && ssh-add``` Вы познакомитесь с тем как при создании ВМ создать своего пользователя в блоке metadata в следующей лекции.;
8. Ответьте, как в процессе обучения могут пригодиться параметры ```preemptible = true``` и ```core_fraction=5``` в параметрах ВМ.

В качестве решения приложите:

- скриншот ЛК Yandex Cloud с созданной ВМ, где видно внешний ip-адрес;
- скриншот консоли, curl должен отобразить тот же внешний ip-адрес;
- ответы на вопросы.


### Решение 1

![1](/img/4.png)

![1](/img/2.png)

![1](/img/w.png)

Вопросы:

Параметры preemptible = true - ВМ, которая работает не более 24 часов и может быть остановлена Compute Cloud в любой момент. После остановки ВМ не удаляется, все ее данные сохраняются. Чтобы продолжить работу, запустите ВМ повторно. Предоставляется с большой скидкой.


Параметр core_fraction = 5 - указывает базовую производительность ядра в процентах. 





------
### Задание 2

1. Замените все хардкод-**значения** для ресурсов **yandex_compute_image** и **yandex_compute_instance** на **отдельные** переменные. К названиям переменных ВМ добавьте в начало префикс **vm_web_** .  Пример: **vm_web_name**.
2. Объявите нужные переменные в файле variables.tf, обязательно указывайте тип переменной. Заполните их **default** прежними значениями из main.tf. 
3. Проверьте terraform plan. Изменений быть не должно. 


### Решение2

main.tf

```
# Создаем VPC сеть
resource "yandex_vpc_network" "develop" {
  name = var.vpc_name
}

# Создаем подсеть
resource "yandex_vpc_subnet" "develop" {
  name           = var.vpc_name
  zone           = var.default_zone
  network_id     = yandex_vpc_network.develop.id
  v4_cidr_blocks = var.default_cidr
}

# Получаем образ ОС
data "yandex_compute_image" "ubuntu" {
  family = var.vm_web_image_family
}

# Создаем виртуальную машину
resource "yandex_compute_instance" "platform" {
  name = var.vm_web_name

  platform_id = var.vm_web_platform_id

  resources {
    cores         = var.vm_web_cores
    memory        = var.vm_web_memory
    core_fraction = var.vm_web_core_fraction
  }

  boot_disk {
    initialize_params {
      image_id = data.yandex_compute_image.ubuntu.image_id
    }
  }

  scheduling_policy {
    preemptible = var.vm_web_preemptible
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.develop.id
    nat       = var.vm_web_nat
  }

  metadata = var.vms_metadata

  allow_stopping_for_update = true
}


```

variables.tf

```
variable "token" {
  type        = string
  description = "OAuth-token; https://cloud.yandex.ru/docs/iam/concepts/authorization/oauth-token"
}

variable "cloud_id" {
  type        = string
  description = "https://cloud.yandex.ru/docs/resource-manager/operations/cloud/get-id"
}

variable "folder_id" {
  type        = string
  description = "https://cloud.yandex.ru/docs/resource-manager/operations/folder/get-id"
}

variable "default_zone" {
  type        = string
  default     = "ru-central1-a"
  description = "https://cloud.yandex.ru/docs/overview/concepts/geo-scope"
}

variable "default_cidr" {
  type        = list(string)
  default     = ["10.0.1.0/24"]
  description = "Список CIDR блоков для подсети"
}

variable "vpc_name" {
  type        = string
  default     = "develop"
  description = "VPC network & subnet name"
}

variable "vms_ssh_root_key" {
  type        = string
  description = "ssh-keygen -t ed25519"
}

variable "vms_resources" {
  type = map(map(any))
  default = {
    vm_web = {
      core_fraction = 5
      cores         = 2
      memory        = 1
    }
    vm_db = {
      core_fraction = 20
      cores         = 2
      memory        = 2
    }
  }
  description = "Resource configurations for VMs"
}

variable "vms_metadata" {
  type = map(string)
  default = {
    "serial-port-enable" = "1"
    "ssh-keys"           = "ubuntu:ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMzPxyrg408uoTpwNEJwtWKaFxH6EbSbbjBHd7i3NepU khramulka@yandex.ru"
  }
  description = "Metadata for VMs"
}

# Новые переменные для ВМ
variable "vm_web_name" {
  type        = string
  default     = "netology-develop-platform-web"
  description = "Имя виртуальной машины"
}

variable "vm_web_platform_id" {
  type        = string
  default     = "standard-v1"
  description = "Платформа ВМ"
}

variable "vm_web_image_family" {
  type        = string
  default     = "ubuntu-2004-lts"
  description = "Семейство образов ОС"
}

variable "vm_web_cores" {
  type        = number
  default     = 2
  description = "Количество ядер"
}

variable "vm_web_memory" {
  type        = number
  default     = 1
  description = "Количество ОЗУ"
}

variable "vm_web_core_fraction" {
  type        = number
  default     = 5
  description = "Доля процессорного времени"
}

variable "vm_web_preemptible" {
  type        = bool
  default     = false
  description = "Использовать преэмптивную ВМ"
}

variable "vm_web_nat" {
  type        = bool
  default     = true
  description = "Включить NAT"
}

```

![1](/img/2.1.png)


------
### Задание 3

1. Создайте в корне проекта файл 'vms_platform.tf' . Перенесите в него все переменные первой ВМ.
2. Скопируйте блок ресурса и создайте с его помощью вторую ВМ в файле main.tf: **"netology-develop-platform-db"** ,  ```cores  = 2, memory = 2, core_fraction = 20```. Объявите её переменные с префиксом **vm_db_** в том же файле ('vms_platform.tf').  ВМ должна работать в зоне "ru-central1-b"
3. Примените изменения.

   
### Решение 3

vms_platform.tf
```
# Переменные для второй ВМ (db)
variable "vm_db_name" {
  type        = string
  default     = "netology-develop-platform-db"
  description = "Имя виртуальной машины"
}


variable "vm_db_platform_id" {
  type        = string
  default     = "standard-v1"
  description = "Платформа ВМ"
}

variable "vm_db_image_family" {
  type        = string
  default     = "ubuntu-2004-lts"
  description = "Семейство образов ОС"
}

variable "vm_db_cores" {
  type        = number
  default     = 2
  description = "Количество ядер"
}

variable "vm_db_memory" {
  type        = number
  default     = 2
  description = "Количество ОЗУ"
}

variable "vm_db_core_fraction" {
  type        = number
  default     = 20
  description = "Доля процессорного времени"
}

variable "vm_db_preemptible" {
  type        = bool
  default     = false
  description = "Использовать преэмптивную ВМ"
}

variable "vm_db_nat" {
  type        = bool
  default     = true
  description = "Включить NAT"
}

variable "vm_db_zone" {
  type        = string
  default     = "ru-central1-b"
  description = "Зона размещения ВМ"
}

```
![1](/img/3.2.png)

------


### Задание 4

1. Объявите в файле outputs.tf **один** output , содержащий: instance_name, external_ip, fqdn для каждой из ВМ в удобном лично для вас формате.(без хардкода!!!)
2. Примените изменения.

В качестве решения приложите вывод значений ip-адресов команды ```terraform output```.


### Решение 4

outputs.md

```
output "vm_info" {
  value = {
    web = {
      instance_name = yandex_compute_instance.platform_web.name
      external_ip   = yandex_compute_instance.platform_web.network_interface.0.nat_ip_address
      fqdn          = yandex_compute_instance.platform_web.fqdn
    },
    db = {
      instance_name = yandex_compute_instance.platform_db.name
      external_ip   = yandex_compute_instance.platform_db.network_interface.0.nat_ip_address
      fqdn          = yandex_compute_instance.platform_db.fqdn
    }
  }
  description = "Информация о созданных виртуальных машинах"
}

```

![1](/img/4.1.png)
------

### Задание 5

1. В файле locals.tf опишите в **одном** local-блоке имя каждой ВМ, используйте интерполяцию ${..} с НЕСКОЛЬКИМИ переменными по примеру из лекции.
2. Замените переменные внутри ресурса ВМ на созданные вами local-переменные.
3. Примените изменения.


### Решение 5

```
locals {
  vm_names = {
    web = "${var.vpc_name}-${var.vm_web_platform_id}-web"
    db  = "${var.vpc_name}-${var.vm_db_platform_id}-db"
  }
}

```

![1](/img/5.1.png)
------

### Задание 6

1. Вместо использования трёх переменных  ".._cores",".._memory",".._core_fraction" в блоке  resources {...}, объедините их в единую map-переменную **vms_resources** и  внутри неё конфиги обеих ВМ в виде вложенного map(object).  
   ```
   пример из terraform.tfvars:
   vms_resources = {
     web={
       cores=2
       memory=2
       core_fraction=5
       hdd_size=10
       hdd_type="network-hdd"
       ...
     },
     db= {
       cores=2
       memory=4
       core_fraction=20
       hdd_size=10
       hdd_type="network-ssd"
       ...
     }
   }
   ```
3. Создайте и используйте отдельную map(object) переменную для блока metadata, она должна быть общая для всех ваших ВМ.
   ```
   пример из terraform.tfvars:
   metadata = {
     serial-port-enable = 1
     ssh-keys           = "ubuntu:ssh-ed25519 AAAAC..."
   }
   ```  
  
5. Найдите и закоментируйте все, более не используемые переменные проекта.
6. Проверьте terraform plan. Изменений быть не должно.



### Решение 6
------
variables.tf
```
variable "token" {
  type        = string
  description = "OAuth-token; https://cloud.yandex.ru/docs/iam/concepts/authorization/oauth-token"
}

variable "cloud_id" {
  type        = string
  description = "https://cloud.yandex.ru/docs/resource-manager/operations/cloud/get-id"
}

variable "folder_id" {
  type        = string
  description = "https://cloud.yandex.ru/docs/resource-manager/operations/folder/get-id"
}

variable "default_zone" {
  type        = string
  default     = "ru-central1-a"
  description = "https://cloud.yandex.ru/docs/overview/concepts/geo-scope"
}

variable "default_cidr" {
  type        = list(string)
  default     = ["10.0.1.0/24"]
  description = "Список CIDR блоков для подсети"
}

variable "vpc_name" {
  type        = string
  default     = "develop"
  description = "VPC network & subnet name"
}

variable "vms_ssh_root_key" {
  type        = string
  description = "ssh-keygen -t ed25519"
}

# Новая переменная для ресурсов ВМ
variable "vms_resources" {
  type = map(object({
    cores         = number
    memory        = number
    core_fraction = number
  }))
  default = {
    web = {
      cores         = 2
      memory        = 1
      core_fraction = 5
    }
    db = {
      cores         = 2
      memory        = 2
      core_fraction = 20
    }
  }
  description = "Ресурсы для ВМ"
}

# Новая переменная для метаданных
variable "vms_metadata" {
  type = map(string)
  default = {
    "serial-port-enable" = "1"
    "ssh-keys"           = "ubuntu:ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMzPxyrg408uoTpwNEJwtWKaFxH6EbSbbjBHd7i3NepU khramulka@yandex.ru"
  }
  description = "Метаданные для ВМ"
}

variable "vm_web_platform_id" {
  type        = string
  default     = "standard-v1"
  description = "Платформа ВМ"
}

variable "vm_web_image_family" {
  type        = string
  default     = "ubuntu-2004-lts"
  description = "Семейство образов ОС"
}

variable "vm_db_platform_id" {
  type        = string
  default     = "standard-v1"
  description = "Платформа ВМ"
}

variable "vm_db_image_family" {
  type        = string
  default     = "ubuntu-2004-lts"
  description = "Семейство образов ОС"
}

# Закомментируем устаревшие переменные
# variable "vm_web_cores" {
#   type        = number
#   default     = 2
#   description = "Количество ядер"
# }

# variable "vm_web_memory" {
#   type        = number
#   default     = 1
#   description = "Количество ОЗУ"
# }

# variable "vm_web_core_fraction" {
#   type        = number
#   default     = 5
#   description = "Доля процессорного времени"
# }

# variable "vm_db_cores" {
#   type        = number
#   default     = 2
#   description = "Количество ядер"
# }

# variable "vm_db_memory" {
#   type        = number
#   default     = 2
#   description = "Количество ОЗУ"
# }

# variable "vm_db_core_fraction" {
#   type        = number
#   default     = 20
#   description = "Доля процессорного времени"
# }

```
![1](/img/6.1.png)
------

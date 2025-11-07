---
title: Как установить NGINX на виртуальную машину Ubuntu
description: Следуя данной инструкции, вы сможете установить веб-сервер NGINX на виртуальную машину под управлением операционной системы Ubuntu.
---

# Установка NGINX на виртуальную машину Ubuntu
_NGINX_&nbsp;&mdash; это веб-сервер для обработки HTTP-запросов, который также может выполнять функции прокси-сервера для протоколов TCP/UDP и почты. В качестве обратного прокси-сервера NGINX поддерживает кэширование данных и балансировку нагрузки. Балансировка нагрузки позволяет распределять запросы между несколькими серверами.

Чтобы установить веб-сервер NGINX на Ubuntu:

1. [Подготовьте инфраструктуру](#before-you-begin).
1. [Установите веб-сервер NGINX](#nginx-install).
1. [Проверьте доступность веб-сервера через внешний IP-адрес ВМ](#nginx-check).

Если созданные ресурсы вам больше не нужны, [удалите их](#clear-out).

## Необходимые платные ресурсы

В стоимость поддержки описываемого решения входит плата за использование вычислительных ресурсов, хранилища и публичного IP-адреса (см. [тарифы Yandex Compute Cloud](https://yandex.cloud/ru/docs/compute/pricing#prices-instance-resources)).

## Перед началом работы {#before-you-begin}

1. [Создайте пару SSH-ключей](https://cloud.yandex.ru/docs/compute/operations/vm-connect/ssh#creating-ssh-keys), которая необходима для подключения к ВМ по протоколу SSH. Открытый ключ размещается на ВМ, а закрытый хранится на устройстве пользователя.
1. Создайте сеть, подсеть, группу безопасности и ВМ одним из способов:

    {% list tabs group=instructions %}

    - Вручную {#console}
  
      1. [Создайте сеть](https://cloud.yandex.ru/docs/vpc/operations/network-create) и [подсеть](https://yandex.cloud/ru/docs/vpc/operations/subnet-create).
      1. [Создайте группу безопасности](https://yandex.cloud/ru/docs/vpc/operations/security-group-create) и добавьте в нее следущие правила для входящего и исходящего трафика:
      
          1. Правило для входящего трафика, разрешающее подключение по SSH:
        
              * **Диапазон портов**&nbsp;&mdash; `22`.
              * **Протокол**&nbsp;&mdash; `TCP`.
              * **Источник**&nbsp;&mdash; `CIDR`.
              * **CIDR блоки**&nbsp;&mdash; `0.0.0.0/0`.

          1. Правило для исходящего трафика, разрешающее все подключения:

              * **Диапазон портов**&nbsp;&mdash; `0-65535`.
              * **Протокол**&nbsp;&mdash; `Any`.
              * **Источник**&nbsp;&mdash; `CIDR`.
              * **CIDR блоки**&nbsp;&mdash; `0.0.0.0/0`.
    
          1. Правило для входящего трафика, разрешающее доступ через внешний IP-адрес:

              * **Диапазон портов**&nbsp;&mdash; `80`.
              * **Протокол**&nbsp;&mdash; `TCP`.
              * **Источник**&nbsp;&mdash; `CIDR`.
              * **CIDR блоки**&nbsp;&mdash; `0.0.0.0/0`.

      1. [Создайте ВМ из публичного образа Linux](https://yandex.cloud/ru/docs/compute/operations/vm-create/create-linux-vm) и укажите следующие параметры:
    
          * **Образ загрузочного диска**&nbsp;&mdash; `Ubuntu`. Выберите последнюю версию или любую, которая вам удобна.
          * **Подсеть**&nbsp;&mdash; выберите ранее созданную подсеть.
          * **Публичный адрес**&nbsp;&mdash; `Автоматически`.
          * **Группы безопасности**&nbsp;&mdash; выберите ранее созданную группу безопасности.
          * Выберите для доступа **SSH-ключ** и добавьте открытую часть SSH-ключа, созданного ранее.

          Чтобы снизить стоимость ВМ, включите опцию `Прерываемая`.

          Параметры вычислительных ресурсов ВМ выберите исходя из предполагаемой нагрузки.

    - Terraform {#tf}
      
      [Terraform](https://www.terraform.io/) позволяет быстро создать облачную инфраструктуру в {{ yandex-cloud }} и управлять ею с помощью файлов конфигураций. В файлах конфигураций хранится описание инфраструктуры на языке HCL (HashiCorp Configuration Language). При изменении файлов конфигураций {{ TF }} автоматически определяет, какая часть вашей конфигурации уже развернута, что следует добавить или удалить.
      
      Terraform распространяется под лицензией [Business Source License](https://github.com/hashicorp/terraform/blob/main/LICENSE), а [провайдер Yandex Cloud для Terraform](https://github.com/yandex-cloud/terraform-provider-yandex) — под лицензией [MPL-2.0](https://www.mozilla.org/en-US/MPL/2.0/).
      
      Подробную информацию о ресурсах провайдера смотрите в документации на сайте [Terraform](https://www.terraform.io/docs/providers/yandex/index.html) или в [зеркале](https://yandex.cloud/ru/docs/terraform).
      
      Если у вас еще нет Terraform, [установите его и настройте провайдер Yandex Cloud](../tutorials/infrastructure-management/terraform-quickstart.md#install-terraform).

      Чтобы создать [сеть](https://cloud.yandex.ru/docs/vpc/operations/network-create), [подсеть](https://yandex.cloud/ru/docs/vpc/operations/subnet-create), [группу безопасности](https://yandex.cloud/ru/docs/vpc/operations/security-group-create) и [ВМ](https://yandex.cloud/ru/docs/compute/operations/vm-create/create-linux-vm):

      1. Опишите в конфигурационном файле создаваемые ресурсы:

          ```hsl
          resource "yandex_compute_disk" "boot-disk" {
            name     = "bootvmdisk"
            type     = "network-hdd"
            zone     = "ru-central1-a"
            size     = 10
            image_id = "fd8slhpjt2754igimqu8"
          }

          resource "yandex_compute_instance" "myvm" {
            name        = "myvm"
            platform_id = "standard-v3"
            zone        = "ru-central1-a"

            resources {
              cores         = 2
              memory        = 4
              core_fraction = 20
            }

            boot_disk {
              disk_id = yandex_compute_disk.boot-disk.id
            }

            network_interface {
              subnet_id          = yandex_vpc_subnet.mysubnet.id
              security_group_ids = [yandex_vpc_security_group.mysg.id]
              nat                = true
            }

            metadata = {
              ssh-keys = "vm-test:ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMeom+5uL+BbOQXujvGuX/6fNn80WqgYI61g********"
            }

            scheduling_policy {
              preemptible = true
            }
          }

          resource "yandex_vpc_network" "mynet" {
            name = "mynet"
          }

          resource "yandex_vpc_subnet" "mysubnet" {
            name           = "mysubnet"
            zone           = "ru-central1-a"
            network_id     = yandex_vpc_network.mynet.id
            v4_cidr_blocks = ["10.1.0.0/24"]
          }

          resource "yandex_vpc_security_group" "mysg" {
            name       = "mysg"
            network_id = yandex_vpc_network.mynet.id
  
            ingress {
              port           = 22
              protocol       = "TCP"
              v4_cidr_blocks = ["0.0.0.0/0"]
            }

            ingress {
              port           = 80
              protocol       = "TCP"
              v4_cidr_blocks = ["0.0.0.0/0"]
            }

            egress {
              protocol       = "ANY"
              v4_cidr_blocks = ["0.0.0.0/0"]
            }
          }
          ```

          Где `yandex_compute_instance.metadata.ssh-keys` — открытая часть SSH-ключа, созданного ранее.

      1. Проверьте корректность настроек.
          
          1. В командной строке перейдите в каталог, в котором расположены актуальные конфигурационные файлы Terraform с планом инфраструктуры.
      
          1. Выполните команду:
      
              ```bash
              terraform validate
              ```
          
          Если в файлах конфигурации есть ошибки, Terraform на них укажет.
      
      1. Создайте ресурсы.
          
          1. Выполните команду для просмотра планируемых изменений:
      
              ```bash
              terraform plan
              ```
          
              Если конфигурации ресурсов описаны верно, в терминале отобразится список изменяемых ресурсов и их параметров. Это проверочный этап: ресурсы не будут изменены.

          1. Если вас устраивают планируемые изменения, внесите их:
              
              1. Выполните команду:
              
                  ```bash
                  terraform apply
                  ```

              1. Подтвердите изменение ресурсов.
              1. Дождитесь завершения операции.

    {% endlist %}


{% endlist %}

## Установите веб-сервер NGINX {#nginx-install}

1. Подключитесь к ВМ:
    
    ```bash
    ssh -i <путь_к_ключу/имя_файла_ключа> <имя_пользователя>@<публичный_IP-адрес_ВМ>
    ```

    Где:

    * `<путь_к_ключу/имя_файла_ключа>`&nbsp;&mdash; путь к приватному ключу, используемому для подключения по SSH.
    * `<имя_пользователя>`&nbsp;&mdash; имя учетной записи пользователя, используемое для подключения к ВМ по SSH.

      Для ВМ, созданной через Terraform, имя учетной записи пользователя по умолчанию&nbsp;&mdash; `ubuntu`.

1. Обновите информацию о пакетах, выполнив команду:

    ```bash
    sudo apt update
    ```

1. Установите NGINX, выполнив команду:
    
    ```bash
    sudo apt install nginx
    ```

1. Проверьте статус NGINX, выполнив команду:

    ```bash
    sudo systemctl status nginx
    ```

    В выводе команды в поле **Active** должно быть указано значение `active (running)`.

## Проверьте доступность веб-сервера через внешний IP-адрес ВМ {#nginx-check}

Чтобы проверить доступность веб-сервера, введите в адресной строке браузера: 

```hsl
http://<публичный_IP-адрес_ВМ>
```

Если отображается страница с приветственным сообщением NGINX, то веб-сервер успешно установлен и работает.

## Удалите созданные ресурсы {#clear-out}

Некоторые ресурсы платные. Чтобы за них не списывалась плата, удалите ресурсы, которые вы больше не будете использовать:

- Вручную {#manual}

  * [Удалите виртуальную машину](https://yandex.cloud/ru/docs/compute/operations/vm-control/vm-delete).
  * Если вы зарезервировали для виртуальной машины публичный статический IP-адрес, [удалите его](https://yandex.cloud/ru/docs/vpc/operations/address-delete).

- Terraform {#tf}

  1. В командной строке перейдите в каталог, в котором расположен актуальный конфигурационный файл Terraform с планом инфраструктуры.
  1. Удалите ресурсы с помощью команды:

      ```bash
      terraform destroy
      ```

      {% note alert %}
      
      Terraform удалит все ресурсы, которые были созданы с его помощью.
      
      {% endnote %}

{% endlist %}
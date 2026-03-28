---
title: Как установить NGINX на виртуальную машину Ubuntu
description: Следуя данной инструкции, вы сможете установить веб-сервер NGINX на виртуальную машину под управлением операционной системы Ubuntu.
---

# Установка NGINX на виртуальную машину Ubuntu
_NGINX_&nbsp;&mdash; это веб-сервер для обработки HTTP-запросов, который также может выполнять функции прокси-сервера для протоколов TCP/UDP и почты. В качестве обратного прокси-сервера NGINX поддерживает кэширование данных и балансировку нагрузки. Балансировка нагрузки позволяет распределять запросы между несколькими серверами.

Этот сценарий описывает процесс установки NGINX на виртуальную машину (ВМ) под управлением операционной системы Ubuntu.

Чтобы установить веб-сервер NGINX на Ubuntu:
1. [Создайте ВМ Ubuntu](#vm-create).
1. [Подключитесь к ВМ Ubuntu](#vm-connect).
1. [Установите веб-сервер NGINX](#nginx-install).
1. [Проверьте доступность веб-сервера через внешний IP-адрес ВМ](#nginx-check).

Если созданные ресурсы вам больше не нужны, [удалите их](#clear-out).

## Перед началом работы {#before-you-begin}

{% note info %}

Если вы используете Terraform, вам необходимо создать только пару SSH-ключей. Остальные ресурсы будут созданы на основе конфигурационного файла.

{% endnote %}

1. [Создайте пару SSH-ключей](https://cloud.yandex.ru/docs/compute/operations/vm-connect/ssh#creating-ssh-keys), которая необходима для подключения к ВМ по протоколу SSH. Открытый ключ размещается на ВМ, а закрытый хранится на устройстве пользователя.
1. При необходимости [создайте сеть](https://cloud.yandex.ru/docs/vpc/operations/network-create) и [подсеть](https://yandex.cloud/ru/docs/vpc/operations/subnet-create).
1. [Создайте группу безопасности](https://yandex.cloud/ru/docs/vpc/operations/security-group-create) и настройте правила для доступа к ВМ по SSH и обеспечения доступности веб-сервера через внешний IP-адрес.

    Добавьте правило для входящего трафика для доступа по SSH:

    * **Диапазон портов**&nbsp;&mdash; `22`.
    * **Протокол**&nbsp;&mdash; `TCP`.
    * **Источник**&nbsp;&mdash; `CIDR`.
    * **CIDR блоки**&nbsp;&mdash; `0.0.0.0/0`.

    Добавьте правило для исходящего трафика для доступа по SSH:

    * **Диапазон портов**&nbsp;&mdash; `0-65535`.
    * **Протокол**&nbsp;&mdash; `Any`.
    * **Источник**&nbsp;&mdash; `CIDR`.
    * **CIDR блоки**&nbsp;&mdash; `0.0.0.0/0`.
    
    Добавьте правило для входящего трафика для доступа через внешний IP-адрес:

    * **Диапазон портов**&nbsp;&mdash; `80`.
    * **Протокол**&nbsp;&mdash; `TCP`.
    * **Источник**&nbsp;&mdash; `CIDR`.
    * **CIDR блоки**&nbsp;&mdash; `0.0.0.0/0`.

## Создайте ВМ Ubuntu {#vm-create}
{% list tabs group=instructions %}

- Консоль управления {#console}

    [Создайте виртуальную машину из публичного образа Linux](https://yandex.cloud/ru/docs/compute/operations/vm-create/create-linux-vm) и укажите следующие параметры:
    
    * **Образ загрузочного диска**&nbsp;&mdash; `Ubuntu`. Выберите последнюю версию или любую, которая вам удобна.
    * **Подсеть**&nbsp;&mdash; выберите ранее созданную подсеть.
    * **Публичный адрес**&nbsp;&mdash; `Автоматически`.
    * **Группы безопасности**&nbsp;&mdash; выберите ранее созданную группу безопасности.
    * Выберите для доступа **SSH-ключ** и добавьте открытую часть SSH-ключа, созданного ранее.

    Параметры вычислительных ресурсов выберите исходя из ваших нужд. Чтобы снизить стоимость ВМ, включите опцию `Прерываемая`.

- Terraform {#tf}
    
    Если у вас еще нет Terraform, [установите его и настройте провайдер Yandex Cloud](https://yandex.cloud/ru/docs/tutorials/infrastructure-management/terraform-quickstart#install-terraform).

    1. Добавьте в конфигурационный файл с расширением `.tf` следующие блоки, описывающие параметры создаваемых ресурсов:

        ```hsl
        resource "yandex_vpc_network" "network-1" {
            name = "my-test-network"
        }

        resource "yandex_vpc_subnet" "subnet-1" {
            name           = "my-test-subnet"
            zone           = "ru-central1-a"
            network_id     = "${yandex_vpc_network.network-1.id}"
            v4_cidr_blocks = ["10.128.0.0/24"]
        }

        resource "yandex_vpc_security_group" "test-sg" {
            name        = "my-test-sg"
            description = "Security group for ubuntu-vm"
            network_id  = "${yandex_vpc_network.network-1.id}"

            ingress {
                protocol       = "TCP"
                description    = "Rule for SSH"
                v4_cidr_blocks = ["0.0.0.0/0"]
                port           = 22
            }

            ingress {
                protocol       = "TCP"
                description    = "Rule for NGINX"
                v4_cidr_blocks = ["0.0.0.0/0"]
                port           = 80
            }

            egress {
                protocol       = "ANY"
                description    = "Rule for SSH"
                v4_cidr_blocks = ["0.0.0.0/0"]
                from_port      = 0
                to_port        = 65535
            }
        }

        resource "yandex_compute_disk" "boot-disk" {
            name     = "bootvmdisk"
            type     = "network-hdd"
            zone     = "ru-central1-a"
            size     = 10
            image_id = "fd8slhpjt2754igimqu8"
        }

        resource "yandex_compute_instance" "vm-1" {
            name        = "ubuntu-vm"
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
                subnet_id          = "${yandex_vpc_subnet.subnet-1.id}"
                security_group_ids = ["${yandex_vpc_security_group.test-sg.id}"]
                nat                = true
            }

            metadata = {
                ssh-keys = "<имя_пользователя>:<содержимое_SSH-ключа>"
            }

            scheduling_policy {
                preemptible = true
            }

        }
        ```
        Где:
        
        * `yandex_vpc_network`&nbsp;&mdash; описание [облачной сети](https://yandex.cloud/ru/docs/vpc/concepts/network).
            * `name`&nbsp;&mdash; имя сети.
        
        * `yandex_vpc_subnet`&nbsp;&mdash; описание подсети, к которой будет подключена ВМ.
            * `name`&nbsp;&mdash; имя подсети.
            * `zone`&nbsp;&mdash; [зона доступности](https://yandex.cloud/ru/docs/overview/concepts/geo-scope).
            * `network_id`&nbsp;&mdash; идентификатор сети.
            * `v4_cidr_blocks`&nbsp;&mdash; список IPv4-адресов, откуда или куда будет поступать трафик. Например, `10.0.0.0/22` или `192.168.0.0/16`. Адреса должны быть уникальными внутри сети. Минимальный размер подсети&nbsp;&mdash; `/28`, а максимальный размер подсети&nbsp;&mdash; `/16`. Поддерживается только IPv4.
        
        * `yandex_vpc_security_group`&nbsp;&mdash; описание [группы безопасности](https://yandex.cloud/ru/docs/vpc/concepts/security-groups).
            * `name`&nbsp;&mdash; имя группы безопасности.
            * `description`&nbsp;&mdash; опциональное описание группы безопасности.
            * `network_id`&nbsp;&mdash; идентификатор сети, которой будет назначена группа безопасности.
            * `ingress` и `egress`&nbsp;&mdash; параметры правил для входящего и исходящего трафика:
                * `protocol`&nbsp;&mdash; протокол передачи трафика. Возможные значения: `tcp`, `udp`, `icmp`, `esp`, `ah`, `any`.
                * `description`&nbsp;&mdash; опциональное описание правила.
                * `v4_cidr_blocks`&nbsp;&mdash; список CIDR и масок подсетей, откуда или куда будет поступать трафик.
                * `port`&nbsp;&mdash; порт для трафика.
                * `from-port`&nbsp;&mdash; первый порт из диапазона портов для трафика.
                * `to-port`&nbsp;&mdash; последний порт из диапазона портов для трафика.
        
        * `yandex_compute_disk`&nbsp;&mdash; описание загрузочного [диска](https://yandex.cloud/ru/docs/compute/concepts/disk):
            * `name`&nbsp;&mdash; имя диска.
            * `type`&nbsp;&mdash; [тип](https://yandex.cloud/ru/docs/compute/concepts/disk#disks_types) диска.
            * `zone`&nbsp;&mdash; зона доступности, в которой будет находиться диск.
            * `size`&nbsp;&mdash; размер диска в ГБ.
            * `image_id`&nbsp;&mdash; идентификатор [образа](https://yandex.cloud/ru/docs/compute/concepts/image) ВМ. Вы можете получить идентификатор образа из [списка публичных образов](https://yandex.cloud/ru/docs/compute/operations/images-with-pre-installed-software/get-list).
        
        * `yandex_compute_instance`&nbsp;&mdash; описание ВМ:
            * `name`&nbsp;&mdash; имя ВМ.
            * `platform_id`&nbsp;&mdash; [платформа](https://yandex.cloud/ru/docs/compute/concepts/vm-platforms).
            * `zone`&nbsp;&mdash; зона доступности, в которой будет находиться ВМ.
            * `resources`&nbsp;&mdash; параметры ресурсов ВМ:
                * `cores`&nbsp;&mdash; количество vCPU.
                * `memory`&nbsp;&mdash; объём RAM в ГБ.
                * `core_fraction`&nbsp;&mdash; гарантированная доля vCPU.     
            * `boot_disk`&nbsp;&mdash; настройки загрузочного диска.
                * `disk_id`&nbsp;&mdash; идентификатор диска.
            * `network_interface`&nbsp;&mdash; настройки [сетевого интерфейса](https://yandex.cloud/ru/docs/compute/concepts/network).
                * `subnet_id`&nbsp;&mdash; идентификатор подсети.
                * `security_group_ids`&nbsp;&mdash; идентификаторы групп безопасности.
                * `nat`&nbsp;&mdash; параметр, отвечающий за автоматическое назначение публичного IP-адреса через NAT при значении `true`.
            * `metadata`&nbsp;&mdash; параметры метаданных.
                * `ssh-keys`&nbsp;&mdash; имя пользователя ВМ и открытая часть SSH-ключа.
            * `scheduling_policy`&nbsp;&mdash; политика планирования.
                * `preemptible`&nbsp;&mdash; параметр, отвечающий за создание прерываемой ВМ при значении `true`.

        {% note info %}

        Если у вас уже есть необходимые ресурсы (облачная сеть, подсеть и группа безопасности), описывать их повторно не нужно. Используйте имена и идентификаторы в соответствующих параметрах.

        {% endnote %}

        {% cut "Пример конфигурационного файла с заранее созданными ресурсами (сетью, подсетью и группой безопасности)" %}

        ```hsl
            resource "yandex_compute_disk" "boot-disk" {
                name     = "bootvmdisk"
                type     = "network-hdd"
                zone     = "ru-central1-a"
                size     = 10
                image_id = "fd8slhpjt2754igimqu8"
            }

            resource "yandex_compute_instance" "vm-1" {
            name        = "ubuntu-vm"
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
                    subnet_id          = "${yandex_vpc_subnet.subnet-1.id}"
                    security_group_ids = ["en****************06"]
                    nat                = true
                }

                metadata = {
                    ssh-keys = "vm-test:ssh-ed25519 AAAAC3********************************************************7s+kzm"
                }

                scheduling_policy {
                    preemptible = true
                }

            }

            resource "yandex_vpc_subnet" "subnet-1" {
                network_id     = "en****************fe"
                v4_cidr_blocks = ["10.128.0.0/24"]
                description    = "Subnet for zone ru-central1-a"
            }
        ```

        Прежде чем переходить к созданию ресурсов, импортируйте подсеть в Terraform, выполнив команду:
        
        ```bash
        terraform import yandex_vpc_subnet.subnet-1 <ID_подсети>
        ```
        
        {% note warning %}

        Если вы не импортируете подсеть, то при последующей проверке конфигурации с помощью команды `terraform validate` возникнет ошибка, так как у Terraform не будет информации об указанном ресурсе.

        {% endnote %}

        {% endcut %}

        Более подробную информацию о ресурсах, которые вы можете создать с помощью Terraform, см. в [документации провайдера](https://terraform-provider.yandexcloud.net/).
    
    1. Создайте ресурсы:
        1. Перейдите в директорию с отредактированным конфигурационным файлом.
        1. Проверьте корректность конфигурационного файла, выполнив команду:
            
            ```bash
            terraform validate
            ```
            Если файл не содержит синтаксических ошибок и является внутренне согласованным, вы увидите сообщение:

            ```bash
            Success! The configuration is valid.
            ```
        1. Проверьте изменения, которые Terraform планирует внести в вашу инфраструктуру, выполнив команду:
            
            ```bash
            terraform plan
            ```            
        1. Примените изменения, предложенные `terraform plan`, выполнив команду:

             ```bash
            terraform apply
            ```
            Для подтверждения изменений введите в терминале `yes` и нажмите **Enter**.
        
        После этого в указанном каталоге будут созданы все требуемые ресурсы. Проверить появление ресурсов и их настройки можно в [консоли управления](https://console.yandex.cloud/).

{% endlist %}
## Подключитесь к ВМ Ubuntu {#vm-connect}
В терминале выполните команду:

```bash
ssh -i <путь_к_ключу/имя_файла_ключа> <имя_пользователя>@<публичный_IP-адрес_ВМ>
```

Где:
* `<путь_к_ключу/имя_файла_ключа>`&nbsp;&mdash; путь к приватному ключу, используемому для подключения по SSH.
* `<имя_пользователя>`&nbsp;&mdash; имя учетной записи пользователя ВМ.

{% note warning %}

Для ВМ, созданной через Terraform, имя учетной записи пользователя по умолчанию&nbsp;&mdash; `ubuntu`.

{% endnote %}

## Установите веб-сервер NGINX {#nginx-install}

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

Чтобы проверить доступность веб-сервера, в адресной строке браузера введите:

```hsl
http://<публичный_IP-адрес_ВМ>
```

Если открылась страница с приветственным сообщением от NGINX, значит веб-сервер успешно установлен и работает.

## Удалите созданные ресурсы {#clear-out}
Удалите ресурсы, которые вы больше не будете использовать, чтобы за них не списывалась плата:
* [Удалите виртуальную машину](https://yandex.cloud/ru/docs/compute/operations/vm-control/vm-delete).
* Если вы зарезервировали для виртуальной машины публичный статический IP-адрес, [удалите его](https://yandex.cloud/ru/docs/vpc/operations/address-delete).
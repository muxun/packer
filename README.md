# Packer

# def

Packer - приложение семейства HarshiCorp для создания образов виртуальных машин  различных платформенных провайдеров - GCE AWS VirtualBox и т.п. для создания golden images.

packer читает шаблон, и с помощью builders запускает VM,  далее провижинами устанавливает необходимое ПО и конфигурации, после чего делает образ и удаляет VM. Образ можно оставить на платформе или как артефакт задеплоить.

# mutable и immutable infrastructure

Вот исчерпывающая цитата от Александра Сулейманова (express42):

> 1. Есть mutable infrastructure, когда мы меняем состояние компонентов после того, как их ввели в эксплутацию (обновляем пакеты, устанавливаем зависимости и т.п.).
2. Есть immutable infrastructure - когда компонент инфраструктуры, по большей части, неизменяем. В идеале - вообще не хранит на себе никакого состояния. Мы не чиним и не лечим такие элементы инфраструктуры, мы запускаем новый инстанс и удаляем старый (если не нужен rollback).
Для первого подхода мы используем подход `fry`, когда запускаем базовый образ и после запуска инстанса обеспечиваем “сходимость” между желаемым и текущим состоянием (в этот момент все может упасть, может разъехаться потом)
Для второго подхода используем `bake` - когда инстанс создается из образа почти в “целевом состоянии” - в нем есть ВСЕ кроме того, что отличается между инстансами (IP, hostname и т.п.). С технической точки зрения стараемся сделать так, чтобы изменения НЕЛЬЗЯ было внести в такую систему (Read-only root и подобное).
Golden Image - это образ для создания однотипных инстансов (для fry - базовая ОС, для bake - полностью собранное окружение)

# Установка  packer

packer поставляется бинарником 

    $ cd ~
    $ wget https://releases.hashicorp.com/packer/1.3.3/packer_1.3.3_linux_amd64.zip
    $ unzip packer_1.3.3_linux_amd64.zip
    $ sudo mv packer /usr/lib
    $  rm packer_1.3.3_linux_amd64.zip

проверяем

    $ packer version
    Packer v1.3.3

# Создание образа

Конфигурационный шаблон для packer написан на json

Шаблон содержит три основных блока:

**variables -** задаем переменные ****

**builders -** определяем провайдера и опции для создания образа

**provisioners -** определяем действия после раскатки ОС - производим настройки системы и конфигурации приложений на созданной VM.

**variables** принимает map, **builders** и **provisioners** - массив map. В одном шаблоне можно задать несколько провайдеров ( builders) и провиженов.
**packer** поддерживает следующие провижины:

- Ansible
- Chef
- Salt
- Shell
- Powershell
- win cmd
- file - for copying file from local dir to VM image

# GCP image

Для примера шаблон для создания образа в GCP

ubuntu.json

    {
       "variables": 
            {
            "project_id": null,
            "source_image_family": null,
            "machine_type": "f1-micro"
            }
            ,
    
      "builders": [
            {
            "type": "googlecompute",
            "project_id": "{{user `project_id`}}",
            "image_name": "reddit-base-{{timestamp}}",
            "image_family": "reddit-base",
            "source_image_family": "{{user `source_image_family`}}",
            "zone": "europe-west1-b",
            "ssh_username": "muxun",
            "machine_type": "{{user `machine_type`}}"
            }
            ],
    
     "provisioners": [
            {
            "type": "shell",
            "script": "script/install_ruby.sh",
            "execute_command": "sudo {{.Path}}"
            },
    
            {
            "type": "shell",
            "script": "script/install_mongodb.sh",
            "execute_command": "sudo {{.Path}}"
            }
    
            ]
    }

где 

## viraibles

**"project_id": null**,  - id проекта

**"source_image_family": null,** - семейство iso 

**"machine_type": "f1-micro"** - тип ВМ GCE

project_id и source_image_family имеют значение null, что означает необходимость обязательного определения и не имеет значения по умолчанию. Эти переменный могут быть определены из командной стройки или из файла **variables.json**

переменная machine_type определена

отдельно создан файлик **variables.json** 

    {
    
    "project_id": "infra-111111",
    "source_image_family": "ubuntu-1604-lts"
    
    }

где определены пользовательские переменные 

запуск билда с файлом переменных происходит командой:

    packer build -var-file=variables.json ubuntu.json

## builders

**"type": "googlecompute"**, - что будет вм для билда образа (тут google compute engine)
**"project_id": "{{user `project_id`}}",** -  id проекта в gcp. тут используется пользовательская переменная из блока variables, которая в свою очередь, требует определения при запуске packer.у нас будет отдельный файл variables.json, где будут заданы эти параметры
**"image_name": "reddit-base-{{timestamp}}",** - имя образа
**"image_family": "reddit-base",** - семейство образов
**"source_image_family": "{{user `source_image_family`}}",** - что взять за базовый образ
**"zone": "europe-west1-b",** - зона в которой запускается vm для создания образа
**"ssh_username": "user",** - временный пользователь, который будет создан для подключения к VM во время билда и выполнения команд провижена
**"machine_type": "{{user `machine_type`}}"** - тип инстанса,запускаемого для билда

## provisioners

используется для установки нужного ПО и и настройки системы и конфигурации внутри созданной VM

так как у нас массив мэпов, мы можем указать несколько провижинеров.в примере стоят два shell провиженира, которые позволяют запускать  bash команды в инстансе

**"type": "shell", -** тип провижена shell
**"script": "script/install_ruby.sh", -** ссылка на скрипт
**"execute_command": "sudo {{.Path}}" -** исполнение команды

содержимое скрипта **install_ruby.sh**

    #!/bin/bash
    set -e
    
    #Install ruby
    apt update
    apt install -y ruby-full ruby-bundler build-essential

Аналогичным образом устроен скрипт **install_mongodb.sh**

# build образа

билд образа происходит командой 

    packer build -var-file=variables.json ubuntu.json

перед билдом можно проверить на правильность конфигурации следующей командой

    packer validate -var-file=variables.json ubuntu.json

после создания образа, он будет доступен в пользовательских образах CGP

# Создание bake образа на примере django сервера

Создадим запеченный образ для развертывания работающего django сервера в GCP

шаблон djangoVM.json

    

скрпит инсталяции python3 и django

    python3 manage.py shell -c "from django.contrib.auth.models import User; User.objects.create_superuser('django', 'django@example.com', '12344321')"
    sudo ufw allow 7777
    python3 manage.py runserver 0.0.0.0:7777

запуск билда

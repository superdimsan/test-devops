## Описание задачи
  - Создать виртуальную машину на базе [Ubuntu 20.04](https://app.vagrantup.com/generic/boxes/ubuntu2004) с помощью [Vagrant](https://www.vagrantup.com/).
  - Настроить виртуальную машину через [Ansible](https://www.ansible.com/).
  - Создать отдельные контейнеры с [Prometheus](https://prometheus.io/) и [Grafana](https://grafana.com/) при помощи [docker-compose.yml](https://docs.docker.com/compose/compose-file/compose-file-v3/).
  - Добавить dashboard [Node Exporter Full](https://grafana.com/grafana/dashboards/1860-node-exporter-full/), который визуализирует метрики запущенной VM.
  - После выполнения команды `vagrant up` в браузере по адресу [http://localhost:3000](http://localhost:3000) должна открываться [Grafana](https://grafana.com/) с [Node Exporter Full](https://grafana.com/grafana/dashboards/1860-node-exporter-full/).

## Запуск проекта:
```shell
git clone git@github.com:superdimsan/test-devops.git &&\
cd test-devops &&\
vagrant up
```
Логин и пароль в grafana: admin / admin

## Подготовка рабочего окружения и настройка Vagrant

Создадим папку для проекта и установим Virtualbox, Vagrant, Ansible:
 
```shell
sudo apt update && \
sudo apt install -y virtualbox vagrant ansible && \
mkdir -p ~/deploy && \
cd deploy && \
vagrant init ubuntu/focal64
```
Добавим в Vagrantfile в текущей директории следующие строки:
```shell
nano Vagrantfile
```

```shell
#Проброс порта 3000, который слушает Grafana, на 3000 порт хостовой машины
  config.vm.network "forwarded_port", guest: 3000, host: 3000

#Добавляем playbook.yml для автоматического развертывания конфигурации виртуальной машины
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
```  

Если виртуальная машина не создаётся и не может скачаться её образ с официального репозитория по причине блокирования соединений из РФ, то добавляем в Vagrantfile альтернативный репозиторий:
```shell
echo "ENV['VAGRANT_SERVER_URL'] = 'https://vagrant.elab.pro' " >> Vagrantfile
```
## Описание установки docker в ansible playbook

Создадим в рабочей директории файл playbook.yml. Опишем синхронизацию списка пакетов в виртуальной машине и установку на ней docker:

```shell
nano playbook.yml 
```

```yml
--- 
- name: Configure self-monitoring host
  hosts: all
  remote_user: vagrant
  become: true

  tasks:

  - name: Install apt utilities
    apt:
        name: "{{item}}"
        state: present
        update_cache: yes
    loop:
        - ca-certificates
        - curl

  - name: Add GPG key
    apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

  - name: Add the repository to Apt sources
    apt_repository:
        repo: deb https://download.docker.com/linux/ubuntu focal stable
        state: present

  - name: Install the Docker packages
    apt:
        name: "{{item}}"
        state: latest
        update_cache: yes
    loop:
        - docker-ce
        - docker-ce-cli
        - containerd.io

  - name: Check docker is active
    service:
        name: docker
        state: started
        enabled: yes

  - name: Ensure group "docker" exists
    ansible.builtin.group:
        name: docker
        state: present

  - name: adding user vagrant to docker group
    user:
        name: vagrant
        groups: docker
        append: yes
```

## Описание установки сервиса сбора метрик Node exporter для Prometheus

Для этого создадим в рабочей папке файл node_exporter.service, в котором пропишем следующую конфигурацию:

```shell
nano node_exporter.service
```

```shell
[Unit]
Description=Node Exporter

[Service]
User=node_exporter
EnvironmentFile=/etc/sysconfig/node_exporter
ExecStart=/usr/sbin/node_exporter $OPTIONS

[Install]
WantedBy=multi-user.target
```

Опишем в playbook.yml установку и запуск сервиса Node Exporter:

```yml
  - name: Create Node Exporter user
    user:
      name: node_exporter
      system: yes
      shell: /usr/sbin/nologin

  - name: Download and extract Node Exporter
    get_url:
        url: "https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz"
        dest: "/tmp/node_exporter-1.7.0.linux-amd64.tar.gz"

  - name: Extract Node Exporter
    ansible.builtin.unarchive:
        src: "/tmp/node_exporter-1.7.0.linux-amd64.tar.gz"
        dest: "/tmp/"
        remote_src: yes

  - name: Copy file with owner and permissions
    ansible.builtin.copy:
        src: /tmp/node_exporter-1.7.0.linux-amd64/node_exporter
        dest: /usr/sbin/node_exporter
        mode: "0755"
        owner: "node_exporter"
        group: "node_exporter"
        remote_src: yes

  - name: Copy node_exporter.service file in /etc/systemd/system/node_exporter.service
    ansible.builtin.copy:
        src: node_exporter/node_exporter.service
        dest: /etc/systemd/system/node_exporter.service
        mode: "0644"
        owner: "root"
        group: "root"
  
  - name: Settings for using node_exporter options
    shell: mkdir -p /etc/sysconfig && touch /etc/sysconfig/node_exporter && OPTIONS="--collector.textfile.directory /var/lib/node_exporter/textfile_collector"

  - name: Starting node_exporter.service
    ansible.builtin.systemd:
      name: node_exporter.service
      daemon_reload: true
      state: started
      enabled: true
```



## Описание настройки развертывания сервисов prometheus и grafana в docker compose.
Создадим в рабочей папке директории ./compose/prometheus, а в них - конфигурационный файл prometheus.yml:

```shell
mkdir -p compose/prometheus && \
touch compose/prometheus/prometheus.yml
```
Укажем необходимые параметры в prometheus.yml:

```shell
scrape_configs:
  - job_name: node
    scrape_interval: 5s
    static_configs:
    - targets: ['localhost:9100']
```

Создадим папку ./compose/grafana и поместим в нее поддиректории с конфигурационными файлами.
Опишем создание сервиса в файле docker-compose.yml, который разместим в директории ./compose:

```shell
nano docker-compose.yml
```

```yml
version: '3.9'

services:

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus:/etc/prometheus/
    container_name: prometheus
    hostname: prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
    ports:
      - 9090:9090
    restart: unless-stopped
    environment:
      TZ: "Europe/Moscow"
    network_mode: host

  grafana:
    image: grafana/grafana
    user: root
    depends_on:
      - prometheus
    ports:
      - 3000:3000
    volumes:
      - ./grafana:/var/lib/grafana
    container_name: grafana
    hostname: grafana
    restart: unless-stopped
    environment:
      - TZ="Europe/Moscow"
      - GF_DASHBOARDS_DEFAULT_HOME_DASHBOARD_PATH=/var/lib/grafana/dashboard/node-exporter-full.json
    network_mode: host

networks:
  host:
    external: true
```

Дополняем playbook.yml описанием настройки и запуска docker compose. Копируем директорию compose на удалённую машину, скачиваем dashboard node-exporter-full в нужную директорию в соответствии с переменной GF_DASHBOARDS_DEFAULT_HOME_DASHBOARD_PATH в docker-compose.yml.

```yml
  - name: Creates directory for docker compose sources
    ansible.builtin.file:
      path: /home/vagrant/compose
      state: directory


  - name: Copy docker compose sources
    copy:
      src: ./compose 
      dest: /home/vagrant/
      mode: "0755"
      owner: "vagrant"
      group: "vagrant"


  - name: Download dashboard node-exporter-full
    get_url:
      url: https://github.com/rfmoz/grafana-dashboards/blob/master/prometheus/node-exporter-full.json
      dest: /home/vagrant/compose/grafana/dashboard/node-exporter-full.json
      mode: '0755'

  - name: Docker compose up
    shell:
      cmd: "cd /home/vagrant/compose/ && docker compose -f /home/vagrant/compose/docker-compose.yml up -d"
```

## Push проекта на Github.

Для загрузки проекта в Github создаем репозиторий superdimsan/test-devops. Генерируем ssh-ключ.
```shell
ssh-keygen -t ed25519 -C "superdimsan@gmail.com"
cat ~/.ssh/id_ed25519.pub
```

Добавим новый ключ в Github->Settings->SSH keys, копируя туда содержимое ~/.ssh/id_ed25519.pub

Для того, чтобы на Github можно было выложить пустые конфигурационные папки grafana, создадим файл .gitignore со следующим содержанием:

```shell
nano .gitignore
```

```shell
# Ignore everything in this directory
*
# Except this file
!.gitignore
```
и скопируем его в необходимые папки
```shell
cp .gitignore compose/grafana/config &&\
cp .gitignore compose/grafana/csv &&\
cp .gitignore compose/grafana/dashboard &&\
cp .gitignore compose/grafana/pdf &&\
cp .gitignore compose/grafana/plugins &&\
cp .gitignore compose/grafana/png &&\
cp .gitignore compose/grafana/provisioning &&\
rm .gitignore
```

Устанавливаем git и выкладываем проект на github:

```shell
sudo apt install -y git && \
git config --global user.name "d.poshivalnikov" && \
git config --global user.email "d.poshivalnikov@yandex.ru" && \
git init && \
git add -A && \
git commit -m "Devops test work" && \
git branch -M main && \
git remote add origin git@github.com:superdimsan/test-devops.git && \
git push -u origin main
```
Установка и настройка проекта производилась на Ubuntu 20.04


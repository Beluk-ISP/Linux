Перед началой работой нам нужно будет установить виртуальную машину на Oracle VM VirtualBox.
При этом для комфортной работы, машине нужно будет выделить:
- 20ГБ+ места на жестком диске
- Более 4096МБ оперативной памяти
- Более 2 ядра процесса
  
Работа будет разделена на 3 части: Docker Compose, Prometheus и VictoriaMetric

# 1. Установка Grafana Stack

1. Установка Docker'а

Установка wget для скачивания файлов.

        sudo yum install wget

![изображение](https://github.com/user-attachments/assets/118c8799-df1c-494e-8e21-19dfc32a617b)

Скачивание файла репозитория Docker CE для CentOS и размещение его в директории /etc/yum.repos.d/.

    sudo wget -P /etc/yum.repos.d/ https://download.docker.com/linux/centos/docker-ce.repo

![изображение](https://github.com/user-attachments/assets/00eeb238-870f-408c-8ee3-48180ea5a7c9)

Установка Docker Engine (docker-ce), клиентских инструментов Docker (docker-ce-cli) и контейнерного рантайма containerd.

        sudo yum install docker-ce docker-ce-cli containerd.io

![изображение](https://github.com/user-attachments/assets/a00c5175-5e07-48ec-8aeb-d0ce7ab28abd)

Включение и немедленный запуск службы Docker.

        sudo systemctl enable docker --now
        
![изображение](https://github.com/user-attachments/assets/5afb3daf-2781-41ad-98c5-6b89011d76fa)

2. Установка Compose

Установка утилиты curl (на CentOS/RHEL) для выполнения HTTP-запросов.

       sudo yum install curl
   
![изображение](https://github.com/user-attachments/assets/bdec9016-760a-43d9-b478-b0d0e5fa6d33)

Использование команды curl для получения последней версии Docker Compose с GitHub API. Фильтрация ответа для извлечения тега с версией.

      COMVER=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep 'tag_name' | cut -d\" -f4)

* В консоль не будет вывода

Загрузка последней версии Docker Compose с официального репозитория GitHub и сохранение её в /usr/bin/docker-compose.

      sudo curl -L "https://github.com/docker/compose/releases/download/$COMVER/docker-compose-$(uname -s)-$(uname -m)" -o /usr/bin/docker-compose

![изображение](https://github.com/user-attachments/assets/f082e453-8406-4905-b907-ced9119dd1cd)

Предоставление прав на выполнение файла docker-compose.

      sudo chmod +x /usr/bin/docker-compose

* В консоль не будет вывода 

Проверка установленной версии Docker Compose.

      sudo docker-compose --version

![изображение](https://github.com/user-attachments/assets/3754d1c0-75b5-487a-ada5-49899441bbda)

4. Делаем Grafana

Установка git'а

        sudo yum install git

![изображение](https://github.com/user-attachments/assets/be886abb-71d7-4e59-8cc6-a28312eb0a4f)

Клонирование репозитория Grafana Stack для Docker с GitHub.

    sudo git clone https://github.com/skl256/grafana_stack_for_docker.git

![изображение](https://github.com/user-attachments/assets/6b6b8c96-9204-4a26-8654-392d63717be0)

Заходим в папку grafana_stack_for_docker

    cd grafana_stack_for_docker

![изображение](https://github.com/user-attachments/assets/b8dbaa3a-117d-4804-be95-a43d3fa533d5)

Cоздаем папки двумя разными способами

Создание каталога для конфигурации Grafana внутри общей директории в Docker Swarm.

     sudo mkdir -p /mnt/common_volume/swarm/grafana/config

Создание нескольких директорий для хранения данных конфигурации и информации для Grafana, Prometheus, Loki, Promtail.

     sudo mkdir -p /mnt/common_volume/grafana/{grafana-config,grafana-data,prometheus-data,loki-data,promtail-data}


Выдаем права

     sudo chown -R $(id -u):$(id -g) {/mnt/common_volume/swarm/grafana/config,/mnt/common_volume/grafana}

Создаем файл

     sudo touch /mnt/common_volume/grafana/grafana-config/grafana.ini

Копирование файлов

     sudo cp config/* /mnt/common_volume/swarm/grafana/config/

Переименовывание файла

     sudo mv grafana.yaml docker-compose.yaml

Запуск сервисов, описанных в docker-compose.yaml, в фоновом режиме.

     sudo docker compose up -d

![изображение](https://github.com/user-attachments/assets/6cf9fb65-8254-498a-ad13-54e9fc23b63c)

После запуска, нам нужно будет на время остановить docker - sudo docker compose stop

После остановки, продолжаем работать дальше

    sudo vi docker-compose.yaml

Затем в docker-compose нужно вставить node-exporter:

    node-exporter:
    image: prom/node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    container_name: exporter < запомнить название name
    hostname: exporter
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --collector.filesystem.ignored-mount-points
      - ^/(sys|proc|dev|host|etc|rootfs/var/lib/docker/containers|rootfs/var/lib/docker/overlay2|rootfs/run/docker/netns|rootfs/var/lib/docker/aufs)($$|/)
    ports:
      - 9100:9100
    restart: unless-stopped
    environment:
      TZ: "Europe/Moscow"
    networks:
      - default

![изображение](https://github.com/user-attachments/assets/f3996512-c3af-462b-8d47-2858d0a49e4a)

Заходим в другую папку

    cd /mnt/common_volume/swarm/grafana/config

Либо

    cd grafana_stack_for_docker/config

Заходим в редактирование файласв

     sudo vi prometheus.yaml

Далее нужно исправить targets: на exporter:9100

![изображение](https://github.com/user-attachments/assets/d68e21a1-f107-42e3-875b-b486394bb82d)

Сохраняем при помощи :wq! и снова запускаем Docker Compose

# 2. Установка Prometheus

Для установки и дальнейшей настройки, нам нужно зайти на сайт: http://localhost:3000

Логин и пароль: Admin

В меню выбираем вкладку Dashboards, нажимаем на кнопку "+ create dashnoard", затем на "+ add visualization" и после на "configure a new data source".
выбираем Prometheus, в connection нужно будет ввести: "http://prometheus:9090".
в authentication нужно будет поставить "Basic authentication" и ввести логин и пароль: admin, после этого нажимаем на "Save & test" и должно показать зеленую галочку
Authentication

![изображение](https://github.com/user-attachments/assets/e5a7193a-e78f-42e3-9105-171a03cf653d)

возвращаемся обратно в Dashboards и нажимаем снова Create Dashboard
нажимаем Import dashboard и в поле "Find and import dashboards" нужно будет ввести "1860", потом нужно будет выбрать Prometheus и импортировать dashboard

![изображение](https://github.com/user-attachments/assets/ffe6ca16-f9ec-4c08-93fa-9c7e8ed6ef72)

В конечном итоге, должно получиться вот так:

![386302996-ab0d7adb-3d14-4138-b2dc-82eba6cf24c9](https://github.com/user-attachments/assets/1a94b40b-563c-4bb0-a5d9-5b78d942336f)

# 3. Делаем VictoriaMetrics

Для начала зайдем в нужную папку

     cd grafana_stack_for_docker

Открываем docker-compose

     sudo vi docker-compose.yaml

После prometheus вставляем vmagent

      vmagent:
    container_name: vmagent
    image: victoriametrics/vmagent:v1.105.0
    depends_on:
      - "victoriametrics"
    ports:
      - 8429:8429
    volumes:
      - vmagentdata:/vmagentdata
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - "--promscrape.config=/etc/prometheus/prometheus.yml"
      - "--remoteWrite.url=http://victoriametrics:8428/api/v1/write"
    restart: always
      # VictoriaMetrics instance, a single process responsible for
      # storing metrics and serve read requests.
      victoriametrics:
        container_name: victoriametrics
        image: victoriametrics/victoria-metrics:v1.105.0
        ports:
          - 8428:8428
          - 8089:8089
          - 8089:8089/udp
          - 2003:2003
          - 2003:2003/udp
          - 4242:4242
        volumes:
          - vmdata:/storage
        command:
          - "--storageDataPath=/storage"
          - "--graphiteListenAddr=:2003"
          - "--opentsdbListenAddr=:4242"
          - "--httpListenAddr=:8428"
          - "--influxListenAddr=:8089"
          - "--vmalert.proxyURL=http://vmalert:8880"
        restart: always

Затем возвращаемся в localhost и переходим во вкладку connection, нажимаем на data sources и затем на prometheus. Нам нужно будет поменять в connection с "http://prometheus:9090" на "http://victoriametrics:8428" и поменять название на "ViktoriaMetric" 

![изображение](https://github.com/user-attachments/assets/f81d318d-1c06-406b-b6ed-38f2a2bb559a)

После возвращаемся в Dashboards и нажимаем на Add visualization, на этот раз выбираем VikMetric, в созданной Dashboard нам нужно будет выбрать вкладку "code"

![изображение](https://github.com/user-attachments/assets/d2352b31-ca3e-4f83-af92-280db7104a7f)

После переходим в терминал и пишем эти команды:

     echo -e "# TYPE OILCOINT_metric1 gauge\nOILCOINT_metric1 0" | curl --data-binary @- http://localhost:8428/api/v1/import/prometheus

     curl -G 'http://localhost:8428/api/v1/query' --data-urlencode 'query=OILCOINT_metric1'

![изображение](https://github.com/user-attachments/assets/6af8ffd5-9072-4e1d-97d9-df0839ce1e11)

Копируем переменную OILCOINT_metric1 и вставляем в code и нажимаем run queries

![изображение](https://github.com/user-attachments/assets/20d90e97-8f10-4428-97b4-be9a56df2277)



## Лаба 2. Установить ELK и подключить данные из Kafka

​                                       <img src="http://data.newprolab.com/public-newprolab-com/de_lab02_elk.png" width="470px" align="center">    

### 1. Установка ELK

ELK — стэк технологий Elasticsearch, Logstash и Kibana. Elasticsearch — это распределенный RESTful поиск. Logstash позволяет принимать на вход данные из разных источников, преобразовывать их и складывать в нужное место, например, в Elasticsearch. Kibana позволяет визуализировать данные. В каком-то виде этот стэк позволяет собрать самостоятельный полноценный пайплайн: от сбора и организации данных до их визуализации и анализа (в том числе есть и инструменты машинного обучения).

#### 1.1. Установка Elasticsearch

Правилом хорошего тона является установка инструментов не через распаковку архивов, а при помощи установки пакетов. У Elasticsearch есть такая возможность. Зайдите на каждую (!) машину и выполните следующие команды:

```bash
$ wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
$ sudo apt-get install apt-transport-https
$ echo "deb https://artifacts.elastic.co/packages/5.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-5.x.list
$ sudo apt-get update && sudo apt-get install elasticsearch
```

Установим сразу же <a href="https://www.elastic.co/products/x-pack">плагин x-pack</a>, позволяющий использовать в том числе machine learning прямо в elasticsearch.

```bash
$ cd /usr/share/elasticsearch/bin/
$ sudo ./elasticsearch-plugin install x-pack
```

Вероятно, вывалится ошибка, ссылающаяся на Java. И правильно, мы же в предыдущей лабе ее установили только на мастере. Установите ее на оставшихся нодах.

**Hint**: для того чтобы была возможность переходить от одной машины к другой напрямую, добавьте свой `private_key`, при помощи которого вы изначально заходите на ваш мастер, в директорию: `~/.ssh/`

Как правило, после этого еще требуется поменять права на этот файл:

```bash
$ chmod 0600 <путь к вашему ключу>
```

После этого вы сможете просто набрать с этой машины, например:

```bash
$ ssh node2
```

И окажетесь на втором сервере. Соответственно, чтобы переходить с любой машины на любую, нужно положить свой ключ в соответствующую папку на всех серверах.

#### 1.2. Конфигурация Elasticsearch

Конфиг-файл Elasticsearch находится отдельно на каждой машине здесь: `/etc/elasticsearch/elasticsearch.yml`.

Для того чтобы работать в режиме кластера, нам нужно будет прописать в конфиг-файле одно и то же название кластера. Если все машины находятся внутри одной сети, то кластер самоорганизуется за счет самостоятельного общения нод.

При попытке попасть в указанную папку может появиться ошибка `permission denied`. Для того чтобы от нее избавиться, нужно поменять на нее права.

```bash
$ sudo chmod -R 0755 /etc/elasticsearch/
```

И после этого:

```bash
$ cd /etc/elasticsearch/
$ sudo nano elasticsearch.yml
```

В нем следует раскомментить и поменять следующие параметры:

* `cluster.name`: *<назовите как-то ваш кластер>* — это название вы будете использовать на всех машинах,
* `node.name`: *<назовите как-то вашу ноду>* — у каждой ноды должно быть свое уникальное название,
* `network.host`: 0.0.0.0
* `http.port`: 9200
* `discovery.zen.ping.unicast.hosts`: ["node1", "node2", "node3"] — имена хостов, к которым можно обращаться, когда нода будет искать кластер. На каждой ноде этот список будет своим (все минус нода, на которой мы меняем конфиг).

Этот файл можно сохранить и скопировать на все остальные ноды. Нужно будет изменить только `node.name`.

Давайте теперь проверим, что всё работает. Запустим на каждой ноде команду:

```bash
$ sudo -i service elasticsearch start
```

А затем:

```bash
$ curl -XGET 'localhost:9200/?pretty'
```

В ответ должны получить что-то подобное:

```
{
  "name" : "node4",
  "cluster_name" : "newprolab",
  "cluster_uuid" : "FKE_a3UTSrifc28ppmkrcQ",
  "version" : {
    "number" : "5.6.3",
    "build_hash" : "1a2f265",
    "build_date" : "2017-10-06T20:33:39.012",
    "build_snapshot" : false,
    "lucene_version" : "6.6.1"
  },
  "tagline" : "You Know, for Search"
}
```

Если встречаете `Failed to connect to localhost port 9200: Connection refused`, то скорее всего у вас Elasticsearch еще просто не стартовал, немного подождите.

Если ссылается на `missing authentication token`, то нужно в конфиг добавить вот эти 2 строчки:

```
xpack.security.enabled: false
xpack.security.authc.token.enabled: false
```

После этого понадобится на каждой ноде сделать рестарт:

```bash
$ sudo -i service elasticsearch restart
```

Проделав это всё на каждой ноде, давайте поймем собрался ли из них кластер Elasticsearch. На любой из нод запустите:

```bash
$ curl -XGET 'localhost:9200/_cluster/health?pretty'
```

В ответ должны получить что-то такое:

```
{
  "cluster_name" : "newprolab",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 4,
  "number_of_data_nodes" : 4,
  "active_primary_shards" : 11,
  "active_shards" : 8,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

Проверьте, чтобы `number_of_nodes` был равен реальному количеству ваших нод. Если всё так, то кластер elasticsearch собран.

#### 1.3. Установка Kibana

Устанавливать Kibana на каждой машине не требуется, но в <a href="https://www.elastic.co/guide/en/kibana/5.6/production.html">документации указано</a>, что если вы будет использовать ее в продакшене на кластере, то лучше одну ноду из Elasticsearch кластера вынести и сделать ее `Coordinating only node`.

Будем двигаться по порядку. Устанавливаем Kibana аналогично — через пакет. Будем делать это на мастер-ноде, поскольку у нее есть статичный ip, на который мы будем через браузер заходить для запуска Kibana.

```bash
$ wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
$ sudo apt-get install apt-transport-https
$ echo "deb https://artifacts.elastic.co/packages/5.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-5.x.list
$ sudo apt-get update && sudo apt-get install kibana
```

Теперь зайдем в конфигурационный файл Elasticsearch и добавим туда следующие строки, чтобы сделать мастер-ноду только координирующей нодой в elasticsearch-кластере:

```
node.master: false
node.data: false
node.ingest: false
```

Сделаем рестарт Elasticsearch сервера.

```bash
$ sudo -i service elasticsearch restart
```

Убедимся, что нода изменила свой статус.

```bash
$ curl -XGET 'localhost:9200/_cluster/health?pretty'
```

Установим плагин x-pack.

```bash
$ cd /usr/share/kibana/bin/
$ sudo ./kibana-plugin install x-pack
```

Зайдем в конфиг Kibana:

```bash
$ cd /etc/kibana/
```

Разкомментим порт и поменяем имя сервера на:

```
server.port: 5601
server.host: 0.0.0.0
```

Стартанем Kibana:

```bash
$ sudo -i service kibana start
```

После чего зайдем через браузер на порт 5601. Логин: elastic. Пароль: changeme. Kibana работает.

#### 1.4. Установка Logstash

Logstash тоже устанавливается только на одну машину.

```bash
$ wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
$ sudo apt-get install apt-transport-https
$ echo "deb https://artifacts.elastic.co/packages/5.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-5.x.list
$ sudo apt-get update && sudo apt-get install logstash
```

Установим опять же x-pack и сразу же плагин для приема сообщений из Kafka:

```bash
$ cd /usr/share/logstash/bin
$ sudo ./logstash-plugin install x-pack
$ sudo ./logstash-plugin install logstash-input-kafka
```

Стартанем Logstash:

```bash
$ sudo -i service logstash start
```

### 2. Привязка ELK к Kafka

А это и есть ваша лаба. Подключите ELK к вашем потоку данных в Kafka. Постройте свою первую визуализацию логов.

<img src="http://data.newprolab.com/public-newprolab-com/de_lab02_kibana.png" align="center">

### 3. Проверка
Проверка осуществляется из Личного кабинета , используя REST API ElasticSearch. Он подключится к указанному вами хосту на 9200 порте. Индекс в ElasticSearch должен совпадать с логином ЛК.


### 4. Ссылки для изучения

* <a href="https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started.html ">Документация по Elasticsearch</a> 
* <a href="https://www.elastic.co/guide/en/kibana/current/introduction.html">Документация по Kibana</a>
* <a href="https://www.elastic.co/guide/en/logstash/current/introduction.html">Документация по Logstash</a> 
* <a href="https://github.com/lmenezes/cerebro">Инcтрумент мониторинга работы Elasticsearch</a>

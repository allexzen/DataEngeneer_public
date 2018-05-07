## Лаба 3. Написать планировщик, который будет токенезировать URL и сохранять в Parquet

​ <img src="http://data.newprolab.com/public-newprolab-com/de_lab03_airflow.png" width="170px" align="center">

### 1. Десериализация логов

Если вы всё сделали в прошлой лабе по инструкции, то скорее всего вы помимо визуализации логов видели что-то примерно такое:

<img src="http://data.newprolab.com/public-newprolab-com/de_lab03_logs.png" align="center">

Внутри поля `message` какие-то непонятные символы. Само сообщение слабо структурированное. Непонятно как с этим работать. Неприятно, в общем. Давайте сделаем по-человечески.

*@Любопытный факт. На этот этап наша команда потратила пару недель. Пробовали разные вещи. Заново переосмысляли Kafka и ELK. Ниже вы увидете итоговый экстракт.*

Если коротко, то Divolte использует avro-схему для сериализации логов. Logstash об этой схеме ничего не знает. Ну и вообще, лучше изначально эту схему задать самим с нуля. Avro — это способ сериализовать данные в формате со встроенной схемой. Сериализованные данные представлены в компактном двоичном формате, который не требует генерации прокси-объектов или программного кода.

Зайдите в домашнюю директорию Divolte, в подпапку `conf`. Создайте там файл `eventrecord.avsc` со следующим содержимым:

```
{
  "type" : "record",
  "name" : "DefaultEventRecord",
  "namespace" : "io.divolte.record",
  "fields" : [ {
    "name" : "detectedDuplicate",
    "type" : "boolean"
  }, {
    "name" : "detectedCorruption",
    "type" : "boolean"
  }, {
    "name" : "firstInSession",
    "type" : "boolean"
  }, {
    "name" : "timestamp",
    "type" : "long"
  }, {
    "name" : "clientTimestamp",
    "type" : "long"
  }, {
    "name" : "remoteHost",
    "type" : "string"
  }, {
    "name" : "referer",
    "type" : [ "null", "string" ],
    "default" : null
  }, {
    "name" : "location",
    "type" : [ "null", "string" ],
    "default" : null
  }, {
    "name" : "viewportPixelWidth",
    "type" : [ "null", "int" ],
    "default" : null
  }, {
    "name" : "viewportPixelHeight",
    "type" : [ "null", "int" ],
    "default" : null
  }, {
    "name" : "screenPixelWidth",
    "type" : [ "null", "int" ],
    "default" : null
  }, {
    "name" : "screenPixelHeight",
    "type" : [ "null", "int" ],
    "default" : null
}, {
    "name" : "partyId",
    "type" : [ "null", "string" ],
    "default" : null
  }, {
    "name" : "sessionId",
    "type" : [ "null", "string" ],
    "default" : null
  }, {
    "name" : "pageViewId",
    "type" : [ "null", "string" ],
    "default" : null
  }, {
    "name" : "eventType",
    "type" : "string",
    "default" : "unknown"
  }, {
    "name" : "userAgentString",
    "type" : [ "null", "string" ],
    "default" : null
  }, {
    "name" : "userAgentName",
    "type" : [ "null", "string" ],
    "default" : null
  }, {
    "name" : "userAgentFamily",
    "type" : [ "null", "string" ],
    "default" : null
  }, {
    "name" : "userAgentVendor",
    "type" : [ "null", "string" ],
    "default" : null
  }, {
    "name" : "userAgentType",
    "type" : [ "null", "string" ],
    "default" : null
  }, {
    "name" : "userAgentVersion",
    "type" : [ "null", "string" ],
    "default" : null
  }, {
    "name" : "userAgentDeviceCategory",
    "type" : [ "null", "string" ],
    "default" : null
}, {
    "name" : "userAgentOsFamily",
    "type" : [ "null", "string" ],
    "default" : null
  }, {
    "name" : "userAgentOsVersion",
    "type" : [ "null", "string" ],
    "default" : null
  }, {
    "name" : "userAgentOsVendor",
    "type" : [ "null", "string" ],
    "default" : null
  } ]
}
```

Это дефолтная схема Divolte. Помимо самой схемы, нужно еще создать groovy-файл, который будет маппить события на странице в эту схему. Подробнее можете прочитать в [документации Divolte](http://divolte-releases.s3-website-eu-west-1.amazonaws.com/divolte-collector/0.5.0/userdoc/html/index.html).

Создайте в той же подпапке файл `mapping.groovy` со следующим содержимым:

```
mapping {
  map duplicate() onto 'detectedDuplicate'
  map corrupt() onto 'detectedCorruption'
  map firstInSession() onto 'firstInSession'
  map timestamp() onto 'timestamp'
  map clientTimestamp() onto 'clientTimestamp'
  map remoteHost() onto 'remoteHost'
  map referer() onto 'referer'
  map location() onto 'location'
  map viewportPixelWidth() onto 'viewportPixelWidth'
  map viewportPixelHeight() onto 'viewportPixelHeight'
  map screenPixelWidth() onto 'screenPixelWidth'
  map screenPixelHeight() onto 'screenPixelHeight'
  map partyId() onto 'partyId'
  map sessionId() onto 'sessionId'
  map pageViewId() onto 'pageViewId'
  map eventType() onto 'eventType'
  map userAgentString() onto 'userAgentString'
  map userAgent().name() onto 'userAgentName'
  map userAgent().family() onto 'userAgentFamily'
  map userAgent().vendor() onto 'userAgentVendor'
  map userAgent().type() onto 'userAgentType'
  map userAgent().version() onto 'userAgentVersion'
  map userAgent().deviceCategory() onto 'userAgentDeviceCategory'
  map userAgent().osFamily() onto 'userAgentOsFamily'
  map userAgent().osVersion() onto 'userAgentOsVersion'
  map userAgent().osVendor() onto 'userAgentOsVendor'
}
```

Теперь эти два файла надо прописать в конфигах Divolte и Logstash.

В конфиг Divolte надо добавить следующее:

```
mappings {
    my_mapping = {
      schema_file = "/home/ubuntu/divolte-collector-0.8.0/conf/eventrecord.avsc"
      mapping_script_file = "/home/ubuntu/divolte-collector-0.8.0/conf/mapping.groovy"
      sources = [browser]
      sinks = [hdfs, kafka]
    }
  }
```

Группа параметров `mappings` находится на том же уровне, что и `global`, например.

В конфиг Logstash добавляем следующее:

```
input {
    kafka {
        codec => avro {
            schema_uri => '/home/ubuntu/divolte-collector-0.8.0/conf/eventrecord.avsc'
            tag_on_failure => 'true'
            encode_as_base64 => 'false'
        }
        bootstrap_servers => '0.0.0.0:6667'
        topics => ['divolte3']
        key_deserializer_class => "org.apache.kafka.common.serialization.ByteArrayDeserializer"
        value_deserializer_class => "org.apache.kafka.common.serialization.ByteArrayDeserializer"
    }
}
```

Последние две строчки в конфиге — это прямо та боль, с которой мы пытались долгое время бороться. По дефолту плагин не понимал то, что присылал Divolte в Кафку. Присылал какие-то непонятные ошибки в логи, ссылаясь на какие-то отрицательные значения длины сообщения или еще что похуже. Проблема в том, что десериализатор по умолчанию для Kafka был десериализатор строк. До 5-й версии Logstash такого не было, поэтому все решения в интернете были для старых версий и не подходили нам.

И последнее, что осталось — это поставить [плагин для Logstash с avro-кодеком](https://www.elastic.co/guide/en/logstash/current/plugins-codecs-avro.html).

```
$ cd /usr/share/logstash/bin
$ ./logstash-plugin install logstash-codec-avro
```

И на всякий случай сделать рестарт:

```
$ sudo -i service logstash restart
```

Рекомендуем еще смотреть в логи, чтобы понять, есть ли какие-то ошибки:

```
$ tail -f /var/log/logstash/logstash-plain.log
```

В итоге вы должны в Kibana увидеть что-то похожее:

<img src="http://data.newprolab.com/public-newprolab-com/de_lab03_logs2.png">

Выглядит хорошо. С этим можно работать.

### 2. Установка Airflow

Airflow — это набор библиотек для разработки, планирования и мониторинга рабочих процессов. Он больше сопоставим с Oozie или Azkaban. Главная особенность Airflow  заключается в разработке процессов, используя язык Python. Следовательно, можно достаточно просто организовать ваш Python-проект, учитывая вашу инфраструктуру, размер команды и так далее. 

Как вы уже знаете, Airflow написан на языке Python, поэтому для начала проверим python-окружение. В примере будет использоваться Python 3, но Airflow поддерживает и Python 2 в том числе. Также для того, чтобы было все по правилам, будет использоваться virtualenv.

```
$ python3 --version
Python 3.5.2
$ virtualenv --version
15.1.0
```

Версия питона не самая свежая, поэтому давайте сделаем апгрейд.

```
$ sudo add-apt-repository ppa:jonathonf/python-3.6
$ sudo apt-get update
$ sudo apt-get install python3.6
```

Проверим, что теперь всё ок.

```
$ python3 --version
Python 3.5.2
```

Python3 по-прежнему ссылается на старую версию. Давайте это исправим:

```
$ cd /usr/bin/
$ sudo unlink python3
$ sudo ln -s python3.6 python3
$ readlink python3
python3.6
$ python3 --version
Python 3.6.3
```

Самостоятельно поставьте virtualenv, если еще не сделали этого.

Далее, создадим рабочий каталог для Airflow и внутри него virtualenv каталог Python 3.

```
$ cd /path/to/my/airflow/workspace
$ virtualenv -p which python3 venv
$ source venv/bin/activate
(venv) $
```

Установим Airflow 1.8:

```
(venv) $ pip install airflow==1.8.0
```

Может с первого раза не взлететь. Может потребоваться поставить `python3.6-dev`:

```
$ sudo apt-get update
$ sudo apt-get install python3.6-dev
```

Теперь нам нужно создать каталог `AIRFLOW_HOME`, где  будут располагаться DAG-файлы и плагины Airflow. После создания каталога установим переменную среды AIRFLOW_HOME.

```
(venv) $ cd /path/to/my/airflow/workspace
(venv) $ mkdir airflow_home
(venv) $ export AIRFLOW_HOME=`pwd`/airflow_home
```

Проверим установился ли Airflow:

```
$ airflow version
[2017-11-26 19:38:19,607] {__init__.py:57} INFO - Using executor SequentialExecutor
[2017-11-26 19:38:19,745] {driver.py:123} INFO - Generating grammar tables from /usr/lib/python3.6/lib2to3/Grammar.txt
[2017-11-26 19:38:19,771] {driver.py:123} INFO - Generating grammar tables from /usr/lib/python3.6/lib2to3/PatternGrammar.txt
  ____________       _____________
 ____    |__( )_________  __/__  /________      __
____  /| |_  /__  ___/_  /_ __  /_  __ \_ | /| / /
___  ___ |  / _  /   _  __/ _  / / /_/ /_ |/ |/ /
 _/_/  |_/_/  /_/    /_/    /_/  \____/____/|__/
   v1.8.0
```

Если команда отработала , то Airflow создал свой конфигурационный файл `airflow.cfg` в `AIRFLOW_HOME`:

```
$ tree
.
├── airflow.cfg
└── unittests.cfg
```

Следующий шаг — выполнить команду, которая будет создавать и инициализировать базу данных потока данных в SQLite:

```
(venv) $ airflow initdb
```

База данных будет создана в `airflow.db` по умолчанию.

Пользовательский интерфейс airflow осуществляется в виде веб-приложения. Запустить его можно, выполнив команду:

```
(venv) $ airflow webserver --port 8081
```

Теперь вы можете посетить пользовательский интерфейс в браузере на порту 8081 на хосте, где Airflow был начат, например: `<your-ip:8081>`

### 3. Токенизация URL и сохранение в Parquet

Вам самостоятельно нужно:

1. Сделать токенизацию поля `location` в ваших логах. 
2. Сохранять последнюю часть этого поля, `timestamp`, `session_id` в Parquet.
3. При помощи Airflow каждый час делать эту токенизацию, каждый день сохранять результат в Parquet.
4. Сохранять эти данные в папку на HDFS: новый день — новый файл.

### 4. Ссылки для изучения

* [Документация Airflow](https://airflow.apache.org/)
### 5. Проверка осуществляется из Личного кабинета.  

Для этого чекер будет подключаться к Airflow (порт 8081) и к HDFS через Rest API (порт 50070, возможно, 50075). В конечном итоге, файлы должны иметь три поля: `timestamp`, `session_id` и `location`. Сами файлы должны располагаться на HDFS в папке results(не забудьте её создать).

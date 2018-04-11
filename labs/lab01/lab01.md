## Лаба 1. **Поднять свой сайт и организовать сбор кликстрима в Kafka**

<img src="https://kafka.apache.org/images/logo.png" width="300px" align="center">  

### 1. Развертывание своего сайта

Скачайте архив со статическим сайтом по <a href="http://data.newprolab.com/data-engineer/website.zip">этой ссылке</a> себе на сервер со статическим IP. Разархивируйте его по пути `/var/www/dataengineer/`.

Следующий шаг — установка nginx.

Выполните следующие команды:

```bash
$ sudo apt-get install -y python-software-properties software-properties-common
$ sudo add-apt-repository -y ppa:nginx/stable
$ sudo apt-get update
$ sudo apt-get install nginx
```

Чтобы ваш сайт поднялся и был доступен из браузера, необходимо создать следующий конфиг в `/etc/nginx/sites-enabled/default`.

```
server {
     listen 80 default_server;
     listen [::]:80;
     server_name _;
     root /var/www/dataengineer;
          location / {
                  index index.html;
                  alias /var/www/dataengineer/skyeng.ru/;
                  default_type text/html;
          }
          location /tracking/ {
                proxy_pass http://localhost:8290/tracking/;
          }
}
```

Теперь в браузере попробуйте набрать ваш ip в браузерной строке, и вы должны будете попасть на свою копию сайта.

### 2. Установка и конфигурирование Divolte

Отлично, копия сайта у нас поднята, кластер развернут. Теперь как-то надо организовать сбор кликстрима с этого сайта на наш кластер. Для этой задачи предлагаем воспользоваться инструментом <a href="http://divolte.io/">Divolte</a>, который позволяет довольно удобно собирать клики и сохранять их в HDFS или отправлять в Kafka. Мы попробуем оба варианта.

Перед установкой этого инструмента нам потребуется установить Java 8-й версии.

На всякий случай проверим, что ее у нас, действительно, нет. 

```bash
$ java -version
```

Если вы видите, что-то подобное, то это значит, что ее у вас нет:

> The program 'java' can be found in the following packages:
>
> * default-jre
>
> * gcj-4.8-jre-headless
>
> * openjdk-7-jre-headless
>
> * gcj-4.6-jre-headless
>
> * openjdk-6-jre-headless
>
>   Try: sudo apt-get install <selected package>

Для установки Java воспользуйтесь следующими командами:

```bash
$ sudo apt-get install python-software-properties
$ sudo add-apt-repository ppa:webupd8team/java
$ sudo apt-get update
$ sudo apt-get install oracle-java8-installer
```

Далее добавим путь к Java в окружение:

```bash
$ sudo nano /etc/environment
```

Там нужно вставить следующую строчку `JAVA_HOME="/usr/lib/jvm/java-8-oracle"` и сохранить файл.

Далее:

```bash
$ source /etc/environment
$ echo $JAVA_HOME
```

Результат должен быть таким:

> /usr/lib/jvm/java-8-oracle

Еще раз проверим:

```bash
$ java -version
```

> java version "1.8.0_151"
> Java(TM) SE Runtime Environment (build 1.8.0_151-b12)
> Java HotSpot(TM) 64-Bit Server VM (build 25.151-b12, mixed mode)

Теперь можем переходить непосредственно к **Divolte**.

Возьмите актуальную версию этого инструмента <a href="http://divolte.io/">отсюда</a> и скачайте себе на мастер.

Далее:

```bash
$ tar -xzf divolte-collector-*.tar.gz
$ cd divolte-collector-*
$ touch conf/divolte-collector.conf
```

Перейдите в папку `conf`. Переименуйте файл `divolte-env.sh.example` в `divolte-env.sh`. Отредактируйте его, добавив туда:

```HADOOP_CONF_DIR=/usr/hdp/2.6.2.0-205/hadoop/conf```

Теперь очередь `divolte-collector.conf`. Туда добавьте следующее:

```
divolte {
  global {
    hdfs {
      client {
        fs.defaultFS = "hdfs://node1.c.data-engineer-173012.internal:8020"
      }
      // Enable HDFS sinks.
      enabled = true

      // Use multiple threads to write to HDFS.
      threads = 2
    }
  }

  sinks {
    // The name of the sink. (It's referred to by the mapping.)
    hdfs {
      type = hdfs

      // For HDFS sinks we can control how the files are created.
      file_strategy {
        // Create a new file every hour
        roll_every = 1 hour

        // Perform a hsync call on the HDFS files after every 1000 records are written
        // or every 5 seconds, whichever happens first.

        // Performing a hsync call periodically can prevent data loss in the case of
        // some failure scenarios.
        sync_file_after_records = 1000
        sync_file_after_duration = 5 seconds

        // Files that are being written will be created in a working directory.
        // Once a file is closed, Divolte Collector will move the file to the
        // publish directory. The working and publish directories are allowed
        // to be the same, but this is not recommended.
        working_dir = "/divolte/inflight"
        publish_dir = "/divolte/published"
      }

      // Set the replication factor for created files.
      replication = 3
    }
  }

  sources {
    a_source {
      type = browser
      prefix = /tracking
    }
  }
}
```

Этот конфиг позволит вам сохранять кликстрим на HDFS. Обратите внимание, что в `fs.defaultFS` вам нужно добавить FQDN вашего сервера.

Чтобы всё заработало, надо сделать две вещи. Первая — это создать на HDFS две папки, которые мы указали в конфиге в `working_dir` и `publish_dir`. Для этого перейдите под юзера `hdfs`:

```bash
$ sudo su hdfs
$ hdfs dfs -mkdir /divolte
$ hdfs dfs -mkdir /divolte/inflight
$ hdfs dfs -mkdir /divolte/published
```

Поменяем права на директорию `divolte`, чтобы был доступ к записи у других юзеров:

```bash
$ hdfs dfs -chmod -R 0777 /divolte
```

Вторая вещь — это добавить скрипт на все страницы вашей копии сайта. Скрипт выглядит следующим образом:

```html
<script type="text/javascript" src="/tracking/divolte.js" defer async></script>
```

Один из способов — это воспользоваться `sed`. Например, такой командой можно добавить скрипт в низ страницы `index.html`:

```bash
sed -i 's#</body>#<script type="text/javascript" src="/tracking/divolte.js" defer async></script> \n</body>#g' index.html
```

**Важно! Подумайте, как это распространить на все страницы.**

Просто поставить `*` не сильно поможет, потому что внутри директории с сайтом существуют поддиректории, и sed будет на них ругаться. Подробнее про sed можно почитать <a href="https://habrahabr.ru/company/ruvds/blog/327530/">здесь</a>. Или придумайте свой способ с нуля.

Как только решите эту задачу, можете запустить `divolte`:

```bash
ubuntu@node1:~/divolte-collector-0.6.0$ ./bin/divolte-collector
```

Увидеть вы должны что-то такое:

> ubuntu@node1:~/divolte-collector-0.5.0/bin$ ./divolte-collector 
>
> 2017-07-12 15:12:29.463Z [main] INFO  [Version]: HV000001: Hibernate Validator 5.4.1.Final
>
> 2017-07-12 15:12:29.701Z [main] INFO  [SchemaRegistry]: Using builtin default Avro schema.
>
> 2017-07-12 15:12:29.852Z [main] INFO  [SchemaRegistry]: Loaded schemas used for mappings: [default]
>
> 2017-07-12 15:12:29.854Z [main] INFO  [SchemaRegistry]: Inferred schemas used for sinks: [hdfs]
>
> 2017-07-12 15:12:30.112Z [main] WARN  [NativeCodeLoader]: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
>
> 2017-07-12 15:12:31.431Z [main] INFO  [Server]: Initialized sinks: [hdfs]
>
> 2017-07-12 15:12:31.551Z [main] INFO  [Mapping]: Using built in default schema mapping.
>
> 2017-07-12 15:12:31.663Z [main] INFO  [UserAgentParserAndCache]: Using non-updating (resource module based) user agent parser.
>
> 2017-07-12 15:12:32.262Z [main] INFO  [UserAgentParserAndCache]: User agent parser data version: 20141024-01
>
> 2017-07-12 15:12:37.363Z [main] INFO  [Slf4jErrorManager]: 0 error(s), 0 warning(s), 87.60016870518768% typed
>
> 2017-07-12 15:12:37.363Z [main] INFO  [JavaScriptResource]: Pre-compiled JavaScript source: divolte.js
>
> 2017-07-12 15:12:37.452Z [main] INFO  [GzippableHttpBody]: Compressed resource: 9828 -> 4401
>
> 2017-07-12 15:12:37.592Z [main] INFO  [BrowserSource]: Registered source[a_source] script location: /tracking/divolte.js
>
> 2017-07-12 15:12:37.592Z [main] INFO  [BrowserSource]: Registered source[a_source] event handler: /tracking/csc-event
>
> 2017-07-12 15:12:37.592Z [main] INFO  [Server]: Initialized sources: [a_source]
>
> 2017-07-12 15:12:37.779Z [main] INFO  [Server]: Starting server on 0.0.0.0:8290
>
> 2017-07-12 15:12:37.867Z [main] INFO  [xnio]: XNIO version 3.3.6.Final
>
> 2017-07-12 15:12:37.971Z [main] INFO  [nio]: XNIO NIO Implementation Version 3.3.6.Final

Если нет и появляется ошибка про JAVA, то перезайдите на машину.

Зайдите теперь на свой сайт и покликайте, попереходите на разные страницы. Вернитесь в терминал и нажмите Ctrl+C. Теперь посмотрите, появилось ли что-то в папке `/divolte/published` на HDFS. Если да, значит у вас всё работает, и вы научились собирать кликстрим!

### 3. Задача

Ваша задача теперь — сделать так, чтобы он собирался не на HDFS, а в Kafka.

### 4. Ссылки для изучения

* Предыстория возникновения Kafka - https://www.confluent.io/blog/stream-data-platform-1 
* Туториал по Kafka - https://www.tutorialspoint.com/apache_kafka/apache_kafka_basic_operations.htm 
* Документация Divolte - http://divolte-releases.s3-website-eu-west-1.amazonaws.com/divolte-collector/0.5.0/userdoc/html/index.html

### 5. Проверка

Проверка осуществляется из [Личного кабинета](http://lk.newprolab.com/) , путем  внешнего подключения консьюмера к вашему хосту на 6667 порте. Веб-сайт должен быть развернут на ноде, хост которого вы ранее отправляли координатору. Топик должен совпадать с логином ЛК. 

Для удаленного подключения к брокеру зайдите в веб-интерфейс управления кластером Ambari, в раздел конфигурирования Кафки. Проверьте, стоит ли в поле *listeners* значение `PLAINTEXT://localhost:6667`. После чего добавьте в разделе *Custom kafka-broker* новое поле *advertised.listeners* и укажите ваш хост в таком виде `PLAINTEXT://*ваш хост*:6667`


## Лаба 5. Real-time пайплайн с Spark Streaming и Grafana. Решение.

Следующее задание зааключается в  построении real-time решения. В данном случае мы будем использовать связку Prometheus + Grafana. 

Для начала сконфигурируем Spark Streaming, чтобы в условиях нашего кластера он не падал.

```Bash
#!/bin/bash
num_executors=4
executor_memory=3g
executor_cores=3
receiver_max_rate=100
receiver_initial_rate=30
spark-submit --master yarn --deploy-mode cluster \
  --name <my-job-name> \
  --class <main-class> \
  --driver-memory 2g \
  --num-executors ${num_executors} --executor-cores ${executor_cores} --executor-memory ${executor_memory} \
  --queue <realtime_queue> \
  --files <hdfs:///path/to/log4j-yarn.properties> \
  --conf spark.driver.extraJavaOptions=-Dlog4j.configuration=log4j-yarn.properties \
  --conf spark.executor.extraJavaOptions=-Dlog4j.configuration=log4j-yarn.properties \
  --conf spark.serializer=org.apache.spark.serializer.KryoSerializer `# Kryo Serializer is much faster than the default Java Serializer` \
  --conf spark.locality.wait=10 `# Increase job parallelity by reducing Spark Delay Scheduling (potentially big performance impact (!)) (Default: 3s)` \
  --conf spark.task.maxFailures=8 `# Increase max task failures before failing job (Default: 4)` \
  --conf spark.ui.killEnabled=false `# Prevent killing of stages and corresponding jobs from the Spark UI` \
  --conf spark.logConf=true `# Log Spark Configuration in driver log for troubleshooting` \
`# SPARK STREAMING CONFIGURATION` \
  --conf spark.streaming.blockInterval=200 `# [Optional] Tweak to balance data processing parallelism vs. task scheduling overhead (Default: 200ms)` \
  --conf spark.streaming.receiver.writeAheadLog.enable=true `# Prevent data loss on driver recovery` \
  --conf spark.streaming.backpressure.enabled=true \
  --conf spark.streaming.backpressure.pid.minRate=10 `# [Optional] Reduce min rate of PID-based backpressure implementation (Default: 100)` \
  --conf spark.streaming.receiver.maxRate=${receiver_max_rate} `# [Spark 1.x]: Workaround for missing initial rate (Default: not set)` \
  --conf spark.streaming.kafka.maxRatePerPartition=${receiver_max_rate} `# [Spark 1.x]: Corresponding max rate setting for Direct Kafka Streaming (Default: not set)` \
  --conf spark.streaming.backpressure.initialRate=${receiver_initial_rate} `# [Spark 2.x]: Initial rate before backpressure kicks in (Default: not set)` \
`# YARN CONFIGURATION` \
  --conf spark.yarn.driver.memoryOverhead=512 `# [Optional] Set if --driver-memory < 5GB` \
  --conf spark.yarn.executor.memoryOverhead=1024 `# [Optional] Set if --executor-memory < 10GB` \
  --conf spark.yarn.maxAppAttempts=4 `# Increase max application master attempts (needs to be <= yarn.resourcemanager.am.max-attempts in YARN, which defaults to 2) (Default: yarn.resourcemanager.am.max-attempts)` \
  --conf spark.yarn.am.attemptFailuresValidityInterval=1h `# Attempt counter considers only the last hour (Default: (none))` \
  --conf spark.yarn.max.executor.failures=$((8 * ${num_executors})) `# Increase max executor failures (Default: max(numExecutors * 2, 3))` \
  --conf spark.yarn.executor.failuresValidityInterval=1h `# Executor failure counter considers only the last hour` \
  </path/to/spark-application.jar>
```

Теперь установим займемся установкой **prometheus** 

```
sudo useradd --no-create-home --shell /bin/false prometheus
sudo useradd --no-create-home --shell /bin/false node_exporter
```

Создадим каталоги для хранения данных и данных.

```
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
```

Теперь зададим права пользователя и группы на новые каталоги для пользователя.

```
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

Скачаем Prometheus 

```
cd ~
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.0.0/prometheus-2.0.0.linux-amd64.tar.gz
```

Затем используйте `sha256sum `команду для создания контрольной суммы загруженного файла:

```
sha256sum prometheus-2.0.0.linux-amd64.tar.gz
```

Сравните результат этой команды с контрольной суммой на странице загрузки Prometheus, чтобы убедиться, что ваш файл является подлинным и не поврежден.

```
Output
e12917b25b32980daee0e9cf879d9ec197e2893924bd1574604eb0f550034d46  prometheus-2.0.0.linux-amd64.tar.gz
```

Теперь распакуйте загруженный архив.

```
tar xvf prometheus-2.0.0.linux-amd64.tar.gz
```

Скопируйте два бинарных файла в `/usr/local/bin`каталог.

```
sudo cp prometheus-2.0.0.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.0.0.linux-amd64/promtool /usr/local/bin/
```

Установите права пользователя и группы на файлы для пользователя `prometheus`

```
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```

Скопируйте файлы `consoles`и `console_libraries`каталоги `/etc/prometheus`.

```
sudo cp -r prometheus-2.0.0.linux-amd64/consoles /etc/prometheus
sudo cp -r prometheus-2.0.0.linux-amd64/console_libraries /etc/prometheus
```

Установите права пользователя и группы на каталоги для пользователя **prometheus** . Использование `-R`флага гарантирует, что право собственности также установлено на файлы внутри каталога.

```
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```

В `/etc/prometheus`директории используйте `nano`или ваш любимый текстовый редактор для создания файла конфигурации с именем `prometheus.yml`. Пока этот файл будет содержать достаточно информации для запуска Prometheus в первый раз.

```
sudo nano /etc/prometheus/prometheus.yml
```

Файл конфигурации Prometheus `/etc/prometheus/prometheus.yml`

```
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
alerting:
  alertmanagers:
  - static_configs:
    - targets: []
    scheme: http
    timeout: 10s
scrape_configs:
- job_name: prometheus
  scrape_interval: 15s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - localhost:9090
- job_name: python_app
  scrape_interval: 15s
  scrape_timeout: 10s
  metrics_path: /
  scheme: http
  static_configs:
  - targets:
    - localhost:8000
```

Теперь задайте права пользователя и группы в файле конфигурации пользователю 

```
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

Запустите Prometheus в качестве пользователя **prometheus** , указав путь как к файлу конфигурации, так и к каталогу данных.

```
sudo -u prometheus /usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries
```

Теперь поставим клиент для Python

```
pip install prometheus_client
```

Теперь напишем скрипт, который в режиме реального времени будет складывать данные в  `prometheus`

```python
import asyncio
import prometheus_client as prom
import logging
import random
format = "%(asctime)s - %(levelname)s [%(name)s] %(threadName)s %(message)s"
logging.basicConfig(level=logging.INFO, format=format)
g1 = prom.Gauge('compute_gauge_rate', 'Random gauge', labelnames=['task_name'])
async def compute_rate(name, rate, delta_min=-100, delta_max=100):
    """Increases or decreases a rate based on a random delta value
    which varies from "delta_min" to "delta_max".
    :name: task_id
    :rate: initial rate value
    :delta_min: lowest delta variation
    :delta_max: highest delta variation
    """
    while True:
        logging.info("name: {} value {}".format(name, rate))
        g1.labels(task_name=name).set(rate)
        rate += random.randint(delta_min, delta_max)
        await asyncio.sleep(1)
import sys
from pyspark import SparkContext, SparkConf
from pyspark.streaming import StreamingContext
from pyspark.streaming.kafka import KafkaUtils
from uuid import uuid1
if __name__ == “__main__”:
    sc = SparkContext(appName=”PythonStreamingRecieverKafkaWordCount”)
    ssc = StreamingContext(sc, 2) 
    broker, topic = sys.argv[1:]
    kvs = KafkaUtils.createStream(ssc, \
                                  broker, \
                                  “raw-event-streaming-consumer”,\{topic:1}) 
    lines = kvs.map(lambda x: x[1])
    top_cnts =20
    counts = lines.flatMap(lambda x: str(x[0]).split(",")).map(lambda word: (word, 1)).reduceByKey(lambda a,b: a+b).takeOrdered(top_cnts, lambda x: =x[1])        
    counts.pprint()
    ssc.start()
    ssc.awaitTermination()
```

Открываем Grafana, выбираем источник данных и строим нужный дашборд.  Запускаем чекер и проверяем. 
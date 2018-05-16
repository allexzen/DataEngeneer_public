

### Лаба 4. Запуск джобы Spark по расписанию и сохранение результата в ClickHouse

Если вы верно прошли GIST по установке новой версии спарка, то в итоге у вас должен отработать следующий скрипт.

```Python
import  os
import  sys
os.environ["PYSPARK_SUBMIT_ARGS"]='--packages com.databricks:spark-csv_2.10:1.2.0 pyspark-shell --num-executors 5'
os.environ["PYSPARK_PYTHON"]='python3'
#os.environ["SPARK_HOME"]='/usr/hdp/current/spark2-client'
spark_home = os.environ.get('SPARK_HOME', None)
if not spark_home:
    raise ValueError('SPARK_HOME environment variable is not set')
sys.path.insert(0, os.path.join(spark_home, 'python'))
sys.path.insert(0, os.path.join(spark_home, 'python/lib/py4j-0.10.6-src.zip'))
os.environ["PYSPARK_PYTHON"] = 'python3'
exec(open(os.path.join(spark_home, 'python/pyspark/shell.py')).read())
```

```
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 2.3.0
      /_/

Using Python version 3.6.4 (default, May 3 2018 18:10:19)
SparkSession available as 'spark'.
```

Если он у вас нормально отрабатывает в jupyter, то можно смело приступать к выполнению лабы. Задача делится на на три части: 

1. Взять паркет-файл с hdfs
2. Выполнить вордкаунт 
3. Положить в кликхаус

Предварительно необходимо создать бд и таблицу в клихаусе. Для этого можно воспользоваться http интерфейсом. В <a href="https://clickhouse.yandex/docs/ru/interfaces/http_interface/">русскоязычной документации </a>можно легко найти, как это сделать. 

```python
from datetime import datetime
from airflow import DAG
from aitflow.operators.python_operator import PythonOperator

import logging
log = logging.getLogger(_name_)

import subprocess
import json
import os 
import sys
import py4j

spark_home = os.environ.get('SPARK_HOME', None)
if not spark_home:
    raise ValueError('SPARK_HOME enviroment variable is not set')
from pyspark import SparkConf
from pyspark.context import SparkContext
from pyspark.sql import SparkSession, SQLContext

#vars
top_cnts =20
default_args = {
    'owner' : 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2018,5,4),
    'email': ['airflow@airflow.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1
}
dag =DAG('DAG_SPARK', description-'SPARK to clickhouse',
        default_args=default_args,
        schedule_interval='5 5 * * *')
HOST = 'http://localhost:8123/'
import requests

def get_clickhouse_data(query, host = HOST, connection_timeout = 1500):
    NUMBER_OF_TRIES = 30
    DELAY = 10
    
    for i in range(NUMBER_OF_TRIES):
        r = requests.post(host, params = {'timeout_before_checking_execution_speed': 120, 'max_execution_time': 6000}
                          , timeout = connection_timeout, data = query)
        if r.status_code == 200:
            return r.text
        else:
            print('ATTENTION: try #%d failed' % i)
            if i != (NUMBER_OF_TRIES-1):
                print(query)
                print(r.text)
                time.sleep(DELAY*(i+1))
            else:
                raise ValueError(r.text)

def upload(table, content, host=HOST):
    '''Uploads data to table in ClickHous'''
    content = content.encode('utf-8')
    query_dict = {
             'query': 'INSERT INTO ' + table + ' FORMAT TabSeparatedWithNames '
        }
    r = requests.post(host, data=content, params=query_dict)
    result = r.text
    if r.status_code == 200:
        return result
    else:
        raise ValueError(r.text)
        
def pre_data_spark4clickhose():
    SparkContext._ensure_initialized()
    conf = SparkConf()
    spark = SparkSession.builder.get0rCreate()
    sc = spark.sparkContext  	    
    pq_file_name='/results/'+datetime.strftime(datetime.now(), "%Y_%m_%d.")+'.parquet'
	log.info('SparkSession: %s', str(spark))
    log.info('sparkContext: %s', str(sc))
    df_elk_data = spark.read.parquet(pq_file_name)
    rdd_elk_data = df_elk_data.select(df_elk_data.location).rdd
    count=rdd_elk_data.flatMap(lambda x: str(x[0]).split(" ")).map(lambda word: (word, 1)).reduceByKey(lambda a,b: a+b).takeOrdered(top_cnts, lambda x: =x[1])
    sc.stop()
    upload('login', count, HOST)
    
    
spark4clickhose_batch_operator = PythonOperator(task_id='spark4clickhose_batch_task', provide_contex=True, python_callable=pre_data_spark4clickhose)
```

Следущий код выполняет поставленные задачи. Стоит заметить, что вордкаунт выполнен с применением rdd.  Однако не стоит зацикливаться именно на нём. Существует множетство подходов. Осталось это зарегистрировать в airflow, как в предыдущей лабе и заняться подключением superset.

Если суперсет не входит в ваш набор hdp, то можно установить его либо через <a href="https://hub.docker.com/r/amancevice/superset/">докер</a>, либо <a href="https://superset.apache.org/installation.html">классическим способом</a>. Здесь будет приведен классический способ. 

```
sudo apt-get install build-essential libssl-dev libffi-dev python-dev python-pip libsasl2-dev libldap2-dev
```

 ```
virtualenv venv
 ```

```bash
pip install --upgrade setuptools pip
```

```bash
# Install superset
pip install superset

# Create an admin user (you will be prompted to set username, first and last name before setting a password)
fabmanager create-admin --app superset

# Initialize the database
superset db upgrade

# Load some data to play with
superset load_examples

# Create default roles and permissions
superset init

# To start a development web server on port 8088, use -p to bind to another port
# superset runserver -d
```

По умолчанию суперсет запускается на 8088 порту.  Ткаже чекер смотрит и через этот же порт. 

Чтобы подключить кликхаус, то нужно поставить диалект для SQLAlchemy, через который суперсет и будет показывать данные. Инструкция по установке находится  <a href="https://github.com/cloudflare/sqlalchemy-clickhouse">здесь</a>. В докере он установлен по умолчанию. Теперь  указываем URI в следующем формате: 

```
clickhouse://username:password@hostname:port/database
```

<img src="img/8.png">

После успешного подключения можно смело писать SQL запросы и строить красивые дашборды.  Например, вот так. 

<img src="img/7.png">



Теперь можно запускать чекер. Лаба решена!
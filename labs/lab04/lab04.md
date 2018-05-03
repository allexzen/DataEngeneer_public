

## Лаба 4. Запуск джобы Spark по расписанию и сохранение результата в ClickHouse

 <img src="http://data.newprolab.com/public-newprolab-com/de_lab04_spark.png" width="170px" align="center"> <img src="https://habrastorage.org/files/d9b/066/e61/d9b066e61e1f480a977d889dc03ded99.png" width="300px" align="center"> <img src="http://data.newprolab.com/public-newprolab-com/de_lab04_superset.png" width="170px" align="center">

Итак, у вас на HDFS есть набор файлов в формате Parquet, имеющих следующую структуру:

`session_id`		 	`timestamp` 			`location (последняя часть поля)`

Каждый файл — это данные за один день сбора логов. Эти файлы создаются автоматически каждый день при помощи планировщика.

Теперь мы из этих предобработанных файлов будем делать небольшую аналитику, используя Apache Spark, ClickHouse и Superset. 

Указанные инструменты, за исключением КликХауса, уже есть в HDP. Но может так получится, что с этими версиями не будет всё запускаться. В этот раз установка новых версий будет уже тоже частью вашей самостоятельной работы (хотя по установке новой версии Spark можете обратиться в Gist, указанный в ссылках ниже).

### 1. Задача

В конце каждого дня вам нужно запустить спарковскую джобу, которая будет:

1. Подгружать parquet-файл за текущий день.
2. Брать остаток вашего `location` и разбивать его на отдельные слова.
3. Делать вордкаунт (подсчитывать количество возникновений того или иного слова в файле) с учетом стоп-слов (какой набор из стоп-слов выбрать — на ваше усмотрение).
4. Сохранять топ-20 слов за день в ClickHouse.

В конце недели построить визуализацию в Superset того, что происходило за неделю (способ визуализации выберите сами).

На этом ваш batch-пайплайн будет завершен. Вы прошли все этапы от сбора кликов с сайта до создания визуализации в BI-инструменте. Поздравляем!

### 2. Установка ClickHouse

1.  Установите jupyter notebook в вашем virtual env:

   ```bash
   $ source venv/venv/bin/activate
   $ pip install notebook
   $ jupyter notebook --generate-config
   ```

2. Замените в файле `~/..jupyter/jupyter_notebook_config.py` параметр `c.NotebookApp.ip` с `localhost` на `0.0.0.0`, предварительно раскомментив его.

3. Поставьте докер, если он ещё не установлен

   ```bash
   $ sudo apt install docker.io
   $ sudo usermod -a -G docker $USER
   ```

ClickHouse можно установить на Linux (подробно про установку написано в документации <https://clickhouse.yandex/>).

На Windows или Mac можно запустить ClickHouse под Docker'ом: первая команда поднимает clickhouse-server на порту 8123, а вторая - позволяет подключиться к консольному ClickHouse клиенту.

`docker run -d --name clickhouse-server --publish=8123:8123 --publish=9000:9000 yandex/clickhouse-serverdocker run -it --rm --link clickhouse-server:9000 yandex/clickhouse-client --host clickhouse-server`

### 3. Ссылки для изучения

* [Документация Spark](https://spark.apache.org/docs/latest/)
* [Gist с параметрами конфигурации Spark, использовавшихся при конфигурировании кластеров на наших программах](https://gist.github.com/shtuder/ccc71ee761b27a0cd36f9031f434257e) 
* [Документация ClickHouse](https://clickhouse.yandex/)
* [Документация Superset](https://superset.incubator.apache.org/)

### 4. Проверка

Проверка будет осуществляться из личного кабинета. Чекер будет подключаться через HTTP-интерфейс кликхауса на порту 8123.

1) Название БД - `dataengineer`
2) Название таблицы - `ваш логин` (например - de.ilya.markin)
3) Названия полей - `date, word, count` (укладываем сортированными по количеству в рамках каждого дня)

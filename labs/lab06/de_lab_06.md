## Лаба 6. Мониторинг и тестирование пайплайна

 <img src="http://data.newprolab.com/public-newprolab-com/de_lab06_kibana.png" width="170px" align="center"> <img src="http://data.newprolab.com/public-newprolab-com/de_lab05_grafana.svg" width="170px" align="center"> 

### 1. Мониторинг пайплайна

Для нужд мониторинга вашего пайплайна у вас есть два инструмента: kibana — для мониторинга Elastic + Logstash, Grafana — для всего остального (например, скорость чтения с диска, очередь в Kafka и др.).

В качестве напоминания: б*о*льшую часть данных в Grafana сама себя не напишет. Вам нужен отдельный инструмент для сбора этих данных: Graphit или Prometeus.

У вас должно получиться что-то подобное (набор метрик на картинке случаен):

<img src="http://data.newprolab.com/public-newprolab-com/de_lab06_grafana_kafka.png">

<p align="center">*Мониторинг Kafka*</p>

<img src="http://data.newprolab.com/public-newprolab-com/de_lab06_grafana_elastic.png">

<p align="center">*Мониторинг Elasticsearch*</p>

### 2. Стресс-тест пайплайна

Во время проверки чекера мы натравим на ваш сайт нашу пушку, которая будет генерировать клики со скоростью более 2000 кликов в секунду в течение 3-х минут. Натравить пушку можно будет в пятницу с утра. В тот же день заработает и сам чекер.

Проект будет считаться засчитанным, если следующие показатели за время проверки не увеличатся более чем на 50% по сравнению с вашими историческими данными:

* для elasticsearch: 
  * `indexing latency` ,
* показатели для kafka:
  * `TotalTimeMs` в broker metrics,
  * `Request latency average` в kafka producer metrics,
  * `ConsumerLag/MaxLag` в Kafka consumer metrics,
* показатели для spark streaming:
  * `active tasks (stacked per executor)`

  * `HDFS reads/executor`

  * `completed tasks per executor`

    ​

### 3. Ссылки для изучения

* [Мониторинг Kafka](https://www.datadoghq.com/blog/monitoring-kafka-performance-metrics/)
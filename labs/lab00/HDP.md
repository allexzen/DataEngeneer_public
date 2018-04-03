## Разворачивание кластера из 4 машин в облаке с помощью Ambari от Hortonworks

### Часть 1. Установить Ambari Server на одной из машин

Вначале зайдём на наш сервер-менеджер.

#### Зайти на сервер №1 (сервер-менеджер)

Заходим на первый по списку сервер по SSH:

`ssh -i gcp.pem ubuntu@<server_manager_public_ip>`

Обратите внимание, что сертификат `gcp.pem` должен быть в вашей текущей директории (либо вам нужно указать полный путь к нему).

В случае, если вы работаете с PuTTY, то вам придётся переформатировать ключ в формат .ppk. Можете почитать — http://stackoverflow.com/questions/3190667/convert-pem-to-ppk-file-format

> Инструкция по PuTTY (для пользователей Windows):
>
> 1. Загружаете приватный ключ.
>
> 2. Запускаете puttygen, Conversions -> Import key. Выбираете скачанный ключ. После этого — Save private key. Соглашаетесь сохранить без passphrase. Сохраняете в `private.ppk`.
>
> 3. Запускаете Putty, в Host Name вводите: `ubuntu@HOST`, где HOST - машина, выданная вам для кластера первой. Далее слева выбираете Connection -> SSH -> Auth, в Private Key указываете на `private.ppk`. 
>
> 4. Возвращаетесь в Sessions влева, вводите в Save Session какое-нибудь имя, и Save. Это чтобы не вводить все второй раз. Потом - Open. Начнется соединение. В первый раз появится предупреджение, согласиться.

#### Ставим Ambari на сервер-менеджер

⚠️ Все действия, описанные в этой части, предполагают их выполнение от имени пользователя `root`. Поэтому не забудьте им представиться (выполните в консоли команду):

`sudo su`

Добавляем репозиторий Ambari в директорию на установочной машине:

`wget -O /etc/apt/sources.list.d/ambari.list http://public-repo-1.hortonworks.com/ambari/ubuntu16/2.x/updates/2.5.2.0/ambari.list`

`apt-key adv --recv-keys --keyserver keyserver.ubuntu.com B9733A7A07513CAD`

Обновляем apt, чтобы получить доступ к новым пакетам Ambari:

`apt-get update`

Дальше можно проверить, загрузились ли нужные пакеты:

`apt-cache showpkg ambari-server`

`apt-cache showpkg ambari-agent`

`apt-cache showpkg ambari-metrics-assembly`

Правильный вывод любой из команд выше выглядит так:

```bash
root@node1:/home/ubuntu# apt-cache showpkg ambari-server
Package: ambari-server
Versions: 
2.5.2.0-298 (/var/lib/apt/lists/public-repo-1.hortonworks.com_ambari_ubuntu16_2.x_updates_2.5.2.0_dists_Ambari_main_binary-amd64_Packages) (/var/lib/dpkg/status)
 Description Language: 
                 File: /var/lib/apt/lists/public-repo-1.hortonworks.com_ambari_ubuntu16_2.x_updates_2.5.2.0_dists_Ambari_main_binary-amd64_Packages
                  MD5: c6d904389bc0d41429b0c7c52796924c


Reverse Depends: 
Dependencies: 
2.5.2.0-298 - openssl (0 (null)) postgresql (2 8.1) python (2 2.6) curl (0 (null)) 
Provides: 
2.5.2.0-298 - 
Reverse Provides: 
```

Теперь мы готовы установить Ambari:

`apt-get install ambari-server`

Пакет весьма увесистый:

> After this operation, 744 MB of additional disk space will be used.

Через несколько минут установка Ambari будет завершена, и вы увидите следующий вывод:

```
Setting up ambari-server (2.4.2.0-136) ...
update-rc.d: warning: default stop runlevel arguments (0 1 6) do not match ambari-server Default-Stop values (0 6)
 Adding system startup for /etc/init.d/ambari-server ...
   /etc/rc0.d/K20ambari-server -> ../init.d/ambari-server
   /etc/rc1.d/K20ambari-server -> ../init.d/ambari-server
   /etc/rc6.d/K20ambari-server -> ../init.d/ambari-server
   /etc/rc2.d/S20ambari-server -> ../init.d/ambari-server
   /etc/rc3.d/S20ambari-server -> ../init.d/ambari-server
   /etc/rc4.d/S20ambari-server -> ../init.d/ambari-server
   /etc/rc5.d/S20ambari-server -> ../init.d/ambari-server
Processing triggers for libc-bin (2.19-0ubuntu6.9) ...
```

Сервер Ambari установлен, теперь перейдём к его настройке.

#### Настройка Ambari

Для настройки Ambari в интерактивном консольном режиме, воспользуемся командой:

`ambari-server setup`
(не забываем, что все команды выполняются из-под пользователя `root`)

На все вопросы установщика отвечаем дефолтно (достаточно нажимать Enter):

`Customize user account for ambari-server daemon [y/n] (n)?`

JDK выбираем версии 1.8.

`Do you accept the Oracle Binary Code License Agreement [y/n] (y)? y`

```
Configuring database...
Enter advanced database configuration [y/n] (n)? n
```

Видим, что установка успешно завершена:

`Ambari Server 'setup' completed successfully.`

Запускаем сервер:

`ambari-server start`

Видим, что всё запустилось: `Ambari Server 'start' completed successfully.` Для тех, кто хочет перепроверить: 

`ps -ef | grep Ambari`

Вывод:

```
root      3718     1 56 11:32 pts/0    00:00:51 /usr/jdk64/jdk1.8.0_77/bin/java -server -XX:NewRatio=3 -XX:+UseConcMarkSweepGC -XX:-UseGCOverheadLimit -XX:CMSInitiatingOccupancyFraction=60 -XX:+CMSClassUnloadingEnabled -Dsun.zip.disableMemoryMapping=true -Xms512m -Xmx2048m -XX:MaxPermSize=128m -Djava.security.auth.login.config=/etc/ambari-server/conf/krb5JAASLogin.conf -Djava.security.krb5.conf=/etc/krb5.conf -Djavax.security.auth.useSubjectCredsOnly=false -cp /etc/ambari-server/conf:/usr/lib/ambari-server/*:/usr/share/java/postgresql-jdbc.jar org.apache.ambari.server.controller.AmbariServer
```

Машина настроена, можно закрыть сессию:

```
exit
exit
```

### Часть 2. Подключиться через Web-интерфейс к Ambari и собрать из остальных нод кластер 

Всю дальнейшую работу в этой части будем вести с помощью веб-интерфейса Ambari. 

Заходим браузером по адресу http://<ambari_server_manager_public_dns>:8080 (вам снова нужен Public IP той машины, которую вы только что настраивали). Он очень удобно записан в logout-сообщении:

```
logout
Connection to 35.187.46.117 closed.
```

В случае наших машин, это http://35.187.46.117:8080

Логин: `admin`

Пароль: `admin`

Нас интересует кнопка Launch Install Wizard:

<img src="http://data.newprolab.com/public-newprolab-com/de_hdp_launch2.png">

Называем наш кластер `newprolab`.

**Select Version**: удалите все репозитории кроме ubuntu16.

<img src="http://data.newprolab.com/public-newprolab-com/de_hdp_ubuntu16.png">


**Install Options**

В Target Hosts указываем *внутренние* DNS-адреса (Private DNS) всех четырёх инстансов. Внутренние адреса оканчиваются на `.internal`. Их можно получить, применив команду `hostname -f`.

В Host Registration Information добавляем свой *private* key `gcp.pem`. С помощью этого ключа скрипт деплоя Ambari будет заходить на каждую перечисленную выше вами машину и настраивать все необходимые сервисы.

SSH User Account: ubuntu

SSH Port Number: 22

<img src="http://data.newprolab.com/public-newprolab-com/de_hdp_fqdn.png">


**Confirm Hosts**

На этом экране проверяется наличие и доступ к указанным вами хостам. Если вы всё сделали правильно, проблем быть не должно, и вы через некоторое время увидите следующий статус (варнинги нам не помешают, продолжаем с ними):

<img src="http://data.newprolab.com/public-newprolab-com/de_hdp_success2.png">

⚠️ Если же вы видите ошибки, вы наверняка:

* Указали Public вместо Private DNS хостнеймы.
* Выбрали какой-то другой ключ.
* Забыли изменить SSH User Account на ubuntu.
* Возможно, что у вас закончился час, и инстансы автоматически удалились.


**Choose Services**

По большому счёту, нам для вашего проекта понадобятся (часть из них не потребуется, но необходима для жизни других компонентов):

* HDFS
* YARN + MapReduce2
* Tez
* Hive
* Pig
* HBase
* Sqoop
* Oozie
* ZooKeeper
* Storm
* Ambari Infra
* Ambari Metrics
* Kafka
* Smart Sense
* Spark2
* Slider
* Druid

Все остальные сервисы нам не нужны, выключите их. Разумеется, если вам хочется поэкспериментировать, мы вас не останавливаем :)

Игнорируем Limited Functionality Warning. Нажимаем Proceed Anyway. Дважды!


**Assign Masters**

Нам сейчас не очень принципиальна конфигурация кластера, поэтому оставляем всё в нетронутом виде и нажимаем Next.


**Assign Slaves and Clients**

Убедитесь, что все машины имеют роли DataNode и NodeManager. Остальные настройки можно оставить по умолчанию.

**Customize Services**

Могут появиться разные предупреждения, отмеченные красным. Кликните на `Show properties with issues`. В большинстве своем там надо проставить пароль на разные базы данных. Где-то потребуется указать свои контактные данные. После того как вы решите все проблемы в этой части, появится возможность нажать `next`.


**Review**

Всё должно быть в порядке, так что нажимайте `Deploy`.

Начинается конфигурирование машин на кластере:

<img src="http://data.newprolab.com/public-newprolab-com/de_hdp_installing.png">

⚠️ Если вы где-то сильно не ошиблись, процедура должна пройти без ошибок. Главная проблема, которая может теоретически возникнуть — закрытые порты. Когда статус хоста меняется на Fail, открывайте Message для этого хоста и смотрите, что написано в логе. Если это действительно похоже на закрытый порт (обычно это некий отказ в доступе: connection refused), то вам стоит обратиться к куратору.

После успешного окончания деплоя Message для всех хостов сменяется на Success:

<img src="http://data.newprolab.com/public-newprolab-com/de_hdp_success.png">

После этого нажимаем Next, смотрим Summary и завершаем процедуру установки кластера. Если вам стабильно не везёт, то в Summary вы увидите ошибки или ворнинги, подробнее о которых будет сказано дальше.

Возможен вариант, когда все установится, но с оранжевым цветом.

<img src="http://data.newprolab.com/public-newprolab-com/de_hdp_warnings.png">

#### Веб-интерфейс управления кластером Ambari

Ambari на рабочем и установленном кластере выглядит так:

<img src="http://data.newprolab.com/public-newprolab-com/de_hdp_ambari.png">

Если возникают проблемы, вы это сразу же видите по красному цвету.

Для нашего кластера наличие нескольких проблем допустимо, но всё равно их лучше  разобрать — для этого нужно пойти непосредственно в мониторинг хоста, в который проще всего попасть из статус-панели верхнего меню:

<img src="http://data.newprolab.com/public-newprolab-com/de_hdp_ambari_red.png">

Смотрим на алерты, переходим во вкладку Summary. Вполне возможно, что сервис просто нужно включить:

<img src="http://data.newprolab.com/public-newprolab-com/de_hdp_stopped.png">

Возможно, придется включить так каждый неработающий сервис на каждой ноде. Что поделать, таков уж софт `¯\_(ツ)_/¯`. 

Если ворнинги останутся на части сервисов, которые мы не собираемся использовать в ходе программы, то можно так и оставить (например, Hive).
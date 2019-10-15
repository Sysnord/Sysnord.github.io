---
title: "Как установить Oracle XE 18c на Linux Fedora 30 Server"
layout: "post"
tags: [Oracle, XE, Linux, Fedora]
---

Появилась необходимость установить Oracle XE 18c на Linux. Выбор пал на Fedora 30 Server, т.к. он бесплатный и на ядре Red Hat Enterprise Linux как и Oraсle Enterprise Linux.
При установке выявились некоторые проблемы и вопросы, и я нашел часть ответов в [блоге](https://gdsotirov.blogspot.com/2018/10/first-impressions-from-oracle-xe-18c.html) коллеги из Болгарии. Но в блоге есть некоторые моменты, которые я хотел бы уточнить и тем самым собрать более полную и актуальную инструкцию на русском языке.

#### Начинаем со скачивания необходимого ПО

* Качаем [RPM файлы](https://www.oracle.com/database/technologies/appdev/xe/quickstart.html) с официального сайта Oracle. Нас интересует раздел Installing Oracle Database XE и часть Red Hat compatible Linux distribution.
* Качаем пакет [compat-libcap1](http://mirror.centos.org/centos/7/os/x86_64/Packages/compat-libcap1-1.10-7.el7.x86_64.rpm) из репозитория CentOS. Без установки этого пакета установка пакета preinstall от Oracle свалится с ошибкой _nothing provides compat-libcap1 needed by oracle..._.

#### Приступаем к установке

Переключаемся на суперпользователя

`su`

Ставим пакет compat-libcap1.Для удобства я переименовал ранее скаченный пает в compat.rpm

`yum -y localinstall compat.rpm` 

После успешной установки compat-libcap1 можно ставить пакет preinstall от Oracle. Я его тоже переименовал в oracle_pre.rpm

`yum -y localinstall oracle_pre.rpm`

Если выше все успешно установилось, то надо установить еще один пакет, иначе установка самой Oracle XE свалится с ошибкой. 

`dnf install libnsl`

Теперь можно приступить к самой установке СУБД Oracle XE.Свой пакет переименовал для удобства в oracle.rpm. Если запускать установку командой yum -y localinstall oracle.rpm как в [официальной инструкии](https://docs.oracle.com/en/database/oracle/oracle-database/18/xeinl/procedure-installing-oracle-database-xe.html), то получим ошибку _package oracle-database-xe-18c-1.0-1.x86_64 does not verify: no digest_. Поэтому запускаем через RPM

`rpm -i --nodigest oracle.rpm`

После утановки СУБД нужно запустить ее конфигурирование, о чем и сообщает сообщение _Oracle home installed successfully and ready to be configured_ в конце установки СУБД. Для запуска конфигурирования запускаем:

`/etc/init.d/oracle-xe-18c configure`

Как только запустится конфигурирование система попросит указать пароль для пользователей sys, system и pbadmin. Дожидаемся завершения процесса.

#### Настройка системы для запуска СУБД

Теперь необходимо прописать переменные среды. Открываем файл _etc/profile_ любым редактором (я пользуюсь Midnight Commander) и в конце файла добавляем строки:
```
export ORACLE_HOME=/opt/oracle/product/18c/dbhomeXE
export ORACLE_SID=XE
pathmunge $ORACLE_HOME/bin after
```
Для настройки доступа извне к СУБД надо открыть порты.

Определяем нужный сетевой интерфейс через который будет происходить подключение к серверу командой
`ifconfig`.
Получаем список интерфейсов с их параметрами. Запоминаем имя нужного интерфейса(для примера у меня он назывался _enp0s3_) и выполняем:

`firewall-cmd --get-zone-of-interface=enp0s3`

Результатом будет строка _FedoraServer_.

Теперь надо указать порт который надо открыть. Нам надо открыть TCP 1521:

`firewall-cmd --permanent --add-port 1521/tcp`

Проверяем, что порт открылся:

`firewall-cmd --list-ports`

В результатах должно быть _1521/tcp_

Перезапускаем сервер и после загрузки снова логинимся под суперпользователем и запускаем СУБД:

`cd /etc/init.d/`

`./oracle-xe-18c start`

Запустится Listener и сам inctance Oracle XE. Для проверки работоспособности СУБД запустим sqlplus:

`cd opt/oracle/product/18c/dbhomeXE/bin`

`./sqlplus`

Если верно прописали переменные среды в _etc/profile_ то увидим приглашение ввести логин. Вводим SYSTEM и потом пароль указанный при конфигурировании СУБД. Если указали верные данные, то увидим сообщение

_Connected to : Oracle Database 18c Express Edition..._ 

_SQL>_

Выполним какой-нибудь SQL запрос:

{% highlight sql %}
Select * from all_users;
{% endhighlight %}

Должно вывести пользователей СУБД

Если отображается то все работает корректно и можно попробовать подключаться извне.


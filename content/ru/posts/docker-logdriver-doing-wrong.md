---
title: "Docker Logdriver: doing it wrong"
date: 2020-06-07T11:28:10+03:00
draft: false
description: >-
  Docker userland-proxy мешает работе logdriver'a, ломая логику async-connect. Как следствие, будут потеряны все логи, отправленные logdriver'ом до момента готовности аггрегатора логов, например, *Fluentd*.
categories: ["devops", "docker", "logs"]
toc: false
displayInMenu: false
displayInList: true
resources:
- name: featuredImage
  src: "Filename of the post's featured image, used as the card image and the image at the top of the article"
  params:
    description: "Docker Logdriver post image"
---

## TL;DR
> Docker userland-proxy мешает работе logdriver'a, ломая логику async-connect. Как следствие, будут потеряны все логи, отправленные logdriver'ом до момента готовности аггрегатора логов, например, *Fluentd*.


## Никто не любит logging driver'ы

Docker Log Driver - итересная штука, созданная, чтобы облегчить сбор логов. Казалось бы, подключил логдрайвер - и забыл. Но не все так просто, и в реальных проектах зачастую не используют logdriver, предпочитая просто парсить логи докера. Почему?

1. Прежде всего - *split logging*, который в Docker CE поддерживается только с некоторыми драйверами. В общем, проще считать, что его нет. Вроде бы, и зачем он нужен, при живой системе логгирования-то? Но на практике бывает очень удобно подключиться к ноде и посмотреть последние события через `docker logs`.

1. Kubernetes лог драйверы не поддерживает. А все либо уже в k8s, либо хотят туда. Nuff said.

1. Работает не очевидно, страшно потерять логи. Читай, *"попробовал настроить, логи теряются, разбираться не хочется"*. Парсить json'ы надежней.

Если все аргументы выше не заставили передумать, приступим. Настраивать будем log driver для *Fluentd*.

## Настройка типичного EFK

Существует масса руководств по настройке docker log driver, да хотя бы прям в доке docker`a. Как выглядит типичный "Getting started guide" для EFK?

Простейший *fluent.conf*:
```
<source>
  @type forward
  @label @out
</source>

<label @out>
  <match>
    @type elasticsearch
    host elasticsearch
    logstash_format true
    <buffer>
      flush_interval 10
    </buffer>
  </match>
</label>
```

Добавляем во *Fluentd* elasticsearch плагин:
```
FROM fluent/fluentd:v1.10-1

USER root

RUN gem install fluent-plugin-elasticsearch \
 && gem sources --clear-all \
 && rm -rf /tmp/* /var/tmp/* /usr/lib/ruby/gems/*/cache/*.gem

USER fluent
```

*docker-compose.yml*:
```
version: "3.6"

services:
  fluentd:
    image: fluentd:efk
    build: .
    ports:
      - 127.0.0.1:24224:24224
    volumes:
      - ./fluent.conf:/fluentd/etc/fluent.conf
    networks:
      - internal

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch-oss:7.7.0
    environment:
      - discovery.type=single-node
    networks:
      - internal
    logging:
      driver: fluentd
      options:
        fluentd-async-connect: "true"
        fluentd-sub-second-precision: "true"

  kibana:
    image: docker.elastic.co/kibana/kibana-oss:7.7.0
    ports:
      - 5601:5601
    depends_on:
      - elasticsearch
    networks:
      - internal
      - external
    logging:
      driver: fluentd
      options:
        fluentd-async-connect: "true"
        fluentd-sub-second-precision: "true"

networks:
  internal:
  external:
```


Абсолютно обычный сетап EFK. Остается добавить только приложение, базу и т.п.

Отдельно остановлюсь на опциях:
* `fluentd-sub-second-precision` - логгировать время с субсекундной точностью. Благодаря этому логи в кибане будут отображаться в правильном порядке (с большей вероятностью).
* `fluentd-async-connect` - если не удается подключиться к *Fluentd*, то не падаем с ошибкой, а продолжаем попытки коннекта, буфферизируя неотправленные логи.

## Что же не так?

Все просто, с подобным конфигом мы будем терять все логи до момента готовности *Fluentd*. Причем, момент готовности != моменту старта контейнера.

Это легко проверить, подняв данный `docker-compose.yml`: логов старта elasticsearch и kibana в самом elasticsearch не будет.

Начинаем разбираться. Кажется, что всему виной опция `fluentd-async-connect`, ведь именно она отвечает за то, чтобы копить логи, пока не станет доступным *Fluentd*. Для подтверждения данной теории можно запустить `tcpdump` и восстановить ход событий:

*(кибану можно исключить из процесса для ясности)*
- стартует *fluentd*
- стартует *elastic*
- *fluentd* ждет готовности *elastic*: порт `24224` еще не открыт
- *elastic* насыпает логов, логи вроде как отправляются в `localhost:24224`, эти логи потеряны
- *fluentd* видит *elastic*, открывает порт, с этого момента логи логгируются, как и ожидалось

Почему же log driver отправляет логи, хотя порт `24224` еще не открыт? А точно ли он не открыт? Это легко проверить более высокоуровневыми средствами.

Пытаемся подключиться к `24224` до старта compose - ожидаемый отлуп:
```
$ telnet localhost 24224
Trying 127.0.0.1...
telnet: Unable to connect to remote host: Connection refused
$
```

Пытаемся подключиться к `24224` сразу после старта compose, т.е. до открытия порта *Fluentd*:
```
$ telnet localhost 24224
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Connection closed by foreign host.
$
```
**WTF?** Кто-то уже слушает порт и сбрасывает соединение!

Пытаемся подключиться к `24224` после того как *Fluentd* "увидел" elasticsearch и, соответственно, выставил порт - ожидаемо, подключились:
```
$ telnet localhost 24224
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
```

## Что делать и кто виноват?

Путем недолгих размышлений было выяснено, что это *docker-proxy*. Отключаем:

*/etc/docker/daemon.json*:
```
{
  "userland-proxy": false
}
```
И все приходит в норму: до старта worker'a *Fluentd* и открытия порта по `24224` никто не отвечает, а благодаря `fluentd-async-connect` log driver пытается отправить логи, повторяя попытки согласно настройкам. И после готовности *Fluentd* логи будут успешно отправлены, без потерь.

### Что за docker-proxy?
Не вдаваясь в подробности, *docker userland-proxy* - это то, что используется сейчас чаще всего совместно с **IPv6**, соответственно, в большинстве случаев, может быть безболезненно отключено.

## Вывод: от IPv6 не уйти.

К сожалению, это не единственная особенность в работе с docker log driver, достаточно взглянуть на багтрекер... В т.ч. такие нюансы - еще один отличный довод в пользу отказа от logdrivers.
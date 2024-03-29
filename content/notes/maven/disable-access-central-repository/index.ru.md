---
title: "Как отключить доступ к central"
date: 2024-03-20T07:32:46+04:00
type: docs
draft: false
description: "Варианты отключения доступа к репозиторию Maven Central в отдельном проекте."
noindex: false
featured: false
pinned: false
comments: true
slug: disable-access-central-repository
categories:
  - skills
tags:
  - maven
images:
  - will-porada-ZaGcU6BxJEc-unsplash.jpg
---
{{% bs/alert warning %}}
Все примеры запускались на Maven 3.9.6

Если вы пользуетесь другой версией, то могут быть расхождения в фактическом поведении сборки.
Если же используемая версия ниже 3.8.1, то примеры конфигурации точно потребуют внесения корректировок.

Более подробно этот вопрос был рассмотрен в статье [Maven. Отключаем доступ к центральному хранилищу](/post/disable-access-central-repository/), включая некоторые подводные камни каждого варианта.
{{% /bs/alert %}}

## Предисловие
Работа на крупных проектах часто связана с необходимостью соблюдать различные требования,
в том числе мириться с ограничениями от службы информационной безопасности.
Одним из таких ограничений в последнее время стал запрет на свободное использование артефактов с центрального maven репозитория.
При этом часто происходит переход от автоматического проксирования на "ручной" процесс загрузки отдельных артефактов,
в том числе с привлечением выделенной группы, которая рассматривает специальные заявки.

В итоге появляется необходимость исключить расхождение между сборкой проекта локально и на CI в закрытом контуре.
## Варианты решения проблемы
Все варианты основываются на создании отдельного конфигурационного файла maven.
Альтернативы из серии "заблокировать адрес на уровне hosts или маршрутов сети" в большинстве случаев не подходят,
так как ограничение надо реализовать только для отдельного проекта.

По запросу `Disable Maven central` чаще всего обсуждают два варианта:
1. переопределение адреса
2. указание зеркала

Сперва их и рассмотрим.
### Переопределение адреса репозитория _central_
Необходимо указать адрес внутреннего репозитория, куда происходит загрузка внешних артефактов.

{{< details "settings-disable-central.xml v1" open >}}
{{< highlight xml "hl_lines=10 12 17 19" >}}
{{< file-content "data/settings-disable-central-1.xml" >}}
{{< /highlight >}}
{{< /details >}}


На первый взгляд данное решение выглядит подходящим, но надо учитывать один важный момент.
В локальном репозитории создаются файлы `_remote.repositories`, в которые записывается идентификатор источника загрузки.

При сборках maven проверяет, что источник ранее сохраненного артефакта присутствует в текущем контексте.
Если в изолированных проектах переопределить _central_ с разными адресами, но при этом использовать одно локальное хранилище,
то можно подключить зависимость из "другого проекта".
### Использование зеркала для репозитория _central_
Можно заменить переопределение репозиториев на добавление зеркала.
{{< details "settings-disable-central.xml v2" open >}}
{{< highlight xml "hl_lines=6-13" >}}
{{< file-content "data/settings-disable-central-2.xml" >}}
{{< /highlight >}}
{{< /details >}}

В метаинформацию по источнику загрузки артефакта будет добавляться уже не идентификатор репозитория, а идентификатор зеркала: _private-nexus-mock-external_.
Это устранит возможность возникновения проблемы с некорректным определением доступности в текущем контексте сборки,
если избегать создание зеркал с одинаковым именем, но разными адресами.
### Блокировка репозитория central
В целом, предыдущий вариант решает необходимую задачу и можно воспользоваться им.

Есть только "концептуальный" момент, что `https://akorolev.dev/repository/common/` — это не зеркало внешних репозиториев, а внутреннее хранилище организации.
В первую очередь пространство предназначено для корпоративных библиотек общего назначения, в которое дополнительно загружаются отдельные внешние зависимости по мере потребностей.

Если используется Maven версии `3.8.1` и выше, то можно попробовать альтернативный вариант.
В данном релизе была обновлена [xml-модель файла настроек](https://maven.apache.org/xsd/settings-1.2.0.xsd) и элемент `mirror` обзавелся вложенным `blocked`.
{{< details "settings-disable-central.xml v3" open >}}
{{< highlight xml "hl_lines=6-14" >}}
{{< file-content "data/settings-disable-central-3.xml" >}}
{{< /highlight >}}
{{< /details >}}
Если при такой конфигурации сборщик будет доходить до обращения к _central_ репозиторию, то в логи будут выводиться соответствующие предупреждения
`Blocked mirror for repositories: [central (https://repo.maven.apache.org/maven2, default, releases)]`, а сборка будет падать в ошибку.

Но на данном варианте тоже была замечена проблема при использовании достаточно старых версий плагинов.
Например, `flatten-maven-plugin` версии `1.1.0` (21 декабря 2018 г.) стабильно пытался проверить доступность через заблокированное зеркало,
так как ошибочно считал, что ранее закешированный артефакт загружен с недоступного в текущем контексте репозитория.
Хотя его банальное обновление до следующей версии `1.2.1` (18 января 2020 г.) устранило данную проблему и проект начинал собираться корректно.

{{% bs/alert secondary %}}
Источник изображения в заголовке Unsplash. Автор [Will Porada](https://unsplash.com/@will0629).
{{% /bs/alert %}}
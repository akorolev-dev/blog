---
title: "Maven. Отключаем доступ к центральному хранилищу"
date: 2024-03-20T05:24:45+04:00
draft: false
description: "Рассмотрим варианты отключения доступа к центральному репозиторию Maven для соблюдения политик безопасности заказчика."
noindex: false
featured: true
pinned: true
comments: true
slug: disable-access-central-repository
categories:
  - training
tags:
  - maven
images:
  - jacob-li-L6HvuhsY2Dw-unsplash-crop.jpg
---
{{% bs/alert info %}}
Все примеры запускались на Maven 3.9.6

Если вы пользуетесь другой версией, то могут быть расхождения в фактическом поведении сборки.
Если же используемая версия ниже 3.8.1, то примеры конфигурации точно потребуют внесения корректировок.
{{% /bs/alert %}}

## Предисловие
Работа на крупных проектах, например в банковском секторе, часто связана с необходимостью соблюдать различные требования,
в том числе мириться с ограничениями от службы информационной безопасности.
Одним из таких ограничений стал запрет на использование артефактов с центрального maven репозитория.

Конечно, речь не идет о полном отказе от общедоступных библиотек.
Но в большинстве случаев уже не получится просто взять понравившуюся версию, которая опубликована на _central_.

Кто-то автоматизирует процесс загрузки новых зависимостей. Только работает она уже не через простое проксирование на внутреннем хранилище,
а с проведением различных проверок, которые не всегда могут завершаться успехом.
Кто-то поступает более кардинально и вводит целый процесс добавления новых артефактов, который не обходится без заведения специальных запросов и "ручного" рассмотрения.

Можно долго дискутировать над разумностью таких подходов, но фактически остается только подстраиваться под обстоятельства.
И тут можно столкнуться с неприятной ситуацией, когда при локальной разработке артефакты загрузятся, а на CI окажется, что их значительная часть отсутствует.

Недавно первой задачей на новом проекте стало значительное обновление стека имеющихся микросервисов.
Целью был переход на Spring Boot 3 версии, а вот текущий стек: Java 8, Spring Boot 2.5, Camunda 7.12 и т.д.
Местами могли попадаться раритетные версии артефактов, потому что на старте проектов кто-то копипастил зависимости из своих старых решений.

При этом предоставленный `settings.xml` содержал несколько репозиториев с внутреннего Nexus,
а об отсутствии какого-либо полноценного проксирования центральных репозиториев я узнал только по факту ошибок на CI.

## Откуда берется Maven Central
Создадим практически минимальный `pom.xml` и `settings.xml`, в котором добавлено два внутренних репозитория:
- Общие артефакты всей организации: _private-nexus-common_.
- Артефакты подразделения, на проекте которого мы работаем: _private-nexus-subdivision_.

Любые внешние артефакты загружаются в _private-nexus-common_ по итогам достаточно долгого процесса.
При этом закончиться он может и формулировкой "Не видим необходимости в добавлении артефакта".

Давайте посмотрим на полный состав pom, по которому будет происходить сборка.

{{< bs/toggle name=first >}}
{{< bs/toggle-item "Создаем эффективный pom" >}}
{{< highlight bash >}}
akorolev.dev maven
$ mvn help:effective-pom -Dverbose -Doutput=effective-pom.xml -s ~/.m2/settings.xml
Picked up JAVA_TOOL_OPTIONS: -Dfile.encoding=UTF-8
[INFO] Scanning for projects...
[INFO]
[INFO] -------------------< dev.akorolev.maven:simple-pom >--------------------
[INFO] Building simple-pom 0.1.0
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- help:3.4.0:effective-pom (default-cli) @ simple-pom ---
[INFO] Effective-POM written to: D:\repositories\articles\maven\effective-pom.xml
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  0.682 s
[INFO] Finished at: 2024-03-19T21:19:22+04:00
[INFO] ------------------------------------------------------------------------
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< bs/toggle-item settings.xml >}}
{{< highlight xml >}}
{{< file-content "data/settings-disable-central-1.xml" >}}
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< bs/toggle-item pom.xml >}}
{{< highlight xml >}}
{{< file-content "data/pom-1.xml" >}}
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< bs/toggle-item effective-pom.xml >}}
{{< highlight xml >}}
{{< file-content "data/effective-pom-1.xml" >}}
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< /bs/toggle >}}

Данных заметно больше, чем мы создали самостоятельно.
И главное, что добавились записи для репозитория _central_.
То есть, если какой-то артефакт не будет найден на _private-nexus-common_ или _private-nexus-subdivision_, то произойдет попытка его загрузки из центрального хранилища.

Эффективный pom.xml — это результат слияния так называемого Super POM, нашего файла проекта и используемых активных профилей из файла настроек.
Super POM можно найти внутри домашней директории используемого maven. Он располагается в `lib/maven-model-builder-3.9.6.jar`

{{< details "org/apache/maven/model/pom-4.0.0.xml" >}}
{{< highlight xml >}}
{{< file-content "data/pom-4.0.0.xml" >}}
{{< /highlight >}}
{{< /details >}}

Именно данный файл является причиной использования в любом проекте центрального репозитория.
Также в нём прописана директория `src/main/java`, как место для создания файлов с исходным кодом и прочие настройки по умолчанию.

## Варианты решения проблемы
Все варианты основываются на создании отдельного конфигурационного файла maven.
Альтернативы из серии "заблокировать адрес на уровне hosts или маршрутов сети" в большинстве случаев не подходят,
так как ограничение надо реализовать только для отдельного проекта.

По запросу `Disable Maven central` чаще всего обсуждают два варианта:
1. переопределение адреса
2. указание зеркала

Сперва их и рассмотрим.

### Переопределение адреса репозитория _central_
Эффективный POM строится методом наследования. Такую аналогию с java.lang.Object даже содержит
[официальная документация Maven](https://maven.apache.org/pom.html#the-super-pom).
Некоторые называют переопределение более логичным, когда речь идет именно про запрет _central_.
Можно встретить аргументы, что зеркала предназначены для "проектных репозиториев", которые добавлены непосредственно в `pom.xml`.
Приводятся доводы, что изменить их адрес можно только так, в отличие от репозиториев, которые прописаны в файле глобальных настроек.

По факту дела обстоят немного иначе. Произведем следующие изменения:
- заменим идентификаторы _private-nexus-common_ на _central_, так как именно сюда загружаются артефакты после успешного рассмотрения заявки
- добавим в `pom.xml` дополнительный репозиторий camunda (_camunda-bpm-nexus_)
- в настройках переопределим адрес для репозитория _camunda-bpm-nexus_
- в настройках переопределим свойство `java.version` на значение 21

В итоге получим `effective-pom.xml`, в котором применились все внесенные изменения из файла настроек.

{{< bs/toggle name=second >}}
{{< bs/toggle-item "settings-disable-central.xml v1" >}}
{{< highlight xml "hl_lines=13 17 34 21-25" >}}
{{< file-content "data/settings-disable-central-2.xml" >}}
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< bs/toggle-item pom.xml >}}
{{< highlight xml "hl_lines=16-20" >}}
{{< file-content "data/pom-2.xml" >}}
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< bs/toggle-item effective-pom.xml >}}
{{< highlight xml "hl_lines=9 14-16 19-21 31-33" >}}
{{< file-content "data/effective-pom-2.xml" >}}
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< /bs/toggle >}}

На первый взгляд данное решение выглядит подходящим, но надо учитывать один важный момент.
Когда происходит загрузка артефакта в локальный репозиторий `D:/.m2/local_repository`, то сохраняется метаинформация об идентификаторе источника в файле `_remote.repositories`.
Например, для `org/apache/maven/shared/maven-shared-utils/3.3.4` его содержимое поменяется следующим образом:
{{< bs/toggle name=third >}}
{{< bs/toggle-item "_remote.repositories v1" >}}
{{< highlight text >}}
{{< file-content "data/_remote.repositories-private-nexus-common" >}}
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< bs/toggle-item "_remote.repositories v2" >}}
{{< highlight text "hl_lines=4 6" >}}
{{< file-content "data/_remote.repositories-with-central" >}}
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< /bs/toggle >}}

То есть, в первый раз происходила загрузка артефакта из _private-nexus-common_.
Но при втором построении maven проверяет, что репозиторий _private-nexus-common_ отсутствует в текущем контексте сборки.
Поэтому производится повторный поиск артефакта среди активного набора репозиториев.

В таком случае в логах будет выводиться соответствующее уведомление:
```bash {hl_lines=[1, 4]}
[INFO] Artifact org.apache.maven.shared:maven-shared-utils:pom:3.3.4 is present in the local repository, but cached from a remote repository ID that is unavailable in current build context, verifying that is downloadable from [central (https://akorolev.dev/repository/common/, default, releases+snapshots)]
Downloading from central: https://akorolev.dev/repository/common/org/apache/maven/shared/maven-shared-utils/3.3.4/maven-shared-utils-3.3.4.pom
Downloaded from central: https://akorolev.dev/repository/common/org/apache/maven/shared/maven-shared-utils/3.3.4/maven-shared-utils-3.3.4.pom (0 B at 0 B/s)
[INFO] Artifact org.apache.maven.shared:maven-shared-utils:jar:3.3.4 is present in the local repository, but cached from a remote repository ID that is unavailable in current build context, verifying that is downloadable from [central (https://akorolev.dev/repository/common/, default, releases+snapshots)]
Downloading from central: https://akorolev.dev/repository/common/org/apache/maven/shared/maven-shared-utils/3.3.4/maven-shared-utils-3.3.4.jar
Downloaded from central: https://akorolev.dev/repository/common/org/apache/maven/shared/maven-shared-utils/3.3.4/maven-shared-utils-3.3.4.jar (0 B at 0 B/s)
```
Но если у вас будет два независимых проекта, в которых _central_ будет переопределен разными адресами, то упомянутый механизм проверки доступности артефакта может отработать некорректно.
Может возникнуть ошибочное решение, что зависимость ранее была загружена из доступного источника, хотя по факту она может оказаться вне доступа для текущего контекста сборки.

### Использование зеркала для репозитория _central_
Можно заменить переопределение репозиториев на добавление зеркала.
Модель POM не предполагает наличие `mirror` элементов, поэтому они в текст эффективного pom не добавляются.
{{< bs/toggle name=fourth >}}
{{< bs/toggle-item "settings-disable-central.xml v2" >}}
{{< highlight xml "hl_lines=8-13" >}}
{{< file-content "data/settings-disable-central-3.xml" >}}
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< bs/toggle-item effective-pom.xml >}}
{{< highlight xml "hl_lines=19-21 27-29 40-42" >}}
{{< file-content "data/effective-pom-3.xml" >}}
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< /bs/toggle >}}

Но в метаинформацию, для указанного ранее артефакта, будет добавляться уже не идентификатор репозитория, а идентификатор зеркала: _private-nexus-mock-external_.
Это устранит возможность возникновения проблемы с некорректным определением доступности в текущем контексте сборки.

### Блокировка репозитория central
В целом, предыдущий вариант решает необходимую задачу и можно воспользоваться им.

Есть только "концептуальный" момент, что `https://akorolev.dev/repository/common/` — это не зеркало внешних репозиториев, а внутреннее хранилище организации.
В первую очередь пространство предназначено для корпоративных библиотек общего назначения, в которое дополнительно загружаются отдельные внешние зависимости по мере потребностей.

Если используется Maven версии `3.8.1` и выше, то можно попробовать альтернативный вариант.
В данном релизе была обновлена xml-модель файла настроек ([v1.2.0](https://maven.apache.org/xsd/settings-1.2.0.xsd)) и элемент `mirror` обзавелся вложенным `blocked`.
{{< highlight xml "hl_lines=8-14" >}}
{{< file-content "data/settings-disable-central-4.xml" >}}
{{< /highlight >}}
Если при такой конфигурации сборщик будет доходить до обращения к _central_ репозиторию, то в логи будут выводиться соответствующие предупреждения
`Blocked mirror for repositories: [central (https://repo.maven.apache.org/maven2, default, releases)]`, а сборка будет падать в ошибку.

Но на данном варианте тоже была замечена проблема при использовании достаточно старых версий плагинов.
Например, `flatten-maven-plugin` версии `1.1.0` (21 декабря 2018 г.) стабильно пытался проверить доступность через заблокированное зеркало,
так как ошибочно считал, что ранее закешированный артефакт загружен с недоступного в текущем контексте репозитория.
Хотя его банальное обновление до следующей версии `1.2.1` (18 января 2020 г.) устранило данную проблему и проект начинал собираться корректно.

## Послесловие
По итогу, только на втором способе настроек я не заметил "подводных камней".

Хотя третий вариант в большей степени соответствует формулировке "запрет на использование".
И в данный момент я остановился на нём, чтобы производить сборку микросервисов на текущем проекте.

Знаете лучшую альтернативу? Поделитесь ей в комментариях.

{{% bs/alert secondary %}}
Источник изображения в заголовке Unsplash. Автор [Jacob Li](https://unsplash.com/@its_jacobli).
{{% /bs/alert %}}
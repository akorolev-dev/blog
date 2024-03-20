---
title: "Обход maven-default-http-blocker"
date: 2024-03-21T07:35:29+04:00
type: docs
draft: false
description: "Как обойти блокировку репозитория, который настроен только для работы по HTTP протоколу."
noindex: false
featured: false
pinned: false
comments: true
slug: maven-default-http-blocker
categories:
  - skills
tags:
  - maven
images:
  - jen-theodore-lU_9C2N46X4-unsplash-crop.jpg
---
{{% bs/alert info %}}
Все примеры запускались на Maven 3.9.6
{{% /bs/alert %}}

## Причины возникновения проблемы
Используя Maven 3.8.1 или более позднюю версию, можно столкнуться с ошибкой.

```bash
$ mvn clean package -s ~/.m2/settings-with-http.xml
Picked up JAVA_TOOL_OPTIONS: -Dfile.encoding=UTF-8
[INFO] Scanning for projects...
[INFO]
[INFO] -------------------< dev.akorolev.maven:simple-pom >--------------------
[INFO] Building simple-pom 0.1.0
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  0.287 s
[INFO] Finished at: 2024-03-21T07:31:59+04:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal on project simple-pom: Could not resolve dependencies for project dev.akorolev.maven:simple-pom:jar:0.1.0:
 Failed to collect dependencies at dev.akorolev.front:app:jar:6.2.0: Failed to read artifact descriptor for dev.akorolev.front:app:jar:6.2.0:
 The following artifacts could not be resolved: dev.akorolev.front:app:pom:6.2.0 (absent):
 Could not transfer artifact dev.akorolev.front:app:pom:6.2.0 from/to maven-default-http-blocker (http://0.0.0.0/):
 Blocked mirror for repositories: [private-nexus-common (http://akorolev.dev:9083/repository/common/, default, releases)] -> [Help 1]
[ERROR]
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR]
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/DependencyResolutionException
```
Данное поведение отражено в примечаниях к выпуску 3.8.1 как исправление [{{< abbr "CVE" >}}-2021-26291](https://maven.apache.org/docs/3.8.1/release-notes.html#cve-2021-26291).
И первым вариантом решения проблемы предлагается мигрировать с {{< abbr "HTTP" >}} на {{< abbr "HTTPS" >}} протокол.
Только в большинстве случаев такие репозитории находятся вне нашего влияния, что делает обновление невозможным.

## Что такое maven-default-http-blocker
Maven использует наследование не только в рамках построения эффективного `POM`, также существует понятие эффективного файла настроек.

{{< bs/toggle name=first >}}
{{< bs/toggle-item "mvn help:effective-settings" >}}
{{< highlight bash >}}
$ mvn help:effective-settings -s ~/.m2/settings-with-http.xml -Doutput=effective-settings.xml
Picked up JAVA_TOOL_OPTIONS: -Dfile.encoding=UTF-8
[INFO] Scanning for projects...
[INFO]
[INFO] -------------------< dev.akorolev.maven:simple-pom >--------------------
[INFO] Building simple-pom 0.1.0
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- help:3.4.0:effective-settings (default-cli) @ simple-pom ---
[INFO] Effective-settings written to: D:\repositories\articles\maven\effective-settings.xml
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  0.957 s
[INFO] Finished at: 2024-03-20T15:23:44+04:00
[INFO] ------------------------------------------------------------------------
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< bs/toggle-item "settings-with-http.xml v1" >}}
{{< highlight xml "hl_lines=16" >}}
{{< file-content "data/settings-with-http-1.xml" >}} 
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< bs/toggle-item effective-settings.xml >}}
{{< highlight xml "hl_lines=6-12 20" >}}
{{< file-content "data/effective-settings-1.xml" >}}
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< /bs/toggle >}}

Эффективные настройки — это результат объединения настроек уровня системы и уровня пользователя.
Первые описаны в файле `conf/settings.xml` внутри домашней директории используемого maven.
Вторые - в рамках заданного нами файла `~/.m2/settings-with-http.xml`.

## Как обойти блокировку репозитория
Возникающий _maven-default-http-blocker_ представляет собой просто заблокированное зеркало, которое задано для всех внешних {{< abbr "HTTP" >}} репозиториев.

И вариант обхода прост. В пользовательских настройках необходимо задать зеркало для нужного нам репозитория, которое не будет заблокированным.
И в рамках набора эффективных настроек оно будет располагаться "раньше".

{{< bs/toggle name=second >}}
{{< bs/toggle-item "settings-with-http.xml v2" >}}
{{< highlight xml "hl_lines=8-12" >}}
{{< file-content "data/settings-with-http-2.xml" >}}
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< bs/toggle-item effective-settings.xml >}}
{{< highlight xml "hl_lines=7-9 12-16 25" >}}
{{< file-content "data/effective-settings-2.xml" >}}
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< bs/toggle-item "Успешная загрузка" >}}
{{< highlight bash >}}
$ mvn clean package -s ~/.m2/settings-with-http.xml
Picked up JAVA_TOOL_OPTIONS: -Dfile.encoding=UTF-8
[INFO] Scanning for projects...
[INFO]
[INFO] -------------------< dev.akorolev.maven:simple-pom >--------------------
[INFO] Building simple-pom 0.1.0
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
Downloading from private-nexus-common-http-unblocker: http://akorolev.dev:9083/repository/common/dev/akorolev/front/app/6.2.0/app-6.2.0.pom
Downloaded from private-nexus-common-http-unblocker: http://akorolev.dev:9083/repository/common/dev/akorolev/front/app/6.2.0/app-6.2.0.pom (9.5 kB at 39 kB/s)
Downloading from private-nexus-common-http-unblocker: http://akorolev.dev:9083/repository/common/dev/akorolev/front/app/6.2.0/app-6.2.0.jar
Downloaded from private-nexus-common-http-unblocker: http://akorolev.dev:9083/repository/common/dev/akorolev/front/app/6.2.0/app-6.2.0.jar (78 kB at 154 kB/s)
...
{{< /highlight >}}
{{< /bs/toggle-item >}}
{{< /bs/toggle >}}


{{% bs/alert secondary %}}
Источник изображения в заголовке Unsplash. Автор [Jen Theodore](https://unsplash.com/@jentheodore).
{{% /bs/alert %}}
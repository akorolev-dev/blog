---
title: "Особенность миграции Camunda на версию 7.14"
date: 2024-04-03T19:25:15+04:00
type: docs
draft: false
description: "Несоответствие в схеме базы данных при миграции на версию 7.14"
noindex: false
featured: true
pinned: false
comments: true
slug: db-schema-discrepancy-during-migration-to-camunda-7.14
categories:
  - education
tags:
  - camunda
images:
  - suzanne-d-williams-VMKBFR6r_jg-unsplash.jpg
---

## Предыстория
В рамках проектных работ потребовалось произвести миграцию Camunda с версии 7.13 до 7.20, так как только она
[рассчитана на работу со Spring Boot 3 версии](https://docs.camunda.org/manual/7.20/user-guide/spring-boot-integration/version-compatibility/).

Для обновления схемы базы данных был сделан журнал миграций Liquibase, сформированный на основании sql из _org.camunda.bpm:camunda-engine_.

{{% bs/alert info %}}
Все работы выполнялись на PostgreSQL, наличие аналогичной особенности на других поддерживаемых СУБД не проверялось.
{{% /bs/alert %}}

## Выявленное расхождение
Для тестирования корректности созданных миграций производилось сравнение со схемой для версии 7.20, которая разворачивается при запуске на пустой БД.

За исключением отличия в порядке некоторых колонок отдельных таблиц, обнаружилось несоответствие в длине строковой колонки `ASSIGNEE_` таблицы `ACT_HI_ACTINST`.
В базе данных для версии 7.20 колонка была на 255 символов, а по итогам применения собственных миграций осталась на 64 символа.

Естественно, сперва подозрение возникло на пропущенный файл обновления между минорными версиями, но оно не подтвердилось.

При дальнейшем анализе несоответствие было локализовано в обновлении на версию 7.14

В скрипте создания схемы данная колонка [задана с длиной 255 символов](https://github.com/camunda/camunda-bpm-platform/blob/30a2732a2b752f7b897625e6a386787ee0c1c510/engine/src/main/resources/org/camunda/bpm/engine/db/create/activiti.postgres.create.history.sql#L56).

А вот в четырех соответствующих скриптах обновления такой модификации не происходит:
- [postgres_engine_7.13_patch_7.13.4_to_7.13.5_1.sql](https://github.com/camunda/camunda-bpm-platform/blob/7.14.0/engine/src/main/resources/org/camunda/bpm/engine/db/upgrade/postgres_engine_7.13_patch_7.13.4_to_7.13.5_1.sql)
- [postgres_engine_7.13_patch_7.13.4_to_7.13.5_2.sql](https://github.com/camunda/camunda-bpm-platform/blob/7.14.0/engine/src/main/resources/org/camunda/bpm/engine/db/upgrade/postgres_engine_7.13_patch_7.13.4_to_7.13.5_2.sql)
- [postgres_engine_7.13_patch_7.13.5_to_7.13.6.sql](https://github.com/camunda/camunda-bpm-platform/blob/7.14.0/engine/src/main/resources/org/camunda/bpm/engine/db/upgrade/postgres_engine_7.13_patch_7.13.5_to_7.13.6.sql)
- [postgres_engine_7.13_to_7.14.sql](https://github.com/camunda/camunda-bpm-platform/blob/7.14.0/engine/src/main/resources/org/camunda/bpm/engine/db/upgrade/postgres_engine_7.13_to_7.14.sql)

## Заключение
В тексте не используются понятия "ошибка" или "дефект", потому что фактически они отсутствуют, если система не будет производить сохранение значения длиной более 64 символов.
Но и не стоит игнорировать возможность, что в новой версии могут произойти изменения, рассчитанные на работу с полем большей длины.

{{% bs/alert secondary %}}
Источник изображения в заголовке Unsplash. Автор [Suzanne D. Williams](https://unsplash.com/@scw1217).
{{% /bs/alert %}}
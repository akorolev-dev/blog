---
title: "Git. Ошибка «invalid path»"
date: 2024-03-18T07:50:31+04:00
type: docs
draft: false
description: "Решение ошибки «невалидный путь» при клонировании репозитория."
noindex: false
featured: true
pinned: false
comments: true
slug: fix-error-invalid-path-git
categories:
  - skills
tags:
  - git
images:
  - zach-lezniewicz-hPcKqp8ztZw-unsplash-crop.jpg
---
{{% bs/alert info %}}
Данная ошибка возникала на Windows 10, при использовании Git 2.44 через Git Bash.

Но аналогичная особенность свойственна и Mac OS. Это является поведением по умолчанию самого Git, без привязки к конкретному клиенту.
{{% /bs/alert %}}

## Предыстория
Находите вы какой-нибудь интересный репозиторий на просторах интернета и решаете склонировать, чтобы поработать с ним локально, а может и внести свой вклад в open source.

Ничего не предвещает проблем, но тут неожиданно возникает ошибка:
```bash  {hl_lines=["10-12"]}
akorolev.dev repositories
$ git clone git@github.com:akorolevdev/project-with-invalid-path-error.git
Cloning into 'project-with-invalid-path-error'...
remote: Enumerating objects: 1884, done.
remote: Counting objects: 100% (963/963), done.
remote: Compressing objects: 100% (485/485), done.
remote: Total 1884 (delta 515), reused 738 (delta 390), pack-reused 921
Receiving objects: 100% (1884/1884), 627.87 KiB | 1.61 MiB/s, done.
Resolving deltas: 100% (778/778), done.
error: invalid path 'licenses/LICENSE.third-party-project.txt '
fatal: unable to checkout working tree
warning: Clone succeeded, but checkout failed.
You can inspect what was checked out with 'git status'
and retry with 'git restore --source=HEAD :/'

akorolev.dev repositories
$ cd project-with-invalid-path-error/ && ls -la
total 8
drwxr-xr-x 1 akorolev 1049089 0 Mar 18 07:50 ./
drwxr-xr-x 1 akorolev 1049089 0 Mar 18 07:50 ../
drwxr-xr-x 1 akorolev 1049089 0 Mar 18 07:50 .git/
```
Клонирование произошло успешно, но рабочая директория не содержит ни одного файла исходного кода.
Вывод команды содержит набор warning, error и fatal сообщений.

## Причины возникновения
Все из-за файла с текстом лицензии одной из зависимостей, с которым нам вряд ли понадобится работать.
Просто по какой-то причине имя этого файла содержит пробел в самом конце, что является нарушением [соглашения об именовании в Windows](https://learn.microsoft.com/en-us/windows/win32/fileio/naming-a-file).

<figure class="text-start">
  <blockquote class="blockquote">
<mark>Do not end a file</mark> or directory <mark>name with a space</mark> or a period.
Although the underlying file system may support such names, the Windows shell and user interface does not.
However, it is acceptable to specify a period as the first character of a name. For example, ".temp".
  </blockquote>
</figure>

При попытке построить рабочее дерево по коммиту из HEAD, Git обнаруживает это несоответствие с операционной системой.
А при выполнении `git status` произойдет вывод длинного списка не созданных файлов в состоянии `deleted`.

## Решение
В [описании параметров конфигурации Git](https://git-scm.com/docs/git-config) можно найти специальную опцию `core.protectNTFS`.
На Windows она включена по умолчанию, а для Mac OS предусмотрена аналогичная `core.protectHFS`.
Отключаем и делаем сброс состояния, который сопровождается сообщением об успехе.
{{% bs/alert warning %}}
В сети часто предлагают отключить данную проверку на уровне глобальной конфигурации.

Но я советую делать это локально, чтобы менять поведение только по мере необходимости и быть в курсе возникновения таких ошибок в новых проектах.
{{% /bs/alert %}}

```bash  {hl_lines=[2,6]}
akorolev.dev project-with-invalid-path-error (main)
$ git config --local core.protectNTFS false

akorolev.dev project-with-invalid-path-error (main)
$ git reset HEAD --hard
HEAD is now at 2e260e3 Update README.md
```
Но не все так гладко, как хотелось бы. Защита, которую мы отключили, активна не просто так.

Если проверить статус, то будет замечен и побочный эффект — файл в рабочей директории создан с другим именем.

```bash  {hl_lines=[9,13]}
akorolev.dev project-with-invalid-path-error (main)
$ git status
On branch main
Your branch is up to date with 'origin/main'.

Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        deleted:    licenses/LICENSE.third-party-project.txt

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        licenses/LICENSE.third-party-project.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

Далее возможны различные варианты, в зависимости от конкретной ситуации. Можно зафиксировать такое переименование, как способ решения проблемы для Windows.
Если это полностью ваш личный проект или проблема локализована в отдельной feature-ветке, то можно задаться вопросом корректировки существующей истории через `filter-repo` или `filter-branch`.

Но в моем случае проект был open source, он активно обновляется (т.е. сообщество может не считать это проблемой), а нам этот файл не нужен в принципе для работы с кодом.

Поэтому в качестве подходящего варианта можно просто прибегнуть к [`sparse-checkout`](https://git-scm.com/docs/git-sparse-checkout), через который исключить полностью директорию _licenses_ в нашей рабочей копии.

{{% bs/alert secondary %}}
Источник изображения в заголовке Unsplash. Автор [Zach Lezniewicz](https://unsplash.com/@zachlez).
{{% /bs/alert %}}
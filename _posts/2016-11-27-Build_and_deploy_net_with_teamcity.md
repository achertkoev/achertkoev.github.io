---
layout: post
title: Разворачиваем ASP.NET приложения и windows сервисы с использованием TeamCity
---

В сегодняшней небольшой заметке хотел бы поделиться информацией о том, как корректно настроить Build step’ы в TeamCity для реализации auto deploy разработанных приложений на remote сервер.

## Разворачиваем ASP.NET (Web API, MVC)

Я разворачиваю свои приложения с использованием [Web Deploy](https://www.iis.net/downloads/microsoft/web-deploy), который предварительно придётся настроить на remote сервере. Подробнее об установке и настройке можно посмотреть [здесь](https://www.iis.net/learn/install/installing-publishing-technologies/installing-and-configuring-web-deploy-on-iis-80-or-later).

Далее, добавляем новый Build step в TeamCity со следующими настройками (о них далее):

![mt command](/images/post/build_deploy_teamcity_solution.png){:class="img-responsive"}

_Solution file path*_ - путь до разворачиваемого приложения из solution’a (в моём случае solution содержит несколько проектов и для deploy’a каждого используется свой собственный build step).

_Configuration_ - наименование конфигурации вашего проекта, например, для [трансформации конфигурационных файлов](https://msdn.microsoft.com/ru-ru/library/dd465318(v=vs.100).aspx), таких как web.config и app.settings.

_Command line parameters_ - [перечень параметров](https://msdn.microsoft.com/en-us/library/microsoft.teamfoundation.build.workflow.activities.msbuild(v=vs.120).aspx), которые будут использованы в качестве аргументов при запуске msbuild.exe (сам процесс build & deploy выполняется именно этим приложением).

Непосредственно за deploy отвечают следующие из них:

Указание о необходимости deploy после успешного build:
```
/p:DeployOnBuild=true1
```

Способ разворачивания:
```
/p:DeployTarget=MSDeployPublish
/p:AllowUntrustedCertificate=True
```

Настройки Web Deploy после конфигурации:
/p:MsDeployServiceUrl=https://62.56.35.72:8172/MsDeploy.axd
/p:Username=Deployuser
/p:Password=Deployuserpassword

Наименование IIS узла на remote сервере:
/p:DeployIisAppPath=testapi.your-company-site.ru

Если всё было выполнено корректно, то результат build log будет выглядеть следующим образом:

![mt command](/images/post/build_deploy_solution_service_output.png){:class="img-responsive"}

## Разворачиваем windows сервисы

При deploy сервисов (windows служб) всё несколько сложнее, т.к. сами сервисы на remote сервере потребуется останавливать и перезапускать. Для этого я решил [отредактировать](https://github.com/ialekseev/DeployRemoteServiceExample/blob/master/deploy_remote_service.ps1) небольшой скрипт написанный на PowerShell.

Для начала добавляем новый Build step в TeamCity со следующими настройками (о них далее):

![mt command](/images/post/build_deploy_teamcity_service.png){:class="img-responsive"}

Runner type - выбираем исполнение PowerShell скрипта

Script - откуда брать содержимое скрипта (из script source или отдельного файла)

Script source - непосредственно сам скрипт.

Я не буду приводить весь скрипт (он [доступен на github](https://github.com/FSou1/DeployWindowsServiceToRemoteServiceExample/blob/master/deploy_windows_service.ps1)), опишу лишь алгоритм:
1. Инициализация переменных;
2. Подключение к remote серверу;
3. Поиск службы по имени на remote сервере с помощью Get-WmiObject (для этого предварительно может потребоваться [конфигурация WinRM](https://technet.microsoft.com/ru-ru/library/hh921475(v=ws.11).aspx) и использование [Get-Credential](https://msdn.microsoft.com/en-us/powershell/reference/5.1/microsoft.powershell.security/get-credential), если TeamCity запущена не от пользователя, у которого есть доступ);
4. Подключение к доступной сетевой папке, в которой хостится служба;
5. Сборка проекта с помощью msbuild;
6. Бекап текущего содержимого в отдельную папку (имя которой содержит время);
7. Остановка ранее найденной службы;
8. Копирование собранных исходников на remote сервер;
9. Запуск службы.










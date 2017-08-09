---
layout: post
title: Azure Monitor- возможности и ограничения
---

## Возможности

### Activity log

In progress.

### Metrics and Events

In progress.

### Log search

Благодаря log search и специализированному языку запросов присутствует возможность выполнять query/filter/aggregate применимо к логам и метрикам, а так же визуализировать итоговые результаты в требуемом виде (table, list, bar).

![log search](/images/post/oms-search-select.png){:class="img-responsive"}

### Autoscale

Горизонтальное масштабирование основанное на наступлении одного из двух видов условий:

1. На основании значения метрики- например, увеличить количество запущенных экземпляров того или иного ресурса в случае превышения использования % CPU или загрузки ОЗУ;
2. На основании расписания- например, если нагрузка в выходные дни снижается, то можно уменьшать и количество экземпляров до необходимого минимума;

![autoscale overview](/images/post/autoscale_overview_v4.png){:class="img-responsive"}

### Azure Service Health

Благодаря такому компоненту как Azure Service Health появляется возможность своевременно узнавать о технических работах и сбоях в инфраструктуре Azure, которые могут затронуть доступность развернутых в облаке сервисов и ресурсов. В качестве оповещений могут выступать уведомления по почте, телефону или webhook.

Nак же появляется возможность заблаговременно подготовиться к плановому обслуживанию Azure.

![azure service health](/images/post/azure-service-health-overview-7.png){:class="img-responsive"}

## Ограничения

Задержка при отправке уведомлений о наступлении событий- ограничением это можно назвать весьма условно, однако, оповещение о срабатывании того или иного правила будет получено в интервале от одной до пяти минут.

Отсутствует вертикальное масштабирование- на данный момент возможно изменение количества запущенных экземпляров (как в большую, так и в меньшую сторону), но не их вычислительных мощностей. Связано это с накладываемыми на такой вид масштабирования ограничениями- возможные отсутствие аппаратных средств и необходимость перезапуска.

Не все Azure services на данный момент помещают информацию в Azure Monitor, однако, это будет реализовано в будущем.

Срок хранения metrics и activity log entry ограничен- он составляет 30 и 90 дней соответственно (для большей продолжительности необходимо подключить Azure storage).

## Заключение

В заключении можно сказать что рассмотренный в данной статье инструмент позволяет:

1. Однозначно ответить кто, когда и какие действия выполнял применимо к ресурсам или инфраструктуре;
2. Получить, проанализировать и визуализировать информацию о выполненных действиях;
3. Добавлять набор правил и действий для реагирования при их наступлении;

Полезные ссылки:

1. [Overview of Azure Monitor](https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-overview-azure-monitor); 
2. [Overview of the Azure Activity Log](https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-overview-activity-logs); 
3. [Overview of metrics in Microsoft Azure](https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-overview-metrics);
4. [Overview of autoscale in Microsoft Azure](https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-overview-autoscale);
5. [Azure Service Health overview](https://docs.microsoft.com/en-us/azure/service-health/service-health-overview);
6. [Supported Resource Types through Azure Resource Health](https://docs.microsoft.com/en-us/azure/service-health/resource-health-checks-resource-types);
7. [Azure Monitor videos](https://azure.microsoft.com/ru-ru/resources/videos/index/?services=monitor);



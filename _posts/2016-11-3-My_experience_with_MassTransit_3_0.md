---
layout: post
title: Опыт использования MassTransit 3.0
---

MassTransit это open source библиотека, разработанная на языке C# для .NET платформы, упрощающая работу с шиной данных, которая используется при построении распределенных приложений и реализации SOA (service oriented architecture).

В качестве message broker могут выступать RabbitMq, Azure Service Bus или In-Memory менеджер (в случае с In-Memory область видимости ограничивается процессом, в котором проинициализирован экземпляр).

Содержание:

1. Команды и события
1.1. Команды
1.2. События
2. Контракты сообщений
3. Роутинг
3.1. Exchange
3.2. Формат сообщения
4. Консьюмеры (Consumer)
5. Конфигурация контейнера DI
6. Наблюдатели (Observer)
7. Новое в MassTransit 3.0
8. Заключение

## Команды и события

В библиотеке заложено 2 основных типа сообщений: команды и события.

### Команды

Сигнализируют о необходимости выполнить некое действие. Для наиболее содержательного наименования команды желательно использовать структуру глагол + существительное:
EstimateConnection, SendSms, NotifyCustomerOrderProcessed.

Работа с командами осуществляется с помощью метода Send (интерфейса ISendEndpoint) и указания получателя endpoint (очереди):

```c#
private static async Task SendSmsCommand(IBus busControl)
{
   var command = new SendSmsCommand
   {
       CommandId = Guid.NewGuid(),
       PhoneNumber = "89031112233",
       Message = "Thank you for your reply"
   };

   var endpoint = await busControl.GetSendEndpoint(AppSettings.CommandEndpoint);
   await endpoint.Send(command);
}
```

### События

Сигнализируют о случившемся событии, которое может быть интересно некоему набору подписчиков (паттерн Observer), которые на эти события реагируют, например: ConnectionEstimated, CallTerminated, SmsSent, CustomerNotified. 

Работа с событиями осуществляется с помощью метода Publish (интерфейса IPublishEndpoint).

В терминологии заложено и основное различие этих типов сообщений — команда доставляется единственному исполнителю (дабы избежать дублирования выполнения):

![mt command](/images/post/mt_command.png){:class="img-responsive"}

Изображение из статьи MassTransit Send vs. Publish

В то время как событие ориентировано на оповещение n-подписчиков, каждый из которых реагирует на случившееся событие по-своему.

![mt event](/images/post/mt_event.png){:class="img-responsive"}

Изображение из статьи MassTransit Send vs. Publish

Иными словами, при запущенных n-консьюмеров (от англ. consumer — потребитель, обработчик), обрабатывающих команду, после её публикации только один из них получит сообщение о ней, в то время как сообщение о событии получит каждый.

## Контракты сообщений

Согласно документации MassTransit, при объявлении контрактов сообщений рекомендуется прибегать к интерфейсам:

```c#
public interface ISendSms {
	Guid CommandId { get; }
	string PhoneNumber { get; }
	string Message { get; }
}
```

Как упоминалось ранее, отправка команд должна осуществляться исключительно с помощью метода Send (интерфейса IBus) и указания адресата (endpoint).

```c#
public interface ISmsSent {
	Guid EventId { get; }
	DateTime SentAtUtc { get; }	
}
```

События отправляются с помощью метода Publish.

## Роутинг

Как распределение сообщений по exchange, так и выбор консьюмеров (о них в этой статье будет рассказано чуть позже) для обработки базируются на runtime типах этих сообщений,- в наименовании используются namespace и имя типа, в случае с generic имя родительского типа и перечень аргументов.

### Exchange

При конфигурации receive endpoint‘a (подключении ранее зарегистрированных консьюмеров) в случае использования в качестве канала доставки сообщений RabbitMq на основании указанных к обработке консьюмерами типов сообщений формируются наименования требуемых exchange, в которые затем эти самые сообщения и будут помещаться.

Аналогичные действия на этапе конфигурации send endpoint‘a выполняются и для команд, для отправки которых также требуются собственные exchange.

На изображении можно увидеть созданные в рамках моего сценария exchange:

![mt exchange](/images/post/mt_exchange.png){:class="img-responsive"}

В том случае, если конфигурируя receive endpoint мы указываем наименование очереди:

```c#
cfg.ReceiveEndpoint(host, "play_with_masstransit_queue", e => e.LoadFrom(container));
```

то в привязках exchange сообщений можно будет увидеть следующую картину:

![mt_config_queue](/images/post/mt_config_queue.png){:class="img-responsive"}

Итоговый путь сообщения, тип которого имплементирует ISmsEvent, будет иметь следующий вид: 

![mt_queue_flow](/images/post/mt_queue_flow.png){:class="img-responsive"}

Если же конфигурация осуществляется без указания очереди:

```c#
cfg.ReceiveEndpoint(host, e=> e.LoadFrom(container));
```

То имена для последнего exchange и очереди формируются автоматически, а по завершению работы они будут удалены: 

![mt_without_queue_flow](/images/post/mt_without_queue_flow.png){:class="img-responsive"}

### Формат сообщения

Говоря о формате сообщения, хотелось бы подробнее остановиться на наименовании (или messageType). За его формирование (заголовков urn:message:) ответственна функция GetUrnForType(Type type). Для примера я добавил для команды ISendSms наследование от ICommand и generic тип:

```c#
public interface ICommand<T>
{
}

public interface ISendSms<T> : ICommand<T>
{
   Guid CommandId { get; }
   string PhoneNumber { get; }
   string Message { get; }
}

class SendSmsCommand : ISendSms<string>
{
   public Guid CommandId { get; set; }
   public string PhoneNumber { get; set; }
   public string Message { get; set; }
}
```

Сформированное сообщение в таком случае будет содержать следующее значение в поле messageType (на основании которого после получения сообщения затем и выбирается ответственный за обработку консьюмер):

```javascript
"messageType": [
    "urn:message:PlayWithMassTransit30.Extension:SendSmsCommand",
    "urn:message:PlayWithMassTransit30.Contract.Command:ISendSms[[System:String]]",
    "urn:message:PlayWithMassTransit30.Contract.Command:ICommand[[System:String]]"
]
```

Кроме messageType сообщение содержит информацию о host, которым оно было отправлено:

```javascript
"host": {
    "machineName": "DESKTOP-SI9OHUR",
    "processName": "PlayWithMassTransit30.vshost",
    "processId": 1092,
    "assembly": "PlayWithMassTransit30",
    "assemblyVersion": "1.0.0.0",
    "frameworkVersion": "4.0.30319.42000",
    "massTransitVersion": "3.4.1.808",
    "operatingSystemVersion": "Microsoft Windows NT 6.2.9200.0"
}
```

Значимую часть payload:

```javascript
"message": {
    "commandId": "7388f663-82dc-403a-8bf9-8952f2ff262e",
    "phoneNumber": "89031112233",
    "message": "Thank you for your reply"
}
```

и иные служебные поля и заголовки.

## Консьюмеры (Consumer)

Консьюмер — это класс, который обрабатывает один или несколько типов сообщений, указываемых при объявлении в наследовании интерфейса IConsumer, где T это тип обрабатываемого данным консьюмером сообщения.

Пример консьюмера, обрабатывающего команду ISendSms и публикующего событие ISmsSent:

```c#
public class SendSmsConsumer : IConsumer<ISendSms<string>>
{
   public SendSmsConsumer(IBusControl busControl)
   {
       _busControl = busControl;
   }

   public async Task Consume(ConsumeContext<ISendSms<string>> context)
   {
       var message = context.Message;

       Console.WriteLine($"[IConsumer<ISendSms>] Send sms command consumed");
       Console.WriteLine($"[IConsumer<ISendSms>] CommandId: {message.CommandId}");
       Console.WriteLine($"[IConsumer<ISendSms>] Phone number: {message.PhoneNumber}");
       Console.WriteLine($"[IConsumer<ISendSms>] Message: {message.Message}");

       Console.Write(Environment.NewLine);
       Console.WriteLine("Публикация события: Смс сообщение отправлено");
       await _busControl.SmsSent(DateTime.UtcNow);
   }

   private readonly IBus _busControl;
}
```

После того, как мы получили команду на отправку смс сообщения и выполнили требуемые действия, мы формируем и отправляем событие о том, что смс доставлено.

Код отправки сообщений я вынес в отдельный Extension класс над IBusControl, там же находится и имплементация самих сообщений:

```c#
public static class BusExtensions
{
   /// <summary>
   /// Отправка смс сообщения
   /// </summary>
   /// <param name="bus"></param>
   /// <param name="host"></param>
   /// <param name="phoneNumber"></param>
   /// <param name="message"></param>
   /// <returns></returns>
   public static async Task SendSms(
       this IBus bus, Uri host, string phoneNumber, string message
   )
   {
       var command = new SendSmsCommand
       {
           CommandId = Guid.NewGuid(),
           PhoneNumber = phoneNumber,
           Message = message
       };

       await bus.SendCommand(host, command);
   }

   /// <summary>
   /// Публикация события об отправке смс сообщения
   /// </summary>
   /// <param name="bus"></param>
   /// <param name="sentAtUtc"></param>
   /// <returns></returns>
   public static async Task SmsSent(
       this IBus bus, DateTime sentAtUtc
   )
   {
       var @event = new SmsSentEvent
       {
           EventId = Guid.NewGuid(),
           SentAtUtc = sentAtUtc
       };

       await bus.PublishEvent(@event);
   }

   /// <summary>
   /// Отправка команды
   /// </summary>
   /// <typeparam name="T"></typeparam>
   /// <param name="bus"></param>
   /// <param name="address"></param>
   /// <param name="command"></param>
   /// <returns></returns>
   private static async Task SendCommand<T>(this IBus bus, Uri address, T command) where T : class
   {
       var endpoint = await bus.GetSendEndpoint(address);
       await endpoint.Send(command);
   }

   /// <summary>
   /// Публикация события
   /// </summary>
   /// <typeparam name="T"></typeparam>
   /// <param name="bus"></param>
   /// <param name="message"></param>
   /// <returns></returns>
   private static async Task PublishEvent<T>(this IBus bus, T message) where T : class
   {
       await bus.Publish(message);
   }
}

class SendSmsCommand : ISendSms<string>
{
   public Guid CommandId { get; set; }
   public string PhoneNumber { get; set; }
   public string Message { get; set; }
}

class SmsSentEvent : ISmsSent
{
   public Guid EventId { get; set; }
   public DateTime SentAtUtc { get; set; }
}
```

На мой взгляд, данное решение вполне удачно позволяет отделить код бизнес-логики от деталей реализации межсистемного (компонентного) взаимодействия и инкапсулировать их в одном месте.

## Конфигурация контейнера DI

На данный момент MassTransit предоставляет возможность использовать следующие популярные контейнеры:

1. Autofac;
2. Ninject;
3. StructureMap;
4. Unity;
5. Castle Windsor.

В случае с UnityContainer потребуется установить nuget package MassTransit.Unity, после чего станет доступен метод расширения LoadFrom:

```c#
public static class UnityExtensions
{
    public static void LoadFrom(this IReceiveEndpointConfigurator configurator, IUnityContainer container);
}
```

Пример использования выглядит следующим образом:

```c#
public static IBusControl GetConfiguredFactory(IUnityContainer container)
{
   if (container == null)
   {
       throw new ArgumentNullException(nameof(container));
   }

   var control = Bus.Factory.CreateUsingRabbitMq(cfg => {
       var host = cfg.Host(AppSettings.Host, h => { });


       // cfg.ReceiveEndpoint(host, e => e.LoadFrom(container));
       cfg.ReceiveEndpoint(host, "play_with_masstransit_queue", e => e.LoadFrom(container));
   });

   control.ConnectConsumeObserver(new ConsumeObserver());
   control.ConnectReceiveObserver(new ReceiveObserver());
   control.ConnectConsumeMessageObserver(new ConsumeObserverSendSmsCommand());
   control.ConnectSendObserver(new SendObserver());
   control.ConnectPublishObserver(new PublishObserver());

   return control;
}
```

В качестве срока жизни консьюмеров в контейнере документация предлагает использовать ContainerControlledLifetimeManager().

## Наблюдатели (Observer)

Для мониторинга процесса обработки сообщений доступно подключение наблюдателей (Observer). Для этого MassTransit предоставляет следующий набор интерфейсов для обработчиков:

1. IReceiveObserver- срабатывает сразу же после получения сообщения и создания RecieveContext;
2. IConsumeObserver — срабатывает после создания ConsumeContext;
3. IConsumeMessageObserver<T> — для наблюдения за сообщениями типа T, в методах которого будет доступно строго-типизированное содержимое сообщения;
4. ISendObserver — для наблюдения за отправляемыми командами;
5. IPublishObserver — для наблюдения за отправляемыми событиями.

Для каждого из них интерфейс IBusControl предоставляет собственный метод подключения, выполнение которого должно быть осуществлено непосредственно перед IBusControl.Start().

В качестве примера далее представлена реализация ConsumeObserver:

```c#
public class ConsumeObserver : IConsumeObserver
{
   public Task PreConsume<T>(ConsumeContext<T> context) where T : class
   {
       Console.WriteLine($"[ConsumeObserver] PreConsume {context.MessageId}");
       return Task.CompletedTask;
   }

   public Task PostConsume<T>(ConsumeContext<T> context) where T : class
   {
       Console.WriteLine($"[ConsumeObserver] PostConsume {context.MessageId}");
       return Task.CompletedTask;
   }

   public Task ConsumeFault<T>(ConsumeContext<T> context, Exception exception) where T : class
   {
       Console.WriteLine($"[ConsumeObserver] ConsumeFault {context.MessageId}");
       return Task.CompletedTask;
   }
}
```

Я не буду приводить код каждого из консьюмеров, т.к. по принципу работы и структуре они схожи. Имплементацию каждого из них можно посмотреть в документации или в исходниках на Github.

Итоговый pipeline получения команды на отправку смс сообщения, её обработки и публикации события о её успешном выполнении выглядит следующим образом:

![mt_result](/images/post/mt_result.png){:class="img-responsive"}

## Новое в MassTransit 3.0

С изменениями, которые коснулись новой версии библиотеки, вы можете ознакомиться в 2-х обзорных статьях автора библиотеки Chris Patterson’а на страницах его блога: MassTransit 3 API Changes и MassTransit v3 Update.

## Заключение

Здесь должно было быть сравнение наиболее популярных библиотек для работы с очередями сообщений, однако, я решил оставить это для отдельной статьи.

Надеюсь, мне удалось провести для вас поверхностное знакомство с библиотекой MassTransit, за гранью которого ещё остаются такие интересные вещи, как транзакционность, персистентность (интеграция с NHibernate, MondoDb, EntityFramework), планировщик отправки сообщений (интеграция с Quartz), state machine (Automatonymous и Saga), логирование (Log4Net, NLog), многопоточность и многое другое.

Исходные коды примеров доступны на Github.

Используемые материалы:
1. Документация MassTransit.




Проекты Web API 2 конфигурируются с помощью OWIN интерфейса IAppBuilder, который призван помочь построить pipeline обработки входящего запроса.

![owin pipeline](/images/post/owin-pipeline.png){:class="img-responsive"}

На изображении выше виден жизненный цикл запроса,- он проходит по всем компонентам цепочки, затем попадает в Web API (что также является отдельным компонентом) и возвращается обратно, формируя или декорируя ответ от сервера.

Для того, чтобы иметь единый контейнер зависимостей нам потребуется создавать его явно в начале обработки запроса и уничтожать по завершению:

1. Начало обработки запроса;
2. Создание контейнера;
3. Использование контейнера в Middleware;
4. Использование контейнера в Web API;
5. Уничтожение контейнера;
6. Завершение обработки запроса.

Для этого нам достаточно сконфигурировать контейнер, зарегистрировать его в Web API (посредством DependencyResolver):

```c#
// Configure our parent container
var container = UnityConfig.GetConfiguredContainer();
            
// Pass our parent container to HttpConfiguration (Web API)
var config = new HttpConfiguration {
    DependencyResolver = new UnityDependencyResolver(container)
};

WebApiConfig.Register(config);
```

написать собственный Middleware, который будет создавать дочерний контейнер:

```c#
public class UnityContainerPerRequestMiddleware : OwinMiddleware
{
    public UnityContainerPerRequestMiddleware(OwinMiddleware next, IUnityContainer container) : base(next)
    {
        _next = next;
        _container = container;
    }

    public override async Task Invoke(IOwinContext context)
    {
        // Create child container (whose parent is global container)
        var childContainer = _container.CreateChildContainer();

        // Set created container to owinContext (to become available at other places using OwinContext.Get<IUnityContainer>(key))
        context.Set(HttpApplicationKey.OwinPerRequestUnityContainerKey, childContainer);

        await _next.Invoke(context);

        // Dispose container that would dispose each of container's registered service
        childContainer.Dispose();
    }

    private readonly OwinMiddleware _next;
    private readonly IUnityContainer _container;
}
```

и использовать его в других Middleware’ах (в моей реализации я сохраняю контейнер в глобальном OwinContext с помощью context.Set, который передаётся в каждый следующий middleware и получаю его с помощью context.Get):

```c#
public class CustomMiddleware : OwinMiddleware
{
    public CustomMiddleware(OwinMiddleware next) : base(next)
    {
        _next = next;
    }

    public override async Task Invoke(IOwinContext context)
    {
        // Get container that we set to OwinContext using common key
        var container = context.Get<IUnityContainer>(HttpApplicationKey.OwinPerRequestUnityContainerKey);

        // Resolve registered services
        var sameInARequest = container.Resolve<SameInARequest>();

        await _next.Invoke(context);
    }

    private readonly OwinMiddleware _next;
}
```

На этом можно было бы закончить, если бы не одно НО.

## Проблема

Middleware Web API внутри себя имеет свой собственный цикл обработки запроса, который выглядит следующим образом:

1. Запрос попадает в HttpServer для начала обработки HttpRequestMessage и передачи его в систему маршрутизации;
2. HttpRoutingDispatcher извлекает данные из запроса и с помощью таблицы Route’ов определяет контроллер, ответственный за обработку;
3. В HttpControllerDispatcher создаётся определённый ранее контроллер и вызывается метод обработки запроса с целью формирования HttpResponseMessage.

За создание контроллера отвечает следующая строка в DefaultHttpControllerActivator:

```c#
IHttpController instance = (IHttpController)request.GetDependencyScope().GetService(controllerType);
```

Основное содержимое метода GetDependencyScope:

```c#
public static IDependencyScope GetDependencyScope(this HttpRequestMessage request) {
    // …

    IDependencyResolver dependencyResolver = request.GetConfiguration().DependencyResolver;
    result = dependencyResolver.BeginScope();

    request.Properties[HttpPropertyKeys.DependencyScope] = result;
    request.RegisterForDispose(result);    

    return result;
}
```

Из него видно, что Web API запрашивает DependencyResolver, который мы для него зарегистрировали в HttpConfiguration и с помощью dependencyResolver.BeginScope() создаёт дочерний контейнер, в рамках которого уже и будет создан экземпляр ответственного за обработку запроса контроллера.

Для нас это значит следующее: контейнер, который мы используем в наших Middleware’ах и в Web API не являются одними и теми же,- больше того, они находятся на одном уровне вложенности, где глобальный контейнер- их общий родитель, т.е.:

1. Глобальный контейнер;
2. Контейнер, созданный в UnityContainerPerRequestMiddleware;
3. Контейнер, созданный в Web API.

Для Web API это выглядит вполне логичным в том случае, когда оно является единственным местом обработки запроса,- контейнер создается вначале и уничтожается в конце (это ровно то, чего мы стараемся добиться). 

Однако, в данный момент Web API является лишь одним из звеньев в pipeline, а значит от создания собственного контейнера придется отказаться,- нашей задачей является переопределить данное поведение и указать контейнер, в рамках которого Web API требуется создавать контроллеры и Resolve’ить зависимости. 

## Решение

Для решения выше поставленной проблемы мы можем реализовать собственный IHttpControllerActivator, в методе Create которого будем получать созданный ранее контейнер и уже в рамках него Resolve’ить зависимости:

```c#
public class ControllerActivator : IHttpControllerActivator
{
    public IHttpController Create(
        HttpRequestMessage request,
        HttpControllerDescriptor controllerDescriptor,
        Type controllerType
    )
    {
        // Get container that we set to OwinContext using common key
        var container = request.GetOwinContext().Get<IUnityContainer>(HttpApplicationKey.OwinPerRequestUnityContainerKey);

        // Resolve requested IHttpController using current container
        // prevent DefaultControllerActivator's behaviour of creating child containers 
        var controller = (IHttpController)container.Resolve(controllerType);

        // Dispose container that would dispose each of container's registered service
        // Two ways of disposing container:
        // 1. At UnityContainerPerRequestMiddleware, after owin pipeline finished (WebAPI is just a part of pipeline)
        // 2. Here, after web api pipeline finished (if you do not use container at other middlewares) (uncomment next line)
        // request.RegisterForDispose(new Release(() => container.Dispose()));

        return controller;
    }
}
```

Для того, чтобы использовать его в Web API всё что нам остаётся, это заменить стандартный HttpControllerActivator в конфигурации:

```c#
var config = new HttpConfiguration {
    DependencyResolver = new UnityDependencyResolver(container)
};

// Use our own IHttpControllerActivator implementation 
// (to prevent DefaultControllerActivator's behaviour of creating child containers per request)
config.Services.Replace(typeof(IHttpControllerActivator), new ControllerActivator());

WebApiConfig.Register(config);
```

Таким образом, мы получаем следующий механизм работы с нашим единым контейнером:

Начало обработки запроса;

Создание дочернего контейнера от глобального;

```c#
var childContainer = _container.CreateChildContainer();
```

Присваивание контейнера в Owin context:

```c#
context.Set(HttpApplicationKey.OwinPerRequestUnityContainerKey, childContainer);
```

Использование контейнера в других Middleware’ах;

```c#
var container = context.Get<IUnityContainer>(HttpApplicationKey.OwinPerRequestUnityContainerKey);
```

Использование контейнера в Web API;

Получение контроллера из Owin context:

```c#
var container = request.GetOwinContext().Get<IUnityContainer>(HttpApplicationKey.OwinPerRequestUnityContainerKey);
```

Создание контроллера на основе этого контейнера:

```c#
var controller = (IHttpController)container.Resolve(controllerType);
```

Уничтожение контейнера;

```c#
childContainer.Dispose();
```

Завершение обработки запроса.

## Результат

Конфигурируем зависимости в соответствии с требуемыми нам их жизненными циклами:

```c#
public static void RegisterTypes(IUnityContainer container)
{
    // ContainerControlledLifetimeManager - singleton's lifetime
    container.RegisterType<IAlwaysTheSame, AlwaysTheSame>(new ContainerControlledLifetimeManager());
    container.RegisterType<IAlwaysTheSame, AlwaysTheSame>(new ContainerControlledLifetimeManager());

    // HierarchicalLifetimeManager - container's lifetime
    container.RegisterType<ISameInARequest, SameInARequest>(new HierarchicalLifetimeManager());

    // TransientLifetimeManager (RegisterType's default) - no lifetime
    container.RegisterType<IAlwaysDifferent, AlwaysDifferent>(new TransientLifetimeManager());
}
```

1. ContainerControlledLifetimeManager - создание единственного экземпляра в рамках приложения;
2. HierarchicalLifetimeManager - создание единственного экземпляра в рамках контейнера (где мы добились того, что контейнер единый в рамках HTTP запроса);
3. TransientLifetimeManager - создание экземпляра при каждом обращении (Resolve).

![unity-per-request-middleware-result](/images/post/unity-per-request-middleware-result.png){:class="img-responsive"}

В изображении выше отображены GetHashCode’ы зависимостей в разрезе нескольких HTTP запросов, где:

1. AlwaysTheSame - singleton объект в рамках приложения;
2. SameInARequest - singleton объект в рамках запроса;
3. AlwaysDifferent - новый экземпляр для каждого Resolve.

Исходники доступны на [**GitHub**](https://github.com/FSou1/UnityPerRequestMiddleware).

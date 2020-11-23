**EventManagement** ponúka abstrakciu nad message-brokermi (RabbitMQ, ..) pre použítie v .Net Core 3.1+ aplikáciach. Jeho implementácia je rozdelená do nasledujúcich troch nugetov: 
* EventManagement
    * Obsahuje abstrakciu a definíciu hlavných funkcionalít.   
* EventManagement.RabbitMQ
    * Implementácia EventManagement-u pre RabbitMQ
* EventManagement.DependencyInjection
    * Extension metódy pre pridanie EM do Microsoft Dependency Injection kontajnera.
## Základné nastavenie


### ASP.NET Core

Pre použitie v ASP.NET Core aplikácii je vhodné spolu s konkrétnou implementáciu EventManagement-u nainštalovať aj EventManagement.DependencyInjection nuget balíček. Vdaka nemu je možné potrebné služby pridať nasledovne: 

```csharp
public void ConfigureServices(IServiceCollection services)
{
  services.AddMvc();

     //Vytvorenie options
     var eventManagementOptions = new EventManagementOptions()
    {
        //Je možné použiť vlastný (de) serializer správ
        Serializer = new Serializer()
    };

    //Pridanie základných funkcionalít
    services.AddEventManagement<BaseEvent>(options: eventManagementOptions)
         //Pridanie implementácie EM pre RabbitMQ
         .AddRabbitMQ(new RabbitMQOptions()
         {
            //Exchange do ktorého sa správy majú zverejňovať
            PublishingExchange = "TestExchange",
            //Počet opakovaní pri neúspešnom pripojení/publishnutí
            RetryCount = 5,
            //Definuje akciu ktorá má nastať ak spracovanie prijatej správy nebude úspešné (Handler vyhodní výnimku)
            ProccesMessageFailsHandler = (ex,model, deliveryTag) => model.BasicReject(deliveryTag, false)
            ConnectionOptions = new ConnectionOptions()
            {
                HostName = "localhost",
                Username = "guest",
                Password = "guest",
                PasswordKey = "",
                Port = 5672,
                VirtualHost = "/",
            }
         });
}
```

## Základy

EventManagement definuje dve základné operácie: Publish a Subscribe 

#### Publish

Správu je možné odoslať pomocou metódy PublishAsync ktorú definuje `IEventBus<TBaseEvent>`
 - `TBaseEvent` je základný typ správy. 
    - Definuje sa v pri registrácii EM do DI 
    - Každá správa odoslaná cez EventBus musí byť podtypom `TBaseEvent`.
    - Pokiaľ programátor chce povoliť odosielanie akejkoľvek správy, odpočúča sa použiť `IEventBus<object>`

Príklad:
Ako prvé je potrebné vytvoriť správu:
```csharp
public class CustomEvent : BaseEvent 
{
    public string Name { get; set; }
}
```
Následne, odoslanie správy v metóde controllera: 

```csharp
    [ApiController]
    public class TestController : ControllerBase
    {
        private readonly IEventBus<BaseEvent> _eventBus;
        //Získanie inštancie EvenBus-u z DI
        public TestController(IEventBus<BaseEvent> eventBus)
        {
            _eventBus = eventBus;
        }

        [HttpPost]       
        public async Task<ActionResult> PostAsync()
        {
            //Vytvorenie správy
            var message = new CustomEvent()
            {
                Name = "Superman"
            };
            
            //Odoslanie správy
            await _eventBus.PublishAsync(message, "SomeCoolRoutingKey" );
            return Ok();
        }
    }
```
### Subscribe
Použijeme rovnaký model správy ako pri príklade na Publish správy.

Ako prvé vytvoríme Handler: 
```csharp
public class CustomEventHandler : IEventHandler<Event>
{
    private readonly ILogger _logger;
    
    public CustomEventHandler(ILogger<CustomEventHandler> logger)
    {
        _logger = logger;
    }

    public Task Handle(Event @event)
    {
        _logger.LogInformation($"Received message from event bus. Event contains Name: {@event.Name}");
        return Task.CompletedTask;
    }
}
```
Handler je taktiež potrebné zaregistrovať v DI:
```csharp
public void ConfigureServices(IServiceCollection services)
{
   services.AddTransient<CustomEventHandler>();
}
```
Nakoniec, zaregistrujem správu a jej handler pre prichádzajúce správy z message-brokera:

```csharp
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        var eventBus = app.ApplicationServices.GetRequiredService<IEventBus<BaseEvent>>();
        eventBus.Subscribe<CustomEvent, CustomEventEventHandler>("SomeCoolQueueName","SomeCoolRoutingKey");
    }
```

### Automatická registrácia handler-ov do DI kontajnera
Keďže správ a ich handler-ov môže aplikácia obsahovať veľké množstvo, je nepraktické registrovať každý nový handler do DI. Preto EventMagement ponúka možnosť zaregistrovania všetkých Handler-ov automaticky.

Pri registrácii je možné EventManagementu predať Assembly kde má Handlery hľadať. Ten následne nájde všetky typy, ktoré implementujú `IEventHandler<>` a zaregistruje ich.
```csharp
  //Predanie assemby, kde majú byť handler-y hľadané
            services.AddEventManagement<BaseEvent>(options: eventManagementOptions,
                                                   assemblies: typeof(Startup).Assembly)
                 .AddRabbitMQ(new RabbitMQOptions()
                 {
                     ConnectionOptions = new ConnectionOptions()
                     {
                         HostName = "localhost",
                         Username = "guest",
                         Password = "guest",
                         PasswordKey = "",
                         Port = 5672,
                         VirtualHost = "/",
                     },

                     PublishingExchange = "TestExchange",
                     RetryCount = 5,
                     ProccesMessageFailsHandler = (ex,model, deliveryTag) => model.BasicReject(deliveryTag, false)
                 });
```
### IReceivePipeline
EventManagement ponúka aj možnosť pripôsobenia akcíí ktoré sa udejú predtým ako správa dorazí až k handleru. Je to veľmi podobné middlewaru v ASP .NET aplikáciach. Každá takáto akcia musí implementovať `IReceivePipeline` interface.

```csharp
 public class ErrorHandlingPipelineAction : IReceivePipeline
    {
        private readonly ILogger _logger;

        public ErrorHandlingPipelineAction(ILogger logger)
        {
            _logger = logger;
        }

        public async Task HandleAsync(object message, ReceivePipelineDelegate next)
        {
            try
            {
                //Invoke the next action in the pipepline
                await next();
            }
            catch(Exception ex)
            {
                _logger.LogError(ex, "Handling message ended with error. Message: {@Message}", message);
            }            
        }
    }
```

Akcie v pipeline sa začnú poúžívať po ich pridaní do DI kontajnera. Volané su v rovnakom poradí, v akom boli pridané do kontajnera.
```csharp
    services.AddTransient<IReceivePipeline,LoggingPipelineAction>();
    services.AddTransient<IReceivePipeline, ErrorHandlingPipelineAction>();
```

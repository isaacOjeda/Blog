
# Introducción

En el desarrollo de software, construir aplicaciones modulares y flexibles es esencial para poder adaptarse a cambios de requerimientos y tecnologías. Los patrones de diseño como **Strategy** y **Factory** son herramientas poderosas que nos permiten crear soluciones robustas, facilitando la extensión y el mantenimiento del código. Estos patrones nos ayudan a diseñar componentes intercambiables, permitiendo que la lógica central (Core) de la aplicación permanezca independiente de los detalles específicos de cada implementación. En este artículo, exploraremos cómo emplear ambos patrones para gestionar diferentes proveedores de notificaciones de forma dinámica y escalable.

Implementaremos un sistema de notificaciones en el que `INotificationsStrategy` actúa como la interfaz central que define el contrato para enviar notificaciones, mientras que el **Factory Pattern** se encarga de construir la estrategia de notificación adecuada según el proveedor configurado. Además, utilizaremos el patrón **Decorator** para simplificar la inyección de dependencias, inspirándonos en la arquitectura de ASP.NET Core. Veremos cómo esta combinación de patrones nos permite intercambiar servicios de notificación de manera sencilla, hacerlos configurables y extender nuestro sistema sin modificar el Core de la aplicación.

> Código Fuente: [DevToPosts/StrategyFactoryPattern at main · isaacOjeda/DevToPosts · GitHub](https://github.com/isaacOjeda/DevToPosts/tree/main/StrategyFactoryPattern)

## Strategy Pattern

El **Strategy Pattern** es un [patrón de diseño de comportamiento](https://refactoring.guru/es/design-patterns/strategy) que permite definir una familia de algoritmos, encapsular cada uno y hacerlos intercambiables. Este patrón permite que el algoritmo varíe independientemente del cliente que lo utiliza.

Utilizando el Strategy Pattern, podemos aplicar diferentes estrategias de ejecución para un mismo comportamiento en función de configuraciones o condiciones externas. Este patrón es ideal cuando necesitamos flexibilidad en la lógica de negocio que se aplicará según diferentes condiciones sin modificar el código principal.

## Factory Pattern

El **Factory Pattern**  es un [patrón de diseño creacional](https://refactoring.guru/es/design-patterns/factory-method) que abstrae el proceso de creación de objetos, permitiendo a las clases delegar la responsabilidad de instanciar sus dependencias. La fábrica selecciona y devuelve la instancia correcta según las necesidades del cliente.

El factory elimina la **necesidad de conocer detalles de las clases concretas en el código del cliente,** centralizando la creación de objetos y reduciendo el acoplamiento. Este patrón es fundamental cuando el tipo de instancia a crear depende de configuraciones externas o de los datos en tiempo de ejecución.

Al combinar el Strategy Pattern con el Factory Pattern, se logra una arquitectura flexible y escalable, ideal para aplicaciones que deben manejar múltiples implementaciones de un servicio (por ejemplo, múltiples proveedores de notificaciones). La fábrica es responsable de crear la estrategia apropiada en tiempo de ejecución, permitiendo al cliente utilizar la estrategia correcta sin necesidad de lógica adicional.

## Ventajas, Desventajas y Consideraciones de Uso

### Ventajas

- **Flexibilidad y Extensibilidad**: Implementar patrones como Strategy y Factory permite integrar fácilmente distintas estrategias o servicios sin afectar el Core de la aplicación. Esto minimiza el riesgo de errores al realizar cambios y facilita la expansión de funcionalidades.
- **Separación de Responsabilidades**: Estos patrones promueven principios como el de Responsabilidad Única y la Inversión de Dependencias, lo que resulta en un código más organizado y fácil de mantener.
- **Facilidad para Pruebas**: Al estar desacopladas las implementaciones, resulta sencillo realizar pruebas unitarias y de integración mediante mocks o stubs de las interfaces.
- **Escalabilidad**: A medida que la aplicación crece, puedes añadir nuevas estrategias o servicios sin modificar la lógica del Core, lo cual facilita la evolución del sistema.

### Desventajas

- **Complejidad Añadida**: En proyectos pequeños o con lógica sencilla, la implementación de estos patrones puede resultar en una complejidad innecesaria, generando más trabajo del que realmente aporta.
- **Impacto en el Rendimiento**: La abstracción adicional puede ralentizar levemente la ejecución. En aplicaciones de alto rendimiento, es fundamental medir si el costo es justificable.
- **Curva de Aprendizaje**: Para desarrolladores sin experiencia en patrones de diseño, esta estructura puede ser difícil de comprender y mantener.

### Consideraciones para su uso

**Cuándo utilizar estos patrones**:

- Si el sistema interactúa con múltiples proveedores o servicios que podrían cambiar, o si el sistema debe ser flexible y fácil de extender, estos patrones son ideales.
- Cuando se usa una arquitectura modular (como Hexagonal o Clean Architecture) y se necesita proteger el Core de la aplicación de los detalles de implementación.

**Cuándo no utilizarlos**:

- En sistemas con una sola implementación fija que no necesita cambiarse.
- En aplicaciones simples, donde la abstracción adicional puede considerarse sobreingeniería.

# Implementando Strategy Pattern Con Factory

En este ejemplo, implementamos una infraestructura de notificaciones utilizando **Strategy** para definir varias maneras de enviar notificaciones (a través de **SendGrid** o **SMTP**) y **Factory** para decidir cuál estrategia utilizar en función de la configuración.
## Application Layer

No entraremos en detalle sobre la organización específica del código, pero estas estrategias suelen emplearse en arquitecturas como **Hexagonal** o **Clean Architecture**. Aunque no nos centraremos en cómo estructurar el código, sí es esencial entender qué elementos forman parte del "Core" de la aplicación y cuáles corresponden a detalles externos de los que queremos aislarnos.

Definir un contrato en el Core es fundamental. Este contrato determina las entradas y salidas esperadas sin depender de detalles de implementación específicos. De esta forma, el Core actúa como una "caja negra": no le interesa cómo se implementa cada detalle, solo que se cumpla el contrato definido. Esta flexibilidad hace que el código sea altamente intercambiable, y aquí es donde los patrones **Strategy** y **Factory** se vuelven especialmente útiles.

> **Nota 💡**: He implementado estas estrategias en diversos escenarios (almacenamiento, pasarelas de pago, procesamiento de documentos, etc.), y este [vídeo](https://www.youtube.com/watch?v=aBOrVRKK3fA&t=1s&ab_channel=JonoWilliams) me inspiró a escribir este artículo, ya que es una técnica valiosa en muchos de mis proyectos.

### Notifications > INotificationsStrategy

Definimos una interfaz `INotificationsStrategy`, que actúa como contrato para las distintas estrategias de notificación. Cada implementación de esta interfaz representará un proveedor de notificaciones distinto (por ejemplo, **SendGrid** y **SMTP**).

```csharp
public interface INotificationsStrategy
{
    Task<SendNotificationResponse> SendNotification(SendNotificationRequest request);
}
```

La interfaz asegura que cualquier servicio de notificación que implementemos tendrá el mismo método `SendNotification`. Esto permite que los clientes (la aplicación) usen cualquier servicio de notificación de manera intercambiable, asegurando consistencia en la estructura del código.

Es fundamental que `SendNotificationRequest` y `SendNotificationResponse` permanezcan en el Core de la aplicación y estén completamente desvinculadas de los detalles específicos de cada proveedor de notificaciones. Estas clases son las piezas de información que el Core necesita para realizar una notificación, y al definirlas de forma abstracta, nos aseguramos de que el Core pueda operar independientemente de los detalles específicos de cada implementación.

Dentro de `SendNotificationRequest`, podríamos incluir datos como el correo electrónico, número de teléfono u otros detalles del usuario que son comunes en las notificaciones, sin que estos datos dependan de un proveedor específico. Cada implementación puede necesitar configuraciones únicas, pero el Core debería mantenerse lo más aislado posible de estas particularidades.

Por ejemplo, una implementación podría requerir una API Key para acceder a su servicio, mientras que otra podría exigir autenticación mediante un usuario y clave secreta para obtener un token. Aunque estas diferencias pueden ser significativas, el Core no debe preocuparse por estos detalles de autenticación y configuración. Los patrones Strategy y Factory nos ayudan a gestionar estas variaciones, permitiéndonos abstraer esas diferencias y mantener el Core limpio y centrado en su función principal.

### Notifications > INotificationsFactory

La `INotificationsFactory` es la interfaz del factory. Define métodos para crear instancias de servicios de notificación basándose en el proveedor configurado.

```csharp
public interface INotificationsFactory
{
    INotificationsStrategy CreateNotificationService();
}
```

La fábrica encapsula la lógica de selección de proveedores. Esto significa que cualquier cambio en el método de selección no afectará a los consumidores de los servicios de notificación.

### Notifications > NotificationsConfig

`NotificationsConfig` es una clase de configuración que contiene las configuraciones de los distintos proveedores de notificaciones.

```csharp
public class NotificationsConfig
{
    public NotificationProvider Provider { get; set; }
    public SendGridConfig? SendGrid { get; set; }
    public SmtpConfig? Smtp { get; set; }
}
  
public class SendGridConfig
{
    //...
}
  
public class SmtpConfig
{
    //...
}
```

La propiedad `Provider` en `NotificationsConfig` define el proveedor predeterminado configurado en nuestra aplicación. Este proveedor se establece a través de `appsettings.json`, lo que facilita configurar y ajustar el servicio de notificaciones sin necesidad de modificar el código.

Si bien en este ejemplo usamos `appsettings.json`, el propósito del **Strategy Pattern** es permitir la intercambiabilidad de estrategias en tiempo de ejecución. Esto significa que, además de ser configurable en el archivo de configuración, la estrategia de notificación podría adaptarse dinámicamente según las preferencias del usuario, del tenant o cualquier otra lógica específica de la aplicación. Este enfoque añade flexibilidad al sistema y ayuda a personalizar el servicio para distintos contextos o necesidades en tiempo real.

## Infrastructure Layer

### Notifications > SendGrid

Implementamos `SendGridNotificationService`, que utiliza el proveedor de SendGrid para enviar notificaciones. Esta clase es una implementación concreta de `INotificationsStrategy`.

```csharp
public class SendGridNotificationService : INotificationsStrategy
{
    private readonly ILogger<SendGridNotificationService> _logger;
  
    public SendGridNotificationService(ILogger<SendGridNotificationService> logger)
    {
        _logger = logger;
    }
  
    public async Task<SendNotificationResponse> SendNotification(SendNotificationRequest request)
    {
        _logger.LogInformation("Using SendGrid to send notification");
  
        // SendGrid logic here
        await Task.Delay(0);
  
        return new SendNotificationResponse(true, "Notification sent successfully");
    }
}
```
### Notifications > Smtp

`SmtpNotificationsService` es otra implementación de `INotificationsStrategy`, que envía notificaciones utilizando un servidor SMTP.

```csharp
public class SmtpNotificationsService : INotificationsStrategy
{
    private readonly ILogger<SmtpNotificationsService> _logger;
  
    public SmtpNotificationsService(ILogger<SmtpNotificationsService> logger)
    {
        _logger = logger;
    }
  
    public async Task<SendNotificationResponse> SendNotification(SendNotificationRequest request)
    {
        _logger.LogInformation("Using SMTP to send notification");
  
        // Smtp logic here
        await Task.Delay(0);
  
        return new SendNotificationResponse(true, "Notification sent successfully");
    }
}
```

Cada clase implementa su propia lógica de envío de notificaciones, pero siguen el mismo contrato `INotificationsStrategy`. Esto asegura que el código cliente puede cambiar entre distintas implementaciones sin conocer detalles de su funcionamiento interno.

### NotificationsService y NotificationsFactory

En `NotificationsFactory`, creamos las instancias de los servicios de notificación basándonos en el proveedor configurado.

```csharp
public class NotificationsFactory(
    ILogger<NotificationsFactory> logger,
    IServiceProvider serviceProvider,
    IOptions<NotificationsConfig> notificationsConfig) : INotificationsFactory
{
    public INotificationsStrategy CreateNotificationService()
    {
        var provider = notificationsConfig.Value.Provider;
  
        logger.LogInformation("Creating notification service for provider {Provider}", provider);
  
        return provider switch
        {
            NotificationProvider.SendGrid => serviceProvider.GetRequiredService<SendGridNotificationService>(),
            NotificationProvider.Smtp => serviceProvider.GetRequiredService<SmtpNotificationsService>(),
            _ => throw new NotImplementedException($"Notification provider {provider} is not implemented")
        };
    }
  
    public INotificationsStrategy CreateNotificationService(NotificationProvider provider)
    {
        //...
    }
}
```

`NotificationsFactory` centraliza la creación de objetos. Dependiendo del proveedor especificado, crea una instancia del servicio correspondiente sin que el cliente necesite saber cómo se instancia.

Ejemplo de `appsettings.json`:

```json
  "Notifications": {
    "Provider": "Smtp",
    "SendGrid": {
      "ApiKey": "SG.1234567890"
    },
    "Smtp": {
      "Server": "smtp.server.com",
      "Port": 587,
      "Username": "username",
      "Password": "password"
    }
  }
```

Aunque en este punto podemos utilizar el Factory directamente en cualquier parte de nuestro código, podemos aplicar el patrón **Decorator** para simplificar aún más el uso de `INotificationsStrategy` y hacer que la creación de instancias sea más transparente. Para ello, crearemos la clase `NotificationService`, que será el punto de acceso principal cuando solicitemos la dependencia `INotificationsStrategy`. Esto significa que nuestro código usará `NotificationService`, mientras que `NotificationService` se encargará internamente de invocar al Factory para crear la instancia de la estrategia correcta.

> **Nota 💡:** Este método de creación de objetos está inspirado en la implementación de `IStringLocalizer` y `IStringLocalizerFactory` en ASP.NET Core. Al usar `IStringLocalizer`, el framework invoca internamente el Factory para crear las instancias requeridas en función de la configuración establecida.
> Más información: [aspnetcore/src/Localization/Abstractions/src/StringLocalizerOfT.cs at main · dotnet/aspnetcore](https://github.com/dotnet/aspnetcore/blob/main/src/Localization/Abstractions/src/StringLocalizerOfT.cs)

> **Nota 2💡:** Explorar el código fuente puede ser de gran ayuda. Revisar, leer y ejecutar el código no solo profundiza el conocimiento de estos conceptos, sino que también puede ayudarte a ver cómo los patrones se aplican en la práctica.


```csharp
public class NotificationService(INotificationsFactory notificationsFactory) : INotificationsStrategy
{
    private INotificationsStrategy notificationsStrategy = notificationsFactory.CreateNotificationService();
  
    public async Task<SendNotificationResponse> SendNotification(SendNotificationRequest request)
    {
        return await notificationsStrategy.SendNotification(request);
    }
}
```

## Program.cs

En el archivo `Program.cs`, configuramos la inyección de dependencias. Aquí registramos `INotificationsFactory`, `INotificationsStrategy` y las implementaciones específicas de cada proveedor.

```csharp
var builder = WebApplication.CreateBuilder(args);
  
builder.Services.AddScoped<INotificationsStrategy, NotificationService>();
builder.Services.AddScoped<INotificationsFactory, NotificationsFactory>();
builder.Services.AddScoped<SendGridNotificationService>();
builder.Services.AddScoped<SmtpNotificationsService>();
  
builder.Services.Configure<NotificationsConfig>(builder.Configuration.GetSection("Notifications"));
  
var app = builder.Build();
  
app.MapPost("/send", async (SendNotificationRequest request, INotificationsStrategy notificationService) =>
{
    var response = await notificationService.SendNotification(request);
  
    return response.IsSuccess ? Results.Ok(response) : Results.BadRequest(response);
});
  
app.Run();
```


# Conclusión

Al combinar los patrones **Strategy** y **Factory**, hemos creado un sistema de notificaciones flexible y fácil de extender, en el que distintas implementaciones de notificación pueden integrarse sin afectar la lógica central de la aplicación. Esta estructura modular permite que el código sea adaptable y escalable, lo que es fundamental en entornos en los que los requisitos y las tecnologías cambian constantemente. La incorporación de un patrón **Decorator** adicional facilita la inyección de dependencias, manteniendo el uso de las notificaciones simple y organizado.

Estos patrones aportan claridad y robustez a la arquitectura del sistema, permitiendo que el Core de la aplicación se mantenga desacoplado de los detalles de implementación específicos de cada proveedor de notificaciones. Este enfoque también fomenta la reutilización y facilita el mantenimiento a largo plazo, ya que se pueden agregar o modificar servicios sin necesidad de realizar cambios significativos. La implementación de patrones de diseño bien aplicados no solo mejora la estructura del código, sino que también enriquece el proceso de desarrollo, proporcionando una base sólida para futuros requerimientos y funcionalidades.
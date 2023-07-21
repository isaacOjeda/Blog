# Introducción

En este tutorial, aprenderemos cómo implementar el patrón Outbox en una aplicación ASP.NET Core junto con Background Workers de .NET para mantener la consistencia y mejorar la escalabilidad en sistemas distribuidos. El Outbox Pattern es una técnica valiosa que nos permite manejar operaciones secundarias, como envío de correos electrónicos, generación de eventos o cualquier operación asíncrona, de manera confiable y consistente, al mismo tiempo que rastreamos cada evento.

> Nota 💡: Que no se pierda la costumbre, aquí puedes ver el código fuente 😄 [DevToPosts/OutboxPatternExample at main · isaacOjeda/DevToPosts (github.com)](https://github.com/isaacOjeda/DevToPosts/tree/main/OutboxPatternExample)

# ¿Qué es el Outbox Pattern?

El Outbox Pattern es un patrón de diseño utilizado en el desarrollo de sistemas distribuidos para garantizar la consistencia y confiabilidad en el procesamiento de operaciones secundarias o eventos en una aplicación. Cuando una aplicación necesita realizar acciones adicionales, como enviar correos electrónicos, generar eventos o realizar operaciones que no son transaccionales, surge la necesidad de asegurar que estas acciones se realicen de manera consistente y confiable.

En el contexto de sistemas distribuidos, donde hay múltiples servicios o componentes que interactúan entre sí, la consistencia se convierte en un desafío. El Outbox Pattern aborda esta problemática separando las operaciones secundarias de la operación principal (por ejemplo, la transacción de base de datos) y almacenándolas en una tabla llamada "Outbox". Luego, un proceso en segundo plano, como un Background Worker, se encarga de leer los mensajes del Outbox y llevar a cabo las operaciones secundarias de manera confiable.

**¿Cuándo es bueno usar el Outbox Pattern?**

El Outbox Pattern es útil en diversas situaciones, especialmente en aplicaciones distribuidas con alta concurrencia y múltiples servicios. Algunos escenarios en los que es recomendable utilizar el Outbox Pattern son:

1. **Envío de correos electrónicos y notificaciones**: Cuando necesitas enviar correos electrónicos de confirmación, notificaciones o mensajes a los usuarios después de ciertas operaciones, como registros o compras, el Outbox Pattern garantiza que estos correos electrónicos se entreguen de manera confiable sin afectar el rendimiento de la operación principal.
2. **Generación de eventos y notificaciones**: Si tu aplicación genera eventos para otros servicios o sistemas, el Outbox Pattern asegura que estos eventos se propaguen incluso si ocurre un fallo en la operación principal.
3. **Procesamiento asincrónico**: Cuando una operación secundaria es asincrónica y no necesita completarse inmediatamente después de la operación principal, el Outbox Pattern te permite realizar el procesamiento en segundo plano sin afectar el flujo principal.
4. **Mejora de la escalabilidad**: Utilizar un Background Worker para procesar los mensajes del Outbox permite liberar recursos de la aplicación principal, lo que contribuye a una mejor escalabilidad y rendimiento del sistema.
5. **Trazabilidad y seguimiento de eventos**: Almacenar los mensajes del Outbox en una tabla de la base de datos permite rastrear las operaciones secundarias realizadas y tener un registro claro de los eventos procesados.

El Outbox Pattern es una valiosa técnica para garantizar la consistencia y confiabilidad en sistemas distribuidos, especialmente cuando se necesitan realizar operaciones secundarias asincrónicas o no transaccionales. Al separar las operaciones secundarias de la operación principal y utilizar un proceso en segundo plano para llevar a cabo estas tareas, podemos mejorar la robustez y escalabilidad de nuestras aplicaciones.

**Diagrama de lo que implementaremos hoy**

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/b4wa7y6a6v0stml9wg2f.png)

_[Ver más grande](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/b4wa7y6a6v0stml9wg2f.png)_

1. **Cliente** (Client): Representa el usuario o sistema que realiza una acción específica, en este caso, realiza una compra en la aplicación enviando una solicitud HTTP POST al **API**.
2. **API**: Web API que recibe la solicitud de compra del cliente. En esta etapa, el API valida los datos recibidos, realiza las comprobaciones necesarias y, si todo está correcto, registra una "orden de cobro" que indica que se debe procesar un pago pendiente. Además, guarda el evento relacionado con la compra en la base de datos.
3. **Database**: Es la base de datos donde el API almacena tanto la orden de cobro como el evento relacionado con la compra. Es importante destacar que ambos registros se realizan en una sola transacción, lo que garantiza la consistencia y evita posibles problemas de integridad.
4. **Respuesta al Cliente**: El API envía una respuesta HTTP exitosa (código de estado 2xx) al cliente para indicar que la compra ha sido recibida y que el pago está pendiente.
5. **Bucle del Procesamiento del Outbox**: Aquí comienza el proceso en segundo plano que se encarga de procesar los eventos pendientes, conocido como el **Worker**. Se inicia un bucle para revisar constantemente si existen mensajes en el "Outbox" (un registro en la base de datos que contiene eventos pendientes).
6. **Worker**: Representa el proceso en segundo plano que realiza el procesamiento de los mensajes pendientes del "Outbox". En este caso, busca los mensajes pendientes en la base de datos.
7. **Consulta de mensajes pendientes**: El **Worker** consulta la base de datos para obtener los mensajes pendientes de procesar.
9. **Procesamiento del mensaje**: Una vez que el **Worker** obtiene un mensaje del "Outbox", procede a procesarlo. En este punto, el **Worker** puede interactuar con sistemas externos u otros servicios según sea necesario para completar la operación secundaria asociada al mensaje.
	- **Nota sobre el Worker**: El **Worker** tiene la flexibilidad de interactuar con sistemas externos o realizar acciones adicionales según la naturaleza del evento. Esto permite que el procesamiento de eventos secundarios sea independiente y escalable.
1. **Actualización de la base de datos**: Después de procesar con éxito el mensaje, el **Worker** actualiza el estado del mensaje en la base de datos para indicar que ha sido procesado satisfactoriamente.
2. **Fin del bucle del Procesamiento del Outbox**: Una vez que el **Worker** ha procesado todos los mensajes pendientes en el "Outbox", finaliza el bucle y espera el próximo ciclo para verificar nuevamente si hay nuevos mensajes pendientes.

El diagrama representa cómo funciona el patrón Outbox en una aplicación. Cuando un cliente realiza una acción que genera un evento secundario (como una compra), el evento se registra inicialmente en el "Outbox" de la base de datos, lo que garantiza la consistencia y durabilidad. Luego, un proceso en segundo plano (el **Worker**) se encarga de procesar los eventos pendientes, lo que permite realizar acciones secundarias de manera asíncrona y confiable sin afectar la operación principal. Esto mejora la escalabilidad y la confiabilidad del sistema, ya que el cliente recibe una respuesta inmediata mientras que las tareas adicionales se procesan de manera asincrónica y robusta.

## Implementando el Outbox Pattern en .NET

Para fines prácticos tendremos tres proyectos en esta solución:
- Api (Aplicación Web vacía)
- ApplicationCore (Librería compartida)
- Worker (.NET Worker)

Para estos 3 hay plantillas utilizando `dotnet` o visualmente desde el IDE, tu decides.

Dentro de `ApplicationCore` tendremos la persistencia, domain entities, evento y uno que otro modelo compartido.

La idea es que `Worker` también utilice esta librería, pero estos detalles son simplemente de ejemplo, se puede hacer muchas formas.

> Nota 💡: Una forma de hacer Worker totalmente independiente de ApplicationCore es que no comparta ninguna persistencia y que haga las consultas del Outbox utilizando Dapper o algo similar directamente, así podría vivir en otra solución y repositorio sin problema.

### Definiendo ApplicationCore

Dentro de "ApplicationCore", creamos las siguientes entidades y eventos:

```csharp
namespace ApplicationCore.Entities;
  
public class OutboxMessage
{
    public OutboxMessage()
    {
        Id = Guid.NewGuid();
        CreatedAt = DateTime.Now;
    }
  
    public Guid Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public string EventType { get; set; }
    public string EventBody { get; set; }
    public bool Finished { get; set; }
    public DateTime? ProcessedAt { get; set; }
    public DateTime? FailedAt { get; set; }
    public string? FailureReason { get; set; }
    public int Retries { get; set; }
    public OutboxMessageType Type { get; set; }
}
  
public enum OutboxMessageType
{
    PaymentCreated
}
```

El `OutboxMessage` representa el evento que ha ocurrido y queremos mantener su estado en la base de datos.

Para este ejemplo, realizaremos pagos de forma asíncrona, por lo que el pago se registrará como "Pending" e inmediatamente el outbox se encargará de hacer lo necesario para procesarlo.

```csharp
namespace ApplicationCore.Entities;

public class Payment
{
	public Guid PaymentId { get; set; }
	public string TokenCard { get; set; }
	public double Amount { get; set; }
	public string Currency { get; set; }
	public string CardHolderEmail { get; set; }
	public string CardHolderName { get; set; }
	public PaymentStatus Status { get; set; }
}

public enum PaymentStatus
{
	Pending,
	Authorized,
	Paid,
	Rejected
}
```

Para cada `Payment` creado, existirá un `PaymentCreatedEvent`. Este evento es el que queremos mantener rastreado, ya que es de vital importancia que su finalización se realice con éxito.

A veces, efectuar pagos suele ser una tarea tardada dependiendo de las cosas que se tengan que hacer, por lo que hacerlo de forma asíncrona mejorará la experiencia de uso y con el outbox aseguraremos su finalización con éxito.

```csharp
namespace ApplicationCore.Common.Events;

public record PaymentCreatedEvent(
	Guid PaymentId,
	string TokenCard,
	double Amount,
	string Currency,
	string CardHolderEmail,
	string CardHolderName);
```

Dentro de la carpeta "Infrastructure", creamos una carpeta llamada "Persistence" y agregamos un archivo llamado "AppDbContext.cs" con la siguiente clase:

```csharp
using ApplicationCore.Entities;
using Microsoft.EntityFrameworkCore;

namespace ApplicationCore.Infrastructure.Persistence;

public class AppDbContext : DbContext
{
	public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
	{
	}

	public DbSet<Payment> Payments => Set<Payment>();
	public DbSet<OutboxMessage> OutboxMessages => Set<OutboxMessage>();
}
```

Nada que comentar, es un `DbContext` común y corriente.

Para finalizar, crearemos dos extensiones para registrar estas dos capas (Core e Infrastructure):

```csharp
using ApplicationCore.Infrastructure.Persistence;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

namespace ApplicationCore;

public static class DependencyInjection
{
	public static IServiceCollection AddApplicationCore(this IServiceCollection services)
	{
		return services;
	}

	public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration config)
	{
		services.AddSqlServer<AppDbContext>(config.GetConnectionString("Default"));
		return services;
	}
}
```

### Definiendo Api

Es importante mencionar que "Api" deberá hacer referencia a "ApplicationCore" para poder acceder a todo lo que acabamos de implementar, por lo que comenzamos configurando el proyecto en "Program.cs" de la siguiente manera:

```csharp
using System.Text.Json;
using ApplicationCore;
using ApplicationCore.Common.Events;
using ApplicationCore.Common.Models;
using ApplicationCore.Entities;
using ApplicationCore.Infrastructure.Persistence;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApplicationCore();
builder.Services.AddInfrastructure(builder.Configuration);

var app = builder.Build();

// Endpoints aquí

app.Run();
```

Hemos agregado el registro de las dependencias definidas en "ApplicationCore" y a continuación, configuraremos los endpoints de la API.

**POST /api/payment**

Con este endpoint registraremos un pago y un mensaje en el Outbox en la base de datos, todo en una misma transacción, por lo que se asegura la consistencia al momento. Posteriormente, el Worker se encargará de hacer su trabajo y procesar el mensaje.

```csharp
app.MapPost("/api/payment", async (PaymentRequest request, AppDbContext context) =>
{
    var payment = new Payment
    {
        PaymentId = Guid.NewGuid(),
        TokenCard = request.TokenCard,
        Amount = request.Amount,
        Currency = request.Currency,
        CardHolderEmail = request.CardHolderEmail,
        CardHolderName = request.CardHolderName,
        Status = PaymentStatus.Pending
    };

    context.Payments.Add(payment);
    context.OutboxMessages.Add(new OutboxMessage
    {
        EventType = "PaymentCreatedEvent",
        EventBody = JsonSerializer.Serialize(new PaymentCreatedEvent
        (
            payment.PaymentId,
            payment.TokenCard,
            payment.Amount,
            payment.Currency,
            payment.CardHolderEmail,
            payment.CardHolderName
        )),
    });

    await context.SaveChangesAsync();

    return Results.Ok(new
    {
        Message = "Payment request created successfully",
        payment.PaymentId
    });
});
```

Aquí el DTO usado para la solicitud:

```csharp
public record PaymentRequest(
    string TokenCard,
    double Amount,
    string Currency,
    string CardHolderEmail,
    string CardHolderName
);
```

**GET /api/payment/status/{paymentId}**

Este endpoint nos permitirá consultar el estado del pago, lo que nos permitirá realizar pruebas. También simularemos algunos pagos que pueden fallar.

```csharp
app.MapGet("/api/payment/status/{paymentId}", async (Guid paymentId, AppDbContext context) =>
{
    var payment = await context.Payments.FindAsync(paymentId);

    if (payment is null)
    {
        return Results.NotFound();
    }

    return Results.Ok(new
    {
        payment.PaymentId,
        Status = payment.Status.ToString()
    });
});
```


### Definiendo Worker

Crearemos un `BackgroundService` que nos ayudará a estar consultando en segundo plano los mensajes que lleguen al Outbox.

Es importante mencionar que esta solución podría no ser completamente escalable, ya que si llegan cientos de miles de mensajes, el proceso podría tardar mucho. Es un ejemplo y se puede mejorar utilizando Azure Functions o alguna solución de "Load Balancing" entre varias instancias de este mismo Worker.

Continuemos creando `OutboxProcessorWorker` de la siguiente manera:

```csharp
using System.Text.Json;
using ApplicationCore.Common.Events;
using ApplicationCore.Entities;
using ApplicationCore.Infrastructure.Persistence;
using Microsoft.EntityFrameworkCore;

namespace Worker;

public class OutboxProcessorWorker : BackgroundService
{
	// Implementación del Background Worker
}
```

Dentro de la clase "OutboxProcessorWorker", agregamos el código para procesar los mensajes del Outbox:

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        using var scope = _serviceProvider.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        var outboxMessages = await dbContext.OutboxMessages
            .Where(x => x.Finished == false)
            .ToListAsync(stoppingToken);

        foreach (var outboxMessage in outboxMessages)
        {
            await HandleOutboxMessage(outboxMessage, dbContext);
        }

        await Task.Delay(1000, stoppingToken);
    }
}

private async Task HandleOutboxMessage(OutboxMessage outboxMessage, AppDbContext dbContext)
{
    switch (outboxMessage.Type)
    {
        case OutboxMessageType.PaymentCreated:
            await HandlePaymentCreated(outboxMessage, dbContext);
            break;
        default:
            _logger.LogWarning("Unknown message type {outboxMessage.Type}", outboxMessage.Type);
            break;
    }
}
```

El código anterior implementa el bucle principal del Background Worker que se ejecuta continuamente para procesar los mensajes del Outbox. Utilizando el `DbContext`, recuperamos todos los mensajes no finalizados del Outbox y los procesamos uno por uno.

A continuación, implementamos la lógica para procesar el evento de pago creado y realizar el envío de correo electrónico y el procesamiento del pago.

```csharp
private async Task HandlePaymentCreated(OutboxMessage outboxMessage,

 AppDbContext dbContext)
{
    var paymentCreatedEvent = JsonSerializer.Deserialize<PaymentCreatedEvent>(outboxMessage.EventBody);

    if (paymentCreatedEvent == null)
    {
        _logger.LogWarning("PaymentCreatedEvent is null");
        return;
    }

    var payment = await dbContext.Payments.FindAsync(paymentCreatedEvent.PaymentId);

    if (payment == null)
    {
        _logger.LogWarning("Payment {paymentCreatedEvent.PaymentId} not found", paymentCreatedEvent.PaymentId);
        return;
    }

    // Simulamos el procesamiento del pago con un retraso aleatorio
    await Task.Delay(new Random().Next(1000, 5000));

    if (new Random().Next(0, 10) < 3)
    {
        // El pago es rechazado aleatoriamente con fines de prueba

        // Incrementamos el contador de reintentos
        outboxMessage.Retries++;

        // Si se superan los intentos máximos, marcamos el mensaje como finalizado con error
        if (outboxMessage.Retries > 3)
        {
            outboxMessage.Finished = true;
            outboxMessage.FailureReason = "Max retries reached";
        }

        await dbContext.SaveChangesAsync();

        return;
    }

    // El pago se procesa exitosamente
    payment.Status = PaymentStatus.Paid;

    // Marcamos el mensaje del Outbox como procesado
    outboxMessage.ProcessedAt = DateTime.Now;
    outboxMessage.Finished = true;

    outboxMessage.FailedAt = null;
    outboxMessage.FailureReason = null;

    await dbContext.SaveChangesAsync();
}
```

En este código, procesamos el evento de pago creado recuperando los datos del mensaje del Outbox y del pago correspondiente de la base de datos. Luego, simulamos el procesamiento del pago con un retraso aleatorio para emular una operación real.

Además, hemos agregado una lógica para rechazar aleatoriamente el pago con fines de prueba. Si el pago es rechazado, incrementamos el contador de reintentos y, si se superan los intentos máximos, marcamos el mensaje como finalizado con error. Por otro lado, si el pago es exitoso, actualizamos el estado del pago y marcamos el mensaje del Outbox como **procesado**.

Finalmente, configuramos el proyecto "Worker" para que ejecute el Background Worker y procese los mensajes del Outbox en segundo plano.

```csharp
using ApplicationCore;

using Worker;

IHost host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostBuilder, services) =>
    {
        services.AddApplicationCore();
        services.AddInfrastructure(hostBuilder.Configuration);

        services.AddHostedService<OutboxProcessorWorker>();
    })
    .Build();

host.Run();
```

En este código, hemos agregado las dependencias de "ApplicationCore" e "Infrastructure" al proyecto "Worker" y configurado el Background Worker "OutboxProcessorWorker" para que se ejecute como un servicio hospedado.

## Probando la Solución

Antes de continuar, necesitamos haber creado las migraciones del `DbContext` y la base de datos. Si lo deseas, puedes descargar el código fuente y ejecutarlo sin problemas.

Debes correr **Api** y **Worker** juntos, y automáticamente el Worker empezará a funcionar en ese loop infinito para leer los mensajes del Outbox.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jxfzzulgd20tqxa2mz95.png)

Si utilizamos la API y enviamos una solicitud POST para crear un pago, podremos ver todo funcionando:

```
### Create Payment

POST {{host}}/api/payment
Content-Type: application/json
  
{
    "tokenCard": "tok_1J4Z3n2eZvKYlo2C0X2X2X2X",
    "amount": 1000,
    "currency": "usd",
    "cardHolderName": "John Doe",
    "cardHolderEmail": "john.doe@mail.com"
}
```

Respuesta:

```json
{ 
    "message": "Payment request created successfully",
    "paymentId": "43c34018-6446-420a-8c8a-0c29607d632b"
}
```

En cuanto el loop del worker vuelva a iniciar, encontrará este Outbox recién creado y lo procesará:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1gqzij6vvmme9c6juby0.png)

En este ejemplo, vemos cómo el pago fue rechazado, pero el Worker lo volverá a intentar en el siguiente ciclo del loop.

Podemos consultar el estatus del pago, y aquí es donde entra en función el propósito del Outbox. Podemos ver si esta acción no transaccional se llevó a cabo o no; si no se pudo realizar la tarea, podemos tomar acciones al respecto y asegurar la consistencia de la información.

```
### Get Payment

GET {{host}}/api/payment/status/43c34018-6446-420a-8c8a-0c29607d632b
Content-Type: application/json
```

```json
{
    "paymentId": "43c34018-6446-420a-8c8a-0c29607d632b",
    "status": "Paid"
}
```

# Conclusión

En este tutorial, hemos aprendido a implementar el patrón Outbox en una aplicación ASP.NET Core junto con Background Workers de .NET para mantener la consistencia y mejorar la escalabilidad en sistemas distribuidos. Al seguir este patrón, hemos logrado separar las operaciones secundarias, como el envío de correos electrónicos y el procesamiento de pagos, del flujo principal de la aplicación, lo que garantiza una mayor confiabilidad y seguimiento de eventos.

Espero que este tutorial te haya resultado útil y te haya brindado una comprensión clara de cómo utilizar el patrón Outbox en tu aplicación ASP.NET Core. Si tienes alguna pregunta o duda, no dudes en consultar la documentación oficial de ASP.NET Core y .NET para obtener más información sobre las tecnologías y conceptos presentados en este tutorial. ¡Buena suerte en tu desarrollo!

# Referencias

- [Outbox Pattern for Microservices Architectures | by Mehmet Ozkaya | Design Microservices Architecture with Patterns & Principles | Medium](https://medium.com/design-microservices-architecture-with-patterns/outbox-pattern-for-microservices-architectures-1b8648dfaa27)
- [Transactional outbox (microservices.io)](https://microservices.io/patterns/data/transactional-outbox.html)
- [Outbox Pattern For Reliable Microservices Messaging (milanjovanovic.tech)](https://www.milanjovanovic.tech/blog/outbox-pattern-for-reliable-microservices-messaging?utm_source=emailoctopus&utm_medium=email&utm_campaign=MNW%20%2326)
- [Milan Jovanović en Twitter: "What could go wrong in these 4 lines of code?](https://twitter.com/mjovanovictech/status/1673572936660164609)
- 
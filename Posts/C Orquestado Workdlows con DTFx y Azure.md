
## Introducción

En la era de las aplicaciones modernas y servicios en la nube, la gestión eficiente de flujos de trabajo largos y persistentes es esencial para el desarrollo de aplicaciones robustas. En este contexto, el Durable Task Framework emerge como una herramienta poderosa para la creación de workflows en entornos C#, ofreciendo capacidades avanzadas de orquestación y manejo de estados. Acompañado por el Azure Storage Emulator, conocido como Azurite, que proporciona una forma local de emular los servicios de almacenamiento de Azure, esta combinación facilita el desarrollo, prueba y depuración de aplicaciones basadas en workflows. A lo largo de este artículo, exploramos la implementación de workflows con estas tecnologías, destacando su utilidad y eficacia en la construcción de sistemas resilientes y escalables.

Cómo siempre, aquí encontrarás el código de este post: [DevToPosts/DurableTask · isaacOjeda/DevToPosts (github.com)](https://github.com/isaacOjeda/DevToPosts/tree/main/DurableTask)
## Durable Task Framework

El Durable Task Framework (DTFx) es una biblioteca que permite a los usuarios escribir flujos de trabajo persistentes de larga duración (llamados orquestaciones) en C# utilizando simples instrucciones de código `async`/`await`. Se utiliza ampliamente dentro de varios equipos en Microsoft para orquestar de manera confiable operaciones de aprovisionamiento, monitoreo y gestión de larga duración. Las orquestaciones se escalan de manera lineal simplemente agregando más máquinas de trabajo. Este marco también se utiliza para alimentar la extensión serverless **Durable Functions de Azure Functions**.

**Características clave del framework:**
- Definición de orquestaciones de código en C# simple.
- Persistencia automática y punto de control del estado del programa.
- Versionado de orquestaciones y actividades.
- Temporizadores asíncronos.
### Desafíos en la Gestión de Transacciones en la Nube

En diversos escenarios, nos encontramos con la necesidad de actualizar estados o ejecutar acciones en múltiples ubicaciones de manera transaccional. Por ejemplo, realizar un cargo en una cuenta de la base de datos A y abonarlo a otra cuenta en la base de datos B debe llevarse a cabo de manera **atómica**. La consistencia en este tipo de operaciones se logra tradicionalmente mediante el uso de transacciones distribuidas que abarcan las operaciones de débito y crédito en las bases de datos A y B, respectivamente.

No obstante, la aplicación estricta de transacciones conlleva desafíos significativos. El uso de bloqueos, inherente a las transacciones, puede ser perjudicial para la escalabilidad, ya que las operaciones subsiguientes que requieren el mismo bloqueo se verían bloqueadas hasta que se libere. Esto representa un importante cuello de botella de escalabilidad para los servicios en la nube, diseñados para ser altamente disponibles y consistentes. Además, incluso si decidimos asumir el impacto de una transacción distribuida, nos encontraríamos con la limitación de que casi ningún servicio en la nube admite efectivamente transacciones distribuidas.
#### Alternativa: Flujos de Trabajo (Workflows)

Una alternativa para lograr consistencia es ejecutar la lógica de débito y crédito dentro de un flujo de trabajo duradero. En este enfoque, el flujo de trabajo realiza las siguientes acciones:

1. **Débito:** Se realiza el débito desde una cuenta en la base de datos A.
2. Si el débito fue exitoso, entonces:
   3. **Crédito:** Se realiza el crédito a una cuenta en la base de datos B.
4. Si el paso anterior falla, se intenta nuevamente hasta alcanzar un umbral definido.
5. Si el crédito aún falla, se deshace el débito en la base de datos A y se envía un correo electrónico de notificación.

En el mejor de los casos, esto nos proporciona consistencia "eventual". Es decir, después de (1) el sistema se encuentra en un estado inconsistente, pero eventualmente alcanza la consistencia después de que se completa el flujo de trabajo. Sin embargo, en el peor escenario, pueden ocurrir diversos problemas: el nodo que ejecuta el código puede fallar en un punto arbitrario, el débito en la base de datos A puede fallar o el crédito en la base de datos B puede fallar.

Para garantizar la consistencia en estos casos, es crucial considerar lo siguiente:

- Las operaciones de débito y crédito deben ser **idempotentes**, es decir, volver a ejecutar la misma operación no tendría efectos adicionales.
- Si el nodo de ejecución se bloquea, debe reiniciarse desde el último lugar donde se realizó una operación exitosa (por ejemplo, #1 o #2 anteriormente).

Estos dos elementos son esenciales para mantener la integridad del sistema. La idempotencia puede ser asegurada por la implementación de las operaciones de débito/crédito, mientras que el reinicio desde el último punto exitoso puede lograrse mediante el seguimiento de la posición actual en alguna base de datos. Sin embargo, gestionar este estado puede volverse engorroso, especialmente a medida que el número de operaciones duraderas aumenta. Aquí es donde un framework para la gestión automática del estado simplificaría significativamente la experiencia de construir flujos de trabajo basados en código. Para esto, usaremos **DTFx**.
### Funcionamiento de los Workflows en el Durable Task Framework

#### ¿Cómo Funcionan los Workflows?

Los Workflows en el Durable Task Framework se componen de orquestaciones, actividades y un conjunto de herramientas que trabajan en conjunto para gestionar flujos de trabajo de larga duración de manera eficiente y confiable.

1. **Orquestaciones:**
   - Las orquestaciones son el corazón de los workflows y representan el flujo general del trabajo a realizar.
   - Programan actividades y coordinan su ejecución.
   - Pueden contener lógica empresarial, manejar decisiones y gestionar el flujo de control.
2. **Actividades:**
   - Las actividades son unidades de trabajo atómicas dentro de un flujo de trabajo.
   - Realizan tareas específicas, como realizar operaciones en una base de datos, enviar correos electrónicos, etc.
   - Pueden ser ejecutadas de manera independiente y son diseñadas para ser **idempotentes**.
3. **Task Hub:**
   - El Task Hub es un componente clave que actúa como contenedor lógico para las entidades.
   - Facilita el intercambio de mensajes confiable entre las orquestaciones y las actividades.
4. **Task Hub Worker:**
   - El Task Hub Worker es el entorno de ejecución para las orquestaciones y actividades.
   - Hospeda las orquestaciones y se encarga de la ejecución de las actividades en el momento adecuado.
5. **Task Hub Client:**
   - El Task Hub Client proporciona APIs para crear, gestionar y consultar instancias de orquestaciones.
   - Facilita la interacción con el Task Hub y permite iniciar nuevas instancias de orquestaciones.

#### ¿Cómo Ayudan los Workflows a Resolver el Problema Planteado?

En el problema descrito anteriormente, donde múltiples acciones deben realizarse de manera transaccional, los workflows ofrecen una alternativa efectiva:

1. **Consistencia:**
   - Al programar lógica de débito y crédito dentro de una orquestación, las transacciones se ejecutan de manera coherente, garantizando consistencia en el sistema.
2. **Escalabilidad:**
   - La ejecución de orquestaciones y actividades es gestionada por el Task Hub Worker, que puede escalar linealmente al agregar más máquinas de trabajo según sea necesario.
3. **Manejo de Errores:**
   - En caso de errores durante la ejecución, los workflows permiten estrategias de manejo de errores, como reintentar operaciones o deshacer acciones realizadas.
4. **Idempotencia:**
   - Las actividades diseñadas como operaciones idempotentes garantizan que reintentar la misma operación no tenga efectos secundarios no deseados.
5. **Persistencia Automática:**
   - El Task Hub facilita la persistencia automática y checkpointing del estado del programa, asegurando que, en caso de falla, se pueda retomar desde el último punto exitoso.

## Creando un Workflow con DTFx

En este ejemplo, vamos a construir un workflow simple utilizando el Durable Task Framework (DTFx). El objetivo es simular un proceso de cobro, generación de factura y manejar posibles errores de manera elegante. Aunque el ejemplo es básico, nos proporcionará una comprensión práctica de cómo funciona DTFx.
### Escenario del Workflow

El workflow consta de los siguientes pasos:

1. **Recibir una Solicitud de Cobro:**
   - Se recibirá una solicitud de cobro.
   - Simularemos la ejecución del cobro, incluyendo segundos aleatorios de espera.
2. **Generar Factura:**
   - Si el cobro se lleva a cabo con éxito, procederemos a generar una factura.
   - Simularemos posibles fallas aleatorias durante la generación de factura, y se implementará un mecanismo de reintento en caso de error.

Este ejemplo, aunque simple, nos proporcionará una visión práctica del funcionamiento de DTFx y cómo gestionar flujos de trabajo duraderos.

Es importante destacar que, aunque DTFx es compatible con .NET Core, no proporciona una forma predeterminada de resolver dependencias. Dado que estaremos utilizando ASP.NET Core y queremos seguir su estilo, exploraremos cómo resolver este aspecto de manera elegante y eficiente en este ejemplo.

Sigamos adelante y detallaremos la implementación paso a paso.
### CreatePaymentActivity

Este activity se "encargará" de realizar el cobro:

```csharp
using DurableTask.Core;
namespace DurableTask.Api.Workflows.CreatePayment;
 
public class CreatePaymentActivity(ILogger<CreatePaymentActivity> logger)
    : AsyncTaskActivity<CreatePaymentRequest, CreatePaymentResponse>
{
    protected override async Task<CreatePaymentResponse> ExecuteAsync(TaskContext context, CreatePaymentRequest input)
    {
        logger.LogInformation("\nCreating payment for order {OrderId} with payment method {PaymentMethodId}\n",
            input.OrderId, input.PaymentMethodId);
 
        await Task.Delay(new Random().Next(1, 5) * 1000);
 
        return new CreatePaymentResponse(Guid.NewGuid().ToString());
    }
}

public record CreatePaymentRequest(string OrderId, string PaymentMethodId);
public record CreatePaymentResponse(string PaymentId);
```

- Se define la clase `CreatePaymentActivity` que hereda de `AsyncTaskActivity<CreatePaymentRequest, CreatePaymentResponse>`, indicando que es una actividad asincrónica que toma una solicitud `CreatePaymentRequest` y devuelve una respuesta `CreatePaymentResponse`.
- En el constructor de la actividad, se inyecta un logger (`ILogger<CreatePaymentActivity>`).
- El método `ExecuteAsync` contiene la lógica principal de la actividad. Registra información, simula un retraso aleatorio y luego devuelve una respuesta `CreatePaymentResponse` con un ID de pago único.
- Las clases `CreatePaymentRequest` y `CreatePaymentResponse` son registros que representan las estructuras de datos utilizadas para la solicitud y la respuesta de la actividad, respectivamente.
### CreateInvoiceActivity

Este Activity se encargará ahora de generar la factura:

```csharp
using DurableTask.Core;
namespace DurableTask.Api.Workflows.CreatePayment;
 
public class CreateInvoiceActivity(ILogger<CreateInvoiceActivity> logger)
    : AsyncTaskActivity<CreateInvoiceRequest, CreateInvoiceResponse>
{
    protected override Task<CreateInvoiceResponse> ExecuteAsync(TaskContext context, CreateInvoiceRequest input)
    {
        if (new Random().Next(0, 10) > 5)
        {
            logger.LogError("Failed to create invoice");
            throw new Exception("Failed to create invoice");
        }
 
        logger.LogInformation("\nCreating invoice for order {OrderId} with payment {PaymentId}\n",
                       input.OrderId, input.PaymentId);
 
        return Task.FromResult(new CreateInvoiceResponse(Guid.NewGuid().ToString()));
    }
}
 
public record CreateInvoiceRequest(string OrderId, string PaymentId);
public record CreateInvoiceResponse(string InvoiceId);
```

- Al igual que la actividad anterior, se define la clase `CreateInvoiceActivity` que hereda de `AsyncTaskActivity<CreateInvoiceRequest, CreateInvoiceResponse>`.
- En el método `ExecuteAsync`, se simula una probabilidad de error al crear una factura.
- Si no hay errores, se registra información sobre la creación de la factura y se devuelve una respuesta `CreateInvoiceResponse` con un ID de factura único.
### PaymentOrchestrator

```csharp
using DurableTask.Core;
using DurableTask.Core.Exceptions;
 
namespace DurableTask.Api.Workflows.CreatePayment;

public class PaymentOrchestrator(ILogger<PaymentOrchestrator> logger)
    : TaskOrchestration<PaymentResponse, CreatePaymentRequest>
{
	public override async Task<PaymentResponse> RunTask(OrchestrationContext context, CreatePaymentRequest input)
	{
		logger.LogInformation("Running Orchestration");
	 
		var paymentResponse = await CreatePayment(context, input);
		var invoiceResponse = await CreateInvoice(context, input, paymentResponse);
	 
		logger.LogInformation("Orchestration completed");
	 
		return new PaymentResponse(paymentResponse.PaymentId, invoiceResponse.InvoiceId);
	}
}

public record PaymentResponse(string PaymentId, string InvoiceId);
```

- La clase `PaymentOrchestrator` hereda de `TaskOrchestration<PaymentResponse, CreatePaymentRequest>`, indicando que es un orquestador que toma una solicitud `CreatePaymentRequest` y devuelve una respuesta `PaymentResponse`.
- En el método `RunTask`, se llama a las actividades `CreatePaymentActivity` y `CreateInvoiceActivity` para realizar el cobro y crear la factura, respectivamente.
- DTFx maneja la persistencia de estados automáticamente. En puntos críticos, como antes y después de realizar llamadas a actividades, el estado de la orquestación se guarda de forma automática.
- Si la orquestación se detiene en algún punto, ya sea debido a un tiempo de espera o a una espera de actividad, el estado actual se guarda de manera persistente en el almacenamiento configurado (generalmente Azure Storage).
- DTFx proporciona capacidades integradas de reintentos y manejo de errores. Por ejemplo, si una actividad falla, se pueden configurar reintentos automáticos.

Definición de `CreatePayment`:

```csharp
private async Task<CreatePaymentResponse> CreatePayment(OrchestrationContext context, CreatePaymentRequest input)
{
    return await context.ScheduleTask<CreatePaymentResponse>(typeof(CreatePaymentActivity), input);
}
```

Aquí simplemente mandamos a llamar el Activity, sin configurar algo especial.

Definición de `CreateInvoice`:

```csharp
private async Task<CreateInvoiceResponse?> CreateInvoice(OrchestrationContext context, CreatePaymentRequest input,
    CreatePaymentResponse paymentResponse)
{
    CreateInvoiceResponse? invoiceResponse;
    try
    {
        var retryOptions = new RetryOptions(TimeSpan.FromSeconds(10), 5);
 
        invoiceResponse = await context.ScheduleWithRetry<CreateInvoiceResponse>(typeof(CreateInvoiceActivity),
            retryOptions, new CreateInvoiceRequest(input.OrderId, paymentResponse.PaymentId));
    }
    catch (TaskFailedException ex)
    {
        logger.LogError(ex, "Failed to create invoice");
 
        return null;
    }
 
    return invoiceResponse;
}
```

1. **Configuración de Reintentos:**
    - Se configuran opciones de reintentos utilizando `RetryOptions`. En este caso, se especifica un intervalo de reintentos de 10 segundos y un máximo de 5 reintentos en caso de fallo.
2. **Llamada a la Actividad con Reintentos:**
    - `context.ScheduleWithRetry` se utiliza para llamar a la actividad `CreateInvoiceActivity` con las opciones de reintentos configuradas.
    - Se pasa una instancia de `CreateInvoiceRequest` que contiene la información necesaria para la actividad.
3. **Manejo de Excepciones:**
    - Se utiliza un bloque `try-catch` para manejar cualquier excepción que pueda ocurrir durante la ejecución de la actividad.
    - En caso de un fallo (capturado como `TaskFailedException`), se registra un error utilizando el logger y se retorna `null` para indicar que la creación de la factura ha fallado.
4. **Retorno de Resultado:**
    - Si la actividad se ejecuta con éxito, la respuesta de la actividad (`invoiceResponse`) se devuelve como resultado.
### TaskHubWorker

`WorkflowWorker` que actúa como un servicio hospedado (`IHostedService`) y representa el Task Hub Worker en el contexto del Durable Task Framework (DTFx). El Task Hub Worker es responsable de ejecutar tareas orquestadas y actividades definidas en la aplicación. Aquí está una explicación detallada del código:

```csharp
using DurableTask.Api.Workflows.CreatePayment;
using DurableTask.AzureStorage;
using DurableTask.Core;
 
namespace DurableTask.Api.Workflows;
 
public class WorkflowWorker(IServiceProvider serviceProvider) : IHostedService
{
    private TaskHubWorker _taskHubWorker = null!;
 
    public async Task StartAsync(CancellationToken cancellationToken)
    {
        var settings = serviceProvider.GetRequiredService<AzureStorageOrchestrationServiceSettings>();
        var azureStorageOrchestrationService = new AzureStorageOrchestrationService(settings);
 
        _taskHubWorker = new TaskHubWorker(azureStorageOrchestrationService);
 
        await _taskHubWorker
            .AddTaskOrchestrations(new ServiceProviderObjectCreator<TaskOrchestration>(typeof(PaymentOrchestrator), serviceProvider))
            .AddTaskActivities(new ServiceProviderObjectCreator<TaskActivity>(typeof(CreatePaymentActivity), serviceProvider))
            .AddTaskActivities(new ServiceProviderObjectCreator<TaskActivity>(typeof(CreateInvoiceActivity), serviceProvider))
            .StartAsync();
 
    }
 
    public Task StopAsync(CancellationToken cancellationToken)
    {
        return _taskHubWorker.StopAsync(true);
    }
}
```

1. **Configuración del Servicio de Orquestación de Almacenamiento Azure:**
    - Se obtienen las configuraciones necesarias para el servicio de orquestación de almacenamiento Azure desde el contenedor de servicios (`serviceProvider`).
    - Se crea una instancia del servicio de orquestación de almacenamiento Azure (`AzureStorageOrchestrationService`) utilizando las configuraciones obtenidas.
2. **Creación del Task Hub Worker:**
    - Se crea una instancia del Task Hub Worker (`TaskHubWorker`) utilizando el servicio de orquestación de almacenamiento Azure.
3. **Configuración y Registro de Orquestaciones y Actividades:**
    - Se configuran y registran las orquestaciones y actividades que el Task Hub Worker debe ejecutar.
    - Se utiliza el método `AddTaskOrchestrations` para agregar las orquestaciones y `AddTaskActivities` para agregar las actividades.
    - En este caso, se utilizan `ServiceProviderObjectCreator` para permitir la resolución de dependencias utilizando el contenedor de servicios (`serviceProvider`).
    - Se inician las orquestaciones y actividades registradas llamando a `StartAsync()`.
4. **Iniciar el Task Hub Worker:**
    - El método `StartAsync()` se encarga de iniciar el Task Hub Worker, lo que permite que comience a procesar orquestaciones y actividades.
5. **Detener el Task Hub Worker:**
    - El método `StopAsync()` se utiliza para detener el Task Hub Worker cuando es necesario. En este caso, se utiliza con `await` para esperar hasta que el Task Hub Worker se detenga antes de completar la tarea.

En resumen, el `WorkflowWorker` se encarga de iniciar y detener el Task Hub Worker, así como de configurar y registrar las orquestaciones y actividades que debe ejecutar. Este componente es fundamental para la ejecución y coordinación de flujos de trabajo duraderos en el Durable Task Framework.

`ServiceProviderObjectCreator<T>` que actúa como un creador de objetos para el framework Durable Task. Su propósito principal es proporcionar una manera de resolver instancias de objetos utilizando un contenedor de servicios (`IServiceProvider`)

```csharp
using DurableTask.Core;
 
namespace DurableTask.Api.Workflows;
 
public class ServiceProviderObjectCreator<T> : ObjectCreator<T>
{
    private readonly Type _prototype;
    private readonly IServiceProvider _serviceProvider;
 
    public ServiceProviderObjectCreator(Type type, IServiceProvider serviceProvider)
    {
        _prototype = type;
        _serviceProvider = serviceProvider;
 
        Initialize(type);
    }
 
    public override T Create()
    {
        return (T)_serviceProvider.GetService(_prototype)!;
    }
 
    private void Initialize(object obj)
    {
        Name = NameVersionHelper.GetDefaultName(obj);
        Version = NameVersionHelper.GetDefaultVersion(obj);
    }
}
```

**Método `Create`:**
- El método `Create` es una implementación del método abstracto en la clase base `ObjectCreator<T>`. Este método se llama para crear una nueva instancia del objeto.
- Utiliza el contenedor de servicios (`_serviceProvider`) para resolver una instancia del tipo especificado (`_prototype`).

Esta clase permite integrar el Durable Task Framework con un contenedor de servicios, lo que facilita la resolución de dependencias y la creación de instancias de objetos dentro de las orquestaciones y actividades del Durable Task Framework.
### Program

Para ya casi terminar, así quedaría la entrada de nuestra aplicación:

```csharp
using DurableTask.Api.Workflows;
using DurableTask.Api.Workflows.CreatePayment;
using DurableTask.Core;
 
var builder = WebApplication.CreateBuilder(args);
 
builder.AddWorkflows();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
 
var app = builder.Build();
 
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
 
app.UseHttpsRedirection();
 
app.MapPost("/api/payments", async (CreatePaymentRequest request, TaskHubClient client) =>
{
    var instanceId = await client.CreateOrchestrationInstanceAsync(typeof(PaymentOrchestrator), request);
 
    return Results.Ok(new
    {
        instanceId
    });
});
 
app.Run();
```

**Configuración Inicial:**
- `AddWorkflows()` se llama para configurar los workflows y servicios relacionados necesarios para DTFx.
**Definición de un Endpoint POST:**
- `MapPost("/api/payments", async (CreatePaymentRequest request, TaskHubClient client) => {...}` define un endpoint para manejar solicitudes HTTP POST a la ruta "/api/payments".
- El handler de este endpoint crea una instancia de orquestación (`PaymentOrchestrator`) utilizando el cliente de Task Hub (`TaskHubClient`). La solicitud (`CreatePaymentRequest`) se pasa como entrada a la orquestación.
- Se retorna un resultado que incluye el ID de la instancia de orquestación creada.
### WorkflowConfigExtensions

Clase de extensión llamada `WorkflowConfigExtensions` que proporciona métodos para configurar y agregar servicios relacionados con workflows:

```csharp
using DurableTask.Api.Workflows.CreatePayment;
using DurableTask.AzureStorage;
using DurableTask.Core;
 
namespace DurableTask.Api.Workflows;
 
internal static class WorkflowConfigExtensions
{
    public static WebApplicationBuilder AddWorkflows(this WebApplicationBuilder builder)
    {
        builder.Services.AddTransient<PaymentOrchestrator>();
        builder.Services.AddTransient<CreatePaymentActivity>();
        builder.Services.AddTransient<CreateInvoiceActivity>();
        // Azure Storage Accounts configuration
        builder.Services.AddTransient(_ => new AzureStorageOrchestrationServiceSettings
        {
            StorageConnectionString = builder.Configuration["DurableTask:StorageConnectionString"],
            TaskHubName = builder.Configuration["DurableTask:HubName"]
        });
        // Task Hub Client
        builder.Services.AddTransient(sc =>
        {
            var serviceSettings = sc.GetRequiredService<AzureStorageOrchestrationServiceSettings>();
            var azureStorageOrchestrationService = new AzureStorageOrchestrationService(serviceSettings);
 
            return new TaskHubClient(azureStorageOrchestrationService);
        });
 
        builder.Services.AddHostedService<WorkflowWorker>();
 
        return builder;
    }
}
```

1. **Registro de Servicios Transient:**
    - Se utilizan métodos `AddTransient` para registrar las clases `PaymentOrchestrator`, `CreatePaymentActivity`, y `CreateInvoiceActivity` como servicios transient. Esto significa que se creará una nueva instancia de estas clases para cada solicitud.
2. **Configuración de Cuentas de Almacenamiento de Azure:**
    - Se utiliza `AzureStorageOrchestrationServiceSettings` para configurar la conexión de almacenamiento de Azure necesario para DTFx.
    - Los valores de la cadena de conexión y el nombre del Task Hub se obtienen de la configuración de la aplicación.
3. **Configuración del Cliente del Task Hub:**
    - Se registra un servicio transient para el cliente del Task Hub (`TaskHubClient`). Este servicio se configura para depender de la configuración de almacenamiento de Azure y crea una instancia del cliente del Task Hub.
4. **Agregar Worker de Workflows como Servicio Hospedado:**
    - Se utiliza `AddHostedService` para registrar el `WorkflowWorker` como un servicio hospedado en la colección de servicios. Esto asegura que el `WorkflowWorker` se inicie y detenga junto con la aplicación.

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "DurableTask": {
    "HubName": "DurableFunctionsHub",
    "StorageConnectionString": "UseDevelopmentStorage=true;"
  }
}
```


### Probando la Solución

Antes de poner en marcha la aplicación, es crucial asegurarse de tener instalado el Emulador de Azure Storage, también conocido como Azurite, ya que el correcto funcionamiento del Worker depende por completo de este servicio.

Una vez que el emulador de almacenamiento está en marcha, podemos ejecutar la aplicación y utilizar Swagger para realizar solicitudes de prueba. A continuación, se muestra un ejemplo de los resultados que se pueden observar en la consola:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/akdndivp70p0rb031ptl.png)

Observamos cómo la simulación de un fallo ocurrió en un par de ocasiones, sin embargo, vemos cómo el Framework se encargó de gestionar automáticamente los reintentos. Finalmente, con éxito, el workflow llegó a su conclusión.
## Conclusión

En conclusión, la implementación de workflows utilizando el Durable Task Framework en conjunto con el Azure Storage demuestra ser una solución robusta y escalable para gestionar procesos largos y persistentes en aplicaciones basadas en C#. La capacidad del framework para manejar automáticamente reintentos en caso de fallos, junto con la flexibilidad proporcionada por Azurite para simular el entorno de almacenamiento de Azure localmente, facilita el desarrollo y la depuración de workflows, garantizando la confiabilidad y consistencia en escenarios críticos. La combinación de estas herramientas ofrece a los desarrolladores una poderosa infraestructura para orquestar flujos de trabajo complejos de manera eficiente y confiable.

## Referencias y Links de interés

- [Home · Azure/durabletask Wiki (github.com)](https://github.com/Azure/durabletask/wiki)
- [Durable Task Framework Internals - Part 1 (Dataflow and Reliability) | Abhik's Blog (abhikmitra.github.io)](https://abhikmitra.github.io/blog/durable-task/)
- [Dependency Injection with Durable Task Framework | Andrew Stevens](https://andrewstevens.dev/posts/dependency-injection-durable-task/)
- [Goodbye long procedural code! Fix it with workflows (youtube.com)](https://www.youtube.com/watch?v=WjzojcyNp4U&ab_channel=CodeOpinion)
- [Durable Task Framework Internals - Part 1 (Dataflow and Reliability) | Abhik's Blog (abhikmitra.github.io)](https://abhikmitra.github.io/blog/durable-task/)
- [Sagas · MassTransit](https://masstransit.io/documentation/configuration/sagas/overview)
- [Sagas • NServiceBus • Particular Docs](https://docs.particular.net/nservicebus/sagas/)
- [Workflow overview | Dapr Docs](https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-overview/)

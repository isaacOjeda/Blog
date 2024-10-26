# Introducción
En este artículo, aprenderemos cómo crear un servicio en segundo plano en ASP.NET Core que se ejecutará según un intervalo de tiempo definido mediante una expresión CRON, similar a las tareas programadas en sistemas Linux.

> **Nota 💡:** Puedes encontrar el [código fuente completo en GitHub](https://github.com/isaacOjeda/DevToPosts/tree/main/BackgroundJob.Cron).

El formato CRON es ampliamente utilizado para expresar horarios de tareas programadas. Se compone de 5 o 6 campos que representan distintas unidades de tiempo, como se muestra a continuación:

```
                                       Allowed values    Allowed special characters   Comment

┌───────────── second (optional)       0-59              * , - /                      
│ ┌───────────── minute                0-59              * , - /                      
│ │ ┌───────────── hour                0-23              * , - /                      
│ │ │ ┌───────────── day of month      1-31              * , - / L W ?                
│ │ │ │ ┌───────────── month           1-12 or JAN-DEC   * , - /                      
│ │ │ │ │ ┌───────────── day of week   0-6  or SUN-SAT   * , - / # L ?                Both 0 and 7 means SUN
│ │ │ │ │ │
* * * * * *
```

Por ejemplo, las siguientes expresiones CRON programan tareas en diferentes intervalos:

| **Expresión** | **Descripción** |
|---|---|
| * * * * * | Cada minuto | 
| 0 0 1 * * | A media noche, en día primero de cada mes |
| 0 0 * * MON-FRI | A las 0:00, de Lunes a Viernes | 

> **Nota 💡:** Para más detalles sobre las expresiones CRON, puedes consultar [Cronos en GitHub](https://github.com/HangfireIO/Cronos).

## Implementación en ASP.NET Core y Hosted Services

Si bien existen soluciones robustas como [HangFire](https://www.hangfire.io/) y [Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-timer?tabs=in-process&pivots=programming-language-csharp) que soportan tareas programadas basadas en CRON, a veces es preferible mantener las cosas simples. En este artículo, exploraremos cómo implementar una solución personalizada utilizando Hosted Services en ASP.NET Core.

### Creación del Proyecto

Primero, creamos un nuevo proyecto web vacío o de consola, dependiendo de nuestras necesidades:

```bash
dotnet new web -o BackgroundJob.Cron
```

A continuación, instalamos la librería Cronos para poder interpretar y manejar expresiones CRON:

```
dotnet add package Cronos
```

### CronBackgroundJob

El core de nuestra implementación es la clase `CronBackgroundJob`, una clase abstracta que ejecuta un proceso en segundo plano según un intervalo definido por una expresión CRON.

```csharp
using Cronos;

namespace BackgroundJob.Cron.Jobs;
  
public abstract class CronBackgroundJob : BackgroundService
{
    private readonly CronExpression _cronExpression;
    private readonly TimeZoneInfo _timeZone;
  
    public CronBackgroundJob(string rawCronExpression, TimeZoneInfo timeZone)
    {
        _cronExpression = CronExpression.Parse(rawCronExpression);
        _timeZone = timeZone;
    }
  
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            DateTimeOffset? nextOccurrence = _cronExpression.GetNextOccurrence(DateTimeOffset.UtcNow, _timeZone);
            if (!nextOccurrence.HasValue)
                return;
  
            var delay = nextOccurrence.Value - DateTimeOffset.UtcNow;
            if (delay.TotalMilliseconds > 0)
            {
                try
                {
                    await Task.Delay(delay, stoppingToken);
                }
                catch (TaskCanceledException)
                {
                    // Handle cancellation if needed
                    return;
                }
            }
  
            try
            {
                await DoWork(stoppingToken);
            }
            catch (Exception ex)
            {
                // Handle or log the exception as needed
            }
        }
    }
  
    protected abstract Task DoWork(CancellationToken stoppingToken);
}
```

El método `ExecuteAsync` es el núcleo del servicio en segundo plano. Este método se ejecuta en un bucle hasta que se solicite la cancelación del servicio (a través del `CancellationToken`).

1. **Cálculo de la Próxima Ejecución:**
    - `nextOccurrence` se calcula utilizando la expresión CRON y la zona horaria, determinando cuándo debe ejecutarse la tarea a continuación.
    - Si no hay una próxima ocurrencia (`!nextOccurrence.HasValue`), el método simplemente retorna y detiene la ejecución.
2. **Espera hasta la Próxima Ejecución:**
    - Si hay una próxima ocurrencia, se calcula el `delay` entre el momento actual y el momento de la próxima ejecución.
    - Si el `delay` es mayor que cero, el servicio espera (`Task.Delay`) hasta ese momento. Esta espera se puede cancelar si se solicita la cancelación a través del `stoppingToken`.
3. **Ejecución de la Tarea:**
    - Una vez transcurrido el `delay`, se ejecuta el método abstracto `DoWork`, que debe ser implementado por cualquier clase que herede de `CronBackgroundJob`. Este método es donde se define la lógica que se desea ejecutar en el intervalo programado.
4. **Manejo de Errores:**
    - Se captura cualquier excepción que ocurra durante la ejecución de `DoWork` para que el servicio pueda manejar o registrar errores sin detener el servicio completo.
### Configuración del Job

Para correr un job basado en `CronBackgroundJob`, necesitamos configurar la expresión CRON y el huso horario a utilizar. Esto lo hacemos mediante la clase `CronSettings`:

```csharp
namespace BackgroundJob.Cron.Jobs;
 
public class CronSettings<T>
{
    public string CronExpression { get; set; } = default!;
    public TimeZoneInfo TimeZone { get; set; } = default!;
}
```

### Registro de Servicios

Para facilitar la integración de nuestros jobs con la configuración, creamos un método de extensión que registra las dependencias necesarias:

```csharp
namespace BackgroundJob.Cron.Jobs;

public static class CronBackgroundJobExtensions
{
    public static IServiceCollection AddCronJob<T>(this IServiceCollection services, Action<CronSettings<T>> options)
        where T: CronBackgroundJob
    {
        if (options == null)
        {
            throw new ArgumentNullException(nameof(options));
        }

        var config = new CronSettings<T>();
        options.Invoke(config);

        if (string.IsNullOrWhiteSpace(config.CronExpression))
        {
            throw new ArgumentNullException(nameof(CronSettings<T>.CronExpression));
        }

        services.AddSingleton<CronSettings<T>>(config);
        services.AddHostedService<T>();

        return services;
    }
}
```

Usamos el *Options Pattern* muy común en ASP.NET para registrar cada background job que necesitemos.

Es obligatorio que se indique una configuración por medio de `CronSettings<T>` y también es obligatorio tener una expresión cron.

### Ejemplo de Job: MySchedulerJob

A continuación, creamos un job simple que hereda de `CronBackgroundJob`:

```csharp
namespace BackgroundJob.Cron.Jobs;

public class MySchedulerJob : CronBackgroundJob
{
    private readonly ILogger<MySchedulerJob> _log;

    public MySchedulerJob(CronSettings<MySchedulerJob> settings, ILogger<MySchedulerJob> log)
        :base(settings.CronExpression, settings.TimeZone)
    {
        _log = log;
    }
    
    protected override Task DoWork(CancellationToken stoppingToken)
    {
        _log.LogInformation("Running... at {0}", DateTime.UtcNow);

        return Task.CompletedTask;
    }
}
```

Este job simplemente registra la fecha y hora en que se ejecuta, lo que nos permite verificar su correcto funcionamiento.

### Configuración Final en `Program.cs`

Por último, registramos nuestro job en el contenedor de dependencias:

```csharp
using BackgroundJob.Cron.Jobs;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCronJob<MySchedulerJob>(options => 
{
    // Corre cada minuto
    options.CronExpression = "* * * * *";
    options.TimeZone = TimeZoneInfo.Local;
});

var app = builder.Build();

app.Run();
```

Al ejecutar la aplicación, verás que el job se ejecuta cada minuto, tal como se especifica en la expresión CRON.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rcfkj1454tgawe1q6nug.png)

# Conclusión

Aunque existen soluciones maduras y robustas para manejar tareas en segundo plano, como HangFire y Azure Functions, una implementación sencilla basada en Hosted Services puede ser la opción ideal cuando necesitas algo ligero y fácil de mantener.

Si tus necesidades evolucionan hacia una mayor escalabilidad o resiliencia, considera migrar a Azure Functions o Hangfire, dependiendo de tus requisitos específicos.

# Referencias
- [Schedule Cron Jobs using HostedService in ASP.NET Core | by Changhui Xu | codeburst](https://codeburst.io/schedule-cron-jobs-using-hostedservice-in-asp-net-core-e17c47ba06)
- [PeriodicTimer: Temporizadores asíncronos en .NET 6 | Variable not found](https://www.variablenotfound.com/2022/02/periodictimer-temporizadores-asincronos.html)
- [HangfireIO/Cronos: Fully-featured .NET library for working with Cron expressions. Built with time zones in mind and intuitively handles daylight saving time transitions (github.com)](https://github.com/HangfireIO/Cronos)
- [PeriodicTimer Class (System.Threading) | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.periodictimer?view=net-6.0)

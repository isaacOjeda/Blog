# Introducción
En este artículo veremos como crear un servicio en segundo plano que se ejecutará según un programa de intervalos. Este será expresado como si fuera una tarea CRON de Linux, que son en esencia, tareas programadas.

> Nota 💡: Aquí encuentras el [código fuente](https://github.com/isaacOjeda/DevToPosts/tree/main/BackgroundJob.Cron)
> 

El formato cron se expresa de la siguiente manera:
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

Donde 5 o 6 caracteres representan el intervalo de tiempo en el que la tarea se ejecutará, ejemplo:

| **Expresión** | **Descripción** |
|---|---|
| * * * * * | Cada minuto | 
| 0 0 1 * * | A media noche, en día primero de cada mes |
| 0 0 * * MON-FRI | A las 0:00, de Lunes a Viernes | 

> Nota 💡: Si quieres conocer más, puedes visitar este repositorio [HangfireIO/Cronnos]([HangfireIO/Cronos: Fully-featured .NET library for working with Cron expressions. Built with time zones in mind and intuitively handles daylight saving time transitions (github.com)](https://github.com/HangfireIO/Cronos))

## Implementación en ASP.NET Core y Hosted Services

Antes de seguir, realizar este tipo de background services en asp.net core es cada vez más fácil, tan fácil que ya existen soluciones como [HangFire]([Hangfire – Background jobs and workers for .NET and .NET Core](https://www.hangfire.io/)) y [Azure Functions]([Timer trigger for Azure Functions | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-timer?tabs=in-process&pivots=programming-language-csharp)) que realizan este tipo de tareas (también basadas en formato CRON). Pero si de igual forma, quisieras aprender hacer tu implementación (a veces es mejor keep it simple) te recomiendo seguir leyendo 🤓.

Para comenzar, crearemos un proyecto web vacío o de consola, da igual ya que no utilizaremos ningún endpoint HTTP, pero puedes mezclarlos sin problema.

```bash
dotnet new web -o BackgroundJob.Cron
```

Instalamos la librería Cronos para poder _parsear_ expresiones CRON.

```
dotnet add package Cronos
```

### CronBackgroundJob (Opción 1)

Esta clase base y abstracta, será la que se encargará de ejecutar un proceso según un intervalo de tiempo, este intervalo será definido como ya lo hemos dicho, con una expresión CRON.

```csharp
using Cronos;

namespace BackgroundJob.Cron.Jobs;

public abstract class CronBackgroundJob : BackgroundService
{

    private PeriodicTimer? _timer;

    private readonly CronExpression _cronExpression;

    private readonly TimeZoneInfo _timeZone;

    public CronBackgroundJob(string rawCronExpression, TimeZoneInfo timeZone)
    {
        _cronExpression = CronExpression.Parse(rawCronExpression);
        _timeZone = timeZone;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        DateTimeOffset? nextOcurrence = _cronExpression.GetNextOccurrence(DateTimeOffset.UtcNow, _timeZone);

        if (nextOcurrence.HasValue)
        {          

            var delay = nextOcurrence.Value - DateTimeOffset.UtcNow;  
            _timer = new PeriodicTimer(delay);

            if (await _timer.WaitForNextTickAsync(stoppingToken))
            {            
                _timer.Dispose();
                _timer = null;

                await DoWork(stoppingToken);

                // Reagendamos
                await ExecuteAsync(stoppingToken);
            }
        }
    }

    protected abstract Task DoWork(CancellationToken stoppingToken);
}
```

- **PeriodicTimer**: Es un nuevo Timer que nos permite "esperar" el siguiente "Tick" del timer. Es decir, si queremos que el timer se ejecute cada 60 segundos, el método `WaitForNextTickAsync` estará en modo `await` hasta que hayan transcurrido esos 60 segundos. Este método regresa `true` si el intervalo se cumplió y nadie canceló la tarea, regresará `false` si el `stoppingToken` canceló la ejecución.
	- Creamos el timer con la diferencia de tiempo (`TimeSpan`) entre la fecha y hora actual y la fecha y hora de la siguiente ocurrencia, es decir: Si estamos **27/10/2022 15:00** y la siguiente ocurrencia es el **27/10/2022 16:00**, hay una diferencia de 1 hora (3600000 milisegundos), hasta que pase ese tiempo, el **PeriodicTimer** lanzará su _Next Tick_.
- **CronExpression**: Nos ayuda a entender una expresión cron, en este caso tenemos que darle una fecha y un uso horario (este opcional) para que se pueda determinar cuándo será la siguiente ocurrencia (o sea, la siguiente fecha y hora en que se debe de correr la tarea)
	- `GetNextOcurrence`: Regresa un DateTimeOffset con la fecha a futuro en donde toca ya correr la tarea.
- **WaitForNextTickAsync**: Este método genera un `Task` que se espera hasta que ocurra el siguiente Tick del Timer.
	- Cada ejecución liberamos el timer para que en el siguiente ciclo volverlo a crear con la siguiente ocurrencia del Cron, ya que esto no necesariamente será una espera estática o igual en cada Tick.
	- Un ejemplo es si pongo que corra de lunes a viernes a las 9PM, la ejecución de jueves a viernes esperará 24 horas, pero de viernes a lunes esperará 72 horas.
- **DoWork**: Este método abstracto será el que se ejecutará en cada ocurrencia, es abstracto porque cada Worker que hagamos, hará una tarea diferente.

Al terminar de correr el `DoWork` de forma recursiva, mandamos a llamar nuevamente la tarea para agendar la siguiente ejecución, esto durará por siempre o hasta que el **stoppingToken** diga lo contrario.

### Opción 2
Después de dar un poco de vueltas, me di cuenta que 

### CronSettings

Para poder correr el worker/job anterior, debemos de poder tener una expresión cron y aparte el uso horario que se quiera considerar.

```csharp
namespace BackgroundJob.Cron.Jobs;
 
public class CronSettings<T>
{
    public string CronExpression { get; set; } = default!;
    public TimeZoneInfo TimeZone { get; set; } = default!;
}
```

### CronBackgroundJobExtensions

Para hacer fácil esta integración entre los options y cada background job, es mejor crear este método de extensión que nos ayudará a registrar cada dependencia de cada job.

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

### MySchedulerJob

Este será el background job de ejemplo:

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

Realmente lo único que hace es escribir en los logs la fecha en la que se ejecutó y así poder comprobar que todo funciona.

### Program

Para finalizar, registramos las dependencias con la extensión que escribimos y vualá, ya podemos correr.

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

Y el resultado:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rcfkj1454tgawe1q6nug.png)

# Conclusión
A pesar de que ya existen soluciones que nos ayuda implementar este tipo de tareas, mantener las cosas simples a veces es la opción que necesitas por que la tarea es simple.

Si necesitas algo que escale, que sea resiliente, flexible a un costo de tiempo bajo, definitivamente te recomiendo irte por Azure Functions. Si no estás en Azure, puedes irte por Hangfire.

Pero si lo que necesitas son tareas programadas y no depender de Azure, los Hosted Services es una buena opción.

# Referencias
- [Schedule Cron Jobs using HostedService in ASP.NET Core | by Changhui Xu | codeburst](https://codeburst.io/schedule-cron-jobs-using-hostedservice-in-asp-net-core-e17c47ba06)
- [PeriodicTimer: Temporizadores asíncronos en .NET 6 | Variable not found](https://www.variablenotfound.com/2022/02/periodictimer-temporizadores-asincronos.html)
- [HangfireIO/Cronos: Fully-featured .NET library for working with Cron expressions. Built with time zones in mind and intuitively handles daylight saving time transitions (github.com)](https://github.com/HangfireIO/Cronos)
- [PeriodicTimer Class (System.Threading) | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.periodictimer?view=net-6.0)

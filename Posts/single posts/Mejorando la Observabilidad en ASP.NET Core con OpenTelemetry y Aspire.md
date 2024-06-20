
## Introducción

En el ámbito del desarrollo de aplicaciones modernas, la observabilidad se ha vuelto crucial para garantizar el rendimiento y la fiabilidad de nuestros sistemas distribuidos. OpenTelemetry ha surgido como un estándar para recopilar datos de telemetría en tiempo real. En este artículo, exploraremos cómo integrar OpenTelemetry en una aplicación ASP.NET Core con Blazor, con un enfoque central en el uso de Aspire. Aspire ofrece un Dashboard potente que nos permite visualizar y analizar datos de telemetría, facilitando la monitorización y el diagnóstico del comportamiento de nuestras aplicaciones en tiempo real. Utilizaremos un ejemplo práctico para demostrar su implementación y cómo puede potenciar nuestra capacidad de observabilidad.
## Dashboard standalone de Aspire

.NET Aspire está emergiendo como una herramienta sumamente poderosa para el desarrollo de aplicaciones _cloud native_ en el ecosistema .NET, además de otras plataformas.

Es fascinante observar lo sencillo que resulta comenzar a desarrollar con Aspire, abordando muchos aspectos que suelen ser tediosos o complicados de manejar. Sin embargo, ¿qué sucede con las aplicaciones ya existentes? No siempre es viable cambiar por completo el enfoque y adoptar Aspire como orquestador en nuestro entorno de desarrollo. Sin embargo, resulta tentador aprovechar las características que ofrece su dashboard.

Es posible que en aplicaciones ya establecidas, ya sean distribuidas o no, ya hayamos resuelto la forma de ejecutarlas. No obstante, Aspire y su dashboard nos brindan una herramienta sumamente valiosa durante la fase de desarrollo.

Contar con un panel de control que nos proporcione logs, métricas y traces es de gran ayuda para comprender nuestra aplicación y cómo se comporta al interactuar con servicios, bases de datos u otros componentes.

Por eso, el hecho de que el dashboard pueda utilizarse con un receptor OTLP (OpenTelemetry Protocol) y visualizar todos estos datos de manera integrada fue lo que realmente me convenció para explorar su uso.

Para ejecutar el Dashboard de Aspire, podemos utilizar el siguiente comando de Docker:

```bash
docker run --rm -it -p 18888:18888 -p 4317:18889 -d --name aspire-dashboard mcr.microsoft.com/dotnet/nightly/aspire-dashboard:8.0.0-preview.4
```

Este comando creará y ejecutará un contenedor Docker llamado `aspire-dashboard`, que aloja el Dashboard de Aspire. El Dashboard estará disponible en el puerto `18888`, mientras que el puerto para el OTLP será el `4317`.

Una vez que el contenedor esté en ejecución, puedes acceder al Dashboard navegando a `http://localhost:18888/` en tu navegador web. Ten en cuenta que inicialmente el Dashboard no mostrará datos hasta que una aplicación comience a enviar información a través del protocolo OTLP.

El Dashboard de Aspire es una herramienta versátil que puede utilizarse con cualquier aplicación que exporte sus datos utilizando el protocolo OTLP. Esto significa que no importa la plataforma en la que estés desarrollando (ya sea .NET, Java, Python, Go, etc.), siempre y cuando puedas configurar tu aplicación para enviar datos mediante OTLP, podrás visualizar y analizar esa información en el Dashboard de Aspire.

> Nota 💡: Como costumbre, el código fuente: [DevToPosts/AspireDashboard/OpenTelemetryExample · isaacOjeda/DevToPosts (github.com)](https://github.com/isaacOjeda/DevToPosts/tree/main/AspireDashboard/OpenTelemetryExample)

### Ejemplo de Aplicación Web

En este ejemplo, crearemos una aplicación web utilizando Blazor Server Side Rendering, utilizando la configuración por defecto que proporciona ASP.NET Core, incluyendo páginas de ejemplo y demás. Luego, agregaremos los siguientes paquetes necesarios para integrar OpenTelemetry:

```xml
<PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.7.0" /> <PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.7.0" /> <PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.7.1" /> <PackageReference Include="OpenTelemetry.Instrumentation.Http" Version="1.7.1" /> <PackageReference Include="OpenTelemetry.Instrumentation.Runtime" Version="1.7.0" />
```

Es importante destacar que todo lo que exploraremos en este artículo es aplicable a cualquier tipo de aplicación desarrollada en ASP.NET Core.

Si deseas comprender primero cómo funciona OpenTelemetry, ya he escrito un artículo al respecto disponible en [este enlace](https://dev.to/isaacojeda/aspnet-core-monitoreo-con-opentelemetry-y-grafana-57m9). Además, su [documentación oficial](https://opentelemetry.io/docs/languages/net/) es bastante detallada y útil para obtener una comprensión más profunda.
### Consumiendo una API externa

Para ilustrar un ejemplo claro de traces, vamos a consumir una API pública y gratuita de Pokémon (¡yeei!).

El procedimiento es simple: crearemos un servicio llamado `PokemonService` para consultar todos los Pokémon disponibles utilizando paginación. Luego, utilizaremos este servicio para mostrar los datos consultados en un componente de Blazor.

```csharp
using System.Diagnostics;
 
namespace OpenTelemetryExample;
 
public class PokemonService(ILogger<PokemonService> logger, HttpClient http)
{
    public const string ActivitySourceName = "DownloadPokemon";
    private static readonly ActivitySource ActivitySource = new(ActivitySourceName);
 
    public async Task<PokemonList> GetPokemonAsync(int pageSize, int currentPage)
    {
        using var activity = ActivitySource.StartActivity("GetPokemon", ActivityKind.Producer);
 
        var url = $"https://pokeapi.co/api/v2/pokemon?limit={pageSize}&offset={currentPage * pageSize}";
        var list = await http.GetFromJsonAsync<PokemonList>(url);
 
        return list ?? new PokemonList
        {
            Next = null,
            Previous = null,
            Results = []
        };
    }
}
 
public class Pokemon
{
    public required string Name { get; init; }
    public required string Url { get; init; }
}
 
public class PokemonList
{
    public int Count { get; init; }
    public required string? Next { get; init; }
    public required string? Previous { get; init; }
    public required List<Pokemon> Results { get; init; }
}
```

La clase `ActivitySource` se utiliza para generar actividades de traces, lo que nos permite registrar eventos relacionados con la descarga de datos de la API de Pokémon. Estos traces pueden ser útiles para entender el rendimiento de la aplicación y depurar posibles problemas.

Dentro del método `GetPokemonAsync`, iniciamos una actividad del trace llamada "GetPokemon" utilizando `ActivitySource.StartActivity`. Esto nos permite registrar el inicio y la finalización de la operación de obtención de datos. Luego, hacemos una solicitud HTTP a la API de Pokémon para obtener una lista de Pokémon, y devolvemos los resultados obtenidos.

Te invito a revisar el código fuente para explorar el componente de Blazor que utiliza el servicio `PokemonService`. Sin embargo, por cuestiones de relevancia para este post, he omitido incluirlo directamente.

Es importante destacar que este ejemplo es solo una muestra y puedes implementar cualquier funcionalidad para probar. Incluso un Endpoint de Minimal API podría ser suficiente para demostrar el funcionamiento del sistema. En este caso particular, el objetivo era demostrar la consulta a un servicio externo y cómo integrarlo en una aplicación Blazor.

### Program.cs

Ahora que hemos configurado el servicio y la vista de ejemplo, el siguiente paso es ajustar el programa principal para configurar OpenTelemetry y todos los exportadores necesarios (de logs, métricas y traces).

Es importante destacar que en este punto no estamos limitados a utilizar exclusivamente Aspire. Dado que todo el sistema es agnóstico en cuanto a la implementación, podríamos optar por utilizar servicios compatibles con el Protocolo OpenTelemetry, como Application Insights o Datadog, según nuestras necesidades y preferencias.

El enfoque que seguiremos es utilizar Aspire en modo desarrollo y, en producción, cualquier opción que esté lista para producción. Esta flexibilidad es una de las características más destacadas, ya que nos permite adaptarnos a diferentes entornos sin estar ligados a una sola solución.

A continuación, presentamos cómo quedará configurado el archivo `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);
 
ConfigureOpenTelemetry(builder);
 
builder.Services.AddHttpClient<PokemonService>();
builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents();
 
var app = builder.Build();
 
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error", createScopeForErrors: true);
    app.UseHsts();
}
 
app.UseHttpsRedirection();
 
app.UseStaticFiles();
app.UseAntiforgery();
 
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode();
 
app.Run();
```

El método `ConfigureOpenTelemetry` es el siguiente:

```csharp
static void ConfigureOpenTelemetry(IHostApplicationBuilder builder)
{
    builder.Logging.AddOpenTelemetry(logging =>
    {
        logging.IncludeFormattedMessage = true;
        logging.IncludeScopes = true;
    });
 
    builder.Services.AddOpenTelemetry()
        .ConfigureResource(c => c.AddService("OpenTelemetryExample"))
        .WithMetrics(metrics =>
        {
            metrics.AddHttpClientInstrumentation()
                .AddRuntimeInstrumentation();
        })
        .WithTracing(tracing =>
        {
            if (builder.Environment.IsDevelopment())
            {
                // We want to view all traces in development
                tracing.SetSampler(new AlwaysOnSampler());
            }
            tracing.AddAspNetCoreInstrumentation();
            tracing.AddHttpClientInstrumentation();
            tracing.AddSource(PokemonService.ActivitySourceName);
        });
 
    // Use the OTLP exporter if the endpoint is configured.
    var useOtlpExporter = !string.IsNullOrWhiteSpace(builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]);
    if (useOtlpExporter)
    {
        builder.Services.Configure<OpenTelemetryLoggerOptions>(logging => logging.AddOtlpExporter());
        builder.Services.ConfigureOpenTelemetryMeterProvider(metrics => metrics.AddOtlpExporter());
        builder.Services.ConfigureOpenTelemetryTracerProvider(tracing => tracing.AddOtlpExporter());
    }
}
```

En este método, definimos cómo queremos gestionar los logs, traces y metrics utilizando las bibliotecas que hemos agregado previamente.

En primer lugar, configuramos los registros (logs) para incluir el mensaje formateado y los ámbitos. Esto nos permite capturar información detallada sobre el estado de la aplicación y los eventos ocurridos durante su ejecución.

Luego, configuramos el sistema de métricas para recopilar datos sobre el rendimiento de la aplicación, como el tiempo de respuesta de las solicitudes HTTP y el rendimiento del tiempo de ejecución. Esto nos ayuda a monitorear y diagnosticar el comportamiento de nuestra aplicación en tiempo real.

Para los traces, establecemos un muestreador que determina qué traces se deben capturar y enviar. En entornos de desarrollo, configuramos un muestreador que captura todas los traces para facilitar la depuración y el análisis. Además, agregamos instrumentación específica para HttpClient y para nuestra clase `PokemonService`, lo que nos permite rastrear las solicitudes HTTP y las operaciones realizadas en nuestro servicio.

Finalmente, configuramos los exportadores OTLP (OpenTelemetry Protocol) para enviar los datos recopilados a un destino específico. Si se ha configurado un endpoint OTLP, utilizamos los exportadores OTLP para enviar registros, métricas y traces. Esto nos proporciona la flexibilidad para dirigir los datos de OpenTelemetry a diferentes sistemas de análisis y monitoreo, como Prometheus, Zipkin o Loki, siguiendo el estándar OTLP.

En resumen, el método `ConfigureOpenTelemetry` establece la base para la instrumentación y observabilidad de nuestra aplicación, permitiéndonos recopilar datos valiosos sobre su rendimiento y comportamiento en tiempo real, y enviarlos a destinos de análisis externos según nuestras necesidades y preferencias.

### Probando la Solución

Ya podemos correr la aplicación de Blazor y Aspire juntos, y empezaremos a ver los logs de la aplicación:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/k65w50xd4jlafiw2k23w.png)

> Nota 💡: Si revisas el código fuente, yo estoy utilizando `docker-compose` por medio de visual studio, es fácil de usar, pero por eso salen por default esos warnings y errores en el startup.

Utilicemos la aplicación para comenzar a generar métricas:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3v8cdpd8x6hb6c5yl9bl.png)

Veremos la consulta de los Pokemons, pero en Aspire nos vamos a los Traces y veremos en efecto el flujo de lo que acaba de suceder:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5yffvgoouqqwrdhb0kf3.png)

Ejemplo de las métricas:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o49j4mojv3ab97xvsvrp.png)

Aquí otro ejemplo de como se vería con un endpoints de Minimal API:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kwjffc7rwtqqae1i0t3c.png)

## Conclusión

En este artículo, hemos explorado cómo integrar OpenTelemetry en una aplicación ASP.NET Core utilizando Blazor, destacando el papel central de Aspire en nuestro sistema de observabilidad. Utilizando un enfoque práctico, demostramos cómo configurar OpenTelemetry, consumir una API externa y visualizar los datos recopilados en el potente Dashboard de Aspire. Esta plataforma nos ofrece una interfaz intuitiva para monitorear y analizar datos de telemetría en tiempo real, lo que nos permite identificar y resolver problemas de manera proactiva. Con la combinación de OpenTelemetry y Aspire, podemos mejorar significativamente la observabilidad de nuestras aplicaciones, optimizar su rendimiento y proporcionar una experiencia excepcional a los usuarios finales.

## Referencias
- [.NET Aspire dashboard is the best tool to visualize your OpenTelemetry data during local development - DEV Community](https://dev.to/asimmon/net-aspire-dashboard-is-the-best-tool-to-visualize-your-opentelemetry-data-during-local-development-9dl)
- [Standalone Aspire dashboard sample app - Code Samples | Microsoft Learn](https://learn.microsoft.com/en-us/samples/dotnet/aspire-samples/aspire-standalone-dashboard/)
- [Getting Started | OpenTelemetry](https://opentelemetry.io/docs/languages/net/getting-started/)

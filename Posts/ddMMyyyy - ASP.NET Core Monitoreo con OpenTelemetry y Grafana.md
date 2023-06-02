## Introducción


## APM (Application Performance Monitoring)
Ya hemos hablado en este blog sobre monitoreo de [aplicaciones utilizando application insights](https://dev.to/isaacojeda/parte-11-aspnet-core-application-insights-y-serilog-3103), lo cual es cierto que ya es una solución completa y es la que yo uso todos los días para monitorear los proyectos en los que participo. Pero no todos usan Azure, así que hoy veremos otras opciones para poder implementar un APM.

Las ventajas de tener un monitoreo eficiente, es poder detectar anomalías y saber responder a ellas, la idea siempre será mejorar el software para que la experiencia de nuestros usuarios sea la mejor.

Tener la mayor visibilidad de como opera nuestro sistema nos da una gran ventaja y es algo que todos deberíamos de tener en nuestros proyectos y hoy lo veremos con puro Open Source.

Lo que debe de tener un APM es lo siguiente:

**Logs**
Siempre nuestras aplicaciones deben de generar logs, ya sean de error o informativos. Estos siempre nos ayudarán a entender en donde falla nuestra aplicación y será más fácil diagnosticar los problemas.

En desarrollo por lo general todos los logs están activados, pero en producción lo más normal es solo mostrar los Warnings/Criticals:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0b3pnkefdh56oa1qfm42.png)
_Ejemplo de logs con Loki y Grafana_

Poder consultar los logs sí o sí nos ayudarán a darnos cuenta de muchos problemas que nuestra aplicación pudiera tener y los desarrolladores podrán solucionar los bugs sin que batallen tanto. Por que por ejemplo, los `StackTrace` y `Exceptions` que ocurren, siempre se loggean y sin problema los podemos consultar para saber donde falló la aplicación. 

**Tracing**
Un trace es un grupo de operaciones o transacciones con [spans](https://www.elastic.co/guide/en/apm/guide/current/data-model-spans.html) que fueron originados por un HTTP Request.

Es muy común que las aplicaciones web se comuniquen con otros servicios o bases de datos por lo que, un trace completo nos ayudaría a seguir toda esa línea de ejecución sin importar que esta se encuentre distribuida en distintos servicios y así fácilmente poder saber donde hay problemas.

Un ejemplo:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/q633vkhrilz2qwcdf53v.png)

Aquí vemos que el request inicial fue `GET /dashboard` y cada servicio generó sus propios spans pero de un mismo trace original, aquí fácilmente podemos localizar en donde falló un request o que es lo que está lento cuando hay algo mal en algún servicio.

**Metrics**
Las métricas también son parte muy importante, estas nos ayudarán conocer muchas cosas sobre nuestra aplicación, como: Uso de CPU, memoria disponible, solicitudes fallidas, tiempo de respuesta por solicitud, etc.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p8n1kuqf90qa9c10mbci.png)

### Infraestructura
Las tecnologías que utilizaremos son las siguientes:
- **Prometheus**
	- Este será nuestra base de datos y el que recolectará toda la información para las **métricas**
- **Loki**
	- Es una base de datos dedicado a los **logs**, este recibirá y guardará los logs de una manera que se puedan obtener después
- **Zipkin**
	- Nos permitirá guardar todos los `traces` y `spans` que la aplicación (o aplicaciones) generen
- **Grafana**
	- Grafana nos ayudará a mostrar toda la información recolectada por los anteriores, aquí podemos crear dashboards y explorar toda la información (aunque zipkin tiene su propia UI y Prometheus también). Los dashboards van a incluir gráficas similares a las capturas mostradas anteriormente y es totalmente libre de configurar a las necesidades de cada quien.
- **OpenTelemetry Collector**: 
	- Este es opcional, pero es recomendado. El OTEL Collector es un intermediario entre mi aplicación y todos los servicios anteriores que hemos mencionado, nuestra aplicación no sabrá a donde va toda la información, solo le importa que se va a comunicar con un protocolo especial de OpenTelemetry y el OTEL Collector se encargará de mandarlo a cada servicio según se requiera.
	- Esto es muy útil cuando tenemos muchos servicios, solo configuramos el OTEL (autenticación, certificados, cosas tediosas) pero las aplicaciones de igual forma solo mandan su información al collector sin preocuparse de los detalles.
	- También es útil cuando quisiéramos cambiar de proveedor de APM, nuestra aplicación no se enterará de nada, solo actualizamos el Collector y ya.
		- Muchos APMs ya soportan OpenTelemetry por que ya es un estándar, por lo que es una muy buena ventaja sí utilizar este collector.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ca59ygk9o2dgtdjzqwcd.png)

Lo que haremos a continuación es lo siguiente:

- Tendremos una Web API que se comunicará con una API Pública del Clima.
- La Web API mandará los Logs, Traces y Metrics por medio del protocolo OTLP de OpenTelemetry y se los mandará al OTEL Collector.
- El OTEL Collector los procesará y los mandará a sus exporters (Loki, Prometheus y Zipkin).
- Configuraremos Grafana para visualizar dashboards con las métricas y logs
- Usaremos Zipkin para ver todos los traces

Comencemos.

#### Docker Compose

Lo primero que necesitamos es configurar la infraestructura, no veremos los detalles de la aplicación Web API pero siempre puedes ver el código fuente para basarte en ello.

```yaml
version: "3"
  
services:
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - opentelemetry
  
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    command: --config.file=/etc/prometheus/prometheus.yaml --storage.tsdb.path=/prometheus
    volumes:
      - ./prometheus.yaml:/etc/prometheus/prometheus.yaml
      - ./tmp/prometheus:/prometheus
    networks:
      - opentelemetry
  
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
      - GF_AUTH_DISABLE_LOGIN_FORM=true
    volumes:
      - ./tmp/grafana/:/var/lib/grafana/    
      - ./grafana-datasource.yaml:/etc/grafana/provisioning/datasources/ds.yaml
    depends_on:
      - prometheus
      - loki
    networks:
      - opentelemetry
  
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    ports:
      - "4317:4317"
      - "4318:4318"
    command: --config=/etc/otel-collector-config.yaml
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    networks:
      - opentelemetry
  
  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"
    networks:
      - opentelemetry
  
networks:
  opentelemetry

    name: opentelemetry-network
```

Resumen por servicio:
- **loki**:
	- Configuración default, de hecho, escrito por Github Copilot (👍🏻)
	- Imagen docker
- prometheus:
- otel-collector:
- zipkin:

##### grafana-datasource.yaml

```yaml
apiVersion: 1
  
datasources:
- name: Prometheus
  type: prometheus
  access: proxy
  orgId: 1
  url: http://prometheus:9090
  basicAuth: false
  isDefault: true
  version: 1
  editable: false
  
- name: Loki
  type: loki
  access: proxy
  orgId: 1
  url: http://loki:3100
  basicAuth: false
  isDefault: true
  version: 1
  editable: false
  
- name: Zipkin
  type: zipkin
  access: proxy
  orgId: 1
  url: http://zipkin:9411
  basicAuth: false
  isDefault: true
  version: 1
  editable: false
```

##### loki-config.yaml

```yaml
auth_enabled: false
  
server:
  http_listen_address: 0.0.0.0
  grpc_listen_address: 0.0.0.0
  http_listen_port: 3100
  
schema_config:
  configs:
    - from: 2020-04-15
      store: boltdb
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 168h
```

#####  prometheus.yaml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  
scrape_configs:
  - job_name: 'otelcollector'
    static_configs:
      - targets: [ 'otel-collector:8889' ]
```

#####  otel-collector-config.yaml

```yaml
receivers:
  otlp:
    protocols:
      http:
      grpc:
  
processors:
  attributes:
    actions:
      - action: insert
        key: loki.attribute.labels
        value: event.domain
  resource:
    attributes:
      - action: insert
        key: loki.resource.labels
        value: service.name
  
  batch:
    timeout: 1s
    send_batch_size: 1024
  
exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
    send_timestamps: true
    resource_to_telemetry_conversion:
      enabled: true
    # const_labels:
    #   exported: "collector"
  
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
    tls:
      insecure: true
  
  zipkin:
    endpoint: http://zipkin:9411/api/v2/spans
  
service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
  
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [zipkin]
  
    logs:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [loki]
```


### Aplicación ASP.NET

#### NuGets a usar

```xml
<PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="7.0.5" />
<PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
<PackageReference Include="Microsoft.Extensions.Http" Version="7.0.0" />

<PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.5.0-rc.1" />
<PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.5.0-rc.1" />
<PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.0.0-rc9.14" />
<PackageReference Include="OpenTelemetry.Instrumentation.EventCounters" Version="1.0.0-alpha.2" />
<PackageReference Include="OpenTelemetry.Instrumentation.Runtime" Version="1.1.0-rc.2" />
<PackageReference Include="OpenTelemetry.Instrumentation.Http" Version="1.0.0-rc9.14" />
<PackageReference Include="OpenTelemetry.Instrumentation.Process" Version="0.5.0-beta.2" />
  
<PackageReference Include="Serilog.AspNetCore" Version="7.0.0" />
<PackageReference Include="Serilog.Enrichers.Thread" Version="3.1.0" />
<PackageReference Include="Serilog.Sinks.Console" Version="4.1.0" />
<PackageReference Include="Serilog.Enrichers.Environment" Version="2.2.0" />
<PackageReference Include="Serilog.Sinks.OpenTelemetry" Version="1.0.0-dev-00208" />
```

TODO: Explicarlos

#### Instrumentor

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;
  
namespace WebApi;
  
public sealed class Instrumentor : IDisposable
{
    public const string ServiceName = "WebApi";
    public ActivitySource Tracer { get; }
    public Meter Recorder { get; }
    public Counter<long> IncomingRequestCounter { get; }
  
    public Instrumentor()
    {
        var version = typeof(Instrumentor).Assembly.GetName().Version?.ToString();
        Tracer = new ActivitySource(ServiceName, version);
        Recorder = new Meter(ServiceName, version);
        IncomingRequestCounter = Recorder.CreateCounter<long>("app.incoming.requests",
            description: "The number of incoming requests to the backend API");
    }
  
    public void Dispose()
    {
        Tracer.Dispose();
        Recorder.Dispose();
    }
}
```

#### Configuración de Logs

```csharp
const string outputTemplate =
    "[{Level:w}]: {Timestamp:dd-MM-yyyy:HH:mm:ss} {MachineName} {EnvironmentName} {SourceContext} {Message}{NewLine}{Exception}";
  
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .Enrich.FromLogContext()
    .Enrich.WithThreadId()
    .Enrich.WithEnvironmentName()
    .Enrich.WithMachineName()
    .WriteTo.Console(outputTemplate: outputTemplate)
    .WriteTo.OpenTelemetry(opts =>
    {
        opts.ResourceAttributes = new Dictionary<string, object>
        {
            ["app"] = "webapi",
            ["runtime"] = "dotnet",
            ["service.name"] = "WebApi"
        };
    })
    .CreateLogger();
  
builder.Host.UseSerilog();
```

#### Configuración de Tracing y Métricas

```csharp
builder.Services.AddSingleton<Instrumentor>();
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddSource(Instrumentor.ServiceName)
        .ConfigureResource(resource => resource
            .AddService(Instrumentor.ServiceName))
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .AddMeter(Instrumentor.ServiceName)
        .ConfigureResource(resource => resource
            .AddService(Instrumentor.ServiceName))
        .AddRuntimeInstrumentation()
        .AddAspNetCoreInstrumentation()
        .AddProcessInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEventCountersInstrumentation(c =>
            {
                c.AddEventSources(
                    "Microsoft.AspNetCore.Hosting",
                    "Microsoft-AspNetCore-Server-Kestrel",
                    "System.Net.Http",
                    "System.Net.Sockets");
            })
        .AddOtlpExporter());
```


#### /api/weather-forecast endpoint

```csharp
app.MapGet("/api/weather-forecast", async (HttpClient http, Instrumentor instrumentor) =>
{
    instrumentor.IncomingRequestCounter.Add(1,
        new KeyValuePair<string, object?>("operation", "GetWeatherForecast"),
        new KeyValuePair<string, object?>("minimal-api-route", "/api/weather-forecast"));
  
    var url = "https://api.open-meteo.com/v1/forecast?latitude=28.68&longitude=-106.04&hourly=temperature_2m";
  
    var response = await http.GetAsync(url);
  
    response.EnsureSuccessStatusCode();
  
    return await response.Content.ReadFromJsonAsync<WeatherForecast>();
});
```


### Dashboards en Grafana

## Conclusión

## Referencias
- [Guide to OpenTelemetry (logz.io)](https://logz.io/learn/opentelemetry-guide/?utm_source=substack&utm_medium=email)
- [Instrumenting C# .Net Apps With OpenTelemetry | Logz.io](https://logz.io/blog/csharp-dotnet-opentelemetry-instrumentation/#addt)
- [bradygaster/dotnet-cloud-native-build-2023 (github.com)](https://github.com/bradygaster/dotnet-cloud-native-build-2023)
- [cecilphillip/grafana-otel-dotnet: Sample setup showing ASP.NET Core observability with Prometheus, Loki, Grafana, Opentelemetry Collector (github.com)](https://github.com/cecilphillip/grafana-otel-dotnet)
- [open-telemetry/opentelemetry-dotnet: The OpenTelemetry .NET Client (github.com)](https://github.com/open-telemetry/opentelemetry-dotnet)
- [.NET | OpenTelemetry](https://opentelemetry.io/docs/instrumentation/net/)
- 
# ASP.NET Core y el Patrón Decorador: Ampliando la Funcionalidad de tus APIs

# Introducción

En la programación, el "Decorator Pattern" (Patrón Decorador) mejora aplicaciones sin complicaciones. Exploraremos su uso en Web APIs con ASP.NET Core. Añadiremos funcionalidad extra sin tocar el código original. Ejemplos incluyen caché y registros, mejorando la flexibilidad y potencia.
## El Patrón Decorador

Este patrón mejora objetos sin modificarlos. Añade capas de funcionalidad sin afectar otras ni el objeto original. Cada decorador se enfoca en una tarea específica, promoviendo un código modular. Flexibilidad para mezclar y combinar decoradores según necesites. Facilita la mantenibilidad a largo plazo.

**Ejemplo práctico: API del Clima con Decoradores**

Exploraremos el patrón decorador en una Web API ASP.NET Core. Crearemos un servicio que consulta datos meteorológicos de la API de Open Meteor.

**Servicio Base:**
Inicialmente, este servicio consumirá la API de Open Meteor para proporcionar datos climáticos.

**Decorador de Caché:**
Optimizará las llamadas a la API almacenando temporalmente resultados en caché y devolviéndolos si la solicitud se repite, sin alterar el código original.

**Decorador de Logs:**
Registrará cada llamada a la API sin afectar la lógica fundamental del servicio base.

**Resumen:**
Los decoradores enriquecen una Web API sin modificar su núcleo. Utilizaremos un servicio de consulta de datos meteorológicos como ejemplo para agregar caché y registros. Esto demuestra la versatilidad y capacidad de extensión del patrón decorador en nuestras aplicaciones web.

¡Exploraremos la implementación y cómo estos decoradores mejoran la eficiencia y el seguimiento de nuestra Web API!
### Proyecto Web API con .NET 8

En este post, he optado por utilizar .NET 8, ya que esta versión introduce una característica extremadamente útil conocida como `KeyedServices`.

Los `KeyedServices` nos brindan la capacidad de registrar dependencias en el `ServiceProvider` y, al mismo tiempo, asignar un nombre distintivo a cada una de ellas. Posteriormente, podemos solicitar estas dependencias utilizando el nombre asociado. Esta característica se vuelve particularmente valiosa en escenarios como el que estamos abordando en este tutorial, donde necesitamos registrar múltiples dependencias del mismo tipo, pero deseamos resolverlas de manera específica en función de la cadena de decoradores que estamos construyendo. A medida que avances en este tutorial, comprenderás mejor cómo esta funcionalidad simplifica y mejora la implementación de patrones decoradores en ASP.NET Core.

#### IWeatherService

Para comenzar, necesitamos definir nuestro contrato:
```csharp
namespace DecoratorPattern.Interfaces;  
  
public interface IWeatherService  
{  
    Task<WeatherForecast?> GetWeatherForecastAsync(double latitude, double longitude);  
}
```

El modelo `WeatherForecast` es el siguiente:

```csharp
public record WeatherForecast(  
    [property: JsonPropertyName("latitude")] double Latitude,  
    [property: JsonPropertyName("longitude")] double Longitude,  
    [property: JsonPropertyName("generationtime_ms")] double GenerationtimeMs,  
    [property: JsonPropertyName("utc_offset_seconds")] int UtcOffsetSeconds,  
    [property: JsonPropertyName("timezone")] string Timezone,  
    [property: JsonPropertyName("timezone_abbreviation")] string TimezoneAbbreviation,  
    [property: JsonPropertyName("elevation")] double Elevation,  
    [property: JsonPropertyName("hourly_units")] HourlyUnits HourlyUnits,  
    [property: JsonPropertyName("hourly")] Hourly Hourly  
);  
  
public record Hourly(  
    [property: JsonPropertyName("time")] IReadOnlyList<string> Time,  
    [property: JsonPropertyName("temperature_2m")] IReadOnlyList<double> Temperature2m  
);  
  
public record HourlyUnits(  
    [property: JsonPropertyName("time")] string Time,  
    [property: JsonPropertyName("temperature_2m")] string Temperature2m  
);
```

> Nota :💡 Se ve un poco feo, pero quise hacer la prueba y para que aprendan que así se pueden hacer `records` con `Attributes` (te sugiero que hagas tus `records` o `classes` como mejor te parezca 🤭).

#### WeatherService (Servicio base / original)

```csharp
using DecoratorPattern.Interfaces;  
  
namespace DecoratorPattern.Services;  
  
public class WeatherService : IWeatherService  
{  
    private readonly IHttpClientFactory _httpClientFactory;  
  
    public WeatherService(IHttpClientFactory httpClientFactory)  
    {
		_httpClientFactory = httpClientFactory;  
    }  
    public async Task<WeatherForecast?> GetWeatherForecastAsync(double latitude, double longitude)  
    {   
		var httpClient = _httpClientFactory.CreateClient();  
		httpClient.BaseAddress = new Uri("https://api.open-meteo.com");  
  
        var response =  
            await httpClient.GetFromJsonAsync<WeatherForecast>(  
                $"v1/forecast?latitude={latitude}&longitude={longitude}&hourly=temperature_2m");  
  
        if (response is null)  
        {
			throw new Exception("Unable to retrieve weather forecast.");  
        }  

        return response;  
    }}
```

**Explicación del Código:**

1. La clase `WeatherService` implementa la interfaz `IWeatherService`. Esta clase se encarga de obtener pronósticos del clima a través de una API externa.
2. **Constructor del Servicio:**
    - El constructor de la clase `WeatherService` toma un parámetro `IHttpClientFactory`, que se utiliza para crear instancias de clientes HTTP. Esto facilita la configuración y gestión de solicitudes HTTP.
3. **Método `GetWeatherForecastAsync`:**
    - Este método toma las coordenadas de latitud y longitud como argumentos y devuelve un pronóstico del clima como un objeto `WeatherForecast`.
    - Se configura la dirección base del cliente HTTP para apuntar a la API externa de Open Meteo.
    - El resultado se deserializa en un objeto `WeatherForecast` utilizando `GetFromJsonAsync`.

#### CachedWeatherService

```csharp
using DecoratorPattern.Interfaces;  
using Microsoft.Extensions.Caching.Memory;  
  
namespace DecoratorPattern.Services;  
  
public class CachedWeatherService : IWeatherService  
{  
    private readonly IWeatherService _weatherService;  
    private readonly IMemoryCache _memoryCache;  
  
    public CachedWeatherService(
	    [FromKeyedServices("WeatherService")] IWeatherService weatherService,  
        IMemoryCache memoryCache)  
    {   
        _weatherService = weatherService;  
        _memoryCache = memoryCache;  
    }  
    public async Task<WeatherForecast?> GetWeatherForecastAsync(double latitude, double longitude)  
    {        
	    var key = $"lat={latitude}&lon={longitude}";  
  
        if (_memoryCache.TryGetValue(key, out WeatherForecast? weatherForecast))  
        {            
	        return weatherForecast;  
        }  
        weatherForecast = await _weatherService.GetWeatherForecastAsync(latitude, longitude);  
  
        _memoryCache.Set(key, weatherForecast, TimeSpan.FromMinutes(15));  
  
        return weatherForecast;  
    }
}
```

**Explicación del Código:**

1. La clase `CachedWeatherService` implementa la interfaz `IWeatherService`. Este servicio extiende la funcionalidad del servicio de pronóstico del clima al agregarle un mecanismo de caché.
2. **Constructor del Servicio con Caché:**
    - El constructor de la clase `CachedWeatherService` toma dos parámetros:
        - `[FromKeyedServices("WeatherService")] IWeatherService weatherService`: Esto indica que estamos inyectando la dependencia `IWeatherService` con la clave "WeatherService", lo que nos permite resolver específicamente el servicio base.
        - `IMemoryCache memoryCache`: Se inyecta una instancia de la interfaz `IMemoryCache`, que se utilizará para almacenar en caché los resultados de las solicitudes.
3. **Método `GetWeatherForecastAsync` con Caché:**
    - En este método, se genera una clave única basada en las coordenadas de latitud y longitud que se utilizan como argumentos de entrada.
    - Se verifica si el pronóstico del clima está en la caché utilizando la clave generada.
    - Si el pronóstico está en la caché, se devuelve directamente desde la caché.
    - Si no está en caché, se llama al servicio base `_weatherService` para obtener el pronóstico del clima.
    - El resultado se coloca en la caché con una duración de 15 minutos utilizando `_memoryCache.Set`.
    - Finalmente, se devuelve el pronóstico del clima obtenido, que puede ser tanto de la caché como del servicio base.

Este código demuestra cómo el patrón decorador puede utilizarse para extender la funcionalidad de un servicio base sin modificar su código, en este caso, agregando una capa de caché para mejorar el rendimiento de las solicitudes de pronóstico del clima.

#### LogWeatherService

```csharp
using DecoratorPattern.Interfaces;  
  
namespace DecoratorPattern.Services;  
  
public class LogWeatherService : IWeatherService  
{  
    private readonly IWeatherService _weatherService;  
    private readonly ILogger<LogWeatherService> _logger;  
  
    public LogWeatherService(
	    [FromKeyedServices("CachedWeatherService")] IWeatherService weatherService,  
        ILogger<LogWeatherService> logger)  
    {   
		_weatherService = weatherService;  
        _logger = logger;  
    }  
    
    public async Task<WeatherForecast?> GetWeatherForecastAsync(double latitude, double longitude)  
    {        
	    _logger.LogInformation("Getting weather forecast for {Latitude}, {Longitude}", latitude, longitude);  
  
        var response = await _weatherService.GetWeatherForecastAsync(latitude, longitude);  
  
        _logger.LogInformation("Got weather forecast for {Latitude}, {Longitude}", latitude, longitude);  
  
        return response;  
    }}
```

**Explicación del Código:**

1. La clase `LogWeatherService` implementa la interfaz `IWeatherService`. Este servicio se utiliza para registrar actividades (logging) relacionadas con la obtención de pronósticos del clima, además de invocar al servicio base para obtener los datos.
2. **Constructor del Servicio de Registro (Logging):**
    - El constructor de la clase `LogWeatherService` toma dos parámetros:
        - `[FromKeyedServices("CachedWeatherService")] IWeatherService weatherService`: Esto indica que estamos inyectando la dependencia `IWeatherService` con la clave "CachedWeatherService", lo que nos permite resolver específicamente el servicio que tiene la capa de caché.
        - `ILogger<LogWeatherService> logger`: Se inyecta una instancia del registro de actividades (logger) que se utilizará para registrar mensajes relacionados con la obtención de pronósticos del clima.
3. **Método `GetWeatherForecastAsync` con Registro (Logging):**
    - En este método, se registra un mensaje informativo utilizando el logger (`_logger.LogInformation`). Este mensaje indica que se está obteniendo el pronóstico del clima y muestra las coordenadas de latitud y longitud utilizadas.
    - Luego, se llama al servicio base `_weatherService` para obtener el pronóstico del clima.
    - Después de obtener la respuesta, se registra otro mensaje informativo que indica la obtención exitosa del pronóstico del clima.
    - Finalmente, se devuelve la respuesta obtenida del servicio base.

Este código demuestra cómo el patrón decorador se utiliza para agregar una capa de registro de actividades a un servicio base, lo que permite rastrear y registrar información relevante sobre las operaciones realizadas sin modificar la lógica fundamental del servicio original.

#### Program.cs

```csharp
using DecoratorPattern.Interfaces;  
using DecoratorPattern.Services;  
  
var builder = WebApplication.CreateBuilder(args);  
  
builder.Services.AddAuthorization();  
builder.Services.AddEndpointsApiExplorer();  
builder.Services.AddSwaggerGen();  
builder.Services.AddMemoryCache();  
builder.Services.AddHttpClient();  
  
// Registrar servicios decorados y sus dependencias  
builder.Services.AddKeyedScoped<IWeatherService, WeatherService>("WeatherService");  
builder.Services.AddKeyedScoped<IWeatherService, CachedWeatherService>("CachedWeatherService");  
builder.Services.AddScoped<IWeatherService, LogWeatherService>();  

var app = builder.Build();  
  
// Configure the HTTP request pipeline.  
if (app.Environment.IsDevelopment())  
{  
    app.UseSwagger();  
    app.UseSwaggerUI();  
}  
  
app.UseHttpsRedirection();  
app.UseAuthorization();  
  
// Definir una ruta para obtener el pronóstico del clima a través del servicio decorado  
app.MapGet("api/weather", (IWeatherService weatherService, double lon, double lat) =>  
    weatherService.GetWeatherForecastAsync(lat, lon));  
  
app.Run();
```

**Explicación del Código:**

1. **Registro de Servicios Decorados:** Se registran los servicios decorados que hemos implementado utilizando el patrón decorador. Esto se realiza utilizando `AddKeyedScoped` para registrar servicios decorados con sus dependencias correspondientes. Por ejemplo, se registra `WeatherService` con la clave "WeatherService", `CachedWeatherService` con la clave "CachedWeatherService", y `LogWeatherService` sin clave adicional. Ya que este es la última capa del decorador, esta es la que los consumidores soliciten por medio de Inyección de dependencias.
2. **Definición de Ruta:** Se define una ruta `/api/weather` que permite obtener el pronóstico del clima utilizando el servicio decorado `IWeatherService`. La ruta especifica los parámetros `lon` y `lat` para la longitud y latitud, respectivamente.

Para concluir, llegó el momento de probar nuestra API utilizando el endpoint que definimos previamente (`https://localhost:7192/api/weather?lon={longitud}&lat={latitud}`). Cuando realizamos una solicitud a esta dirección, obtendremos una respuesta similar a la siguiente:

```json
{
    "latitude": 0,
    "longitude": -0,
    "generationtime_ms": 0.02300739288330078,
    "utc_offset_seconds": 0,
    "timezone": "GMT",
    "timezone_abbreviation": "GMT",
    "elevation": 1461,
    "hourly_units": {
        "time": "iso8601",
        "temperature_2m": "°C"
    },
    "hourly": {
        "time": ["2023-10-03T00:00"],
        "temperature_2m": [30.7]
    }
}
```

Lo más relevante aquí es observar lo que sucede en la consola. Al realizar la primera llamada al endpoint, obtendremos un resultado similar al siguiente:

![Captura de Pantalla 1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/29g5lxro97xx88odc8bc.png)

El flujo de ejecución es el siguiente:
- El servicio "LogWeather" registra en la consola que se va a consultar el pronóstico del clima.
- "LogWeather" llama al servicio "CachedWeather".
- "CachedWeather" verifica que no haya datos en la caché y, al no encontrar ninguno, llama al servicio "Weather".
- El servicio original "Weather" consume la API externa para obtener los datos del pronóstico del clima.

Sin embargo, si realizamos una segunda llamada al mismo endpoint, notaremos que algo ha cambiado, ya que ahora existe información en la caché:

![Captura de Pantalla 2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/piowvrvpyjlwt038xq5g.png)

En este caso, el flujo es diferente:
- El servicio "LogWeather" registra en la consola que se va a consultar el pronóstico del clima nuevamente.
- "LogWeather" llama al servicio "CachedWeather".
- "CachedWeather" verifica que los datos ya se encuentren en caché, por lo que evita llamar al servicio "Weather" nuevamente.

Este comportamiento demuestra cómo los decoradores enriquecen la funcionalidad del servicio original "Weather" al agregar la capa de caché y registro de actividades, mejorando así el rendimiento y la trazabilidad de nuestras solicitudes.

# Conclusión

En resumen, hemos explorado el "Decorator Pattern" y su aplicación en el desarrollo de Web APIs con ASP.NET Core. A lo largo de este artículo, hemos visto cómo esta técnica nos permite extender la funcionalidad de nuestros servicios de una manera elegante y modular, sin necesidad de realizar cambios drásticos en el código original. Hemos observado ejemplos prácticos de cómo decorar nuestros servicios con características como la caché y el registro de actividades, lo que ha demostrado ser beneficioso para mejorar el rendimiento y la trazabilidad de nuestras aplicaciones.

Esperamos que esta introducción al Patrón Decorador te haya inspirado a explorar más a fondo esta técnica y a considerar su aplicación en tus propios proyectos. Con su capacidad para adaptarse a las necesidades cambiantes y mejorar la mantenibilidad del código, el Patrón Decorador se convierte en una herramienta valiosa en el kit de herramientas de cualquier desarrollador de aplicaciones web modernas. ¡Así que sigue experimentando y enriqueciendo tus aplicaciones con esta técnica poderosa!

Mejorando Rendimiento y Escalabilidad: Guía de Caché Distribuido en ASP.NET Core
# Introducción

En el ámbito dinámico de las aplicaciones web modernas, la optimización del rendimiento y la escalabilidad son esenciales. El uso estratégico de la caché es una herramienta fundamental para mejorar la velocidad y la capacidad de escalar, especialmente en entornos de múltiples servidores o en la nube.

ASP.NET Core ofrece una solución potente: el caché distribuido. Esta herramienta permite compartir la caché entre varios servidores, manteniéndola como un servicio externo accesible para todas las instancias de la aplicación. Esta capacidad distribuida garantiza la coherencia de los datos, incluso en entornos con múltiples nodos de servidor.

En este artículo, exploraremos cómo implementar y utilizar el caché distribuido en ASP.NET Core. Desde la configuración inicial hasta la integración con diferentes proveedores de caché, pasando por ejemplos de código prácticos, descubriremos cómo aprovechar al máximo esta potente funcionalidad para mejorar el rendimiento y la escalabilidad de tus aplicaciones.

## ¿Qué es la Caché Distribuida?

La caché distribuida es un recurso compartido entre varios servidores, mantenido por lo general como un servicio externo a los servidores que acceden a él. Esta estrategia de almacenamiento puede marcar la diferencia en la eficiencia de nuestras aplicaciones, garantizando la coherencia y accesibilidad de los datos entre múltiples servidores y sobreviviendo a reinicios o actualizaciones de servidores.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/m9ctqxdzxzfv6w42iwa3.png)

### Ventajas de la Caché Distribuida

- **Coherencia de Datos:** La información se mantiene consistente entre distintos servidores.
- **Resistente a reinicios:** Sobrevive a reinicios y deployments de servidores.
- **Uso eficiente de memoria:** No depende de la memoria local de cada servidor.
- **Escalabilidad**: Cuando una aplicación utiliza caché y esta tiene escalado horizontal, es necesario tener una caché distribuida para que todas las instancias accedan a la misma información.

### Cuándo Utilizar Caché Distribuida

La implementación de una caché distribuida puede ser beneficiosa en varios escenarios, especialmente cuando se requiere mejorar el rendimiento y la eficiencia de las aplicaciones. Algunos casos comunes donde el uso de caché es altamente recomendado incluyen:

- Acceso Frecuente a Datos Estáticos o Poco Cambiantes
- Acceso a información costosa de sistemas externos o bases de datos
- Escalabilidad

### Implementaciones de Caché Distribuida en ASP.NET Core

#### Redis

Redis, un almacén de datos en memoria, es una opción robusta para una caché distribuida. En este artículo, aprenderemos cómo configurar y utilizar Redis en conjunto con ASP.NET Core para mejorar el rendimiento de nuestras aplicaciones.
#### SQL Server

Utilizar SQL Server como caché distribuida puede ser una opción sólida. Sin embargo, es esencial entender sus implicaciones y cómo implementarlo correctamente para evitar impactos negativos en el rendimiento.

## Proyecto DistributedCacheExample (.NET 8)

El código fuente de este ejemplo se encuentra disponible en [DevToPosts/DistributedCacheExample · isaacOjeda/DevToPosts](https://github.com/isaacOjeda/DevToPosts/tree/main/DistributedCacheExample).

Para iniciar, creamos un proyecto web vacío (o API) y agregamos las siguientes dependencias:

```xml
<PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="8.0.0" />
<PackageReference Include="Microsoft.Extensions.Caching.StackExchangeRedis" Version="8.0.0" />
<PackageReference Include="Microsoft.VisualStudio.Azure.Containers.Tools.Targets" Version="1.19.5" />
<PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
```

La interfaz `IDistributedCache` ofrece métodos para escribir y leer caché, lo que normalmente involucra guardar objetos o cualquier tipo de dato en caché.

El siguiente servicio facilita la lectura y escritura en la caché:

```csharp
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;
 
namespace DistributedCacheExample;
 
public class DistributedCacheService(IDistributedCache distributedCache)
{
    public async Task<T?> GetCachedItem<T>(string key)
        where T : class
    {
        var dataInBytes = await distributedCache.GetAsync(key);
 
        if (dataInBytes is null)
        {
            return null;
        }
 
        var rawJson = System.Text.Encoding.UTF8.GetString(dataInBytes);
 
        return JsonSerializer.Deserialize<T>(rawJson);
    }
 
    public async Task SaveItem<T>(T item, string key, int expirationInMinutes)
    {
        var dataJson = JsonSerializer.Serialize(item);
        var dataInBytes = System.Text.Encoding.UTF8.GetBytes(dataJson);
 
        await distributedCache.SetAsync(key, dataInBytes, new DistributedCacheEntryOptions
        {
            AbsoluteExpiration = DateTimeOffset.UtcNow.AddMinutes(expirationInMinutes)
        });
    }
}
```

Esta clase maneja las operaciones de lectura y escritura en la caché, todos los datos los serializamos de JSON a bytes o viceversa según aplique. 

En Program.cs registramos las dependencias necesarias e incluimos un ejemplo de Endpoint para mostrar cómo podemos utilizar la caché.

En este ejemplo simulamos la consulta del clima desde un servicio externo. Deseamos evitar realizar esta consulta con frecuencia ya que los datos climáticos no cambian rápidamente. En el ejemplo, guardamos en caché los datos por 5 minutos, pero esto puede variar según el caso de uso.

```csharp
using DistributedCacheExample;
 
var builder = WebApplication.CreateBuilder(args);

// Registro de servicios
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

builder.Services.AddScoped<DistributedCacheService>();
// 
builder.Services.AddDistributedMemoryCache();
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("RedisCache");
    options.InstanceName = "DistributedCacheExample";
});
 
var app = builder.Build();
 
// Configuración del pipeline de solicitud HTTP
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
 
 
app.MapGet("api/weather", async (DistributedCacheService cache) =>
{
    string[] summaries = {
        "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
    };
 
    var data = await cache.GetCachedItem<IEnumerable<WeatherForecast>>("GetWeatherForecast");
 
    if (data is null)
    {
        data = Enumerable.Range(1, 5).Select(index => new WeatherForecast
        {
            Date = DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
            TemperatureC = Random.Shared.Next(-20, 55),
            Summary = summaries[Random.Shared.Next(summaries.Length)]
        }).ToArray();
 
        await cache.SaveItem(data, "GetWeatherForecast", expirationInMinutes: 5);
    }
 
    return data;
});
 
app.Run();
 
public class WeatherForecast
{
    public DateOnly Date { get; set; }
    public int TemperatureC { get; set; }
    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
    public string? Summary { get; set; }
}
```

La función `AddDistributedMemoryCache` registra las dependencias como `IDistributedCache`, pero se limita a proporcionar soporte para caché en memoria dentro de la misma instancia. A pesar de no ser un verdadero sistema de caché distribuido, resulta útil cuando estamos en una etapa de desarrollo y no contamos con una infraestructura disponible.

Por otro lado, al añadir `AddStackExchangeRedisCache`, estamos incorporando la implementación de `IDistributedCache`, esta vez con Redis. Sin embargo, es fundamental tener la infraestructura necesaria para poder probar y utilizar Redis en un entorno de desarrollo. Este enfoque nos permite aprovechar las capacidades de Redis como sistema de caché distribuido y representa una opción sólida cuando ya buscamos tener algo en producción.

Además agregamos un endpoint que simula la consulta al clima desde un servicio externo. Si los datos climáticos no están en la caché, se generan datos aleatorios para 5 días y se almacenan en la caché con una expiración de 5 minutos. Si los datos están en la caché, se devuelven desde la caché para evitar consultas frecuentes al servicio externo.

También se agregó la cadena de conexión para la comunicación con Redis en el archivo de configuración:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "RedisCache": "redis"
  }
}
```
#### Infraestructura

Antes de ejecutar la aplicación, es necesario tener Redis en funcionamiento.

En este ejemplo, opté por utilizar Docker y docker-compose, pero tienes libertad para elegir el método que prefieras.
##### Archivo Docker

El siguiente archivo Docker fue generado por Visual Studio al añadir soporte para docker-compose. Puedes seguir este ejemplo o escribirlo manualmente (o simplemente ignorarlo, no es imprescindible):

```d
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443
 
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["DistributedCacheExample/DistributedCacheExample.csproj", "DistributedCacheExample/"]
RUN dotnet restore "DistributedCacheExample/DistributedCacheExample.csproj"
COPY . .
WORKDIR "/src/DistributedCacheExample"
RUN dotnet build "DistributedCacheExample.csproj" -c Release -o /app/build
 
FROM build AS publish
RUN dotnet publish "DistributedCacheExample.csproj" -c Release -o /app/publish /p:UseAppHost=false
 
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "DistributedCacheExample.dll"]
```

##### Archivo Docker Compose

Para ejecutar mi API y Redis juntos, usamos este archivo `docker-compose` para orquestar la infraestructura con su propia red de forma sencilla:

```d
version: '3.4'
 
services:
  distributedcacheexample:
    image: ${DOCKER_REGISTRY-}distributedcacheexample
    build:
      context: .
      dockerfile: DistributedCacheExample/Dockerfile
    ports:
      - 7198:443
      - 5106:80      
    networks:
      - balusoft
 
  redis:
    image: redis/redis-stack:latest
    container_name: redis-stack-server
    restart: always
    ports:
      - 6379:6379
      - 8001:8001
    networks:
      - balusoft
    volumes:
      - redis:/redis/data
 
networks:
  balusoft:
    name: balusoft-network
 
volumes:
  redis:
    driver: local
```

Con esto será suficiente para correr la aplicación.
### Probando el Caché

Si estás usando Visual Studio, puedes ejecutar el proyecto a través de Docker Compose:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/d8cmkboja1hsa8tkc0r2.png)

Pero sin problemas, se puede correr también desde VS Code y la Terminal utilizando el comando `docker compose up`:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wgd9unbdrxivqnnl3bbc.png)

Independientemente de cómo hayas ejecutado `docker-compose`, en Docker Desktop podrás ver los servicios activos junto con los puertos asignados:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ujl23tli43cnnim4cm33.png)

En esta configuración, estamos usando `redis-stack-server`, un conjunto de tecnologías aparte del caché que ofrece Redis.

Si accedes al puerto `localhost:8001`, se abrirá un explorador de Redis:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gcy62t519spztcugcc9u.png)

Ahora, al dirigirte a Swagger en la API (por ejemplo, en mi caso, se encuentra en `http://localhost:5106/swagger/`), verás el endpoint de ejemplo que creamos:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2vxfxvvgx9a3w33u053a.png)

Al ejecutar este endpoint, observarás que se crea una entrada de caché en Redis:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/q1m2gckgq7m0x6l9lorx.png)


# Conclusión

El caché distribuido en ASP.NET Core se revela como una herramienta imprescindible para optimizar el rendimiento y la escalabilidad de tus aplicaciones. La capacidad de compartir datos en caché entre múltiples servidores, garantizando coherencia y supervivencia a reinicios o despliegues, abre un abanico de posibilidades para mejorar la eficiencia y la experiencia del usuario.

Al aprender a configurar y utilizar el caché distribuido, descubrimos cómo aprovechar su flexibilidad para adaptarse a diferentes escenarios y proveedores. Desde Redis y SQL Server hasta soluciones personalizadas como NCache, las opciones son amplias y versátiles, permitiendo adaptar la implementación a las necesidades específicas de cada aplicación.

La capacidad de almacenar datos comunes en caché y minimizar las consultas repetitivas a bases de datos o servicios externos se traduce en mejoras significativas de rendimiento. Además, el control sobre la caducidad de los datos nos otorga la posibilidad de equilibrar la frescura de la información con la eficiencia del sistema.

En resumen, el uso estratégico del caché distribuido representa un pilar fundamental en la mejora del rendimiento y la escalabilidad de las aplicaciones ASP.NET Core. Dominar esta herramienta no solo potencia la velocidad y la eficiencia, sino que también contribuye a una experiencia de usuario más fluida y satisfactoria.



# Referencias
- [Overview of caching in ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/overview?view=aspnetcore-8.0)
- [mgravell/DistributedCacheDemo: basic example of helper APIs for IDistributedCache (github.com)](https://github.com/mgravell/DistributedCacheDemo)

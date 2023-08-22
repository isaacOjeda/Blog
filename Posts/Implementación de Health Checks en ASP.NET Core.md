Los *health checks* son un elemento esencial en el desarrollo de aplicaciones, ya que permiten garantizar que todos los componentes y servicios estén funcionando de manera adecuada. En ASP.NET Core, la implementación de health checks es una tarea sencilla gracias a los paquetes y configuraciones específicas disponibles.

En esta guía, exploraremos paso a paso cómo implementar health checks en una aplicación ASP.NET Core utilizando el paquete `AspNetCore.HealthChecks.*`.

### Configuración de los Paquetes

Lo primero que debemos hacer es agregar las dependencias necesarias en el archivo `csproj`:

```xml
<PackageReference Include="AspNetCore.HealthChecks.ApplicationStatus" Version="7.0.0" />  
<PackageReference Include="AspNetCore.HealthChecks.SendGrid" Version="7.0.0" />  
<PackageReference Include="AspNetCore.HealthChecks.SqlServer" Version="7.0.0" />  
<PackageReference Include="AspNetCore.HealthChecks.UI" Version="7.0.2" />  
<PackageReference Include="AspNetCore.HealthChecks.UI.Client" Version="7.1.0" />  
<PackageReference Include="AspNetCore.HealthChecks.UI.InMemory.Storage" Version="7.0.0" />
```

### Configuración de los Health Checks

En el archivo de inicio de tu aplicación (`Program.cs`), procedemos a configurar los health checks:

```csharp
using HealthChecks.ApplicationStatus.DependencyInjection;  
using HealthChecks.UI.Client;  
using Microsoft.AspNetCore.Diagnostics.HealthChecks;  
using Microsoft.Extensions.Diagnostics.HealthChecks;  
  
var builder = WebApplication.CreateBuilder(args);  
  
// Configuración de health checks  
builder.Services  
    .AddHealthChecks()  
    .AddApplicationStatus(name: "api_status", tags: new[] { "api" })  
    .AddSqlServer(       
        connectionString: builder.Configuration.GetConnectionString("Default")!,  
        name: "sql",  
        failureStatus: HealthStatus.Degraded,  
        tags: new[] { "db", "sql", "sqlserver" })  
    .AddSendGrid(apiKey: builder.Configuration["SendGrid:Key"]!, tags: new[] { "email", "sendgrid" })  
    .AddCheck<ServerHealthCheck>("server_health_check", tags: new []{"custom", "api"});  
```

Esta configuración establece varios health checks:
- **Application Status**: Comprueba si el servicio está en funcionamiento, mediante la detección de un posible cierre de la aplicación.
- **SQL Server**: Verifica el estado de la base de datos a través de un simple `SELECT 1;`, sin generar carga de CPU.
- **SendGrid**: Valida la configuración y disponibilidad del servicio de correos SendGrid.
- **Custom**: Agregamos uno personalizado para implementar la lógica pertinente y determinar el estado del servicio.

Además de estas verificaciones, el repositorio [AspNetCore.Diagnostics.HealthChecks](https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks) ofrece una amplia gama de health checks predefinidos para componentes clave en aplicaciones ASP.NET Core, como otras bases de datos o servicios como Rabbitmq.

### ServerHealthCheck - Health Check Personalizado

A continuación, se muestra un ejemplo de implementación para el health check personalizado `ServerHealthCheck`:

```csharp
using Microsoft.Extensions.Diagnostics.HealthChecks;  
  
namespace Web;  
  
public class ServerHealthCheck : IHealthCheck  
{  
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context,  
        CancellationToken cancellationToken = default)  
    {  
        var isServerHealthy = CheckServerStatus();  
  
        return Task.FromResult(isServerHealthy  
            ? HealthCheckResult.Healthy("El servidor está en funcionamiento.")  
            : HealthCheckResult.Unhealthy("El servidor no está respondiendo."));  
    }  
  
    private bool CheckServerStatus()  
    {  
        // Implementa aquí la lógica para verificar el estado del servidor.  
        return true; // Simulamos que el servidor está en funcionamiento  
    }  
}
```

### Configuración de la Interfaz de Usuario de Health Checks

Para añadir la interfaz de usuario de health checks, debemos incluir la siguiente configuración:

```csharp
builder.Services  
    .AddHealthChecksUI()  
    .AddInMemoryStorage();
```

El repositorio [AspNetCore.Diagnostics.HealthChecks](https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks) ofrece opciones de persistencia flexibles para la interfaz de usuario de health checks. Puedes elegir entre varias opciones de almacenamiento, como InMemory Storage, SQL Server Storage, Redis Storage y Elasticsearch Storage, para almacenar y acceder a los resultados de las verificaciones de salud.

### Endpoints

En el mismo archivo `Program.cs`, configuramos las rutas y los endpoints con los health checks:

```csharp
var app = builder.Build();  
  
app.MapGet("/", () => "¡Hola, mundo!");  
app.MapHealthChecks("/healthz", new HealthCheckOptions()  
{  
    Predicate = _ => true,  
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse  
});  
app.MapHealthChecksUI();  
  
app.Run();
```

### appsettings.json

Configura el archivo `appsettings.json` con las cadenas de conexión y claves necesarias:

```json
{  
  "Logging": {  
    "LogLevel": {  
      "Default": "Information",  
      "Microsoft.AspNetCore": "Warning"  
    }  
  },  
  "AllowedHosts": "*",  
  "ConnectionStrings": {  
    "Default" : "<Tu cadena de conexión>"  
  },  
  "SendGrid": {  
    "Key": "<Clave de API de Sendgrid>"  
  },  
  "HealthChecksUI": {  
    "HealthChecks": [  
      {  
        "Name": "API Demo",  
        "Uri": "http://localhost:<puerto>/healthz"  
      }  
    ],  
    "Webhooks": [],  
    "EvaluationTimeInSeconds": 10,  
    "MinimumSecondsBetweenFailureNotifications": 60  
  }  
}
```

### Casos de Uso y Ventajas de los Health Checks

Los *health checks* tienen múltiples aplicaciones y beneficios significativos:

- **Monitoreo Proactivo**: Identifican problemas antes de que impacten a los usuarios, permitiendo acciones preventivas.
- **Ajuste de Escala**: Los sistemas de orquestación de contenedores pueden usar health checks para determinar cuándo escalar instancias.
- **Notificaciones de Estado**: Configuran alertas que notifican sobre fallos en componentes o servicios.
- **Reparación Automática**: En entornos de microservicios, los health checks pueden activar acciones automáticas de recuperación.

### Ventajas de los Health Checks

- **Detección Temprana de Problemas**: Identifican problemas en etapas tempranas, mejorando la experiencia del usuario.
- **Toma de Decisiones Informada**: Facilitan decisiones informadas sobre escalado y gestión de recursos.
- **Mayor Confiabilidad**: Mejoran la disponibilidad y la confiabilidad al abordar problemas de manera rápida.
- **Ahorro de Tiempo y Recursos**: Reducen el tiempo necesario para solucionar problemas inesperados.

En conclusión, implementar health checks en una aplicación ASP.NET Core brinda visibilidad sobre el estado de servicios y componentes, fortaleciendo la confiabilidad de la aplicación y asegurando una experiencia de usuario fluida.

## Probando los Health Checks

Al ejecutar la aplicación, puedes acceder a la ruta `/healthz` para ver el estado de los health checks configurados:

```json
{
  "status": "Healthy",
  "totalDuration": "00:00:00.1591922",
  "entries": {
    "api_status": {
      "data": {},
      "duration": "00:00:00.0000854",
      "status": "Healthy",
      "tags": [
        "api"
      ]
    },
    "sql": {
      "data": {},
      "duration": "00:00:00.0048486",
      "status": "Healthy",
      "tags": [
        "db",
        "sql",
        "sqlserver"
      ]
    },
    "sendgrid": {
      "data": {},
      "duration": "00:00:00.1590220",
      "status": "Healthy",
      "tags": [
        "email",
        "sendgrid"
      ]
    },
    "server_health_check": {
      "data": {},
      "description": "El servidor está en funcionamiento.",
      "duration": "00:00:00.0000300",
      "status": "Healthy",
      "tags": [
        "custom",
        "api"
      ]
    }
  }
}
```

La interfaz de usuario de health checks, accesible en `/healthchecks-ui`, muestra una visualización gráfica de estos resultados.

## Conclusión

La implementación de health checks es una práctica esencial para asegurar la disponibilidad y robustez de las aplicaciones. Gracias al paquete `AspNetCore.HealthChecks` en ASP.NET Core, la configuración y monitorización del estado de los servicios y componentes es simple y eficaz. Al invertir en health checks, estarás mejor preparado para enfrentar problemas y garantizar una experiencia de usuario sin problemas.

## Referencias

- [Health checks en ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-7.0)
- [Xabaril/AspNetCore.Diagnostics.HealthChecks: Enterprise HealthChecks for ASP.NET Core Diagnostics Package (GitHub)](https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks)
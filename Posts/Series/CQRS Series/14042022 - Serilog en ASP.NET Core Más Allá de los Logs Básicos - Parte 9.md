## ¿Qué es el Logging?

El logging, o registro, es la práctica de mantener bitácoras que registran el comportamiento de un sistema. Estos registros se almacenan en algún lugar o se pueden visualizar a través de una interfaz. A menudo, el logging no recibe la atención que merece, pero cuando los problemas comienzan a surgir en un sistema en producción, los logs se convierten en una herramienta invaluable. Tener logs bien estructurados y ricos en información es esencial para resolver problemas de manera efectiva.

El logging desempeña diversos roles, pero su función principal es ayudar a investigar las causas subyacentes de los problemas, analizar errores, evaluar el rendimiento y, en algunos casos, auditar el sistema. ASP.NET Core ya incluye un sistema de logging que, de forma predeterminada, envía logs a la consola y a los registros de eventos de Windows. Sin embargo, para ir más allá de los simples logs en la consola, se requiere una configuración y esfuerzo adicionales.

## ¿Qué es Serilog?

Serilog es una herramienta poderosa que facilita el registro de datos para diagnóstico en aplicaciones ASP.NET Core. Serilog se destaca por su facilidad de configuración y una API limpia que se adapta sin problemas a diferentes versiones de .NET. A diferencia de otras bibliotecas de logging, Serilog no convierte automáticamente los parámetros pasados en los logs en cadenas de texto. En cambio, conserva la estructura de los objetos, lo que facilita la visualización y búsqueda de información valiosa.

En lugar de realizar logs de texto, Serilog realiza logs de datos. Para aprovechar al máximo Serilog, es fundamental utilizar "message templates", lo que significa no concatenar cadenas de texto al hacer logs. Estos message templates permiten crear logs estructurados y efectivos.

## Ejemplo de Message Template

```csharp
var input = new { Latitude = 25, Longitude = 134 };
var time = 34;

log.Information("Processed {@SensorInput} in {TimeMS:000} ms.", input, time);
```

El resultado en la consola se verá así:

```bash
09:14:22 [Information] Processed { Latitude: 25, Longitude: 134 } in 034 ms. 
```

El operador `@` en "@SensorInput" indica a Serilog que debe mantener la estructura del objeto pasado como argumento. Si omitimos este operador, Serilog llamará al método `ToString()` para mostrar el valor del objeto. Este ejemplo presenta dos propiedades, "SensorInput" y "TimeMS", que se guardarán en el evento de log, no solo como texto en la consola.

Las propiedades capturadas se pueden visualizar en formato JSON de la siguiente manera:

```json
{ "SensorInput": { "Latitude": 25, "Longitude": 134 }, "TimeMS": 34 }
```

## Sinks: Almacenamiento de Logs

Un aspecto fundamental de Serilog son los "sinks". Los sinks son destinos donde le indicamos a Serilog que almacene los logs. Pueden ser archivos en formato JSON, registros en una base de datos, eventos en Application Insights y muchas otras opciones. El uso de "message templates" es esencial, ya que, si bien es posible que en la consola no se muestre toda la información, al almacenarla en un lugar como Elastic Search, podemos realizar búsquedas por usuario, tipo de operación y otros criterios.

Serilog también se presenta como una alternativa a Audit.NET, que se explica en otro [post](https://dev.to/isaacojeda/parte-7-aspnet-creando-un-sistema-auditable-31nf). La idea es aprender sobre las alternativas disponibles al usar ASP.NET.

Almacenar los logs en Application Insights resulta muy útil, ya que esta herramienta permite agrupar los logs según las solicitudes y operaciones realizadas en su Web API. Realizar un seguimiento de esta información es fundamental al diagnosticar problemas de rendimiento o fallos en la aplicación.

## Ejemplo de Log en Application Insights

![Descripción de la imagen](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kv8ys9m81t98etrb4kdw.png)

En la imagen, se ha resaltado el message template en rojo, pero se ha indicado a Serilog que agregue más propiedades al "Log Event". Aunque estas propiedades no sean visibles en la consola, estarán disponibles en los sinks.

## Configuración de Serilog en MediatRExample

Esta publicación es la parte 9 de una serie que se centra en ASP.NET Core y temas relevantes para todos los desarrolladores. Para comenzar, debemos instalar Serilog para ASP.NET Core en su aplicación Web API:

```bash
dotnet add package Serilog.AspNetCore
```

La configuración es sencilla, ya que se integra perfectamente con `ILogger<T>`, que es una herramienta que probablemente haya estado utilizando todo el tiempo. Solo es necesario realizar una configuración en el archivo "Program.cs":

```csharp
// ... código omitido
var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog();

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .CreateLogger();

// ... código omitido

try
{
    Log.Information("Iniciando Web API");

    await SeedProducts();

    Log.Information("Corriendo en:");
    Log.Information("https://localhost:7113");
    Log.Information("http://localhost:5144");

    app.Run();

}
catch (Exception ex)
{
    Log.Fatal(ex, "Host terminated unexpectedly");

    return;
}
finally
{
    Log.CloseAndFlush();
}

// ... código omitido
```

En este punto, la sección "Log" en "appsettings.json" ya no se necesita y se puede eliminar, ya que no se utilizará. El message template que se utiliza en el "AuditLogBehavior" es el siguiente:

```csharp
_logger.LogInformation("{RequetsName}: {@User} with request {@Request}", typeof(TRequest).Name, _currentUserService.User.Id, request);
```

Cuando ejecute la aplicación, verá logs con la siguiente estructura:

![Descripción de la imagen](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/b14vjpdzegkdmmf5s8ej.png)

Si desea almacenar los logs en un archivo en el mismo servidor web (lo que facilita la consulta en tiempo real, especialmente si usa Azure), simplemente realice la siguiente configuración en "Program.cs":

```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.File("log.txt", rollingInterval: RollingInterval.Day)
    .CreateLogger();
```

Con esta configuración, la aplicación creará archivos de texto con el nombre "log{yyyyMMdd}.txt" en la raíz del proyecto. Los archivos contendrán registros detallados.

Este sink de archivos resulta útil, pero lo es aún más si utiliza sinks como Elastic Search o Application Insights. Estas opciones permiten la búsqueda y el seguimiento de las solicitudes de su aplicación en producción.
## Conclusión

El logging y el tracing son temas cruciales para diagnosticar problemas en una aplicación. Aunque en este artículo se ha explorado cómo crear logs y enviarlos a sinks de archivos y la consola, es importante investigar y elegir las opciones que mejor se adapten a las necesidades de su proyecto. Como siempre, no dude en ponerse en contacto conmigo en [Twitter](https://twitter.com/balunatic) si tiene alguna pregunta.

## Referencias

- [serilog/serilog-aspnetcore - Github](https://github.com/serilog/serilog-aspnetcore#serilogaspnetcore---)
- [serilog/serilog - Github](https://github.com/serilog/serilog/wiki/Getting-Started)
- [Guía definitiva para el logging en .NET](https://www.loggly.com/ultimate-guide/net-logging-basics/)
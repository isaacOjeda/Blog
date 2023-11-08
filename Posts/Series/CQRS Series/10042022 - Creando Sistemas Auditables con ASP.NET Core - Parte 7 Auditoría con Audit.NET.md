# Introducción
En nuestra serie de publicaciones sobre ASP.NET Core, continuamos explorando diversos aspectos del desarrollo, y en esta ocasión, nos adentraremos en la creación de sistemas auditables.

Si deseas revisar el código de este artículo, lo tienes disponible [aquí](https://github.com/isaacOjeda/AspNetCoreMediatRExample/tree/parte-7).

# Sistemas Auditables
En el mundo del desarrollo de software, especialmente en entornos corporativos y bajo ciertos estándares como **ISO 27001**, es esencial poder registrar las acciones realizadas en un sistema. En muchos casos, se requiere mantener un historial de al menos **90 días** de todas las operaciones efectuadas en el sistema para fines de auditoría. No obstante, el período de 90 días puede resultar excesivo en situaciones con alta concurrencia de usuarios o en aplicaciones críticas con una elevada carga de solicitudes.

Por lo tanto, es fundamental abordar este requisito de manera inteligente. ¿Por qué debemos auditar? Porque en ocasiones suceden eventos desafortunados y es necesario identificar quién ejecutó operaciones críticas en el sistema.

En este artículo, exploraremos dos enfoques para implementar la auditoría en nuestro sistema, y cabe destacar que a menudo combino ambos enfoques: **AuditableEntity** y la biblioteca **Audit.NET**.
# Entidades Auditables
Un Entity Auditable se refiere a la capacidad de rastrear quién creó y editó todas las entidades de nuestra base de datos. Esta funcionalidad no debería agregar una carga significativa al crear comandos, ya que es una tarea repetitiva. Por lo tanto, configuraremos nuestro `DbContext` para que se encargue de esta tarea por nosotros.

Para hacer que nuestras entidades sean auditables, crearemos una entidad base que todas las demás entidades heredarán:
```csharp
namespace MediatrExample.ApplicationCore.Domain;
public class BaseEntity
{
    public DateTime? CreatedAt { get; set; }
    public string? CreatedBy { get; set; }
    public DateTime? LastModifiedByAt { get; set; }
    public string? LastModifiedBy { get; set; }
}
```
Luego, actualizaremos nuestra única entidad de ejemplo de la siguiente manera:
```csharp
public class Product : BaseEntity // <----
{
   // Código omitido
}
```
Este cambio en la entidad `Product` agrega cuatro nuevas propiedades, las cuales se actualizarán posteriormente durante las migraciones y en la base de datos.

En una publicación anterior, ya implementamos la autenticación y autorización de usuarios. Para llevar a cabo la auditoría, necesitamos identificar a los usuarios de manera individual.

Para lograr esto, actualizaremos nuestro `DbContext` para que almacene automáticamente la información cada vez que guardemos el contexto de la base de datos. El siguiente fragmento de código muestra cómo hacerlo:
```csharp
using MediatrExample.ApplicationCore.Common.Interfaces;
using MediatrExample.ApplicationCore.Domain;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

namespace MediatrExample.ApplicationCore.Infrastructure.Persistence;

public class MyAppDbContext : IdentityDbContext<IdentityUser>
{
    private readonly CurrentUser _user;

    public MyAppDbContext(
        DbContextOptions<MyAppDbContext> options,
        ICurrentUserService currentUserService) : base(options)
    {
        _user = currentUserService.User;
    }

    public DbSet<Product> Products => Set<Product>();

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        foreach (var entry in ChangeTracker.Entries<BaseEntity>())
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedBy = _user.Id;
                    entry.Entity.CreatedAt = DateTime.UtcNow;
                    break;

                case EntityState.Modified:
                    entry.Entity.LastModifiedBy = _user.Id;
                    entry.Entity.LastModifiedByAt = DateTime.UtcNow;
                    break;
            }
        }

        return await base.SaveChangesAsync(cancellationToken);
    }
}
```
En este código, destacamos dos aspectos importantes:
- Hemos inyectado `ICurrentUserService` para acceder al usuario actual que realiza la operación.
- Hemos sobrescrito `SaveChangesAsync` para guardar automáticamente la información del usuario que realiza la operación.

Dentro de `SaveChangesAsync` ocurre algo especial:
- El método `ChangeTracker.Entries<BaseEntity>` nos proporciona todos los registros que se han creado o modificado en el `DbContext`. Es importante destacar que todas las entidades deben heredar de `BaseEntity` para que esto funcione. Dependiendo de la operación realizada (crear o modificar), se actualizan automáticamente los campos correspondientes de la entidad modificada.

Este proceso automatizado es realmente beneficioso, ya que no requerirá una intervención manual constante.

Para poder continuar y crear las migraciones necesarias, debemos actualizar la implementación de `ICurrentUserService`, ya que podría causar problemas si no manejamos adecuadamente los posibles valores `null` al crear migraciones u otras operaciones que no implican una solicitud HTTP.

> Nota 👀: Si tienes alguna pregunta sobre el código, te animo a visitar el repositorio con el código de esta publicación.

```csharp
public CurrentUserService(IHttpContextAccessor httpContextAccessor)
{
    _httpContextAccessor = httpContextAccessor;

    // Es posible que la aplicación se esté inicializando.
    if (_httpContextAccessor is null || _httpContextAccessor.HttpContext is null)
    {
        User = new CurrentUser(Guid.Empty.ToString(), string.Empty, false);
        return;
    }

    // Existe una solicitud HTTP, pero el usuario no está autenticado.
    var httpContext = _httpContextAccessor.HttpContext;
    if (httpContext!.User!.Identity!.IsAuthenticated == false)
    {
        User = new CurrentUser(Guid.Empty.ToString(), string.Empty, false);
        return;
    }

    var id = httpContext.User.Claims
        .FirstOrDefault(q => q.Type == ClaimTypes.Sid)!
        .Value;

    var userName = httpContext.User!.Identity!.Name ?? "Unknown";

    User = new CurrentUser(id, userName, true);
}
```

Hemos agregado una nueva propiedad al registro `CurrentUser` para determinar si el usuario está autenticado o no. Esto se ha hecho para evitar problemas cuando el `DbContext` accede al usuario actual, ya que al inicializar el DbContext, como sucede en las migraciones en modo de desarrollo, el `CurrentUser` podría no existir. De esta manera, siempre se inicializa la propiedad User para evitar inconvenientes.

> Nota 👀: Cabe destacar que podríamos haber verificado si el usuario es nulo al usarlo en el `DbContext`. La elección de esta implementación dependerá de tus preferencias, pero lo importante es comprender el concepto de la implementación.

Con estas modificaciones, ya podemos crear migraciones desde el proyecto WebApi utilizando los siguientes comandos:

```bash
dotnet ef migrations add AddedBaseEntity -o Infrastructure/Persistence/Migrations -p ..\MediatrExample.ApplicationCore\
dotnet ef database update
```

Es importante destacar que la forma en la que realizamos la migración difiere de otros artículos. En el artículo 6, reestructuramos el proyecto para adoptar una arquitectura de Vertical Slice, lo que ha influido en el enfoque actual.

> Nota 👀: Si experimentas errores, la solución más sencilla suele ser eliminar el archivo de la base de datos SQLite y volver a ejecutar los comandos anteriores.

Una vez que se haya ejecutado la migración, podremos crear o editar productos (se ha añadido el comando para editar, que puedes consultar en el código fuente). Observarás cómo se guarda la información en la base de datos, como se muestra en la tabla de productos a continuación:

![Tabla Products](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fnulsa9qgfo0l54sguit.png)

De esta manera, cumplimos con el requisito de auditoría de manera sencilla. Cada creación o modificación quedará registrada en cada entidad.

> Nota 👀: El campo **CreatedBy** con el Guid vacío fue creado por el método Seed dentro de **Program.cs**.

Es importante tener en cuenta que esta no es una bitácora completa de cambios o registros auditables, sino el primer paso para facilitar el acceso a esta información. Esta implementación es útil cuando se necesita mostrar quién creó un registro o quién lo modificó, como suele ser necesario en un catálogo de clientes.

## Implementando Audit.NET
Audit.NET es una biblioteca que simplifica la implementación de requisitos de auditoría. Ofrece una variedad de extensiones y opciones para la persistencia de registros de auditoría.

> Nota 👀: Con Audit.NET, puedes integrarlo con Web API, MVC, Entity Framework, SignalR, entre otros. También proporciona opciones de persistencia en SQL Server, MySQL, Azure Storage Tables, Azure Storage Blobs, Elastic Search, y muchas otras.

A continuación, crearemos un nuevo decorador de MediatR para registrar las operaciones en un registro de auditoría. Utilizaremos **Azure Storage Accounts** y Blobs para la persistencia de registros, ya que esperamos manejar una gran cantidad de registros y no queremos que afecte al rendimiento o los costos de almacenamiento.

Es fundamental diseñar cómo guardamos los registros de auditoría, de modo que sea posible consultar la información, ya sea por tipo de operación o por el usuario que la realizó. Debemos tener en cuenta que, en última instancia, podríamos acumular **millones** de registros.

Otra lección importante que hemos aprendido es que no queremos guardar registros de auditoría de todas las operaciones (consultas y comandos) en un sistema. Recientemente, nos dimos cuenta de que este tipo de mecanismos puede afectar el rendimiento del sistema, ya que registramos SIEMPRE en la bitácora, ya sea una consulta o un comando. Algunas consultas son mucho más frecuentes que ciertos comandos, por lo que no es necesario registrar todo en la bitácora.

Para evitar este problema y solo registrar en los registros de auditoría las operaciones que nos interesa auditar (que deberían ser todos los comandos), crearemos un atributo para decorar los `IRequest` y permitiremos que el Behavior determine si es necesario registrar la operación en la bitácora antes de ejecutarla.

### Instalación de Audit.NET

Audit.NET proporciona un completo mecanismo de recolección de información, incluyendo datos como la duración de la operación y los cambios realizados. Es altamente flexible y útil. Para obtener más información sobre sus capacidades, puedes visitar su repositorio en [GitHub](https://github.com/thepirat000/Audit.NET).

El paquete de Azure Storage Blobs es esencial para almacenar los registros en blobs de una cuenta de almacenamiento de Azure. Para probar esto, es necesario tener instalado el emulador de Azure Storage, que suele estar incluido en Visual Studio. Si no lo tienes o no estás seguro de qué se trata, no dudes en preguntar y con gusto te proporcionaré más información sobre el tema.

Para instalar estos paquetes, debemos dirigirnos al proyecto **ApplicationCore** y ejecutar los siguientes comandos:

```bash
dotnet add package Audit.NET
dotnet add package Audit.NET.AzureStorageBlobs
```

### ApplicationCore -> Common -> Attributes -> AuditLogAttribute

En esta sección, hemos creado un atributo simple para determinar qué `IRequest` debe ser auditado. No requiere información adicional, ya que su único propósito es identificar los comandos que queremos guardar en la bitácora.

```csharp
namespace MediatrExample.ApplicationCore.Common.Attributes;

/// <summary>
/// Atributo para determinar que IRequest debe ser auditado
/// </summary>
[AttributeUsage(AttributeTargets.Class, Inherited = true)]
public class AuditLogAttribute : Attribute
{
}
```

### ApplicationCore -> Common -> Behaviours -> AuditLogsBehavior

Para llevar a cabo la auditoría, hemos creado un decorador (**Behavior**) de MediatR. Este decorador se encargará de registrar las operaciones en un registro de auditoría.

```csharp
using Audit.Core;
using MediatR;
using MediatrExample.ApplicationCore.Common.Attributes;
using MediatrExample.ApplicationCore.Common.Interfaces;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using System.Reflection;

namespace MediatrExample.ApplicationCore.Common.Behaviours;

public class AuditLogsBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
     where TRequest : IRequest<TResponse>
{
    private readonly ICurrentUserService _currentUserService;
    private readonly ILogger<AuditLogsBehavior<TRequest, TResponse>> _logger;
    private readonly IConfiguration _config;

    public AuditLogsBehavior(
        ICurrentUserService currentUserService,
        ILogger<AuditLogsBehavior<TRequest, TResponse>> logger,
        IConfiguration config)
    {
        _currentUserService = currentUserService;
        _logger = logger;
        _config = config;
    }

    public async Task<TResponse> Handle(TRequest request, CancellationToken cancellationToken, RequestHandlerDelegate<TResponse> next)
    {
        _logger.LogInformation("User {@User} with request {@Request}", _currentUserService.User, request);

        IAuditScope? scope = null;
        var auditLogAttributes = request.GetType().GetCustomAttributes<AuditLogAttribute>();
        if (auditLogAttributes.Any() && _config.GetValue<bool>("AuditLogs:Enabled"))
        {
            // El IRequest cuenta con el atributo [AuditLog] para ser auditado
            scope = AuditScope.Create(_ => _
                .EventType(typeof(TRequest).Name)
                .ExtraFields(new
                {
                    _currentUserService.User,
                    Request = request
                }));
        }

        var result = await next();

        if (scope is not null)
        {
            await scope.DisposeAsync();
        }

        return result;
    }
}
```

Aquí está el resumen de lo que ocurre en este comportamiento:

- `_logger.LogInformation`: En primer lugar, estamos registrando la solicitud realizada. Esto es útil en modo de desarrollo para obtener información adicional sobre cada consulta o comando que se ejecuta a través del mediador. Utilizamos un log template, que es una práctica recomendada para el registro de eventos. El uso de log templates facilita la implementación de soluciones de registro avanzadas, como Serilog y el envío de registros a Elastic Search (tema que podría abordarse en un artículo futuro). Es importante evitar concatenar cadenas de texto al registrar eventos y utilizar templates como se muestra aquí.
- Buscamos si el `IRequest` actual contiene el atributo `[AuditLog]`. Como mencioné antes, no queremos guardar registros de auditoría de todas las operaciones (solo de los comandos), por lo que esta condición es esencial.
- Si el atributo `[AuditLog]` está presente y la configuración indica que los registros de auditoría están habilitados, utilizamos los métodos proporcionados por Audit.NET para auditar la operación.
- Creamos un _scope_ de auditoría de Audit.NET para medir el tiempo transcurrido y realizar otras acciones que se pueden agregar.

Además, hemos registrado este Behavior y hemos agregado el atributo `[AuditLog]` a los comandos que deseamos auditar, como por ejemplo, `CreateProductCommand`.

```csharp
[AuditLog]
public class CreateProductCommand
 // Código omitido...
```

También hemos registrado el pipeline behavior en **ApplicationCore -> DependencyInjection**, donde habíamos registrado el comportamiento anterior:

```csharp
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(AuditLogsBehavior<,>));
```

Por último, en **appsettings.json**, hemos añadido una nueva sección para la configuración:

```json
"AuditLogs": {
  "Enabled":  true,
  "ConnectionString": "UseDevelopmentStorage=true"
}
```

Esta sección es útil, ya que en modo de desarrollo o en otros entornos, es posible que no deseemos almacenar registros de auditoría. Esto nos proporciona un control sobre la habilitación de los registros.

Con esto, ya podemos ejecutar la Web API y observar su comportamiento. Aunque aún no hemos configurado la cuenta de almacenamiento, Audit.NET generará archivos JSON en la raíz del proyecto de manera predeterminada.

Al ejecutar el comando `CreateProductCommand`, se generará un registro similar al siguiente:

```json
{
    "Environment": {
        "UserName": "isaac",
        "MachineName": "DELL-G5",
        "DomainName": "DELL-G5",
        "CallingMethodName": "MediatrExample.ApplicationCore.Common.Behaviours.AuditLogsBehavior\u00602\u002B\u003CHandle\u003Ed__3.MoveNext()",
        "AssemblyName": "MediatrExample.ApplicationCore, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null",
        "Culture": "es-MX"
    },
    "EventType": "CreateProductCommand",
    "StartDate": "2022-04-09T19:01:34.3460697Z",
    "EndDate": "2022-04-09T19:01:34.6543085Z",
    "Duration": 308,
    "User": {
        "Id": "759aa503-f916-4962-96ed-be0b416b5632",
        "UserName": "test_user",
        "IsAuthenticated": true
    },
    "Request": {
        "Description": "Random product",
        "Price": 558
    }
}
```

Este registro contiene información detallada sobre la operación auditada, incluyendo la duración, el usuario, y los datos de la solicitud.

Como puedes observar, en el Audit Scope hemos incluido las propiedades que consideramos más importantes: la solicitud (Request) y el usuario actual. De esta forma, cualquier operación realizada por un usuario quedará registrada de manera adecuada.

Sin embargo, para garantizar la seguridad y la integridad de estos registros, es necesario configurar un lugar adecuado donde almacenarlos. Aquí es donde entra en juego la configuración del Azure Storage Account.

El paquete NuGet que instalamos previamente proporciona métodos sencillos para configurar la persistencia de registros de auditoría. En el código a continuación, se muestra cómo configurar el almacenamiento en Azure Storage Blobs:

```csharp
public static IServiceCollection AddPersistence(this IServiceCollection services, IConfiguration configuration)
{
    // Omitido...
 
    Configuration.Setup()
        .UseAzureStorageBlobs(config => config
            .WithConnectionString(configuration["AuditLogs:ConnectionString"])
            .ContainerName(ev => $"mediatrlogs{DateTime.Today:yyyyMMdd}")
            .BlobName(ev =>
            {
                var currentUser = ev.CustomFields["User"] as CurrentUser;

                return $"{ev.EventType}/{currentUser?.Id}_{DateTime.UtcNow.Ticks}.json";
            })
        );

    return services;
}
```

Aquí se configuran los siguientes aspectos:

- **WithConnectionString**: Se proporciona la cadena de conexión del Azure Storage Account.
- **ContainerName**: Los archivos de registro se almacenan en contenedores, y se crea un contenedor diferente para cada día, utilizando el formato **mediatrlogs20220409**, por ejemplo.
- **BlobName**: Se establece la ruta en la que se guardarán en el contenedor. Los registros se agrupan por carpetas según el nombre del comando, y el nombre del archivo incluye el ID del usuario. Esto facilita la búsqueda de registros por ID de usuario y permite ver todas las acciones realizadas por ese usuario.

Cuando visualices esto en el Azure Storage Explorer, se verá de la siguiente manera:

![Imagen](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/72gw8sb0agggl2jjy7z1.png)

Si exploras una carpeta en particular:

![Imagen](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6g1ik8akctmh17weaxya.png)

De esta manera, puedes buscar registros por ID de usuario. Sin embargo, ten en cuenta que, aunque puedes buscar por día, no es posible definir un rango de horas al realizar búsquedas. Si necesitas realizar búsquedas más detalladas, podría ser preferible utilizar Azure Storage Tables u otra solución que permita almacenar y buscar una gran cantidad de información.

Un ejemplo del contenido de un registro en formato JSON es el siguiente:

```json
{
    "Environment": {
        "UserName": "isaac",
        "MachineName": "DELL-G5",
        "DomainName": "DELL-G5",
        "CallingMethodName": "MediatrExample.ApplicationCore.Common.Behaviours.AuditLogsBehavior\u00602\u002B\u003CHandle\u003Ed__4.MoveNext()",
        "AssemblyName": "MediatrExample.ApplicationCore, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null",
        "Culture": "es-MX"
    },
    "EventType": "UpdateProductCommand",
    "StartDate": "2022-04-10T00:00:58.8335969Z",
    "EndDate": "2022-04-10T00:00:58.8364306Z",
    "Duration": 3,
    "User": {
        "Id": "759aa503-f916-4962-96ed-be0b416b5632",
        "UserName": "test_user",
        "IsAuthenticated": true
    },
    "Request": {
        "ProductId": 1,
        "Description": "iPhone SE 2022",
        "Price": 11599
    }
}
```

En resumen, esta implementación de auditoría con Audit.NET y Azure Storage Blobs es una herramienta valiosa para crear sistemas auditables de manera efectiva. A pesar de la complejidad de los sistemas de auditoría, las herramientas actuales hacen que sea relativamente sencillo implementar registros de auditoría. Espero que esta información te haya resultado útil. Puedes acceder al [código de este post aquí](https://github.com/isaacOjeda/AspNetCoreMediatRExample/tree/parte-7).

# Referencias
- [Audit.NET](https://github.com/thepirat000/Audit.NET)
- [Audit.NET AzureStorageBlobs](https://github.com/thepirat000/Audit.NET/blob/master/src/Audit.NET.AzureStorageBlobs/README.md)
- [Jason Taylor - Clean Architecture](https://github.com/jasontaylordev/CleanArchitecture)
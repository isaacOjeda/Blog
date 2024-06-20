## Introducción

ASP.NET Core Minimal APIs proporciona una forma sencilla y elegante de crear endpoints para nuestra aplicación web sin la necesidad de usar controladores tradicionales. Sin embargo, a medida que la cantidad de endpoints crece, puede volverse tedioso y poco práctico registrar cada uno de ellos manualmente en el archivo `Program.cs`. Afortunadamente, podemos abordar este problema y facilitar aún más el proceso de registro utilizando la librería Scrutor.

## Retos con la Registración de Endpoints en Minimal APIs

Si bien los Endpoint Groups y los métodos de extensión pueden ayudarnos a organizar los grupos de endpoints, todavía enfrentamos un desafío cuando tenemos un gran número de ellos. Registrar 100 grupos de endpoints manualmente en `Program.cs` es una tarea engorrosa y propensa a errores. A diferencia de lo que ocurre con los controladores tradicionales en MVC, donde este proceso es más sencillo.

## Registro Automático de Endpoints con Scrutor

Scrutor es una librería para .NET Core que nos permite simplificar el proceso de escaneo y registro de servicios en el contenedor de inyección de dependencias de nuestra aplicación. Facilita el uso de la inyección de dependencias, un patrón de diseño que nos permite proporcionar instancias de objetos a una clase en lugar de que la propia clase cree esas instancias directamente.

Las principales características de Scrutor incluyen:

1. **Escaneo automático de ensamblados:** Scrutor nos permite escanear automáticamente los ensamblados en busca de tipos que implementen ciertas interfaces o cumplan con determinadas condiciones.
2. **Registración automática de servicios:** Con Scrutor, podemos registrar automáticamente los servicios encontrados durante el escaneo en el contenedor de inyección de dependencias, evitando así tener que hacerlo manualmente.
3. **Personalización del escaneo:** Es posible personalizar el escaneo para filtrar qué tipos se incluirán o excluirán del registro.
4. **Reemplazo de servicios existentes:** Scrutor nos permite especificar si deseamos reemplazar los servicios existentes por los encontrados durante el escaneo, lo que puede ser útil para realizar pruebas unitarias o cambiar implementaciones en diferentes ambientes.
5. **Reglas de convención:** Podemos utilizar reglas de convención para facilitar la clasificación y el registro de servicios basados en ciertas convenciones de nomenclatura o estructura.

## Implementando el Registro Automático de Endpoints con Scrutor

Para comenzar a utilizar Scrutor, primero debemos agregarlo como una dependencia en nuestro proyecto. Podemos hacerlo agregando el siguiente paquete NuGet a nuestro archivo `.csproj`:

```xml
<ItemGroup>
  <PackageReference Include="Scrutor" Version="4.2.2" />
</ItemGroup>
```

Luego, necesitamos crear una interfaz que servirá como marcador para nuestros endpoints. Esta interfaz, llamada `IEndpoint`, nos permitirá detectar todas las clases de endpoints en tiempo de ejecución:

```csharp
namespace MinimalApiScrutor;
  
public interface IEndpoint
{
    void MapEndpoint(IEndpointRouteBuilder endpoints);
}
```

A continuación, utilizaremos métodos de extensión para buscar e invocar automáticamente los endpoints que implementan la interfaz `IEndpoint`. Primero, crearemos la clase `RouteExtensions`:

```csharp
using System.Reflection;

namespace MinimalApiScrutor.Extensions;
  
public static class RouteExtensions
{
    public static IServiceCollection AddEndpoints(this IServiceCollection services)
    {
        services.Scan(scan => scan
            .FromAssemblies(Assembly.GetExecutingAssembly())
            .AddClasses(classes => classes.AssignableTo<IEndpoint>())
            .AsImplementedInterfaces()
            .WithTransientLifetime()
        );              
  
        return services;
    }
  
    public static WebApplication MapEndpoints(this WebApplication app)
    {
        var endpoints = app.Services.GetServices<IEndpoint>();
  
        foreach (var endpoint in endpoints)
        {
            endpoint.MapEndpoint(app);
        }
  
        return app;
    }
}
```

Explicación de los métodos:

1. **AddEndpoints():** Este método de extensión se aplica a un objeto `IServiceCollection`, que es el contenedor de inyección de dependencias en ASP.NET Core. La función de este método es escanear el ensamblado actual (el ensamblado que contiene el código actual) en busca de clases que implementen la interfaz `IEndpoint`. Luego, registra automáticamente estas clases en el contenedor de inyección de dependencias con un ciclo de vida de tipo "transient" (se crea una nueva instancia cada vez que se solicita). En resumen, este método busca e inicializa todas las clases de endpoints que implementen `IEndpoint` y las registra en el contenedor.
2. **MapEndpoints():** Este otro método de extensión se aplica a un objeto `WebApplication`, que es el punto de entrada para configurar y ejecutar la aplicación web en ASP.NET Core. La función de este método es recuperar todas las implementaciones de `IEndpoint` que fueron registradas previamente en el contenedor de inyección de dependencias mediante `AddEndpoints()`. Luego, invoca el método `MapEndpoint(app)` en cada una de estas implementaciones para mapear los endpoints correspondientes en la aplicación web. En resumen, este método toma todas las clases de endpoints que se registraron y las asocia con las rutas y métodos HTTP en los que fueron definidas, permitiendo que estén disponibles para ser accedidas desde el exterior.

En conjunto, estos dos métodos hacen que el proceso de registro y mapeo de endpoints sea automático. Al utilizar la extensión `AddEndpoints()` en `Program.cs`, se asegura que todos los endpoints implementados en la aplicación sean detectados y registrados sin necesidad de hacerlo manualmente uno por uno. Luego, mediante la extensión `MapEndpoints()`, todos esos endpoints son mapeados y listos para ser utilizados.

La ventaja de utilizar Scrutor en este contexto radica en su capacidad para buscar y registrar automáticamente las clases de endpoints sin necesidad de hacerlo explícitamente, lo que simplifica el código y lo hace más fácil de mantener y escalar a medida que la aplicación crece y evoluciona.

Finalmente, en nuestro archivo `Program.cs`, usaremos estos métodos de extensión para registrar automáticamente todos los endpoints:

```csharp
using MinimalApiScrutor.Extensions;
  
var builder = WebApplication.CreateBuilder(args);
  
builder.Services.AddEndpoints(); // <---
  
var app = builder.Build();
  
app.MapGet("/", () => "Hello World!");
  
app.MapEndpoints(); // <---
  
app.Run();
```

Con `AddEndpoints()`, registramos todos los endpoints encontrados, y con `MapEndpoints()`, los agregamos al pipeline para que estén disponibles según la ruta y el método HTTP en los que se registraron.

## Ejemplos de Endpoints

A continuación, te presento ejemplos de cómo implementar endpoints. En este caso, crearemos dos clases que representan endpoints diferentes:

```csharp
namespace MinimalApiScrutor.Features.Users;
  
public class GetUsers : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder endpoints)
    {
        endpoints.MapGet("/api/users", () => new List<User>
        {
            new User { Id = Guid.NewGuid(), Name = "User 1", Email = "user@mail.com"},
            new User { Id = Guid.NewGuid(), Name = "User 2", Email = "user2@mail.com"},
        });
    }
}
```

```csharp
namespace MinimalApiScrutor.Features.Users;
  
public class CreateUser : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder endpoints)
    {
        endpoints.MapPost("/api/users", (CreateUserRequest request) =>
        {
            var user = new User
            {
                Id = Guid.NewGuid(),
                Name = request.Name,
                Email = request.Email,
                Password = request.Password
            };
  
            // TODO: Save user to database
  
            return Results.Ok(new CreateUserResponse
            {
                UserId = user.Id
            });
        });
  
    }
}
  
public class CreateUserRequest
{
    public string Name { get; set; } = default!;
    public string Email { get; set; } = default!;
    public string Password { get; set; } = default!;
}
  
public class CreateUserResponse
{
    public Guid UserId { get; set; } = default!;
}
```

Estos ejemplos muestran cómo organizar los diferentes endpoints en clases separadas, lo que facilita la administración y el mantenimiento de la aplicación.

Aquí ya no tienes que hacer cosa alguna, por que estos endpoints serán registrados y mapeados automáticamente por los métodos de extensión.

## Conclusión

En resumen, Scrutor es una herramienta que simplifica el proceso de registro automático de endpoints en aplicaciones ASP.NET Core Minimal APIs (realmente, de cualquier cosA). Al utilizar Scrutor, podemos evitar el registro manual tedioso y propenso a errores, y en su lugar, permitir que la librería realice el escaneo y registro automático de nuestros servicios. Esto no solo hace que nuestro código sea más limpio y fácil de entender, sino que también mejora la organización y el mantenimiento de nuestra aplicación. Con Scrutor, podemos aprovechar al máximo el patrón de diseño de inyección de dependencias y desarrollar aplicaciones más robustas y escalables.
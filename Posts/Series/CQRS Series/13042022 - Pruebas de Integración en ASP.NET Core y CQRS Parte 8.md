# Introducción
Bienvenidos a la siguiente entrega de nuestra serie de publicaciones sobre ASP.NET Core y CQRS. En esta ocasión, nos centraremos en un tema fundamental: las Pruebas de Integración y cómo implementarlas en tu proyecto.

Si quieres ver el código completo de este artículo, puedes consultarlo en mi [repositorio de GitHub 🐙](https://github.com/isaacOjeda/AspNetCoreMediatRExample/tree/parte-8).

> Actualización Importante 💡: Inicialmente, planeé que esta serie fuera breve y sencilla, pero terminé escribiendo múltiples artículos sobre diversos temas. En este post, utilizaré SQLite y recrearé la base de datos como parte de cada prueba. Sin embargo, más adelante, decidí cambiar a SQL Server y utilizar **Respawn** para restablecer la base de datos. En futuras publicaciones, profundizaré en el uso de Respawn, que permite eliminar registros con DELETE en lugar de recrear tablas completas en la base de datos.

# Pruebas de Integración
Las pruebas de Integración desempeñan un papel crucial para asegurarnos de que los módulos desarrollados funcionen de manera conjunta en la aplicación final. En resumen, estas pruebas nos permiten verificar que todas las piezas se ensamblen correctamente.

En nuestro caso, hemos estado desarrollando Queries y Comandos que se ejecutan a través de una Web API. Las pruebas de integración implican conectar todos los componentes de la aplicación, crear un entorno de pruebas y ejecutar estos Queries y Comandos para confirmar su funcionamiento.

A diferencia de las pruebas unitarias, donde nos enfocamos en aspectos específicos y pequeños, las pruebas de integración abarcan la configuración completa, incluyendo pipelines, filtros, decoradores y todo lo que forma parte de la Web API.

En este contexto, no necesitamos preocuparnos tanto por crear simulaciones (Mocks) debido a que estamos utilizando una base de datos SQLite. En la vida real, es posible que empleemos una base de datos como SQL Server y existen métodos sencillos para crear bases de datos de prueba, como un Local DB, que se pueden ejecutar sin problemas en Github Actions o Azure DevOps.

Es importante destacar que no podemos utilizar bases de datos en memoria, ya que esto no se considera una prueba de integración completa. Queremos evaluar el comportamiento de la base de datos desde la solicitud hasta la persistencia.
## Tecnologías a utilizar
En esta sección, te presentamos las tecnologías que vamos a emplear en nuestras pruebas de integración:

- [NUnit](https://nunit.org/): Utilizaremos este framework de pruebas para estructurar y ejecutar nuestras pruebas.
- [FluentAssertions](https://github.com/fluentassertions/fluentassertions): Esta biblioteca nos brindará la capacidad de realizar afirmaciones (assertions) de manera flexible y fácil de comprender, lo que simplificará la validación de resultados.
- [Respawn](https://github.com/jbogard/Respawn): Aunque no lo utilizaremos en este artículo, es una herramienta valiosa cuando trabajamos con SQL Server y deseamos restablecer la base de datos de pruebas sin tener que eliminar tablas. Si deseas explorar más sobre su uso, puedes visitar mi [repositorio](https://github.com/isaacOjeda/MinimalApiArchitecture), donde realizo pruebas de integración con SQL Server Local DB y Respawn.

## Proyecto MediatRExample.IntegrationTests
El proyecto que emplearemos para nuestras pruebas será de tipo NUnit 3 Test. Puedes crearlo en Visual Studio o utilizando el comando `dotnet` de la siguiente manera:

```bash
dotnet new nunit -o tests/MediatRExample.IntegrationTests
```

Si estás utilizando `dotnet`, asegúrate de agregar el proyecto recién creado a la solución con el comando `dotnet sln add <Ruta al .csproj>`.

A continuación, agregaremos las siguientes dependencias al proyecto:

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.NET.Test.Sdk" Version="16.11.0" />
  <PackageReference Include="NUnit" Version="3.13.2" />
  <PackageReference Include="NUnit3TestAdapter" Version="4.0.0" />
  <PackageReference Include="coverlet.collector" Version="3.1.0" />

  <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="6.0.0" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="6.0.3" />
  <PackageReference Include="FluentAssertions" Version="6.2.0" />
</ItemGroup>

<ItemGroup>
  <ProjectReference Include="..\..\src\MediatrExample.ApplicationCore\MediatrExample.ApplicationCore.csproj" />
  <ProjectReference Include="..\..\src\MediatrExample.WebApi\MediatrExample.WebApi.csproj" />
</ItemGroup>
```

Es importante destacar la importancia del paquete `Microsoft.AspNetCore.Mvc.Testing`, ya que nos permitirá crear un servidor falso (fake) para realizar llamadas HTTP en un entorno de pruebas.

## Arrange, Act & Assert
La metodología "Arrange, Act & Assert" simplifica la forma en que abordaremos las pruebas. Básicamente, se divide en tres pasos:

- **Arrange:** En esta etapa, preparamos todo lo necesario para ejecutar una prueba de un caso de uso específico. Por ejemplo, si deseamos probar la creación de productos, debemos considerar distintos escenarios, como un usuario con un rol de administrador y campos completos, entre otros.
- **Act:** Aquí ejecutamos la prueba propiamente dicha.
- **Assert:** En esta fase, verificamos si los resultados obtenidos a partir de los preparativos del "Arrange" coinciden con los resultados esperados. Aquí determinamos si la prueba ha sido exitosa.

Para facilitar especialmente la parte del "Arrange", crearemos una clase que servirá de base para todas nuestras pruebas.

### IntegrationTests -> ApiWebApplicationFactory
Para realizar pruebas de integración y establecer un entorno de prueba que nos permita realizar llamadas HTTP, aprovecharemos lo que ofrece el paquete Mvc.Testing. A partir de .NET 6, se introdujo la asombrosa clase llamada `WebApplicationFactory<T>`, que nos permite inicializar la aplicación de manera completamente aislada en la memoria, lo que nos permite realizar llamadas HTTP de prueba.

Dado que tenemos un proyecto de Web API que contiene un `Program.cs` para inicializar todo un host y permitir llamadas HTTP desde el exterior en un puerto específico, lo que nos permite esta [`WebApplicationFactory`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.testing.webapplicationfactory-1?view=aspnetcore-6.0) es hacer lo mismo pero con el propósito de inicializarlo de forma aislada y crear un servidor de prueba (**TestServer**).

Con este **TestServer**, podemos crear clientes HTTP y realizar llamadas de prueba, todo en un entorno controlado. Esto es fundamental, ya que estas pruebas se pueden automatizar en nuestro flujo de Continuous Delivery, como GitHub Actions (que planeo abordar más adelante).

La creación de este factory del TestServer es muy sencilla. Solo heredamos de `WebApplicationFactory`:

```csharp
using MediatrExample.WebApi;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.Extensions.Hosting;

namespace MediatRExample.IntegrationTests;
public class ApiWebApplicationFactory : WebApplicationFactory<Api>
{   
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");

        base.ConfigureWebHost(builder);
    }

    protected override IHost CreateHost(IHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Configura cualquier Mock o similares aquí.
        });

        return base.CreateHost(builder);
    }
}
```

Puntos importantes a destacar aquí:

- La clase `Api`: Agregué esta clase en el proyecto **WebApi** para permitir una búsqueda de ensamblados (Assembly Scan) por parte del factory y que pueda localizar un `Program.cs` (donde inicializamos y configuramos la Web API).
- `ConfigureWebHost`: Esta función permite configurar el Web Host, y aunque por ahora solo establecemos un entorno distinto (por defecto sería **Development**, y en producción sería **Production**), es útil para que puedas ver cómo podemos configurar el **TestServer** de manera diferente a como se haría en Desarrollo o Producción. Para lograr esto, he creado un nuevo archivo de configuración llamado **appsettings.Testing.json** en el proyecto WebApi.
- `CreateHost`: Al crear el Host, podemos especificar dependencias diferentes de las que se usarían en el proyecto en producción. A veces necesitamos hacer Mocks de ciertas partes del sistema. Por ejemplo, en mi proyecto [Minimal APIs](https://github.com/isaacOjeda/MinimalApiArchitecture), uso SQL Server en la aplicación normal, pero en el **TestServer** empleo SQL Server Local DB (archivos .mdf en lugar de una base de datos en un servidor). Esto permite que las pruebas se ejecuten sin problemas en GitHub Actions o Azure DevOps para evaluar bases de datos reales.

### IntegrationTests -> TestBase
La clase `TestBase` es una herramienta valiosa que nos ayudará a preparar nuestras pruebas de manera más práctica y reutilizable. Con ella, podremos abordar varios escenarios de prueba, como:

- Realizar llamadas autenticadas en nombre de usuarios específicos.
- Probar tanto usuarios autenticados como usuarios anónimos.
- Evaluar operaciones con usuarios autenticados que pueden tener permisos (roles) limitados o roles completos.
- Verificar los resultados de las operaciones, por ejemplo, comprobar si un producto se ha creado correctamente en la base de datos.
- Autenticarnos y utilizar tokens JWT válidos en las pruebas, tal como se mencionó anteriormente.
- Mantener una base de datos limpia para cada prueba, lo que significa que cada prueba tendrá su propio caso de uso.
- Tener la capacidad de crear información específica en ciertas pruebas y omitirla en otras.

Dentro del proyecto de NUnit, creamos la clase `TestBase`, que facilita todas estas tareas. Aquí está el código:

```csharp
using MediatR;
using MediatrExample.ApplicationCore.Features.Auth;
using MediatrExample.ApplicationCore.Infrastructure.Persistence;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using NUnit.Framework;
using System;
using System.Linq.Expressions;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Threading.Tasks;

namespace MediatRExample.IntegrationTests;

public class TestBase
{
    protected ApiWebApplicationFactory Application;

    // Crea un usuario de prueba según los parámetros
    public async Task<(HttpClient Client, string UserId)> CreateTestUser(string userName, string password, string[] roles)
    {
        using var scope = Application.Services.CreateScope();
        var userManager = scope.ServiceProvider.GetRequiredService<UserManager<IdentityUser>>();

        var newUser = new IdentityUser(userName);

        await userManager.CreateAsync(newUser, password);

        foreach (var role in roles)
        {
            await userManager.AddToRoleAsync(newUser, role);
        }

        var accessToken = await GetAccessToken(userName, password);

        var client = Application.CreateClient();
        client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);

        return (client, newUser.Id);
    }

    // Al finalizar cada prueba, se restablece la base de datos
    [TearDown]
    public async Task Down()
    {
        await ResetState();
    }

    // Crea un HttpClient con un JWT válido para un usuario Admin
    public Task<(HttpClient Client, string UserId)> GetClientAsAdmin() =>
        CreateTestUser("user@admin.com", "Pass.W0rd", new string[] { "Admin" });

    // Crea un HttpClient con un JWT válido para un usuario predeterminado
    public Task<(HttpClient Client, string UserId)> GetClientAsDefaultUserAsync() =>
        CreateTestUser("user@normal.com", "Pass.W0rd", Array.Empty<string>());

    // Libera recursos al finalizar todas las pruebas
    [OneTimeTearDown]
    public void RunAfterAnyTests()
    {
        Application.Dispose();
    }

    // Inicializa la API y la base de datos antes de comenzar las pruebas
    [OneTimeSetUp]
    public void RunBeforeAnyTests()
    {
        Application = a new ApiWebApplicationFactory();

        EnsureDatabase();
    }

    // Atajo para ejecutar IRequests con el Mediador
    public async Task<TResponse> SendAsync<TResponse>(IRequest<TResponse> request)
    {
        using var scope = Application.Services.CreateScope();

        var mediator = scope.ServiceProvider.GetRequiredService<ISender>();

        return await mediator.Send(request);
    }

    // Atajo para agregar entidades a la base de datos
    protected async Task<TEntity> AddAsync<TEntity>(TEntity entity) where TEntity : class
    {
        using var scope = Application.Services.CreateScope();

        var context = scope.ServiceProvider.GetService<MyAppDbContext>();

        context.Add(entity);

        await context.SaveChangesAsync();

        return entity;
    }

    // Atajo para buscar entidades por clave primaria
    protected async Task<TEntity> FindAsync<TEntity>(params object[] keyValues) where TEntity : class
    {
        using var scope = Application.Services.CreateScope();

        var context = scope.ServiceProvider.GetService<MyAppDbContext>();

        return await context.FindAsync<TEntity>(keyValues);
    }

    // Atajo para buscar entidades según un criterio
    protected async Task<TEntity> FindAsync<TEntity>(Expression<Func<TEntity, bool>> predicate) where TEntity : class
    {
        using var scope = Application.Services.CreateScope();

        var context = scope.ServiceProvider.GetService<MyAppDbContext>();

        return await context.Set<TEntity>().FirstOrDefaultAsync(predicate);
    }

    // Asegura la creación de la base de datos
    private void EnsureDatabase()
    {
        using var scope = Application.Services.CreateScope();
        var context = scope.ServiceProvider.GetService<MyAppDbContext>();

        context.Database.EnsureCreated();
    }

    // Atajo para autenticar a un usuario para las pruebas
    private async Task<string> GetAccessToken(string userName, string password)
    {
        using var scope = Application.Services.CreateScope();

        var result = await SendAsync(new TokenCommand
        {
            UserName = userName,
            Password = password
        });

        return result.AccessToken;
    }

    // Asegura la limpieza de la base de datos
    private async Task ResetState()
    {
        using var scope = Application.Services.CreateScope();
        var context = scope.ServiceProvider.GetService<MyAppDbContext>();
		// En un escenario real, podríamos usar Respawn para restablecer la base de datos en lugar de eliminar y recrear tablas.
        context.Database.EnsureDeleted();
        context.Database.EnsureCreated();

        await MyAppDbContextSeed.SeedDataAsync(context);
    }
}
```

Esta clase `TestBase` proporciona una base sólida para la realización de pruebas de integración, facilitando la preparación y ejecución de pruebas de diferentes escenarios. Cada método tiene una descripción para ayudar a comprender su propósito.

> Nota 👀: `MyAppDbContextSeed` es una clase que he agregado en la capa de persistencia para centralizar los datos de prueba. Te invito a visitar el repositorio para obtener más información.

### IntegrationTests -> Features -> Products -> GetProductsQueryTests

Ahora que hemos establecido la infraestructura para las pruebas, podemos centrarnos en las pruebas específicas. Por ejemplo, veamos cómo probar el Query `GetProductsQuery`. Aquí tenemos dos casos de prueba para ilustrar cómo se hace:

```csharp
using FluentAssertions;
using MediatrExample.ApplicationCore.Features.Products.Queries;
using NUnit.Framework;
using System.Collections.Generic;
using System.Net.Http;
using System.Net.Http.Json;
using System.Threading.Tasks;

namespace MediatRExample.IntegrationTests.Features.Products;
public class GetProductsQueryTests : TestBase
{

    [Test]
    public async Task Products_Obtained_WithAuthenticatedUser()
    {
        // Arrange
        var (Client, UserId) = await GetClientAsDefaultUserAsync();

        // Act
        var products = await Client.GetFromJsonAsync<List<GetProductsQueryResponse>>("/api/products");

        // Assert
        products.Should().NotBeNullOrEmpty();
        products?.Count.Should().Be(2);
    }

    [Test]
    public async Task Products_ProducesException_WithAnonymUser()
    {
        // Arrange
        var client = Application.CreateClient();

        // Act and Assert
        await FluentActions.Invoking(() =>
                client.GetFromJsonAsync<List<GetProductsQueryResponse>>("/api/products"))
                    .Should().ThrowAsync<HttpRequestException>();
    }
}
```

Aquí tenemos dos casos de uso para probar el Query. En el primer caso, estamos realizando una consulta exitosa utilizando una llamada HTTP autenticada. En el segundo caso, estamos realizando una llamada sin un token JWT, por lo que esperamos que la llamada falle y genere una excepción.

Para evaluar los resultados de las pruebas, utilizamos la biblioteca `FluentAssertions`, que facilita la comprobación de los resultados de manera clara y concisa.

Es importante ser descriptivo en la nomenclatura de las pruebas, ya que al revisar los informes de pruebas, el nombre del método es la principal fuente de información. Esto ayuda a comprender qué se está probando sin necesidad de leer todo el código.

### IntegrationTests -> Features -> Products -> CreateProductCommandTests

Siguiendo el ejemplo, ahora veamos cómo probar la creación de productos. Aquí presentamos varios casos de prueba:

```csharp
using FluentAssertions;
using MediatrExample.ApplicationCore.Domain;
using MediatrExample.ApplicationCore.Features.Products.Commands;
using NUnit.Framework;
using System.Net;
using System.Net.Http;
using System.Net.Http.Json;
using System.Threading.Tasks;

namespace MediatRExample.IntegrationTests.Features.Products;
public class CreateProductCommandTests : TestBase
{

    [Test]
    public async Task Product_IsCreated_WhenValidFieldsAreProvided_AndUserIsAdmin()
    {
        // Arrange
        var (Client, UserId) = await GetClientAsAdmin();

        // Act
        var command = new CreateProductCommand
        {
            Description = "Test Product",
            Price = 999
        };

        var result = await Client.PostAsJsonAsync("api/products", command);

        // Assert

        FluentActions.Invoking(() => result.EnsureSuccessStatusCode())
            .Should().NotThrow();

        var product = await FindAsync<Product>(q => q.Description == command.Description);

        product.Should().NotBeNull();
        product.Description.Should().Be(command.Description);
        product.Price.Should().Be(command.Price);
        product.CreatedBy.Should().Be(UserId);
    }

    [Test]
    public async Task Product_IsNotCreated_WhenInvalidFieldsAreProvided_AndUserIsAdmin()
    {
        // Arrange
        var (Client, UserId) = await GetClientAsAdmin();

        // Act
        var command = new CreateProductCommand
        {
            Description = "Test Product",
            Price = 0
        };

        var result = await Client.PostAsJsonAsync("api/products", command);

        // Assert
        FluentActions.Invoking(() => result.EnsureSuccessStatusCode())
            .Should().Throw<HttpRequestException>();

        result.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }

    [Test]
    public async Task Product_IsNotCreated_WhenValidFieldsAreProvided_AndDefaultUser()
    {
        // Arrange
        var command = new CreateProductCommand
        {
            Description = "Test Product",
            Price = 999
        };
        var (Client, UserId) = await GetClientAsDefaultUserAsync();

        // Act        
        var result = await Client.PostAsJsonAsync("api/products", command);

        // Assert
        FluentActions.Invoking(() => result.EnsureSuccessStatusCode())
            .Should().Throw<HttpRequestException>();

        result.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }
}
```

En estos casos de prueba, estamos validando que:

- Se pueden crear productos solo si el usuario tiene el rol **Admin** y los campos son correctos.
- No se pueden crear productos incluso si el usuario es **Admin** si los campos son incorrectos.
- No se pueden crear productos con un usuario **Default** (sin permisos de Admin).

En el primer caso de prueba, después de crear un producto, consultamos nuevamente la base de datos para verificar si se creó correctamente. Utilizamos el método `FindAsync` para buscar el producto recién creado y comprobar sus propiedades.

> Nota 👀: Es importante recordar que este ejemplo puede no ser perfecto y puede haber casos de error, como productos con el mismo nombre. Sin embargo, este código sirve como guía base.

Todo lo que hemos configurado en `TestBase` se puede utilizar en estas pruebas para facilitar la preparación y ejecución de las mismas. Aunque la base de datos de prueba es SQLite, este enfoque también es válido para SQL Server y LocalDb en pruebas.

# Conclusión
Hemos aprendido cómo realizar pruebas de integración en ASP.NET Core con .NET 6+. La creación de un servidor de prueba es ahora extremadamente sencilla gracias a clases como `WebApplicationFactory`.

Si tienes alguna pregunta o necesitas más información, no dudes en contactarme a través de [mi cuenta de Twitter](https://twitter.com/balunatic).

También puedes visitar mi [canal de YouTube](https://www.youtube.com/channel/UCVWNtfrGkpMsUymOQzndBRQ).
# Referencias
- [IntegrationTest - Martin Fowler](https://martinfowler.com/bliki/IntegrationTest.html)
- [jasontaylordev/CleanArchitecture](https://github.com/jasontaylordev/CleanArchitecture)

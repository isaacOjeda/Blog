# Introducción
Continuando con esta serie de publicaciones sobre temas de interés en ASP.NET Core y CQRS.

Hoy nos dedicaremos a aprender un poco sobre las Pruebas de Integración y su implementación.

El código completo lo puedes ver en mi [GitHub 🐙](https://github.com/isaacOjeda/AspNetCoreMediatRExample/tree/parte-8).

> Nota de Actualización 💡: Al iniciar esta serie de CQRS pensé en hacer algo corto y sencillo pero al final terminé haciendo muchos artículos sobre distintos temas. En este post utilizo SQLite y elimino y recreo la base de datos como parte del proceso de cada prueba.
> Esto no es óptimo cuando se tienen muchas tablas y en partes posteriores decidí utilizar SQLServer y Respawn para el reset de la base de datos, próximamente escribiré más sobre respawn pero en escencia lo que hace es eliminar los registros con DELETEs en lugar de eliminar y recrear las tablas de la BD para hacer el reset.


# Pruebas de Integración
Las pruebas de Integración determinan si los módulos desarrollados independientemente funcionan correctamente una vez que todo se conecta en la aplicación final.

Es decir, nosotros hemos estado desarrollando Queries y Comandos que son ejecutados por una Web API. Una prueba de integración consiste en conectar todos los componentes de la aplicación y crear un entorno de pruebas y ejecutar todos esos Queries y Comandos para verificar su funcionalidad.

A diferencia del Unit Testing, en Integration Tests se busca probar toda la configuración, pipelines, filters, behaviors, y todo eso que se conecta al final en la Web API. Unit Testing solo abarca cosas pequeñas y funcionalidad específicas.

Con Integration Tests no batallaremos demasiado con hacer Mocks, ya que estamos usando una base de datos SQLite. **En la vida real, usaremos una base de datos tal vez de SQL Server y existen formas sencillas de crear bases de datos de pruebas como un Local DB que perfectamente pueden correr en Github Actions o Azure DevOps.**

**No podemos usar bases de datos en memoria** porque realmente no se considera una prueba de integración, ya que queremos también probar cómo se comporta la base de datos. Desde el Request hasta la persistencia.

## Tecnologías a usar
Vamos a utilizar lo siguiente:
- [NUnit]([NUnit.org](https://nunit.org/)) como framework de testing
- [FluentAssertions](https://github.com/fluentassertions/fluentassertions) para Asserts muy flexibles y entendibles
- [Respawn](https://github.com/jbogard/Respawn) para resetear base de datos
  - Este realmente no lo usaremos en este post, pero es bien útil cuando usamos SQL Server y queremos resetear la BD de pruebas sin tener que eliminar tablas
  - En este [repositorio](https://github.com/isaacOjeda/MinimalApiArchitecture) hago pruebas de integración con SQL Server Local DB y Respawn

## Proyecto MediatRExample.IntegrationTests
El proyecto que usaremos será del tipo NUnit 3 Test, lo podemos crear utilizando Visual Studio o con dotnet:

```bash
dotnet new nunit -o tests/MediatRExample.IntegrationTests
```
Si usas `dotnet`, hay que agregar el proyecto creado a la solución con `dotnet sln add <Ruta al .csproj>`.

A este proyecto le agregamos las siguientes dependencias:
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

El paquete `Microsoft.AspNetCore.Mvc.Testing` es muy importante, ya que esta nos va a permitir crear un servidor *fake* para poder hacer llamadas Http en un ambiente de pruebas.

## Arrange, Act & Assert
Esta forma de ver las pruebas me facilita comprender como vamos a hacer testing. Básicamente trata de:
- **Arrange.** Consiste en preparar todo para realizar una prueba de un caso de uso en específico
  - Tal vez queremos probar que podamos crear productos, por lo que debemos de considerar distintos escenarios (usuario con Rol Admin, campos completos, etc)
- **Act.** Consiste en efectuar la prueba
- **Assert.** Consiste en verificar si los resultados según el Arrange que hicimos llevaron a los resultados esperados. Aquí se determina si la prueba es exitosa o no.

Para facilitar la parte de **Arrange** (principalmente) crearemos una clase en la que se basarán todas nuestras pruebas.

### IntegrationTests -> ApiWebApplicationFactory
Para poder hacer pruebas de integración y poder tener un ambiente de Testing que nos permita realizar llamadas HTTP, vamos a aprovechar lo que el paquete Mvc.Testing ofrece. Partiendo de .NET 6 (no recuerdo muy bien) se agregó esta asombrosa clase llamada `WebApplicationFactory<T>` que nos permite inicializar la aplicación 100% aislado en memoria para poder realizar llamadas HTTP de prueba.

Nosotros tenemos un proyecto Web API, el cual contiene un `Program.cs` que inicializa todo un Host para poder recibir llamadas HTTP desde el exterior y en cierto puerto. Lo que nos permite este [`WebApplicationFactory`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.testing.webapplicationfactory-1?view=aspnetcore-6.0) es hacer lo mismo, pero con la intención de inicializarlo de forma aislada y crear un **TestServer**.

Teniendo este **TestServer** podemos crear clientes Http y realizar llamadas de prueba, todo sucediendo ahí mismo. Esto es importante, ya que estas pruebas las podemos automatizar en nuestro pipeline de Continuous Delivery (como GitHub Actions, que tengo planeado hablar de ellos después).

Para crear este factory del TestServer es muy sencillo, solo heredamos de `WebApplicationFactory`:

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
            // Configurar cualquier Mock o similares
        });

        return base.CreateHost(builder);
    }
}
```

Lo importante resaltar aquí:
- Clase `Api`: Es una clase que agregué en el proyecto **WebApi** para poder hacer un _Assembly Scan_ y que el factory pueda saber en donde buscar un `Program.cs` (que es donde inicializamos y configuramos la Web API)
- `ConfigureWebHost`: Permite configurar el Web Host pero lo único que hacemos por ahora es solo especificar un Environment distinto (por default sería **Development**, y en Release sería **Production**)
  - Para que esto funcione, he creado un nuevo appsettings en WebApi llamado **appsettings.Testing.json**
    - Esto para mostrar cómo podemos configurar nuestro **TestServer** de distinta forma a como se usaría en Desarrollo o Producción
- `CreateHost`: Al crear el Host podemos especificar dependencias distintas a las que se usarían en el proyecto en producción. A veces vamos a necesitar hacer Mocks de ciertas cosas
  - Un ejemplo muy claro lo tengo en mi proyecto de [Minimal APIs](https://github.com/isaacOjeda/MinimalApiArchitecture): Aquí uso SQL Server cuando corro la aplicación normal, pero en el **TestServer** uso SQL Server Local Db (archivos .mdf en lugar de una base de datos en un servidor). Sin problema lo puedo ejecutar en GitHub Actions o Azure DevOps para efectuar pruebas de integración en bases de datos reales.


### IntegrationTests -> TestBase
`TestBase` nos ayudará a preparar nuestras pruebas de una forma más práctica y reutilizable.

Para poder realizar pruebas en nuestro proyecto debemos de poder hacer lo siguiente:
- Hacer llamadas autenticadas como un usuario en particular
- Queremos probar usuarios autenticados y usuarios anónimos
- Queremos probar operaciones con usuarios autenticados, pero con permisos (roles) limitados o roles completos
- Queremos poder verificar las operaciones (ejem. Si creamos un producto, debemos poder verificar si ese producto **sí** se creó en la base de datos)
- Debemos de poder probar la autenticación y usar JWTs válidos para las pruebas (lo mencionado anteriormente)
- Debemos de poder mantener un estado válido en la base de datos en cada prueba. Es decir, queremos una base de datos limpia por cada prueba, ya que cada prueba tendrá su propio caso de uso
- En ciertas pruebas vamos a necesitar que exista cierta información y en otras no.

Dentro de este proyecto de NUnit, crearemos la clase `TestBase` que nos servirá para lo mencionado anteriormente:

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

    /// <summary>
    /// Crea un usuario de prueba según los parámetros
    /// </summary>
    /// <returns></returns>
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

    /// <summary>
    /// Al terminar cada prueba, se resetea la base de datos
    /// </summary>
    /// <returns></returns>
    [TearDown]
    public async Task Down()
    {
        await ResetState();
    }

    /// <summary>
    /// Crea un HttpClient incluyendo un JWT válido con usuario Admin
    /// </summary>
    public Task<(HttpClient Client, string UserId)> GetClientAsAdmin() =>
        CreateTestUser("user@admin.com", "Pass.W0rd", new string[] { "Admin" });

    /// <summary>
    /// Crea un HttpClient incluyendo un JWT válido con usuario default
    /// </summary>
    public Task<(HttpClient Client, string UserId)> GetClientAsDefaultUserAsync() =>
        CreateTestUser("user@normal.com", "Pass.W0rd", Array.Empty<string>());

    /// <summary>
    /// Libera recursos al terminar todas las pruebas
    /// </summary>
    [OneTimeTearDown]
    public void RunAfterAnyTests()
    {
        Application.Dispose();
    }

    /// <summary>
    /// Inicializa la API y la BD antes de iniciar las pruebas
    /// </summary>
    [OneTimeSetUp]
    public void RunBeforeAnyTests()
    {
        Application = new ApiWebApplicationFactory();

        EnsureDatabase();
    }

    /// <summary>
    /// Shortcut para ejecutar IRequests con el Mediador
    /// </summary>
    public async Task<TResponse> SendAsync<TResponse>(IRequest<TResponse> request)
    {
        using var scope = Application.Services.CreateScope();

        var mediator = scope.ServiceProvider.GetRequiredService<ISender>();

        return await mediator.Send(request);
    }

    /// <summary>
    /// Shortcut para agregar Entities a la BD
    /// </summary>
    protected async Task<TEntity> AddAsync<TEntity>(TEntity entity) where TEntity : class
    {
        using var scope = Application.Services.CreateScope();

        var context = scope.ServiceProvider.GetService<MyAppDbContext>();

        context.Add(entity);

        await context.SaveChangesAsync();

        return entity;
    }

    /// <summary>
    /// Shortcut para buscar entities por primary key
    /// </summary>
    protected async Task<TEntity> FindAsync<TEntity>(params object[] keyValues) where TEntity : class
    {
        using var scope = Application.Services.CreateScope();

        var context = scope.ServiceProvider.GetService<MyAppDbContext>();

        return await context.FindAsync<TEntity>(keyValues);
    }

    /// <summary>
    /// Shortcut para buscar entities según un criterio
    /// </summary>
    protected async Task<TEntity> FindAsync<TEntity>(Expression<Func<TEntity, bool>> predicate) where TEntity : class
    {
        using var scope = Application.Services.CreateScope();

        var context = scope.ServiceProvider.GetService<MyAppDbContext>();

        return await context.Set<TEntity>().FirstOrDefaultAsync(predicate);
    }

    /// <summary>
    /// Se asegura de crear la BD
    /// </summary>
    private void EnsureDatabase()
    {
        using var scope = Application.Services.CreateScope();
        var context = scope.ServiceProvider.GetService<MyAppDbContext>();

        context.Database.EnsureCreated();
    }

    /// <summary>
    /// Shortcut para autenticar un usuario para pruebas
    /// </summary>
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
    /// <summary>
    /// Se asegura de limpiar la BD
    /// </summary>
    /// <returns></returns>
    private async Task ResetState()
    {
        using var scope = Application.Services.CreateScope();
        var context = scope.ServiceProvider.GetService<MyAppDbContext>();
		// Aquí en un caso real, podemos usar Respawn para reiniciar la BD en lugar
		// de eliminar tablas y recrearlas
        context.Database.EnsureDeleted();
        context.Database.EnsureCreated();

        await MyAppDbContextSeed.SeedDataAsync(context);
    }
}
```

Para un code snippet sí es bastante código, pero en cada método agregué la descripción de lo que hace.

**Como siempre, es una idea general de como facilitar las pruebas, si ves que estoy haciendo una burrada, por favor dime.**

> Nota 👀: `MyAppDbContextSeed` es una clase que agregué en persistencia para tener un solo lugar de datos de prueba. Visita el repositorio para más información.
>  

### IntegrationTests -> Features -> Products -> GetProductsQueryTests

Ya tenemos la _infraestructura_ para hacer pruebas, lo anterior solo lo hacemos una vez y ya, servirá para todas las pruebas a futuro que hagamos.

Como ejemplo, veremos como probar el Query `GetProductsQuery`. Esta prueba necesita de un usuario autenticado, veremos cómo se hace:
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
        // Arrenge
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
        // Arrenge
        var client = Application.CreateClient();

        // Act and Assert
        await FluentActions.Invoking(() =>
                client.GetFromJsonAsync<List<GetProductsQueryResponse>>("/api/products"))
                    .Should().ThrowAsync<HttpRequestException>();
    }
}
```

Aquí tenemos dos casos de uso para probar el Query. En el primer caso de uso es una consulta exitosa porque utilicé una llamada HTTP autenticada, la 2da es una llamada sin JWT, por lo que debe de ser rechazada.

Aquí `FluentAssertions` nos ayuda a comprobar bien fácil si la prueba es válida o no.

Hay muchas opiniones en la nomenclatura de las pruebas, pero la idea es que se describan solas, ya que al leer los reportes de testing solo veremos el nombre del método y nada más. Así que ser descriptivo en el método es buena idea para saber qué es lo que se está probando.

Ejemplo de cómo lo muestra VS y VS Code:
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/szrvlhyvuggziin0vvbw.png)
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ge11ra6vr3mygiwppinv.png)
 
### IntegrationTests -> Features -> Products -> CreateProductCommandTests
Como un ejemplo más veremos como probar la creación de productos (todas las pruebas que hice las puedes ver en el repositorio para una mejor idea)

Por lo que aquí tenemos más casos de uso:

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
        // Arrenge
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
        // Arrenge
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
        // Arrenge
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

Aquí estamos probando y validando que:
- Se puedan crear productos solo si el usuario tiene el rol **Admin** y los campos son correctos
- No se puedan crear productos, aunque sea usuario **Admin** si los campos son incorrectos
- No se puedan crear productos con usuario **Default** (sin permiso Admin)

En el primer test y en el _Assert_ hacemos algo muy interesante. Para poder verificar que un producto se guardó correctamente, lo que hacemos es que lo **volvemos** a consultar a la base de datos usando el shortcut `FindAsync` (que es un query directo al DbContext).

Al hacerlo de esta forma simplemente consultamos el producto que se supone que se acaba de crear y verificamos si se creó.

> Nota 👀: Aquí puede haber errores, porque puede haber productos con el mismo nombre. Al final, esto es un ejemplo y guía base.
> 

Todo lo que hicimos en `TestBase` lo podemos ir usando aquí, no necesariamente tiene que ser así, pero es una idea que a mí me ha funcionado.

Seguimos usando SQLite pero esto funciona perfectamente en un SQL Server y LocalDb para pruebas.

# Conclusión
Aprendimos qué son las pruebas de integración y como implementarlas en ASP.NET Core (.NET 6+). Crear un **TestServer** es extremadamente fácil ahora con clases como `WebApplicationFactory`.

Como siempre cualquier pregunta que tengas puedes contactarme en mi [twitter](https://twitter.com/balunatic).

También tengo un canal de [Youtube](https://www.youtube.com/channel/UCVWNtfrGkpMsUymOQzndBRQ).
# Referencias
- [IntegrationTest - Martin Fowler](https://martinfowler.com/bliki/IntegrationTest.html)
- [jasontaylordev/CleanArchitecture](https://github.com/jasontaylordev/CleanArchitecture)
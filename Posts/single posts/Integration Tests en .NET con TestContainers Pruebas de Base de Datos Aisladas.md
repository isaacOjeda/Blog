## Introducción a las Pruebas de Integración

En el desarrollo de software, las pruebas de integración son esenciales para verificar que los distintos módulos de una aplicación trabajen correctamente en conjunto. A diferencia de las pruebas unitarias, que se centran en unidades individuales de código, las pruebas de integración se enfocan en asegurarse de que los componentes del sistema interactúan adecuadamente entre sí y, en muchos casos, dependen de servicios externos, como bases de datos, servicios de autenticación o APIs de terceros.

En este artículo, vamos a trabajar con una Web API de ejemplo que utiliza SQL Server como base de datos. La idea es configurar un entorno de pruebas de integración utilizando [TestContainers](https://www.testcontainers.org/), una librería que permite levantar contenedores Docker de forma programática y en tiempo de ejecución, facilitando así la creación de entornos de prueba aislados y controlados.

Para ejecutar nuestras pruebas, usaremos **xUnit** como framework de testing y **Respawn** para resetear la base de datos entre pruebas, lo que ayuda a garantizar que cada prueba comience con un estado limpio y consistente. Gracias a TestContainers, podemos levantar un contenedor de SQL Server específicamente para nuestras pruebas, eliminando la dependencia de una base de datos local o compartida y asegurando que el entorno sea reproducible en cualquier máquina.

El código fuente completo de esta configuración y las pruebas está disponible en [este repositorio de GitHub](https://github.com/isaacOjeda/DevToPosts/tree/main/TestContainers). Si quieres seguir este tutorial, puedes clonar el repositorio y ajustar los pasos según sea necesario para tu propio proyecto.

## Herramientas y Librerías Utilizadas

Para implementar pruebas de integración con un entorno aislado y controlado, usaremos las siguientes librerías:

- **xUnit**: Un framework de pruebas en .NET, ideal para organizar y ejecutar nuestras pruebas de integración.
- **Testcontainers.MsSql**: Permite levantar un contenedor de SQL Server en Docker exclusivamente para nuestras pruebas, evitando dependencias locales y asegurando un entorno consistente.
- **Respawn**: Resetea la base de datos entre pruebas, manteniendo un estado limpio y libre de datos residuales para cada ejecución.
- **FluentAssertions**: Simplifica la escritura de aserciones, haciéndolas más legibles y expresivas.
- **Microsoft.AspNetCore.Mvc.Testing**: Utilizada para crear un servidor web en memoria para poder exponer los endpoints y poder ejecutar las pruebas de inicio a fin.

## Creación de un Contenedor SQL Server con TestContainers

En esta sección, configuraremos un contenedor de SQL Server para nuestras pruebas de integración usando **TestContainers** y un **ICollectionFixture** en xUnit. Esto permitirá que todas las pruebas dentro de una misma colección compartan una instancia de contenedor SQL Server, evitando configuraciones repetitivas y asegurando un entorno consistente.

### Uso de ICollectionFixture y CollectionDefinition en xUnit

xUnit proporciona el concepto de `ICollectionFixture` para compartir configuraciones entre pruebas en una misma colección. Con `ICollectionFixture`, podemos crear una configuración compartida que solo se inicializa una vez y está disponible para todas las pruebas que pertenecen a una colección específica. Esto es ideal cuando tenemos configuraciones costosas o que queremos evitar inicializar en cada prueba, como en este caso, un contenedor Docker para SQL Server.

La configuración se organiza en dos partes:

- **ICollectionFixture**: Define una clase de configuración compartida, en este caso `SqlServerContainerFixture`, que contiene toda la lógica para levantar y detener el contenedor SQL Server.
- **CollectionDefinition**: Agrupa las pruebas que comparten la misma configuración. Asociamos las pruebas a esta colección usando el nombre de la colección (en este caso, `"SqlServerContainerFixture"`).

### Creación del Contenedor SQL Server

A continuación, se muestra el código de `SqlServerContainerFixture`, que configura y administra el ciclo de vida de un contenedor SQL Server. Este fixture implementa `IAsyncLifetime`, que permite ejecutar lógica de inicialización y limpieza de recursos de forma asincrónica.

```csharp
public class SqlServerContainerFixture : IAsyncLifetime
{
    public const string FixtureName = "SqlServerContainerFixture";
    public MsSqlContainer SqlServerContainer { get; private set; }
    public string ConnectionString => SqlServerContainer.GetConnectionString();

    public SqlServerContainerFixture()
    {
        // Configuramos el contenedor SQL Server usando TestContainers
        SqlServerContainer = new MsSqlBuilder()
            .WithImage("mcr.microsoft.com/mssql/server:2022-latest") // Imagen de SQL Server
            .WithPassword("Passw0rd!") // Contraseña obligatoria para SQL Server
            .WithPortBinding(14330, 1433) // Asignación del puerto
            .Build();
    }

    // Inicializa el contenedor de forma asincrónica antes de que las pruebas comiencen a ejecutarse
    public async Task InitializeAsync()
    {
        await SqlServerContainer.StartAsync();
    }

    // Detiene y limpia el contenedor después de que todas las pruebas hayan finalizado
    public async Task DisposeAsync()
    {
        await SqlServerContainer.StopAsync();
    }
}
```

1. **Propiedades y Constructor**: 
   - `SqlServerContainer`: Es el contenedor de SQL Server configurado a través de la clase `MsSqlBuilder` de TestContainers, donde especificamos la imagen docker, la contraseña y el puerto para el contenedor.
   - `ConnectionString`: Proporciona una cadena de conexión a SQL Server, obtenida directamente del contenedor para que las pruebas puedan conectarse a esta base de datos.
   
2. **Async Lifecycle con IAsyncLifetime**:
   - `InitializeAsync()`: Este método se ejecuta antes de que cualquier prueba en la colección se ejecute. Llama a `StartAsync()` para levantar el contenedor de SQL Server.
   - `DisposeAsync()`: Este método se ejecuta una vez que todas las pruebas de la colección han finalizado, liberando recursos al detener el contenedor con `StopAsync()`.

#### Definición de la Colección de Pruebas

A continuación, definimos la colección de pruebas asociada al fixture `SqlServerContainerFixture`. Esto se hace con `[CollectionDefinition]`, que simplemente establece un nombre para la colección y vincula el fixture a todas las pruebas de la misma colección:

```csharp
[CollectionDefinition(SqlServerContainerFixture.FixtureName)]
public class DatabaseCollection : ICollectionFixture<SqlServerContainerFixture>
{
    // Esta clase no necesita código. Su único propósito es asociar
    // el CollectionDefinition con ICollectionFixture.
}
```

- `[CollectionDefinition]`: Asocia un nombre de colección, en este caso `"SqlServerContainerFixture"`, con el fixture `SqlServerContainerFixture`.
- `ICollectionFixture<SqlServerContainerFixture>`: Define la dependencia del fixture en todas las pruebas que pertenezcan a la colección.

Al agrupar nuestras pruebas en esta colección, cada una podrá acceder al contenedor de SQL Server compartido, eliminando la necesidad de inicializar un nuevo contenedor para cada prueba individualmente. Esto hace que las pruebas sean más eficientes y garantiza un entorno de datos consistente.

## Configuración de Reseteo de Base de Datos con Respawn y WebApplicationFactory

### Uso de `WebApplicationFactory`

**`WebApplicationFactory`** es una clase proporcionada por Microsoft que permite crear un servidor de pruebas en memoria basado en la configuración de la aplicación real. Esto es útil cuando necesitas probar la integración de tu Web API, ya que te permite simular el comportamiento de la aplicación sin necesidad de un servidor físico o configuración externa.

En este caso, estamos creando un **`WebApplicationFixture`** que extiende `WebApplicationFactory<Program>`. Esta clase personalizada actúa como un "wrapper" para iniciar la aplicación en un entorno de pruebas, y también maneja la configuración de la base de datos y su reseteo entre pruebas utilizando **Respawn**.

### Configuración de Respawn para el Reseteo de la Base de Datos

**Respawn** es una librería que facilita el reseteo rápido y eficiente de la base de datos, lo cual es especialmente útil en pruebas de integración. Lo que hace Respawn es eliminar los datos de las tablas entre pruebas, asegurando que cada prueba tenga un entorno limpio y consistente sin la necesidad de restaurar toda la base de datos.

En nuestro `WebApplicationFixture`, estamos utilizando **`IAsyncLifetime`** para manejar la inicialización y limpieza asincrónica de los recursos antes y después de que se ejecuten las pruebas. En particular, se usa para la configuración de Respawn:

```csharp
public class WebApplicationFixture(SqlServerContainerFixture sqlServerContainerFixture)
    : WebApplicationFactory<Program>, IAsyncLifetime
{
    private Respawner? _respawner;

    // Inicialización de Respawn y base de datos
    async Task IAsyncLifetime.InitializeAsync()
    {
        using var scope = Services.CreateScope();
        var dbSeed = scope.ServiceProvider.GetRequiredService<AppDbContextSeed>();

        // Inicializar Respawn
        _respawner = await Respawner.CreateAsync(sqlServerContainerFixture.ConnectionString, new RespawnerOptions
        {
            TablesToIgnore = ["__EFMigrationsHistory"]
        });

        // Asegurar que la base de datos esté creada y con datos de prueba
        await dbSeed.EnsureCreatedAsync();
        await dbSeed.SeedAsync();
    }

    // Reseteo de la base de datos después de las pruebas
    async Task IAsyncLifetime.DisposeAsync()
    {
        if (_respawner is null)
        {
            return;
        }

        // Reseteamos la base de datos a su estado inicial
        await _respawner.ResetAsync(sqlServerContainerFixture.ConnectionString);
    }
}
```

1. **`InitializeAsync()`**:
   - **`Respawner.CreateAsync()`**: Configuramos Respawn con la cadena de conexión del contenedor SQL Server para reiniciar las tablas relevantes antes de cada clase de pruebas. Las tablas especificadas en `TablesToIgnore` no se resetean.
   - **`dbSeed.EnsureCreatedAsync()` y `dbSeed.SeedAsync()`**: Aseguran que la base de datos esté creada y se tengan datos iniciales para las pruebas.
2. **`DisposeAsync()`**:
   - Cuando las pruebas de una clase han finalizado, **`DisposeAsync()`** es responsable de ejecutar el método **`ResetAsync()`** de Respawn, que limpia la base de datos, dejando las tablas listas para la siguiente clase de pruebas.

#### Configuración de la Base de Datos con `ConfigureWebHost`

Dentro de la clase **`WebApplicationFixture`**, también estamos utilizando el método **`ConfigureWebHost`** para ajustar la configuración de la aplicación de pruebas, especialmente la conexión a la base de datos. Esto es necesario porque la aplicación por defecto podría estar configurada para usar una base de datos local o un entorno diferente, por lo que tenemos que "reconfigurar" la cadena de conexión para que apunte al contenedor SQL Server levantado con TestContainers:

```csharp
protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    builder.UseEnvironment("IntegrationTests");
    builder.ConfigureTestServices(services =>
    {
        // Eliminar configuraciones anteriores de la base de datos
        var descriptor = services.SingleOrDefault(
            d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));

        services.Remove(descriptor!);

        var descriptorAppDbContext = services.SingleOrDefault(
            d => d.ServiceType == typeof(AppDbContext));

        services.Remove(descriptorAppDbContext!);

        // Configurar la base de datos para que utilice el contenedor SQL Server
        services.AddDbContext<AppDbContext>(options =>
        {
            options.UseSqlServer(sqlServerContainerFixture.ConnectionString);
        });
    });
}
```

1. **`UseEnvironment("IntegrationTests")`**: Establece el entorno de pruebas para que la aplicación cargue las configuraciones específicas para pruebas de integración (de ser necesario, podríamos tener un `appsettings.IntegrationTests.json`).
2. **`ConfigureTestServices()`**: Reconfigura los servicios de la aplicación para que la conexión de la base de datos apunte al contenedor SQL Server. Primero, eliminamos cualquier configuración previa de `DbContext` y luego agregamos la nueva configuración que utiliza la cadena de conexión proporcionada por el contenedor.

Con esta configuración, **`WebApplicationFixture`**:

- Levanta la aplicación en memoria utilizando **`WebApplicationFactory`** para simular el servidor real.
- Utiliza **Respawn** para reiniciar la base de datos antes de cada clase de pruebas, asegurando un entorno limpio.
- Configura la conexión a la base de datos para que apunte al contenedor SQL Server levantado con **TestContainers**.

## Escribir una Prueba de Integración Básica

Las pruebas de integración en este contexto validan la comunicación entre nuestra aplicación y la base de datos SQL Server. Usando `WebApplicationFixture` y `SqlServerContainerFixture`, simulamos el entorno de la aplicación y reiniciamos el estado de la base de datos para cada clase de pruebas, asegurando que cada prueba comience desde un estado limpio.

### Estructura de la Clase de Prueba `CreateProductTests`

La clase `CreateProductTests` contiene dos pruebas básicas para el endpoint de creación de productos:
- Una prueba para validar que la creación de un producto es exitosa.
- Una prueba para verificar el comportamiento cuando se intenta crear un producto con una categoría inexistente.

```csharp
namespace TestContainers.Integration.Tests.Endpoints.ProductTests;

[Collection(SqlServerContainerFixture.FixtureName)]
public class CreateProductTests : IClassFixture<WebApplicationFixture>
{
    private readonly WebApplicationFixture _factory;
    private readonly AppDbContext _dbContext;
    private readonly HttpClient _client;

    public CreateProductTests(WebApplicationFixture factory)
    {
        _factory = factory;
        _dbContext = factory.DbContext;
        _client = factory.HttpClient;
    }
```

- **[Collection]**: Esta anotación asegura que `SqlServerContainerFixture` se utilice para todas las pruebas dentro de esta clase, compartiendo la misma configuración de base de datos.
- **`IClassFixture<WebApplicationFixture>`**: `WebApplicationFixture` proporciona acceso a un cliente HTTP en memoria (`HttpClient`) para simular llamadas HTTP y a `AppDbContext` para realizar operaciones directas en la base de datos durante la configuración de pruebas.

#### Primera Prueba: Creación Exitosa de un Producto

La prueba `CreateProduct_ReturnsProduct` verifica que la creación de un producto con una categoría válida sea exitosa y que los datos devueltos sean correctos.

```csharp
[Fact]
public async Task CreateProduct_ReturnsProduct()
{
    // Arrange
    var category = await GetFirstCategory();
    var product = new Product
    {
        Name = "Test Product",
        Description = "Test Description",
        Price = 10.0m,
        CategoryId = category.Id
    };

    // Act
    var response = await _client.PostAsJsonAsync("/products", product);

    // Assert
    response.EnsureSuccessStatusCode();

    var createdProduct = await response.Content.ReadFromJsonAsync<Product>();

    createdProduct.ShouldNotBeNull();
    createdProduct.Name.ShouldBeEqualTo(product.Name);
    createdProduct.Price.ShouldBeEqualTo(product.Price);
}
```

1. **Arrange**: Se obtiene una categoría válida desde la base de datos (`GetFirstCategory`) y se crea un objeto `Product` con propiedades válidas.
2. **Act**: Se envía una solicitud HTTP `POST` al endpoint `/products` con los datos del producto. Esto simula una creación de producto en el sistema.
3. **Assert**: Se verifica que la respuesta sea exitosa y que los datos del producto creado coincidan con los esperados.

#### Segunda Prueba: Creación Fallida de un Producto con Categoría Inexistente

La prueba `CreateProduct_Fails_WhenCategoryDoesNotExist` verifica que la creación de un producto falle cuando se especifica una categoría inválida.

```csharp
[Fact]
public async Task CreateProduct_Fails_WhenCategoryDoesNotExist()
{
    // Arrange
    var product = new Product
    {
        Name = "Test Product",
        Description = "Test Description",
        Price = 10.0m,
        CategoryId = 0 // Categoría inválida
    };

    // Act
    var response = await _client.PostAsJsonAsync("/products", product);

    // Assert
    response.StatusCode.ShouldBeEqualTo(HttpStatusCode.BadRequest);
}
```

1. **Arrange**: Se crea un objeto `Product` con una categoría inexistente (CategoryId = 0).
2. **Act**: Se envía una solicitud HTTP `POST` al endpoint `/products` con el producto, esperando un fallo.
3. **Assert**: Se verifica que el servidor responde con un estado `BadRequest` (400), indicando que la categoría especificada no existe.

#### Método Auxiliar `GetFirstCategory`

El método `GetFirstCategory` es una función auxiliar que facilita la recuperación de la primera categoría disponible en la base de datos. Este método permite a las pruebas obtener un `CategoryId` válido para productos.

```csharp
private async Task<Category> GetFirstCategory()
{
    return await _dbContext.Categories.FirstAsync();
}
```

Este enfoque de pruebas ayuda a mantener las pruebas de integración eficientes y confiables, verificando los comportamientos principales del endpoint de creación de productos y manejando casos tanto exitosos como de error.


## Cuando Usar Integration Tests o Unit Tests

Las pruebas de software se dividen principalmente en **pruebas unitarias** e **integración**, y cada tipo tiene su propósito y contexto de uso. La elección entre uno u otro depende del objetivo de la prueba y del nivel de aislamiento necesario. A continuación, te explicamos cuándo es adecuado utilizar cada tipo de prueba.

### Unit Tests: Aislando el Comportamiento de una Funcionalidad Específica

Las **pruebas unitarias** son el tipo más básico de pruebas. Están diseñadas para verificar el comportamiento de una unidad de código de manera aislada, sin depender de recursos externos como bases de datos, servicios web o sistemas de archivos. Este tipo de pruebas se centran en una pequeña parte del sistema, como un método o función individual, asegurando que se ejecute correctamente bajo diferentes condiciones.

**Cuándo usar Unit Tests**:
- Cuando deseas comprobar el comportamiento de una **función o método específico** en aislamiento.
- Cuando necesitas verificar que una **lógica de negocio interna** funciona correctamente.
- Para asegurar que el código cumple con los requisitos del **comportamiento esperado**.
- Cuando los cambios realizados no deberían afectar otras partes del sistema (por ejemplo, al refactorizar código o agregar nuevas características pequeñas).
  
**Ventajas**:
- **Rápidos de ejecutar**: debido a que son pequeños y no requieren recursos externos.
- **Aislamiento**: puedes asegurarte de que las dependencias externas no interfieran.
- **Fáciles de automatizar**: las pruebas unitarias suelen ser fáciles de configurar y ejecutar en una suite de pruebas automatizadas.

### Integration Tests: Validando la Interacción entre Componentes

Las **pruebas de integración** van un paso más allá, probando cómo diferentes unidades de código interactúan entre sí y con sistemas externos, como bases de datos, API, o sistemas de mensajería. Se aseguran de que los componentes del sistema funcionen juntos de la forma esperada.

**Cuándo usar Integration Tests**:
- Cuando necesitas verificar cómo los **módulos o servicios interactúan entre sí**.
- Si deseas probar la integración con **sistemas externos** como bases de datos, servicios web, o colas de mensajes.
- Para validar que los cambios no afecten negativamente la interacción de diferentes capas del sistema.
- Para garantizar que la **configuración de infraestructura** y las dependencias externas (como bases de datos) estén correctamente conectadas y funcionando.

**Ventajas**:
- **Cobertura completa de flujo**: Validan el flujo real de datos a través de los diferentes componentes del sistema, lo que aumenta la confianza en la funcionalidad de alto nivel.
- **Identificación de problemas de integración**: Detectan problemas en la forma en que los componentes interactúan entre sí, lo cual no se podría verificar con pruebas unitarias.
- **Reducción de errores en producción**: Aseguran que el sistema se comporte correctamente en un entorno más cercano al real.

#### ¿Unit Tests o Integration Tests? ¿Cuándo usar cada uno?

A menudo, la clave está en la **combinación adecuada** de ambos tipos de pruebas:

- **Unit Tests**: Son el primer paso en el ciclo de vida del desarrollo, permitiendo detectar errores rápidamente y de forma aislada.
- **Integration Tests**: Deben complementarse con las pruebas unitarias para garantizar que todos los componentes funcionen correctamente cuando se ensamblen.

Es decir:
- Si estás desarrollando una pequeña funcionalidad que no interactúa con sistemas externos, **prioriza las pruebas unitarias**.
- Si estás trabajando con interacción entre sistemas, bases de datos o servicios externos, **añade pruebas de integración** para validar estos comportamientos más complejos.

Ambos tipos de pruebas son fundamentales para una estrategia de pruebas completa, y saber cuándo usar cada uno te ayudará a mantener una buena cobertura de pruebas y mejorar la calidad de tu software.
## Consideraciones para Escenarios Complejos

Cuando trabajamos con escenarios más allá de las simples operaciones CRUD, es crucial que nuestras pruebas de integración sean flexibles y realistas. A continuación, algunos puntos a considerar:

#### Datos de Configuración Inicial (Seed Data)

En escenarios complejos, es probable que necesites un conjunto de datos predefinido para representar un estado inicial específico de la base de datos. Utilizar datos de configuración inicial (o "seed data") permite que las pruebas se ejecuten en condiciones que simulan situaciones de negocio más avanzadas.

- **Estrategia de Configuración**: Puedes agregar datos de configuración en la fase de inicialización del `WebApplicationFixture`. Esto permite que los datos de prueba específicos estén listos cada vez que se ejecuten las pruebas.
- **Uso de Factories o Builders**: Para configurar datos complejos, considera el uso de patrones como `Factory` o `Builder` que ayuden a crear objetos con propiedades adecuadas para el escenario, manteniendo el código de prueba más limpio y reutilizable.

#### Simulación de Estados de la Aplicación

Algunas pruebas pueden requerir que la aplicación esté en un estado específico antes de ejecutar el test (como el "estado de procesamiento" o "finalizado" de una orden). Para simular estos estados:

- **Configuración de Estados con Estados Mock o Flags**: Configura los estados directamente en las entidades de prueba, o utiliza servicios mock (como mocks de notificaciones, correos electrónicos o servicios externos) para simular estados sin requerir que la aplicación ejecute todos los pasos.
- **Test Doubles**: Para escenarios que dependen de servicios externos, considera el uso de "doubles" (dummies, mocks, etc.) que representen estos servicios sin necesidad de hacer llamadas reales. Esto ayuda a centrar la prueba en la lógica de negocio sin complicaciones adicionales.

#### Verificación de Efectos Colaterales

Cuando las pruebas de integración implican actualizaciones de datos o envío de eventos a otros sistemas, es importante verificar que los efectos colaterales ocurren como se espera.

- **Simulación de Eventos**: Para los sistemas basados en eventos, puedes verificar que los eventos se envíen correctamente o incluso simular la recepción de eventos que la aplicación debe procesar.
- **Validación de Estados Finales**: Después de realizar una operación, verifica que las tablas relevantes y los estados de las entidades reflejen los cambios esperados.

Los escenarios complejos pueden agregar valor significativo a las pruebas de integración, ayudando a identificar problemas difíciles de reproducir en condiciones normales. Aplicar una configuración cuidadosa, incluyendo datos de configuración inicial, control de transacciones, y mocks de estados y dependencias, permitirá manejar estos escenarios de forma efectiva y aumentar la robustez de las pruebas de integración.

## Conclusión

Las pruebas de integración son esenciales para garantizar que los componentes de una aplicación funcionen de manera coherente y sin problemas cuando se combinan. A lo largo de este tutorial, hemos visto cómo configurar un entorno de pruebas sólido y flexible utilizando herramientas como **xUnit**, **Testcontainers**, **Respawn**, y **WebApplicationFactory** en .NET, permitiendo realizar pruebas con una base de datos SQL Server en contenedor y manteniendo un servidor de pruebas en memoria.

Al seguir las prácticas descritas, desde el uso de contenedores hasta la optimización de la infraestructura de pruebas, puedes asegurar que tu aplicación se mantenga estable a lo largo del ciclo de desarrollo. Este enfoque no solo minimiza errores que solo podrían detectarse en producción, sino que también facilita la identificación y resolución de problemas antes de que escalen.

Recuerda que, aunque estas pruebas pueden ser más costosas en términos de tiempo de ejecución y configuración que las pruebas unitarias, la inversión en pruebas de integración robustas proporciona un valor significativo. Estas pruebas ayudan a prevenir fallos graves y aseguran una experiencia de usuario final consistente. Adoptar buenas prácticas de optimización y aprovechar herramientas modernas para la gestión de entornos de prueba te permitirá mantener una suite de pruebas rápida y eficiente, fortaleciendo la confiabilidad de tus aplicaciones en el tiempo.

## Referencias

- [WebApplicationFactory Class (Microsoft.AspNetCore.Mvc.Testing) | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.testing.webapplicationfactory-1?view=aspnetcore-8.0)
- [Integration tests in ASP.NET Core | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-8.0)
- [Shared Context between Tests > xUnit.net](https://xunit.net/docs/shared-context#assembly-fixture)
- [Microsoft SQL Server - Testcontainers for .NET](https://dotnet.testcontainers.org/modules/mssql/)

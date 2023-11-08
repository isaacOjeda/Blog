# Introducción

En esta publicación, exploraremos un tema del que ya hemos hablado en ocasiones anteriores, pero esta vez queremos profundizar más y dar inicio a una serie de artículos que nos permitirán explorar diferentes patrones de diseño al crear servicios web.

Estamos hablando de CQRS, un patrón que se ha convertido en mi enfoque predeterminado para el diseño de sistemas en los últimos años. CQRS tiene sus ventajas y, hasta el momento, no he experimentado ningún inconveniente significativo con su uso.

Esperamos que este artículo te resulte útil. Como siempre, puedes encontrar el código relacionado en mi repositorio de GitHub, donde puedes acceder a este [código aquí](https://github.com/isaacOjeda/DevToPosts/tree/post-part1/MediatrValidationExample).

# ¿Qué es CQRS?

En [publicaciones anteriores](https://dev.to/isaacojeda/ddd-cqrs-aplicando-domain-events-en-aspnet-core-o6n), mencioné algunas razones por las cuales utilizar CQRS es una excelente idea, especialmente cuando estamos trabajando con bibliotecas como [MediatR](https://github.com/jbogard/MediatR). Aunque el enfoque de esa publicación era diferente, se conecta perfectamente porque MediatR nos brinda una gran facilidad para diversos aspectos del diseño de sistemas. En esta ocasión, vamos a repasar qué es CQRS.

**Command Query Responsibility Segregation**, o CQRS en sus siglas, es un patrón de diseño que ha ganado popularidad en los últimos años. La idea fundamental detrás de CQRS es dividir lógicamente el flujo de nuestra aplicación en dos corrientes principales:

- **Comandos (Commands)**: Estos son responsables de modificar el estado del dominio y no son idempotentes.
- **Consultas (Queries)**: Se encargan de obtener información del estado del dominio y representan operaciones idempotentes.

Si pensamos en un CRUD (Crear, Leer, Actualizar y Borrar), los comandos corresponden a las operaciones **Crear (Create)**, **Actualizar (Update)** y **Borrar (Delete)**, mientras que las consultas se relacionan con la operación **Leer (Read)**.

La siguiente imagen ilustra cómo funciona esta separación de responsabilidades:

![Descripción de la imagen](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1sljrzcv8vwe94id0k63.png)

Como se puede apreciar, la aplicación se divide en dos conceptos fundamentales: comandos (commands) y consultas (queries). Aunque la idea principal de CQRS también involucra dividir el almacén de datos en dos (uno maestro y otro replicado) para la lectura y la escritura, la noción de dividirlo de manera lógica funciona de manera eficiente en el diseño del sistema, incluso si se utiliza una única base de datos (aunque también es factible implementar el uso de bases de datos físicamente separadas).

## ¿Qué problema se intenta resolver?

El enfoque tradicional para diseñar aplicaciones en "n-capas" generalmente implica dividirlas en tres capas: Interfaz de usuario, Lógica de Negocio y Almacenamiento de Datos.

Al principio, esto puede no parecer un problema, pero surgen dificultades en cuanto al mantenimiento y la falta de flexibilidad para agregar nuevas características, depurar el código y otros desafíos.

En sistemas de "n-capas," a menudo terminamos con enormes repositorios que contienen todas las operaciones que se pueden realizar en una entidad. También solemos tener servicios que se vuelven cada vez más grandes con el tiempo.

La clave aquí es la segregación de responsabilidades en el mantenimiento de un sistema. Modificar una función no debería afectar a áreas completamente diferentes. Imagina tener una clase llamada `ProductosService` que contiene todas las operaciones relacionadas con los productos. Esto puede convertirse en un problema cuando el sistema crece, se unen nuevos miembros al equipo y la curva de aprendizaje es empinada. Cuando un desarrollador junior necesita modificar una función, es natural tener miedo de romper algo, ya que toda la funcionalidad está fuertemente acoplada en el servicio o repositorio.

La separación en Queries y Commands y, aún mejor, en [Vertical Slices](https://dev.to/isaacojeda/vertical-slice-architecture-36ng) (Características) permite mantener un código organizado. Agregar nuevas funcionalidades simplemente significa agregar más Queries o Commands en lugar de modificar servicios o repositorios gigantes.

Además, esta estructura facilita las pruebas. Un servicio puede tener dependencias para diversas operaciones sobre una entidad, lo que significa que necesitarás una serie de "mocks" para probar una función específica. En cambio, un Command solo incluye lo que necesita para funcionar, sin afectar a otras funcionalidades. Cada Command se encuentra encapsulado, y modificar uno no debería afectar a otros.

Por supuesto, es importante saber cuándo refactorizar. Si un Command realiza una tarea que también es realizada por otro Command, es hora de considerar otros patrones como Strategy o decoradores y realizar una refactorización. Además, es esencial encontrar un equilibrio entre no repetir código (DRY - Don't Repeat Yourself) y cumplir con el principio de Responsabilidad Única (aunque puede ser un desafío, con el tiempo te acostumbrarás).

## Patrón Mediador

El patrón del mediador se trata simplemente de definir un objeto que encapsula cómo otros objetos interactúan entre sí. En lugar de tener dos o más objetos que dependen directamente de otros objetos, estos objetos toman dependencia directa de un "mediador", y este mediador se encarga de gestionar las interacciones entre ellos:

![Descripción de la imagen](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nhcxnwlm1nh6e04tk4zg.png)

Como se muestra en el diagrama, `SomeService` envía un mensaje al mediador, y el mediador a su vez llama a otros servicios para que realicen acciones basadas en el mensaje recibido. `SomeService` no necesita saber nada acerca de los otros servicios que actúan según su solicitud; solo comunica al mediador lo que necesita que se haga.

La razón por la que el patrón del mediador es tan útil es la misma razón por la que utilizamos patrones como la [Inversión de Control (IoC)](https://code-maze.com/dependency-injection-aspnet/). Nos permite desacoplar por completo componentes, pero aún así permite que interactúen entre sí. Cuanto menos tenga que preocuparse un componente para funcionar, más sencillo será desarrollarlo, mantenerlo y probarlo.

## MediatR: Facilitando la Implementación de CQRS y el Patrón del Mediador

MediatR es una implementación del patrón mediador que ocurre completamente en el mismo proceso de la aplicación (_in-process_), y es una herramienta fundamental para crear sistemas basados en CQRS. Toda la comunicación entre el usuario y la capa de persistencia se gestiona a través de **MediatR**.

Es importante destacar que MediatR se ejecuta dentro del mismo proceso (_in-process_), lo que es una limitación clave. Dado que .NET maneja todas las interacciones entre objetos en el mismo proceso, MediatR no es apropiado si deseamos separar los Queries y Commands en aplicaciones distintas (es decir, si buscamos sistemas completamente independientes).

Para escenarios en los que se requiere esta separación, es preferible utilizar algún Message Broker, como se discutió en [una publicación anterior](https://dev.to/isaacojeda/integration-events-implementando-comunicacion-entre-servicios-con-masstransit-y-aspnet-13f0).

# Implementando CQRS en ASP.NET Core

La idea detrás de la implementación de CQRS en ASP.NET Core, específicamente en una Web API, es delegar la responsabilidad de procesar cada solicitud (*Request*) a un "Manejador" (*Handler*) en lugar de hacerlo en el controlador, como se mencionó anteriormente.

¿Por qué hacer esto? Hay varias razones, pero una de las más importantes es asegurarse de que todo el procesamiento de las solicitudes en la API no dependa directamente de los controladores. En su lugar, delegamos esta responsabilidad a una capa en la "Aplicación Central" (siguiendo los principios de Clean Architecture o Vertical Slices).

En .NET 7, es posible que incluso comencemos a utilizar Minimal APIs debido a la mejora de rendimiento. Los controladores ya no realizan tareas de procesamiento, solo reciben la solicitud y pueden hacer estos cambios sin inconvenientes.

Para implementar CQRS en ASP.NET Core utilizando MediatR (y como ejemplo, una base de datos SQLite), utilizaremos los siguientes paquetes en un proyecto Web API (puedes crearlo con `dotnet new webapi`):

```xml
<PackageReference Include="Swashbuckle.AspNetCore" Version="6.2.3" />
<PackageReference Include="FluentValidation.AspNetCore" Version="10.4.0" />
<PackageReference Include="MediatR.Extensions.Microsoft.DependencyInjection" Version="10.0.1" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="6.0.3" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="6.0.3">
   <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
   <PrivateAssets>all</PrivateAssets>
</PackageReference>
```

Puedes ignorar el código de ejemplo que viene con la plantilla (como las clases Weather, entre otras) y trabajar con la siguiente estructura en un solo proyecto (aunque a largo plazo, es recomendable considerar cómo dividir tus proyectos, ya sea en dos o más proyectos dentro de una misma solución, etc.):

```
Controllers/
Domain/
Features/
├─ Products/
Infrastructure/
├─ Persistence/
```

En este ejemplo, estamos siguiendo los conceptos típicos que usaríamos en una arquitectura limpia (Clean Architecture). No importa si todo se encuentra en un solo proyecto por ahora; con el tiempo, podrás tomar decisiones sobre cómo dividir tus proyectos (en uno o varios proyectos dentro de la misma solución, etc.).
## Domain

En esta sección, no hay mucho que explicar, ya que simplemente utilizaremos una clase `Product` para este ejemplo:

```csharp
namespace MediatrValidationExample.Domain;
public class Product
{
    public int ProductId { get; set; }
    public string Description { get; set; } = default!;
    public double Price { get; set; }
}
```

Es importante notar que aquí usamos el operador `default!` simplemente para inicializar un string con un valor predeterminado y decirle al compilador que nunca será `null`. Es importante mencionar que esto es técnicamente incorrecto, ya que el valor predeterminado de un string es `null`. Son detalles técnicos divertidos, ¡pero no siempre son precisos!
## Infrastructure → Persistence

Como de costumbre, vamos a utilizar Entity Framework Core para la persistencia. Aquí tienes el código para definir el contexto de la base de datos:

```csharp
using MediatrValidationExample.Domain;
using Microsoft.EntityFrameworkCore;

namespace MediatrValidationExample.Infrastructure.Persistence;
public class MyAppDbContext : DbContext
{
    public MyAppDbContext(DbContextOptions<MyAppDbContext> options) : base(options)
    { }

    public DbSet<Product> Products => Set<Product>();
}
```

Para crear la base de datos y su migración inicial, ejecutamos los siguientes comandos:

```bash
dotnet ef migrations add FirstMigration -o Infrastructure/Persistence/Migrations
```

```bash
dotnet ef database update
```

## Features → Products → Queries

Esta sección representa el **núcleo de la aplicación**, donde ubicaremos las **consultas** y **comandos** necesarios para la Web API. Empecemos con un ejemplo sencillo de cómo consultar productos.

La forma en que mostraremos cómo estructurar las consultas y comandos es una práctica que adopté recientemente de la Arquitectura de Vertical Slices. Si deseas obtener más información sobre el tema, también escribí [un artículo al respecto](https://dev.to/isaacojeda/vertical-slice-architecture-36ng).

En resumen, la idea es colocar todo lo necesario en un solo archivo (la solicitud, el manejador, los validadores, los mapeadores, los modelos, etc.). Como menciono en el artículo, si es necesario, se puede refactorizar (aunque es otro tema, pero queda a tu criterio cómo hacerlo).

```csharp
using MediatR;
using MediatrValidationExample.Infrastructure.Persistence;

namespace MediatrValidationExample.Features.Products.Queries;

public class GetProductQuery : IRequest<GetProductQueryResponse>
{
    public int ProductId { get; set; }
}

public class GetProductQueryHandler : IRequestHandler<GetProductQuery, GetProductQueryResponse>
{
    private readonly MyAppDbContext _context;

    public GetProductQueryHandler(MyAppDbContext context)
    {
        _context = context;
    }
    public async Task<GetProductQueryResponse> Handle(GetProductQuery request, CancellationToken cancellationToken)
    {
        var product = await _context.Products.FindAsync(request.ProductId);

        return new GetProductQueryResponse
        {
            Description = product.Description,
            ProductId = product.ProductId,
            Price = product.Price
        };
    }
}

public class GetProductQueryResponse
{
    public int ProductId { get; set; }
    public string Description { get set; } = default!;
    public double Price { get; set; }
}
```

Lo más importante aquí es prestar atención a las interfaces `IRequest<T>` y `IRequestHandler<T>`.

`IRequest<T>` es el mensaje que especifica la tarea a realizar, solicitado por **SomeService** y dirigido a uno o más **manejadores** (como se muestra en la imagen anterior).

En otras palabras, el mediador tomará la `IRequest<T>` y la enviará a los manejadores registrados. Estos manejadores saben qué mensajes pueden recibir y cómo llevar a cabo la tarea.

En este caso, `GetProductQuery` es una `IRequest<T>` que representa la búsqueda de un producto. `IRequest<T>` incluye un tipo genérico para especificar el tipo de objeto que se devolverá, ya que, en este caso, estamos realizando una consulta y obteniendo el estado del dominio.

En otros tiempos, habríamos creado un `ProductsService` o un `ProductsRepository` con un método `GetById`. Sin embargo, en este enfoque, la clase representa la operación a realizar, en lugar de un método adicional en una clase con múltiples métodos.

Esto es lo que me encanta de este patrón; sí, tendremos muchos archivos y carpetas, pero serán archivos pequeños y fáciles de buscar gracias a los potentes editores de texto y entornos de desarrollo integrados (IDEs).

El `GetProductQueryHandler` es el manejador del mismo `Query` que definimos anteriormente. Dado que ambos están en el mismo archivo, podríamos decir que el `Request` y el `Handler` están acoplados entre sí, pero aislados del resto del código. Agregar funcionalidad o probarla simplemente implica trabajar con lo que se encuentra en este archivo y nada más.

```csharp
using MediatR;
using MediatrValidationExample.Infrastructure.Persistence;
using Microsoft.EntityFrameworkCore;

namespace MediatrValidationExample.Features.Products.Queries;

public class GetProductsQuery : IRequest<List<GetProductsQueryResponse>>
{
}

public class GetProductsQueryHandler : IRequestHandler<GetProductsQuery, List<GetProductsQueryResponse>>
{
    private readonly MyAppDbContext _context;

    public GetProductsQueryHandler(MyAppDbContext context)
    {
        _context = context;
    }

    public Task<List<GetProductsQueryResponse>> Handle(GetProductsQuery request, CancellationToken cancellationToken) =>
        _context.Products
            .AsNoTracking()
            .Select(s => new GetProductsQueryResponse
            {
                ProductId = s.ProductId,
                Description = s.Description,
                Price = s.Price
            })
            .ToListAsync();
}

public class GetProductsQueryResponse
{
    public int ProductId { get; set; }
    public string Description { get; set; } = default!;
    public double Price { get; set; }
}
```

En este otro ejemplo, la interfaz `IRequest<T>` está vacía, pero si quisiéramos buscar productos, agregar paginación, ordenación, etc., todo se haría en esta clase `GetProductsQuery`, ya que representa la solicitud que recibe la API (lo veremos en el controlador).

Todos los `Queries` deben incluir el método `AsNoTracking`, ya que se tratan de consultas y no necesitan actualizar el estado de las entidades.

## Features → Products → Commands

Los comandos son donde finalmente se actualizarán las entidades. En publicaciones posteriores, mostraré cómo agregar validaciones, decoradores y otras funcionalidades que son fáciles de implementar gracias a otras bibliotecas como **FluentValidation** y **MediatR**, que ya estamos utilizando.

```csharp
using MediatR;
using MediatrValidationExample.Domain;
using MediatrValidationExample.Infrastructure.Persistence;

namespace MediatrValidationExample.Features.Products.Commands;

public class CreateProductCommand : IRequest
{
    public string Description { get; set; } = default!;
    public double Price { get; set; }
}

public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand>
{
    private readonly MyAppDbContext _context;

    public CreateProductCommandHandler(MyAppDbContext context)
    {
        _context = context;
    }

    public async Task<Unit> Handle(CreateProductCommand request, CancellationToken cancellationToken)
    {
        var newProduct = new Product
        {
            Description = request.Description,
            Price = request.Price
        };

        _context.Products.Add(newProduct);

        await _context.SaveChangesAsync();

        return Unit.Value;
    }
}
```

Aquí, la única información que necesitamos del request son el nombre del producto que deseamos registrar y su precio. Seguimos utilizando la interfaz de MediatR `IRequest`, aunque en este caso no es necesario un tipo genérico, ya que los comandos generalmente no devuelven información.

## Controladores

Dentro de los controladores, finalmente utilizaremos el mediador. Así es como se ve en la práctica:

```csharp
using MediatR;
using MediatrValidationExample.Features.Products.Commands;
using MediatrValidationExample.Features.Products.Queries;
using Microsoft.AspNetCore.Mvc;

namespace MediatrValidationExample.Controllers;

[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;

    public ProductsController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpGet]
    public Task<List<GetProductsQueryResponse>> GetProducts() => _mediator.Send(new GetProductsQuery());

    [HttpPost]
    public async Task<IActionResult> CreateProduct([FromBody] CreateProductCommand command)
    {
        await _mediator.Send(command);

        return Ok();
    }

    [HttpGet("{ProductId}")]
    public Task<GetProductQueryResponse> GetProductById([FromRoute] GetProductQuery query) =>
        _mediator.Send(query);
}
```

A través de la inyección de dependencias, solicitamos el mediador utilizando la interfaz `IMediator`. Una vez que inicializamos la solicitud `IRequest` correspondiente, simplemente la enviamos al mediador, que se encargará de determinar qué manejadores deben ejecutar la solicitud.

En `CreateProduct`, la `IRequest` (también conocida como comando) se recibe desde el cuerpo de la solicitud (como una clase POCO), lo que permite recibir y serializarla sin ningún problema.

En `GetProductById`, la `IRequest` (también conocida como consulta) se obtiene del segmento de la URL. Aquí es importante que el nombre en el segmento coincida con la propiedad correspondiente para que hagan juego.

En `GetProducts`, inicializamos la solicitud manualmente, ya que no estamos recibiendo ningún dato adicional en la solicitud, pero podría utilizarse `[FromQuery]` para recibir parámetros adicionales.

## Conclusión

Hemos aprendido cómo configurar CQRS utilizando MediatR en un proyecto de [ASP.NET](http://ASP.NET) Core Web API.

Vimos cómo encapsular cada funcionalidad de nuestra API en archivos individuales, cada uno representando una consulta o un comando.

Utilizar CQRS tiene sus ventajas, aunque podría tener desventajas. A medida que el sistema crece, cada miembro nuevo del equipo sin experiencia en este patrón tendrá que superar una curva de aprendizaje. Sin embargo, en última instancia, es para un bien mayor.

El diseño de sistemas mantenibles debe ser una meta para cada desarrollador y arquitecto de soluciones, ya que alguien en el futuro tendrá que mantener el sistema. Hacer que ese proceso sea menos doloroso es lo mejor que podemos hacer.

Esta división de conceptos nos ha ayudado mucho en los proyectos más recientes de mi equipo. Agregar funcionalidad o modificarla no debería ser una tarea complicada.

# Referencias

- [CQRS Validation Pipeline with MediatR and FluentValidation - Code Maze (code-maze.com)](https://code-maze.com/cqrs-mediatr-fluentvalidation/)
- [CQRS and MediatR in ASP.NET Core - Code Maze (code-maze.com)](https://code-maze.com/cqrs-mediatr-in-aspnet-core/)
- [Implementing the microservice application layer using the Web API | Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-application-layer-implementation-web-api)

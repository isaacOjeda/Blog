# Introducción

En el [artículo anterior](https://dev.to/isaacojeda/parte-1-cqrs-y-mediatr-implementando-cqrs-en-aspnet-56oe), exploramos cómo configurar y empezar a utilizar CQRS en un proyecto Web API en ASP.NET Core.

Hasta donde lo dejamos, la funcionalidad básica funciona bien, pero hay características que aún no hemos abordado y aprovechado. Mi objetivo con esta continuación es seguir desarrollando un proyecto bien establecido.

En este artículo, nos centraremos en agregar validación de solicitudes mediante [FluentValidation](https://fluentvalidation.net/). Esta biblioteca es altamente eficaz y, cuando se combina con **MediatR**, se vuelve aún más potente (aunque, en realidad, se puede utilizar sin MediatR sin problemas).

El código actualizado correspondiente a este artículo se encuentra disponible [aquí](https://github.com/isaacOjeda/DevToPosts/tree/post-part2/MediatrValidationExample) en el repositorio DevToPosts/MediatrValidationExample.
# Validando Solicitudes

MediatR nos brinda la capacidad de implementar el patrón decorador, al que se refiere como "Behaviours". 

Un "Behaviour" nos permite añadir comportamientos a nuestro pipeline de ejecución del mediador, lo que, en otras palabras, significa que antes de ejecutar el _handler_ específico del mediador para un `IRequest<T>`, podemos indicar que se realicen acciones adicionales, como la validación.

En este artículo, nos enfocaremos en agregar un comportamiento que valide las solicitudes del tipo `IRequest<T>` que se están a punto de ejecutar a través del mediador. Si se detectan errores de validación, detendremos la ejecución y generaremos una excepción.

## Validación de CreateProductCommand

Desde el artículo anterior, ya hemos incorporado las bibliotecas necesarias. Ahora, lo único que debemos hacer es agregar un validador al comando existente en `CreateProductCommand`:

```csharp
public class CreateProductValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductValidator()
    {
        RuleFor(r => r.Description).NotNull();
        RuleFor(r => r.Price).NotNull().GreaterThan(0);
    }
}
```

Como su nombre lo indica, el validador nos permite definir reglas de validación de manera "Fluent". Esto significa que podemos establecer las reglas de validación necesarias mediante funciones encadenadas.

Sin embargo, en este momento, estas reglas de validación no se aplicarán automáticamente. Debemos configurar MediatR para que realice esta validación en lugar de que sea responsabilidad del controlador. ¿Por qué? Porque deseamos diseñar nuestra aplicación de modo que no dependa de la capa de presentación, en este caso, la Web API. En general, solemos utilizar anotaciones de datos (incluso FluentValidation) para las validaciones que ocurren en el controlador. Aunque esto puede ser aceptable en ciertos casos, si trasladamos toda la lógica de negocio a la capa Application Core (MediatR), resulta más beneficioso.

La transición desde controladores de Web API a Minimal APIs se realizaría sin problemas en este escenario, ya que las validaciones de las solicitudes ocurren en la capa Application Core.

> Nota 💡: Aunque actualmente no estemos siguiendo una estructura de Clean Architecture en este proyecto, los conceptos y las ideas que estamos abordando son aplicables a cualquier tipo de proyecto.

## Excepciones Personalizadas

Para llevar a cabo la validación de distintos tipos, haremos uso de excepciones personalizadas, las cuales nos permitirán actuar de acuerdo al tipo de error que se produzca.

```csharp
using FluentValidation.Results;

namespace MediatrValidationExample.Exceptions;
public class ValidationException : Exception
{
    public ValidationException()
        : base("Se han producido uno o más errores de validación.")
    {
        Errors = new Dictionary<string, string[]>();
    }

    public ValidationException(IEnumerable<ValidationFailure> failures)
        : this()
    {
        Errors = failures
            .GroupBy(e => e.PropertyName, e => e.ErrorMessage)
            .ToDictionary(failureGroup => failureGroup.Key, failureGroup => failureGroup.ToArray());
    }

    public IDictionary<string, string[]> Errors { get; }
}
```

Esta excepción se utiliza para almacenar los errores de validación que pueden ocurrir al procesar una solicitud.

En esencia, mantenemos el estilo que ya ofrece la Web API con el uso del atributo `[ApiController]`. Sin embargo, dado que ahora estamos trabajando con MediatR, debemos implementarlo nosotros mismos.

> Nota 💡: Esto podría considerarse la primera desventaja de utilizar CQRS y MediatR: agrega complejidad. La Web API ya incorpora esta funcionalidad, pero al desear administrarla a través del mediador, debemos implementarla manualmente. La buena noticia es que esto solo se hace una vez y luego se aprovecha.

```csharp
namespace MediatrValidationExample.Exceptions;
public class NotFoundException : Exception
{
    public NotFoundException()
        : base()
    {
    }

    public NotFoundException(string message)
        : base(message)
    {
    }

    public NotFoundException(string message, Exception innerException)
        : base(message, innerException)
    {
    }

    public NotFoundException(string name, object key)
        : base($"No se encontró la entidad \"{name}\" ({key}).")
    {
    }
}
```

Esta excepción resulta útil cuando se busca una entidad y no se encuentra.

```csharp
namespace MediatrValidationExample.Exceptions;
public class ForbiddenAccessException : Exception
{
    public ForbiddenAccessException() : base() { }
}
```

Utilizamos esta excepción cuando se intenta eliminar algo (o realizar cualquier acción) y el usuario no tiene los permisos necesarios (spoiler: no la utilizaremos en este artículo).

## Utilizando NotFound en GetProductQuery

Ahora que hemos creado excepciones personalizadas, podemos empezar a utilizarlas. Un ejemplo claro es el uso de `NotFoundException` cuando se busca un producto por ID y este no existe. En ese caso, lanzaremos un error. Así que, actualizaremos lo siguiente en `GetProductQuery`:

```csharp
var product = await _context.Products.FindAsync(request.ProductId);

if (product is null)
{
   throw new NotFoundException(nameof(Product), request.ProductId);
   // Opción 2: throw new NotFoundException();
}

return new GetProductQueryResponse
{
    Description = product.Description,
    ProductId = product.ProductId,
    Price = product.Price
};
```

La intención aquí es proporcionar detalles en la respuesta de error sobre lo que se intentó buscar y con qué ID. Sin embargo, esto no es necesario en todos los casos.

## Comportamiento de Validación (patrón decorador)

Ahora, crearemos un decorador que envolverá cada ejecución del _handler_:

![Descripción de la imagen](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s6a0p78agackcp1usc9u.png)

Los Behaviors de MediatR funcionan de manera similar a cómo lo hace un Middleware en ASP.NET. Ejecutan alguna acción y luego delegan la ejecución al siguiente elemento en la cadena (esperando una respuesta).

Podemos utilizar decoradores para una variedad de tareas, y MediatR nos permite configurar los que necesitemos sin problemas.

![Descripción de la imagen](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pg2gbrglncze1cxk9ycl.png)

*Fuente: Vertical Slice Architecture por Jimmy Bogard*

Nuestro decorador se encargará de validar el `IRequest<T>` antes de que su _handler_ sea ejecutado. Si se detecta un problema de validación, lanzará la excepción correspondiente (`ValidationException`).

```csharp
using FluentValidation;
using MediatR;
using ValidationException = MediatrValidationExample.Exceptions.ValidationException;

namespace MediatrValidationExample.Behaviours;

public class ValidationBehaviour<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
     where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehaviour(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(TRequest request, CancellationToken cancellationToken, RequestHandlerDelegate<TResponse> next)
    {
        if (_validators.Any())
        {
            var context = new ValidationContext<TRequest>(request);

            var validationResults = await Task.WhenAll(
                _validators.Select(v =>
                    v.ValidateAsync(context, cancellationToken)));

            var failures = validationResults
                .Where(r => r.Errors.Any())
                .SelectMany(r => r.Errors)
                .ToList();

            if (failures.Any())
                throw new ValidationException(failures);
        }
        return await next();
    }
}
```

Lo que hace este decorador es recibir la solicitud (`Request`) y validarla con todos los validadores posibles de FluentValidation que se encuentren en el proyecto.

Como recordarás, al agregar un validador, heredamos de la clase `AbstractValidator<T>`. Estos validadores se registran en el contenedor de dependencias (lo que abordaremos más adelante) y simplemente validan la solicitud.

Si se detectan errores en la solicitud (según lo determine **FluentValidation**), se lanzará la excepción de validación.

Sin embargo, si dejamos todo como está, nuestra aplicación generará una Unhandled Exception (excepción no controlada), lo cual no es deseable.

Queremos poder manejar este tipo de excepciones personalizadas de manera adecuada para que se devuelva una respuesta acorde al tipo de excepción.

Para lograr esto, utilizaremos un Action Filter de MVC (sí, esta parte está acoplada a la presentación, pero podríamos utilizar un Unhandled Exception Middleware que se encargue de ello).

> Nota 💡: Si deseas obtener más información sobre MediatR y Behaviors, visita este enlace [Behaviors · jbogard/MediatR Wiki (github.com)](https://github.com/jbogard/MediatR/wiki/Behaviors)

## Filtro de Excepciones Personalizado

No queremos que estas excepciones personalizadas provoquen un fallo en la aplicación. En su lugar, deseamos que se manejen de acuerdo al tipo de excepción y que se devuelva un error siguiendo el estándar **Problem Details**.

```csharp
using MediatrValidationExample.Exceptions;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

namespace MediatrValidationExample.Filters;
public class ApiExceptionFilterAttribute : ExceptionFilterAttribute
{
    public override void OnException(ExceptionContext context)
    {
        switch (context.Exception)
        {
            case ValidationException validationEx:
                HandleValidationException(context, validationEx);
                break;
            case NotFoundException notFoundEx:
                HandleNotFoundException(context, notFoundEx);
                break;
            case ForbiddenAccessException:
                HandleForbiddenAccessException(context);
                break;
            default:
                HandleUnknownException(context);
                break;
        }

        base.OnException(context);
    }


    private void HandleValidationException(ExceptionContext context, ValidationException exception)
    {
        var details = new ValidationProblemDetails(exception.Errors)
        {
            Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1"
        };

        context.Result = new BadRequestObjectResult(details);

        context.ExceptionHandled = true;
    }

    private void HandleInvalidModelStateException(ExceptionContext context)
    {
        var details = new ValidationProblemDetails(context.ModelState)
        {
            Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1"
        };

        context.Result = new BadRequestObjectResult(details);

        context.ExceptionHandled = true;
    }

    private void HandleNotFoundException(ExceptionContext context, NotFoundException exception)
    {
        var details = new ProblemDetails()
        {
            Type = "https://tools.ietf.org/html/rfc7231#section-6.5.4",
            Title = "No se encontró el recurso especificado.",
            Detail = exception.Message
        };

        context.Result = new NotFoundObjectResult(details);

        context.ExceptionHandled = true;
    }

    private void HandleForbiddenAccessException(ExceptionContext context)
    {
        var details = new ProblemDetails
        {
            Status = StatusCodes.Status403Forbidden,
            Title = "Prohibido",
            Type = "https://tools.ietf.org/html/rfc7231#section-6.5.3"
        };

        context.Result = new ObjectResult(details)
        {
            StatusCode = StatusCodes.Status403Forbidden
        };

        context.ExceptionHandled = true;
    }

    private void HandleUnknownException(ExceptionContext context)
    {
        if (!context.ModelState.IsValid)
        {
            HandleInvalidModelStateException(context);
            return;
        }

        var details = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "Se produjo un error al proces

ar la solicitud.",
            Type = "https://tools.ietf.org/html/rfc7231#section-6.6.1"
        };

        context.Result = new ObjectResult(details)
        {
            StatusCode = StatusCodes.Status500InternalServerError
        };

        context.ExceptionHandled = true;
    }
}
```

## ¿Qué hace este Exception Filter?

Cuando ocurre un error no manejado en la API, este Exception Filter se activa. Su objetivo es devolver un detalle del problema siguiendo un estándar definido aquí: [RFC 7807 - Problem Details for HTTP APIs (ietf.org)](https://datatracker.ietf.org/doc/html/rfc7807).

No es necesario que leas el estándar, pero si te preguntas de dónde provienen los tipos de errores mostrados a continuación, se derivan de él.

Tenemos tres Exception Handlers para nuestras tres excepciones personalizadas:

- **HandleValidationException**: Devuelve un HTTP 400 indicando los errores encontrados en la solicitud

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "Se han producido uno o más errores de validación.",
  "status": 400,
  "errors": {
    "Price": [
      "'Price' debe ser mayor que '0'."
    ]
  }
}
```

- **HandleNotFoundException**: Devuelve un HTTP 404 para cualquier entidad que no se encuentre

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.4",
  "title": "No se encontró el recurso especificado.",
  "status": 404,
  "detail": "Entidad \"Product\" (2323) no se encontró."
}
```

- **HandleForbiddenAccessException**: Devuelve un HTTP 403 y se utiliza principalmente cuando una acción no está permitida para el usuario. En este caso no lo estamos usando, pero seguramente lo usarás en tu aplicación cuando haya usuarios y roles.

- **HandleUnknownException**: Siempre pueden ocurrir excepciones que realmente no sabemos qué son (como la maldición del **NullReferenceException**).

## Actualizando Program.cs

Para que todo lo que hemos hecho surta efecto, debemos registrar las dependencias de FluentValidation, el comportamiento de MediatR y el Exception Filter en el Service Collection de la API.

```csharp
// código omitido ...
builder.Services.AddControllers(options =>
    options.Filters.Add<ApiExceptionFilterAttribute>())
        .AddFluentValidation();
builder.Services.Configure<ApiBehaviorOptions>(options =>
    options.SuppressModelStateInvalidFilter = true);
builder.Services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehaviour<,>)); // Actualización
// ... código omitido
```

Modificamos la configuración predeterminada de `ApiBehaviorOptions` para que la validación automática que agrega `[ApiController]` ya no surta efecto, ya que queremos utilizar nuestra validación personalizada.

El Exception Filter se agrega de forma global al registrar los Controladores, aunque, sin problemas, también podemos agregarlo individualmente con `[ApiException]` en cada controlador (pero no es práctico).

Además, se registran todos los validadores que heredan de `AbstractValidator` y los servicios necesarios de FluentValidation.

De esta manera, deberías ser capaz de ejecutar la API y comenzar a realizar pruebas con productos que no existen o intentando crear uno que no cumple con las validaciones.

# Conclusión

A pesar de que estamos agregando complejidad a nuestra API, podemos estar seguros de que habrá beneficios en proyectos que crecen de manera indefinida.

Las validaciones de las solicitudes forman parte del Application Core y no deberían realizarse desde la capa de presentación. MediatR y FluentValidation realmente nos resuelven este problema. Aunque pueda parecer complicado configurar esto, solo se hace una vez y luego cualquier desarrollador puede usarlo simplemente empleando los `AbstractValidators`.

# Referencias

- [Behaviors · jbogard/MediatR Wiki (github.com)](https://github.com/jbogard/MediatR/wiki/Behaviors)
- [FluentValidation • Home](https://fluentvalidation.net/)
- [jasontaylordev/CleanArchitecture: Clean Architecture Solution Template for .NET 6 (github.com)](https://github.com/jasontaylordev/CleanArchitecture)
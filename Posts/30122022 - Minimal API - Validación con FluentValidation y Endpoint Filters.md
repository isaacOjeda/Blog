# Introducción

Minimal APIs sigue siendo un tema nuevo dentro de la comunidad .NET y creo que aún son pocos los que han usado Minimal APIs en un proyecto en producción.

Yo sin duda lo he estado usando, pero no en nada grande por ahora. Pero he estado explorando todas sus funcionalidades y por ahora lo que más me gusta es que realmente puedes estructurar tus proyectos de la forma que gustes.

Tengo un repositorio (In Progress 👷‍♂️) donde estoy haciendo varios tipos de proyectos con Minimal APIs (Source: [GitHub (isaacOjeda/MinimalApiExperiments)]([isaacOjeda/MinimalApiExperiments (github.com)](https://github.com/isaacOjeda/MinimalApiExperiments))) donde exploro un Clean Architecture, un Vertical Slice architecture y uno Plain (sin MediatR, puro Minimal API).

> Nota 💡: El código fuente de este post lo puedes encontrar [aquí](https://github.com/isaacOjeda/DevToPosts/tree/main/MinimalAPIFluentValidation/MinimalAPIFluentValidation)

El primer "pero" que encontré al estar haciendo la versión "Plain" (sin mis queridos decoradores de MediatR) fue que no había una forma "incluida" de cómo hacer validaciones sin tener que repetir el mismo código una y otra vez.

El `ModelState` tal cual no existe en Minimal API (o eso creo) y el `ModelState.IsValid` que solíamos usar en MVC pues ya no está disponible.

Damian Edwards (de los jefes de asp.net) hizo esta librería [MiniValidator](https://github.com/DamianEdwards/MiniValidation)) el cual tienes que hacer esto para validar:

```csharp
app.MapPost("/widgets/custom-validation", (WidgetWithCustomValidation widget) =>
    !MiniValidator.TryValidate(widget, out var errors)
        ? Results.ValidationProblem(errors)
        : Results.Created($"/widgets/{widget.Name}", widget));
```

`MiniValidator.TryValidate(widget, out var errors)` ejecuta la validación utilizando **DataAttributes** (como el `[Required]`) y regresa un `bool` indicando el resultado de la validación y con el parámetro de salida regresa los errores en caso de que existan.

> Hablando de performance, esto seguro es muy rápido, recuerda que las cosas "mágicas" (o sea, las cosas que usan reflection) pueden terminar siendo lentas, si lo que necesitas es high performance, debes de evitarte las formas "mágicas" de hacer las cosas (pero high performance de verdad), si no, eres como el 90% del resto como nosotros que usamos reflection al por mayor sin ningún problema.

Personalmente esto no me gusta, tener que validar en cada endpoint, ya que siento que es un "retroceso" comparado con Web API y lo que se agregó con `[ApiController]` en versiones pasadas. Por lo que he estado explorando como hacer esto sin repetir código y mejor aún, utilizando **FluentValidation**.

## Fluent Validation

FluentValidation ya es una librería muy popular, implementada en muchas plantillas que nos podemos encontrar en GitHub.

Esta librería me gusta usarla porque la validación se separa del modelo y puedes sin agregar "ruido" crear validaciones muy complejas.

Utilizaremos el paquete `FluentValidation.DependencyInjectionExtensions` que cuenta con extensiones para registrar los validadores en el contenedor de dependencias, no es necesario agregar `FluentValidation`, ya viene incluida en ese paquete.

Con `dotnet` o editando el **csproj** agregamos el paquete NuGet:

```xml
<PackageReference Include="FluentValidation.DependencyInjectionExtensions" Version="11.3.0" />
```

### Validando con Endpoint Filters

Los endpoints filters básicamente son decoradores, que nos permiten agregar comportamiento sobre el endpoint que se ejecutará. Es lo mismo que tenemos con los Action Filters de MVC.

En .NET 7 se agregaron varias funcionalidades que nos permitirán hacer esta tarea más fácil (Endpoint filters y Endpoint Groups).

> Nota 💡: Los Endpoint filters son un poco diferentes, la razón es porque son menos "mágicos" y favorecen el rendimiento, al igual que todo en Minimal APIs (en esencia, buscan ser AOT friendly).

Para poder hacer un mecanismo "automático" de validación, vamos a apoyarnos con un atributo que indicará cuando un parámetro de un Endpoint debe de ser validado.

```csharp
namespace MinimalAPIFluentValidation.Common.Attributes;

[AttributeUsage(AttributeTargets.Parameter, AllowMultiple = false)]
public class ValidateAttribute : Attribute
{
}
```

Esta clase solo será un "identificador", no tendrá implementado nada.

Posteriormente crearemos un Endpoint Filter Factory, donde "crearemos" un filter según se necesite (según el parámetro a validar).

```csharp
using FluentValidation;
using MinimalAPIFluentValidation.Common.Attributes;
using System.Net;
using System.Reflection;
 
namespace MinimalAPIFluentValidation.Common;
 
public static class ValidationFilter
{
    /// <summary>
    /// Filter Factory
    /// 
    /// Si en el Endpoint actual existe [Validator] y AbstractValidator asociados,
    /// se creará un delegate con el "Endpoint Filter"
    /// </summary>
    /// <param name="context"></param>
    /// <param name="next"></param>
    /// <returns></returns>
    public static EndpointFilterDelegate ValidationFilterFactory(EndpointFilterFactoryContext context, EndpointFilterDelegate next)
    {
        IEnumerable<ValidationDescriptor> validationDescriptors = GetValidators(context.MethodInfo, context.ApplicationServices);
 
        if (validationDescriptors.Any())
        {
            return invocationContext => Validate(validationDescriptors, invocationContext, next);
        }
 
        // dejar pasar
        return invocationContext => next(invocationContext);
    }
 
    /// <summary>
    /// Endpoint Filter que valida cualquier objeto con [Validate] y sus AbstractValidator
    /// </summary>
    /// <param name="validationDescriptors"></param>
    /// <param name="invocationContext"></param>
    /// <param name="next"></param>
    /// <returns></returns>
    private static async ValueTask<object?> Validate(IEnumerable<ValidationDescriptor> validationDescriptors, EndpointFilterInvocationContext invocationContext, EndpointFilterDelegate next)
    {
        foreach (ValidationDescriptor descriptor in validationDescriptors)
        {
            var argument = invocationContext.Arguments[descriptor.ArgumentIndex];
 
            if (argument is not null)
            {
                var validationResult = await descriptor.Validator.ValidateAsync(
                    new ValidationContext<object>(argument)
                );
 
                if (!validationResult.IsValid)
                {
                    return Results.ValidationProblem(validationResult.ToDictionary(),
                        statusCode: (int)HttpStatusCode.UnprocessableEntity);
                }
            }
        }
 
        return await next.Invoke(invocationContext);
    }
 
    /// <summary>
    /// Busca los validadores de cualquier clase en los parámetros
    /// que tenga el atributo [Validate]
    /// </summary>
    /// <param name="methodInfo"></param>
    /// <param name="serviceProvider"></param>
    /// <returns></returns>
    static IEnumerable<ValidationDescriptor> GetValidators(MethodInfo methodInfo, IServiceProvider serviceProvider)
    {
        ParameterInfo[] parameters = methodInfo.GetParameters();
 
        for (int i = 0; i < parameters.Length; i++)
        {
            ParameterInfo parameter = parameters[i];
 
            if (parameter.GetCustomAttribute<ValidateAttribute>() is not null)
            {
                Type validatorType = typeof(IValidator<>).MakeGenericType(parameter.ParameterType);
 
                // Note that FluentValidation validators needs to be registered as singleton
                IValidator? validator = serviceProvider.GetService(validatorType) as IValidator;
 
                if (validator is not null)
                {
                    yield return new ValidationDescriptor { ArgumentIndex = i, Validator = validator };
                }
            }
        }
    }
 
 
    private class ValidationDescriptor
    {
        public required int ArgumentIndex { get; init; }
        public required IValidator Validator { get; init; }
    }
}
```

Lo que sucede aquí:
- `ValidationFilterFactory`: Este método es el que se asociará con cada endpoint,  en esencia, con la presencia de un validador, creará un Endpoint Filter que ejecuta la validación. Si no hay validadores, sigue con la ejecución del endpoint sin afectar en nada.
- `Validate`: Según los validadores que se encontraron (si se encontraron), ejecutará la validación utilizando FluentValidation. Si existe un error de validación, regresará un `ValidationProblem`.
- `GetValidators`: El Filter Factory nos da una descripción del método (AKA el endpoint) que se va a ejecutar junto con sus parámetros.
	- Con `GetParameters` se consiguen todos los parámetros del endpoint y buscamos que alguno de estos tenga el atributo `[Validate]`, por lo cual significa que tendrá un `AbstractValidator` asociado, por lo que se buscará validar en el Filter.
	- Al confirmar que el parámetro tiene el atributo `[Validate]` procedemos a buscar los validadores asociados (registrados Singleton, ya que el Filter Factory se ejecuta de esta forma).
	- En la presencia de un validador, lo regresamos indicando el índice de este parámetro (lo necesitaremos más adelante)

> Nota 💡: Necesitamos el ArgumentIndex ya que la forma de acceder al "DTO" a validar, necesitamos indicar el índice en donde está colocado en nuestra función endpoint, esto es raro, pero por loa misma razón de performance  (quiero pensar) se tuvo que hacer así.

Y listo, es lo que necesitamos, ya podemos empezar a validar DTOs o modelos que sean recibidos en los endpoints.

### Cómo usar el Factory Filter

Hagamos un ejemplo de validación, utilizando el ejemplo de siempre. Creación de un producto:


```csharp
using FluentValidation;
using Microsoft.AspNetCore.Http.HttpResults;
using MinimalAPIFluentValidation.Common.Attributes;
 
namespace MinimalAPIFluentValidation.Features;
  
public class CreateProductCommand
{
    public double Price { get; set; }
    public string Description { get; set; } = default!;
    public int CategoryId { get; set; }
}
  
public static class CreateProductHandler
{
    public static Ok Handler([Validate] CreateProductCommand request, ILogger<CreateProductCommand> logger)
    {
        // TODO: Save Entity...
 
        logger.LogInformation("Saving {0}", request.Description);
 
        return TypedResults.Ok();
    }
}
 
public class CreateProductValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductValidator()
    {
        RuleFor(r => r.Description).NotEmpty();
        RuleFor(r => r.Price).GreaterThan(0);
        RuleFor(r => r.CategoryId).GreaterThan(0);
    }
}
```

Contamos con tres cosas aquí:
- Un DTO (AKA, Comando)
- El Handler del endpoint
- El Validador

El Handler básicamente es el endpoint, por lo que aquí indicamos que el `CreateProductCommand` se tiene que validar.

> Nota 💡: Estamos utilizando TypedResults, agregados recientemente en .NET 7

La intención es crear los Queries y Comandos que necesitemos para este feature "Product" (Revisa el ejemplo [Plain](https://github.com/isaacOjeda/MinimalApiExperiments/tree/main/PlainMinimalApi) de Minimal API Experiments para una mejor referencia) y agregarlos a un **Endpoint Group**.

Esto para decirle al grupo de endpoints que utilice el Factory Filter que acabamos de crear (entre otras cosas de Swagger).

```csharp
using MinimalAPIFluentValidation.Common;
 
namespace MinimalAPIFluentValidation.Features.Products;
 
public static class ProductsEndpoints
{
    public static RouteGroupBuilder MapProducts(this WebApplication app)
    {
        var group = app.MapGroup("api/products");
 
        group.MapPost("/", CreateProductHandler.Handler)
            .WithName("CreateProduct");
 
        // other endpoints here...
 
        group.WithTags(new string[] { "Products" });
        group.WithOpenApi();
 
        group.AddEndpointFilterFactory(ValidationFilter.ValidationFilterFactory);
 
        return group;
    }
}
```

La idea de crearlo así, es para no agregar los detalles de cada endpoint, simplemente es el registro de un grupo de endpoints relacionados.

Lo importante es la llamada `AddEndpointFilterFactory`, ya que esto ocurre en el grupo de endpoints, esto se aplicará a todo el grupo.

> Nota 💡: Ya sé ya sé, esto se convirtió en una especie controller 🤣

> Nota 2 💡: La idea de Minimal APIs, es organizarlo como tu gustes, ya que KISS (keep it simple stupid) y YAGNI (You aren't gonna need it).

> Nota 3 💡: No es importante que lo hagas así, lo importante es el uso de FluentValidation combinado con Endpoint Filters, la organización de esta forma sigue siendo opinión mia.

Por último, hacemos uso del grupo de endpoints en **Program.cs** y registramos todos los validadores que puedan existir en el proyecto:

```csharp
using FluentValidation;
using MinimalAPIFluentValidation.Features.Products;
 
var builder = WebApplication.CreateBuilder(args);
 
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddValidatorsFromAssemblyContaining<Program>(ServiceLifetime.Singleton); // <-- FluentValidation
 
var app = builder.Build();
 
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
 
app.UseHttpsRedirection();
 
app.MapProducts();  // <--- Group
 
app.Run();
```

## Probando la validación con Swagger

Si corremos ahora, se abrirá Swagger:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/893qngr6rf4mgbde61g6.png)

Y ya puedes confirmar que la validación esté entrando en vigor:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fso4q8v2tqixbsmt1e7n.png)

Y con validación positiva:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9f1wsylz0dn0l6ib8w6e.png)

# Conclusión
Esto puede ser una opción para validar de forma fácil tus endpoints en Minimal APIs. Me gustaría que fuera algo que ya viniera incluido, en realidad no sé si hay planes de agregar algo similar en versiones futuras de .NET, pero por ahora, esto puede ser una forma de hacerlo.

De igual forma, en el repositorio anteriormente mencionado, existen modos de validación utilizando decoradores de MediatR y Fluent Validation, si te interesa eres libre de ver el código y explorar por tu cuenta.

Por último, este tipo de cosas, son de esas que se arreglan descargando algún paquete de NuGet, ya que realmente es algo que ocurre "detrás de cámaras" y no nos afecta en el propósito de la aplicación que estamos realizando, pero es un feature técnico que se necesita.

# Referencias
- [Minimal API Validation with FluentValidation | Khalid Abuhakmeh](https://khalidabuhakmeh.com/minimal-api-validation-with-fluentvalidation)
- [Filters in Minimal API apps | Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/min-api-filters?view=aspnetcore-7.0#register-a-filter-using-an-endpoint-filter-factory)
- [DamianEdwards/MiniValidation: A minimalist validation library for .NET built atop the existing features in `System.ComponentModel.DataAnnotations` namespace (github.com)](https://github.com/DamianEdwards/MiniValidation)
- [Installation — FluentValidation documentation](https://docs.fluentvalidation.net/en/latest/installation.html)
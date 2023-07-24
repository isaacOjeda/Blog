# Introducción

En este post veremos de forma rápida el cómo crear un método de extensión para Entity Framework para poder crear ordenamiento de columnas de la base de datos de una forma dinámica según como el cliente de la API necesite el ordenamiento.

> Nota 💡: El código de este post lo encuentras [aquí]([isaacOjeda/AspNetCoreMediatRExample at parte-13 (github.com)](https://github.com/isaacOjeda/AspNetCoreMediatRExample/tree/parte-13))

# Dynamic sort con Linq Expressions

Algo que queremos evitar al dar la posibilidad de que se ordene por cualquier propiedad de nuestro modelo, es el deber tener un `switch/case` o `ifs` gigantes donde se evalúe cada posibilidad de ordenamiento, y esto repetirlo en cada endpoint donde queramos ofrecer ordenamiento.

No es bonito ni práctico, así que haremos uso de LINQ Expressions para lograr esta tarea. La idea es simplemente decir que propiedad queremos ordenar y bajo qué dirección (ascendente o descendente).

Linq expressions nos ayudan a crear expresiones lambda pero que son dinámicas en runtime, por lo que digamos, que partiendo de una expresión que parte de ella es generada con un string, se compila en runtime y genera el query de entity framework que queremos.

Para esto crearemos un método de extensión de `IQueryable<TEntity>` para poderlo usar como lo haríamos de forma habitual, pero en lugar de establecer una expresión lambda fuertemente tipada, le pasaremos un string que representa nuestra expresión de ordenamiento

## ApplicationCore > Common > Extensions

```csharp
using System.Linq.Expressions;
  
namespace MediatrExample.ApplicationCore.Common.Extensions;
  
public static class EFCoreExtensions
{
    public static IQueryable<TEntity> OrderBy<TEntity>(this IQueryable<TEntity> source, string orderByStrValues)
        where TEntity : class
    {
        var queryExpr = source.Expression;
        var command = orderByStrValues.ToUpper().EndsWith("DESC") ? "OrderByDescending" : "OrderBy";
        var propertyName = orderByStrValues.Split(' ')[0].Trim();
  
        var type = typeof(TEntity);
        var property = type.GetProperties()
            .Where(item => item.Name.ToLower() == propertyName.ToLower())
            .FirstOrDefault();
  
        if (property == null)
            return source;

        // p
        var parameter = Expression.Parameter(type, "p");
        // p.Price
        var propertyAccess = Expression.MakeMemberAccess(parameter, property);
        // p => p.Price
        var orderByExpression = Expression.Lambda(propertyAccess, parameter);

        // Ejem. final: .OrderByDescending(p => p.Price)
        queryExpr = Expression.Call(
            type: typeof(Queryable),
            methodName: command,
            typeArguments: new Type[] { type, property.PropertyType },
            queryExpr,
            Expression.Quote(orderByExpression));
  
        return source.Provider.CreateQuery<TEntity>(queryExpr); ;
    }

}
```

Aquí utilizamos la clase `Expression` que nos ayuda a crear expresiones lambdas de forma dinámica. Por ejemplo, una expresión lambda la podemos conocer de esta forma:

```csharp
context.Products.OrderBy(p => p.Price).Select(s => s.Description);
```

Donde `p => p.Price` se evaluará en runtime para generar al final un query, al igual que `s => s.Description`. Esto si lo hacemos en un `IQueryable` se traduce a un Query SQL según el provider que estemos usando.

Estas expresiones lambda son fuertemente tipadas, ya están definidas, ya se sabe que es lo que harán y con que propiedades, la idea aquí es generarlas de forma dinámica en runtime según los datos de entrada de nuestro endpoint. Cada llamada de `Expression` crea partes de la expresión final y terminan siendo unidas con `Expression.Call` anexándolo a la expresión del Query original (por que previamente pudimos haber llamado `.Where` p `.Select`).

Suponiendo que `orderByStrValues` es `"price desc"` (para ordenar de forma descendente según la propiedad `Price`), nos estaría generando la siguiente expresión lambda en runtime:

```csharp
query.OrderByDescending(p => p.Price)
```

Por lo que con `CreateQuery` ya lo convertimos en un `IQueryable` para que sea traducido a un query SQL al ser ejecutado (porque después de esta expresión, podemos también llamar otras expresiones).

Para poder usarlo, necesitamos recibir en nuestro endpoint el campo a ordenar y la dirección:

```csharp
public class GetProductsQuery : IRequest<List<GetProductsQueryResponse>>
{
    public string? SortDir { get; set; }
    public string? SortProperty { get; set; }
}
```

Y en el handler:

```csharp
using MediatrExample.ApplicationCore.Common.Extensions;

// código omitido...

    public Task<List<GetProductsQueryResponse>> Handle(GetProductsQuery request, CancellationToken cancellationToken) =>
        _context.Products
            .AsNoTracking()
            .OrderBy($"{request.SortProperty} {request.SortDir}")
            .ProjectTo<GetProductsQueryResponse>(_mapper.ConfigurationProvider)
            .ToListAsync();
```

> Nota 💡: Recuerden que este Query lo hicimos en las primeras partes de esta serie de posts, siempre puedes revisar el código [aquí]([isaacOjeda/AspNetCoreMediatRExample: ASP.NET Demo usado en mi blog https://dev.to/isaacOjeda (github.com)](https://github.com/isaacOjeda/AspNetCoreMediatRExample))

Y también actualizamos el controller:

```csharp
    /// <summary>
    /// Consulta los productos
    /// </summary>
    /// <returns></returns>
    [HttpGet]
    public Task<List<GetProductsQueryResponse>> GetProducts([FromQuery] GetProductsQuery query) =>
        _mediator.Send(query);
```

## Probando el Ordenamiento

La solución ya contiene Swagger, pero para evitar screenshots, con cualquier RestClient podemos probarlo:

```
### GET Products

GET {{host}}/api/products?sortDir=desc&sortProperty=price
Content-Type: application/json
Authorization: Bearer {{token}}
```

Resultado:
```json
[
  {
    "productId": "eQPDkwoYX31vMKGJ",
    "description": "Product 02",
    "price": 52200,
    "listDescription": "Product 02 - $52,200.00"
  },
  {
    "productId": "L1dwWxoz2omzN89g",
    "description": "Product 01",
    "price": 16000,
    "listDescription": "Product 01 - $16,000.00"
  }
]
```

Este método ya lo podrías usar en cualquier Entity, ya que se hace directamente en el `IQueryable` y si en SQL esa columna permite ser ordenada, sin problema se podrá hacer.

# Conclusión 
La creación de expresiones linq dinámicas o incluso el uso de reflection suele ser un tema difícil de asimilar, pero te invito a que depures el código y veas cómo funciona paso a paso. Intenta modificarlo y hacer otro tipo de expresiones dinámicas, relamente podrías hacer cualquier cosa.

Con este método de extension ya lo podrás usar siempre en cualquier query y en cualquier entity, por lo que te quita un super dolor de cabeza para soportar ordenamiento dinámico en tus proyectos.

# Introducción

En el post de hoy veremos algo muy sencillo pero muy útil a la hora de hacer Queries en Entity Framework: La paginación.

La paginación de la información realmente es algo necesario cuando hablamos de información frecuentemente accedida y que con el paso del tiempo va creciendo.

Según el motor de base de datos que utilices esta operación se puede llevar a cabo de distintas formas, pero la gran mayoría utiliza un motor SQL, esto debería de ser igual para todos y más si usamos un ORM como Entity Framework Core.

> Nota 💡: El código fuente siempre lo encuentras en mi perfil: [isaacOjeda/AspNetCoreMediatRExample at parte-14 (github.com)](https://github.com/isaacOjeda/AspNetCoreMediatRExample/tree/parte-14)

# Implementando Paginación

Realizar esta tarea es muy sencillo, empezaremos creando una clase que envolverá los resultados paginados:

```csharp
namespace MediatrExample.ApplicationCore.Common.Models;
  
public class PagedResult<T>
    where T: class
{
    public IEnumerable<T> Results { get; set; } = Enumerable.Empty<T>();
    public int RowsCount { get; set; }
    public int PageCount { get; set; }
    public int PageSize { get; set; }
    public int CurrentPage { get; set; }
}
```

Es útil saber lo siguiente:
- `RowsCount`: Tendrá cuantos resultados tiene el query antes de ser paginado
- `PageCount`: Nos dice cuántas páginas se pueden visitar según el query, esto es útil para tener en el UI un paginador y poder decir "Página 1 de 130"
- `PageSize`: Aunque este dato viene desde la solicitud, de igual forma me ha resultado útil regresarlo aquí, este no es obligatorio
- `CurrentPage`: La página actual que se solicitó por lo que `Results` vendrán los resultados de la página actual

Hacemos esta clase genérica para poderla utilizar con cualquier DTO y no tener que estar implementándolo en cada caso en particular.

Para seguir, crearemos un método de extensión para hacerlo más fácil al hacer los queries:

```csharp
public static class EFCoreExtensions
{
    public static async Task<PagedResult<TEntity>> GetPagedResultAsync<TEntity>(this IQueryable<TEntity> source, int pageSize, int currentPage)
        where TEntity: class
    {
        var rows = source.Count();
        var results = await source
            .Skip(pageSize * (currentPage - 1))
            .Take(pageSize)
            .ToListAsync();
  
        return new PagedResult<TEntity>
        {
            CurrentPage = currentPage,
            PageCount = (int)Math.Ceiling((double)rows / pageSize),
            PageSize = pageSize,
            Results = results,
            RowsCount = rows
        };
    }

   // código omitido...
}
```

Aquí estamos realizando dos queries, es necesario para saber cuántos registros son en total antes de paginar, este se puede cachear de alguna forma para evitar hacerlo siempre, pero por ahora lo dejamos así.

Lo que hace la paginación es el `Skip` que se brinca los "n" registros que se le indique y `Take` agarra los "n" registros restantes que quedan después de haber "skipeado". O sea, la paginación funciona de la siguiente manera;

Si `pageSize` es `10` y `currentPage` es `5` el query tendrá un `Skip` de `40` registros y siempre tendrá un `Take` de `10` ya que serán los elementos que queremos mostrar por página. Una forma de visualizarlo:
 
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xadnpklyjd39xivobljd.png)

Los elementos en gris fueron _skipeados_ (no se muestran todos, pero digamos que van del 1 al 40) y los elementos en verde fueron tomados con el `Take` para ser consultados, los que no tienen fondo, son elementos de la siguiente página, ya que solo estamos consultando de la base de datos de 10 en 10 (pero claro puedes elegir el tamaño de la página como lo desees).

Para utilizarlo, nos vamos al Query original que ya teníamos y hacemos uso del nuevo método de extensión que acabamos de agregar:

```csharp
public class GetProductsQuery : IRequest<PagedResult<GetProductsQueryResponse>>
{
    public string? SortDir { get; set; }
    public string? SortProperty { get; set; }

    public int CurrentPage { get; set; }   // NEW!!
    public int PageSize { get; set; }      // NEW!!
}

public class GetProductsQueryHandler : IRequestHandler<GetProductsQuery, PagedResult<GetProductsQueryResponse>>
{
    private readonly MyAppDbContext _context;
    private readonly IMapper _mapper;
  
    public GetProductsQueryHandler(MyAppDbContext context, IMapper mapper)
    {
        _context = context;
        _mapper = mapper;
    }
  
    public Task<PagedResult<GetProductsQueryResponse>> Handle(GetProductsQuery request, CancellationToken cancellationToken) =>
        _context.Products
            .AsNoTracking()
            .OrderBy($"{request.SortProperty} {request.SortDir}")
            .ProjectTo<GetProductsQueryResponse>(_mapper.ConfigurationProvider)
            .GetPagedResultAsync(request.PageSize, request.CurrentPage);  // NEW!!
}
```

`CurrentPage` y `PageSize` vendrán del cliente, ya que él es el que decide el tamaño de las páginas (pero por supuesto no es mala idea validar que siempre se haga la paginación y no de un tamaño tan exagerado, si el rendimiento es lo que estamos cuidando al hacer la paginación, también debemos de considerar eso).

El método `GetPagedResultAsync` es el que mandamos a llamar al final para terminar con el query (ya que podemos tener filtros, selects y otras cosas) y por lo tanto ahora nuestro query ahora regresa un `PagedResult` y no una colección de productos directamente.

> Nota 💡: Es importante que, al hacer paginación, siempre debe de existir un `OrderBy` ya que si no se hace eso podrían obtenerse resultados aleatorios en la paginación


También actualizamos el Controller para regresar un `PagedResult`:

```csharp
    /// <summary>
    /// Consulta los productos
    /// </summary>
    /// <returns></returns>
    [HttpGet]
    public Task<PagedResult<GetProductsQueryResponse>> GetProducts([FromQuery] GetProductsQuery query) =>
        _mediator.Send(query);
```


Y si hacemos pruebas con cualquier cliente HTTP:

```http
### GET Products
GET {{host}}/api/products?pageSize=5&currentPage=3&sortDir=desc&sortProperty=price
Content-Type: application/json
Authorization: Bearer {{token}}
```

> Nota 💡: El ordenamiento lo vimos en este [post]([[Parte 13] EF Core: Dynamic Sort con Linq Expressions - DEV Community](https://dev.to/isaacojeda/parte-13-ef-core-dynamic-sort-con-linq-expressi-nj9)) por si te interesa

Y el resultado:

```json
{
  "results": [
    {
      "productId": "JqpBAvow7oYVkaDb",
      "description": "Product 10",
      "price": 892,
      "listDescription": "Product 10 - $892.00"
    },
    {
      "productId": "eQPDkwoYXg31vMKG",
      "description": "Product 100",
      "price": 889,
      "listDescription": "Product 100 - $889.00"
    },
    {
      "productId": "Qz87dMy6n0ymlkX1",
      "description": "Product 93",
      "price": 875,
      "listDescription": "Product 93 - $875.00"
    },
    {
      "productId": "jNDdgVoEPD361nQ8",
      "description": "Product 92",
      "price": 855,
      "listDescription": "Product 92 - $855.00"
    },
    {
      "productId": "24gZjVOlrV3KELa6",
      "description": "Product 42",
      "price": 850,
      "listDescription": "Product 42 - $850.00"
    }
  ],
  "rowsCount": 100,
  "pageCount": 20,
  "pageSize": 5,
  "currentPage": 3
}
```

> Nota  💡:  Siempre revisa el código fuente (en [isaacOjeda/AspNetCoreMediatRExample at parte-14 (github.com)](https://github.com/isaacOjeda/AspNetCoreMediatRExample/tree/parte-14)) ya que en este ejemplo actualicé el método Seed para tener muchos productos de prueba en la base de datos

# Conclusión
Siempre debemos de incluir la páginación del lado del servidor, ya que hacerlo en el front-end aunque a veces es práctico, en un algún momento existirán problemas de rendimiento. Traer todo de un solo golpe de la base de datos puede ser costoso en tiempo de CPU y RAM en tu aplicación, por eso siempre te recomiendo que desde un inicio busques implementarlo en esos lugares donde estás seguro de que la información crecerá con el paso del tiempo.

Por ejemplo, cuando tenemos catálogos de configuración del sistema, donde si mucho existirán menos de 100 registros, tal vez ahí sí decidiría no implementar la paginación, ya que son listados poco accedidos y con poca información, pero cuando hablamos de listados frecuentemente consultados, es obligatorio implementar la paginación, ya que si no lo haces seguro te enfrentarás con problemas de rendimiento.

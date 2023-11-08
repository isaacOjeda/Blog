# Introducción

En esta serie de artículos, continuaremos explorando la implementación de CQRS con MediatR. Hasta ahora, hemos aprendido a utilizar CQRS y a validar solicitudes. En esta publicación, nos centraremos en cómo aprovechar AutoMapper para el mapeo de entidades.

El mapeo de objetos es una tarea común en el desarrollo de aplicaciones, especialmente cuando dividimos nuestra aplicación en áreas técnicas distintas. Los DTOs (Data Transfer Objects) se utilizan para mostrar información relevante en los puntos finales de la aplicación, y a menudo involucran múltiples entidades del dominio. Esto hace que sea importante tener la capacidad de mapear estos objetos de manera eficiente.

Con el enfoque de CQRS, trabajamos con un concepto simple: **Solicitud** -> **Manejador** -> **Respuesta**.

Por lo tanto, es común que necesitemos mapear una solicitud a una entidad de dominio cuando ejecutamos un comando. Además, también necesitamos mapear una entidad de dominio a una respuesta en las consultas. Esta tarea puede volverse repetitiva y monótona. Afortunadamente, existe una solución que simplifica este proceso: [AutoMapper](https://automapper.org/).

Si deseas acceder al código fuente actualizado mencionado en este artículo, lo encontrarás aquí: [MediatrValidationExample](https://github.com/isaacOjeda/DevToPosts/tree/post-part3/MediatrValidationExample).
# ¿Qué es AutoMapper?

AutoMapper es una herramienta que se utiliza para mapear objetos entre sí, es decir, copiar los datos de un objeto a otro de manera automática. Esto simplifica el proceso de asignación de propiedades de un objeto de un tipo (Type A) a otro objeto de un tipo diferente (Type B).

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/iebsymp5tofzo4wrz7ed.png)

Para ilustrarlo, consideremos un ejemplo con una lista de productos que hemos discutido en artículos anteriores. Tenemos la entidad `Product` y el DTO (Data Transfer Object) `GetProductsQueryResponse`.

```csharp
public class Product
{
    public int ProductId { get; set; }
    public string Description { get; set; } = default!;
    public double Price { get; set; }
}
```

```csharp
public class GetProductsQueryResponse
{
    public int ProductId { get; set; }
    public string Description { get; set; } = default!;
    public double Price { get; set; }
}
```

Ambas clases son casi idénticas en estructura, pero tienen propósitos diferentes. AutoMapper (y otros mapeadores similares) se utilizan para evitar la tarea tediosa de asignar manualmente las propiedades de una clase de Tipo A a una clase de Tipo B.

Este tipo de asignación es común y a menudo involucra clases con muchas propiedades. Automatizar este proceso nos ahorra tiempo y evita errores.

En el caso de `GetProductsQuery`, normalmente tendríamos que realizar un mapeo manual como este:

```csharp
.Select(s => new GetProductsQueryResponse
    {
        ProductId = s.ProductId,
        Description = s.Description,
        Price = s.Price
    })
```

La asignación repetitiva que vemos aquí es lo que AutoMapper nos ayuda a evitar, y aunque en este ejemplo sea sencillo (involucrando solo tres propiedades), en casos más complejos se vuelve aún más valioso.

## Agregando AutoMapper

Para empezar a usar AutoMapper en tu proyecto, necesitas agregar el siguiente paquete:

```bash
dotnet add package AutoMapper.Extensions.Microsoft.DependencyInjection
```

Aunque podrías instalar solamente `AutoMapper`, este paquete ya lo incluye y además facilita el registro de los mapeadores como dependencias en tu proyecto.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/75tviykluu0c4er3rjc2.png)

### Registrando AutoMapper

Para que todas las funcionalidades de AutoMapper funcionen correctamente, es importante registrar la configuración necesaria en el archivo Program. Esto se hace de la siguiente manera:

```csharp
// código...
builder.Services.AddAutoMapper(Assembly.GetExecutingAssembly());
// código...
```

De esta forma, se buscarán y registrarán automáticamente todos los perfiles de mapeo en tu proyecto, asegurando su funcionamiento correcto.

## Creando Perfiles de Mapeo

Un perfil (Profile) en AutoMapper describe cómo la herramienta debe mapear un objeto de un tipo a otro. A veces, el mapeo es automático, pero en otras ocasiones es necesario proporcionar información adicional para guiar a AutoMapper.

Siguiendo el ejemplo de `GetProductsQuery`, en el mismo archivo donde definimos este query, vamos a agregar la clase `GetProductsQueryProfile` que describirá el mapeo que deseamos realizar.

```csharp
public class GetProductsQueryProfile : Profile
{
    public GetProductsQueryProfile() =>
        CreateMap<Product, GetProductsQueryResponse>();
}
```

Este proceso es bastante sencillo. Aquí, utilizamos el método `CreateMap<TSource, TDestination>()`. Como su nombre indica, el primer parámetro es el tipo de origen y el segundo es el tipo de destino.

Cuando AutoMapper necesita realizar un mapeo, buscará todos los perfiles registrados. Si no encuentra un perfil adecuado, generará una excepción.

### IMapper y Consultas (Queries)

Para aplicar este mapeo en consultas, podemos hacerlo directamente desde Entity Framework al trabajar con `IQueryable<T>`. Esta es una forma eficaz de generar consultas a partir de un perfil de mapeo.

Así, actualizamos el manejador del query de la siguiente manera:

```csharp
public class GetProductsQueryHandler : IRequestHandler<GetProductsQuery, List<GetProductsQueryResponse>>
{
    private readonly MyAppDbContext _context;
    private readonly IMapper _mapper;

    public GetProductsQueryHandler(MyAppDbContext context, IMapper mapper)
    {
        _context = context;
        _mapper = mapper;
    }

    public Task<List<GetProductsQueryResponse>> Handle(GetProductsQuery request, CancellationToken cancellationToken) =>
        _context.Products
            .AsNoTracking()
            .ProjectTo<GetProductsQueryResponse>(_mapper.ConfigurationProvider)
            .ToListAsync();
}
```

Aquí, utilizamos la extensión `ProjectTo<TDestionation>()` para aplicar el mapeo según el origen. El origen es la entidad `Product` y el destino es `GetProductsQueryResponse`.
### Mapeo Explícito con ForMember

El perfil de mapeo que definimos anteriormente es un mapeo directo, pero en ocasiones necesitamos especificar cómo mapear una propiedad en particular. Esto puede ser necesario cuando las propiedades no tienen el mismo nombre o cuando su obtención es más compleja.

Para ilustrar este caso, supongamos que queremos agregar una nueva propiedad llamada `ListDescription` que combina el nombre y el precio en una sola propiedad. Aunque esto no tenga mucho sentido, es un ejemplo sencillo para mostrar cómo abordar este tipo de situaciones. Para agregar esta propiedad al objeto de respuesta, actualizamos el perfil de mapeo de la siguiente manera:

```csharp
public class GetProductsQueryProfile : Profile
{
    public GetProductsQueryProfile() =>
        CreateMap<Product, GetProductsQueryResponse>()
            .ForMember(dest =>
                dest.ListDescription,
                opt => opt.MapFrom(src => $"{src.Description} - {src.Price:c}"));
}
```

Utilizamos `ForMember` para especificar cómo mapear una propiedad en particular. En este caso, la nueva propiedad `ListDescription` se compone de dos propiedades existentes, incluso el precio se muestra con formato de moneda (Currency). Esto es algo que AutoMapper no puede deducir automáticamente, por lo que es útil tener la capacidad de definir mapeos específicos, especialmente cuando se trabaja con entidades más complejas.

Si ejecutamos la consulta a través de Swagger, veremos el perfil de mapeo en acción:

```json
[
  {
    "productId": 1,
    "description": "Product 01",
    "price": 16000,
    "listDescription": "Product 01 - $16,000.00"
  },
  {
    "productId": 2,
    "description": "Product 02",
    "price": 52200,
    "listDescription": "Product 02 - $52,200.00"
  }
]
```

Este ejemplo ilustra cómo utilizar AutoMapper para simplificar el mapeo de objetos y cómo adaptarlo a casos de mapeo más complejos cuando sea necesario.
### IMapper y Comandos

También es posible realizar mapeo en la dirección opuesta, es decir, desde un DTO a una entidad.

En este caso, no estamos tratando con consultas, por lo que no podemos usar `ProjectTo<TDestionation>`. Sin embargo, podemos utilizar **IMapper** para mapear objetos individualmente.

```csharp
public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand>
{
    private readonly MyAppDbContext _context;
    private readonly IMapper _mapper;

    public CreateProductCommandHandler(MyAppDbContext context, IMapper mapper)
    {
        _context = context;
        _mapper = mapper;
    }

    public async Task<Unit> Handle(CreateProductCommand request, CancellationToken cancellationToken)
    {
        var newProduct = _mapper.Map<Product>(request);

        _context.Products.Add(newProduct);

        await _context.SaveChangesAsync();

        return Unit.Value;
    }
}

public class CreateProductCommandMapper : Profile
{
    public CreateProductCommandMapper() =>
        CreateMap<CreateProductCommand, Product>();
}
```

Aquí, hemos creado un perfil como en los ejemplos anteriores, pero en lugar de utilizar `ProjectTo<TDestionaiton>`, empleamos el método `Map<TDestionation>()` proporcionado por **IMapper**.

Dado que el perfil existe, el mapeo de las propiedades se realiza sin problemas.
## ¿Observas un nuevo problema?

Puede notarse que hemos creado varios perfiles que son prácticamente iguales, con la única diferencia de los tipos `TSource` y `TDestionation`.

Si bien hemos resuelto un problema, hemos introducido otro (aunque no sea grave). Sin embargo, es posible solucionar esto de manera más elegante utilizando **Reflection**.

Jason Taylor y su plantilla de Clean Architecture abordan este problema de una manera muy elegante. Puedes encontrar más detalles en [CleanArchitecture/src/Application/Common/Mappings en GitHub](https://github.com/jasontaylordev/CleanArchitecture/tree/main/src/Application/Common/Mappings).

Si deseas explorar esto en más detalle en español, no dudes en indicármelo y ampliaremos este artículo.
# Conclusión

El uso de bibliotecas como AutoMapper debería ser una práctica estándar en cualquier proyecto. Es una herramienta útil, ya que el mapeo de objetos es una tarea común en el desarrollo. Podemos realizarlo manualmente o de forma automática, pero siempre requeriremos algún tipo de mapeo.

Espero que hayas encontrado útil este artículo. Si tienes alguna pregunta, no dudes en dejarla en los comentarios 💬.
# Referencias

- [¿Qué es AutoMapper y cómo usarlo en ASP.NET Core (pragimtech.com)](https://www.pragimtech.com/blog/blazor/using-automapper-in-asp.net-core/)
# Introducción

En esta entrega de la serie de artículos, abordaremos cómo realizar consultas seguras a los productos que previamente hemos creado. Es posible que no se haya dimensionado adecuadamente el problema al que se enfrentan las aplicaciones al utilizar "IDs secuenciales" que pueden ser modificados directamente desde la URL. En el contexto de la API que estamos desarrollando, esto puede no parecer un problema significativo, ya que se trata de un ejemplo. Sin embargo, en la vida real, este tipo de problemas son bastante comunes, aunque no siempre se les presta la debida atención.

> Puedes encontrar el código fuente de este artículo en: [MediatrValidationExample](https://github.com/isaacOjeda/DevToPosts/tree/post-part4/MediatrValidationExample)
# Top 10 OWASP - A01 Broken Access Control

Uno de los riesgos de seguridad más comunes y destacados se encuentra en el punto principal del Top 10 de OWASP: **A01 Broken Access Control**. En el año 2017, esta vulnerabilidad ocupaba el quinto lugar, pero en 2021 ascendió al primer puesto. El A01 engloba diversas malas prácticas que pueden comprometer el control de acceso en nuestras aplicaciones, y en este artículo nos enfocaremos en resolver una de ellas:

> Eludir las verificaciones de control de acceso al **modificar la URL** (alteración de parámetros o navegación forzada), el estado interno de la aplicación o la página HTML, o al utilizar una herramienta de ataque para modificar las solicitudes a la API.

Es bastante habitual en nuestras aplicaciones el uso de IDs Int32 secuenciales, una práctica que muchos programadores empleamos. Sin embargo, en términos de seguridad, esta elección plantea un desafío significativo. Imagina que tenemos una URL como esta: https://myapp.com/Payments/3232, donde supuestamente se muestra el pago con el ID 3232. Si somos lo suficientemente astutos, podríamos comenzar a explorar otros pagos realizados, incluso aquellos que no nos corresponden. Podríamos intentar con cualquier número secuencial que se nos ocurra.

De esta manera, si la URL no verifica los permisos de acceso, nuestra aplicación presenta una vulnerabilidad seria. Es cierto que en muchos casos se verifica la autorización, asegurándonos de que el usuario tenga los permisos adecuados para acceder al recurso. Sin embargo, ¿qué sucede si nuestro sitio es público, como YouTube? No debemos permitir que los usuarios intenten acceder a IDs predecibles, como **3232**, o que intenten adivinar otros números por completo.
Aquí tienes una versión mejorada del texto:
# HashIds: Genera IDs únicos y no secuenciales

[Hashids](https://github.com/ullmark/hashids.net) es una herramienta que nos brinda una solución eficaz a este problema. Se trata de una pequeña librería que nos permite generar IDs de manera similar a lo que hace **Youtube**, a partir de números enteros.

Hashids ofrece las siguientes características:

- Crea IDs únicos a partir de números enteros.
- Genera IDs no secuenciales, lo que significa que no se pueden predecir como un número entero común.
- Permite almacenar varios números enteros en un solo hash ID.
- Es altamente personalizable, lo que significa que puedes utilizar los caracteres que desees en los IDs generados.
## Instalación

La instalación de esta biblioteca es sencilla. Solo necesitas ejecutar el siguiente comando:

```bash
dotnet add package hashids.net
```
## ¿Cómo utilizarlo?

El funcionamiento de Hashids es muy simple. Puedes crear un objeto Hashids con una "salt" que elijas:

```csharp
var hashids = new Hashids("esta es mi salt");
var hash = hashids.Encode(12345);
```

Esto te devolverá un hash como este:

```
NkK9
```

Para recuperar el ID original a partir del hash, puedes hacer lo siguiente:

```csharp
var hashids = new Hashids("esta es mi salt");
var numbers = hashids.Decode("NkK9");
```

Obtendrás una matriz que contiene los números enteros originales:

```
[ 12345 ]
```

Si deseas explorar más sobre las funciones y características de Hashids, puedes visitar su [repositorio en GitHub](https://github.com/ullmark/hashids.net).
## Integrándolo con Queries y Comandos

Dado que este artículo es una continuación de la implementación de CQRS con MediatR, ahora vamos a integrar Hashids con los queries y comandos que hemos desarrollado hasta el momento.

Para facilitar su uso, crearemos métodos de extensión que permitirán la conversión directa de enteros (`int`) a cadenas (`string`) ya hasheadas y viceversa.

> Nota 💡: Es importante tener en cuenta que, aunque hemos optado por utilizar estos métodos de extensión por su simplicidad y su compatibilidad con AutoMapper, es posible que existan alternativas que permitan una implementación aún más eficiente. Siempre es válido explorar otras opciones para lograr los mejores resultados en tus proyectos.

**Helpers/AppHelpers.cs**

```csharp
using HashidsNet;

namespace MediatrValidationExample.Helpers;

public static class AppHelpers
{
    public const string HashIdsSalt = "s3cret_s4lt";

    public static string ToHashId(this int number) =>
        GetHasher().Encode(number);

    public static int FromHashId(this string encoded) =>
        GetHasher().Decode(encoded).FirstOrDefault();

    private static Hashids GetHasher() => new(HashIdsSalt, 8);
}
```

Este conjunto de métodos de extensión nos facilitará realizar acciones como `IntProperty.ToHashId()`. Los métodos de extensión son una herramienta valiosa en este contexto.

El "Salt" se utiliza para generar los hashes a partir de ese valor, como si fuera una "llave". Es importante tener en cuenta que no se trata de un hash en el sentido tradicional, ya que es reversible.

### GetProductsQuery

Para implementar esta funcionalidad, es necesario modificar el tipo de dato de `ProductId` a `string`. Además, ajustaremos el perfil de mapeo (Mapping Profile) para llevar a cabo esta conversión:

```csharp
public class GetProductsQueryProfile : Profile
{
    public GetProductsQueryProfile() =>
        CreateMap<Product, GetProductsQueryResponse>()
            .ForMember(dest =>
                dest.ListDescription,
                opt => opt.MapFrom(mf => $"{mf.Description} - {mf.Price:c}"))
            .ForMember(dest =>
                dest.ProductId,
                opt => opt.MapFrom(mf => mf.ProductId.ToHashId()));

}
```

Recordemos que estamos utilizando **AutoMapper** y estos tutoriales están estructurados de forma secuencial. Si aún no has trabajado con AutoMapper, te recomendamos revisar estos tutoriales para comprender su funcionamiento.

Si ejecutamos esta configuración utilizando Swagger, obtendremos una respuesta similar a la siguiente:

```json
[
  {
    "productId": "Wxoz2omz",
    "description": "Product 01",
    "price": 16000,
    "listDescription": "Product 01 - $16,000.00"
  },
  {
    "productId": "kwoYX31v",
    "description": "Product 02",
    "price": 52200,
    "listDescription": "Product 02 - $52,200.00"
  }
]
```

Como puedes observar, hemos dejado de utilizar IDs secuenciales, y en cuanto al rendimiento, podemos estar seguros de que esta implementación no añade una carga significativa.

Estos IDs, aunque los números enteros puedan ser secuenciales, se convierten en hashes que resultan imposibles de adivinar (al menos, a menos que se conozca el Salt).

### GetProductQuery

En este caso, vamos a buscar un producto por su ID, por lo que es necesario realizar la conversión de vuelta a un número entero.

> Nota: El `ProductId` del request debe convertirse a `string`. Siempre puedes visitar el repositorio asociado a este artículo para confirmar la forma en que se realiza todo el proceso.

```csharp
var product = await _context.Products.FindAsync(request.ProductId.FromHashId());

if (product is null)
{
    throw new NotFoundException(nameof(Product), request.ProductId);
}

return _mapper.Map<GetProductQueryResponse>(product);
```

La implementación de estos métodos de extensión facilita enormemente todas estas operaciones.

Podemos consultar el producto con el ID "Wxoz2o" y debería encontrarse sin problemas.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wcduh0w2j9q4nrxk53b0.png)

# Conclusión

Si bien podría parecer repetitivo tener que realizar la codificación y decodificación continuamente, esta es una opción que puedes considerar en tus proyectos. La aplicación de Hashids resulta efectiva para proteger las URLs, y aunque no es necesario aplicarla en todos los recursos de tus aplicaciones, puede ser una solución útil cuando sea necesario garantizar la seguridad de los datos.

# Referencias

- [A01 Broken Access Control - OWASP Top 10:2021](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [ullmark/hashids.net: A small .NET package to generate YouTube-like hashes from one or many numbers. (github.com)](https://github.com/ullmark/hashids.net)

# Introducción
En aplicaciones modernas, la localización es un aspecto crucial para ofrecer una experiencia personalizada a los usuarios de diferentes culturas e idiomas. ASP.NET Core nos proporciona una manera sencilla de implementar la localización utilizando archivos de recursos, pero este enfoque puede no ser ideal para todos los escenarios. Los archivos de recursos (archivos XML), pueden volverse difíciles de gestionar a medida que crecen en tamaño y número, especialmente cuando se trata de aplicaciones que soportan múltiples idiomas y requieren actualizaciones frecuentes.

En este artículo, exploraremos cómo podemos ir más allá de las opciones predeterminadas y construir una solución de localización personalizada utilizando una base de datos como fuente de los textos localizados. Este enfoque nos ofrece una mayor flexibilidad y desacoplamiento del código, permitiendo agregar y modificar traducciones sin necesidad de redeploys ni cambios en el código fuente.

Veremos cómo implementar esta solución utilizando ASP.NET Core, Entity Framework Core, y el concepto de `IStringLocalizer` e `IStringLocalizerFactory` para crear una infraestructura que pueda adaptarse a las necesidades específicas de cualquier proyecto. Al final de este artículo, tendrás una base sólida para implementar localización basada en bases de datos, permitiendo a tus aplicaciones manejar traducciones de manera más eficiente y escalable.

# Localización personalizada en ASP.NET Core

La implementación predeterminada de localización en ASP.NET Core se basa en archivos de recursos (resources files). Estos archivos son un arma de doble filo: puedes amarlos u odiarlos.

Cuando trabajas con múltiples idiomas y una gran cantidad de textos por traducir, los archivos XML pueden volverse difíciles de mantener y gestionar. A menos que estés usando Visual Studio, manejar estos archivos puede ser un verdadero dolor de cabeza.

Algunos frameworks, como [ABP](https://abp.io/docs/1.0/Localization), ofrecen soluciones basadas en archivos JSON, lo cual es un avance. Sin embargo, lo que quiero demostrar en este artículo es que no estamos limitados a las opciones que otros nos ofrecen. Podemos implementar la localización de la manera que mejor se adapte a nuestras necesidades.

En la siguiente sección, exploraremos cómo implementar una solución de localización que no se base en archivos XML o JSON, sino en una base de datos.
## Localización por Base de Datos
Implementar la localización utilizando una base de datos tiene sus ventajas y desventajas, que vamos a analizar en detalle.

**Ventajas**

- **Desacoplamiento del Proyecto**: Una de las principales ventajas es que los recursos de localización no están embebidos en el código ni en el repositorio, lo que permite que las traducciones evolucionen sin necesidad de modificar el código ni realizar nuevos deployments. Esto facilita la gestión y reduce el tiempo de inactividad.
- **Facilidad de Expansión**: Agregar nuevos idiomas es tan simple como insertar registros adicionales en la base de datos. No es necesario cambiar el código ni hacer nuevos despliegues, lo que permite una expansión rápida y menos propensa a errores.
- **Centralización y Mantenimiento**: Al centralizar todas las traducciones en una base de datos, es más fácil mantener y auditar los recursos. Puedes aplicar reglas de negocio, implementar sistemas de validación y tener una vista centralizada de todos los textos traducidos.
- **Interfaz de Usuario para Edición**: Puedes desarrollar una interfaz administrativa que permita a los usuarios no técnicos agregar o modificar traducciones directamente en la base de datos, sin necesidad de la intervención del equipo de desarrollo.

**Desventajas**

- **Rendimiento y Caché**: Es crucial implementar un sistema de caché efectivo, ya que acceder a la base de datos para cada traducción podría impactar negativamente el rendimiento de la aplicación.
- **Complejidad de Gestión**: Aunque esta solución es flexible, la gestión de las traducciones en la base de datos puede ser complicada, especialmente en sistemas con múltiples culturas e idiomas. Se requiere una estrategia clara para evitar duplicidades, inconsistencias y garantizar que las traducciones estén siempre actualizadas.
- **Dependencia de la Base de Datos**: La localización depende de la disponibilidad y el rendimiento de la base de datos. Si la base de datos experimenta problemas, la localización podría verse afectada, impactando la experiencia del usuario final.
- **Sin Soporte Out-of-the-Box**: A diferencia de los archivos de recursos predeterminados, esta solución requiere más configuración y personalización, lo que implica un mayor esfuerzo inicial en términos de desarrollo y pruebas.

### LocalizationResource

Para implementar la localización en una base de datos, utilizaremos Entity Framework Core para manejar la persistencia. La clase `LocalizationResource` será la encargada de representar cada recurso localizado.

```csharp
namespace CustomLocalization.Localization.Data;
 
public class LocalizationResource
{
    public int Id { get; set; }
    public string Culture { get; set; }
    public string Key { get; set; }
    public string Value { get; set; }
    public string AssemblyName { get; set; }
}
```

Las propiedades incluidas son:
- **Id**: Identificador único del recurso.
- **Culture**: La cultura del recurso (por ejemplo, es-MX, en-US).
- **Key**: La clave utilizada para identificar el recurso.
- **Value**: La cadena de texto traducida.
- **AssemblyName**: El nombre del ensamblado donde se encuentra el recurso localizado (esto es útil cuando se localizan clases específicas).

### LocalizationDbContext

El contexto de la base de datos también es bastante sencillo y sigue las convenciones estándar de Entity Framework Core:

```csharp
using Microsoft.EntityFrameworkCore;
 
namespace CustomLocalization.Localization.Data;
 
public class LocalizationDbContext(DbContextOptions<LocalizationDbContext> options) : DbContext(options)
{
    public DbSet<LocalizationResource> LocalizationResources { get; set; }
}
```

### DatabaseStringLocalizer

Aquí es donde comienza la parte interesante. Necesitaremos dos componentes clave: `IStringLocalizer` y `IStringLocalizerFactory`. Estos trabajan en conjunto, de manera similar a cómo ASP.NET Core utiliza archivos XML de recursos, para construir un `IStringLocalizer<SomeClass>` utilizando el Factory que vamos a definir.

La implementación de `IStringLocalizer` utilizando Entity Framework Core podría verse así:

```csharp
using CustomLocalization.Localization.Data;
using Microsoft.Extensions.Localization;
 
namespace CustomLocalization.Localization;
 
public class DatabaseStringLocalizer : IStringLocalizer
{
    private readonly LocalizationDbContext _context;
    private readonly string _culture;
    private readonly string _assemblyName;
    private readonly List<LocalizationResource> _assemblyResources;
    public DatabaseStringLocalizer(LocalizationDbContext context, string culture, string assemblyName)
    {
        _context = context;
        _culture = culture;
        _assemblyName = assemblyName;

         // TODO: Guardar en caché es un MUST
        _assemblyResources = _context.LocalizationResources
            .Where(r => r.Culture == _culture && r.AssemblyName == _assemblyName)
            .ToList();
    }
 
    public LocalizedString this[string name]
    {
        get
        {
            var resource = _assemblyResources.FirstOrDefault(r => r.Key == name);
            if (resource == null)
            {
                return new LocalizedString(name, name, true);
            }
 
            return new LocalizedString(name, resource.Value, false);
        }
    }
 
    public LocalizedString this[string name, params object[] arguments]
    {
        get
        {
            var resource = _assemblyResources.FirstOrDefault(r => r.Key == name);
            if (resource == null)
            {
                return new LocalizedString(name, name, true);
            }
 
            return new LocalizedString(name, resource.Value, false);
        }
    }
 
    public IEnumerable<LocalizedString> GetAllStrings(bool includeParentCultures)
    {
        return _assemblyResources.Select(r => new LocalizedString(r.Key, r.Value, false));
    }
}
```

#### Explicación del Código

En la clase `DatabaseStringLocalizer`, estamos implementando la interfaz `IStringLocalizer`, que es esencial para manejar la localización en ASP.NET Core. Este `StringLocalizer` se conecta a la base de datos para buscar las cadenas de texto traducidas según la cultura actual y el ensamblado que se está utilizando.

- **Constructor**: El constructor toma como parámetros el contexto de base de datos `LocalizationDbContext`, la cultura actual y el nombre del ensamblado. Luego, consulta la base de datos para obtener todos los recursos localizados que coincidan con la cultura y el ensamblado, almacenándolos en una lista.
- **Indexer**: El indexador `this[string name]` se utiliza para obtener el texto traducido correspondiente a una clave. Si no se encuentra ninguna traducción, se devuelve la clave original, indicando que la cadena no ha sido localizada. ==Esto es útil para evitar errores en caso de que falten traducciones==.
- **GetAllStrings**: Este método devuelve todas las cadenas localizadas disponibles para la cultura y ensamblado especificados. Es útil para casos donde se necesita mostrar una lista completa de traducciones, como en una interfaz de administración.

### StringLocalizationFactory

El `StringLocalizationFactory` es responsable de crear instancias de `IStringLocalizer` basadas en la cultura actual y el recurso solicitado. Este factory sigue el patrón de diseño Factory, delegando la creación de objetos a una clase especializada.

```csharp
using CustomLocalization.Localization.Data;
using Microsoft.Extensions.Localization;
using System.Globalization;
 
namespace CustomLocalization.Localization;
 
public class StringLocalizationFactory(IServiceProvider serviceProvider) : IStringLocalizerFactory
{
    public IStringLocalizer Create(Type resourceSource)
    {
        var currentCulture = CultureInfo.CurrentCulture.Name;
        var assembly = resourceSource.FullName;
        using var scope = serviceProvider.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<LocalizationDbContext>();
 
        return new DatabaseStringLocalizer(context, currentCulture, assembly);
    }
 
    public IStringLocalizer Create(string baseName, string location)
    {
        var currentCulture = CultureInfo.CurrentCulture.Name;
        using var scope = serviceProvider.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<LocalizationDbContext>();
 
        return new DatabaseStringLocalizer(context, currentCulture, baseName);
    }
}
```

#### Explicación del Código

El `StringLocalizationFactory` facilita la creación de `IStringLocalizer` utilizando la cultura actual y el recurso solicitado. Es un factory singleton, lo que significa que crea instancias de localizadores según sea necesario, y utiliza un scope para manejar las dependencias.

- **Create(Type resourceSource)**: Este método toma como parámetro un tipo (`Type`) que representa la clase o ensamblado que solicita el `IStringLocalizer`. Utilizando la cultura actual y el nombre completo del ensamblado, crea un nuevo `DatabaseStringLocalizer` que obtiene las traducciones correspondientes de la base de datos.
- **Create(string baseName, string location)**: Este método es una sobrecarga del anterior y permite crear un `IStringLocalizer` utilizando un nombre base (`baseName`) y una ubicación (`location`). Es útil cuando se desea especificar manualmente el nombre y la ubicación de los recursos de localización.

### HomeEndpoint

En el siguiente ejemplo, se demuestra cómo cualquier clase puede tener su propia localización, similar a cómo funciona en ASP.NET Core de forma predeterminada.

El endpoint que se presenta a continuación simplemente retorna la traducción del texto `Hello, World!`. Este enfoque sigue el mismo estilo de ASP.NET Core, permitiendo desarrollar aplicaciones sin preocuparse inicialmente por la localización. Los textos se proporcionan como claves en un idioma predeterminado, generalmente el inglés.

Este estilo es altamente flexible, ya que no es necesario realizar ninguna configuración adicional durante el desarrollo. Solo en producción se deben proporcionar las traducciones correspondientes. Si una traducción no existe, se devolverá la clave original, que representa el texto en el idioma predeterminado.

```csharp
using Microsoft.Extensions.Localization;
 
namespace CustomLocalization.Endpoints;
 
public class HomeEndpoint
{
    public static IResult Index(IStringLocalizer<HomeEndpoint> localizer)
    {
        return Results.Ok(localizer["Hello, World!"]);
    }
}
```

`IStringLocalizer` se puede utilizar en cualquier parte de la aplicación, como en un Controller, una vista de Razor, Blazor Server, entre otros.
### Program

En esta sección configuramos las dependencias necesarias.

Como podrás observar, no estamos agregando directamente el `DatabaseStringLocalizer` al contenedor de dependencias. En su lugar, estamos utilizando la implementación predeterminada de ASP.NET Core llamada `StringLocalizer`.

Esta clase utiliza internamente el `IStringLocalizerFactory`, que en nuestro caso es `StringLocalizationFactory`. Aunque este nombre podría cambiarse a algo más descriptivo que haga referencia a la base de datos 😅, lo que hace esta fábrica es crear instancias del `DatabaseStringLocalizer`, lo que sigue un patrón de fábrica (Factory Pattern). Este patrón permite una abstracción clara entre la lógica de creación de objetos y su uso.

```csharp
var builder = WebApplication.CreateBuilder(args);
 
// Add services to the container.
// other services...

builder.Services.AddSingleton<IStringLocalizerFactory, StringLocalizationFactory>();
builder.Services.AddTransient(typeof(IStringLocalizer<>), typeof(StringLocalizer<>));

builder.Services.AddDbContext<LocalizationDbContext>(opts => opts
    .UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
    
// other services...

var app = builder.Build();

// Localización de cada Request va primero que todo
app.UseRequestLocalization(new RequestLocalizationOptions
{
    DefaultRequestCulture = new RequestCulture("en-US"),
    SupportedCultures = new List<CultureInfo>
    {
        new("en-US"),
        new("es-MX"),
        new("de-DE")
    },
    SupportedUICultures = new List<CultureInfo>
    {
        new("en-US"),
        new("es-MX"),
        new("de-DE")
    }
});
 
// Configure the HTTP request pipeline.
// other middelwares...

app.MapGet("/api", HomeEndpoint.Index);

// stuff...

```

### Explicación de `UseRequestLocalization`

El middleware `UseRequestLocalization` se encarga de configurar la localización para cada request. Aquí se define la cultura predeterminada (`en-US`) y las culturas que la aplicación soporta (`en-US`, `es-MX` y `de-DE`). Este middleware se ejecuta antes que cualquier otro, asegurando que la cultura esté correctamente establecida para el procesamiento de la solicitud.

Si en la base de datos existe la localización adecuada, la aplicación retornará el texto configurado según la cultura del sistema.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/j71fs1z6tiq4880iq9q7.png)

Al realizar una solicitud al endpoint y establecer el encabezado HTTP `Accept-Language`, obtendremos el siguiente resultado. Para más información, puedes consultar la [documentación oficial](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/localization/select-language-culture?view=aspnetcore-8.0).

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gbkw07kgsfqzin08wul5.png)



# Conclusión
La localización es una característica clave en el desarrollo de aplicaciones globales, y ASP.NET Core ofrece una solución incorporada que es suficiente para la mayoría de los proyectos. Sin embargo, en situaciones donde se requiere un mayor control o flexibilidad, como en el manejo de grandes volúmenes de traducciones o la necesidad de actualizar idiomas sin redeploys, una localización personalizada puede ser una excelente opción.

En este artículo, hemos explorado cómo implementar una solución de localización basada en bases de datos, utilizando `IStringLocalizer` y `IStringLocalizerFactory` personalizados con Entity Framework Core. Este enfoque permite desacoplar los recursos localizados del código fuente, facilitando la gestión de traducciones y la adición de nuevos idiomas de manera más dinámica.

Es importante destacar que este es solo un ejemplo de cómo se puede personalizar la localización en ASP.NET Core. Las posibilidades son prácticamente ilimitadas: podrías extender esta idea para almacenar traducciones en archivos JSON, consumirlas desde una API externa, o incluso combinar varios métodos según las necesidades específicas de tu proyecto.

Aunque la solución incorporada de ASP.NET Core es suficiente en muchos casos, tener la capacidad de personalizar y extender la localización abre la puerta a nuevas oportunidades para crear aplicaciones más flexibles y adaptables. Espero que este artículo te haya dado una perspectiva valiosa sobre cómo puedes llevar la localización en tus proyectos al siguiente nivel.
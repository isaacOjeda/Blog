# Introducción
En este artículo, no presentaremos conceptos revolucionarios, pero es esencial para continuar explorando los aspectos y trucos de ASP.NET. Inicialmente, evitamos complicar las cosas dividiendo todo en proyectos y capas. Sin embargo, al adoptar la Vertical Slice Architecture, encontramos una forma sencilla y bien estructurada de hacerlo.

He abordado este tema en otras ocasiones, y puedes consultar mi [repositorio](https://github.com/isaacOjeda/MinimalApiArchitecture), un [vídeo](https://www.youtube.com/watch?v=UTboacYR5tY&ab_channel=IsaacOjeda) y este [artículo](https://dev.to/isaacojeda/vertical-slice-architecture-36ng) donde exploramos en profundidad la **Vertical Slice Architecture**.

Si deseas obtener más información sobre este tema, te recomiendo visitar esos recursos.

> 💡 Nota: Puedes encontrar el código correspondiente a este artículo en [este repositorio](https://github.com/isaacOjeda/DevToPosts/tree/post-part6/MediatrValidationExample). Y, como siempre, no dudes en contactarme en [Twitter](https://twitter.com/balunatic).

# Refactorizando la Solución
Este conjunto de publicaciones surgió a partir de una idea principal: implementar CQRS y validaciones con FluentValidation (de ahí el nombre de la solución, MediatRValidationExample 🤣). A medida que avanzamos, este proyecto se expandió y ya tengo planeadas 10 o más partes.

![Solución original](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wefg6hpnamrj6nyt6qeu.png)

Actualmente, la solución se presenta como un único proyecto web que alberga todo el código. Aunque está organizado según preocupaciones técnicas, examinemos detenidamente cada elemento para refrescar nuestra comprensión:

## Presentación
La capa de presentación o UI, en términos prácticos, abarca todo lo relacionado con la API web. A continuación, un resumen de lo que comprende esta capa:

- **Controladores**: Los controladores son una parte integral de la interfaz de usuario, y mantendremos esta estructura intacta en su ubicación actual.
- **Filtros**: Los filtros se aplican a los controladores, por lo tanto, están estrechamente relacionados con la presentación.
- **Servicios** (en parte): En el directorio "Servicios", encontramos la implementación de `CurrentUserService`. Esta clase está vinculada al contexto HTTP, pero se ha diseñado con una interfaz para facilitar la abstracción necesaria.
  - Nota: Siempre es fundamental abstraer cuando es necesario, en lugar de crear abstracciones sin un propósito claro.

## Application Core
Dentro de la Vertical Slice Architecture, aquí es donde se encuentra el resto de la aplicación. En comparación con la Clean Architecture, aquí abarcamos tanto el dominio (Domain), el núcleo (Core), la persistencia (Persistence) e infraestructura (Infraestructura).

¿Por qué los agrupamos? Este es otro tema que te animo a explorar en el contenido mencionado anteriormente.

En resumen, para el núcleo de la aplicación en su estado actual, tenemos:

- **Behaviours**: Decoradores añadidos utilizando **MediatR** que incorporan reglas de negocio y otras funcionalidades específas de la lógica de la aplicación.
- **Exceptions**: Excepciones personalizadas que, al igual que los comportamientos, aportan lógica y reglas al núcleo.
- **Helpers** (conocidas como Utils): Este es un elemento un tanto aleatorio que se introdujo en la publicación sobre Hash IDs, pero dado que solo se utilizan en el core, permanece en el core.
- **Dominio**: Todo lo relacionado con el dominio (value objects, entities, enums, entity exceptions, domain services, etc.).
- **Features**: Todos los segmentos funcionales de la aplicación.
- **Infrastructure**: Adaptadores y servicios para comunicarse con servicios externos.
- **Persistence**: La base de datos (EF Core).

Una vez que se complete la refactorización, la solución se verá como se muestra a continuación:

![Solución refactorizada](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wntwp5bxn0sqkib10arw.png)

Si estás siguiendo estos tutoriales, te recomiendo que **no realices la refactorización** de inmediato. En cambio, te sugiero descargar el código y analizarlo. Además, ten en cuenta que la parte 7 de esta serie de publicaciones será un tanto diferente 🤭.

## Actualización de la Inyección de Dependencias
Ahora permitimos que cada proyecto registre sus propias dependencias (Web y Core), y he añadido dos clases con extensiones para facilitar este proceso: la clase `DependencyInjection`.

Aquí tienes una versión revisada de los párrafos:

### ApplicationCore -> DependencyInjection

```csharp
using FluentValidation;
using MediatR;
using MediatrExample.ApplicationCore.Common.Behaviours;
using MediatrExample.ApplicationCore.Infrastructure.Persistence;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Identity;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.IdentityModel.Tokens;
using System.Reflection;
using System.Text;

namespace MediatrExample.ApplicationCore;
public static class DependencyInjection
{
    public static IServiceCollection AddApplicationCore(this IServiceCollection services)
    {
        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
        services.AddMediatR(Assembly.GetExecutingAssembly());
        services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehaviour<,>));
        services.AddAutoMapper(Assembly.GetExecutingAssembly());

        return services;
    }

    public static IServiceCollection AddPersistence(this IServiceCollection services, string connectionString)
    {
        services.AddSqlite<MyAppDbContext>(connectionString);

        return services;
    }

    public static IServiceCollection AddSecurity(this IServiceCollection services, IConfiguration config)
    {

        services
            .AddIdentityCore<IdentityUser>()
            .AddRoles<IdentityRole>()
            .AddEntityFrameworkStores<MyAppDbContext>();

        services
            .AddHttpContextAccessor()
            .AddAuthorization()
            .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options =>
            {
                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuer = true,
                    ValidateAudience = true,
                    ValidateLifetime = true,
                    ValidateIssuerSigningKey = true,
                    ValidIssuer = config["Jwt:Issuer"],
                    ValidAudience = config["Jwt:Audience"],
                    IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(config["Jwt:Key"]))
                };
            });

        return services;
    }
}
```

Realmente podrías combinar todos estos métodos en uno solo, pero la segmentación actual es útil para comprender su propósito.

> Nota 💡: Estas son pautas de referencia. Puedes elegir lo que mejor se adapte a tus necesidades y preferencias. El tema de la Vertical Slice es muy interesante, te animo a investigar más sobre él y explorar los recursos de [Jimmy Boggard](https://jimmybogard).

### WebApi -> DependencyInjection

```csharp
using FluentValidation.AspNetCore;
using MediatrExample.ApplicationCore.Common.Interfaces;
using MediatrExample.WebApi.Filters;
using MediatrExample.WebApi.Services;
using Microsoft.AspNetCore.Mvc;
using Microsoft.OpenApi.Models;

namespace MediatrExample.WebApi;

public static class DependencyInjection
{
    public static IServiceCollection AddWebApi(this IServiceCollection services)
    {
        services.AddEndpointsApiExplorer();

        services.AddSwaggerGen(c =>
        {
            c.SwaggerDoc("v1", new OpenApiInfo
            {
                Title = "My API",
                Version = "v1"
            });
            c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
            {
                In = ParameterLocation.Header,
                Description = "Please insert JWT with Bearer into field",
                Name = "Authorization",
                Type = SecuritySchemeType.ApiKey
            });

            c.AddSecurityRequirement(new OpenApiSecurityRequirement
            {
                {
                    new OpenApiSecurityScheme
                    {
                        Reference = new OpenApiReference
                        {
                            Type = ReferenceType.SecurityScheme,
                            Id = "Bearer"
                        }
                    },
                    new string[] { }
                }
            });
        });

        services.AddControllers(options =>
            options.Filters.Add<ApiExceptionFilterAttribute>())
            .AddFluentValidation();
        services.Configure<ApiBehaviorOptions>(options =>
            options.SuppressModelStateInvalidFilter = true);

        services.AddScoped<ICurrentUserService, CurrentUserService>();

        return services;
    }
}
```

Es esencial mantener una organización limpia y legible en tu código, y dividirlo en extensiones facilita la comprensión cuando se revisa el `Program`.

### WebApi -> Program

```csharp
using MediatrExample.ApplicationCore;
using MediatrExample.ApplicationCore.Domain;
using MediatrExample.ApplicationCore.Infrastructure.Persistence;
using MediatrExample.WebApi;
using Microsoft.AspNetCore.Identity;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddWebApi();
builder.Services.AddApplicationCore();
builder.Services.AddPersistence(builder.Configuration.GetConnectionString("Default"));
builder.Services.AddSecurity(builder.Configuration);

var app = builder.Build();

// Configurar la canalización de solicitudes HTTP.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

await SeedProducts();

app.Run();

// Seed omitido...
```

La organización del `Program` ahora es mucho más limpia, y cualquier aspecto específico se puede encontrar fácilmente en la extensión correspondiente.

# Conclusión
En resumen, no hay mucho que concluir. Mi objetivo era explicar el proceso de refactorización para las próximas publicaciones, y al principio, mi enfoque era mantenerlo simple.

Siempre he seguido una estructura de carpetas coherente, por lo que esta refactorización no debería presentar ningún problema.
## Introducción

Crear APIs es lo más común hoy en día y aunque es el tema de momento, también existen muchas formas de implementar la autenticación y así poder protegerlas de uso no autorizado. 

Con OAuth OpenID Connect podemos tener una autenticación y autorización muy robusta, pero a veces queremos hacer cosas simples, pero aun así seguras.

El uso de API Keys es super común y todos lo hacen, si buscas un API del Clima, información de países, población, etc. Todos te otorgarán un API Key para darte acceso a este tipo de información.

Contar con un API Key para cada aplicación cliente nos sirve para muchas cosas, la principal es para identificar quien está haciendo la llamada.

También sirve para poder revocar el acceso según el API Key (o limitar su uso).

Y si la API permite modificaciones, también podemos saber que operaciones realizó cada aplicación cliente según su API Key

## API Key Custom Authentication

Realizar validación por API Key se puede realizar de muchas formas, pero lo que me gusta hacer y no es complicado, es hacer un esquema de autenticación para poder soportar el mecanismo que ya existe en ASP.NET.

Utilizando este modo podemos tener claims y un Identity, por lo que podemos agregar la info que se necesite y todo lo que ya conocemos que podemos hacer cuando utilizamos JWT o Cookies.

Pero bueno, para seguir, crearemos un proyecto vacío con `dotnet new web`.

> Nota 💡: Como siempre, aquí está el [código](https://github.com/isaacOjeda/DevToPosts/tree/main/ApiKeyCustomAuth) para que lo revises de una forma más fácil.

### Entities > ApiKey

La idea de este ejemplo es poder tener varias API Keys para así poder identificar a las aplicaciones que están haciendo uso de nuestra API.

Así podemos deshabilitar API Keys sin problema y poder especificar permisos o scopes que el API Key podrá tener:

```csharp
namespace ApiKeyCustomAuth.Entities;
  
public class ApiKey
{
    public int ApiKeyId { get; set; }
    public Guid Key { get; set; }
    public string Name { get; set; }
}
```

Por ahora, solo queremos un nombre que lo identifique y la llave "privada" que usará para ser autenticado en la API

### Data > ApiDbContext

De una forma sencilla, tendremos este contexto para poder guardar las API Keys (se puede hacer de muchas formas, no necesitas obligatoriamente una base de datos).

```csharp
using ApiKeyCustomAuth.Entities;
using Microsoft.EntityFrameworkCore;
  
namespace ApiKeyCustomAuth.Data;
  
public class ApiDbContext : DbContext
{
    public ApiDbContext(DbContextOptions<ApiDbContext> options)
        : base(options)
    { }
  
    public DbSet<ApiKey> ApiKeys => Set<ApiKey>();
}
```

### Auth >  ApiKeySchemeOptions

Cuando creamos un esquema de autenticación personalizado, debemos de indicar la configuración que le podemos dar.

Podríamos no necesitar nada que configurar, pero en este caso puse solo como ejemplo, la posibilidad de personalizar el HTTP Header en donde se buscará el API Key. Por default, pues será `Authorization`.

```csharp
using Microsoft.AspNetCore.Authentication;
using Microsoft.Net.Http.Headers;
  
namespace ApiKeyCustomAuth.Auth;
  
public class ApiKeySchemeOptions : AuthenticationSchemeOptions
{
    public const string Scheme = "ApiKeyScheme";
  
    /// <summary>
    /// Nombre del Header donde se buscará la API Key
    /// Default: Authorization
    /// </summary>
    /// <value></value>
    public string HeaderName { get; set; } = HeaderNames.Authorization;
}
```

### Auth > ApiKeySchemeHandler

El Handler es llamado cada vez que ASP.NET necesita autenticar una llamada HTTP, ya que solo tendremos un esquema, pueste será siempre el que se mande a llamar.

```csharp
using System.Security.Claims;
using System.Text.Encodings.Web;
using ApiKeyCustomAuth.Data;
using Microsoft.AspNetCore.Authentication;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Options;
using Microsoft.Net.Http.Headers;

namespace ApiKeyCustomAuth.Auth; 

public class ApiKeySchemeHandler : AuthenticationHandler<ApiKeySchemeOptions>
{
    private readonly ApiDbContext _context;
 
    public ApiKeySchemeHandler(ApiDbContext context, IOptionsMonitor<ApiKeySchemeOptions> options,

        ILoggerFactory logger, UrlEncoder encoder, ISystemClock clock) : base(options, logger, encoder, clock)
    {
        _context = context;
    }

    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.ContainsKey(Options.HeaderName))
        {
            return AuthenticateResult.Fail("Header Not Found.");
        }
  
        var headerValue = Request.Headers[Options.HeaderName];
  
        var apiKey = await _context.ApiKeys
            .AsNoTracking() // TODO: Usar caché es buena idea
            .FirstOrDefaultAsync(a => a.Key.ToString() == headerValue);

        if (apiKey is null)
        {
            return AuthenticateResult.Fail("Wrong Api Key.");
        }
     
        var claims = new Claim[]
        {
            new Claim(ClaimTypes.NameIdentifier, $"{apiKey.ApiKeyId}"),
            new Claim(ClaimTypes.Name, apiKey.Name)
        };

        var identiy = new ClaimsIdentity(claims, nameof(ApiKeySchemeHandler));
        var principal = new ClaimsPrincipal(identiy);
        var ticket = new AuthenticationTicket(principal, Scheme.Name);

        return AuthenticateResult.Success(ticket);
    }
}
```

Resumen:
- Según la configuración, buscamos que `HeaderName` exista, si no, pues la llamada HTTP es no autorizada
- Si sí existe el `HeaderName`, aquí debería de venir el API Key, por lo que buscamos el Entity ApiKey según ese valor. Si No existe, es una llamada no autorizada
- Si la llamada es autorizada, necesitamos crear una "Identidad" que estará disponible en todo el Scope de la llamada HTTP, por lo que aquí agregamos los Claims que necesitamos
	- Aquí podemos agregar roles y más información que nos puede ser útil para identificar a la aplicación actual que hace la llamada

### Program

Para finalizar, hay que configurar toda la aplicación y conectar los cables (más un endpoint de prueba)

```csharp
using ApiKeyCustomAuth.Auth;
using ApiKeyCustomAuth.Data;
using ApiKeyCustomAuth.Entities;
using Microsoft.EntityFrameworkCore;
  
var builder = WebApplication.CreateBuilder(args);
  
builder.Services.AddDbContext<ApiDbContext>(options =>
    options.UseInMemoryDatabase(nameof(ApiDbContext)));
  
builder.Services.AddAuthorization();
builder.Services.AddAuthentication(ApiKeySchemeOptions.Scheme)
    .AddScheme<ApiKeySchemeOptions, ApiKeySchemeHandler>(
        ApiKeySchemeOptions.Scheme, options =>
        {
            options.HeaderName = "X-API-KEY";
        });
  
var app = builder.Build();
  
app.UseAuthentication();
app.UseAuthorization();
  
app.MapGet("/", (HttpRequest request) =>
{
    return new
    {
        request.HttpContext.User.Identity.Name,
        Claims = request.HttpContext.User.Claims
            .Select(s => new
            {
                s.Type,
                s.Value
            })
    };
}).RequireAuthorization();
  
await Seed();
  
app.Run();
  
async Task Seed()
{
    using var scope = app.Services.CreateScope();
    var context = scope.ServiceProvider.GetService<ApiDbContext>();
  
    if (!await context.ApiKeys.AnyAsync())
    {
        context.ApiKeys.Add(new ApiKey
        {
            Key = Guid.Parse("0e6b2066-9e98-4783-8c82-c3530aa8a197"),
            Name = "App 1"
        });
  
        context.ApiKeys.Add(new ApiKey
        {
            Key = Guid.Parse("607de3e9-2d01-430d-a6e1-d2ff8b6cfcf0"),
            Name = "App 2"
        });
  
        await context.SaveChangesAsync();
    }
}
```

Utilizaremos una base de datos en memoria solo para fines didácticos, pero lo ideal es que sea una base de datos normal.

En el método `Seed()` estamos creando dos API Keys de prueba para confirmar que todo funcione.

Con `AddAuthentication` estamos registrando nuestro esquema de autenticación, simplemente indicando la clase con la configuración que puede tener y el handler. En este caso estamos indicando que el header que queremos usar para ingresar el API Key es el header `X-API-KEY`.

El endpoint de ejemplo lo único que hace es acceder a los Claims que se establecieron desde el Authentication Handler. Es por eso que me gusta usar un esquema así, porque ya está integrado con el mismo ASP.NET y no utilizamos servicios aparte.

Con `request.HttpContext.User.Identity.Name` accedemos al Claim llamado `ClaimTypes.NameIdentifier` por lo que ASP.NET solito hace esas vinculaciones.

Si usamos `ClaimTypes.RoleName` y utilizamos `User.IsInRole(roleName)` funcionaría también.

## Probando la autenticación

Para probar es muy sencillo, con cualquier HTTP Client realizamos las siguientes pruebas:

```
GET {{host}}
X-API-KEY: 0e6b2066-9e98-4783-8c82-c3530aa8a197
```

Y de respuesta:
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/481n3c1eh28g9501u2dz.png)

Si utilizo un API Key inválido tendremos el resultado **Unauthorized**:
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/sqfgx848gixwuim6lfpm.png)

De igual forma, si uso el otro API Key de ejemplo:
```
GET {{host}}

X-API-KEY: 607de3e9-2d01-430d-a6e1-d2ff8b6cfcf0
```

Respuesta:
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7h895qgcd9wq4tqsh2z1.png)


## Conclusión

Hoy aprendiste a hacer un esquema de autenticación personalizado en ASP.NET Core, en este caso para el uso de APIs, esto es útil para este tipo de autenticación muy ad-hoc a como cada uno la quiera hacer.

Pero recuerda que ASP.NET Core ya tiene implementados varios mecanismos de autenticación (como Bearer Tokens o Cookies).

A veces necesitamos autenticación por Cookies pero con usuarios personalizados (sin Identity Core) en este caso, no es necesario crear un esquema de autenticación nuevo (ya que ya existe, se llama Cookie Authentication), pero para otro tipo de autenticación (Por Header, por Query Param, etc) podemos crearlo creando un Handler personalizado.

## Referencias
- [Working with custom authentication schemes in ASP.NET Core 6.0 Web API | Matteo Contrini](https://blog.contrini.it/aspnet-core-authentication-schemes/)
- [How to Custom Authentication Scheme in ASP.NET Core - Referbruv](https://referbruv.com/blog/implementing-custom-authentication-scheme-and-handler-in-aspnet-core-3x/)
- [How To Implement API Key Authentication In ASP.NET Core - YouTube](https://www.youtube.com/watch?v=CV6VdBR86co&ab_channel=MilanJovanovi%C4%87)
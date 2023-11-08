# Introducción
La autenticación mediante Bearer Tokens está en pleno auge, y a pesar de que ya he abordado este tema previamente en mi blog ([ASP.NET Core 6: Autenticación JWT y Identity Core](https://dev.to/isaacojeda/aspnet-core-6-autenticacion-jwt-y-identity-core-170i)), he decidido retomarlo.

La razón detrás de volver a hablar sobre JSON Web Tokens (JWTs) es mantener la coherencia en nuestra serie de publicaciones acerca de ASP.NET Core y CQRS con MediatR. En publicaciones futuras, necesitaremos contar con un sistema de autenticación y autorización sólido.

El objetivo final es construir una solución completa desde cero, desglosando cada concepto a lo largo de la serie.

Si deseas acceder al código fuente de este artículo, lo encontrarás en la rama correspondiente de mi repositorio en GitHub: [branch](https://github.com/isaacOjeda/DevToPosts/tree/post-part5/MediatrValidationExample).
# Autenticación con JWT Bearer
Cuando hablamos de JSON Web Tokens (JWT), a menudo se menciona el tema de [OpenID Connect](https://dev.to/isaacojeda/aspnet-core-servidor-de-autenticacion-con-openid-connect-59kh), y se vuelve aún más complejo cuando se implementa un Identity Server. Sin embargo, es importante entender que JWT es un mecanismo que se utiliza en el contexto de OpenID Connect, pero podemos aprovecharlo de manera independiente en la autenticación de nuestras aplicaciones.

> Nota 👀: Si deseas conocer en detalle qué son los JWT, te invito a revisar mi artículo anterior: [ASP.NET Core 6: Autenticación JWT y Identity Core](https://dev.to/isaacojeda/aspnet-core-6-autenticacion-jwt-y-identity-core-170i).

## Instalación de ASP.NET Identity Core y JWT Bearer
Para comenzar con la implementación de la autenticación, primero debemos agregar tres paquetes NuGet a nuestro proyecto:

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Microsoft.AspNetCore.Identity
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
```

ASP.NET Identity Core es un sistema de "Membership" que nos brinda una solución integral para la gestión de usuarios, autenticación y autorización. Es muy útil y altamente recomendado utilizarlo, ya que nos evita tener que reinventar la rueda.
## Actualizando el DbContext
Identity Core se basa principalmente en Entity Framework. En nuestro proyecto, ya disponemos de un `DbContext`, el cual seguiremos utilizando. No obstante, es necesario actualizarlo para que ahora sea compatible con las entidades que ofrece Identity.

```csharp
using MediatrValidationExample.Domain;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

namespace MediatrValidationExample.Infrastructure.Persistence;
public class MyAppDbContext : IdentityDbContext<IdentityUser> // <-----
{
    public MyAppDbContext(DbContextOptions<MyAppDbContext> options) : base(options)
    { }

    public DbSet<Product> Products => Set<Product>();
}
```

Lo más importante a destacar aquí es que ahora heredamos de `IdentityDbContext<TUser>` en lugar de `DbContext`. El tipo genérico `TUser` representa al usuario, y la clase `IdentityUser` es la implementación predeterminada que ofrece Identity. En otro artículo que mencioné previamente, ampliamos la clase `IdentityUser` para añadir las propiedades necesarias. Sin embargo, en este caso, por simplicidad, la mantendremos con la implementación predeterminada.

### Actualizando la Base de Datos
Estamos utilizando una base de datos SQLite, pero este proceso es igualmente aplicable a SQL Server u otros sistemas compatibles con EF Core. Para actualizar la base de datos, necesitamos agregar una migración correspondiente y luego aplicarla:

```bash
dotnet ef migrations add AddedIdentityCore -o Infrastructure/Persistence/Migrations
dotnet ef database update
```

## Generando JWTs
Antes de poder autorizar a los usuarios, debemos autenticarlos. Para lograrlo, crearemos un nuevo `Command` que realizará esta tarea.

> Nota 👀: Recuerda que el propósito de esta serie de tutoriales es seguir utilizando el enfoque CQRS.
> Nota 2: El enfoque tradicional hubiera sido colocar este archivo en la ruta Features/Auth/Command/TokenCommand.cs, pero por diversión, omitimos la palabra "Command" 🤣.

### Autenticación: Features -> Auth -> TokenCommand

Hemos creado este comando para autenticar a los usuarios mediante sus nombres de usuario y contraseñas.

```csharp
using MediatR;
using MediatrValidationExample.Exceptions;
using Microsoft.AspNetCore.Identity;
using Microsoft.IdentityModel.Tokens;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;

namespace MediatrValidationExample.Features.Auth;
public class TokenCommand : IRequest<TokenCommandResponse>
{
    public string UserName { get; set; } = default!;
    public string Password { get set; } = default!;
}

public class TokenCommandHandler : IRequestHandler<TokenCommand, TokenCommandResponse>
{
    private readonly UserManager<IdentityUser> _userManager;
    private readonly IConfiguration _config;

    public TokenCommandHandler(UserManager<IdentityUser> userManager, IConfiguration config)
    {
        _userManager = userManager;
        _config = config;
    }

    public async Task<TokenCommandResponse> Handle(TokenCommand request, CancellationToken cancellationToken)
    {
        // Verificamos las credenciales con Identity
        var user = await _userManager.FindByNameAsync(request.UserName);

        if (user is null || !await _userManager.CheckPasswordAsync(user, request.Password))
        {
            throw new ForbiddenAccessException();
        }

        var roles = await _userManager.GetRolesAsync(user);

        // Generamos un token en función de los claims
        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.Sid, user.Id),
            new Claim(ClaimTypes.Name, user.UserName)
        };

        foreach role in roles
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }

        var securityKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config["Jwt:Key"]));
        var credentials = new SigningCredentials(securityKey, SecurityAlgorithms.HmacSha256Signature);
        var tokenDescriptor = new JwtSecurityToken(
            issuer: _config["Jwt:Issuer"],
            audience: _config["Jwt:Audience"],
            claims: claims,
            expires: DateTime.Now.AddMinutes(720),
            signingCredentials: credentials);

        var jwt = new JwtSecurityTokenHandler().WriteToken(tokenDescriptor);

        return new TokenCommandResponse
        {
            AccessToken = jwt
        };
    }
}

public class TokenCommandResponse
{
    public string AccessToken { get; set; } = default!;
}
```

Siguiendo mi artículo anterior:

- **Verificación de credenciales**: Utilizamos Identity de ASP.NET para gestionar a los usuarios (aunque tiene más funcionalidades, en este caso solo estamos utilizando esta parte) y sus roles. UserManager proporciona una amplia gama de métodos para la gestión de usuarios, sus contraseñas y sus roles.
- **Generación del JWT**: A partir de los claims generados según el usuario autenticado, creamos el JWT. Este código es prácticamente un "boilerplate" y siempre será el mismo. Lo importante es observar que utilizamos la configuración del archivo appsettings, la misma que se empleará para verificar el JWT al realizar solicitudes.

> Nota 👀: `ForbiddenAccessException` es una excepción personalizada que creamos en publicaciones anteriores, pero la estamos utilizando aquí.

La configuración necesaria que debes agregar es la siguiente:

```json
  "Jwt": {
    "Issuer": "WebApiJwt.com",
    "Audience": "localhost",
    "Key": "S3cr3t_K3y!.123_S3cr3t_K3y!.123"
  }
```

El `Issuer` y el `Audience` no son relevantes en este contexto; son más importantes cuando se utiliza **OpenID Connect**, pero por ahora son requisitos.

La `Key`, en cambio, es crucial, ya que se trata de nuestro _secret_ para la encriptación simétrica.

Sistemas como Identity Server o OpenIddict utilizan la encriptación asimétrica, utilizando RSA y certificados, lo cual es otro tema interesante que podríamos abordar en otro momento.

### AuthController
Para exponer nuestro comando y permitir su uso, utilizaremos este controlador de API:

```csharp
using MediatR;
using MediatrValidationExample.Features.Auth;
using Microsoft.AspNetCore.Mvc;

namespace MediatrValidationExample.Controllers;

[ApiController]
[Route("api/auth")]
public class AuthController : ControllerBase
{
    private readonly IMediator _mediator;

    public AuthController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpPost]
    public Task<TokenCommandResponse> Token([FromBody] TokenCommand command) =>
        _mediator.Send(command);
}
```

De manera general, estamos invocando nuestro comando de la misma manera que lo hemos hecho en artículos anteriores.
## Configuración Final

Ahora que hemos escrito el código para generar JWTs, es hora de completar la configuración de las dependencias y decirle a la Web API que utilice un esquema de autenticación (en este caso, Bearer Tokens).

```csharp
// código omitido...

[Authorize] // <---
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{

// ...código omitido
```

Utilizamos el atributo `[Authorize]` en el controlador para requerir un esquema de autenticación. ASP.NET Core admite uno o varios esquemas de autenticación diferentes. En este caso, estamos utilizando Bearer Tokens, pero es posible combinarlos con autenticación mediante cookies u otras formas según sea necesario. Aunque es común utilizar solo un esquema, en ocasiones puede ser necesario utilizar dos o más, aunque en la mayoría de los casos uno es suficiente.

La configuración se divide en dos partes:

```csharp
// Identity Core
builder.Services
    .AddIdentityCore<IdentityUser>()
    .AddRoles<IdentityRole>()
    .AddEntityFrameworkStores<MyAppDbContext>();
```
En esta sección configuramos todas las dependencias de Identity, como la implementación de `TUser` a utilizar, la de `TRoles` y el contexto que se empleará.

> Nota 👀: Como mencioné anteriormente, Identity Core es un framework de autenticación y autorización completo que ofrece funcionalidades como Claims, Roles, Policies, etc.

```csharp
// Autenticación y autorización
builder.Services
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
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
        };
    });
```
En esta sección, configuramos la autenticación y autorización mediante Bearer Tokens.

> Nota 👀: Si deseas obtener más información sobre este tema, te invito a revisar mis artículos anteriores sobre [JWT](https://dev.to/isaacojeda/aspnet-core-6-autenticacion-jwt-y-identity-core-170i) y [OpenID](https://dev.to/isaacojeda/aspnet-core-servidor-de-autenticacion-con-openid-connect-59kh).

## Actualización de Swagger
La plantilla de Web API por defecto incluye una configuración básica de Swagger. Para probar la autenticación con Bearer Tokens, debemos indicar a Swagger que necesitamos incluir un JWT en el encabezado **Authorization**.

```csharp
builder.Services.AddSwaggerGen(c =>
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
    c.AddSecurityRequirement(new OpenApiSecurityRequirement {
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
```
Esta configuración es un ejemplo básico de Swagger, pero Swashbuckle ofrece muchas más opciones de configuración. Puedes explorarlas según tus necesidades.

## Seed de Usuarios
Anteriormente, ya teníamos un método de Seed para datos de prueba. Ahora, lo actualizamos de la siguiente manera:

```csharp
async Task SeedProducts()
{
    using var scope = app.Services.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<MyAppDbContext>();
    var userManager = scope.ServiceProvider.GetRequiredService<UserManager<IdentityUser>>();

    // código omitido...

    var testUser = await userManager.FindByNameAsync("test_user");
    if (testUser is null)
    {
        testUser = new IdentityUser
        {
            UserName = "test_user"
        };

        await userManager.CreateAsync(testUser, "Passw0rd.1234");
        await userManager.CreateAsync(new IdentityUser
          {
              UserName = "other_user"
          }, "Passw0rd.1234");
    }
}
```

Aquí estamos creando dos usuarios, que serán útiles para las pruebas de autorización que realizaremos más adelante. Ahora estamos listos para probar casi todo 👍🏽.

## Probando la Autenticación
Ejecutamos la aplicación y se abrirá Swagger:
![Imagen de Swagger](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/y6vaiif8l2i5k74gf495.png)
El icono del candado **Authorize** es una configuración adicional que agregamos en el **Program**, lo que permite a Swagger incluir un JWT en las solicitudes.

Desde aquí, solo te queda realizar pruebas. Intenta consultar productos o crearlos, y verás que no podrás hacerlo a menos que estés autenticado con tu usuario y contraseña.

Utiliza el endpoint **/api/auth/** para generar JWTs basados en las credenciales que establecimos en el método Seed. Utiliza el botón Authorize para agregar el JWT en el encabezado **Authorization**:
 
 ![Imagen de Autorización en Swagger](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jkjozbd9lmg2ordc22to.png)

## Agregando Autorización

La autorización comienza siendo sencilla, pero puede volverse más compleja. Por lo general, utilizo la autorización basada en roles, especialmente cuando ya estoy usando Identity.

Realmente, ya tenemos todo configurado, solo necesitas agregar los roles en la base de datos y asignarlos a un usuario para probar.

Actualizamos nuestro método de **Seed** y agregamos lo siguiente al final:
```csharp
// Código omitido
var roleManager = scope.ServiceProvider.GetRequiredService<RoleManager<IdentityRole>>();
var adminRole = await roleManager.FindByNameAsync("Admin");
if (adminRole is null)
{
    await roleManager.CreateAsync(new IdentityRole
    {
        Name = "Admin"
    });
    await userManager.AddToRoleAsync(testUser, "Admin");
}
```

Aquí creamos un rol llamado **Admin** y lo asignamos a nuestro usuario de prueba (test_user), que previamente habíamos consultado en este método.

Los roles en Identity deben registrarse en la base de datos, ya que suelen ser fijos. Es común tenerlos en un método de Seed como este.

La clase `IdentityRole` es la implementación predeterminada de un rol, pero suelo extenderla mediante herencia para agregar más propiedades, como descripción y categoría del rol (aunque esto depende de los requisitos).

La autorización basada en roles tiene como objetivo permitir que solo los usuarios con el rol **Admin** puedan crear productos. Por lo tanto, actualizamos el método de creación de la siguiente manera:

```csharp
  /// <summary>
  /// Crea un producto nuevo
  /// </summary>
  /// <param name="command"></param>
  /// <returns></returns>
  [HttpPost]
  [Authorize(Roles = "Admin")] // <----- Autorización por rol
  public async Task<IActionResult> CreateProduct([FromBody] CreateProductCommand command)
  {
      await _mediator.Send(command);

      return Ok();
  }
```

Nuevamente, utilizamos el atributo `[Authorize]`, pero esta vez especificamos que este método requiere un rol en particular.

Si ejecutas la solución nuevamente, notarás que se ha creado un rol llamado **Admin** en la tabla **AspNetRoles**, y se ha establecido una relación en la tabla **AspNetUserRoles**:
![Imagen de la tabla AspNetRoles](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bfg3fn6iel5en7wlax0n.png)
![Imagen de la tabla AspNetUserRoles](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/y1npe06r1fq6bojm3eh8.png)
Al crear el JWT, revisamos esta relación. Es decir, al autenticar a un usuario, consultamos sus roles para agregarlos al JWT de manera que ASP.NET lo interprete como roles válidos.

Cuando se intenta autorizar a un usuario, ASP.NET verifica los **Claims** del JWT para determinar si tiene acceso o no.

Puedes realizar pruebas con los dos usuarios que hemos creado: **test_user** tiene el rol **Admin**, mientras que **other_user** no lo tiene. Explora con ambos usuarios para ver cómo se comporta la autorización y si funciona correctamente. 

Con esta configuración de autenticación, puedes utilizar expresiones como `User.IsInRole("Admin")` para verificar si el usuario actual tiene un rol específico. Esto es muy útil en la mayoría de los escenarios de autorización.

## Acceso al Usuario Actual

Acceder al usuario actual es una parte importante de la autorización y autenticación en una aplicación. Para lograrlo, necesitamos una forma de acceder al contexto actual del usuario. Sin embargo, es esencial que esta funcionalidad esté desacoplada del contexto web, ya que podríamos necesitar acceder a la funcionalidad de **Application Core** en diferentes contextos, como aplicaciones de consola.

Para lograr esto, creamos una abstracción llamada `ICurrentUserService` que nos permitirá acceder al usuario actual. Su definición es la siguiente:

```csharp
namespace MediatrValidationExample.Services;
public interface ICurrentUserService
{
    CurrentUser User { get; }

    bool IsInRole(string roleName);
}

public record CurrentUser(string Id, string UserName);
```

Aquí estamos definiendo el contrato para acceder al usuario actual. En este momento, solo necesitamos el ID y el nombre de usuario del usuario actual. También estamos abstrayendo la forma en que se determina si un usuario tiene un rol o no. Esto puede ser útil para realizar pruebas unitarias.

La implementación de `ICurrentUserService` se basa en el **HttpContext** y se ve así:

```csharp
using System.Security.Claims;

namespace MediatrValidationExample.Services;
public class CurrentUserService : ICurrentUserService
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public CurrentUserService(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;

        var id = _httpContextAccessor.HttpContext.User.Claims
            .FirstOrDefault(q => q.Type == ClaimTypes.Sid)
            .Value;

        var userName = _httpContextAccessor.HttpContext.User.Identity.Name;

        User = new CurrentUser(id, userName);
    }

    public CurrentUser User { get; }

    public bool IsInRole(string roleName) =>
        _httpContextAccessor.HttpContext!.User.IsInRole(roleName);
}
```

Esta implementación se encuentra fuertemente acoplada al **HttpContext** y está diseñada para entornos de aplicaciones web. `HttpContext.User` se inicializa automáticamente en ASP.NET cuando usamos Bearer Tokens, ya que hemos indicado que se espera un JWT en el encabezado **Authorization**. La función `HttpContext.User.IsInRole` se utiliza para verificar si el usuario tiene un rol, lo que es posible porque hemos incluido los roles en el JWT como Claims.

> Nota 👀: En futuras publicaciones, es posible que dividamos esta aplicación en diferentes proyectos siguiendo un enfoque de arquitectura de segmentación vertical (Vertical Slice Architecture).

### Actualizando AuthController

Actualizamos el controlador de autorización para incluir el uso de `ICurrentUserService`:

```csharp
    [Authorize]
    [HttpGet("me")]
    public IActionResult Me([FromServices] ICurrentUserService currentUser)
    {
        return Ok(new
        {
            currentUser.User,
            IsAdmin = currentUser.IsInRole("Admin")
        });
    }
```

`ICurrentUserService` solo es utilizable si el usuario actual está autenticado; probablemente obtendremos errores si se intenta acceder a `/me` si no hay un JWT 🤭.

> Nota 👀: Antes de ejecutar el proyecto, debemos registrar el servicio como una dependencia con `builder.Services.AddScoped<ICurrentUserService, CurrentUserService>()`.

La respuesta que obtendrás al llamarlo con Swagger será la siguiente:

```json
{
  "user": {
    "id": "308e554d-4251-47f9-9617-726dff6562ef",
    "userName": "other_user"
  },
  "isAdmin": false
}
```

Si pruebas con el usuario Admin, obtendrás:

```json
{
  "user": {
    "id": "f28cf715-2171-4c0e-9ba5-f2bbbb958f63",
    "userName": "test_user"
  },
  "isAdmin": true
}
```

Puedes ver que la función `IsInRole` funciona sin problemas.

Con esto, hemos definido una abstracción que nos permite acceder al usuario actual que realiza la solicitud. No importa si en el futuro decidimos cambiar de una aplicación web a una de consola; la funcionalidad de **Application Core** debería seguir funcionando sin problemas.

> Nota: Esta parte se incluye solo para explicar cómo se podría utilizar `ICurrentUserService`. La propiedad `IsAdmin` también es un ejemplo.

# Conclusión

Hemos agregado autenticación y autorización a nuestra aplicación, la cual hemos construido en estos 5 artículos (hasta ahora). Utilizando ASP.NET Identity Core, hemos simplificado en gran medida la seguridad de usuarios, ya que no es necesario modificar ningún algoritmo de cifrado ni de hash para guardar las contraseñas de manera segura. Identity Core también ofrece funcionalidades adicionales, como la generación de códigos para restablecer contraseñas o confirmar correos electrónicos, pero dejaremos eso para futuras publicaciones.

Espero que esta información te sea útil. Si tienes alguna pregunta, no dudes en contactarme en [Twitter](https://twitter.com/balunatic), y con gusto te ayudaré con cualquier consulta.
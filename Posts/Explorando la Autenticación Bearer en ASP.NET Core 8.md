# Introducción

La autenticación y autorización son aspectos fundamentales en el desarrollo de aplicaciones web, y en la versión .NET 8, se ha simplificado la implementación de autenticación mediante Bearer Tokens. En este artículo, exploraremos cómo aprovechar Identity Core y esta nueva funcionalidad para configurar la autenticación de manera rápida y sencilla en aplicaciones .NET Core.

> Nota :💡 Aquí te dejo el código fuente de este ejemplo: [DevToPosts/IdentityApiAuth at main · isaacOjeda/DevToPosts (github.com)](https://github.com/isaacOjeda/DevToPosts/tree/main/IdentityApiAuth)

# Autenticación por Bearer Tokens en .NET 8 

En la versión .NET 8, se ha facilitado la configuración de autenticación mediante Bearer Tokens. Hasta ahora, los desarrolladores a menudo tenían que realizar esta implementación de forma manual, pero con Identity Core y las nuevas características de .NET 8, es más rápido que nunca tener una solución de autenticación lista para usar.

Para comenzar, necesitaremos los siguientes paquetes (ten en cuenta que las versiones pueden haber cambiado desde la redacción de este artículo):

```xml
<PackageReference Include="Microsoft.AspNetCore.Identity.EntityFrameworkCore" Version="8.0.0-rc.2.23480.2" />
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.0-rc.2.23480.1" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="8.0.0-rc.2.23480.1">
  <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  <PrivateAssets>all</PrivateAssets>
</PackageReference>
<PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
```


Los paquetes utilizados incluyen:

- **Identity.EntityFrameworkCore**: Este paquete permite integrar Identity Core y Entity Framework para gestionar la identidad de los usuarios.
- **EntityFrameworkCore.SqlServer**: Utilizaremos SQL Server como sistema de persistencia para almacenar datos de autenticación.
- **EntityFrameworkCore.Tools**: Proporciona herramientas para generar migraciones y scripts de base de datos.
- **Swashbuckle.AspNetCore**: Este paquete es necesario para implementar Swagger, que facilita la documentación de la API.

Para mantener nuestro ejemplo sencillo, creamos una clase `User` que hereda de `IdentityUser`:

```csharp
using Microsoft.AspNetCore.Identity;

namespace IdentityApiAuth.Models;

public class User : IdentityUser
{
    public string? FirstName { get; set; }
    public string? LastName { get; set; }
    public string? Address { get; set; }
}
```

Además, configuramos el contexto de la base de datos:

```csharp
using IdentityApiAuth.Models;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

namespace IdentityApiAuth.Data;

public class AppDbContext : IdentityDbContext<User>
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
    }
}
```

## Configuración de Dependencias

La configuración de dependencias es esencial para generar tokens y refresh tokens. En este punto, debemos configurar los servicios y endpoints necesarios. Gracias a Identity Core, gran parte de esta configuración ya se encuentra disponible de manera predeterminada:

```csharp
using IdentityApiAuth.Data;
using IdentityApiAuth.Models;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Configuración de Entity Framework Core
builder.Services.AddDbContext<AppDbContext>(options => options
    .UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// Configuración de Autenticación y Autorización
builder.Services
    .AddAuthorization()
    .AddAuthentication()
    .AddBearerToken(IdentityConstants.BearerScheme);

// Configuración de Identity Core
builder.Services
    .AddIdentityCore<User>()
    .AddRoles<IdentityRole>()
    .AddEntityFrameworkStores<AppDbContext>()
    .AddApiEndpoints();

// Configuración de Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
```

Si estamos en un entorno de desarrollo, habilitamos Swagger y su interfaz de usuario para la documentación:

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

Luego, configuramos los middlewares de autenticación y autorización:

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

Por último, definimos los endpoints de la API y las reglas de autorización:

```csharp
app.MapIdentityApi<User>();

app.MapGet("/", () => Results.Redirect("/swagger"));
app.MapGet("/me", (HttpContext httpContext) =>
{
    return new
    {
        httpContext.User.Identity.Name,
        httpContext.User.Identity.AuthenticationType,
        Claims = httpContext.User.Claims.Select(s => new
        {
            s.Type, s.Value
        }).ToList()
    };
})
.RequireAuthorization();
```

## Probando los Endpoints

Para verificar la implementación exitosa de autenticación por Bearer Tokens en nuestra aplicación, utilizaremos el cliente HTTP Client de Visual Studio Code para realizar diversas pruebas en los siguientes endpoints.

### Registro de Usuario

Primero, procederemos a registrar un nuevo usuario en la aplicación. Para ello, haremos una solicitud POST al endpoint `{{host}}/register` con los siguientes datos de usuario:

```json
### New User
POST {{host}}/register
Content-Type: application/json
  
{
  "username": "isaac",
  "password": "Passw0rd.1",
  "email": "isaac.ojeda@mail.com"
}
```

La respuesta exitosa será la siguiente:

```
HTTP/1.1 200 OK 
Content-Length: 0 
Connection: close 
Date: Fri, 20 Oct 2023 23:11:12 GMT 
Server: Kestrel 
Alt-Svc: h3=":5194"; ma=86400
```

En este punto, todas las validaciones se realizan según la configuración de Identity Core. Si intentamos repetir el registro con el mismo nombre de usuario, recibiremos un mensaje de error:

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "DuplicateUserName": [
      "Username 'isaac' is already taken."
    ]
  }
}
```

### Inicio de Sesión

Luego, procederemos al inicio de sesión en la aplicación. Haremos una solicitud POST al endpoint `{{host}}/login` con las credenciales del usuario:

```json
### Login
POST {{host}}/login
Content-Type: application/json
  
{
    "username": "isaac",
    "password": "Passw0rd.1"
}
```

La respuesta será un token de acceso (Bearer Token) que se utilizará en llamadas posteriores:

```json
{
  "token_type": "Bearer",
  "access_token": "CfDJ8BO1QsNIvpdOgb3Usybzpi...",
  "expires_in": 3600,
  "refresh_token": "CfDJ8BO1QsNIvpdOgb3Usybzp..."
}
```

Este token de acceso se debe incluir en el encabezado de autorización (Authorization) en todas las solicitudes que requieran autenticación, como el endpoint `/me`

### Consulta de Datos del Usuario Autenticado

Para obtener información sobre el usuario autenticado, haremos una solicitud GET al endpoint `{{host}}/me`. Esta solicitud debe incluir el Bearer Token en el encabezado de autorización:

```json
GET {{host}}/me
Content-Type: application/json
Authorization: Bearer {{token}}
```

La respuesta contendrá detalles del usuario, como su nombre, tipo de autenticación y claims (reclamaciones):

```json
{
  "name": "isaac",
  "authenticationType": "Identity.Application",
  "claims": [
    {
      "type": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier",
      "value": "35daf905-8445-40fa-8500-32686311035c"
    },
    {
      "type": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name",
      "value": "isaac"
    },
    {
      "type": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
      "value": "isaac.ojeda@mail.com"
    },
    {
      "type": "AspNet.Identity.SecurityStamp",
      "value": "7RONWQSKG3MRMSLN65345U623KOHH4A3"
    },
    {
      "type": "amr",
      "value": "pwd"
    }
  ]
}
```

### Renovación de Token (Refresh Token)

Si necesitas renovar un token de acceso, puedes hacerlo a través del endpoint `/refresh`. Realiza una solicitud POST al endpoint con el token de refresco (refresh token):

```json
### Refresh
POST {{host}}/refresh
Content-Type: application/json
  
{
  "refreshToken": "{{refresh}}"
}
```

La respuesta contendrá un nuevo token de acceso y su respectivo refresh token:

```json
{
  "token_type": "Bearer",
  "access_token": "CfDJ8BO1QsNIvpdOgb3U...",
  "expires_in": 3600,
  "refresh_token": "CfDJ8BO1QsNIvpdOgb3..."
}
```

### Explora Más Endpoints

Para una experiencia completa, te recomendamos explorar la documentación de Swagger o su definición para descubrir todos los endpoints disponibles en esta implementación. Encontrarás una variedad de endpoints útiles que pueden adaptarse a tus necesidades y mejorar la funcionalidad de tu aplicación.

# Conclusión

En este artículo, hemos explorado cómo configurar la autenticación mediante Bearer Tokens en .NET 8 de manera rápida y sencilla. Gracias a Identity Core y las nuevas características de .NET 8, podemos implementar una solución de autenticación eficaz y documentar nuestra API con Swagger. Esta mejora simplifica significativamente el proceso de autenticación y autorización en aplicaciones .NET Core, lo que es beneficioso tanto para desarrolladores como para usuarios.

# Referencias
- [Introducing the Identity API endpoints (andrewlock.net)](https://andrewlock.net/exploring-the-dotnet-8-preview-introducing-the-identity-api-endpoints/)
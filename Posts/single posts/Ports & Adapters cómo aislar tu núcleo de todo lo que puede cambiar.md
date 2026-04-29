# Ports & Adapters: cómo aislar tu núcleo de todo lo que puede cambiar

> *"El núcleo de tu aplicación no debería saber si el correo lo manda SendGrid, Mailgun o un servidor SMTP de Docker levantado a las 2am."*

## Introducción

Cuando empezamos un proyecto nuevo, el instinto natural es conectar todo rápido: llamamos a SendGrid directamente desde el servicio de negocio, usamos `HttpClient` con la URL hardcodeada, leemos la configuración en el lugar menos pensado. Funciona. Por un rato.

El problema aparece cuando necesitas correr las pruebas sin mandar correos reales, o cuando el cliente cambia de proveedor de email a mitad del proyecto, o cuando en desarrollo no tienes acceso al servidor SMTP de producción. De repente, lo que "funcionaba" se convierte en un obstáculo.

El patrón **Ports & Adapters**, introducido por Alistair Cockburn bajo el nombre de *Arquitectura Hexagonal*, propone una idea elegante y poderosa: **el núcleo de tu aplicación define contratos (puertos), y las implementaciones concretas (adaptadores) se enchufan desde afuera**. El core no sabe —ni le importa— quién está del otro lado.

Este artículo nace de mi experiencia trabajando con este patrón en proyectos reales con .NET. No como ejercicio académico, sino como herramienta de trabajo que ha simplificado enormemente el mantenimiento, las pruebas y los despliegues en múltiples ambientes.

## Desarrollo

### El problema concreto: el correo electrónico

Tomemos uno de los casos más comunes: **enviar correos electrónicos**. En un proyecto típico manejamos al menos tres ambientes:

| Ambiente     | Proveedor de email                              |
|--------------|-------------------------------------------------|
| Desarrollo   | Servidor SMTP local en Docker (ej. Mailpit)     |
| QA / Staging | Servidor SMTP interno de la empresa             |
| Producción   | SaaS como SendGrid, con tracking y analytics    |

Si el código de negocio sabe con quién está hablando, tienes un problema: cambiar de proveedor implica tocar el núcleo. Eso va en contra de uno de los principios más valiosos en diseño de software: **aislar lo que cambia de lo que no cambia**.
### El port: define el contrato, no la implementación

Un **puerto** es simplemente una interfaz. Dice *qué* se puede hacer, sin decir *cómo*.

```csharp
public interface IEmailSender
{
    Task SendAsync(EmailMessage message, CancellationToken cancellationToken = default);
}

public record EmailMessage(
    string To,
    string Subject,
    string HtmlBody,
    string? PlainTextBody = null
);
```

Tu lógica de negocio depende únicamente de `IEmailSender`. No sabe nada de SMTP, ni de SendGrid, ni de credenciales. Solo conoce el contrato.

```csharp
public class UserRegistrationService
{
    private readonly IEmailSender _emailSender;

    public UserRegistrationService(IEmailSender emailSender)
    {
        _emailSender = emailSender;
    }

    public async Task RegisterAsync(RegisterUserCommand command)
    {
        // ... lógica de registro ...

        await _emailSender.SendAsync(new EmailMessage(
            To: command.Email,
            Subject: "¡Bienvenido!",
            HtmlBody: $"<h1>Hola, {command.Name}</h1><p>Tu cuenta ha sido creada.</p>"
        ));
    }
}
```

El núcleo queda limpio. Nada de dependencias externas, nada que cambie cuando cambias de proveedor.

### Los adapters: implementaciones intercambiables

Cada ambiente tiene su propio **adaptador**, que implementa el mismo puerto.

**Adaptador SMTP** (para desarrollo con Docker):

```csharp
public class SmtpEmailSender : IEmailSender
{
    private readonly SmtpOptions _options;

    public SmtpEmailSender(IOptions<SmtpOptions> options)
    {
        _options = options.Value;
    }

    public async Task SendAsync(EmailMessage message, CancellationToken cancellationToken = default)
    {
        using var client = new SmtpClient();
        await client.ConnectAsync(_options.Host, _options.Port, SecureSocketOptions.None, cancellationToken);

        var mail = new MimeMessage();
        mail.To.Add(MailboxAddress.Parse(message.To));
        mail.Subject = message.Subject;
        mail.Body = new TextPart(TextFormat.Html) { Text = message.HtmlBody };

        await client.SendAsync(mail, cancellationToken);
        await client.DisconnectAsync(true, cancellationToken);
    }
}
```

**Adaptador SendGrid** (para producción):

```csharp
public class SendGridEmailSender : IEmailSender
{
    private readonly SendGridClient _client;
    private readonly SendGridOptions _options;

    public SendGridEmailSender(IOptions<SendGridOptions> options)
    {
        _options = options.Value;
        _client = new SendGridClient(_options.ApiKey);
    }

    public async Task SendAsync(EmailMessage message, CancellationToken cancellationToken = default)
    {
        var msg = MailHelper.CreateSingleEmail(
            from: new EmailAddress(_options.FromEmail, _options.FromName),
            to: new EmailAddress(message.To),
            subject: message.Subject,
            plainTextContent: message.PlainTextBody,
            htmlContent: message.HtmlBody
        );

        await _client.SendEmailAsync(msg, cancellationToken);
    }
}
```

Cada adaptador es independiente. Si mañana migras de SendGrid a Mailgun, creas un nuevo adaptador y listo. El núcleo no se toca.

### El Factory: construir el adaptador correcto según la configuración

Aquí es donde el patrón cobra su mayor fuerza en la práctica. En lugar de registrar manualmente el adaptador en `Program.cs` y cambiarlo por ambiente, uso un **factory** que lee la configuración y devuelve la implementación adecuada.

**Configuración en `appsettings.json`:**

```json
{
  "Email": {
    "Provider": "SendGrid",
    "SendGrid": {
      "ApiKey": "SG.xxx",
      "FromEmail": "noreply@miapp.com",
      "FromName": "Mi Aplicación"
    },
    "Smtp": {
      "Host": "localhost",
      "Port": 1025
    }
  }
}
```

**Factory para el registro:**

```csharp
public static class EmailServiceExtensions
{
    public static IServiceCollection AddEmailSender(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        var provider = configuration["Email:Provider"] ?? "Smtp";

        services.AddTransient<IEmailSender>(sp =>
        {
            return provider switch
            {
                "SendGrid" => new SendGridEmailSender(
                    sp.GetRequiredService<IOptions<SendGridOptions>>()),

                "Smtp" => new SmtpEmailSender(
                    sp.GetRequiredService<IOptions<SmtpOptions>>()),

                _ => throw new InvalidOperationException($"Proveedor de email no soportado: {provider}")
            };
        });

        services.Configure<SendGridOptions>(configuration.GetSection("Email:SendGrid"));
        services.Configure<SmtpOptions>(configuration.GetSection("Email:Smtp"));

        return services;
    }
}
```

Y en `Program.cs`, limpio y simple:

```csharp
builder.Services.AddEmailSender(builder.Configuration);
```

Cambiar de proveedor es ahora una línea en el `appsettings` del ambiente correspondiente. Sin recompilar, sin tocar el núcleo.

### El decorador: comportamiento sin contaminar los adaptadores

Un patrón que combina perfectamente con esto es el **Decorator**. Imagina que quieres agregar logging a todos los envíos de email, independientemente del proveedor:

> Ya he hablado sobre este patrón en este blog [en este artículo](https://dev.to/isaacojeda/aspnet-core-y-el-patron-decorador-ampliando-la-funcionalidad-de-tus-apis-5jk)

```csharp
public class LoggingEmailSenderDecorator : IEmailSender
{
    private readonly IEmailSender _inner;
    private readonly ILogger<LoggingEmailSenderDecorator> _logger;

    public LoggingEmailSenderDecorator(
        IEmailSender inner,
        ILogger<LoggingEmailSenderDecorator> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task SendAsync(EmailMessage message, CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("Enviando correo a {To} con asunto '{Subject}'", message.To, message.Subject);

        try
        {
            await _inner.SendAsync(message, cancellationToken);
            _logger.LogInformation("Correo enviado correctamente a {To}", message.To);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error al enviar correo a {To}", message.To);
            throw;
        }
    }
}
```

Para enchufar el decorador, puedes usar **Scrutor** (librería NuGet para decoradores en .NET DI) o hacerlo manualmente:

```csharp
// Con Scrutor
services.Decorate<IEmailSender, LoggingEmailSenderDecorator>();

// O manualmente
services.AddTransient<IEmailSender>(sp =>
{
    var inner = /* tu factory aquí */;
    var logger = sp.GetRequiredService<ILogger<LoggingEmailSenderDecorator>>();
    return new LoggingEmailSenderDecorator(inner, logger);
});
```

> También utilizar Keyed Services del DI es útil cuando queremos tener varios decoradores, en mi artículo mencionado anteriormente hablo sobre ello.

El logging aplica a cualquier adaptador. Si mañana agregas un adaptador nuevo, el decorador lo cubre automáticamente. Sin duplicar código.

### El adaptador `Null`: tu mejor aliado en pruebas

Para pruebas unitarias o escenarios donde no quieres efectos secundarios reales, el **Null Object Pattern** aplicado como adaptador es invaluable:

```csharp
public class NullEmailSender : IEmailSender
{
    public Task SendAsync(EmailMessage message, CancellationToken cancellationToken = default)
        => Task.CompletedTask; // No hace nada. Y está bien.
}
```

También puedes tener una versión que almacena los mensajes en memoria para hacer aserciones en las pruebas:

```csharp
public class InMemoryEmailSender : IEmailSender
{
    public List<EmailMessage> SentMessages { get; } = new();

    public Task SendAsync(EmailMessage message, CancellationToken cancellationToken = default)
    {
        SentMessages.Add(message);
        return Task.CompletedTask;
    }
}
```

```csharp
// En tus pruebas
var emailSender = new InMemoryEmailSender();
var service = new UserRegistrationService(emailSender);

await service.RegisterAsync(new RegisterUserCommand("test@example.com", "Isaac"));

Assert.Single(emailSender.SentMessages);
Assert.Equal("test@example.com", emailSender.SentMessages[0].To);
```

Pruebas rápidas, sin dependencias externas, sin configuración extra.

### Más allá del email: el patrón escala a todo

Este mismo enfoque aplica a cualquier dependencia de infraestructura:

- **Almacenamiento de archivos**: `IFileStorage` con adaptadores para `AzureBlobStorage`, `LocalFileSystem`, `S3`
- **Notificaciones push**: `IPushNotificationSender` con adaptadores para Firebase, APNS, etc.
- **SMS**: `ISmsSender` con Twilio en producción, log en desarrollo
- **Caché**: `ICacheProvider` sobre `IMemoryCache` o `IDistributedCache`
- **Pagos**: `IPaymentGateway` con Stripe, Conekta, o un mock para pruebas

El patrón es el mismo: **define el contrato en el núcleo, implementa fuera, conecta con un factory**.

## Conclusión

Ports & Adapters no es un patrón complicado. Su fuerza está en una idea simple: **el núcleo de tu aplicación debería poder existir sin importarle nada del mundo exterior**. No le importa si el correo va por SMTP o SendGrid. No le importa si los archivos están en Azure Blob o en disco local. No le importa si estás en desarrollo, QA o producción.

Lo que me ha resultado más valioso en la práctica es la combinación Factory + Decorador:

- El **Factory** decide qué adaptador construir según la configuración del ambiente.
- El **Decorador** agrega comportamiento transversal (logging, reintentos, métricas) sin contaminar ningún adaptador.
- El **Null/InMemory adapter** hace que las pruebas sean rápidas y deterministas.

Cuando alguien en el equipo dice "hay que cambiar de proveedor de email", la respuesta es: *creamos un adaptador nuevo, actualizamos el appsetting, listo*. Sin miedo, sin regresiones inesperadas, sin refactors dolorosos.

Ese es el tipo de arquitectura que te deja dormir tranquilo en producción.

---

## Referencias y recursos para aprender más

- **Alistair Cockburn — Hexagonal Architecture** (artículo original, 2005)  
  https://alistair.cockburn.us/hexagonal-architecture/

- **Microsoft Docs — Dependency Injection en ASP.NET Core**  
  https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection

- **Scrutor — Decorator support for Microsoft DI**  
  https://github.com/khellang/Scrutor

- **MailKit — Cliente SMTP para .NET**  
  https://github.com/jstedfast/MailKit

- **Mailpit — Servidor SMTP para desarrollo con Docker**  
  https://mailpit.axllent.org/

- **Mark Seemann — Dependency Injection in .NET** *(libro recomendado)*  
  https://www.manning.com/books/dependency-injection-principles-practices-patterns

- **Clean Architecture — Robert C. Martin** *(el contexto más amplio de este patrón)*  
  https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html

*¿Usas este patrón en tus proyectos? ¿Con qué combinaciones de adaptadores has trabajado? Déjame tu experiencia en los comentarios.*
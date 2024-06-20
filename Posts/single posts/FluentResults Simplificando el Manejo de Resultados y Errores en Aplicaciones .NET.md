## Introducción

En el vasto mundo del desarrollo de software en .NET, el manejo de resultados de operaciones puede volverse complejo. Tradicionalmente, se ha dependido en gran medida de excepciones para señalar fallas en las operaciones. Sin embargo, FluentResults surge como una alternativa que transforma la forma en que se gestionan estos resultados.

**FluentResults** es una biblioteca liviana diseñada específicamente para resolver un problema común en el desarrollo de software en .NET. En lugar de utilizar excepciones para manejar fallos en operaciones, esta biblioteca retorna un objeto indicando el éxito o fracaso de una operación.

En esencia, FluentResults ofrece una forma más estructurada y orientada a objetos para representar y procesar los resultados de operaciones. Esto significa que en lugar de lanzar excepciones, se utiliza un objeto `Result` que puede contener tanto mensajes de error detallados como mensajes de éxito, permitiendo un manejo más preciso y elaborado de los resultados.

### Beneficios Clave de FluentResults

- **Contenedor Generalizado:** Funciona en múltiples contextos (ASP.NET MVC/WebApi, WPF, DDD, etc.).
- **Almacena Múltiples Errores:** Permite almacenar varios errores en un solo Resultado.
- **Errores y Éxitos Elaborados:** Capacidad para almacenar objetos de Error y Éxito más detallados en lugar de simples mensajes de error en formato de cadena.
- **Orientado a Objetos:** Diseño de errores y éxitos de forma orientada a objetos.
- **Gestión Jerárquica de Errores:** Almacena la causa raíz con una cadena de errores de manera jerárquica.
- **Compatibilidad Multiplataforma:** Ofrece soporte para .NET Standard, .NET Core, .NET 5+ y .NET Full Framework, lo que facilita su integración en diversas aplicaciones.

### ¿Por Qué Resultados en Lugar de Excepciones?

El _Result Pattern_ para indicar el éxito o fracaso de una operación no es una idea nueva, y se origina en los lenguajes de programación funcional. Con FluentResults, este patrón se implementa en el contexto de .NET/C#, ofreciendo una alternativa sólida al uso extensivo de excepciones para controlar el flujo del programa.

Para profundizar en los beneficios y las mejores prácticas del result pattern, puedes consultar el siguiente artículo [Exceptions for Flow Control de Vladimir Khorikov](https://enterprisecraftsmanship.com/posts/exceptions-for-flow-control/), el cual explora en qué escenarios tiene sentido el uso del patrón Resultado y cuándo no. Además, la lista de [Mejores Prácticas](https://github.com/altmann/FluentResults#samplesbest-practices) y [Recursos Interesantes sobre el Result Pattern](https://github.com/altmann/FluentResults#interesting-resources-about-result-pattern) ofrecen información valiosa para comprender este enfoque en profundidad.

FluentResults no solo simplifica el manejo de resultados, sino que también promueve un código más claro y estructurado al ofrecer una forma más robusta de gestionar los resultados de operaciones.
#### Uso de Excepciones

Las excepciones son ideales para situaciones excepcionales e inesperadas que interrumpen el flujo normal del programa. Algunos casos en los que las excepciones son más adecuadas incluyen:

- **Errores Irrecuperables:** Situaciones donde una operación crítica falla y el programa no puede continuar de manera significativa.
- **Condiciones Inesperadas:** Problemas imprevistos como falta de recursos o errores de lógica.
- **Estructuras Existentes:** Donde el código ya está construido en torno al manejo de excepciones y cambiarlo podría ser costoso o disruptivo.
#### Uso de FluentResults

Por otro lado, FluentResults ofrece una manera estructurada de manejar resultados y errores esperados, lo que puede ser más adecuado en situaciones donde:

- **Errores Esperados:** Cuando la situación de error es predecible y parte del flujo normal del programa.
- **Necesidad de Detalles y Contexto:** Donde se requiere información detallada sobre el error para tomar decisiones o proporcionar retroalimentación.
- **Mejor Control de Flujo:** Para mantener el control del flujo del programa sin interrupciones abruptas.
### Escenarios Híbridos y Buenas Prácticas

En muchos casos, una combinación de ambos enfoques puede ser la mejor estrategia. Utilizar excepciones para manejar condiciones inesperadas y problemas críticos, mientras que FluentResults puede ser valioso para el manejo estructurado de errores predecibles y para mantener un flujo de programa más controlado.

En última instancia, la elección entre FluentResults y excepciones debe basarse en la naturaleza y la gravedad del error, así como en las necesidades específicas de la aplicación y los objetivos de diseño.

A continuación, veremos como puedes usar FluentResults.
#### Creación de un Resultado Exitoso y Fallido

```csharp
// Crear un resultado que indica éxito
Result successResult = Result.Ok();

// Crear un resultado que indica fracaso con un mensaje
Result errorResult = Result.Fail("Este es un mensaje de error");

// Crear un resultado que indica fracaso con un objeto Error personalizado
Result customErrorResult = Result.Fail(new Error("Error personalizado"));
```

#### Uso de Result para Métodos con Valor de Retorno

```csharp
// Método con valor de retorno y uso de Result<T>
public Result<int> Divide(int a, int b)
{
    if (b == 0)
        return Result.Fail<int>("No se puede dividir por cero");

    return Result.Ok(a / b);
}
```

#### Procesamiento de Resultados

```csharp
Result<int> divisionResult = Divide(10, 2);

if (divisionResult.IsSuccess)
{
    int resultValue = divisionResult.Value;
    Console.WriteLine($"El resultado de la división es: {resultValue}");
}
else
{
    IEnumerable<IError> errors = divisionResult.Errors;
    Console.WriteLine($"Hubo errores en la operación:");
    foreach (var error in errors)
    {
        Console.WriteLine($"- {error.Message}");
    }
}
```

#### Creación de Resultados Basados en Condiciones

```csharp
string input = "123";

// Crear un resultado basado en una condición
var parseResult = Result.OkIf(int.TryParse(input, out int number), "Fallo al analizar el número");

if (parseResult.IsSuccess)
{
    int parsedNumber = parseResult.Value;
    Console.WriteLine($"Número analizado con éxito: {parsedNumber}");
}
else
{
    Console.WriteLine($"Error al analizar el número: {parseResult.Errors.First().Message}");
}
```

#### Encadenamiento de Mensajes de Error y Éxito

```csharp
var chainedResult = Result.Fail("Error 1")
    .WithError("Error 2")
    .WithError("Error 3")
    .WithSuccess("Éxito 1");

// Procesamiento del resultado encadenado
if (chainedResult.IsFailed)
{
    IEnumerable<IError> errors = chainedResult.Errors;
    Console.WriteLine("Hubo errores:");
    foreach (var error in errors)
    {
        Console.WriteLine($"- {error.Message}");
    }
}

if (chainedResult.IsSuccess)
{
    IEnumerable<ISuccess> successes = chainedResult.Successes;
    Console.WriteLine("Hubo éxitos:");
    foreach (var success in successes)
    {
        Console.WriteLine($"- {success.Message}");
    }
}
```

### Ejemplo utilizando MediatR

Imaginemos un escenario donde tienes un sistema de gestión de usuarios y quieres usar MediatR para manejar las solicitudes de creación de nuevos usuarios. Vamos a definir un comando, un manejador y cómo usar FluentResults para manejar los resultados.

#### Definición del Comando

```csharp
public class CreateUserCommand : IRequest<Result>
{
    public string UserName { get; set; }
    public string Email { get; set; }
    // Otros campos del usuario...
}
```

#### Manejador de la Solicitud

```csharp
public class CreateUserCommandHandler : IRequestHandler<CreateUserCommand, Result>
{
    public async Task<Result> Handle(CreateUserCommand request, CancellationToken cancellationToken)
    {
        // Lógica para crear un nuevo usuario
        // Supongamos que hay alguna validación aquí...

        if (string.IsNullOrWhiteSpace(request.UserName) || string.IsNullOrWhiteSpace(request.Email))
        {
            return Result.Fail("El nombre de usuario y el correo electrónico son obligatorios.");
        }

        // Simulación de crear el usuario
        // ...

        return Result.Ok();
    }
}
```

#### Uso de MediatR para Enviar la Solicitud

```csharp
var mediator = /* Inyectar Mediator aquí */;
var createUserCommand = new CreateUserCommand
{
    UserName = "UsuarioNuevo",
    Email = "nuevo@usuario.com"
};

var result = await mediator.Send(createUserCommand);

if (result.IsSuccess)
{
    Console.WriteLine("El usuario se creó correctamente.");
}
else
{
    Console.WriteLine("Hubo errores al crear el usuario:");
    foreach (var error in result.Errors)
    {
        Console.WriteLine($"- {error.Message}");
    }
}
```

En este ejemplo, definimos un comando `CreateUserCommand` que representa la solicitud de creación de un usuario. Luego, tenemos un manejador `CreateUserCommandHandler` que implementa la lógica de creación de usuarios y devuelve un resultado usando FluentResults.

Finalmente, en el código de uso, enviamos la solicitud al Mediator y manejamos el resultado obtenido. Esto nos permite manejar de manera clara y estructurada los resultados de la operación de creación de usuarios, ya sea un éxito o un error, y proporciona detalles específicos sobre los problemas encontrados en caso de error.

### Definición de una Clase de Error Personalizada

```csharp
public class EmailValidationError : Error
{
    public EmailValidationError(string email)
        : base($"El correo electrónico '{email}' no es válido.")
    {
        Metadata.Add("Field", "Email");
    }
}
```

Esta clase `EmailValidationError` hereda de `Error` y nos permite personalizar el mensaje de error relacionado con un correo electrónico no válido. Además, agrega metadatos para identificar el campo específico asociado al error.

#### Uso de la Clase de Error en un Escenario

Supongamos que tenemos un servicio que valida la entrada de un formulario de registro de usuarios y queremos utilizar esta clase de error personalizada para identificar problemas con el campo de correo electrónico.

```csharp
public class UserRegistrationService
{
    public Result ValidateUserRegistration(string username, string email)
    {
        var errors = new List<IError>();

        if (string.IsNullOrWhiteSpace(username))
        {
            errors.Add(new Error("El nombre de usuario es obligatorio."));
        }

        if (string.IsNullOrWhiteSpace(email) || !IsValidEmail(email))
        {
            errors.Add(new EmailValidationError(email));
        }

        if (errors.Any())
        {
            return Result.Fail(errors);
        }

        // Lógica adicional de validación o registro de usuario...
        
        return Result.Ok();
    }

    private bool IsValidEmail(string email)
    {
        // Lógica de validación de correo electrónico...
        return /* Verificación de validez de email */;
    }
}
```

En este ejemplo, `UserRegistrationService` contiene un método `ValidateUserRegistration` que toma un nombre de usuario y un correo electrónico como entrada y valida ambos campos. Si encuentra problemas, crea instancias de errores personalizados como `EmailValidationError` y los agrega a una lista de errores. Si no hay errores, devuelve un resultado exitoso (`Result.Ok()`).

Esta implementación nos permite utilizar la clase `EmailValidationError` para representar un error específico relacionado con la validación del correo electrónico, lo que facilita la identificación y el manejo de problemas específicos en la lógica de validación del usuario.

## Conclusión

La integración de FluentResults en tus aplicaciones .NET puede cambiar la forma en que manejas los resultados de operaciones. Al adoptar este enfoque, tu código puede volverse más claro, estructurado y resistente a errores, permitiéndote manejar de manera eficiente los resultados exitosos y los casos de error de manera consistente.

Algunos puntos clave a considerar al trabajar con FluentResults:

1. **Gestión de Resultados Estructurada:** Con FluentResults, puedes representar de manera estructurada tanto los resultados exitosos como los errores, lo que facilita su manipulación y procesamiento.
2. **Clases de Error Personalizadas:** Crear clases de error personalizadas te permite detallar y clasificar errores específicos, agregando contextos y metadatos relevantes para identificar mejor las causas raíz de los problemas.
3. **Integración con MediatR u otros Patrones:** Utilizar FluentResults junto con patrones populares como MediatR te permite manejar de manera efectiva los resultados de solicitudes y respuestas, brindando una mejor experiencia de desarrollo y manejo de errores.
4. **Resistencia a Errores Mejorada:** Al enfocarte en devolver objetos de resultado en lugar de lanzar excepciones, puedes mejorar la resistencia de tu aplicación, controlar el flujo del programa y gestionar las condiciones esperadas sin el uso excesivo de bloques try-catch.

Al adoptar prácticas como el retorno de objetos de resultado en lugar de excepciones, tu código puede volverse más sólido y predecible. Aprovechar las capacidades de FluentResults puede ser un paso significativo para mejorar la calidad y mantenibilidad de tu código en entornos .NET.

## Referencias
- [altmann/FluentResults: A generalised Result object implementation for .NET/C# (github.com)](https://github.com/altmann/FluentResults)
- [Get Rid of Exceptions in Your Code With the Result Pattern (youtube.com)](https://www.youtube.com/watch?v=WCCkEe_Hy2Y&ab_channel=MilanJovanovi%C4%87)
- [Flow Control | ASP.NET 6 REST API Following CLEAN ARCHITECTURE & DDD Tutorial | Part 5 (youtube.com)](https://www.youtube.com/watch?v=tZ8gGqiq_IU&t=7s&ab_channel=AmichaiMantinband)

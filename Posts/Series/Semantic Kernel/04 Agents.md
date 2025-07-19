## 🧠 Crea tu propio agente conversacional con Semantic Kernel

¿Te imaginas construir un asistente conversacional que entienda lenguaje natural y, además, sea capaz de ejecutar funciones reales de tu sistema? Con el **Agent Framework de Semantic Kernel**, eso ya no es solo posible, sino sorprendentemente sencillo.

En este tutorial te voy a mostrar cómo crear un **agente inteligente especializado en gestión de facturas**, diseñado para ayudar a abogados en una notaría pública. Este agente no solo entiende lo que le pides, sino que también puede:

- Consultar el estado de una factura.
- Crear prefacturas automáticamente.
- Ver facturas vencidas.
- Marcar facturas como pagadas.
- Obtener información detallada de un cliente.

Todo esto lo logra invocando funciones reales en C#, conectadas a una base de datos real y expuestas mediante un plugin personalizado.

Durante el artículo veremos paso a paso:

- Cómo configurar Semantic Kernel con Azure OpenAI.
- Cómo definir tus propias funciones con `[KernelFunction]`.
- Cómo crear un `ChatCompletionAgent` con instrucciones personalizadas.
- Cómo ejecutar al agente dentro de una API HTTP.
- Y cómo mantener conversaciones con historial persistente.

Además, el proyecto incluye un cliente web de ejemplo (tipo chat) para que puedas probarlo todo en vivo y adaptar la solución a tus propios casos de uso.

> 🔗 Todo el código está disponible en este repositorio:  
> [github.com/isaacOjeda/DevToPosts/tree/main/SemanticKernelSeries/SemanticKernelLearning04](https://github.com/isaacOjeda/DevToPosts/tree/main/SemanticKernelSeries/SemanticKernelLearning04)

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/c7f8juqe3rrci13y7v6c.png)
## 🧠 ¿Qué es el Agent Framework en Semantic Kernel?

El **Agent Framework** es una de las capacidades más potentes introducidas en las versiones recientes de [Semantic Kernel](https://github.com/microsoft/semantic-kernel): un sistema que permite **crear agentes conversacionales inteligentes** con la capacidad de razonar, planificar e invocar funciones definidas por el desarrollador para lograr un objetivo.

A diferencia de un chatbot tradicional basado en reglas, un agente en Semantic Kernel:

- Comprende instrucciones en lenguaje natural.
- Decide qué acciones tomar en base al contexto disponible.
- Invoca funciones (también llamadas “skills” o “plugins”) que tú defines en código.
- Puede encadenar múltiples pasos de razonamiento para llegar a un resultado.

Este modelo es ideal para construir asistentes especializados con conocimientos de negocio y herramientas personalizadas.

### 🎯 Caso de uso: un asistente de facturación

En este tutorial vamos a construir un ejemplo real usando el Agent Framework: un **asistente conversacional para abogados en una notaría pública**, cuya función es **gestionar facturas** mediante lenguaje natural.

El asistente será capaz de:

- Verificar si una factura está pagada o vencida.
- Crear prefacturas para clientes nuevos.
- Consultar el listado de facturas pendientes de pago.
- Marcar una factura como pagada.
- Obtener información detallada de un cliente y su historial de facturación.

Todo esto será posible gracias a una capa de funciones en C# expuestas al agente mediante anotaciones `[KernelFunction]`, y alimentadas por un servicio de backend que accede a la base de datos.

### 🧬 ¿Por qué usar agentes en lugar de comandos o controladores tradicionales?

Con el Agent Framework:

- El usuario no necesita conocer los comandos disponibles.
- El agente elige por sí mismo la función que debe usar.
- Puedes cambiar o extender el comportamiento del sistema sin modificar el frontend.
- La experiencia es más natural, fluida y adaptable a distintos usuarios.

Esto hace que construir asistentes LLM especializados con Semantic Kernel sea una excelente opción para **interfaces inteligentes** sobre sistemas de negocio existentes, especialmente en entornos con operaciones rutinarias, como la gestión de facturas.

## ⚙️ Requisitos previos y setup del entorno

Antes de construir nuestro asistente de facturación, necesitamos preparar nuestro proyecto con las dependencias necesarias y configurar el acceso a un modelo de lenguaje compatible (en este caso, Azure OpenAI).

Este ejemplo está construido sobre .NET 9, con soporte para SQLite como base de datos local, y aprovecha la integración entre Semantic Kernel y Azure OpenAI.

> 🗂️ **Puedes consultar el código completo y funcional de este proyecto en el repositorio:**  
> 🔗 [github.com/isaacOjeda/DevToPosts/tree/main/SemanticKernelSeries/SemanticKernelLearning04](https://github.com/isaacOjeda/DevToPosts/tree/main/SemanticKernelSeries/SemanticKernelLearning04)  
> Allí encontrarás todos los archivos de configuración, modelos, servicios y ejemplos necesarios para correr este asistente de forma local o integrarlo en tu aplicación.

### 📦 Paquetes necesarios

El Agent Framework se encuentra dentro del paquete `Microsoft.SemanticKernel.Agents.Core`, por lo que debes incluir explícitamente las dependencias en tu archivo `.csproj`.

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.SemanticKernel" Version="1.60.0" />
  <PackageReference Include="Microsoft.SemanticKernel.Agents.Core" Version="1.60.0" />
  <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="9.0.7" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="9.0.0" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="9.0.0" />
  <PackageReference Include="Microsoft.VisualStudio.Azure.Containers.Tools.Targets" Version="1.22.1" />
</ItemGroup>
```

Esto nos da acceso al núcleo de Semantic Kernel, las capacidades de agentes, así como herramientas necesarias para la persistencia y desarrollo web.

### 🧪 Configurando el Kernel

El kernel es el núcleo que conecta el modelo de lenguaje con tus plugins. En este ejemplo, usamos Azure OpenAI para procesar instrucciones en lenguaje natural. A continuación se muestra cómo configurar el kernel con los valores de `appsettings.json`:

```csharp
public static IServiceCollection AddSemanticKernel(this IServiceCollection services, IConfiguration configuration)
{
    var azureOpenAIConfig = configuration.GetSection("AzureOpenAI");

    services.AddAzureOpenAIChatCompletion(
        deploymentName: azureOpenAIConfig["DeploymentName"]!,
        apiKey: azureOpenAIConfig["ApiKey"]!,
        endpoint: azureOpenAIConfig["Endpoint"]!,
        modelId: azureOpenAIConfig["ModelId"]!
    );

    services.AddTransient(serviceProvider => new Kernel(serviceProvider));

    return services;
}
```

La configuración en `appsettings.json` se vería algo así:

```json
"AzureOpenAI": {
  "DeploymentName": "notaria-assistant",
  "ApiKey": "TU_API_KEY",
  "Endpoint": "https://tuservicio.openai.azure.com/",
  "ModelId": "gpt-35-turbo"
}
```

Con esto, el `Kernel` queda disponible vía inyección de dependencias y listo para clonar, personalizar o conectar a nuestros agentes.

### 🧱 Servicios base y modelos de dominio

Nuestro agente necesita acceder a datos del mundo real, por lo que construimos un servicio `InvoiceService` con métodos como:

- `GetInvoiceByNumberAsync`
- `CreateInvoiceAsync`
- `MarkInvoiceAsPaidAsync`
- `GetUnpaidInvoicesAsync`
- `GetCustomerByEmailAsync`

Este servicio será inyectado en el plugin que veremos en la próxima sección, y expone la lógica de negocio que el agente puede usar para resolver instrucciones. Si ya tienes una capa de servicios en tu aplicación, puedes integrarla directamente.

## 🧩 Definir tus funciones (skills/plugins)

Una de las capacidades más poderosas del Agent Framework es permitir que tus agentes ejecuten **funciones reales escritas en C#**. Estas funciones se registran como "plugins" usando decoradores especiales, y luego el agente las puede invocar cuando detecta que son necesarias para cumplir una instrucción del usuario.

En este caso, hemos creado un plugin llamado `InvoicesPlugin`, que expone cinco funciones relacionadas con facturación. Cada una está decorada con el atributo `[KernelFunction]`, lo que la hace visible para Semantic Kernel.

### 🧠 ¿Qué es un plugin?

Un plugin en Semantic Kernel es simplemente una **clase con métodos públicos asincrónicos** decorados con `[KernelFunction]`. Además, puedes usar `[Description]` en los parámetros y métodos para mejorar la comprensión del agente sobre lo que hace cada función.

> 🗂️ El código completo del plugin puede consultarse en el repositorio:  
> [Ver InvoicesPlugin.cs en GitHub](https://github.com/isaacOjeda/DevToPosts/tree/main/SemanticKernelSeries/SemanticKernelLearning04)

### 🧾 Ejemplo 1: Verificar el estado de una factura

```csharp
[KernelFunction]
[Description("Verifica el estado de pago de una factura específica usando su número de factura.")]
public async Task<string> VerifyPaymentAsync(
    [Description("Número de la factura a verificar (ej: INV-202412-0001)")] string numeroFactura)
{
    var invoice = await invoiceService.GetInvoiceByNumberAsync(numeroFactura);

    if (invoice == null)
        return $"❌ No se encontró ninguna factura con el número: {numeroFactura}";

    var result = new StringBuilder();
    result.AppendLine($"📋 **Factura: {invoice.InvoiceNumber}**");
    result.AppendLine($"🔸 Estado: {GetStatusText(invoice.Status)}");
    // ...otros campos visuales

    return result.ToString();
}
```

> 💡 **Nota:**  
> En este ejemplo, todas las funciones del plugin reciben y retornan datos en formato `string`, ya que el enfoque está en mantener el ejemplo simple y fácil de seguir.  
> Sin embargo, el Agent Framework de Semantic Kernel **soporta objetos complejos como parámetros de entrada y salida**. Puedes retornar clases personalizadas, listas u objetos anidados, y el agente será capaz de interpretarlos, formatearlos o incluso razonar sobre ellos en sus respuestas.
> 
> En un escenario real, podrías devolver una `InvoiceDetail` con propiedades estructuradas en lugar de una cadena formateada, lo que abre la puerta a integraciones más ricas o adaptadas al canal (por ejemplo, APIs, UIs o chatbots).

Este método se encarga de buscar una factura por número y devolver su estado, fecha de vencimiento, monto y otra información útil. Como puedes ver, el resultado se formatea para ser legible y amigable, incluyendo emojis para mejorar la experiencia.

### ✍️ Ejemplo 2: Crear una prefactura

```csharp
[KernelFunction]
[Description("Realiza una prefactura para un cliente usando email, descripción, monto y días hasta el vencimiento.")]
public async Task<string> CreateInvoiceDraftAsync(
    string clienteEmail,
    string descripcion,
    decimal monto,
    int diasVencimiento = 30)
{
    var customer = await invoiceService.GetCustomerByEmailAsync(clienteEmail);
    if (customer == null)
        return $"❌ No se encontró ningún cliente con el email: {clienteEmail}";

    var dueDate = DateTime.UtcNow.AddDays(diasVencimiento);
    var invoice = await invoiceService.CreateInvoiceAsync(
        customer.Id, descripcion, monto, dueDate, "Factura generada automáticamente");

    return $"✅ Prefactura creada: {invoice.InvoiceNumber} para {customer.Name}";
}
```

Este método genera una factura en estado "borrador", útil para cuando un abogado quiere adelantar una facturación para un cliente específico.

### 🧠 ¿Por qué usar `[KernelFunction]`?

Al decorar tus funciones con este atributo, estás dando al agente la capacidad de:

- Descubrir qué herramientas tiene disponibles.
- Seleccionar de forma automática la función correcta en base a la intención del usuario.
- Combinar funciones si es necesario (por ejemplo, buscar cliente → generar factura).

Gracias a esto, no necesitas escribir prompts complejos ni reglas condicionales. El agente decide qué hacer en tiempo de ejecución.

## 🤖 Crear un agente y asignarle objetivos

Una vez que tenemos definido nuestro plugin con las funciones necesarias, es hora de **crear un agente** que lo utilice. Un agente es una instancia del tipo `ChatCompletionAgent` que sabe cómo interactuar con un modelo de lenguaje, tiene acceso a un conjunto de funciones y puede operar con base en instrucciones personalizadas.

En este paso lo conectamos todo: el kernel, el plugin, y la intención del asistente.

### 🧠 ¿Qué es un `ChatCompletionAgent`?

El `ChatCompletionAgent` es una implementación del Agent Framework de Semantic Kernel que permite interactuar con modelos de lenguaje compatibles (como GPT-4 o GPT-3.5) de forma conversacional. Este tipo de agente:

- Usa un kernel preconfigurado con modelos y funciones.
- Puede tener instrucciones personalizadas (“persona” o “rol”).
- Decide automáticamente qué función usar, si corresponde.
- Mantiene un contexto conversacional.

### 🏗️ Registrando el agente con DI

En este ejemplo, configuramos el agente como un servicio usando la extensión `AddAgents`. Esto permite que el agente se cree con acceso al kernel y al `InvoicesPlugin`:

```csharp
services.AddKeyedTransient<ChatCompletionAgent>("AssistantAgent", (sp, _) =>
{
    var kernel = sp.GetRequiredService<Kernel>();
    var agentKernel = kernel.Clone();

    // Registramos el plugin de facturas
    var invoiceService = sp.GetRequiredService<InvoiceService>();
    agentKernel.ImportPluginFromObject(new InvoicesPlugin(invoiceService));

    var agent = new ChatCompletionAgent()
    {
        Name = "Asistente",
        Instructions = """
            Eres un asistente de abogados en una notaría pública. 
            Tu tarea es ayudar a los abogados a gestionar y verificar el estado de las facturas.
            
            Funciones disponibles:
            - Verificar el estado de pago de facturas específicas
            - Crear prefacturas para clientes
            - Consultar facturas pendientes de pago
            - Marcar facturas como pagadas
            - Consultar información de clientes

            Siempre proporciona información clara y detallada sobre el estado de las facturas.
            Usa emojis para hacer las respuestas más visuales y fáciles de entender.
        """,
        Kernel = agentKernel,
        Arguments = new KernelArguments(
            new OpenAIPromptExecutionSettings()
            {
                FunctionChoiceBehavior = FunctionChoiceBehavior.Auto()
            })
    };

    return agent;
});
```

> 🧩 Este agente se construye a partir de un `Kernel` clonado. Así puedes tener múltiples agentes con diferentes plugins, instrucciones o configuraciones sin interferencias entre ellos.

### 🗣️ Instrucciones: cómo le decimos al agente quién es

La propiedad `Instructions` permite definir la personalidad, contexto y objetivos del agente. Aquí estamos usando un sistema de "prompt largo" al estilo system message:

```csharp
Instructions = """
    Eres un asistente de abogados en una notaría pública. 
    Tu tarea es ayudar a los abogados a gestionar y verificar el estado de las facturas.
    ...
"""
```

Este mensaje guía al modelo para que entienda el contexto en el que opera, el lenguaje que debe usar y el tipo de resultados esperados.

### ⚙️ Control de funciones automáticas

El fragmento clave para habilitar la selección automática de funciones es:

```csharp
Arguments = new KernelArguments(
    new OpenAIPromptExecutionSettings()
    {
        FunctionChoiceBehavior = FunctionChoiceBehavior.Auto()
    })
```

Esto permite que el modelo analice la intención del usuario y decida si debe invocar alguna función del plugin registrado. No es necesario indicarle explícitamente el nombre de la función: el agente elige por sí mismo en función del mensaje recibido.

Con este paso, ya tenemos un **asistente especializado** corriendo sobre Semantic Kernel, con acceso completo a las funciones del plugin de facturación y configurado para operar en el contexto de una notaría pública.

## 🚀 Ejecutar el agente y manejar conversaciones

Una vez que nuestro agente está registrado y conectado a sus funciones, llega el momento de **ponerlo en acción**. En este ejemplo, creamos una API HTTP que permite a cualquier cliente (como una app web o móvil) interactuar con el asistente de forma conversacional.

Todo el ciclo se orquesta desde un conjunto de endpoints dentro de `AgentEndpoints.cs`.

### 🌐 Endpoint para iniciar una conversación

El primer paso en cualquier interacción es **crear un hilo de conversación**. Esto permite almacenar el historial y mantener contexto entre múltiples mensajes.

```csharp
group.MapPost("/start", async (ConversationService conversationService) =>
{
    var conversationId = await conversationService.CreateConversationAsync();
    return Results.Ok(new { ConversationId = conversationId });
});
```

Este endpoint genera un `ConversationId` único que luego será utilizado en cada mensaje posterior. La conversación se almacena en la base de datos con su historial asociado.

### 💬 Enviar una pregunta al agente

El segundo endpoint es el más importante: recibe un mensaje del usuario y devuelve la respuesta del agente. Lo interesante es que **reconstruye el historial completo** antes de invocar el modelo, lo que permite mantener coherencia y continuidad.

```csharp
group.MapPost("/", async (AskQuestionRequest request,
    [FromKeyedServices("AssistantAgent")] ChatCompletionAgent agent,
    ConversationService conversationService) =>
{
    // ... Obtener historial desde base de datos
    // ... Agregar mensaje del usuario actual
    // ... Crear ChatHistoryAgentThread y lanzar la invocación
});
```

### 📜 Cómo se construye el historial

El historial de conversación se crea a partir de los mensajes previos almacenados en la base de datos:

```csharp
var chatHistory = new ChatHistory();
foreach (var dbMessage in conversation.Messages)
{
    if (dbMessage.Role == "user")
        chatHistory.AddUserMessage(dbMessage.Content);
    else if (dbMessage.Role == "assistant")
        chatHistory.AddAssistantMessage(dbMessage.Content);
}
```

Después se añade el nuevo mensaje del usuario, y se encapsula todo en un `ChatHistoryAgentThread`, que es el tipo de contexto que entiende el agente:

```csharp
chatHistory.AddUserMessage(request.Question);
var thread = new ChatHistoryAgentThread(chatHistory, request.ThreadId);
```

### 🧠 Ejecutar al agente y procesar respuestas

El agente se invoca de forma asíncrona usando `InvokeAsync`, lo que permite recibir múltiples respuestas si el modelo decide dividir su salida (especialmente útil cuando hay invocación de funciones):

```csharp
var responses = agent.InvokeAsync(request.Question, thread);

var allResponses = new List<string>();
await foreach (var agentResponse in responses)
{
    var content = agentResponse.Message.Content ?? string.Empty;
    if (!string.IsNullOrWhiteSpace(content))
    {
        allResponses.Add(content);
    }
}
```

Luego se combinan todas las respuestas en un solo mensaje y se guarda nuevamente en la base de datos:

```csharp
var finalResponse = string.Join("\n\n", allResponses);
await conversationService.AddMessageAsync(request.ThreadId, finalResponse, "assistant");
```

### ✅ Respuesta final

El resultado es una experiencia completamente conversacional, persistente y con funciones automatizadas. El cliente (por ejemplo, una SPA en Blazor o React) solo necesita enviar preguntas con un `ConversationId` y mostrar las respuestas generadas.

### 📌 ¿Por qué usar historial y contexto?

Esto es clave para que el agente pueda entender preguntas como:

> “¿Y esa factura de Juan ya se pagó?”

Sin historial, el modelo no sabría a qué se refiere “esa factura”. Al mantener contexto, el agente puede hacer inferencias y ejecutar funciones con mayor precisión.

Con esto, tenemos todo lo necesario para una experiencia de agente conversacional **real, persistente, funcional y extensible**.


> 📁 **Nota:**  
> Recuerda que el código completo está disponible en el repositorio, incluyendo el contexto de la base de datos y la implementación completa de `ConversationService`, que se encarga de crear conversaciones, almacenar mensajes y reconstruir el historial.
> 
> 🔗 [Ver el proyecto en GitHub](https://github.com/isaacOjeda/DevToPosts/tree/main/SemanticKernelSeries/SemanticKernelLearning04)

## 🧰 Extras, buenas prácticas y posibles extensiones

Hasta este punto ya tienes un agente conversacional funcional, conectado a un modelo de lenguaje, con un plugin personalizado, y persistencia completa de las conversaciones. Sin embargo, el Agent Framework de Semantic Kernel ofrece muchas oportunidades para llevar tu solución al siguiente nivel.

A continuación, te comparto algunas ideas para extender o mejorar tu asistente.

### 🔗 Agentes con múltiples plugins

El `Kernel` puede importar múltiples plugins, no solo uno. Por ejemplo, podrías tener:

- `InvoicesPlugin`: para gestión de facturas.
- `CalendarPlugin`: para gestionar eventos o vencimientos.
- `EmailPlugin`: para enviar notificaciones automáticas.

```csharp
agentKernel.ImportPluginFromObject(new CalendarPlugin(...));
agentKernel.ImportPluginFromObject(new EmailPlugin(...));
```

Esto permite que el mismo agente decida qué herramienta usar dependiendo del mensaje del usuario, sin que tengas que escribir lógica adicional.

### 🧠 Agentes que planifican múltiples pasos

El Agent Framework incluye soporte experimental para **planificación automática**, donde el modelo puede decidir ejecutar múltiples funciones encadenadas para lograr un objetivo complejo.

Por ejemplo:

> “Crea una prefactura para Carlos y márcala como pagada.”

Esto podría generar un plan con dos pasos: `CreateInvoiceDraftAsync` → `MarkInvoiceAsPaidAsync`.  
El agente puede manejar estas secuencias si se configura con `FunctionCalling` en modo `Auto()` y si las funciones están bien descritas.

### 🧪 Testing y depuración

Cuando construyes agentes que llaman funciones reales, es importante:

- Probar con mensajes ambiguos o incompletos para ver cómo reacciona el agente.
- Monitorear los logs de funciones invocadas.
- Verificar qué parámetros está eligiendo el modelo.
- Utilizar `FunctionResult` para obtener metadata sobre las llamadas.

También puedes capturar los mensajes generados por el modelo para analizarlos más adelante, por ejemplo, almacenando las decisiones o planes generados en una tabla de auditoría.

### ⚠️ Cuida la experiencia conversacional

Algunos consejos prácticos para que el asistente se sienta más natural:

- Usa emojis y lenguaje cercano si el público lo permite (como hicimos aquí).
- Asegúrate de que las respuestas siempre incluyan lo esencial, incluso si no hay datos (ej: “no hay facturas vencidas”).
- Controla el tamaño del historial para evitar prompts excesivamente largos.
- Maneja errores con mensajes claros: si una factura no existe, si el email es inválido, etc.

### 🧩 Integración con interfaces visuales

Aunque este ejemplo se enfoca en la API HTTP y el backend del agente, la interacción no termina ahí. Puedes conectar este agente conversacional a diferentes interfaces gráficas según tu contexto:

- Una aplicación web (por ejemplo, Blazor, React, Angular).
- Un chatbot embebido en tu intranet o sitio público.
- Clientes móviles nativos.
- Canales como Microsoft Teams, Telegram o WhatsApp.

> 💬 **Importante:**  
> El repositorio del proyecto incluye un **chat funcional ya implementado** que se comunica con esta API y permite mantener conversaciones con el agente.  
> Este cliente está disponible como referencia y punto de exploración: puedes ver cómo se estructuran los hilos, cómo se muestra el historial y cómo se consume el endpoint `/api/agent`.

🔗 [Ver el repositorio en GitHub](https://github.com/isaacOjeda/DevToPosts/tree/main/SemanticKernelSeries/SemanticKernelLearning04)

Este ejemplo es ideal para experimentar con el flujo completo: desde el frontend que envía preguntas, hasta el agente que responde y persiste el historial.

### 🌱 ¿Qué más podrías construir con este enfoque?

El Agent Framework es ideal para automatizar tareas repetitivas en sistemas internos. Algunos ejemplos reales donde este enfoque encaja:

- Asistentes de soporte técnico interno (consultar errores, reiniciar servicios).
- Asistentes legales que redactan contratos o extraen cláusulas.
- Agentes que combinan datos de distintas fuentes (facturas + clientes + CRM).
- Robots de backoffice que ayudan a generar informes o consolidar información.

> 📌 **Consejo final:**  
> Piensa en tu agente como un _colega digital_: cuantas más herramientas le des, más tareas podrá resolver sin que tú tengas que intervenir manualmente.


## ✅ Conclusión

El **Agent Framework de Semantic Kernel** abre una puerta poderosa para construir aplicaciones inteligentes que combinan lenguaje natural con lógica de negocio real. En este artículo, creamos un **agente conversacional especializado en facturación**, capaz de razonar sobre peticiones del usuario e invocar funciones de una aplicación en .NET.

Hemos visto cómo:

- Configurar un `Kernel` con Azure OpenAI.
- Exponer funciones reales como plugins usando `[KernelFunction]`.
- Construir y personalizar un agente con instrucciones específicas.
- Mantener conversaciones persistentes con historial contextual.
- Integrar todo en una API funcional, lista para usarse desde un cliente.

Este enfoque es aplicable a muchísimos casos reales: desde automatización interna hasta asistentes legales, técnicos o administrativos.

> 💡 Lo mejor de todo es que puedes extender esta base fácilmente: agregar nuevos plugins, integrar fuentes de datos adicionales, o escalar hacia canales como webchat, Teams o bots en producción.

Si te interesa explorar más sobre cómo usar Semantic Kernel en escenarios del mundo real, o quieres construir asistentes más sofisticados, este es apenas el comienzo.

🔗 **Recuerda que el código completo está disponible aquí:**  
[github.com/isaacOjeda/DevToPosts/tree/main/SemanticKernelSeries/SemanticKernelLearning04](https://github.com/isaacOjeda/DevToPosts/tree/main/SemanticKernelSeries/SemanticKernelLearning04)

## 📚 Referencias

Si quieres seguir explorando el Agent Framework de Semantic Kernel y todas sus posibilidades en C#, aquí tienes enlaces oficiales de documentación que complementan lo visto en este artículo:

- [Semantic Kernel Agent Framework | Microsoft Learn](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/?pivots=programming-language-csharp)
- [Semantic Kernel Agent Architecture | Microsoft Learn](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-architecture?pivots=programming-language-csharp)
- [The Semantic Kernel Common Agent API surface | Microsoft Learn](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-api?pivots=programming-language-csharp)
- [Configuring Agents with Semantic Kernel Plugins. | Microsoft Learn](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-functions?pivots=programming-language-csharp)
- [Exploring the Semantic Kernel ChatCompletionAgent | Microsoft Learn](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/agent-types/chat-completion-agent?pivots=programming-language-csharp)
- [Develop AI Agents on Azure - Training | Microsoft Learn](https://learn.microsoft.com/en-us/training/paths/develop-ai-agents-on-azure/?source=learn)

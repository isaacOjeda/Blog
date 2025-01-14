### **1. Introducción**

En este tutorial, aprenderás cómo usar **Semantic Kernel** para integrar modelos de lenguaje avanzados y construir un servicio REST que resuma texto. Utilizaremos **Ollama** como motor local de modelos de lenguaje, lo que nos permitirá evitar el uso de servicios en la nube en esta fase de desarrollo.

El objetivo principal es configurar un entorno de trabajo funcional en C# que permita a los desarrolladores:

1. Conectar un modelo de lenguaje local (_llama3.2_).
2. Crear un kernel para manejar habilidades personalizadas.
3. Exponer la funcionalidad como un endpoint REST en una API ASP.NET Core.

Al finalizar, tendrás un servicio de API que podrá recibir texto como entrada y devolver un resumen en una sola oración. Esto no solo te ayudará a entender cómo usar Semantic Kernel, sino que también te dará una base sólida para construir aplicaciones prácticas basadas en IA generativa.

### **2. Introducción a Semantic Kernel**

#### **2.1. ¿Qué es Semantic Kernel?**

**Semantic Kernel** es un framework desarrollado por Microsoft para integrar modelos de lenguaje avanzados en aplicaciones. Permite a los desarrolladores conectar diferentes motores de IA generativa, definir habilidades personalizadas mediante prompts y construir flujos complejos de manera programática.

En este tutorial, usaremos Semantic Kernel junto con **Ollama**, un motor local de modelos de lenguaje, para crear y gestionar una habilidad de generación de texto que resuma contenido.

#### **2.2. Componentes Claves**

1. **Kernel**:  
    El núcleo del framework que conecta el modelo de IA con la lógica de la aplicación. En este caso, lo configuramos para usar Ollama como motor de generación de texto.
2. **Skills** (_Habilidades_):  
    Son capacidades específicas que se pueden programar en el kernel. Aquí definimos una habilidad básica para resumir texto, utilizando prompts personalizados.
3. **Text Generation Service**:  
    Una abstracción que maneja las interacciones con el modelo de lenguaje. Se utiliza para enviar el texto de entrada y obtener la respuesta generada por el modelo.
4. **Ollama**:  
    Una solución local para ejecutar modelos de lenguaje, que elimina la necesidad de servicios en la nube y facilita el desarrollo en entornos locales. En el código, configuramos **Ollama** para usar el modelo `llama3.2`.

### **3. Desarrollo del Ejemplo**

En esta sección, construiremos un API REST que utiliza **Semantic Kernel** y el modelo de lenguaje **llama3.2** de **Ollama** para resumir texto en una sola oración. El objetivo es entender cómo configurar el entorno y crear una habilidad básica que pueda ser utilizada desde cualquier aplicación a través de un endpoint.

#### **3.1. Configuración con Aspire**

Antes de entrar en los detalles del API, configuramos el entorno utilizando **Aspire**. Este framework facilita la construcción de aplicaciones distribuidas, simplificando la gestión de dependencias y servicios.

Lo que he hecho para simplificar la configuración, es utilizar Visual Studio para crear la solución ejemplo de Aspire con .NET 9:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9av06ysmd1bwoiyyj7is.png)

Esta plantilla, también incluye un proyecto de Blazor, en este tutorial no lo utilizaremos, pero al finalizar, puedes utilizar ese proyecto para consumir los endpoints realizados y ya tener una aplicación funcional.

El código de Aspire tiene los siguientes propósitos:

- **Configurar Ollama:** Define el modelo `llama3.2` como el motor de lenguaje. Esto nos permite trabajar con un modelo local, eliminando la necesidad de infraestructura compleja en desarrollo.
- **Modularidad:** Cada proyecto (como `SemanticKernelLearning01_ApiService`) se configura como un módulo independiente, permitiendo una estructura clara y extensible.

**Nota:** Usar Ollama local es ideal para pruebas y desarrollo, ya que no requiere configurar una infraestructura en la nube. Sin embargo, en producción puedes cambiar a OpenAI o Azure OpenAI sin modificar el código, aprovechando la flexibilidad de Semantic Kernel.

#### **Código de Aspire:**

Primero necesitamos los siguientes paquetes:

```xml
  <ItemGroup>
    <PackageReference Include="Aspire.Hosting.AppHost" Version="9.0.0" />
    <PackageReference Include="CommunityToolkit.Aspire.Hosting.Ollama" Version="9.1.0" />
  </ItemGroup>
```

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var ollama =
    builder
        .AddOllama("ollama")// <-- Utilizará docker para usar ollama y sus modelos 
        .WithDataVolume()   // <-- Volumen de Docker para persistir modelos descargados
        .WithOpenWebUI();   // <-- UI Estilo ChatGPT

// Descarga el modelo llama3.2 con nombre "llama"
var llamaModel = ollama.AddModel("llama", "llama3.2");

// Nuestra API depende de Ollama, por lo que se referencia y espera a que esté listo
builder.AddProject<Projects.SemanticKernelLearning01_ApiService>("apiservice")
    .WithReference(llamaModel)
    .WaitFor(llamaModel);

builder.Build().Run();
```

Aquí configuramos Ollama como motor de generación de texto y vinculamos el modelo `llama3.2`. También iniciamos el servicio API como un módulo dentro del proyecto.

#### **3.2. Configuración del API REST**

En el siguiente bloque de código, configuramos un API REST utilizando **ASP.NET Core** y **Semantic Kernel**. Aquí se define un endpoint que acepta texto como entrada y devuelve su resumen generado por el modelo de lenguaje.

**Puntos clave del código:**

1. **Kernel y Ollama:** Se inicializa el kernel de Semantic Kernel y se conecta con Ollama usando la configuración definida en Aspire.
2. **Habilidad de Resumen:** La lógica para resumir texto se encapsula en un prompt que el modelo interpreta para generar respuestas.
3. **Exposición del Endpoint:** Se define un endpoint `/api/summarizer` que recibe un objeto JSON con el texto a resumir y devuelve el resultado.

#### **Código del API REST:**

Paquetes necesarios:

```xml
  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="9.0.0" />
    <PackageReference Include="Microsoft.SemanticKernel" Version="1.33.0" />
    <PackageReference Include="Microsoft.SemanticKernel.Connectors.Ollama" Version="1.33.0-alpha" />
  </ItemGroup>
```

**Program.cs**

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();
builder.Services.AddProblemDetails();
builder.Services.AddOpenApi();

var kernel = builder.Services.AddKernel();

var (endpoint, modelId) = GetOllamaConnectionString();

#pragma warning disable SKEXP0070
kernel.AddOllamaTextGeneration(modelId, endpoint); // Está en alpha, puede cambiar.
#pragma warning restore SKEXP0070

var app = builder.Build();

app.UseExceptionHandler();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.MapDefaultEndpoints();

app.MapPost("/api/summarizer", async (
    TextCompletionRequest request,
    ITextGenerationService textGenerationService) =>
{
    var prompt = $"""
                 Summarize the following text in one sentence:  

                 {request.Text}
                 """;

    var response = await textGenerationService.GetTextContentsAsync(prompt);

    return response;
});

app.Run();

```

#### **3.3. Explicación General del Flujo**

1. **Creación del Kernel:**  
    Se configura el kernel para usar Ollama como motor de generación de texto con el modelo `llama3.2`.
2. **Definición del Endpoint:**  
    El endpoint `/api/summarizer` recibe un JSON con una propiedad `Text`.
3. **Generación del Resumen:**  
    El texto recibido se envía al modelo junto con un prompt que instruye al modelo para resumirlo en una oración.
4. **Devolución del Resultado:**  
    El resumen generado se devuelve como respuesta al cliente que llamó al API.

Código restante:

```csharp
(Uri endpoint, string modelId) GetOllamaConnectionString()
{
    // Este ConnectionString es establecida por Aspire
    var connectionString = builder.Configuration.GetConnectionString("llama");

    var connectionBuilder = new DbConnectionStringBuilder
    {
        ConnectionString = connectionString
    };

    Uri endpoint = new Uri(connectionBuilder["Endpoint"].ToString());
    string modelId = connectionBuilder["Model"].ToString();

    return (endpoint, modelId);
}

// Simple DTO
public record TextCompletionRequest(string Text);
```

> Nota 💡: Recuerda que siempre puedes descargar el código desde desde este [Repositorio](https://github.com/isaacOjeda/DevToPosts/tree/main/SemanticKernelSeries/SemanticKernelLearning01)
> 

#### **3.4 Probando la solución** 

Si corremos el proyecto de Aspire, automáticamente orquestará lo necesario para que la API pueda comunicarse con ollama:


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tv7ykzjwjcw13l0q4l7h.png)

Su primera ejecución será lenta, ya que tendrá que descargar el modelo `llama3.2`.

Existen muchos modelos que podemos ejecutar, tanto para Text Generation y para Embeddings (revisado en próxmos post), puedes revisar más aquí: [Ollama](https://ollama.com/).

Probemos el endpoint (utilizando Visual Studio y un archivo .http):

```
@host = https://localhost:7384

### Text Generation
POST {{host}}/api/summarizer
Content-Type: application/json

{
    "text": "Mariana found an ancient book in her grandfather’s library, with a bookmark pointing to a page that read, “Recite these words and your destiny will change.” Intrigued, she read it out loud, and in an instant, she found herself in a bustling market in a medieval town. \rNo one seemed surprised by her modern clothing; in fact, one merchant greeted her as if he knew her. As she searched for answers, an old man explained that she was the heir to a lost legacy and had to make a choice: stay and lead a kingdom on the brink of chaos, or return to her everyday life. \rWith the book in her hands, Mariana took a deep breath and closed her eyes, choosing the challenge she had secretly always wanted."
}

```

Respuesta:

```json
[
  {
    "text": "Mariana discovers an ancient book that transports her to a medieval town, where she learns she is the heir to a lost legacy and must make a choice between returning to her ordinary life or leading a kingdom on the brink of chaos.",
    "modelId": "llama3.2"
  }
]
```

La ejecución de modelos con ollama claro que suele ser más lento, para acelerar su ejecución debemos de contar con una GPU y de preferencia Nvidia. 

Para operaciones sencillas, me ha funcionado bien y como vemos, la respuesta ha funcionado, todo ejecutandose en nuestra computadora.
### **4. Conclusión**

En este tutorial, exploramos los fundamentos de **Semantic Kernel** y cómo integrarlo con **Ollama** para construir un servicio funcional capaz de generar resúmenes de texto. A través de este ejercicio:

- Aprendiste a configurar un entorno básico utilizando **Aspire**, simplificando la administración de servicios distribuidos.
- Descubriste cómo conectar Semantic Kernel con un motor de lenguaje local, como **llama3.2** de Ollama, para evitar complicaciones de infraestructura en desarrollo.
- Implementaste un API REST con un endpoint práctico que utiliza generación de texto como una habilidad central.

Este ejercicio muestra lo versátil que puede ser Semantic Kernel para crear aplicaciones inteligentes que aprovechan los modelos de lenguaje. Además, la flexibilidad de la arquitectura permite cambiar fácilmente entre diferentes proveedores de modelos, como OpenAI o Azure, sin necesidad de modificar el código base.

Este tutorial es solo el inicio. A partir de aquí, puedes explorar y agregar nuevas habilidades, integrar bases de datos vectoriales para contextualizar las respuestas o expandir el API con capacidades adicionales.

### **5. Próximos Pasos**

Ahora que ya tienes una base sólida para trabajar con **Semantic Kernel** y **Ollama**, considera avanzar con los siguientes temas para expandir tus conocimientos:

1. **Ampliar tus habilidades en Semantic Kernel:**
    - Aprende a crear habilidades más complejas con múltiples pasos y dependencias.
    - Explora cómo usar conectores para integrar datos externos, como archivos, APIs, o bases de datos.
2. **Incorporar contexto mediante bases de datos vectoriales:**
    - Experimenta con herramientas como Qdrant o Pinecone para agregar contexto a las respuestas basadas en vectores.
    - Usa embeddings para conectar preguntas con datos relevantes en tiempo real.
3. **Migrar a un entorno de producción:**
    - Configura la misma API para usar OpenAI o Azure OpenAI como motores de generación de texto.
    - Aprende a manejar credenciales y seguridad en aplicaciones en producción.
4. **Construir una interfaz gráfica:**
    - Usa **Blazor** para crear una interfaz interactiva donde los usuarios puedan interactuar con tus habilidades de Semantic Kernel.
5. **Desarrollar tutoriales más avanzados:**
    - Enseña a otros cómo resolver problemas reales, como análisis de sentimientos, clasificación de texto o generación de contenido adaptado al contexto.

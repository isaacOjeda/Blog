## **Introducción**

En esta tercera parte de nuestra serie sobre **Semantic Kernel**, nos adentramos en la integración de **Embeddings** y **Retrieval-Augmented Generation (RAG)** para mejorar la generación de contenido y la recuperación de información. Utilizando herramientas como **Ollama** para la creación de embeddings y **Qdrant** como vector store, podemos construir sistemas más inteligentes capaces de generar respuestas más precisas basadas en datos externos y actualizados, en lugar de depender únicamente de los datos con los que fue entrenado el modelo. A través de esta configuración, aprovechamos la sinergia entre el poder de los modelos de lenguaje y la capacidad de búsqueda semántica para proporcionar soluciones más efectivas a las consultas de los usuarios. En este artículo, exploraremos cómo implementar estas tecnologías en **Semantic Kernel**, con ejemplos prácticos para configurar los servicios y gestionar los embeddings de manera eficiente.

### **¿Qué son los Embeddings?**

Los **embeddings** son representaciones numéricas de texto en un espacio vectorial de alta dimensión. Estas representaciones permiten medir la similitud semántica entre palabras, frases o documentos completos. En el contexto de **Semantic Kernel**, los embeddings se utilizan para mejorar la búsqueda de información y la generación de respuestas basadas en conocimiento almacenado.

### **Características clave de los Embeddings:**

- Capturan el significado semántico del texto.
- Permiten encontrar contenido relacionado aunque no se utilicen exactamente las mismas palabras.
- Se almacenan en bases de datos especializadas llamadas **vector stores** (como **Qdrant**).
- Se generan con modelos de lenguaje avanzados, como los proporcionados por **Ollama** u **Open AI**.

### **¿Qué es RAG (Retrieval-Augmented Generation)?**

**Retrieval-Augmented Generation (RAG)** es una técnica que combina la recuperación de información con la generación de respuestas mediante un modelo de lenguaje. En lugar de confiar solo en los datos con los que fue entrenado el modelo, RAG permite buscar información relevante en bases de conocimiento externas y utilizarla para generar respuestas más precisas y actualizadas.

#### **Funcionamiento de RAG en Semantic Kernel:**

1. **Consulta del usuario**: Un usuario hace una pregunta o solicitud.
2. **Búsqueda semántica**:
    - Se generan embeddings de la consulta.
    - Se comparan con los embeddings almacenados en el vector store (Qdrant) para recuperar información relevante.
3. **Generación de respuesta**:
    - La información recuperada se pasa como contexto al modelo de lenguaje.
    - Se genera una respuesta más precisa y fundamentada en los datos encontrados.

### **Beneficios de RAG en Semantic Kernel:**

- **Mejor comprensión de consultas**: Permite obtener respuestas relevantes incluso si el usuario no usa palabras exactas.
- **Conocimiento actualizado**: Se pueden agregar nuevos datos al sistema sin necesidad de reentrenar el modelo de IA.
- **Optimización del procesamiento**: Reduce el costo computacional al recuperar solo la información relevante en lugar de analizar todo el corpus de datos.

## **Configurando Semantic Kernel con Embeddings y un Vector Store**

En esta sección, configuraremos **Semantic Kernel** para trabajar con **Embeddings** y un **Vector Store**, utilizando **Aspire** para orquestar los servicios.

Los servicios principales que vamos a configurar son:

- **llama3.2**: Modelo generador de texto.
- **nomic-embed-text**: Modelo generador de embeddings.  
- **Qdrant**: Almacenamiento y recuperación de embeddings.  
- **Aspire**: Orquestación y despliegue de los servicios.

### **Configuración de Aspire**

Aspire nos permite definir y gestionar los servicios necesarios en un solo archivo de configuración. En este caso, configuramos **Qdrant** como nuestro vector store y **llama3.2** y **nomic-embed-text** como proveedor de embeddings.

> Nota 💡: Esto ya lo hemos visto en los otros posts, siempre usa como referencia el código fuente ([DevToPosts/SemanticKernelSeries/SemanticKernelLearning03 at main · isaacOjeda/DevToPosts](https://github.com/isaacOjeda/DevToPosts/tree/main/SemanticKernelSeries/SemanticKernelLearning03)) para más información.

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Configuración de Qdrant
var qdrant = builder.AddQdrant("qdrant")
    .WithLifetime(ContainerLifetime.Persistent);

// Configuración de Ollama
var ollama =
    builder.AddOllama("ollama")
        .WithDataVolume()
        .WithOpenWebUI();

// Modelos de Ollama
var llamaModel = ollama.AddModel("llama", "llama3.2");
var embedding = ollama.AddModel("embed-text", "nomic-embed-text");

// Configuración del API Service con referencias a los modelos y Qdrant
var apiService = builder.AddProject<Projects.SemanticKernelLearning03_ApiService>("apiservice")
    .WithReference(llamaModel)
    .WithReference(embedding)
    .WithReference(qdrant);

// Construcción y ejecución de Aspire
builder.Build().Run();
```

-  **Qdrant** se configura como un servicio persistente.
	- Se hace de esta forma para que no sea tan lento de inicializar en cada inicio de Aspire.
-  **Ollama** se configura con almacenamiento de datos y una interfaz web.  
	-  Se añaden los modelos **"llama3.2"** para generación de texto y **"nomic-embed-text"** para embeddings.  
- Por último se agrega nuestra API en ASP.NET Core.
### **Configuración de Semantic Kernel con Aspire**

Aspire se encargará de inyectar las **cadenas de conexión** en la API, según los nombres definidos en la configuración.

Para hacer que **Semantic Kernel** utilice correctamente los servicios orquestados por Aspire, debemos procesar estas cadenas de conexión. Como lo hemos hecho en ejemplos anteriores, parsearemos la información recibida y estructuraremos mejor nuestra configuración.

Para lograr esto, **separaremos la configuración de dependencias en un archivo específico**, lo que nos permitirá mantener un código más limpio y modular.

#### **Creando `DependencyConfig.cs` para gestionar las dependencias**

En este archivo, crearemos el método de extensión `AddSemanticKernel`, el cual:

- Obtiene las cadenas de conexión inyectadas por Aspire.  
- Extrae los detalles necesarios de **Ollama** (para generación de texto y embeddings).  
- Extrae la información de conexión a **Qdrant** (Vector Store).  
- Registra los servicios de **Semantic Kernel** con las dependencias configuradas.  
- Agrega un servicio de **búsqueda semántica** (`ITextSearch`) que permite realizar búsquedas en una colección de vectores, el cual usaremos más adelante.

Aquí está el código en `DependencyConfig.cs`:

```csharp
public static class DependencyConfig
{
    public static void AddSemanticKernel(this WebApplicationBuilder builder)
    {
        var (endpoint, completionModel) =
            GetModelDetailsFromConnectionString(builder.Configuration.GetConnectionString("llama")!);
        var (_, embeddingModel) =
            GetModelDetailsFromConnectionString(builder.Configuration.GetConnectionString("embed-text")!);
        var (host, port, apiKey) =
            GetQdrantDetailsFromConnectionString(builder.Configuration.GetConnectionString("qdrant")!);

        builder.Services.AddKernel()
            .AddOllamaChatCompletion(completionModel, endpoint)
            .AddOllamaTextEmbeddingGeneration(embeddingModel, endpoint)
            .AddQdrantVectorStore(host: host, port: port, apiKey: apiKey);

        builder.Services.AddKeyedTransient<ITextSearch>(BlogPost.VectorName, (sp, _) =>
        {
            var vectorStore = sp.GetRequiredService<IVectorStore>();
            var embeddingGenerationService = sp.GetRequiredService<ITextEmbeddingGenerationService>();

            var blogposts = vectorStore.GetCollection<Guid, BlogPost>(BlogPost.VectorName);

            blogposts.CreateCollectionIfNotExistsAsync().ConfigureAwait(true);

            var textSearch = new VectorStoreTextSearch<BlogPost>(blogposts, embeddingGenerationService);

            return textSearch;
        });
    }
}
```

**Explicación del código**

1. **Obtenemos los detalles de conexión**    
    - `GetModelDetailsFromConnectionString` extrae el modelo y endpoint desde las cadenas de conexión de **Ollama** (chat y embeddings).
    - `GetQdrantDetailsFromConnectionString` extrae el **host, puerto y API key** de Qdrant.
2. **Registramos los servicios de Semantic Kernel**
    - `AddOllamaChatCompletion` para generación de texto con **Ollama**.
    - `AddOllamaTextEmbeddingGeneration` para generación de **embeddings**.
    - `AddQdrantVectorStore` para almacenar y recuperar datos desde **Qdrant**.
3. **Configuramos un servicio de búsqueda semántica (`ITextSearch`)**
    - Se obtiene la colección de **vectores** asociada a `BlogPost`.
    - Si la colección no existe, se **crea automáticamente** en Qdrant.
    - Se inicializa un objeto `VectorStoreTextSearch<BlogPost>`, permitiendo realizar **búsquedas semánticas** en la colección.

## **Almacenando Embeddings en Qdrant**

Para almacenar datos en Qdrant, primero debemos definir nuestro modelo de datos que será indexado en la base de datos vectorial.  
En este ejemplo, queremos guardar una serie de posts y realizar búsquedas semánticas sobre su contenido.


> Advertencia ⚠️: Las búsquedas vectoriales que estamos viendo aquí están en **Preview (Alpha)**, por lo que podrían cambiar en el futuro. Actualmente, **no se recomienda su uso en producción**, a menos que estés dispuesto a evolucionar junto con el framework.
> Anteriormente, se utilizaban los **Memory de Semantic Kernel** para búsquedas similares, pero estos están en proceso de ser considerados **"Legacy"**, por lo que su uso a largo plazo podría no ser viable.

### **Definición del modelo `BlogPost`**

El modelo `BlogPost` representará las entradas del blog en nuestra base de datos vectorial.

```csharp
public class BlogPost
{
    public const string VectorName = "blogposts";

    [VectorStoreRecordKey]
    [TextSearchResultLink]
    public Guid BlogPostId { get; set; }

    [VectorStoreRecordData(IsFilterable = true)]
    [TextSearchResultName]
    public string Title { get; set; }

    [VectorStoreRecordData(IsFullTextSearchable = true)]
    [TextSearchResultValue]
    public string Description { get; set; }

    [VectorStoreRecordVector(768, DistanceFunction.DotProductSimilarity, IndexKind.Hnsw)]
    public ReadOnlyMemory<float>? DescriptionEmbedding { get; set; }

    [VectorStoreRecordData(IsFilterable = true)]
    public string[] Tags { get; set; }
}
```

**Atributos de `BlogPost`**:

- **`BlogPostId`**: Identificador único del post. Se marca como clave primaria con `[VectorStoreRecordKey]` y como enlace de resultados de búsqueda con `[TextSearchResultLink]`.
- **`Title`**: Almacena el título del post. Usado como filtro con `[VectorStoreRecordData(IsFilterable = true)]` y como nombre principal en los resultados de búsqueda con `[TextSearchResultName]`.
- **`Description`**: Contiene el contenido del post. Es indexado para búsquedas de texto completo con `[VectorStoreRecordData(IsFullTextSearchable = true)]` y su valor es mostrado en los resultados con `[TextSearchResultValue]`.
- **`DescriptionEmbedding`**: Representación vectorial de `Description`. Almacenado como un vector de 768 dimensiones usando `[VectorStoreRecordVector]`, con `DotProductSimilarity` como función de distancia y `Hnsw` para indexación eficiente.
- **`Tags`**: Etiquetas relacionadas con el post. Pueden ser usadas como filtros en consultas con `[VectorStoreRecordData(IsFilterable = true)]`.
### **Creando un endpoint para almacenar `BlogPost` en Qdrant**

Para guardar nuevos `BlogPost`, crearemos un endpoint que recibirá los datos del post, generará su embedding y lo almacenará en la base de datos vectorial.

Nuestro servicio ya tiene configurados `ITextEmbeddingGenerationService` (para generar embeddings) e `IVectorStore` (para interactuar con Qdrant).

```csharp
public static class CreateBlogPost
{
    public record Request(string Title, string Description, string[] Tags);

    public record Response(Guid BlogPostId);

    public class Handler
    {
        private readonly ITextEmbeddingGenerationService _embeddingService;
        private readonly IVectorStore _vectorStore;

        public Handler(ITextEmbeddingGenerationService embeddingService, IVectorStore vectorStore)
        {
            _embeddingService = embeddingService;
            _vectorStore = vectorStore;
        }

        public async Task<Response> Handle(Request request, CancellationToken ct)
        {
            var blogposts = _vectorStore.GetCollection<Guid, BlogPost>(BlogPost.VectorName);

            await blogposts.CreateCollectionIfNotExistsAsync(ct);

            var embeddingContents =
                await _embeddingService.GenerateEmbeddingAsync(request.Description, cancellationToken: ct);

            var newBlogPost = new BlogPost
            {
                BlogPostId = Guid.NewGuid(),
                Title = request.Title,
                Description = request.Description,
                DescriptionEmbedding = embeddingContents,
                Tags = request.Tags
            };

            await blogposts.UpsertAsync(newBlogPost, cancellationToken: ct);

            return new Response(newBlogPost.BlogPostId);
        }
    }
}
```


**Explicación del código**

1. **Manejador (`Handler`)**
    - Recibe los servicios `ITextEmbeddingGenerationService` y `IVectorStore` a través del constructor.
2. **Lógica del método `Handle`**
    - Obtiene la colección `blogposts` de Qdrant.
    - Si la colección no existe, la crea con `CreateCollectionIfNotExistsAsync()`.
    - Genera el embedding del `Description` usando `_embeddingService.GenerateEmbeddingAsync()`.
    - Crea un `BlogPost` con los datos proporcionados y el embedding generado.
    - Guarda el post en Qdrant con `UpsertAsync()`, que inserta o actualiza el registro.

> **Nota 💡**: Para ver cómo este handler se integra con Minimal APIs, revisa el código fuente del proyecto. Para mantener este artículo conciso, omitiremos detalles específicos.

#### **Probando el endpoint con `api.http`**

Para poblar la base de datos vectorial, podemos hacer una solicitud HTTP de prueba.

Ejemplo en `SemanticKernelLearning03.ApiService.http`:

```
@SemanticKernelLearning03.ApiService_HostAddress = https://localhost:7391

POST {{SemanticKernelLearning03.ApiService_HostAddress}}/api/blog-posts
Content-Type: application/json

{
  "Title": "Introducción a ASP.NET Core",
  "Description": "Una guía completa sobre cómo comenzar con ASP.NET Core y sus características principales.",
  "Tags": ["ASP.NET Core", "C#", "Web Development"]
}
```

### **Verificando los datos en Qdrant**

Si ejecutamos nuestra solución con Aspire y realizamos la solicitud anterior, podemos inspeccionar los datos en el dashboard de Qdrant y verificar que la colección y los registros existen.

#### **Consideraciones sobre el tamaño de los vectores**

- Este ejemplo usa **nomic-embed-text**, que genera vectores de **768 dimensiones**.
- Qdrant debe estar configurado para aceptar este tamaño de vector.
- Si usas modelos de OpenAI u otros proveedores, verifica el tamaño del vector antes de insertarlo en Qdrant.

## **Recuperando Información desde Qdrant**

Para demostrar cómo realizar búsquedas semánticas en los vectores almacenados en Qdrant, podemos hacerlo de distintas maneras: usando el vector directamente o utilizando una abstracción que **Semantic Kernel** nos proporciona, llamada `ITextSearch`.

`ITextSearch` tiene distintas implementaciones, como `BingTextSearch` (que realiza búsquedas en Bing), pero en nuestro caso usaremos `VectorStoreTextSearch`, que está diseñado específicamente para realizar búsquedas en bases de datos vectoriales.

Esta clase necesita conocer:

- **El modelo de embeddings** que estamos usando.
- **El nombre de la colección en Qdrant** donde haremos la búsqueda.

El proceso realmente es sencillo y podríamos hacerlo manualmente construyendo consultas directamente contra la base de datos vectorial. Sin embargo, en este caso, aprovecharemos `ITextSearch` para simplificar el proceso.

> Nota 💡: Para más información sobre búsquedas vectoriales en Semantic Kernel, consulta la documentación oficial de Microsoft:  
> [Vector search using Semantic Kernel Vector Store connectors (Preview) | Microsoft Learn](https://learn.microsoft.com/en-us/semantic-kernel/concepts/vector-store-connectors/vector-search?pivots=programming-language-csharp)

### **Implementación del Endpoint `SearchBlogPost`**

Para realizar la búsqueda semántica, crearemos un nuevo endpoint llamado `SearchBlogPost`, que permitirá encontrar los **posts más relevantes** basados en una consulta en lenguaje natural.

```csharp
public static class SearchBlogPost
{
    public record Request(string Query);

    public record Response(Guid BlogPostId, string Title, string Description);

    public class Handler([FromKeyedServices(BlogPost.VectorName)] ITextSearch textSearch)
    {
        public async Task<List<Response>> Handle(Request request, CancellationToken ct)
        {
            KernelSearchResults<TextSearchResult> textResults =
                await textSearch.GetTextSearchResultsAsync(request.Query, new()
                {
                    Top = 2,
                    Skip = 0,
                }, ct);

            List<Response> responses = new();
            await foreach (TextSearchResult result in textResults.Results.WithCancellation(ct))
            {
                responses.Add(new(
                    Guid.Parse(result.Link),
                    result.Name,
                    result.Value));
            }

            return responses;
        }
    }
}
```

**Explicación del Código**

1. **Uso de `ITextSearch`**
    - Lo recibimos en el constructor con `[FromKeyedServices(BlogPost.VectorName)]`. Esto asegura que estamos inyectando la instancia correcta configurada para nuestra colección de **blog posts** en Qdrant.
	    - Esto lo hemos configurado en `DependencyConfig` anteriormente.
2. **Realizamos la búsqueda semántica**
    - Llamamos a `GetTextSearchResultsAsync(...);`, donde:
        - `request.Query` es el término de búsqueda ingresado.
        - `Top = 2` indica que queremos los **dos resultados más relevantes**.
        - `Skip = 0` significa que no saltaremos ningún resultado.
3. **Procesamos los resultados**
    - Iteramos sobre `textResults.Results` para extraer:
        - `result.Link`: Contiene el `BlogPostId` almacenado como string, lo convertimos a `Guid`.
        - `result.Name`: Corresponde al título del blog post.
        - `result.Value`: Contiene la descripción del blog post.
4. **Devolvemos la lista de resultados**
    - Cada `Response` representa un **blog post relevante** basado en la búsqueda semántica.

Una vez que el endpoint `SearchBlogPost` está registrado, podemos comenzar a realizar búsquedas semánticas para encontrar los posts más relevantes según el texto ingresado.

**Ejemplo de Consulta**

Podemos hacer una búsqueda con la siguiente solicitud:

```
### Busquedas Semanticas
@Query=arquitectura de software
GET {{SemanticKernelLearning03.ApiService_HostAddress}}/api/blog-posts?Query={{Query}}
```

**Respuesta Esperada**

Si hay blog posts relevantes en la base de datos, la API devolverá un listado de los más coincidentes. Un ejemplo de respuesta podría ser:

```json
[
  {
    "blogPostId": "949b7164-2021-4fcc-b58e-9887fdd00d0e",
    "title": "Clean Architecture: Diseño de Software Modular y Escalable",
    "description": "Clean Architecture es un enfoque de diseño de software ..."
  },
  {
    "blogPostId": "2340f1ac-e810-4067-90f5-a71899c6d42a",
    "title": "Principios SOLID en el Desarrollo de Software",
    "description": "Los principios SOLID son un conjunto de cinco principios de..."
  }
]
```

Si revisamos el repositorio, podemos notar que hay posts de distintos temas (por ejemplo, **Machine Learning, Bases de Datos, DevOps**), pero la búsqueda regresó los más relevantes según el texto ingresado. Esto demuestra que el motor de búsqueda semántico está funcionando correctamente.
#### **Personalizando la Búsqueda**

Podemos ajustar la consulta cambiando:
- **La cantidad de resultados (`Top`)**: Actualmente devuelve **dos posts**, pero podríamos aumentar este valor si queremos más opciones.
- **El modelo de embeddings**: Si cambiamos el modelo de embeddings usado para almacenar los vectores, los resultados pueden variar en precisión y relevancia.

## **Integración con RAG (Retrieval-Augmented Generation)**

En esta sección, vamos a implementar un nuevo endpoint que va más allá de una búsqueda semántica. En lugar de solo buscar contenido relevante, vamos a generar texto de manera dinámica a partir de una consulta proporcionada por el usuario. Lo interesante aquí es que el contexto para la generación de la respuesta será el contenido de los **Posts del Blog** que coincidan con la consulta del usuario.

### **¿Cómo funciona?**

Si el usuario hace una pregunta como:

```
cuál es el objetivo principal de clean architecture?
```

En lugar de simplemente devolver una lista de posts relacionados, el modelo de Semantic Kernel buscará los posts más relevantes relacionados con la consulta, y usará esos resultados como contexto para generar una respuesta detallada. El modelo no solo devolverá contenido estático, sino que también incorporará la información más relevante y actualizada disponible en los posts, permitiendo una respuesta más precisa y contextualizada.

Esta técnica es útil porque, por lo general, los modelos de lenguaje (LLMs) se entrenan con datos estáticos que no contienen información actualizada o específica del dominio. Al integrar la **Recuperación de Información (RAG)**, aprovechamos los datos disponibles en tiempo real para mejorar la relevancia y exactitud de las respuestas generadas.

### **¿Cómo lo implementamos?**

En el post anterior, vimos cómo crear **Plugins** o **Funciones Semánticas**, y cómo Semantic Kernel se encarga de invocar automáticamente esos plugins cuando es necesario. Ahora, vamos a utilizar un plugin de búsqueda vectorial para recuperar los posts relevantes y luego construir un **prompt personalizado** que permita al modelo generar una respuesta coherente utilizando esos posts como contexto.

#### **El Endpoint y el Código**

El endpoint que vamos a implementar es el siguiente:

```csharp
public static class QuestionsBlogPost
{
    public record Request(string Query);

    public record Response(string Content);

    public class Handler(Kernel kernel, [FromKeyedServices(BlogPost.VectorName)] ITextSearch textSearch)
    {
        public async Task<Response> Handle(Request request, CancellationToken ct)
        {
            var searchPlugin = textSearch.CreateWithGetTextSearchResults("SearchPlugin");
            kernel.Plugins.Add(searchPlugin);

            string promptTemplate = """
                                    Utiliza solamente los siguientes contenidos para contestar las preguntas realizadas. 
                                    Si no sabes la respuesta, hazle saber al usuario que no se encontró información relevante.

                                    Al final de tu respuesta, incluye el nombre del contenido utilizado como referencia.

                                    {{#with (SearchPlugin-GetTextSearchResults query)}}  
                                        {{#each this}}  
                                        Name: {{Name}}
                                        Value: {{Value}}
                                        -----------------
                                        {{/each}}  
                                    {{/with}}  

                                    Pregunta:

                                    {{query}}
                                    """;

            KernelArguments arguments = new() { { "query", request.Query } };

            HandlebarsPromptTemplateFactory promptTemplateFactory = new();
            var response = await kernel.InvokePromptAsync(
                promptTemplate,
                arguments,
                templateFormat: HandlebarsPromptTemplateFactory.HandlebarsTemplateFormat,
                promptTemplateFactory: promptTemplateFactory,
                cancellationToken: ct);

            return new Response(response.ToString());
        }
    }
}

```

**Explicación del Código**

- **`Handler`**: Esta clase maneja la lógica de la consulta, invoca el plugin de búsqueda, crea el prompt con el contexto adecuado y genera la respuesta. Recibe un objeto **`Kernel`** y una instancia de **`ITextSearch`** que se utiliza para realizar la búsqueda de contenido relevante en los posts del blog.    
- **`searchPlugin`**: Este es el plugin de búsqueda que se crea utilizando el método **`CreateWithGetTextSearchResults`**. Este plugin se encarga de realizar la búsqueda semántica en los posts y recuperar los resultados más relevantes. El plugin se añade al **`Kernel`** para que pueda ser invocado cuando sea necesario.
- **`promptTemplate`**: Es el template que se utilizará para generar el texto. En este caso, se utiliza **Handlebars** para permitir la inserción de los resultados de la búsqueda dentro del prompt. La idea es que el modelo genere una respuesta usando solamente los contenidos encontrados en la búsqueda. El template también incluye instrucciones para que el modelo devuelva el nombre y el valor de los posts utilizados como referencia.

```
{{#with (SearchPlugin-GetTextSearchResults query)}}  
    {{#each this}}  
    Name: {{Name}}  
    Value: {{Value}}  
    -----------------
    {{/each}}  
{{/with}}  
```
- Al final, se agrega la pregunta del usuario para que el modelo la utilice como parte de la respuesta:
```
Pregunta:
{{query}}
```
- **`KernelArguments`**: Este objeto contiene los argumentos que se pasan al modelo, en este caso, la consulta realizada por el usuario.
- **`InvokePromptAsync`**: Este método invoca el modelo utilizando el template y los argumentos proporcionados, lo que genera una respuesta basada en los contenidos recuperados.

**Flujo Completo**

7. **El usuario hace una consulta** (por ejemplo, "¿Qué es Clean Architecture?").
8. El **`Kernel`** busca los posts relevantes utilizando el **plugin de búsqueda vectorial**.
9. El **prompt personalizado** es generado e incluye los resultados de la búsqueda.
10. El **modelo LLM** genera una respuesta utilizando los posts recuperados como contexto.
11. La **respuesta generada** se devuelve al usuario, incluyendo la referencia a los contenidos utilizados.

Este enfoque permite mejorar la calidad de las respuestas generadas por el modelo, ya que se aprovecha la información más actualizada y relevante disponible en el dominio. Además, al usar el contenido de los posts, el modelo tiene un contexto más específico y adaptado a las necesidades del usuario, lo que mejora la relevancia y precisión de la respuesta.

**Probando el Endpoint**

Una vez que hemos configurado y creado nuestro endpoint, es hora de probarlo. Para ello, podemos hacer una solicitud HTTP como la siguiente, que ilustrará cómo interactuar con el servicio de generación de texto mediante la integración de RAG.

Ejemplo de solicitud HTTP:

```
###
@Question=cuál es el objetivo principal de clean architecture?
GET {{SemanticKernelLearning03.ApiService_HostAddress}}/api/blog-posts/qa?Query={{Question}}
```

Cuando se ejecuta esta solicitud, obtendremos una respuesta estructurada que incluye no solo la respuesta generada por el modelo, sino también la referencia al contenido utilizado para generar esa respuesta.

Respuesta esperada:

```
El objetivo principal de Clean Architecture es promover la separación de preocupaciones y la modularidad en el diseño de software, lo que permite a las aplicaciones ser más mantenibles y escalables a lo largo del tiempo.

Reference: Clean Architecture: Diseño de Software Modular y Escalable.
```

En este caso, la respuesta del endpoint es completamente dinámica y se genera utilizando los resultados de búsqueda obtenidos de los posts del blog relacionados con la pregunta. Como puedes observar, no solo obtenemos un resultado que responde a la consulta, sino que también proporcionamos el nombre del contenido utilizado como referencia, lo cual permite al usuario conocer la fuente de la información generada.

En el ejemplo, configuramos el `promptTemplate` para que el modelo de lenguaje solo utilizara los contenidos relevantes encontrados a través de la búsqueda de texto vectorial. De esta forma, garantizamos que la generación del modelo sea específica y relevante para el contexto proporcionado por los posts del blog.

Este template establece claramente la regla para que el modelo solo use los resultados de búsqueda como contexto, lo cual optimiza la relevancia de las respuestas generadas.

**Escalabilidad y Flexibilidad del Prompt**

El prompt puede ser adaptado para ser más específico y mejorar la calidad de las respuestas generadas, por ejemplo, agregando instrucciones más detalladas sobre el formato de la respuesta o proporcionando una estructura más precisa para guiar al modelo. La flexibilidad del sistema permite ajustar el nivel de detalle según la necesidad del caso de uso.

**Ejemplo de otro Prompt:**

```
Enlista los puntos más importantes al querer optimizar consultas SQL
```

**Resultado generado:**

```
¡Claro! A continuación, te presento los puntos más importantes para optimizar consultas SQL:

1. **Índices**: Los índices ayudan a acelerar la búsqueda de datos en una base de datos. Deben ser creados en las columnas que se utilizan en la cláusula WHERE o en la cláusula ORDER BY.
2. **Normalización y desnormalización de datos**: La normalización ayuda a reducir la cantidad de datos duplicados en una base de datos, mientras que la desnormalización puede mejorar el rendimiento en ciertas situaciones.
.
.
.

Referencia: Optimización de Consultas SQL en Aplicaciones .NET
```

Como vemos, el modelo genera una lista de los puntos más importantes y proporciona la referencia adecuada para la información. Este tipo de integración con Semantic Kernel y RAG hace que sea posible tener respuestas altamente relevantes y específicas basadas en contenido actual y relacionado.
## Conclusión

A lo largo de este post, hemos aprendido a configurar y utilizar Semantic Kernel para aprovechar el poder de los embeddings y la técnica de Retrieval-Augmented Generation (RAG). Desde la generación y almacenamiento de vectores en Qdrant, hasta la realización de búsquedas semánticas y la generación de respuestas contextualizadas mediante prompts personalizados, hemos visto cómo combinar distintas tecnologías para crear soluciones de IA avanzadas y especializadas.

Esta integración no solo mejora la precisión y relevancia de las respuestas, sino que también demuestra el potencial de unir funciones semánticas, RAG y bases de datos vectoriales para abordar desafíos reales en el manejo de información. Con estas herramientas, podrás desarrollar aplicaciones que se adaptan al contexto de tus datos, ofreciendo respuestas dinámicas y actualizadas.

¡El camino hacia aplicaciones inteligentes y contextualmente precisas está abierto! Sigue experimentando y optimizando estas técnicas para llevar tus proyectos al siguiente nivel.

## Referencias

- [Using the Semantic Kernel Vector Store text search (Preview) | Microsoft Learn](https://learn.microsoft.com/en-us/semantic-kernel/concepts/text-search/out-of-the-box-textsearch/vectorstore-textsearch?pivots=programming-language-csharp)
- [Semantic Kernel Text Search Plugins (Preview) | Microsoft Learn](https://learn.microsoft.com/en-us/semantic-kernel/concepts/text-search/text-search-plugins?pivots=programming-language-csharp)
- [Semantic Kernel Text Search with Vector Stores (Preview) | Microsoft Learn](https://learn.microsoft.com/en-us/semantic-kernel/concepts/text-search/text-search-vector-stores?pivots=programming-language-csharp)
- [Using the Handlebars prompt template language | Microsoft Learn](https://learn.microsoft.com/en-us/semantic-kernel/concepts/prompts/handlebars-prompt-templates?pivots=programming-language-csharp)
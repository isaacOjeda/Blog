### **1. Introducción**

En el tutorial [anterior](https://dev.to/isaacojeda/semantic-kernel-crea-un-api-para-generacion-de-texto-con-ollama-y-aspire-686), exploramos cómo configurar y utilizar **Semantic Kernel** junto con **Aspire** y **Ollama** para crear un API básico que generaba resúmenes de texto. Ahora, vamos a dar un paso más y enfocarnos en una de las características más potentes de Semantic Kernel: los **plugins**.

En este tutorial, aprenderás a crear y utilizar plugins en Semantic Kernel. Los plugins son clases o componentes que encapsulan funciones específicas, permitiendo extender las capacidades del kernel con nuevas habilidades personalizadas.

Para ejemplificarlo, desarrollaremos dos plugins:

- Un plugin que retorna la hora actual en formato UTC.
- Un plugin que utiliza datos de geolocalización y clima para proporcionar información meteorológica de una ciudad específica.

Los plugins en Semantic Kernel permiten agregar habilidades modulares y reutilizables. Esto no solo facilita la escalabilidad del proyecto, sino que también abre las puertas para integrar servicios externos o lógica personalizada de manera sencilla y eficiente.

> Nota 💡: Aquí podrás encontrar el código fuente de este tutorial: [DevToPosts/SemanticKernelSeries/SemanticKernelLearning02 at main · isaacOjeda/DevToPosts](https://github.com/isaacOjeda/DevToPosts/tree/main/SemanticKernelSeries/SemanticKernelLearning02)

### **2. Qué son los Plugins en Semantic Kernel**

#### **¿Qué es un Plugin en Semantic Kernel?**

Un **plugin** en Semantic Kernel es una clase que contiene métodos decorados con atributos especiales, como `[KernelFunction]`, para exponer su funcionalidad al kernel. Esto permite que esas funciones sean llamadas por el kernel como si fueran partes integradas de su sistema.

#### **¿Por qué usar Plugins?**

- **Modularidad:** Los plugins encapsulan lógica específica, lo que los hace fáciles de mantener y reutilizar.
- **Flexibilidad:** Puedes crear plugins para realizar tareas como interactuar con APIs externas, procesar datos o realizar cálculos.
- **Integración sencilla:** Los plugins se registran fácilmente en el kernel, haciéndolos disponibles para su uso en el API o dentro de flujos más complejos.

#### **Componentes clave de un Plugin**

1. **Métodos decorados con `[KernelFunction]`:** Esto marca las funciones que el kernel puede invocar.
2. **Descripción:** Los métodos pueden incluir descripciones para documentar su propósito, facilitando su descubrimiento.
3. **Dependencias externas:** Los plugins pueden interactuar con servicios externos, como APIs, utilizando patrones conocidos como `IHttpClientFactory`.

#### **Plugins que construiremos en este tutorial**

- **TimeInformationService:** Un plugin que proporciona la hora actual en UTC.
- **WeatherInformationService:** Un plugin que utiliza una API externa para obtener información de geolocalización y clima.

Ambos plugins serán integrados en nuestro kernel y expuestos a través del API que desarrollamos en el tutorial anterior.

#### **¿Cómo funcionan los Plugins en Semantic Kernel?**

Los plugins registrados en el kernel pueden ser invocados en tiempo de ejecución mediante prompts o directamente desde el código. El kernel gestiona automáticamente la ejecución de estos métodos, permitiendo integrarlos en flujos de procesamiento complejos.

### **3. Desarrollo del Ejemplo**

Ahora que entendemos qué son los plugins en **Semantic Kernel**, vamos a desarrollar e integrar dos plugins personalizados en nuestro proyecto:

1. **TimeInformationService:** Un plugin simple que devuelve la hora actual en UTC.
2. **WeatherInformationService:** Un plugin más avanzado que utiliza una API externa para obtener información sobre el clima en una ciudad.

#### **Paso 1: Crear los Plugins**

##### **TimeInformationService**

El siguiente plugin proporciona la hora actual en formato UTC. Es un ejemplo sencillo para entender cómo funcionan los plugins básicos en Semantic Kernel.

```csharp
/// <summary>
/// A plugin that returns the current time.
/// </summary>
public class TimeInformationService
{
    [KernelFunction]
    [Description("Retrieves the current time in UTC.")]
    public string GetCurrentUtcTime() => DateTime.UtcNow.ToString("R");
}
```

Aquí destacamos:

- El atributo `[KernelFunction]` expone el método al kernel.
- El método devuelve la hora en formato "R" (RFC1123), que es legible y estándar.

##### **WeatherInformationService**

Este plugin es más complejo. Obtiene información sobre una ciudad (coordenadas, país, etc.) y también el clima actual utilizando servicios externos.

```csharp
public class WeatherInformationService(IHttpClientFactory httpClientFactory)
{
    [KernelFunction]
    [Description("Get weather information for a city.")]
    public async Task<WeatherInfo> GetWeatherByCity(string cityName)
    {
        if (string.IsNullOrWhiteSpace(cityName))
            throw new ArgumentException("City name cannot be null or empty.", nameof(cityName));

        // Step 1: Get city coordinates
        var city = await GetCityCoordinatesAsync(cityName);

        // Step 2: Get weather data
        var weatherResult = await GetWeatherDataAsync(city.Latitude, city.Longitude);

        return new WeatherInfo
        {
            City = city.Name,
            Country = city.Country,
            Temperature = weatherResult.CurrentWeather.Temperature,
            WindSpeed = weatherResult.CurrentWeather.Windspeed,
            WeatherCode = weatherResult.CurrentWeather.Weathercode
        };
    }

    private async Task<GeocodingResult> GetCityCoordinatesAsync(string cityName)
    {
        // ...For more info, check the source code
    }

    private async Task<WeatherApiResponse> GetWeatherDataAsync(double latitude, double longitude)
    {
        // ...For more info, check the source code
    }
}
```

Puntos clave:

- Este plugin utiliza `IHttpClientFactory` para realizar llamadas HTTP a APIs externas.
	- Como puedes ver, la inyección de dependencias funciona sin problema, por lo que aquí podríamos inyectar lo que se necesite que el plugin realice
- El Kernel utiliza los nombres de los métodos y los parámetros para darles significado y así usarlo según el prompt solicitado.

#### **Paso 2: Registrar los Plugins en el Kernel**

Para que el kernel pueda usar estos plugins, debemos registrarlos en la configuración del servicio:

```csharp
var kernel = builder.Services.AddKernel();

// Registrar el servicio de hora
kernel.ImportPlugin<TimeInformationService>();

// Registrar el servicio de clima
kernel.ImportPlugin<WeatherInformationService>();
```

#### **Paso 3: Exponer los Plugins Utilizando el Kernel**

En lugar de llamar directamente a los métodos de los servicios, utilizaremos el **Kernel** de Semantic Kernel para invocar los plugins mediante prompts. Esto nos permitirá combinar capacidades y hacer el uso más dinámico de los plugins.

Haremos la prueba con un simple endpoint que invoque el prompt

```csharp
app.MapPost("/api/chat", async (ChatRequest request, Kernel kernel) =>
{
    var settings = new OpenAIPromptExecutionSettings()
    {
        FunctionChoiceBehavior = FunctionChoiceBehavior.Auto()
    };

	// Aquí podríamos hacer un Prompt más explícito para el modelo si nuestra intención fuera
	// solo contestar preguntas sobre el clima.

    var response = await kernel.InvokePromptAsync(request.Question, new(settings));

    return Results.Ok(new
    {
        Result = response.ToString()
    });
});
```

#### **Explicación del Código**

1. **Uso del Kernel**:
    - El método `InvokePromptAsync` utiliza el kernel para procesar el prompt y determinar qué función de los plugins debe ejecutarse.
    - En este caso, el Kernel automáticamente seleccionará el método correcto (`GetCurrentUtcTime` o `GetWeatherByCity`) basado en el texto del prompt.
2. **Integración Flexible**:
    - La estructura permite enviar cualquier pregunta o texto, dejando que el Kernel interprete y seleccione el método adecuado.
	    - También podemos hacer prompts donde seamos explícitos de que Plugin queremos que el modelo utilice, pero por ahora lo dejamos en automático y que el modelo decida.


### **4. Probar los Endpoints**

Con los endpoints configurados y los plugins integrados en el **Kernel**, es momento de validarlos y observar cómo funcionan.

#### **Ejemplo 1: Consultar el Clima**

Utilizando **Postman**, envía la siguiente solicitud al endpoint configurado:

```json
{
  "question": "Cuál es el clima actual en Londres"
}
```

Al depurar la aplicación, notarás que el modelo entiende la intención del prompt, lo asocia con nuestro plugin y determina que queremos el clima actual de Londres. Esto sucede porque hemos registrado una función específica que devuelve esta información:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/b1omsz3jsf1r2gqu742m.png)

La respuesta sería algo similar a:

```json
{
    "result": "El clima actual en Londres es frío y nublado, con una temperatura de 8.6°C y un viento que sopla a 6.8 km/h. La codificación del tiempo es 3, lo que indica condiciones climáticas mezcladas, con algunas nubes y posibles lluvias."
}
```

#### **Ejemplo 2: Consultar Información de una Ciudad**

Ahora, realiza una consulta diferente, como esta:

```json
{
  "question": "Cuál es la población actual de Madrid?"
}
```

En este caso, el modelo invocará automáticamente otra función registrada en el plugin que se encarga de obtener información sobre una ciudad:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7pf76oqlglt0322h9whj.png)

La respuesta será algo como:

```json
{
    "result": "La población actual de Madrid es de aproximadamente 3,255,594 personas, según la información proporcionada por el servicio de información geográfica."
}
```

¿Notas el poder detrás de esta integración? Con esta configuración, el **Kernel** puede realizar tareas que antes requerían múltiples servicios separados. Todo se reduce a cómo estructuramos los prompts y las funciones de nuestros plugins.

¡Y esto es solo el principio! Aún no hemos explorado características avanzadas como **memoria semántica** y **RAG (Retrieval-Augmented Generation)**, pero con lo que ya tenemos, podemos construir soluciones extremadamente flexibles y personalizadas para cualquier dominio o industria.

Con prompts más específicos, puedes resolver problemas complejos y automatizar procesos avanzados. La única limitación es tu creatividad y las necesidades de tu proyecto.

### **5. Extender Funcionalidades y Consideraciones Finales**

Ahora que hemos visto cómo registrar plugins y probarlos con el **Kernel**, reflexionemos sobre cómo podemos extender estas funcionalidades para adaptarlas a necesidades más complejas. Aquí algunas ideas y recomendaciones para el siguiente nivel:

#### **Incorporar Más Plugins**

La flexibilidad de **Semantic Kernel** nos permite agregar más funciones a nuestros plugins o crear nuevos plugins para cubrir otras áreas de interés, como:

- **Análisis de texto:** Procesar grandes volúmenes de datos textuales para obtener resúmenes, extraer palabras clave o identificar entidades.
- **Integraciones externas:** Conectar con APIs de terceros para realizar tareas como manejo de correos electrónicos, acceso a sistemas empresariales o realizar transacciones.
- **Procesamiento de imágenes:** Usar servicios como OCR para reconocer texto en imágenes o realizar análisis visual.

#### **Usar Memoria Semántica**

La **memoria semántica** de Semantic Kernel puede almacenar y recuperar información contextualmente relevante. Por ejemplo:

- Recordar interacciones previas para mejorar la experiencia del usuario.
- Contextualizar consultas complejas basándose en datos históricos.
- Implementar personalización profunda en aplicaciones que requieran un historial de acciones o preferencias.

#### **Optimización de Prompts**

Un diseño adecuado de prompts es clave para mejorar la precisión de los resultados. Algunas recomendaciones:

- **Sea específico:** Prompts detallados producen resultados más enfocados.
- **Use ejemplos:** Proporcionar ejemplos en el prompt puede ayudar al modelo a entender la intención de manera más clara.
- **Evalúe y ajuste:** Experimente con diferentes configuraciones para obtener mejores resultados en diferentes escenarios.

#### **Escalabilidad y Producción**

Si bien en estos ejemplos usamos **Ollama** localmente para evitar problemas de infraestructura, en un entorno de producción puedes cambiar sin modificar tu código base y usar servicios como **OpenAI** o **Azure OpenAI** para aprovechar su escalabilidad y confiabilidad. Esto implica:

- Definir claves API para entornos seguros.
- Configurar políticas de acceso según las necesidades de tu organización.
- Monitorear el uso y los costos para optimizar el rendimiento.

#### **Casos de Uso Empresariales**

Con los elementos ya implementados, podemos visualizar algunos casos prácticos donde estas capacidades sean útiles:

- **Asistentes de soporte:** Resolver consultas comunes en tiempo real con información siempre actualizada.
- **Automatización de tareas repetitivas:** Por ejemplo, generar reportes basados en datos o notificaciones programadas.
- **Sistemas personalizados:** Aplicaciones inteligentes que se adapten dinámicamente a las necesidades de los usuarios, como dashboards interactivos o chatbots avanzados.
### **Conclusión**

Con **Semantic Kernel**, hemos transformado lo que tradicionalmente sería una combinación de servicios aislados en una solución unificada y potente. Desde entender intenciones en lenguaje natural hasta ejecutar acciones basadas en plugins específicos, el Kernel nos proporciona una plataforma flexible y extensible para la creación de aplicaciones inteligentes.

La verdadera ventaja de esta tecnología radica en su capacidad de evolución. Puedes comenzar con casos de uso simples, como los ejemplos de este tutorial, y escalar hacia soluciones más complejas que integren múltiples fuentes de datos, memoria semántica y funcionalidades avanzadas.

Tu creatividad y las necesidades de tu proyecto dictarán el camino. Ahora que tienes las bases, ¿qué construirás con **Semantic Kernel**?

### **Ejercicio 1: Crear una Habilidad Básica en Semantic Kernel**

**Objetivo**: Aprender a crear una habilidad personalizada y consumirla desde un API.

**Pasos**:

1. **Configura Semantic Kernel**:
    
    - Crea un proyecto en C# e instala Semantic Kernel desde NuGet.
    - Configura el kernel con un modelo local como Ollama.
2. **Define una Habilidad Simple**:
    
    - Crea una habilidad llamada `SummarizerSkill` que tome un texto como entrada y devuelva un resumen corto.
    - Usa un prompt como este:
        
        text
        
        Copiar código
        
        `Summarize the following text in one sentence:   {{$input}}`
        
3. **Crea un Endpoint en tu API**:
    
    - Diseña un endpoint que reciba texto, lo pase a la habilidad y retorne el resumen generado.

**Resultados Esperados**:  
Un API funcional que resuma cualquier texto de entrada en una sola oración.

---

### **Ejercicio 2: Uso de Memoria Semántica para Almacenar y Recuperar Datos**

**Objetivo**: Implementar memoria semántica para guardar y recuperar contexto en Semantic Kernel.

**Pasos**:

1. **Configura Memoria en Semantic Kernel**:
    
    - Usa un proveedor de memoria simple como `VolatileMemory` (almacén en memoria).
2. **Guardar Datos en Memoria**:
    
    - Diseña un endpoint para guardar información en memoria. Por ejemplo:
        - **Input**: "El modelo llama2 es un modelo generativo avanzado."
        - **Tag**: "modelo".
3. **Recuperar Datos Relevantes**:
    
    - Crea otro endpoint que tome una consulta y recupere datos relevantes desde la memoria usando `kernel.Memory.SearchAsync`.

**Resultados Esperados**:  
Un flujo básico donde puedas guardar información contextual y recuperarla según una consulta.

---

### **Ejercicio 3: Composición de Habilidades (Planner Básico)**

**Objetivo**: Aprender a usar el planner de Semantic Kernel para combinar múltiples habilidades.

**Pasos**:

1. **Crea Múltiples Habilidades**:
    
    - Define dos habilidades:
        - **SummarizerSkill**: Resume texto (del ejercicio 1).
        - **TranslatorSkill**: Traduce texto a otro idioma. Ejemplo de prompt:
            
            text
            
            Copiar código
            
            `Translate the following text into Spanish:   {{$input}}`
            
2. **Usa el Planner**:
    
    - Configura el planner para coordinar las habilidades.
    - Ejemplo de flujo:
        - Resumir un texto en inglés.
        - Traducir el resumen al español.
3. **Crea un Endpoint**:
    
    - Diseña un endpoint que tome texto como entrada y aplique ambas habilidades en secuencia.

**Resultados Esperados**:  
Una API que reciba texto, lo resuma y traduzca el resultado automáticamente.

---

### **Ejercicio 4: Generación de Texto con Contexto Dinámico**

**Objetivo**: Aprender a usar variables dinámicas en prompts para personalizar respuestas.

**Pasos**:

1. **Define una Habilidad Personalizada**:
    
    - Crea una habilidad llamada `CustomResponseSkill` con este prompt:
        
        text
        
        Copiar código
        
        `You are a helpful assistant. Answer the following question as concisely as possible:   Question: {{$question}}   Context: {{$context}}`  
        
2. **Crea un Endpoint con Contexto Dinámico**:
    
    - Diseña un endpoint que tome un contexto y una pregunta como parámetros, luego pase ambas variables al prompt.
3. **Prueba el Flujo**:
    
    - Usa ejemplos como:
        - **Contexto**: "Semantic Kernel es un marco de trabajo para crear aplicaciones basadas en IA."
        - **Pregunta**: "¿Qué es Semantic Kernel?"

**Resultados Esperados**:  
Respuestas dinámicas y personalizadas basadas en el contexto proporcionado.

---

### **Ejercicio 5: Habilidad con Lógica Avanzada (Condicionales en Prompts)**

**Objetivo**: Crear una habilidad que use lógica avanzada en prompts para generar diferentes salidas según la entrada.

**Pasos**:

1. **Define una Habilidad Compleja**:
    
    - Crea una habilidad llamada `SupportSkill` con este prompt:
        
        text
        
        Copiar código
        
        `You are a support assistant. Respond according to the priority level:   - If priority is "high", provide an urgent response.   - If priority is "low", provide a detailed response.   Priority: {{$priority}}   Question: {{$question}}`  
        
2. **Diseña un Endpoint**:
    
    - El endpoint debe tomar una pregunta y un nivel de prioridad como parámetros y devolver una respuesta basada en las reglas del prompt.
3. **Prueba Diferentes Escenarios**:
    
    - **Ejemplo 1**:
        - **Pregunta**: "¿Qué hacer si mi servidor falla?"
        - **Prioridad**: "high".
    - **Ejemplo 2**:
        - **Pregunta**: "¿Cómo optimizar una consulta en SQL?"
        - **Prioridad**: "low".

**Resultados Esperados**:  
Respuestas generadas con diferentes tonos o niveles de detalle según la prioridad especificada.
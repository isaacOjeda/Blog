# EF Core: tu query funciona, tus pruebas pasan… y estás leyendo 50,000 filas para devolver 3

El código lleva meses en producción. Nadie se ha quejado. Todas las pruebas pasan.

Luego llega un cliente grande, con años de historial acumulado, y esa pantalla que siempre cargó “bien” empieza a tardar varios segundos. A veces ni siquiera falla: simplemente se vuelve torpe, pesada, impredecible.

Empiezas a revisar y descubres algo incómodo: el problema siempre estuvo ahí. Solo necesitaba suficientes datos para hacerse visible.

Ese es uno de los patrones más traicioneros cuando trabajas con EF Core. No porque el código esté roto, sino porque **funciona**. Devuelve los datos correctos. Pasa QA. Sobrevive en producción durante meses. Y aun así, por debajo, puede estar ejecutando consultas mucho más caras de lo que parecen.

En este artículo quiero mostrar dos trampas comunes:

- cuando materializas demasiado pronto con `ToList()` y el filtro termina ocurriendo en memoria;
- cuando dejas lógica no traducible dentro de la proyección y la consulta deja de ser tan eficiente como parece.

La idea no es memorizar reglas raras de EF Core. La idea es desarrollar un reflejo: **siempre revisar el SQL real que se genera**.

## Por qué casi nadie lo detecta a tiempo

Este problema rara vez aparece de golpe.

En local, todo corre sobre una base pequeña, con datos de prueba y una máquina rápida. Las consultas tardan milisegundos y nadie siente la necesidad de inspeccionar el SQL.

En QA o staging, la infraestructura puede parecerse a producción, pero el volumen de datos sigue sin representar el mundo real. El problema ya existe, solo que todavía no pesa.

En producción, con clientes pequeños, los tiempos siguen siendo aceptables. No hay alertas. No hay tickets. No hay razón aparente para abrir el capó.

Y entonces llega el cliente con años de historial, decenas de miles de registros y patrones reales de uso. Ahí, por primera vez, la consulta muestra su costo verdadero.

Lo más difícil de este escenario no es corregirlo. Lo más difícil es que cuando por fin se vuelve visible, el código ya lleva tanto tiempo vivo que nadie recuerda con claridad por qué se escribió así.

> En EF Core, el peligro no siempre es que algo falle. A veces el peligro es que funcione… pero de la forma más cara posible.

## Problema 1: el `ToList()` que mueve el filtro fuera de SQL

### El origen: una excepción legítima

Todo empieza con algo bastante normal: quieres usar lógica de negocio propia dentro de un `Where`.

```csharp
var pedidos = await context.Pedidos
    .Where(p => EsDelUltimoMes(p.FechaCreacion))
    .Select(p => new PedidoResumenDto
    {
        Cliente = p.Cliente.Nombre,
        Total = p.Total
    })
    .ToListAsync();

private static bool EsDelUltimoMes(DateTime fecha) =>
    fecha >= DateTime.Now.AddMonths(-1);
```

Ese código no es traducible tal como está. `EsDelUltimoMes` es un método local de C#, y EF Core no sabe convertirlo a SQL. En EF Core moderno, eso normalmente termina en una excepción de traducción.

Algo como esto:

```text
InvalidOperationException: The LINQ expression could not be translated.
Either rewrite the query in a form that can be translated, or switch to
client evaluation explicitly by inserting a call to 'AsEnumerable',
'AsAsyncEnumerable', 'ToList', or 'ToListAsync'.
```

La excepción no miente. Pero es muy fácil leerla de forma peligrosa.

### La “solución” que parece arreglar todo

Alguien ve que el mensaje menciona `ToList()`, lo agrega, y el problema desaparece:

```csharp
var pedidos = context.Pedidos
    .ToList()
    .Where(p => EsDelUltimoMes(p.FechaCreacion))
    .Select(p => new PedidoResumenDto
    {
        Cliente = p.Cliente.Nombre,
        Total = p.Total
    })
    .ToList();
```

Y sí: ahora funciona. Compila, devuelve resultados correctos y pasa las pruebas.

Pero ya no estás filtrando en la base de datos. Estás trayendo todos los registros primero, y luego aplicando la lógica en memoria.

En otras palabras, el SQL que sale ya no se parece a la intención original. Se parece más a esto:

```sql
SELECT p.Id, p.ClienteId, p.Total, p.Activo, p.FechaCreacion, ...
FROM Pedidos
```

Sin `WHERE`. Sin proyección útil. Sin límite.

Si la tabla tiene 50,000 filas, esas 50,000 filas viajan completas desde la base de datos hasta tu aplicación. Solo después haces el filtro en C# para devolver quizá 3 resultados al usuario.

Ese es el tipo de problema que no se nota con 200 registros, pero sí con años de operación acumulada.

### La forma correcta

La solución real es expresar el filtro usando operadores que EF Core sí conozca y pueda traducir.

```csharp
var fechaLimite = DateTime.Now.AddMonths(-1);

var pedidos = await context.Pedidos
    .Where(p => p.FechaCreacion >= fechaLimite)
    .Select(p => new PedidoResumenDto
    {
        Cliente = p.Cliente.Nombre,
        Total = p.Total
    })
    .ToListAsync();
```

Ahora sí el filtro vive donde debe vivir: en SQL.

```sql
SELECT c.Nombre, p.Total
FROM Pedidos p
LEFT JOIN Clientes c ON p.ClienteId = c.Id
WHERE p.FechaCreacion >= '2025-02-14'
```

La regla práctica aquí no es “nunca uses métodos”. Es esta: **mientras la lógica se exprese con operadores y funciones que EF Core conoce, podrá empujarla a SQL**. Cuando no pueda, corres el riesgo de que la evaluación ocurra fuera de la base de datos o de que EF directamente falle al traducir.

Si necesitas lógica de negocio más compleja, aplícala después de materializar, pero solo sobre un conjunto de datos que ya filtraste correctamente en la base.

> Nota: en algunos escenarios complejos, usar Raw SQL es perfectamente válido. No es una derrota; es una herramienta más.

## Problema 2: la proyección parece pequeña, pero la consulta deja de ser eficiente

El segundo problema es más silencioso.

Aquí el `Where` sí se traduce. La consulta sí filtra en la base de datos. Los resultados son correctos. El detalle está en la proyección.

### El escenario

```csharp
var pedidos = await context.Pedidos
    .Where(p => p.FechaCreacion >= fechaLimite)
    .Select(p => new PedidoResumenDto
    {
        Cliente = p.Cliente.Nombre,
        Total = p.Total,
        Estatus = ObtenerEtiqueta(p.Activo)
    })
    .ToListAsync();

private static string ObtenerEtiqueta(bool activo) =>
    activo ? "Activo" : "Inactivo";
```

A simple vista, parece una proyección pequeña. Solo quieres tres campos.

El problema es que `ObtenerEtiqueta` sigue siendo lógica local de C#. EF Core no puede traducir ese método directamente. En la proyección final, eso puede hacer que parte del trabajo termine resolviéndose del lado del cliente.

Y cuando eso pasa, la consulta puede traer **más información de la necesaria** para completar esa proyección en memoria.

La parte peligrosa no es solo “que funcione”. Es que desde el código parece una consulta mínima, cuando en realidad el SQL puede estar cargando más columnas de las que tú creías.

No siempre será literalmente un `SELECT *`, pero sí puede ocurrir que EF traiga suficiente información como para volver la proyección más costosa de lo que aparenta en C#.

### La forma correcta

Si el formateo no necesita vivir en SQL, sepáralo de la consulta:

```csharp
var pedidos = await context.Pedidos
    .Where(p => p.FechaCreacion >= fechaLimite)
    .Select(p => new PedidoResumenDto
    {
        Cliente = p.Cliente.Nombre,
        Total = p.Total,
        EsActivo = p.Activo
    })
    .ToListAsync();

// Dejar que la capa presentación muestre los datos
// o
// 👇🏼
foreach (var pedido in pedidos)
    pedido.Estatus = ObtenerEtiqueta(pedido.EsActivo);
```

Así la base de datos solo devuelve lo necesario, y el formateo ocurre después, en memoria, pero ya sobre un conjunto pequeño y bien proyectado.

Si la lógica sí puede expresarse con algo traducible, entonces mejor dejarla en la consulta:

```csharp
var pedidos = await context.Pedidos
    .Where(p => p.FechaCreacion >= fechaLimite)
    .Select(p => new PedidoResumenDto
    {
        Cliente = p.Cliente.Nombre,
        Total = p.Total,
        Estatus = p.Activo ? "Activo" : "Inactivo"
    })
    .ToListAsync();
```

Una expresión ternaria simple como esa normalmente sí se traduce a `CASE WHEN` en SQL.

## Cómo inspeccionar lo que EF Core realmente está haciendo

Si no ves el SQL, estás adivinando.

Y adivinar con consultas que hoy corren sobre 500 filas y mañana correrán sobre 5 millones suele salir caro.

### Opción 1: logs en desarrollo

Puedes habilitar logs del contexto para ver el SQL ejecutado:

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
           .LogTo(Console.WriteLine, LogLevel.Information)
           .EnableSensitiveDataLogging());
```

`EnableSensitiveDataLogging()` conviene dejarlo solo en desarrollo.

En la salida, busca entradas como esta:

```text
Executed DbCommand (X ms) [Parameters=[...], CommandType='Text', CommandTimeout='30']
```

Ahí está la consulta real.

### Opción 2: `ToQueryString()`

Cuando quieres revisar una consulta específica, esta suele ser la forma más directa:

```csharp
var query = context.Pedidos
    .Where(p => p.FechaCreacion >= fechaLimite)
    .Select(p => new PedidoResumenDto
    {
        Cliente = p.Cliente.Nombre,
        Total = p.Total,
        Estatus = p.Activo ? "Activo" : "Inactivo"
    });

Console.WriteLine(query.ToQueryString());
var pedidos = await query.ToListAsync();
```

`ToQueryString()` funciona sobre cualquier `IQueryable` antes de materializarlo. Si tienes dudas sobre traducción, úsalo. Te ahorra suposiciones.

## Un detalle extra que vale oro en consultas de lectura

Si esa consulta solo existe para mostrar datos y no vas a modificar las entidades después, vale la pena considerar `AsNoTracking()`:

```csharp
var pedidos = await context.Pedidos
    .AsNoTracking()
    .Where(p => p.FechaCreacion >= fechaLimite)
    .Select(p => new PedidoResumenDto
    {
        Cliente = p.Cliente.Nombre,
        Total = p.Total
    })
    .ToListAsync();
```

No arregla problemas de traducción ni evita consultas mal diseñadas. Pero sí reduce trabajo del change tracker en escenarios de solo lectura, y eso puede sumar bastante cuando la pantalla consulta muchos registros.

## Sobre asistentes de código y queries “que funcionan”

Los asistentes de código suelen optimizar para darte algo que compile, que se vea razonable y que devuelva el resultado esperado.

Eso está bien. Pero una consulta correcta funcionalmente no siempre es una consulta sana desde el punto de vista de traducción o performance.

El asistente no ve el volumen real de tu tabla. No está mirando el plan de ejecución. No sabe si ese query hoy devuelve 20 registros o si mañana va a inspeccionar medio millón para construir una pantalla.

**Por eso esta revisión sigue siendo responsabilidad del developer**: no basta con que la consulta compile y pase pruebas. Hay que confirmar qué SQL genera y dónde se está ejecutando realmente cada parte de la lógica.

## Resumen

|Situación|Qué ocurre|Cómo detectarlo|
|---|---|---|
|Método local no traducible dentro de `Where`|EF Core normalmente falla al traducir o te obliga a mover la evaluación fuera de SQL|Excepción de traducción|
|`ToList()` antes del filtro|La consulta se materializa completa y el filtrado ocurre en memoria|Revisando el código y el SQL generado|
|Lógica no traducible en la proyección final|Parte de la proyección puede resolverse en cliente y traer más datos de los necesarios|`ToQueryString()`, logs y revisión de columnas consultadas|

La idea central de todo esto es simple: **no confíes en que una query está bien solo porque funciona**.

En EF Core, una consulta puede ser correcta en resultados y aun así ser innecesariamente cara. Y la forma más confiable de descubrirlo no es mirando el LINQ: es mirando el SQL.

## ¿Qué sigue?

Este es el primero de una serie sobre cosas que EF Core hace de forma poco obvia y que solo se vuelven dolorosas cuando los datos son reales.

El siguiente artículo va sobre el problema N+1: esa consulta que en desarrollo parece hacer unas pocas llamadas a la base, pero en producción termina haciendo cientos o miles.

Si alguna vez te pasó algo parecido, seguramente no fue porque el código estuviera “mal” en apariencia. Fue porque el costo real estaba escondido en un lugar que casi nadie revisa a tiempo.
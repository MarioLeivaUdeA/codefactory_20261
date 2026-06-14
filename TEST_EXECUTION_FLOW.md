# Flujo de Ejecución Completo de una Prueba Screenplay

## Tabla de Contenidos
1. [Visión General](#visión-general)
2. [Phase 1: Modelado de Datos (TestDataFactory)](#phase-1-modelado-de-datos-testdatafactory)
3. [Phase 2: Tasks (Acciones)](#phase-2-tasks-acciones)
4. [Phase 3: Questions (Consultas)](#phase-3-questions-consultas)
5. [Phase 4: Step Definitions (Orquestación)](#phase-4-step-definitions-orquestación)
6. [Phase 5: Ejecución Completa](#phase-5-ejecución-completa)
7. [Ejemplo Detallado: Happy Path](#ejemplo-detallado-happy-path)

---

## Visión General

El flujo de una prueba Screenplay en este proyecto sigue el patrón **Arrange → Act → Assert**:

```
TestDataFactory (datos)
    ↓
Task (ejecutar acción)
    ↓
Question (consultar resultado)
    ↓
StepDefinition (orquestar y verificar)
```

---

## Phase 1: Modelado de Datos (TestDataFactory)

### Propósito
El `TestDataFactory` es una fábrica de datos que genera mapas con los valores necesarios para las pruebas. No es un mapper de tablas Cucumber; es un **generador de datos programático**.

### Archivo
`src/test/java/com/example/codefactory/screenplay/models/TestDataFactory.java`

### Estructura y construcción

```java
public class TestDataFactory {
    // Genera un Map<String, Object> con datos válidos de envío
    public static Map<String, Object> validShipmentData() {
        Map<String, Object> data = new HashMap<>();
        
        // Anida un Map de persona para remitente
        data.put("sender", validPersonData());
        
        // Anida un Map de persona para destinatario
        data.put("recipient", validPersonData());
        
        // Agrega campos simples con sus valores
        data.put("tipoServicio", "EXPRESS");
        data.put("nivelPrioridad", 1);
        data.put("fechaEnvio", LocalDateTime.now().plusDays(1).toString());
        data.put("fechaEstimada", LocalDateTime.now().plusDays(3).toString());
        data.put("costoTotal", 150.0);
        data.put("instruccionesEnvio", "Handle with care");
        
        // Devuelve el Map listo para serializar a JSON
        return data;
    }
}
```

### Qué es cada parte

#### `Map<String, Object>`
- Es una colección de **pares clave-valor**.
- `String` = nombre del campo (como `"tipoServicio"`, `"sender"`).
- `Object` = puede ser cualquier cosa: otro `Map`, `String`, número, fecha, etc.

#### `new HashMap<>()`
- Crea un contenedor vacío de pares.

#### `data.put(clave, valor)`
- Agrega una entrada al mapa.
- Ejemplo: `data.put("tipoServicio", "EXPRESS")` → `{"tipoServicio": "EXPRESS"}` en JSON.

#### `validPersonData()`
- Es otro método que genera un `Map` con datos de persona.
- Se reutiliza para remitente y destinatario, generando datos únicos cada vez (UUID).

### Métodos disponibles

| Método | Propósito |
|--------|-----------|
| `validShipmentData()` | Envío con todos los campos requeridos. |
| `minimumShipmentData()` | Envío con solo campos mínimos. |
| `invalidShipmentData()` | Envío intencionalmente sin campos requeridos (para probar validación). |
| `validPackageData()` | Datos de paquete válido. |
| `validPersonData()` | Datos de persona con UUID único. |
| `validSenderData()` | Alias para `validPersonData()`. |
| `validVehicleData()` | Datos de vehículo. |
| `validLogisticEventData(String eventType)` | Evento logístico con tipo específico. |

---

## Phase 2: Tasks (Acciones)

### Propósito
Una `Task` en Screenplay es una **acción que el actor intenta realizar**. Implementa la interfaz `Task` de Serenity y encapsula una operación HTTP o una secuencia de pasos.

### Responsabilidades de una Task
1. Recibir datos (como un `Map`).
2. Ejecutar una interacción (POST, GET, etc.).
3. Extraer información de la respuesta (si aplica).
4. Recordar valores en el actor para pasos posteriores.

### Ejemplo 1: `CreateShipment` (POST)

#### Archivo
`src/test/java/com/example/codefactory/screenplay/tasks/CreateShipment.java`

#### Construcción

```java
public class CreateShipment implements Task {
    
    // Campo: recibe el mapa de datos del envío
    private final Map<String, Object> shipmentData;

    // Constructor: asigna los datos
    public CreateShipment(Map<String, Object> shipmentData) {
        this.shipmentData = shipmentData;
    }

    // Método de fábrica: facilita la construcción
    public static CreateShipment withData(Map<String, Object> data) {
        // `Tasks.instrumented` envuelve la tarea para Serenity Screenplay
        return Tasks.instrumented(CreateShipment.class, data);
    }

    // Método principal: lo que el actor ejecuta
    @Override
    public <T extends Actor> void performAs(T actor) {
        // 1. El actor intenta hacer un POST a /api/v1/shipments
        actor.attemptsTo(
            Post.to("/api/v1/shipments")
                .with(request -> request
                    .contentType(ContentType.JSON)
                    .body(shipmentData))  // Envía el Map como body JSON
        );

        // 2. Extrae valores de la respuesta
        String id = actor.asksFor(TheResponseBody.jsonPath()).getString("id");
        String tracking = actor.asksFor(TheResponseBody.jsonPath()).getString("codigoRastreo");

        // 3. Valida que se extrajeron correctamente
        if (id == null || tracking == null) {
            throw new AssertionError("CreateShipment task could not extract shipment id or tracking code");
        }

        // 4. Recuerda los valores en el actor para pasos posteriores
        actor.remember("currentShipmentId", id);
        actor.remember("currentTrackingCode", tracking);
    }
}
```

#### Qué hace cada línea
| Línea | Propósito |
|-------|-----------|
| `private final Map<String, Object> shipmentData` | Almacena los datos del envío. |
| `Tasks.instrumented(...)` | Envuelve la tarea para que Serenity pueda rastrearla. |
| `actor.attemptsTo(...)` | Ejecuta la acción del actor. |
| `Post.to("/api/v1/shipments")` | Define una petición POST a ese endpoint. |
| `.body(shipmentData)` | Serializa el Map a JSON y lo envía. |
| `actor.asksFor(...)` | Usa una Question para consultar la respuesta. |
| `actor.remember(...)` | Guarda un valor en la memoria del actor. |

### Ejemplo 2: `RegisterLogisticEvent` (POST con validación)

```java
public class RegisterLogisticEvent implements Task {
    
    private final String shipmentId;
    private final Map<String, Object> eventData;

    // Constructor y fábrica similares a CreateShipment
    public RegisterLogisticEvent(String shipmentId, Map<String, Object> eventData) {
        this.shipmentId = shipmentId;
        this.eventData = eventData;
    }

    public static RegisterLogisticEventBuilder forShipment(String shipmentId) {
        return new RegisterLogisticEventBuilder(shipmentId);
    }

    // Builder inner class: permite sintaxis fluida
    public static class RegisterLogisticEventBuilder {
        private final String shipmentId;

        public RegisterLogisticEventBuilder(String shipmentId) {
            this.shipmentId = shipmentId;
        }

        public RegisterLogisticEvent withData(Map<String, Object> data) {
            return Tasks.instrumented(RegisterLogisticEvent.class, shipmentId, data);
        }
    }

    @Override
    public <T extends Actor> void performAs(T actor) {
        // POST a /api/v1/tracking/shipments/{id}/events
        actor.attemptsTo(
            Post.to("/api/v1/tracking/shipments/{id}/events")
                .with(request -> request
                    .pathParam("id", shipmentId)  // Sustituye {id}
                    .contentType(ContentType.JSON)
                    .body(eventData))
        );

        // Intenta extraer el ID del evento (si falla, lo ignora)
        try {
            String id = actor.asksFor(TheResponseBody.jsonPath()).getString("id");
            if (id != null) {
                actor.remember("currentEventId", id);
            }
        } catch (Exception ignored) {
            // Mala práctica: oculta errores
        }
    }
}
```

#### Diferencias con `CreateShipment`
- Usa un **Builder** para sintaxis fluida: `forShipment(id).withData(data)`.
- Incluye `{id}` en la URL y lo sustituye con `pathParam`.
- Tiene un `catch (Exception ignored)` que **oculta fallos** (mala práctica, pero presente).

### Ejemplo 3: `GetShipmentInfo` (GET)

```java
public class GetShipmentInfo implements Task {
    
    private final String trackingCode;

    public GetShipmentInfo(String trackingCode) {
        this.trackingCode = trackingCode;
    }

    public static GetShipmentInfo byTrackingCode(String code) {
        return Tasks.instrumented(GetShipmentInfo.class, code);
    }

    @Override
    public <T extends Actor> void performAs(T actor) {
        // GET a /api/v1/shipments/tracking/{code}
        actor.attemptsTo(
            Get.resource("/api/v1/shipments/tracking/{code}")
               .with(request -> request.pathParam("code", trackingCode))
        );
        // No extrae ni recuerda nada; la pregunta lo hace después
    }
}
```

#### Diferencias
- Es una tarea de lectura (GET).
- No extrae valores ni llama `actor.remember(...)`.
- La extracción ocurre en una `Question` posterior.

---

## Phase 3: Questions (Consultas)

### Propósito
Una `Question` en Screenplay es una **consulta que el actor hace para obtener información** de la última respuesta HTTP. Implementa la interfaz `Question<T>` donde `T` es el tipo de dato devuelto.

### Responsabilidades de una Question
1. Acceder a la última respuesta HTTP.
2. Extraer un valor específico (status, cuerpo JSON, etc.).
3. Devolverlo para una aserción.

### Ejemplo 1: `TheResponseStatus`

#### Archivo
`src/test/java/com/example/codefactory/screenplay/questions/TheResponseStatus.java`

#### Construcción

```java
public class TheResponseStatus implements Question<Integer> {
    
    // Método de fábrica
    public static TheResponseStatus code() {
        return new TheResponseStatus();
    }

    // Método principal: devuelve el status HTTP
    @Override
    public Integer answeredBy(Actor actor) {
        // SerenityRest es un helper que accede a la última respuesta
        return SerenityRest.lastResponse().statusCode();
    }
}
```

#### Qué hace
- Accede a la última respuesta HTTP vía `SerenityRest.lastResponse()`.
- Extrae el código de estado (p. ej., `201`, `404`, `400`).
- Lo devuelve como `Integer`.

### Ejemplo 2: `TheResponseBody`

```java
public class TheResponseBody implements Question<JsonPath> {
    
    public static TheResponseBody jsonPath() {
        return new TheResponseBody();
    }

    @Override
    public JsonPath answeredBy(Actor actor) {
        // Devuelve un JsonPath para navegación de JSON
        return SerenityRest.lastResponse().jsonPath();
    }
}
```

#### Qué hace
- Devuelve un `JsonPath` que permite consultar el JSON:
  - `.getString("id")` → extrae el campo `id` como string.
  - `.getInt("status")` → extrae el campo `status` como int.
  - Etc.

### Ejemplo 3: `ShipmentHistory`

```java
public class ShipmentHistory implements Question<String> {
    
    private final String shipmentId;

    public ShipmentHistory(String shipmentId) {
        this.shipmentId = shipmentId;
    }

    public static ShipmentHistory forShipment(String shipmentId) {
        return new ShipmentHistory(shipmentId);
    }

    @Override
    public String answeredBy(Actor actor) {
        // Devuelve todo el cuerpo de la respuesta como String
        return SerenityRest.lastResponse().asString();
    }
}
```

#### Nota
- Recibe un parámetro (`shipmentId`), pero no lo usa realmente.
- Solo devuelve el cuerpo completo como string.
- Es un patrón válido pero podría optimizarse.

---

## Phase 4: Step Definitions (Orquestación)

### Propósito
Un `StepDefinition` es la **implementación de un paso de Cucumber**. Orquesta:
1. La creación de datos (TestDataFactory).
2. La ejecución de tasks (actor.attemptsTo).
3. La verificación con questions (actor.should).

### Archivo
`src/test/java/com/example/codefactory/screenplay/stepdefinitions/ManageShipmentSteps.java`

### Construcción completa

```java
public class ManageShipmentSteps {

    // ============ GIVEN ============
    
    @Given("the logistics operator is ready to interact with the API")
    public void operatorIsReady() {
        // Solo obtiene al actor; el hook lo configura con la habilidad CallAnApi
        OnStage.theActorCalled("Logistics Operator");
    }

    // ============ WHEN ============
    
    @When("they create a new shipment with complete details")
    public void createShipmentWithCompleteDetails() {
        // 1. Obtiene el actor en escena
        var actor = OnStage.theActorInTheSpotlight();
        
        // 2. Genera datos válidos desde TestDataFactory
        var data = TestDataFactory.validShipmentData();
        
        // 3. Crea una Task que el actor ejecuta
        actor.attemptsTo(
            CreateShipment.withData(data)
        );
    }

    // ============ THEN ============
    
    @Then("the shipment is successfully created")
    public void shipmentSuccessfullyCreated() {
        var actor = OnStage.theActorInTheSpotlight();
        
        // Verifica que el status sea 201
        actor.should(
            seeThat("response status", 
                    TheResponseStatus.code(),  // Question
                    equalTo(201))               // Matcher
        );
        
        // Valida que se haya guardado el shipmentId
        String shipmentId = actor.recall("currentShipmentId");
        if (shipmentId == null) {
            throw new AssertionError("Shipment ID was not remembered by CreateShipment task");
        }
    }

    @Then("they can register a package to the shipment")
    public void registerPackage() {
        var actor = OnStage.theActorInTheSpotlight();
        
        // Obtiene el ID del envío que se guardó anteriormente
        String shipmentId = actor.recall("currentShipmentId");
        
        // Genera datos de paquete
        var packageData = TestDataFactory.validPackageData();
        
        // Ejecuta la tarea
        actor.attemptsTo(
            RegisterPackage.forShipment(shipmentId).withData(packageData)
        );
        
        // Verifica
        actor.should(
            seeThat("response status", TheResponseStatus.code(), equalTo(201))
        );
    }
}
```

#### Qué hace cada parte

| Anotación | Propósito |
|-----------|-----------|
| `@Given` | Arranque: prepara el contexto. |
| `@When` | Acción: ejecuta la tarea del actor. |
| `@Then` | Verificación: comprueba el resultado. |

| Método | Propósito |
|--------|-----------|
| `OnStage.theActorCalled()` | Obtiene o crea un actor por nombre. |
| `OnStage.theActorInTheSpotlight()` | Obtiene el actor actual. |
| `actor.recall(clave)` | Lee un valor que el actor recordó. |
| `actor.attemptsTo(Task)` | Ejecuta una tarea. |
| `actor.should(Matcher)` | Verifica una condición. |
| `seeThat(descripción, Question, Matcher)` | Construye una aserción. |

---

## Phase 5: Ejecución Completa

### Flujo general de Cucumber

1. **Lector de Features**: Cucumber lee `manage_shipment.feature`.
2. **Mapper de pasos**: Cucumber encuentra el paso en un `StepDefinition`.
3. **Ejecución del hook**: Se ejecuta `@Before` de `HookSteps`.
4. **Ejecución del step**: Se ejecuta el método anotado con `@When`, etc.
5. **Generación de reporte**: Serenity Screenplay registra todo.

### Flujo dentro de un paso

```
1. StepDefinition (@When)
   ↓
2. TestDataFactory.validShipmentData()
   → Devuelve Map<String, Object> con datos
   ↓
3. actor.attemptsTo(CreateShipment.withData(data))
   ↓
4. CreateShipment.performAs(actor)
   → Post.to("/api/v1/shipments").with(...body(shipmentData))
   → actor.asksFor(TheResponseBody.jsonPath()).getString("id")
   → actor.remember("currentShipmentId", id)
   ↓
5. StepDefinition (@Then)
   ↓
6. actor.should(seeThat("response status", TheResponseStatus.code(), equalTo(201)))
   ↓
7. TheResponseStatus.answeredBy(actor)
   → SerenityRest.lastResponse().statusCode()
   → Devuelve Integer (p. ej., 201)
   ↓
8. Matcher verifica: 201 == 201 ✓
```

---

## Ejemplo Detallado: Happy Path

### Feature
```gherkin
@manage @happy-path
Scenario: Successfully create a shipment and register all components
  Given the logistics operator is ready to interact with the API
  When they create a new shipment with complete details
  Then the shipment is successfully created
  And they can register a package to the shipment
  And they can register a sender to the shipment
  And they can link a vehicle to the shipment
```

### Ejecución paso a paso

#### Paso 1: `Given the logistics operator is ready to interact with the API`

```java
@Given("the logistics operator is ready to interact with the API")
public void operatorIsReady() {
    OnStage.theActorCalled("Logistics Operator");
}
```

**Qué ocurre:**
1. Cucumber ejecuta el hook `@Before` de `HookSteps`.
2. El hook configura el actor "Logistics Operator" con la habilidad `CallAnApi.at(baseUrl)`.
3. El step definition solo confirma que el actor existe.

---

#### Paso 2: `When they create a new shipment with complete details`

```java
@When("they create a new shipment with complete details")
public void createShipmentWithCompleteDetails() {
    OnStage.theActorInTheSpotlight().attemptsTo(
        CreateShipment.withData(TestDataFactory.validShipmentData())
    );
}
```

**Qué ocurre:**

a) **TestDataFactory.validShipmentData()**
```java
Map<String, Object> data = new HashMap<>();
data.put("sender", {nombre: "John Doe xyz", telefono: "555-1234", ...});
data.put("recipient", {nombre: "Jane Doe abc", ...});
data.put("tipoServicio", "EXPRESS");
data.put("nivelPrioridad", 1);
data.put("fechaEnvio", "2026-06-06T12:34:56");
data.put("fechaEstimada", "2026-06-08T12:34:56");
data.put("costoTotal", 150.0);
data.put("instruccionesEnvio", "Handle with care");
// Devuelve el Map
```

b) **CreateShipment.withData(data)**
```java
public static CreateShipment withData(Map<String, Object> data) {
    return Tasks.instrumented(CreateShipment.class, data);
    // Devuelve una instancia de CreateShipment envuelta
}
```

c) **actor.attemptsTo(task)**
```java
// El actor ejecuta:
actor.attemptsTo(Post.to("/api/v1/shipments")
    .with(request -> request
        .contentType(ContentType.JSON)
        .body(shipmentData)));
        
// RestAssured serializa el Map a JSON:
// POST /api/v1/shipments
// Content-Type: application/json
// {
//   "sender": {"nombre": "John Doe xyz", ...},
//   "recipient": {"nombre": "Jane Doe abc", ...},
//   "tipoServicio": "EXPRESS",
//   ...
// }
```

d) **El servidor responde (201 Created)**
```json
{
  "id": "123",
  "codigoRastreo": "TRK-2026060512345",
  "estadoActual": "CREADO",
  ...
}
```

e) **CreateShipment extrae valores**
```java
String id = actor.asksFor(TheResponseBody.jsonPath()).getString("id");
// Devuelve "123"

String tracking = actor.asksFor(TheResponseBody.jsonPath()).getString("codigoRastreo");
// Devuelve "TRK-2026060512345"

actor.remember("currentShipmentId", id);          // id = "123"
actor.remember("currentTrackingCode", tracking);  // tracking = "TRK-2026060512345"
```

---

#### Paso 3: `Then the shipment is successfully created`

```java
@Then("the shipment is successfully created")
public void shipmentSuccessfullyCreated() {
    var actor = OnStage.theActorInTheSpotlight();
    
    actor.should(
        seeThat("response status", TheResponseStatus.code(), equalTo(201))
    );
    
    String shipmentId = actor.recall("currentShipmentId");
    if (shipmentId == null) {
        throw new AssertionError("Shipment ID was not remembered...");
    }
}
```

**Qué ocurre:**

a) **TheResponseStatus.code()** crea una Question que devuelve `Integer`.

b) **TheResponseStatus.answeredBy(actor)**
```java
@Override
public Integer answeredBy(Actor actor) {
    return SerenityRest.lastResponse().statusCode();
    // Accede a la respuesta del último POST
    // Devuelve 201
}
```

c) **Matcher** verifica: `201 == 201` ✓

d) **actor.recall("currentShipmentId")** obtiene `"123"` de la memoria.

e) **Validación**: Si `id != null`, el paso pasa.

---

#### Paso 4: `And they can register a package to the shipment`

```java
@Then("they can register a package to the shipment")
public void registerPackage() {
    String shipmentId = actor.recall("currentShipmentId");  // "123"
    
    actor.attemptsTo(
        RegisterPackage.forShipment(shipmentId)
                      .withData(TestDataFactory.validPackageData())
    );
    
    actor.should(
        seeThat("response status", TheResponseStatus.code(), equalTo(201))
    );
}
```

**Qué ocurre:**

a) **TestDataFactory.validPackageData()**
```java
Map<String, Object> data = new HashMap<>();
data.put("peso", 10.5);
data.put("largo", 20.0);
data.put("ancho", 15.0);
data.put("alto", 30.0);
data.put("descripcion", "Electronics");
// Devuelve el Map
```

b) **RegisterPackage.performAs(actor)**
```java
actor.attemptsTo(
    Post.to("/api/v1/shipments/{id}/package")
        .with(request -> request
            .pathParam("id", "123")  // Sustituye {id}
            .contentType(ContentType.JSON)
            .body(packageData))
);
// POST /api/v1/shipments/123/package
// { "peso": 10.5, ... }
```

c) **Respuesta del servidor (201 Created)**
```json
{
  "id": "456",
  "shipmentId": "123",
  "peso": 10.5,
  ...
}
```

d) **Verificación**: `201 == 201` ✓

---

#### Pasos 5 y 6: `And they can register a sender...` y `And they can link a vehicle...`

Mismo flujo que el paso 4, pero con:
- `RegisterSender.forShipment(shipmentId).withData(TestDataFactory.validSenderData())`
- `LinkVehicle.toShipment(shipmentId).withData(TestDataFactory.validVehicleData())`

---

### Resumen del flujo completo

```
Feature: manage_shipment.feature
  ↓
Scenario: Successfully create a shipment...
  ↓
@Before Hook (HookSteps)
  → Configura actor "Logistics Operator"
  → Le da la habilidad CallAnApi
  ↓
@Given step
  → Confirma que el actor existe
  ↓
@When step (CreateShipment)
  → TestDataFactory.validShipmentData() genera Map
  → CreateShipment envía POST con el Map
  → Servidor responde 201 + JSON
  → CreateShipment extrae id y tracking
  → actor.remember(...) guarda valores
  ↓
@Then step (Verificación)
  → TheResponseStatus.code() devuelve 201
  → Matcher verifica 201 == 201 ✓
  → actor.recall(...) obtiene valores guardados
  ↓
@And step (RegisterPackage)
  → actor.recall("currentShipmentId") obtiene "123"
  → TestDataFactory.validPackageData() genera Map
  → RegisterPackage envía POST a /api/v1/shipments/123/package
  → Servidor responde 201 + JSON
  → Verificación 201 == 201 ✓
  ↓
@And step (RegisterSender)
  → Mismo flujo: POST a /api/v1/shipments/123/sender
  ↓
@And step (LinkVehicle)
  → Mismo flujo: POST a /api/v1/shipments/123/vehicle
  ↓
Serenity genera reporte HTML con todos los pasos y verificaciones
```

---

## Componentes Clave

### Hook: `HookSteps`

```java
public class HookSteps {
    
    @LocalServerPort
    private int port;

    @Before
    public void setTheStage() {
        // 1. Configura el "escenario" de Serenity
        OnStage.setTheStage(new OnlineCast());
        
        // 2. Construye la URL base
        String baseUrl = "http://localhost:" + port;
        
        // 3. Crea un actor y le da la habilidad de hacer llamadas HTTP
        OnStage.theActorCalled("Logistics Operator")
               .whoCan(CallAnApi.at(baseUrl));
    }
}
```

**Propósito:**
- Se ejecuta **antes de cada escenario**.
- Configura el entorno: actor, URL, capacidades.

### Runner: `CucumberRunnerTest`

```java
@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(key = GLUE_PROPERTY_NAME, 
    value = "com.example.codefactory.screenplay.stepdefinitions")
@ConfigurationParameter(key = PLUGIN_PROPERTY_NAME, 
    value = "io.cucumber.core.plugin.SerenityReporterParallel,pretty,timeline:build/test-results/timeline")
public class CucumberRunnerTest {
}
```

**Propósito:**
- Ejecuta todos los `.feature` bajo `src/test/resources/features`.
- Busca step definitions en el paquete indicado.
- Genera reportes de Serenity.

---

## Conclusión

El flujo de ejecución de una prueba Screenplay en este proyecto es:

1. **TestDataFactory** genera datos en formato `Map<String, Object>`.
2. **Task** ejecuta acciones HTTP, extrae valores, los recuerda en el actor.
3. **Question** consulta la última respuesta HTTP.
4. **StepDefinition** orquesta todo: crea datos, ejecuta tareas, verifica resultados.
5. **Hook** prepara el entorno antes de cada escenario.
6. **Cucumber/Serenity** ejecuta los pasos y genera reportes HTML.

Cada componente tiene una responsabilidad clara y se construye siguiendo el patrón Screenplay: el actor es el sujeto principal, las tareas son sus acciones, las preguntas son sus consultas, y los pasos son la orquestación narrativa del test.

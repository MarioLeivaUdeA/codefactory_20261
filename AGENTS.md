# AGENTS.md — Sistema de Tracking Logístico

## 1. Propósito del proyecto

Este repositorio contiene el backend de un **Sistema de Tracking Logístico tipo FedEx**, desarrollado con **Spring Boot** y pensado para simular un contexto real de operación logística.  
El objetivo principal es permitir el **registro, consulta y trazabilidad completa de envíos**, incluyendo:

- creación de envíos;
- asociación de remitente y destinatario;
- generación de identificadores únicos de seguimiento;
- registro de eventos logísticos;
- actualización del estado del envío;
- consulta del historial completo de movimientos;
- generación de reportes operativos sobre tiempos, retrasos y volumen de envíos.

Este proyecto está planteado como un **caso académico**, pero la arquitectura y las decisiones técnicas deben mantenerse **realistas y defendibles en un contexto empresarial**.

---

## 2. Contexto de negocio

Las empresas de logística gestionan miles de envíos diarios entre centros de distribución, vehículos de transporte y destinos finales. Para clientes y operadores, la visibilidad del estado de cada envío es fundamental para garantizar confianza en el servicio.

Muchas organizaciones aún dependen de sistemas fragmentados o actualizaciones manuales que dificultan conocer con precisión dónde se encuentra un paquete y cuál es su estado actual dentro de la cadena logística.

La empresa busca desarrollar una plataforma que permita:

- registrar eventos logísticos;
- ofrecer trazabilidad completa de cada envío;
- consultar el ciclo de vida de un paquete;
- mejorar la transparencia del servicio;
- reducir consultas manuales de clientes;
- proporcionar información operativa para optimizar rutas y tiempos de entrega.

---

## 3. Problema a resolver

Desarrollar un sistema que permita **registrar y consultar el ciclo de vida de un envío** dentro de una red logística.

El sistema debe contemplar:

- registro de envíos y datos del remitente y destinatario;
- generación de identificadores únicos de seguimiento;
- registro de eventos logísticos en cada punto de la cadena de transporte;
- actualización del estado del paquete según su etapa logística;
- consulta del historial completo de movimientos de un envío;
- reportes operativos sobre tiempos de tránsito, retrasos y volumen de envíos.

---

## 4. Alcance funcional esperado

### 4.1 Gestión de envíos
Permite crear y administrar un envío dentro del sistema.

Debe incluir:

- datos del remitente;
- datos del destinatario;
- fecha de envío;
- fecha estimada de entrega;
- costo total;
- instrucciones de envío;
- estado actual;
- código de rastreo.

### 4.2 Identificación y tracking
Cada envío debe tener un **código de rastreo único** que permita:

- consultar el envío públicamente o internamente;
- identificar el paquete de forma unívoca;
- enlazar el envío con su historial y eventos.

### 4.3 Registro de eventos logísticos
El sistema debe registrar eventos como:

- paquete recibido;
- salió del centro logístico;
- llegó a la ciudad destino;
- en reparto;
- entregado.

Cada evento debe incluir:

- tipo de evento;
- fecha y hora;
- ubicación o punto logístico;
- observaciones o detalle si aplica.

### 4.4 Gestión del estado del envío
El estado actual del envío debe actualizarse a medida que ocurren eventos válidos.

Las transiciones de estado deben ser controladas para evitar inconsistencias, por ejemplo:

- no se debe pasar de `ENTREGADO` a `EN_REPARTO`;
- no se debe marcar como entregado un paquete que aún no salió de origen.

### 4.5 Historial y trazabilidad
El historial de tracking debe permitir:

- ver la secuencia completa de movimientos;
- filtrar eventos por fecha;
- filtrar eventos por ubicación;
- consultar el detalle de cada cambio de estado.

### 4.6 Reportes operativos
La solución debe poder soportar consultas para:

- tiempo total de tránsito;
- retrasos frente a la fecha estimada;
- volumen de envíos por estado, fecha o tramo;
- análisis de trazabilidad.

---

## 5. Decisión arquitectónica

La solución se implementará como un **monolito modular** en Spring Boot, organizado con:

- **Clean Architecture** / **Hexagonal Architecture**;
- separación por dominios funcionales;
- integración asíncrona basada en eventos;
- persistencia relacional en **Supabase PostgreSQL**;
- despliegue orientado a Azure.

### 5.1 Por qué esta arquitectura
Esta elección es la más realista para el contexto del proyecto porque:

- el dominio tiene suficiente complejidad para justificar separación modular;
- no es necesario dividir en microservicios desde el inicio;
- facilita el mantenimiento y la defensa académica;
- permite introducir asincronismo sin sobrecomplicar el sistema;
- mantiene independencia entre dominio, aplicación e infraestructura.

### 5.2 Principio rector
**El dominio no debe depender de Spring, de la base de datos ni del bus de mensajería.**

Toda dependencia técnica debe vivir en infraestructura.

---

## 6. Estilo de arquitectura

### 6.1 Arquitectura general
- **Monolito modular**
- **Clean Architecture**
- **Hexagonal Architecture**
- **EDA ligera** para eventos de dominio
- **CQRS ligero** para separar escritura y consulta cuando sea útil

### 6.2 Qué significa en este proyecto
- La lógica de negocio vive en el dominio.
- Los casos de uso orquestan operaciones.
- Los controladores exponen la API HTTP.
- Los adaptadores de infraestructura conectan con PostgreSQL, Supabase y Azure Service Bus.
- Las consultas de historial y reportes pueden optimizarse con proyecciones o consultas dedicadas.

---

## 7. Stack tecnológico esperado

### Backend
- Java
- Spring Boot
- Spring Web
- Spring Data JPA
- Spring Validation
- Spring Security si se requiere autenticación/autorización
- Lombok opcional, si el equipo lo permite
- MapStruct opcional para mapeo entre DTOs y entidades

### Persistencia
- PostgreSQL administrado por Supabase

### Mensajería / asincronismo
- Azure Service Bus

### Infraestructura Azure
- Azure Container Apps o Azure App Service para despliegue del backend
- Azure Key Vault para secretos
- Application Insights para monitoreo y trazas

---

## 8. Modelo de dominio

El modelo de datos base del proyecto incluye las entidades principales:

### 8.1 Persona
Representa remitente o destinatario.

Campos esperados:
- id
- nombre
- teléfono
- correo electrónico
- dirección
- referencias

### 8.2 Envío
Representa la operación logística principal.

Campos esperados:
- id_envio
- id_remitente
- id_destinatario
- tipo_servicio
- nivel_prioridad
- codigo_rastreo
- estado_actual
- fecha_envio
- fecha_estimada
- fecha_actualizacion
- costo_total
- instrucciones_envio

### 8.3 Paquete
Representa el objeto físico transportado.

Campos esperados:
- id_paquete
- id_envio
- peso
- largo
- ancho
- alto
- descripción

### 8.4 EventoLogístico
Entidad que debe existir explícitamente para soportar trazabilidad real.

Debe guardar:
- id
- id_envio
- tipo_evento
- estado_resultante
- fecha_evento
- ubicación
- observaciones
- metadata opcional

### 8.5 UbicaciónLogística
Entidad o concepto de referencia para centros de distribución, ciudades, vehículos o puntos de tránsito.

Puede modelarse como:
- tabla propia,
- catálogo,
- o referencia textual si el alcance es simple.

### 8.6 Estado del envío
Puede manejarse como:
- `enum` de dominio;
- catálogo;
- máquina de estados.

Debe ser controlado y validado.

---

## 9. Reglas de negocio importantes

- Cada envío debe tener un código de rastreo único.
- Un envío puede tener múltiples eventos logísticos.
- El estado actual del envío es un resumen del último evento válido.
- El historial completo debe conservar todos los eventos en orden cronológico.
- Las transiciones de estado deben validarse antes de persistir cambios.
- El sistema debe permitir consultar un envío por ID y por código de tracking.
- El sistema debe soportar consultas de historial por fecha y ubicación.
- Los reportes operativos no deben alterar el modelo transaccional principal.

---

## 10. Organización por módulos

El backend debe dividirse por contexto funcional.  
No se recomienda organizarlo solo por capas técnicas globales.

### Módulos sugeridos

#### 10.1 shipment
Responsable de la creación y consulta de envíos.

Incluye:
- modelo de dominio de envío;
- casos de uso de creación y consulta;
- adaptadores de persistencia;
- controlador REST.

#### 10.2 tracking
Responsable del registro de eventos logísticos y consulta de trazabilidad.

Incluye:
- eventos logísticos;
- historial;
- consultas filtradas;
- controlador REST;
- posibles consumidores de eventos.

#### 10.3 shared
Componentes comunes del sistema.

Incluye:
- excepciones;
- respuesta de error;
- utilidades compartidas;
- contratos comunes si son realmente transversales.

#### 10.4 config
Configuración global del proyecto.

Incluye:
- beans;
- serialización JSON;
- documentación OpenAPI;
- configuración de seguridad si aplica;
- configuración de mensajería.

---

## 11. Estructura de carpetas recomendada

```text
com.logistics.tracking
├── TrackingApplication.java
├── config
├── shared
│   ├── exception
│   ├── handler
│   └── dto
├── shipment
│   ├── domain
│   │   ├── model
│   │   ├── policy
│   │   ├── event
│   │   └── port
│   ├── application
│   │   ├── command
│   │   ├── query
│   │   └── dto
│   ├── infrastructure
│   │   ├── persistence
│   │   │   ├── entity
│   │   │   ├── repository
│   │   │   └── adapter
│   │   ├── messaging
│   │   └── mapper
│   └── presentation
└── tracking
    ├── domain
    │   ├── model
    │   ├── event
    │   └── port
    ├── application
    │   ├── command
    │   ├── query
    │   └── dto
    ├── infrastructure
    │   ├── persistence
    │   │   ├── entity
    │   │   ├── repository
    │   │   └── adapter
    │   └── mapper
    └── presentation
```

---

## 12. Convenciones de implementación

### 12.1 Dominio
El dominio debe ser puro.

No debe contener:
- `@RestController`
- `@Service`
- `@Repository`
- `@Entity`
- anotaciones de mensajería
- detalles de persistencia
- lógica HTTP

Debe contener:
- entidades de negocio;
- value objects;
- reglas;
- políticas;
- eventos de dominio;
- puertos.

### 12.2 Aplicación
La capa de aplicación contiene los casos de uso.

Puede usar:
- `@Service`
- `@Transactional`
- `@Transactional(readOnly = true)`

Debe:
- coordinar el dominio;
- orquestar persistencia;
- publicar eventos;
- no contener reglas de negocio complejas que correspondan al dominio.

### 12.3 Infraestructura
Aquí viven los detalles técnicos:

- `@Entity`
- `@Repository`
- adaptadores de persistencia
- adaptadores de mensajería
- clientes externos
- configuración de Spring

### 12.4 Presentación
Debe contener:
- `@RestController`
- `@RequestMapping`
- `@PostMapping`
- `@GetMapping`
- `@PutMapping`
- `@Valid`
- `@PathVariable`
- `@RequestBody`

---

## 13. Anotaciones de Spring Boot esperadas

### 13.1 Clase principal
```java
@SpringBootApplication
public class TrackingApplication { }
```

### 13.2 Controladores
```java
@RestController
@RequestMapping("/api/shipments")
@RequiredArgsConstructor
```

```java
@RestController
@RequestMapping("/api/tracking")
@RequiredArgsConstructor
```

### 13.3 Casos de uso
```java
@Service
@RequiredArgsConstructor
@Transactional
```

o

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
```

### 13.4 Repositorios de persistencia
```java
@Repository
public interface ShipmentJpaRepository extends JpaRepository<ShipmentEntity, Long> { }
```

### 13.5 Entidades JPA
```java
@Entity
@Table(name = "shipments")
```

### 13.6 Adaptadores
```java
@Component
@RequiredArgsConstructor
```

### 13.7 Manejo global de errores
```java
@RestControllerAdvice
public class GlobalExceptionHandler { }
```

### 13.8 Validación
```java
@Valid
@NotNull
@NotBlank
@Email
@Size
@Positive
```

### 13.9 DTOs
Pueden usar:
- `record` si el equipo lo permite,
- o clases Java simples con getters/setters.

---

## 14. Clases y archivos principales esperados

### 14.1 Capa de presentación
- `ShipmentController`
- `TrackingController`
- `ReportController` si aplica
- `GlobalExceptionHandler`

### 14.2 Capa de aplicación
- `CreateShipmentUseCase`
- `GetShipmentByTrackingCodeUseCase`
- `RegisterLogisticEventUseCase`
- `UpdateShipmentStatusUseCase`
- `GetShipmentHistoryUseCase`
- `GetOperationalReportsUseCase`

### 14.3 Capa de dominio
- `Shipment`
- `Package`
- `Person`
- `LogisticEvent`
- `Location`
- `TrackingCode`
- `ShipmentStatus`
- `ShipmentStatusTransitionPolicy`
- eventos de dominio como:
  - `ShipmentCreatedEvent`
  - `LogisticEventRegisteredEvent`
  - `ShipmentStatusChangedEvent`

### 14.4 Capa de infraestructura
- `ShipmentEntity`
- `PackageEntity`
- `PersonEntity`
- `LogisticEventEntity`
- `ShipmentJpaRepository`
- `LogisticEventJpaRepository`
- `ShipmentRepositoryAdapter`
- `LogisticEventRepositoryAdapter`
- `EventPublisherAdapter`
- `ShipmentMapper`
- `LogisticEventMapper`

### 14.5 Configuración
- `BeanConfiguration`
- `JacksonConfig`
- `OpenApiConfig`
- `SecurityConfig` si aplica

---

## 15. Flujo funcional esperado

### 15.1 Crear envío
1. El controlador recibe la solicitud.
2. El caso de uso valida la información.
3. Se genera el tracking code.
4. Se construye el agregado de envío.
5. Se persiste en Supabase.
6. Se publica un evento de dominio si corresponde.

### 15.2 Registrar evento logístico
1. El controlador recibe el evento.
2. El caso de uso busca el envío.
3. Se valida la transición de estado.
4. Se crea el evento logístico.
5. Se actualiza el estado del envío.
6. Se guarda todo de forma transaccional.
7. Se publica evento asíncrono al bus si aplica.

### 15.3 Consultar historial
1. El controlador recibe el tracking code o id.
2. El caso de uso consulta eventos asociados.
3. Se ordenan por fecha.
4. Se devuelve el historial completo.

---

## 16. Asincronismo y eventos

### 16.1 Cuándo usar asincronismo
El asincronismo se usará para:

- publicación de eventos de dominio;
- actualización de proyecciones de lectura;
- preparación de reportes;
- futuros procesos de notificación;
- integración desacoplada con otros módulos.

### 16.2 Qué no hacer
No convertir todo el sistema en asincrónico.  
Las operaciones críticas de escritura deben seguir siendo transaccionales.

### 16.3 Recomendación técnica
Para un contexto realista, usar:

- **Outbox Pattern** para evitar pérdida de eventos;
- **Azure Service Bus** como mensajería;
- consumidores separados de los casos de uso principales.

---

## 17. Reglas de persistencia

### 17.1 Base de datos
La base relacional es **Supabase PostgreSQL**.

### 17.2 Recomendaciones
- no mezclar lógica de negocio dentro de entidades JPA;
- mapear entre dominio y persistencia mediante adapters o mappers;
- mantener claves primarias simples;
- usar restricciones de integridad referencial;
- guardar el historial como eventos persistidos, no como texto libre;
- evitar sobrescribir información histórica.

### 17.3 Relaciones recomendadas
- un envío tiene un remitente;
- un envío tiene un destinatario;
- un envío tiene uno o varios paquetes;
- un envío tiene muchos eventos logísticos;
- cada evento puede tener una ubicación asociada.

---

## 18. Reportes y consultas de lectura

Los reportes deben separarse conceptualmente de la lógica de escritura.

Se espera poder consultar:

- tiempo promedio de tránsito;
- envíos por estado;
- retrasos frente a fecha estimada;
- volumen de envíos por día o rango de fechas;
- eventos por ubicación;
- trazabilidad por tracking code.

Si se implementan proyecciones, estas deben actualizarse por eventos y no reemplazar el modelo transaccional principal.

---

## 19. Calidad de código esperada

### 19.1 Principios
- claridad sobre complejidad innecesaria;
- nombres explícitos;
- módulos cohesionados;
- bajo acoplamiento;
- validaciones en la capa correcta;
- código mantenible.

### 19.2 Nomenclatura
Usar nombres en inglés para clases, métodos y variables técnicas.

Ejemplos:
- `Shipment`
- `LogisticEvent`
- `TrackingCode`
- `CreateShipmentUseCase`
- `ShipmentRepositoryPort`

Los textos visibles al usuario pueden mantenerse en español si la API o la UI lo requieren.

### 19.3 Reglas para archivos
- un archivo debe tener una sola responsabilidad principal;
- evitar clases “Dios”;
- no mezclar lógica de controller con reglas de negocio;
- no meter consultas SQL manuales en la capa de presentación.

---

## 20. Manejo de errores

Debe existir un manejo global de excepciones.

### Excepciones comunes
- `DomainException`
- `ValidationException`
- `NotFoundException`
- `InvalidShipmentStateException`
- `TrackingCodeAlreadyExistsException`

### Respuesta estándar de error
Debe incluir:
- timestamp;
- código HTTP;
- mensaje;
- detalle opcional;
- path o endpoint afectado.

---

## 21. Pruebas recomendadas

### 21.1 Unitarias
Para:
- políticas de transición de estado;
- generación de tracking code;
- casos de uso;
- reglas de dominio.

### 21.2 Integración
Para:
- persistencia con JPA;
- mapeos entidad-dominio;
- endpoints REST;
- publicación o consumo de eventos.

### 21.3 Qué priorizar
Primero probar:
- creación de envío;
- registro de evento;
- validación de estado;
- consulta de historial.

---

## 22. Criterios de implementación para nuevas tareas

Cuando se agregue una nueva feature, seguir este orden:

1. identificar el módulo funcional;
2. definir o actualizar el modelo de dominio;
3. crear o modificar el caso de uso;
4. agregar puertos si hace falta;
5. implementar adaptadores;
6. exponer el endpoint REST;
7. agregar validaciones;
8. escribir pruebas;
9. revisar consistencia con el historial y eventos.

---

## 23. Restricciones de diseño

No introducir sin necesidad:

- microservicios;
- Redis;
- buses complejos adicionales;
- CQRS completo con demasiada infraestructura;
- Event Sourcing completo;
- arquitectura distribuida sin justificación;
- múltiples bases de datos;
- capas sobrantes sin valor real.

La meta es una solución **realista, escalable y defendible**, no una sobrearquitectura.

---

## 24. Objetivo final de la solución

La aplicación debe demostrar que el sistema:

- registra envíos correctamente;
- rastrea eventos logísticos de punta a punta;
- mantiene un historial confiable;
- valida estados y transiciones;
- permite consulta por tracking code;
- soporta análisis operativos básicos;
- está diseñada con una arquitectura limpia y profesional.

---

## 25. Guía para trabajar en este repositorio

Cuando desarrolles aquí:

- respeta la separación por módulos;
- evita acoplar dominio con Spring;
- documenta claramente cada caso de uso;
- mantén la trazabilidad de los eventos;
- conserva el historial completo de movimientos;
- usa asincronismo solo donde aporte valor;
- prioriza legibilidad sobre sofisticación.

---

## 26. Resumen ejecutivo

Este proyecto es un backend de tracking logístico implementado en Spring Boot, con una arquitectura modular y realista para un entorno empresarial.  
La solución se apoya en:

- dominio bien delimitado;
- persistencia relacional con Supabase PostgreSQL;
- eventos asíncronos para trazabilidad y evolución;
- despliegue en Azure;
- separación clara entre presentación, aplicación, dominio e infraestructura.

La prioridad arquitectónica es construir un sistema **mantenible, trazable y escalable de forma razonable**.

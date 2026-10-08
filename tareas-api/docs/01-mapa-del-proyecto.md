# Mapa del proyecto: tareas-api

## ¿Qué hace este proyecto?

Sistema de gestión de tareas pendientes que permite crear, completar, eliminar y listar tareas con diferentes niveles de prioridad y fechas límite. Utiliza una arquitectura en capas (dominio, repositorio, servicio) con almacenamiento en memoria y está construido con Java 17 y Maven.

## Tabla de clases

| Clase/Interfaz | Responsabilidad | Depende de |
|----------------|-----------------|------------|
| `Tarea` | Entidad de dominio que representa una tarea con atributos (id, título, descripción, prioridad, fecha límite, estado) | `Prioridad` |
| `Prioridad` | Enum que define los niveles de prioridad (BAJA, MEDIA, ALTA) | Ninguna |
| `TareaNoEncontradaException` | Excepción personalizada para operaciones sobre tareas inexistentes | Ninguna |
| `TareaRepositorio` | Capa de persistencia en memoria usando LinkedHashMap; operaciones CRUD | `Tarea` |
| `TareaServicio` | Lógica de negocio y casos de uso (crear, completar, eliminar, listar, reportes) | `Tarea`, `Prioridad`, `TareaRepositorio`, `TareaNoEncontradaException` |
| `App` | Punto de entrada de la aplicación de consola; demo con datos de ejemplo | `TareaServicio`, `Prioridad` |
| `TareaRepositorioTest` | Tests unitarios del repositorio | `TareaRepositorio`, `Tarea`, `Prioridad` |
| `TareaServicioTest` | Tests unitarios de la lógica de negocio | `TareaServicio`, `Prioridad` |

## ¿Cómo se compila y se prueba?

```bash
# Compilar el proyecto
mvn compile

# Ejecutar todos los tests
mvn test

# Ejecutar la aplicación de consola
mvn -q exec:java

# Limpiar, compilar, probar y empaquetar
mvn clean install
```

## Tres cosas sospechosas o incompletas en el código original

### 1. Cálculo de días con signo invertido en `TareaServicio.diasRestantes()`

El método que calcula los días hasta la fecha límite tenía el orden de las fechas invertido, causando que devolviera valores negativos cuando debían ser positivos y viceversa. El cálculo correcto debe restar la fecha actual de la fecha límite, no al revés.

**Ubicación**: `TareaServicio.java` (método `diasRestantes()`)

### 2. Filtro de prioridad con `>` en lugar de `>=` en `TareaServicio.listarPorPrioridadMinima()`

El método que filtra tareas por prioridad mínima usa el operador `>` (mayor que) en lugar de `>=` (mayor o igual que). Esto excluye las tareas que tienen exactamente la prioridad mínima solicitada, cuando la semántica de "prioridad mínima" debería incluirlas.

**Ubicación**: `TareaServicio.java` (método `listarPorPrioridadMinima()`)

### 3. Falta de validación de ID en `TareaServicio.completar()` y `eliminar()`

Los métodos `completar()` y `eliminar()` no validan si el ID proporcionado existe antes de intentar operar sobre la tarea. Si el repositorio devuelve `null` al buscar el ID, los métodos fallan con `NullPointerException` en lugar de lanzar la excepción `TareaNoEncontradaException` que está diseñada para este propósito.

**Ubicación**: `TareaServicio.java` (métodos `completar()` y `eliminar()`)

# tareas-api

Gestor de tareas pendientes: repositorio de práctica para la mentoría técnica con Claude (Generation México, CH70 Java).

## Descripción

`tareas-api` es una aplicación de consola Java que implementa un sistema de gestión de tareas pendientes. Permite crear, completar, eliminar y listar tareas con diferentes niveles de prioridad y fechas límite. El proyecto utiliza una arquitectura en capas (repositorio, servicio, dominio) y está construido con Maven.

## Tecnología

- **Java 17** (compilador y runtime)
- **Maven 3.x** (gestión de dependencias y build)
- **JUnit 5.10.2** (testing)

## Comandos de desarrollo

### Compilación y ejecución

```bash
# Compilar el proyecto
mvn compile

# Ejecutar la aplicación de consola
mvn -q exec:java

# Limpiar y compilar desde cero
mvn clean compile
```

### Testing

```bash
# Ejecutar todos los tests
mvn test

# Ejecutar tests con información detallada
mvn test -X

# Ejecutar solo una clase de test específica
mvn test -Dtest=TareaServicioTest
```

### Empaquetado

```bash
# Generar JAR del proyecto
mvn package

# Limpiar y generar JAR
mvn clean package
```

### Otras operaciones útiles

```bash
# Ver el árbol de dependencias
mvn dependency:tree

# Verificar que el proyecto esté correctamente configurado
mvn validate

# Ejecutar todos los ciclos: limpiar, compilar, probar y empaquetar
mvn clean install
```

## Estructura del proyecto

```
tareas-api/
├── pom.xml                          # Configuración de Maven, dependencias y plugins
├── src/
│   ├── main/
│   │   └── java/
│   │       └── mx/generation/tareas/
│   │           ├── App.java                        # Punto de entrada: demo de consola
│   │           ├── Tarea.java                      # Entidad de dominio: representa una tarea
│   │           ├── Prioridad.java                  # Enum: BAJA, MEDIA, ALTA
│   │           ├── TareaRepositorio.java           # Capa de persistencia en memoria
│   │           ├── TareaServicio.java              # Lógica de negocio (casos de uso)
│   │           └── TareaNoEncontradaException.java # Excepción personalizada
│   └── test/
│       └── java/
│           └── mx/generation/tareas/
│               ├── TareaRepositorioTest.java       # Tests unitarios del repositorio
│               └── TareaServicioTest.java          # Tests unitarios del servicio
└── target/                          # Directorio generado por Maven (ignorado en git)
    ├── classes/                     # Clases compiladas (.class)
    ├── test-classes/                # Clases de test compiladas
    ├── surefire-reports/            # Reportes XML/TXT de los tests
    └── *.jar                        # Artefacto empaquetado (si se ejecutó mvn package)
```

## Descripción de archivos principales

### Capa de dominio

- **`Tarea.java`**: Clase que representa una tarea pendiente con atributos como id, título, descripción, prioridad, fecha límite y estado de completada. Incluye validaciones básicas y el método `marcarCompletada()`.

- **`Prioridad.java`**: Enum que define los tres niveles de prioridad: BAJA, MEDIA y ALTA (ordenados de menor a mayor).

- **`TareaNoEncontradaException.java`**: Excepción que se lanza cuando se intenta operar sobre una tarea con un id que no existe en el repositorio.

### Capa de persistencia

- **`TareaRepositorio.java`**: Implementa el almacenamiento en memoria usando un `LinkedHashMap<Integer, Tarea>`. Proporciona operaciones CRUD básicas:
  - `guardar()`: Asigna un id consecutivo y almacena la tarea
  - `buscar()`: Obtiene una tarea por id (devuelve null si no existe)
  - `eliminar()`: Elimina una tarea por id
  - `todas()`: Devuelve una lista con todas las tareas
  - `buscarPorTitulo()`: Búsqueda parcial en títulos
  - `contar()`: Cuenta total de tareas almacenadas

### Capa de servicio

- **`TareaServicio.java`**: Contiene la lógica de negocio y casos de uso. Actúa como fachada que usaría un controlador REST o una CLI. Métodos principales:
  - `crear()`: Crea y guarda una nueva tarea
  - `completar()`: Marca una tarea como completada
  - `eliminar()`: Elimina una tarea
  - `listarPendientes()`: Devuelve solo las tareas no completadas
  - `listarPorPrioridadMinima()`: Filtra tareas pendientes por prioridad
  - `diasRestantes()`: Calcula días hasta la fecha límite
  - `generarReporte()`: Genera un reporte de texto con estadísticas y estado del proyecto

### Aplicación de consola

- **`App.java`**: Punto de entrada (`main`). Crea un `TareaServicio`, inserta algunas tareas de ejemplo, completa una de ellas y muestra el reporte generado. Útil para demostrar el funcionamiento del sistema.

### Tests

- **`TareaRepositorioTest.java`**: Suite de tests unitarios para verificar las operaciones CRUD del repositorio, asignación de ids, búsqueda por título, etc.

- **`TareaServicioTest.java`**: Tests unitarios de la lógica de negocio, incluyendo creación de tareas, marcado como completadas, filtrado por prioridad, cálculo de días restantes y generación del reporte.

## Arquitectura

El proyecto sigue una arquitectura en capas sencilla:

```
App.java (consola)
    ↓
TareaServicio (lógica de negocio / casos de uso)
    ↓
TareaRepositorio (persistencia en memoria)
    ↓
Tarea, Prioridad (entidades de dominio)
```

Esta separación permite:
- Testear cada capa de forma independiente
- Cambiar la implementación del repositorio (por ejemplo, a base de datos) sin afectar el servicio
- Agregar nuevos clientes (REST API, CLI, GUI) que usen el mismo `TareaServicio`

## TODOs conocidos

- **Validación de título vacío**: El constructor de `Tarea` actualmente acepta títulos vacíos (`""`). Ver comentario en `Tarea.java:19`.

## Notas sobre el proyecto

Este es un proyecto educativo diseñado para practicar:
- Arquitectura en capas
- Testing con JUnit 5
- Maven como herramienta de build
- Buenas prácticas en Java (immutabilidad donde es posible, encapsulación, etc.)
- Identificación y corrección de bugs comunes

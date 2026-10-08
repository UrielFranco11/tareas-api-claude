# tareas-api

Gestor de tareas pendientes en Java 17. Proyecto de práctica de la mentoría
**«Programar con Claude»** (Generation México · CH70 Java · 8-oct-2026).

## Descripción

Sistema de gestión de tareas con arquitectura en capas que permite crear, completar, eliminar y listar tareas con diferentes niveles de prioridad (BAJA, MEDIA, ALTA) y fechas límite. Incluye persistencia en memoria, lógica de negocio y generación de reportes.

## Requisitos

- Java 17 o superior (`java -version`)
- Maven 3.8+ (`mvn -v`)

## Comandos

```bash
mvn test          # corre las pruebas (JUnit 5)
mvn -q exec:java  # corre la demo de consola (App.java)
mvn compile       # compila el proyecto
mvn clean package # genera el JAR
```

## Estructura del Proyecto

```
src/main/java/mx/generation/tareas/
├── App.java                        # Punto de entrada (demo)
├── Tarea.java                      # Entidad de dominio
├── Prioridad.java                  # Enum de prioridades
├── TareaRepositorio.java           # Capa de persistencia
├── TareaServicio.java              # Lógica de negocio
└── TareaNoEncontradaException.java # Excepción personalizada

src/test/java/mx/generation/tareas/
├── TareaRepositorioTest.java       # Tests del repositorio
└── TareaServicioTest.java          # Tests del servicio
```

## Mejoras y Correcciones Implementadas

Durante la sesión de mentoría se identificaron y corrigieron 4 bugs críticos y se realizó 1 refactorización mayor en la capa de servicio. Todos los cambios mantienen compatibilidad con la API existente y están validados por la suite completa de tests.

### 1. Validación de ID Inexistente en `completar()`

**Problema:** El método `completar()` lanzaba `NullPointerException` cuando se intentaba completar una tarea con un ID inexistente, en lugar de la excepción documentada `TareaNoEncontradaException`.

**Solución:** Se reemplazó la llamada directa a `repositorio.buscar(id)` por el método privado `obtener(id)` que encapsula la validación:
```java
// Antes (incorrecto - NullPointerException)
Tarea tarea = repositorio.buscar(id);
tarea.marcarCompletada(); // NPE si id no existe

// Después (correcto - reutilización de lógica)
Tarea tarea = obtener(id); // Lanza TareaNoEncontradaException
tarea.marcarCompletada();
```

**Resultado:** Ahora lanza `TareaNoEncontradaException` con mensaje descriptivo. Reutiliza el método privado `obtener(id)` en lugar de duplicar validación.

**Commit:** `cbb82cc` - fix: completar lanza TareaNoEncontradaException si el id no existe

### 2. Corrección del Cálculo de Días Restantes

**Problema:** El método `diasRestantes()` tenía los parámetros invertidos en `ChronoUnit.DAYS.between()`, causando que devolviera valores con signo opuesto al esperado (negativo para futuro, positivo para pasado).

**Solución:** Se corrigió el orden de los parámetros en el cálculo temporal:
```java
// Antes (incorrecto - signo invertido)
return ChronoUnit.DAYS.between(tarea.getFechaLimite(), hoy);
// Fechas futuras daban negativo, pasadas daban positivo

// Después (correcto - matemática esperada)
return ChronoUnit.DAYS.between(hoy, tarea.getFechaLimite());
// Fechas futuras dan positivo, pasadas dan negativo
```

**Resultado:** Ahora cumple con su contrato documentado: valores positivos para fechas futuras, 0 para hoy, negativos para fechas vencidas.

**Commit:** `601985a` - fix: corregir signo invertido en el calculo de dias restantes

### 3. Corrección del Filtro de Prioridad

**Problema:** El método `listarPorPrioridadMinima()` usaba operador estricto `>` en lugar de inclusivo `>=`, excluyendo incorrectamente las tareas con la prioridad exacta solicitada. El JavaDoc del método especifica "prioridad igual o mayor", pero la implementación solo devolvía las mayores.

**Solución:** Se cambió el operador de comparación para incluir la igualdad:
```java
// Antes (incorrecto - solo mayores, excluye iguales)
if (t.getPrioridad().ordinal() > minima.ordinal())

// Después (correcto - inclusivo, igual o mayor)
if (t.getPrioridad().ordinal() >= minima.ordinal())
```

**Resultado:** Ahora incluye correctamente las tareas con prioridad "igual o mayor" a la indicada. Por ejemplo, `listarPorPrioridadMinima(BAJA)` devuelve todas las tareas (BAJA, MEDIA y ALTA).

**Commit:** `d1e2b9f` - fix: incluir prioridad minima en listarPorPrioridadMinima

### 4. Validación de ID Inexistente en `eliminar()`

**Problema:** El método `eliminar()` devolvía silenciosamente `false` cuando el ID no existía, sin lanzar excepción. Esto era inconsistente con otros métodos del servicio (`completar()`, `diasRestantes()`) que sí lanzan `TareaNoEncontradaException`.

**Solución:** Se agregó validación antes de la operación de eliminación, reutilizando el método privado `obtener(id)`:
```java
// Antes (comportamiento inconsistente - retorna false silenciosamente)
public boolean eliminar(int id) {
    return repositorio.eliminar(id); // false si no existe, sin excepción
}

// Después (correcto - consistente con API del servicio)
public boolean eliminar(int id) {
    obtener(id); // Valida existencia, lanza TareaNoEncontradaException
    return repositorio.eliminar(id);
}
```

**Resultado:** Ahora lanza `TareaNoEncontradaException` cuando el ID no existe, manteniendo consistencia con el resto del API del servicio. Reutiliza la lógica de validación existente en lugar de duplicarla.

**Commit:** `e7a7873` - fix: lanzar TareaNoEncontradaException al intentar eliminar un id inexistente

### 5. Refactorización del Método `generarReporte()`

**Problema:** El método `generarReporte()` presentaba múltiples code smells:
- Concatenación de strings con operador `+` (O(n²) en memoria por creación de objetos inmutables)
- Lógica monolítica de ~50 líneas mezclando cálculo de estadísticas y formateo
- Condicionales anidados difíciles de seguir
- Contador manual de tareas cuando existe `repositorio.contar()`

**Solución:** Se aplicó refactorización Extract Method y uso de `StringBuilder`:
- Reemplazo de concatenación de strings (`String r = r + ...`) por `StringBuilder` para eficiencia O(n)
- Extracción de 5 métodos privados con responsabilidades únicas:
  - `contarCompletadas()`: Cuenta tareas completadas
  - `contarVencidas(LocalDate)`: Cuenta tareas pendientes vencidas sin condicionales anidados
  - `contarAltasPendientes()`: Cuenta tareas pendientes de alta prioridad
  - `calificarAvance(int, int)`: Genera texto de estado según porcentaje (EXCELENTE/BIEN/ATRASADO)
  - `listarPendientesTexto()`: Genera listado formateado de tareas pendientes
- Eliminación de condicionales anidados usando operador lógico `&&` (de 3 niveles a 1)
- Uso del método `repositorio.contar()` existente

```java
// Antes (concatenación ineficiente - crea 10+ objetos String temporales)
String r = "";
r = r + "Total: " + total + "\n";
if (total > 0) {
    if (porcentaje >= 80) {
        r = r + "Estado: EXCELENTE (" + porcentaje + "%)\n";
    } else if (porcentaje >= 50) {
        // ...
    }
}

// Después (StringBuilder + métodos cohesivos)
StringBuilder sb = new StringBuilder();
sb.append("Total: ").append(total).append("\n");
sb.append(calificarAvance(total, completadas)); // Lógica aislada
```

**Resultado:** 
- Código más mantenible: cada método tiene una responsabilidad clara y nombre descriptivo
- Mejor rendimiento: StringBuilder evita creación de objetos temporales
- Mayor testeabilidad: métodos privados pueden ser probados independientemente
- **Salida idéntica**: el test `reporteConTareas` valida que no hubo cambios funcionales

**Commit:** `636412f` - refactor: usar StringBuilder y extraer métodos privados en generarReporte

## Suite de Pruebas

El proyecto cuenta con una suite completa de tests unitarios que validan tanto la funcionalidad correcta como el manejo de casos de error:

### Tests de `TareaServicioTest.java` (12 tests)
- **Creación y modificación de tareas:**
  - `crearYRecuperarTarea`: Validación del flujo de creación
  - `completarTarea`: Cambio de estado a completada
  - `completarInexistenteLanzaExcepcion`: Validación de excepción (agregado)
  
- **Eliminación:**
  - `eliminarTareaExitosa`: Eliminación correcta y retorno `true`
  - `eliminarIdInexistenteLanzaExcepcion`: Validación de excepción (agregado)

- **Filtrado y listado:**
  - `listarPendientesExcluyeCompletadas`: Filtro por estado
  - `listarPorPrioridadMinimaBajaDevuelveMediaYAlta`: Filtro inclusivo por prioridad
  - `listarPorPrioridadMinimaBajaConPendientesSoloMedia`: Casos límite

- **Cálculo de días:**
  - `diasRestantesDeUnaTareaFutura`: Fechas futuras (positivo)
  - `diasRestantesSinFechaEsMaximo`: Tareas sin fecha límite

- **Reportes:**
  - `reporteConTareas`: Validación carácter por carácter de formato
  - `reporteSinTareas`: Caso vacío

### Tests de `TareaRepositorioTest.java` (5 tests, 1 skipped)
- Operaciones CRUD básicas
- Asignación de IDs consecutivos
- Búsqueda por título

### Resultados de Pruebas

```
Tests run: 17, Failures: 0, Errors: 0, Skipped: 1
BUILD SUCCESS
```

Todas las pruebas pasan exitosamente, incluyendo:
- 5 tests de `TareaRepositorioTest` (1 skipped)
- 12 tests de `TareaServicioTest`

## Funcionalidades Principales

- ✅ Crear tareas con título, descripción, prioridad y fecha límite
- ✅ Marcar tareas como completadas
- ✅ Eliminar tareas con validación de existencia
- ✅ Listar tareas pendientes
- ✅ Filtrar tareas por prioridad mínima
- ✅ Calcular días restantes hasta la fecha límite
- ✅ Generar reportes con estadísticas y estado del proyecto
- ✅ Manejo robusto de errores con excepciones personalizadas

## Resumen de Commits

La rama `feat/flujo-con-claude` incluye 5 commits atómicos que implementan las mejoras documentadas:

```
e7a7873 - fix: lanzar TareaNoEncontradaException al intentar eliminar un id inexistente
d1e2b9f - fix: incluir prioridad minima en listarPorPrioridadMinima
601985a - fix: corregir signo invertido en el calculo de dias restantes
636412f - refactor: usar StringBuilder y extraer métodos privados en generarReporte
cbb82cc - fix: completar lanza TareaNoEncontradaException si el id no existe
```

Cada commit es independiente, con un único propósito, mensaje descriptivo en [Conventional Commits](https://www.conventionalcommits.org/) y validado por tests.

## Próximas Mejoras Pendientes

- **Validación de título vacío**: El constructor de `Tarea` acepta títulos vacíos (`""`). Pendiente agregar validación según comentario en `Tarea.java:19`.

## Documentación Adicional

Para más detalles sobre la arquitectura, estructura de archivos y comandos avanzados, consulta el archivo `CLAUDE.md` en la raíz del proyecto.

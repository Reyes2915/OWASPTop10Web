# OWASP Top 10: Manejo Inadecuado de Condiciones Excepcionales

> **Resumen Ejecutivo:** Analiza los riesgos de seguridad derivados de una gestión deficiente de errores y excepciones en aplicaciones web. Explora cómo la no prevención, la no detección y las respuestas inadecuadas pueden desencadenar denegaciones de servicio (DoS), exposición de información sensible y corrupción de transacciones, proporcionando estrategias de mitigación y estandarización a nivel organizacional.

---

## 📋 Tabla de Contenidos
- [1. Conceptos Fundamentales](#1-conceptos-fundamentales)
- [2. Principios, Implementación o Metodología](#2-principios-implementación-o-metodología)
- [3. Buenas Prácticas y Trade-offs](#3-buenas-prácticas-y-trade-offs)
- [4. Cheat Sheet / Referencia Rápida](#4-cheat-sheet--referencia-rápida)

---

## 1. Conceptos Fundamentales

### Diferencia entre Error y Condición Excepcional

* **Error:** Es el resultado definitivo de una operación que ha fallado. Representa un estado final sobre el cual la operación ya finalizó de forma fallida.
* **Condición Excepcional:** Es una situación anormal o inesperada durante la ejecución que deja al sistema en un **estado indeterminado o inconsistente** (por ejemplo, una transacción donde no se sabe si se descontó el dinero o si llegó al destino). Requiere una intervención o tratamiento específico para llevar la aplicación de vuelta a un estado seguro.

### Tipos de Fallos en la Gestión de Excepciones (OWASP)

1. **No Prevención:** La condición anormal era predecible pero no se configuró el entorno o las validaciones previas para evitar que ocurriera.
2. **No Detección:** Ocurre la condición excepcional pero el sistema carece de mecanismos para identificarla y capturarla oportunamente.
3. **Mala Respuesta:** Se detecta el fallo pero la respuesta implementada es incorrecta, manteniendo o amplificando la inconsistencia del sistema en lugar de resolverla.

### Vector de Riesgos Principal

* **Agotamiento de Recursos (DoS):** Reintentos automáticos tras fallos de carga o procesos no finalizados sin liberación de memoria o espacio en disco.
* **Fuga de Información (Information Disclosure):** Exposición de trazas de pila (*stack traces*), rutas del sistema, versiones de frameworks o consultas SQL internas al usuario final.
* **Corrupción del Estado de Transacciones:** Operaciones compuestas (ej. transferencias) que quedan a medias por interrupciones de red o fallos de conexión a BD sin mecanismo de reversión.

---

## 2. Principios, Implementación o Metodología

### Escenarios Prácticos de Vulnerabilidad

#### Escenario A: Exposición de Trazas de Pila por Validación Tipológica
Al ingresar un tipo de dato no esperado en un parámetro numérico (`GET /product?id=test`), la aplicación no valida el tipo de entrada (`string` vs `integer`).

```text
// Respuesta con error interno expuesto al cliente:
java.lang.IllegalArgumentException: Invalid input format 'test' for type integer
    at com.example.store.ProductController.get(ProductController.java:42)
    at org.apache.struts2.dispatcher.filter.StrutsPrepareAndExecuteFilter.doFilter(...)
Server: Apache Struts 2.3.31
Database: SQL Error, Account ID unknown in table 'customers'
```

> ⚠️ **Advertencia:** La exposición de *stack traces* revela el lenguaje de backend (Java), frameworks usados (`Apache Struts 2.3.31`), estructura de la base de datos y líneas de código exactas, facilitando vectores de ataque dirigidos.

#### Escenario B: Inconsistencia en Transacciones Multipaso
En una transferencia bancaria de dos pasos (1. Descontar origen → 2. Abonar destino), una falla de comunicación entre el paso 1 y 2 sin un manejo adecuado provoca la pérdida de consistencia financiera.

```
[Cuenta Origen] --(-$200)--> [Error de Red] --X--> [Cuenta Destino (+$0)]
                                   │
                           (Sin Rollback)
                                   │
                   ┌───────────────┴───────────────┐
                   ▼                               ▼
      Inconsistencia de datos             Pérdida de fondos
```

### Arquitectura para la Gestión Segura de Excepciones

1. **Tratamiento Cercano al Origen:** Capturar la excepción en el punto exacto de ejecución para mantener el contexto operativo.
2. **Rollback Automático en Transacciones:** Si una operación compuesta falla en cualquiera de sus etapas, se debe revertir el estado o registrar un evento de compensación.
3. **Manejador Global de Excepciones (Global Exception Handler):** Actuar como última línea de defensa centralizada para capturar errores no controlados y formatear respuestas genéricas orientadas al usuario.
4. **Agregación y Deduplicación de Logs:** Agrupar excepciones repetitivas para evitar el colapso del sistema de monitorización o SIEM.

---

## 3. Buenas Prácticas y Trade-offs

| Aspectos Positivos / Recomendados | Riesgos / Errores Comunes |
| :--- | :--- |
| **Mensajes Genéricos al Usuario:** Retornar códigos de error estandarizados sin detalles técnicos. | **Trazas en Producción:** Mostrar *stack traces*, nombres de tablas o versiones de frameworks al cliente. |
| **Rollback Transaccional:** Deshacer cambios parciales ante fallos de conexión o dependencias externas. | **Estado Incompleto:** Permitir que una transacción continúe tras un fallo intermediate en la base de datos. |
| **Agregación de Logs de Errores:** Registrar la primera ocurrencia con contadores estadísticos de repetición. | **Saturación de Logs (Log Flooding):** Escribir una entrada por cada error repetido en bucles de reintentos. |
| **Rate Limiting y Cuotas de Recursos:** Limitar endpoints o recursos críticos para prevenir agotamiento por reintentos. | **Reintentos Infinitos:** Mantener reservas de disco/memoria en reintentos fallidos sucesivos sin un límite. |
| **Validación Estricta de Entradas:** Comprobar tipos de datos y estructuras previa al procesamiento de la lógica. | **Confianza implícita en la entrada:** Intentar parsear tipos incompatibles directamente en capa de persistencia. |

> 💡 **Punto Clave:** La gestión de excepciones debe estandarizarse a nivel organizacional (no solo por proyecto) para garantizar que las herramientas de observabilidad y sistemas SIEM puedan parsear, correlacionar y alertar eventos de seguridad de forma consistente.

---

## 4. Cheat Sheet / Referencia Rápida

### Checklist de Implementación Segura

- [ ] **Validación de Tipo y Formato:** Sanitizar y validar tipos de entrada (ej. casteos de string a int) antes de pasarlos a la lógica de negocio o BD.
- [ ] **Sanitización de Salidas de Error:** Ocultar detalles del sistema (clases, BD, versiones) y enviar únicamente un identificador único de evento al usuario (`Error ID: #10492`).
- [ ] **Manejador Global de Errores:** Configurar *catch-all* globales para interceptar fallos imprevistos a nivel de aplicación web.
- [ ] **Gestión de Recursos (`try-with-resources` / `finally`):** Asegurar la liberación explícita de archivos, sockets o conexiones a BD tras una excepción.
- [ ] **Límite de Reintentos y Rate Limiting:** Establecer máximos de reintentos para cargas de archivos u operaciones costosas con cuotas de infraestructura.
- [ ] **Transaccionalidad Atomica (ACID):** Aplicar patrones de reversión automática (Rollback) en transacciones multipaso ante fallos intermedios.
- [ ] **Agregación de Logs para SIEM:** Formatear los registros de errores con esquemas estructurados (ej. JSON) y agrupar excepciones idénticas con contadores.
- [ ] **Requisitos de Seguridad Previos:** Definir cómo responderá la aplicación ante cada caso límite (*edge case*) desde la fase de diseño.
- [ ] **Pruebas de Estrés y PenTesting:** Evaluar el comportamiento del sistema simulando fallos de red, datos corruptos e inyecciones de tipos.
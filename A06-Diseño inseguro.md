# OWASP Top 10: A06 - Insecure Design (Diseño Inseguro)

**Resumen Ejecutivo:** Guía de referencia técnica sobre la categoría A06: Insecure Design del OWASP Top 10. Se analiza la diferencia fundamental entre fallos de diseño y defectos de implementación, los vectores de ataque comunes basados en abusos de lógica de negocio y las estrategias clave de AppSec para integrar la seguridad desde la arquitectura.

---

## 📋 Tabla de Contenidos
1. [Conceptos Fundamentales](#1-conceptos-fundamentales)
2. [Principios, Implementación o Metodología](#2-principios-implementación-o-metodología)
3. [Buenas Prácticas y Trade-offs](#3-buenas-prácticas-y-trade-offs)
4. [Cheat Sheet / Referencia Rápida](#4-cheat-sheet--referencia-rápida)

---

## 1. Conceptos Fundamentales

El **Diseño Inseguro (Insecure Design)** representa la falta o ineficacia de controles de seguridad en las etapas conceptuales y de arquitectura de una aplicación, independientemente de si la implementación técnica posterior es perfecta.

### Diferenciación Clave

*   **Fallo de Diseño (Design Flaw):** Ocurre cuando el control de seguridad necesario no fue previsto en la arquitectura o el mecanismo concebido es intrínsecamente ineficaz. El código puede estar perfectamente escrito según las especificaciones, pero el comportamiento del sistema sigue siendo vulnerable porque no existen reglas o salvaguardas de diseño.
*   **Defecto de Implementación (Implementation Bug):** Ocurre cuando existía un control de seguridad correctamente especificado en la fase de diseño, pero durante el desarrollo se omitió, se configuró mal o se programó con errores.

```text
                  ┌─────────────────────────────────────────┐
                  │    Fase de Arquitectura y Diseño        │
                  └────────────────────┬────────────────────┘
                                       │
                  ┌────────────────────┴────────────────────┐
                  │ ¿Existe un control eficaz diseñado?     │
                  └──────────┬───────────────────┬──────────┘
                             │                   │
                     NO (Fallo de Diseño)    SÍ (Control Diseñado)
                             │                   │
                             ▼                   ▼
                  ┌────────────────────┐ ┌────────────────────┐
                  │ Comportamiento     │ │ ¿Se codificó bien? │
                  │ Inseguro por       │ └─────────┬──────────┘
                  │ Definición         │           │
                  └────────────────────┘    NO (Defecto de Impl.)
                                                   │
                                                   ▼
                                         ┌────────────────────┐
                                         │  Vulnerabilidad    │
                                         │  en Código         │
                                         └────────────────────┘
```

---

## 2. Principios, Implementación o Metodología

### Tres Pilares del Diseño Seguro (OWASP)

1.  **Definición de Requisitos y Recursos:** Identificar el riesgo de negocio, clasificar la información sensible a proteger y determinar los recursos necesarios desde el inicio.
2.  **Diseño Seguro (Secure Design):** Evaluar amenazas, modelar controles de acceso/lógica de negocio y definir el comportamiento esperado del sistema ante situaciones anómalas o errores.
3.  **Ciclo de Vida de Desarrollo Seguro (SSDLC):** Integrar la seguridad de forma holística en la fase de concepción, pruebas, despliegue y mantenimiento.

### Escenarios de Ataque y Análisis de Casos de Abuso

#### Escenario 1: Recuperación de Cuentas con Preguntas Secretas
*   **Problema:** El flujo permite restablecer contraseñas respondiendo preguntas personales (ej. *"¿Nombre de tu primera mascota?"*).
*   **Fallo de Diseño:** La información utilizada no constituye un secreto criptográfico ni confidencial (es fácilmente obtenible mediante ingeniería social o OSINT en redes sociales). Aunque la función sea programada sin bugs, la lógica subyacente es insegura.

#### Escenario 2: Reservas Masivas sin Depósito / Garantía
*   **Problema:** Un sistema de reservas de cine permite apartar entradas sin exigir pago, depósito o límite de tasa (*rate limiting*).
*   **Fallo de Diseño / Caso de Abuso:** Un atacante automatiza peticiones HTTP para acaparar 600 asientos instantáneamente. No requiere explotar ningún fallo de desbordamiento o inyección; explota el propio proceso permitido por la lógica de negocio.

#### Escenario 3: Compras Automatizadas con Bots (Acaparamiento de Inventario)
*   **Problema:** Lanzamiento de productos con stock limitado sin mecanismos anti-bot o validación de comportamiento humano.
*   **Fallo de Diseño:** Las transacciones se procesan técnicamente bien, pero el diseño no consideró el comportamiento automatizado abusivo, permitiendo la reventa y denegación de servicio a usuarios legítimos.

#### Escenario 4: Confianza Excesiva en Validaciones Frontend (Laboratorio Práctico)
*   **Problema:** Modificación arbitraria de precios al añadir un producto al carrito o completar el checkout.
*   **Análisis de Vulnerabilidad:** Las comprobaciones de precio existen únicamente en el cliente (navegador).

**Flujo del Ataque (Ejemplo mediante Intercepción de Petición HTTP):**

1. Petición Original del Cliente (Capturada en Proxy / Burp Suite):
```http
POST /cart HTTP/1.1
Host: tienda-vulnerable.com
Content-Type: application/x-www-form-urlencoded

productId=1&quantity=1&price=1337.00
```

2. Manipulación del Atacante (Bypassing Frontend Controls):
```http
POST /cart HTTP/1.1
Host: tienda-vulnerable.com
Content-Type: application/x-www-form-urlencoded

productId=1&quantity=1&price=5.00
```

3. **Causa Raíz:** El backend confía en el campo `price` enviado por la solicitud del usuario en lugar de consultar la base de datos o validar el importe en el servidor antes de autorizar la orden.

### Principios Arquitectónicos de Mitigación

*   **Validación en Múltiples Capas (Defense in Depth):** Las validaciones en frontend sirven para la experiencia de usuario (UX) y para mitigar carga innecesaria, pero las únicas validaciones con valor de seguridad real son las ejecutadas en el servidor (backend).
*   **Aislamiento Multi-Tenant:** En arquitecturas compartidas (*multi-tenant*), el aislamiento entre organizaciones debe garantizarse en la lógica de negocio, consultas de datos y capa de red, nunca mediante la simple ocultación visual en la UI.
*   **Segmentación de Red:** Colocar bases de datos y servicios críticos en redes privadas/VPCs accesibles de forma exclusiva mediante el principio de menor privilegio.

---

## 3. Buenas Prácticas y Trade-offs

| Aspectos Positivos / Recomendados | Riesgos / Errores Comunes |
| :--- | :--- |
| **Integrar AppSec desde la fase 0:** Evaluar riesgos y controles antes de escribir código. | **Diseño orientado únicamente al "Happy Path":** Asumir que el usuario siempre usará la interfaz de la forma prevista. |
| **Utilizar Patrones y Librerías Estándar:** Emplear marcos de trabajo consolidados para autenticación, autorización y encriptación. | **Reinventar la Rueda:** Intentar implementar mecanismos criptográficos o de sesión propios en lugar de componentes auditados. |
| **Modelado de Amenazas Continuo:** Realizar sesiones de Threat Modeling tanto al inicio como tras cambios significativos de arquitectura. | **Delegar Seguridad Únicamente al Frontend:** Confiar en inputs o parámetros de precios/roles que proceden del navegador del cliente. |
| **Definición de "Casos de Abuso":** Incluir historias de usuario negativas/de abuso junto con los requisitos funcionales. | **Modelado de Amenazas Aislado:** Tratar la seguridad como un evento puntual antes del lanzamiento y no como proceso continuo. |

> 💡 **Punto Clave:** El usuario legítimo sigue la interfaz; el atacante interactúa directamente con la lógica subyacente y los vectores de la red. Una aplicación puede ser 100% libre de bugs de código y seguir siendo totalmente vulnerable si su diseño permite flujos de abuso de negocio.

---

## 4. Cheat Sheet / Referencia Rápida

### Checklist de Diseño Seguro para Senior Tech Leads

- [ ] **Threat Modeling en Historias de Usuario:** ¿Cada historia crítica especifica la matriz de autorización, casos de borde y qué ocurre si la operación no está autorizada?
- [ ] **Validación Estricta Server-Side:** ¿Todas las reglas financieras, de estado y de cálculo de precios se revalidan en el backend antes de persistir la transacción?
- [ ] **Defensa Anti-Automatización / Abuso Lógico:** ¿Existen controles (rate limiting, CAPTCHA, tokens de un solo uso o depósitos) para operaciones susceptibles de acaparamiento o spam?
- [ ] **Autenticación Fuerte:** ¿Se han eliminado las preguntas de seguridad y métodos débiles de recuperación de cuenta en favor de MFA/TOTP o enlaces de un solo uso de vida corta?
- [ ] **Aislamiento Multi-Tenant:** ¿Las consultas a la base de datos fuerzan explícitamente el filtro `tenant_id` en la capa de persistencia/ORM sin confiar en parámetros enviados por la URL?
- [ ] **Pruebas de Casos de Abuso:** ¿El suite de pruebas (unitarias/integración) incluye tests automatizados que simulen manipulación de parámetros HTTP y comportamiento fuera del flujo esperado?
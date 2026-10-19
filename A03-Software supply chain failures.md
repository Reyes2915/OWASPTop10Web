# Gestión de Seguridad en la Cadena de Suministro de Software (OWASP Top 10)

> **Resumen Ejecutivo:** Documento técnico de referencia sobre la identificación, análisis y mitigación de fallos en la cadena de suministro de software. Explora los vectores de ataque comunes (incluyendo dependencias transitivas y contaminación de pipelines CI/CD), demuestra un caso de estudio practico de DOM-XSS/Prototype Pollution vía librerías obsoletas y detalla las mejores prácticas recomendadas por OWASP.

---

## 📋 Tabla de Contenidos
- [1. Conceptos Fundamentales](#1-conceptos-fundamentales)
- [2. Principios, Implementación o Metodología](#2-principios-implementación-o-metodología)
- [3. Buenas Prácticas y Trade-offs](#3-buenas-prácticas-y-trade-offs)
- [4. Cheat Sheet / Referencia Rápida](#4-cheat-sheet--referencia-rápida)

---

## 1. Conceptos Fundamentales

### Definición
La **cadena de suministro de software** comprende todos los componentes externos, herramientas, plataformas y procesos que intervienen durante el ciclo de vida del desarrollo, construcción, distribución y actualización de una aplicación. Esto incluye:
- Librerías y frameworks de terceros.
- Herramientas de desarrollo, IDEs y estaciones de trabajo.
- Sistemas de integración y despliegue continuo (CI/CD) y repositorios de artefactos.
- Registros de imágenes de contenedores y servicios SaaS integrados.

### Vectores de Falla y Riesgo
Los riesgos no provienen únicamente del código propio, sino de la confianza ciega en elementos de terceros. Las fallas principales se dividen en:

1. **Falta de seguimiento de dependencias transitivas:**
   - Una aplicación puede incluir directamente la `Librería A (v2.4)`, pero esta incluye internamente la `Librería B`. Sin visibilidad sobre `Librería B`, vulnerabilidades graves pueden ser introducidas de forma inadvertida.
2. **Ausencia de separación de funciones (SoD):**
   - Inexistencia de controles que impidan que un único actor modifique código e impulse cambios a producción sin revisión independiente o aprobaciones (Pull Requests / Merge Requests).
3. **Uso de fuentes no confiables:**
   - Descarga de dependencias desde repositorios no oficiales o pirateados para "ahorrar costes", los cuales suelen incluir código malicioso inyectado (*backdoors*).
4. **Contaminación del Pipeline CI/CD:**
   - Si el sistema de construcción (CI/CD) carece de controles adecuados, un atacante puede alterar la etapa de empaquetado para inyectar código malicioso directamente en los artefactos de producción, aun cuando el código fuente en el repositorio esté limpio.
5. **Configuraciones deficientes y falta de Hardening:**
   - Inexistencia de MFA en repositorios, secretos harcodeados, permisos laxos y falta de parches en herramientas de desarrollo o estaciones de trabajo.

---

## 2. Principios, Implementación o Metodología

### Caso Práctico: Explotación de DOM-XSS mediante Librería Obsoleta

A continuación se documenta el flujo de trabajo para evaluar y explotar una vulnerabilidad en una librería de terceros desactualizada mediante pruebas de seguridad dinámicas (DAST):

#### Flujo de Análisis de Vulnerabilidad (DOM Invader & Burp Suite)

1. **Configuración del Entorno de Pruebas:**
   - Utilizar el navegador embebido de Burp Suite (Chromium).
   - Habilitar la extensión **DOM Invader**.
   - Configurar la extensión: Activar la detección de **Prototype Pollution** y recargar el sitio web bajo análisis.

2. **Identificación de Vectores:**
   - Abrir Developer Tools -> Pestaña **DOM Invader**.
   - Ejecutar `Scan for gadgets` para auditar automáticamente el código JavaScript en busca de puntos de inyección.

3. **Explotación Controlada:**
   - Al finalizar el escaneo automatizado, DOM Invader identifica un vector de contaminación de prototipos.
   - Ejecutar la acción `Exploit`. La herramienta inyecta un payload JavaScript neutro mostrando una alerta `alert(1)`, confirmando la ejecución de código en el contexto del navegador del usuario (DOM-XSS).

4. **Análisis de Causa Raíz (Root Cause Analysis):**
   - Inspección del Stack Trace en la consola del navegador.
   - Identificación del archivo origen de la falla: `ga.js`.
   - **Diagnóstico:** El archivo `ga.js` corresponde a una versión antigua y descontinuada de Google Analytics que contiene vulnerabilidades conocidas de sanitización de entradas, permitiendo DOM-XSS y Prototype Pollution.

```javascript
// Diagnóstico conceptual: Ejemplo de cómo una versión obsoleta procesa entradas no verificadas
// ga.js (Versión vulnerable de terceros)
function parseQueryString(url) {
    var params = {};
    var query = url.split('?')[1];
    if (query) {
        var pairs = query.split('&');
        for (var i = 0; i < pairs.length; i++) {
            var pair = pairs[i].split('=');
            // Vulnerabilidad: Asignación directa a propiedades sin sanitizar (Prototype Pollution / DOM XSS)
            params[decodeURIComponent(pair[0])] = decodeURIComponent(pair[1]);
        }
    }
    return params;
}
```

---

## 3. Buenas Prácticas y Trade-offs

| Aspectos Positivos / Recomendados | Riesgos / Errores Comunes |
| :--- | :--- |
| **Generación y mantenimiento continuo de SBOM (Software Bill of Materials):** Inventariar componentes directos y transitivos. | Confiar únicamente en la revisión inicial al momento de instalar la librería sin dar seguimiento continuo. |
| **Escaneo automatizado de vulnerabilidades (SCA):** Uso de herramientas como OWASP Dependency-Track, Dependency-Check o Retire.js asociadas a bases CVE/NVD/OSV. | Mantener dependencias sin soporte o desactualizadas para evitar "gastos en horas-hombre" o refactorización. |
| **Descarga desde fuentes oficiales y verificación de firma:** Consumir únicamente paquetes con firmas digitales e integridad verificable. | Descargar componentes de repositorios alternativos, de terceros o pirateados para ahorrar licencias. |
| **Despliegues escalonados (Canary / Blue-Green):** Probar las actualizaciones de dependencias en entornos acotados antes del *rollout* global. | Aplicar parches o actualizaciones masivas simultáneas en todos los servidores de producción sin estrategia de rollback. |
| **Separación de funciones (SoD) y PRs protegidas:** Requerir revisiones de código y autenticación multifactor (MFA) para promover cambios. | Permitir que una sola persona configure CI/CD, programe código y realice despliegues a producción directamente. |
| **Inmutabilidad y versionado de Infraestructura como Código (IaC):** Promover el mismo artefacto firmado entre entornos y versionar archivos IaC. | Reconstruir el artefacto en cada entorno (Dev/Staging/Prod), introduciendo inconsistencias. |

> 💡 **Punto Clave:** Cuando la actualización de una librería vulnerable no sea técnicamente viable a corto plazo por romper compatibilidad, se debe evaluar la migración a un componente alternativo o aplicar **Parches Virtuales** (por ejemplo, reglas en WAF o filtros intermediarios) para mitigar el vector de ataque mientras se refactoriza el sistema.

---

## 4. Cheat Sheet / Referencia Rápida

### Checklist de Seguridad para la Cadena de Suministro

- [ ] **Inventario (SBOM):** ¿Existe un SBOM actualizado que contemple librerías directas y transitivas?
- [ ] **Análisis de Dependencias (SCA):** ¿Se ejecutan análisis automatizados en el pipeline CI/CD integrando bases CVE/NVD/OSV?
- [ ] **Control de Repositorios:** ¿Está activado el MFA obligatorio, la protección de ramas (`main`/`release`) y la restricción de directos a producción?
- [ ] **Gestión de Secretos:** ¿Se ha verificado que no existen credenciales, tokens o claves privadas hardcodeadas en repositorios o scripts IaC?
- [ ] **Integridad de Artefactos:** ¿Los artefactos generados por el pipeline de CI/CD están firmados digitalmente y sellados con marca de tiempo?
- [ ] **Hardening del Pipeline:** ¿Están aislados los entornos de construcción y cuentan con secretos específicos e independientes por ambiente (Dev/Stg/Prod)?
- [ ] **Gestión de Desuso:** ¿Se identificaron e inhabilitaron librerías *unmaintained* o sin soporte activo?

### Herramientas Recomendadas por OWASP
- **Software Composition Analysis (SCA):** `OWASP Dependency-Track`, `OWASP Dependency-Check`.
- **Análisis JavaScript / Frontend:** `Retire.js`.
- **Bases de Datos de Vulnerabilidades:** `NVD` (National Vulnerability Database), `OSV` (Open Source Vulnerabilities).
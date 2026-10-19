# OWASP Top 10: Configuraciones de Seguridad Incorrectas (Security Misconfiguration)

> **Resumen Ejecutivo:** Documentación técnica sobre la categoría de *Security Misconfiguration* de OWASP Top 10. Se definen las causas comunes de vulnerabilidades por malas configuraciones, se analiza una demostración práctica de filtrado de información crítica mediante Burp Suite y se detallan las recomendaciones arquitectónicas para implementar hardening y despliegues seguros.

---

## 📋 Tabla de Contenidos
- [1. Conceptos Fundamentales](#1-conceptos-fundamentales)
- [2. Principios, Implementación o Metodología](#2-principios-implementación-o-metodología)
- [3. Buenas Prácticas y Trade-offs](#3-buenas-prácticas-y-trade-offs)
- [4. Cheat Sheet / Referencia Rápida](#4-cheat-sheet--referencia-rápida)

---

## 1. Conceptos Fundamentales

La **Configuración de Seguridad** comprende los ajustes que determinan cómo se protegen los recursos, servicios y componentes dentro de una aplicación. La seguridad de un componente no depende únicamente de que funcione correctamente a nivel funcional; un sistema puede responder sin errores aparentes mientras opera bajo condiciones de alto riesgo.

### Definición del Problema
Una **Configuración de Seguridad Incorrecta (Security Misconfiguration)** se presenta cuando un sistema, aplicación o servicio en la nube utiliza parámetros que introducen vulnerabilidades o deshabilitan protecciones esenciales.

OWASP identifica **8 escenarios principales** que generan esta vulnerabilidad:

1. **Funcionalidades innecesarias habilitadas/instaladas:** Mantenimiento de puertos, servicios, páginas de ejemplo (ej. `/samples`), cuentas o frameworks de prueba en entornos de producción que amplían la superficie de ataque.
2. **Cuentas y credenciales predeterminadas:** Uso de usuarios y contraseñas de fábrica no modificados (ej. `admin/admin`), permitiendo el compromiso inmediato del sistema.
3. **Exposición de mensajes de error detallados:** Ausencia de un manejo de errores centralizado, lo que expone excepciones técnicas (*stack traces*, rutas internas, componentes) que facilitan el reconocimiento al atacante.
4. **Encabezados y directivas de seguridad ausentes o inseguras:** Falta de cabeceras HTTP de protección como `Content-Security-Policy` (CSP), dejando la aplicación vulnerable a XSS, Clickjacking o secuestro de sesiones.
5. **Ausencia de Hardening:** Falta de endurecimiento de componentes e infraestructura, incluyendo permisos laxos o lectura pública no intencionada en almacenamiento cloud (ej. buckets S3).
6. **Funciones de seguridad deshabilitadas o mal configuradas:** No parametrizar ni activar las nuevas capacidades de seguridad incluidas en actualizaciones o releases de software.
7. **Configuraciones inseguras por retrocompatibilidad:** Mantener componentes o protocolos obsoletos (*legacy*) que fuerzan la degradación del nivel de seguridad de toda la plataforma.
8. **Valores de configuración inseguros en la pila de software:** Establecer intencionalmente o por omisión parámetros laxos en servidores web, bases de datos, frameworks o librerías.

---

## 2. Principios, Implementación o Metodología

### Caso Práctico: Information Disclosure via Debug Page (PortSwigger Lab)

A continuación se detalla la metodología para identificar y explotar una fuga de información provocada por una configuración en modo depuración (*debug mode*).

#### Objetivo
Obtener la variable de entorno sensible `secret_key` expuesta debido a una mala configuración del entorno PHP.

#### Flujo de Explotación Paso a Paso

1. **Mapeo de la aplicación con Burp Suite:**
   - Navegar por el sitio objetivo utilizando el navegador integrado de Burp Suite.
   - En la pestaña **Target > Sitemap**, inspeccionar la estructura de recursos cargados por la aplicación.

2. **Identificación del recurso mal configurado:**
   - Identificar archivos de configuración o diagnóstico expuestos públicamente. En este escenario, el sitemap revela la ruta:
     ```text
     /cgi-bin/phpinfo.php
     ```

3. **Análisis de la respuesta e información expuesta:**
   - Acceder al recurso expuesto desde el navegador.
   - La vista de `phpinfo()` revela detalles internos del sistema: rutas del servidor, versiones del entorno, directivas de PHP y variables de entorno (`Environment`).

4. **Extracción del secreto:**
   - Filtrar la sección `Environment` hasta localizar la variable clave:
     ```text
     secret_key = <VALOR_SECRET_KEY>
     ```

> ⚠️ **Advertencia:** Los archivos de diagnóstico como `phpinfo.php` o endpoints de *actuators/debug* nunca deben ser accesibles en entornos de producción.

---

## 3. Buenas Prácticas y Trade-offs

| Aspectos Positivos / Recomendados | Riesgos / Errores Comunes |
| :--- | :--- |
| **Despliegue Seguro e Integrado:** Definir e implementar el hardening como parte integral del pipeline CI/CD, no como un paso posterior. | **Seguridad por Oscuridad o Postergada:** Desplegar la aplicación y postergar la configuración de seguridad para "después de verificar que funciona". |
| **Automatización del Hardening:** Utilizar la infraestructura como código (IaC) para garantizar una línea base (*baseline*) idéntica en todos los entornos. | **Ajustes Manuales e Improvisados:** Aplicar cambios de configuración directamente en los servidores de producción sin versionar. |
| **Entornos Mínimos:** Eliminar frameworks, dependencias, cuentas predeterminadas y páginas de muestra no utilizadas. | **Superficie de Ataque Innecesaria:** Mantener librerías de prueba o servicios activos "por si acaso" se necesitan a futuro. |
| **Aislamiento y Segmentación:** Separar recursos de distintos clientes mediante segmentación de red, contenedores e identidades independientes por entorno. | **Reutilización de Credenciales:** Compartir las mismas claves o bases de datos entre entornos de Dev, QA y Producción. |
| **Gestión de Secretos Dinámica:** Utilizar identidades administradas (IAM Roles, Managed Identities) y credenciales de corta duración. | **Secretos Hardcodeados:** Almacenar API keys, passwords o tokens en el código fuente, archivos de configuración o scripts de despliegue. |
| **Manejo Centralizado de Errores:** Configurar respuestas genéricas para el usuario final y registrar *stack traces* internamente. | **Exposición de Stack Traces:** Mostrar errores HTTP 500 con detalles técnicos del framework o la base de datos en el frontend. |

> 💡 **Punto Clave:** La seguridad de una aplicación debe comprobarse y auditarse de forma continua tras cada despliegue. Si no es posible automatizar las verificaciones, deben ejecutarse revisiones manuales con una frecuencia mínima anual o adaptada a la tasa de cambios de la infraestructura.

---

## 4. Cheat Sheet / Referencia Rápida

### Checklist de Hardening Esencial

- [ ] **Desactivación de Modos Debug:** Garantizar que `DEBUG=False` o equivalente esté configurado en producción.
- [ ] **Limpieza de Recursos:** Eliminar archivos de prueba (`phpinfo.php`, `/samples`, `/test`),Endpoints de diagnóstico no autenticados y credenciales por defecto.
- [ ] **Cabeceras HTTP de Seguridad Básicas:**
  - `Content-Security-Policy`: Restringir el origen de ejecución de scripts y recursos.
  - `X-Frame-Options`: Prevenir ataques de Clickjacking (`DENY` o `SAMEORIGIN`).
  - `X-Content-Type-Options`: Prevenir MIME-sniffing (`nosniff`).
  - `Strict-Transport-Security` (HSTS): Forzar el uso de conexiones seguras HTTPS.
- [ ] **Permisos Cloud e IAM:**
  - Verificar que los buckets S3 o almacenamiento de blobs no tengan acceso de lectura público por defecto.
  - Asignar permisos bajo el principio de menor privilegio (*Least Privilege*).
- [ ] **Gestión de Secretos:**
  - Sustituir credenciales estáticas por soluciones de gestión de identidades (*Managed Identities*, *Vaults*).
  - Asegurar la separación total de credenciales entre los entornos de Desarrollo, QA y Producción.
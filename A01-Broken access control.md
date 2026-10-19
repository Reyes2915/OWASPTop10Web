# OWASP Top 1: Control de Acceso Roto (Broken Access Control)

> **Resumen Ejecutivo:** Este documento sintetiza los conceptos, vectores de ataque y estrategias de mitigación asociadas al fallo de Control de Acceso Roto (Broken Access Control), clasificado como el riesgo #1 en el OWASP Top 10. Se analiza la distinción fundamental entre autenticación y autorización, los escenarios típicos de vulnerabilidad (IDOR, Forced Browsing, escalada de privilegios, manipulación de metadatos/CORS) y el conjunto de controles defensivos del lado del servidor necesarios para garantizar el principio de mínimo privilegio.

---

## 📋 Tabla de Contenidos
- [1. Conceptos Fundamentales](#1-conceptos-fundamentales)
- [2. Principios, Implementación o Metodología](#2-principios-implementación-o-metodología)
- [3. Buenas Prácticas y Trade-offs](#3-buenas-prácticas-y-trade-offs)
- [4. Cheat Sheet / Referencia Rápida](#4-cheat-sheet--referencia-rápida)

---

## 1. Conceptos Fundamentales

### Definición de Control de Acceso
El **Control de Acceso** es el mecanismo de seguridad encargado de restringir los recursos, datos y funcionalidades que un usuario o proceso puede ejecutar dentro de una aplicación, en función de sus permisos explícitamente asignados.

### Diferenciación Clave: Autenticación vs. Autorización

| Concepto | Mecanismo | Pregunta Clave |
| :--- | :--- | :--- |
| **Autenticación (AuthN)** | Verificación de identidad (credenciales, MFA, SSO). | *¿Quién es el usuario?* |
| **Autorización (AuthZ)** | Validación de derechos de acceso y matriz de permisos. | *¿Qué tiene permitido hacer este usuario autenticado?* |

> ⚠️ **Advertencia:** Una autenticación exitosa **no implica** acceso irrestricto. Un fallo en el control de acceso ocurre cuando la capa de autorización permite a un usuario ejecutar acciones o consultar datos fuera del alcance asignado a su rol o perfil.

### Impacto del Control de Acceso Roto
* **Exposición de información confidencial:** Acceso no autorizado a PII, reportes o datos financieros de terceros.
* **Modificación o eliminación no autorizada de datos:** Manipulación de registros por usuarios sin privilegios.
* **Ejecución de funciones de negocio restringidas:** Acceso a paneles administrativos o endpoints de gestión por cuentas convencionales.

---

## 2. Principios, Implementación o Metodología

### Vectors de Vulnerabilidad Comunes

1. **Modificación de URLs y Parámetros del Navegador:**
   * Ocurre cuando las reglas de autorización dependen de variables enviadas por el cliente (`/report?approved=false` $\rightarrow$ `/report?approved=true`).
   * *Causa Raíz:* Confianza ciega en parámetros manipulables del lado del cliente.

2. **Referencia Insegura Directa a Objetos (IDOR / BOLA):**
   * Ocurre cuando se accede o modifica un recurso utilizando identificadores directos proporcionados por el cliente (`GET /account/1001` $\rightarrow$ `GET /account/1002`) sin validar la propiedad o permiso sobre dicho recurso.

3. **Navegación Forzada (Forced Browsing):**
   * Acceso directo a endpoints o recursos restringidos omitiendo la interfaz o flujo esperado (ej. acceder a `/internal_reports` o `/admin` directamente vía URL sin haber iniciado sesión o sin contar con el rol adecuado).

4. **Violación del Principio de Mínimo Privilegio:**
   * Asignación excesiva de permisos a roles operativos (ej. usuario de soporte con permisos para eliminar usuarios). La falta de mantenimiento continuo de la matriz RBAC/ABAC agrava este problema en sistemas de alta escala.

5. **Falta de Control a Nivel de Métodos HTTP / APIs:**
   * Permitir el uso de verbos HTTP sensibles (`POST`, `PUT`, `DELETE`) en endpoints de API sin verificar la autorización específica para dicha acción.

6. **Manipulación de Metadatos y Configuración Insegura:**
   * Alteración de tokens JWT no firmados o mal validados, cookies de sesión o campos `hidden` en formularios HTML.
   * **CORS Permisivo:** Políticas CORS mal configuradas que permiten a orígenes no confiables leer respuestas de la API.

---

### Caso Práctico de Laboratorio: *Unprotected Admin Functionality*

* **Escenario:** Panel de administración no protegido indexado inadvertidamente.
* **Vector:** Reconocimiento manual mediante inspección de `/robots.txt`.
* **Hallazgo:**
  ```http
  User-agent: *
  Disallow: /administrator-panel
  ```
* **Explotación:** Acceso directo navegando a `https://target.com/administrator-panel` omitiendo comprobaciones de rol/sesión, permitiendo la ejecución de acciones administrativas (ej. eliminación de usuarios).

---

### Patrones de Mitigación e Implementación Defensiva

#### Regla de Oro: Enforzamiento del lado del servidor
Todas las comprobaciones de autorización **deben** ejecutarse estrictamente en el servidor. Nunca se debe confiar en parámetros, cookies o datos provenientes del cliente.

#### Ejemplo de Implementación Insegura vs. Segura (Pseudo-código)

**❌ Inseguro (Validación en Frontend / Parámetro en URL):**
```python
# EL SERVIDOR CONFÍA EN UN PARÁMETRO ENVIADO POR EL CLIENTE
@app.route('/view_report')
def view_report():
    is_approved = request.args.get('approved')
    if is_approved == 'true':
        return render_template('confidential_report.html')
    return "Acceso denegado", 403
```

**✅ Seguro (Enforzamiento en Servidor con Contexto de Sesión):**
```python
# EL SERVIDOR VERIFICA LA SESIÓN AUTENTICADA Y LA MATRIZ DE PERMISOS
@app.route('/view_report')
@require_authentication
def view_report():
    user = get_current_user_from_session()
    
    if not user.has_permission('READ_CONFIDENTIAL_REPORTS'):
        log_authorization_failure(user.id, resource='/view_report')
        return "Acceso denegado", 403
        
    return render_template('confidential_report.html')
```

---

## 3. Buenas Prácticas y Trade-offs

| Aspectos Positivos / Recomendados | Riesgos / Errores Comunes |
| :--- | :--- |
| **Denegación por defecto (*Deny by Default*):** Todo recurso o endpoint debe ser inaccesible a menos que exista una regla explícita de permiso. | **Permisividad por defecto:** Exponer rutas o endpoints por omisión y proteger solo URL específicas. |
| **Mecanismos Reutilizables de Autorización:** Utilizar middleware o librerías auditadas para enforzar el control de acceso en toda la app. | **Lógica de autorización dispersa:** Fragmentar controles de acceso condicionales dentro de la lógica de negocio de cada módulo. |
| **Validación de Propiedad de Recursos (Anti-IDOR):** Verificar que `session.user_id == resource.owner_id`. | **Confiar en identificadores del cliente:** Asumir que el ID enviado en la petición pertenece al usuario autenticado. |
| **Invalidación Estricta de Sesión:** Destruir identificadores de sesión en servidor al hacer logout y usar JWTs de corta duración. | **Tokens/Sesiones imperecederas:** Tokens JWT sin tiempo de expiración (`exp`) o sin mecanismo de revocación/OAuth. |
| **Control de Exposición de Archivos:** Deshabilitar directory listing y restringir metadatos/backups (`.git`, `.env`). | **Exposición de artefactos:** Dejar archivos de respaldo o repositorios expuestos en carpetas públicas (`/public`). |
| **Rate Limiting y Logging de Auditoría:** Registrar intentos fallidos de autorización para análisis forense y limitar tasa de peticiones en APIs. | **Ausencia de registros:** Ignorar fallos de autorización continuos, impidiendo la detección de ataques de fuerza bruta o escaneo. |

> 💡 **Punto Clave:** El control de acceso debe diseñarse de forma centralizada y reutilizable. La lógica de negocio no debe depender de banderas enviadas por el navegador ni de URLs ocultas (seguridad por oscuridad).

---

## 4. Cheat Sheet / Referencia Rápida

### Checklist de Seguridad para Control de Acceso (SDLC)

- [ ] **Denegación Por Defecto:** ¿Está restringido todo recurso/endpoint a menos que se autorice explícitamente?
- [ ] **Enforzamiento en Servidor:** ¿Se realizan todas las comprobaciones de autorización exclusivamente en el servidor?
- [ ] **Mitigación de IDOR:** ¿Se valida que el usuario autenticado tiene derecho sobre el `object_id` solicitado en la petición?
- [ ] **Mínimo Privilegio:** ¿Los roles de usuario están acotados estrictamente a las operaciones necesarias para sus funciones?
- [ ] **Control de Métodos HTTP:** ¿Están protegidos de manera uniforme los verbos `GET`, `POST`, `PUT`, `DELETE` en las APIs?
- [ ] **Gestión de Sesiones & JWT:**
  - [ ] ¿Se invalidan las sesiones en el servidor tras el *logout*?
  - [ ] ¿Tienen los tokens JWT un tiempo de vida corto (`exp`)?
  - [ ] ¿Existe un mecanismo de revocación/renovación seguro?
- [ ] **Hardening de Servidor:**
  - [ ] Listing de directorios desactivado.
  - [ ] Archivos confidenciales (`.git`, `.env`, copias de respaldo) fuera de la raíz web publica (`webroot`).
  - [ ] Políticas CORS restringidas únicamente a orígenes confiables explicitados.
- [ ] **Monitoreo y Resiliencia:**
  - [ ] *Rate limiting* activo en endpoints críticos y APIs.
  - [ ] Logging y alertas ante fallos repetidos de autorización (`HTTP 403`).
- [ ] **Pruebas Automatizadas:** ¿Se incluyen pruebas unitarias e integración en el pipeline CI/CD que verifiquen violaciones de control de acceso (ej. usuario estándar intentando ejecutar acciones de admin)?
# OWASP Top 10: Inyecciones – Fundamentos, Vectores de Ataque y Mitigación

**Resumen Ejecutivo:** Documento técnico de referencia sobre la vulnerabilidad de Inyecciones (OWASP Top 10). Detalla la mecánica de explotación (SQLi, Command Injection, XSS, NoSQL, ORM, LDAP), el impacto de interpretar entradas no confiables como instrucciones y las defensas efectivas basadas en la separación de datos y comandos.

---

## 📋 Tabla de Contenidos

1. [Conceptos Fundamentales](#1-conceptos-fundamentales)
2. [Principios, Implementación o Metodología](#2-principios-implementación-o-metodología)
3. [Buenas Prácticas y Trade-offs](#3-buenas-prácticas-y-trade-offs)
4. [Cheat Sheet / Referencia Rápida](#4-cheat-sheet--referencia-rápida)

---

## 1. Conceptos Fundamentales

### Definición y Mecánica

Una inyección se produce cuando datos no confiables (suministrados por un cliente o fuente externa) se envían a un intérprete (base de datos, motor SO, servidor LDAP, navegador) como parte de un comando o consulta sin la debida validación, filtrado o sanitización.

Esto provoca que el intérprete procese la entrada del usuario como código/instrucción ejecutable, alterando la lógica prevista por la aplicación.

```
[ Entrada del Usuario ] ──( Concatenación Insegura )──> [ Intérprete ] ──> [ Ejecución Maliciosa ]
```

### Causas Raíz

1. **Confianza implícita en la entrada:** Considerar seguros los datos recibidos de parámetros HTTP, campos de formulario o encabezados.
2. **Consultas dinámicas e inseguras:** Construcción de sentencias mediante concatenación directa de cadenas.
3. **Uso incorrecto o ausente de ORMs/APIs:** Invocación directa a motores subyacentes o deshabilitación de la parametrización nativa en consultas complejas.
4. **Falta de sanitización/validación de entrada:** Omisión de esquemas estrictos de validación en el lado del servidor.

### Principales Vectores de Inyección

- **Inyección SQL (SQLi):** Alteración de la estructura de consultas SQL en bases de datos relacionales para eludir autenticación, extraer datos masivos o modificar registros.
- **Inyección de Comandos (OS Command Injection):** Inserción de metacaracteres del sistema operativo (p. ej., `;`, `&&`, `|`) para ejecutar binarios no autorizados en el servidor host.
- **Cross-Site Scripting (XSS):** Inyección de código ejecutable (JavaScript/HTML) que el servidor almacena o refleja, ejecutándose en el contexto del navegador de otros usuarios.
- **Inyección NoSQL:** Manipulación de consultas en bases de datos documentales/clave-valor (e.g., MongoDB) mediante el envío de objetos de consulta u operadores maliciosos.
- **Inyección en ORM:** Explotación de vulnerabilidades cuando el ORM ejecuta consultas en bruto (*raw queries*) o cuando la abstracción es mal utilizada.
- **Inyección LDAP:** Manipulación de los filtros de búsqueda hacia servidores de directorio (Active Directory / LDAP), permitiendo escalada de privilegios o bypass de autenticación.

---

## 2. Principios, Implementación o Metodología

### 2.1 Anatómicamente: Explotación y Bypass de Logins (SQLi)

#### Consulta Insegura por Concatenación

```sql
-- Código vulnerable en el backend
SELECT * FROM users WHERE username = 'USER_INPUT' AND password = 'PASSWORD_INPUT';
```

#### Vector de Ataque 1: Bypass por Expresión Tautológica (`OR 1=1`)

- **Entrada enviada:** `1' OR 1=1 --`
- **Consulta resultante:**
  ```sql
  SELECT * FROM users WHERE username = '1' OR 1=1 --' AND password = '...';
  ```
- **Efecto:** La condición `1=1` evalúa siempre a verdadero (`TRUE`), anulando los filtros y devolviendo todos los registros de la tabla.

#### Vector de Ataque 2: Truncamiento por Comentario de Línea

- **Entrada en username:** `administrator' --`
- **Entrada en password:** `cualquier_valor`
- **Consulta resultante:**
  ```sql
  SELECT * FROM users WHERE username = 'administrator' --' AND password = 'cualquier_valor';
  ```
- **Efecto:** Los caracteres `--` fuerzan al motor SQL a ignorar el resto de la consulta (la verificación de contraseña), devolviendo la sesión del usuario `administrator`.

---

### 2.2 Inyección de Comandos del Sistema Operativo

#### Escenario
Una funcionalidad web consulta registros DNS mediante la ejecución interna del comando `nslookup`.

#### Explotación
- **Entrada normal:** `example.com`
- **Entrada maliciosa:** `example.com; whoami`
- **Línea de comandos ejecutada en el servidor:**
  ```bash
  nslookup example.com; whoami
  ```
- **Impacto:** El intérprete ejecuta el segundo comando (`whoami`), revelando el usuario bajo el cual corre el servicio web. Un atacante puede encadenar esto para establecer reverse shells, descargar malware o tomar control total del servidor.

---

### 2.3 Cross-Site Scripting (XSS)

#### Escenario
Un sistema de comentarios que no sanitiza ni codifica la salida HTML.

#### Explotación
- **Entrada en el campo de comentario:**
  ```html
  <script>alert('XSS');</script>
  ```
- **Resultado:** El script se persiste en la base de datos o se refleja en el DOM del usuario. Al cargar la página, el navegador interpreta la etiqueta `<script>` y ejecuta el código en el contexto de la sesión de la víctima.

---

### 2.4 Arquitectura de Protección y Prevención

Para prevenir inyecciones, la regla fundamental de arquitectura es **separar los datos de los comandos y consultas**.

```
  Datos del Usuario   ──> [ Consulta Parametrizada / ORM ] ──> [ Motor SQL ]
(Tratados como texto)         (Estructura compilada)         (Ejecución Segura)
```

1. **Consultas Parametrizadas (Prepared Statements):** Aseguran que la estructura de la consulta se compile previamente en el motor de base de datos. Cualquier parámetro del usuario es tratado estrictamente como literal de datos, nunca como código ejecutable.
2. **Uso Correcto de ORM:** Los frameworks ORM abstraen la creación de consultas. Sin embargo, se deben evitar las llamadas a consultas SQL crudas (*raw SQL queries*) dentro del ORM.
3. **Validaciones de Entrada (Lista Blanca / Tipado Estricto):** Verificar en el backend que el dato cumple con el formato, tipo y rango esperado (p. ej., validar que un ID sea estrictamente un entero numérico) antes de procesarlo.
4. **Escape Contextual:** Utilizado en contextos específicos donde no sea posible la parametrización directa. Requiere aplicar las reglas de escape exactas para el intérprete particular (HTML, JS, SQL, Shell).

---

## 3. Buenas Prácticas y Trade-offs

| Aspectos Positivos / Recomendados | Riesgos / Errores Comunes |
| :--- | :--- |
| **Separación estructural de datos y código:** Priorizar consultas parametrizadas (*Prepared Statements*) y APIs seguras. | **Concatenación de cadenas:** Construir sentencias SQL, LDAP o comandos mediante compilación dinámica de strings. |
| **Uso de ORM estandarizado:** Delegar la generación de consultas al ORM sin recurrir a invocaciones *raw SQL*. | **Confianza en validaciones cliente/frontend:** Implementar controles solo en JS de navegador, fácilmente eludibles. |
| **Validación de entrada por lista blanca (Allow-listing):** Restringir tipos de datos, patrones de expresiones regulares y rangos permitidos en el servidor. | **Uso de listas negras (Block-listing):** Intentar filtrar caracteres especiales individuales (p. ej., eliminar `'` o `;`), lo cual es ineficaz frente a desvíos (*bypasses*). |
| **Principio de menor privilegio:** Configurar las cuentas de acceso a base de datos con los permisos mínimos estrictamente necesarios (evitar roles *sysadmin* o *root*). | **Escape inconsistente:** Aplicar funciones de escape sin considerar el contexto exacto del intérprete objetivo. |

> ⚠️ **Advertencia:** La validación de entrada y el escape de caracteres no son defensas completas por sí solas. Un cambio en el contexto del sistema o en la construcción de la consulta puede anular la eficacia del escape. La defensa principal debe ser siempre la parametrización de consultas y la separación explícita entre datos y comandos.

---

## 4. Cheat Sheet / Referencia Rápida

### Reglas de Oro para la Prevención de Inyecciones

1. **Jamás concatenar entradas de usuario:** Ningún dato proviniente de un cliente debe unirse mediante operadores de cadena (`+`, `.`, `${}`) a sentencias ejecutables.
2. **Parametrizar por defecto:** Emplear *Prepared Statements* / Placeholders (`?`, `:param`) en todas las interacciones con bases de datos relacionales y NoSQL.
3. **Manejo de llamadas al SO:** Evitar invocar comandos de shell directamente (`exec()`, `system()`). Usar las APIs o librerías nativas del lenguaje de programación.
4. **Validar tipos en el Servidor:** Forzar tipado estricto (enteros, UUIDs, enums, fechas) en el backend antes de pasar parámetros a cualquier capa lógica.
5. **Codificación de Salida (XSS):** Aplicar *contextual encoding* (e.g., HTML Entity Encoding) antes de renderizar datos dinámicos en la interfaz del cliente.

### Patrones de Código: Vulnerable vs. Seguro

#### SQL Injection

- ❌ **Vulnerable:**
  ```python
  cursor.execute("SELECT * FROM users WHERE name = '" + user_input + "'")
  ```
- ✅ **Seguro (Parametrizado):**
  ```python
  cursor.execute("SELECT * FROM users WHERE name = %s", (user_input,))
  ```

#### OS Command Injection

- ❌ **Vulnerable:**
  ```python
  os.system("nslookup " + domain)
  ```
- ✅ **Seguro (API nativa / Subprocess con lista explícita):**
  ```python
  subprocess.run(["nslookup", domain], check=True)
  ```
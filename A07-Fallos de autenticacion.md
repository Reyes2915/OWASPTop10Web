# OWASP Top 10: Fallos de Autenticación y Gestión de Sesiones

**Resumen Ejecutivo:** Documentación técnica integral sobre los vectores de ataque en mecanismos de autenticación y gestión de sesiones (OWASP Top 10). Analiza desde la enumeración de usuarios mediante discrepancias en respuestas hasta vulnerabilidades por fijación, reutilización e invalidación deficiente de tokens (JWT), proporcionando estrategias concretas de mitigación bajo estándares NIST y OWASP.

---

## 📋 Tabla de Contenidos
1. [Conceptos Fundamentales](#1-conceptos-fundamentales)
2. [Principios, Implementación o Metodología](#2-principios-implementación-o-metodología)
3. [Buenas Prácticas y Trade-offs](#3-buenas-prácticas-y-trade-offs)
4. [Cheat Sheet / Referencia Rápida](#4-cheat-sheet--referencia-rápida)

---

## 1. Conceptos Fundamentales

### Definición y Alcance
La autenticación es el proceso criptográfico/lógico mediante el cual una aplicación verifica la identidad declarada por un sujeto (usuario, proceso o servicio). En las arquitecturas web modernas, la seguridad de la autenticación trasciende la validación inicial de credenciales (login); abarca el ciclo de vida completo de la sesión y los mecanismos de autorización posteriores.

### Taxonomía de Fallos de Autenticación
Los fallos ocurren cuando el sistema permite el acceso legítimo a un actor no autorizado debido a deficiencias en el diseño o implementación de los controles.

```text
                  ┌───────────────────────────────────────────────┐
                  │    Vectores de Fallo en Autenticación         │
                  └───────────────────────┬───────────────────────┘
                                          │
    ┌───────────────────────┬─────────────┴─────────────┬──────────────────────┐
    │                       │                           │                      │
┌───▼───────────────────┐ ┌─▼─────────────────────────┐ ┌─▼──────────────────┐ ┌─▼───────────────────┐
│ Ataques Automatizados │ │ Gestión de Sesión Débil  │ │ Credenciales Débiles│ │ Fallos de Contexto  │
│ - Credential Stuffing │ │ - Fixation / Reuse       │ │ - Password Leaks    │ │ - Wrong Scope/Aud   │
│ - Password Spraying   │ │ - Invalidación incompleta│ │ - Defaults en Admin │ │ - Búsquedas URL     │
└───────────────────────┘ └──────────────────────────┘ └─────────────────────┘ └─────────────────────┘
```

### Principales Vectores de Vulnerabilidad
1. **Ataques Automatizados y Credential Stuffing:** Automatización masiva de intentos con bases de datos de credenciales filtradas. Incluye variantes híbridas (como mutaciones de contraseñas conocidas, ej. `winter2025` a `winter2026`).
2. **Ausencia o Bypassing de MFA (Multi-Factor Authentication):** Dependencia exclusiva de un único factor (conocimiento/contraseña) sin exigir factores secundarios válidos (posesión/tokens, inherencia/biometría).
3. **Fijación de Sesión (Session Fixation):** Conservación del identificador de sesión pre-autenticación asignado a un cliente anónimo después de una autenticación exitosa.
4. **Invalidación Deficiente de Tokens/Sesiones:** Revocación únicamente en el cliente (UI) manteniendo la validez del token/cookie en el lado del servidor o en tokens auto-contenidos (JWT sin blacklist/mecanismo de revocación activa).
5. **Enumeración de Entidades:** Exposición de mensajes de error o comportamientos diferenciados (longitud de respuesta, tiempos de procesamiento) que revelan la existencia de cuentas en la base de datos.
6. **Inadecuada Validación de Contexto/Auditoría en Tokens:** Aceptación de tokens de acceso o identidad emitidos para otros contextos, audiencias (`aud`) o aplicaciones cliente.

---

## 2. Principios, Implementación o Metodología

### Vector Práctico: Enumeración de Usuarios vía Discrepancias de Respuesta
Cuando un punto de autenticación devuelve mensajes diferenciados según la validez del identificador de usuario, la aplicación permite mapear cuentas válidas en el sistema.

#### Análisis del Vector (Caso Burp Suite Intruder)

1. **Captura del Endpoint de Login:**
   ```http
   POST /login HTTP/1.1
   Host: target-app.com
   Content-Type: application/x-www-form-urlencoded

   username=test&password=test
   ```

2. **Respuestas Diferenciadas:**
   * **Usuario inválido:**
     ```http
     HTTP/1.1 200 OK
     Content-Length: 3248

     {"status": "error", "message": "Invalid username"}
     ```
   * **Usuario válido (contraseña incorrecta):**
     ```http
     HTTP/1.1 200 OK
     Content-Length: 3250

     {"status": "error", "message": "Invalid password"}
     ```

3. **Metodología de Explotación Automatizada:**
   * **Herramienta:** Burp Suite (Módulo Intruder).
   * **Ataque:** Sniper / Simple List iterando la variable `username` manteniendo fija la contraseña.
   * **Criterio de Detección:** Filtrado/Ordenamiento por la métrica de longitud de respuesta (*Length*) o discrepancias de respuesta HTTP para aislar desviaciones del patrón estandarizado (ej. variación de $N$ bytes por cambio del mensaje `Invalid username` a `Invalid password`).

### Mitigación Técnica e Implementación Segura

#### A. Respuesta Uniforme y Genérica
Independientemente de si el usuario existe o la contraseña es errónea, el servidor debe responder de forma estandarizada.

```json
// BAD: Permite enumeración
{ "error": "User does not exist" }
{ "error": "Incorrect password" }

// GOOD: Mensaje estandarizado
{ "status": "fail", "message": "Invalid credentials" }
```

#### B. Protección Integral del Mantenimiento de Sesión
1. **Regeneración tras Autenticación:** Asignar un identificador de sesión completamente nuevo tras cualquier cambio en el estado de autenticación.
2. **Atributos de Galletas de Sesión (Cookies):**
   * `Secure`: Transmisión exclusiva mediante TLS/HTTPS.
   * `HttpOnly`: Impide acceso vía JavaScript (`document.cookie`), mitigando robos por XSS.
   * `SameSite=Strict` o `Lax`: Protección contra CSRF (Cross-Site Request Forgery).

#### C. Validación Estricta de JSON Web Tokens (JWT)
Al utilizar JWTs para sesiones sin estado, la verificación debe incluir:
* **Firma Criptográfica** (`alg` y clave pública/secreta).
* **Emisor (`iss`):** Mapeo del emisor autorizado.
* **Audiencia (`aud`):** Coincidencia estricta con el ID/URL del servicio receptor.
* **Expiración (`exp`) y No antes de (`nbf`).**

```python
# Ejemplo de validación segura de JWT en backend (Python)
import jwt

def verify_session_token(token: str, expected_audience: str, public_key: str):
    try:
        payload = jwt.decode(
            token,
            public_key,
            algorithms=["RS256"],
            audience=expected_audience,
            issuer="https://auth.company.com",
            options={
                "verify_signature": True,
                "verify_exp": True,
                "verify_iss": True,
                "verify_aud": True,
            }
        )
        return payload
    except jwt.PyJWTError as e:
        raise UnauthorizedException("Invalid token or scope")
```

---

## 3. Buenas Prácticas y Trade-offs

| Aspectos Positivos / Recomendados | Riesgos / Errores Comunes |
| :--- | :--- |
| **Implementar MFA Obligatorio:** Basado en TOTP, FIDO2/WebAuthn o tokens de hardware. | **MFA Opcional o Bypassable:** Flujos alternativos (ej. API/recuperación) sin MFA. |
| **Respuestas Genéricas Estandarizadas:** Mismo mensaje y tiempo de procesamiento en login/registro. | **Diferenciación de Errores:** Errores específicos (Usuario no encontrado vs Contraseña errónea). |
| **Políticas Basadas en NIST SP 800-63B:** Enfocadas en longitud ($\ge 8-12$ caracteres) y verificación contra filtraciones (HaveIBeenPwned/HIBP). | **Complejidad Arbitraria e Inefectiva:** Exigir combinaciones complejas que fuerzan patrones predecibles (`P@ssword1!`). |
| **Regeneración e Invalidación de Sesiones:** Invalidar el identificador previo al autenticar y revocar completamente en backend al hacer logout. | **Fijación de Sesión:** Reutilizar la cookie previa al inicio de sesión o revocar solo en el cliente (UI). |
| **Control de Tasa (Rate Limiting) Inteligente:** Limitación por IP/Fingerprint y backoff progresivo para evitar bloqueo de cuentas legítimas. | **Bloqueos Masivos por Cuenta:** Bloqueo estricto por usuario tras $N$ intentos que deriva en Denegación de Servicio (DoS). |
| **Uso de IdP / Marcos Consolidados:** Delegar autenticación a proveedores probados (OAuth2/OIDC, Keycloak, Auth0). | **Mecanismos Custom "In-House":** Desarrollar lógica de sesión o criptografía desde cero. |

> 💡 **Punto Clave:** La seguridad de la autenticación exige defensa en profundidad: validar la fuerza y unicidad de la credencial, garantizar la invisibilidad del estado del usuario mediante respuestas uniformes, y mantener la integridad absoluta del identificador de sesión durante todo su ciclo de vida.

---

## 4. Cheat Sheet / Referencia Rápida

### Checklist de Auditoría / Hardening de Autenticación

- [ ] **Prevención de Enumeración de Usuarios**
  - [ ] ¿Los mensajes de error en `/login`, `/register` y `/forgot-password` son idénticos?
  - [ ] ¿Los tiempos de respuesta ante usuarios existentes e inexistentes son homogéneos?
- [ ] **Gestión de Contraseñas (Estándar NIST SP 800-63B)**
  - [ ] Comprobación obligatoria contra bases de datos de contraseñas filtradas / vulneradas.
  - [ ] Eliminación de credenciales por defecto (*default credentials*) en entornos de producción/admin.
  - [ ] Sin rotación periódica forzada salvo ante sospecha confirmada de compromiso.
- [ ] **Seguridad de Sesiones y Cookies**
  - [ ] Regeneración obligatoria de Session ID inmediatamente después del inicio de sesión.
  - [ ] Cookies configuradas con flags: `Secure`, `HttpOnly`, `SameSite=Strict` / `Lax`.
  - [ ] Prohibido el transporte de tokens/Session IDs en parámetros URL.
  - [ ] Invalidación server-side efectiva en evento de logout y por timeout de inactividad.
- [ ] **Validación de Tokens (JWT/OAuth2)**
  - [ ] Verificación explícita de campos `iss` (Issuer) y `aud` (Audience).
  - [ ] Restricción de ámbito (*scopes*) para impedir reutilización en aplicaciones no autorizadas.
  - [ ] Mecanismo de revocación/blacklist para tokens activos ante logout.
- [ ] **Protección Contra Automatización y DoS**
  - [ ] Rate-limiting por IP y cliente con retardo progresivo (*throttling*).
  - [ ] Registro/Auditoría de eventos de fallos de autenticación con alertas por anomalías.
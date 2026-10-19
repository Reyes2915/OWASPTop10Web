# OWASP Top 10: Fallos Criptográficos (Cryptographic Failures)

**Resumen Ejecutivo:** Análisis técnico de las vulnerabilidades asociadas a fallos en la implementación y gestión criptográfica según OWASP Top 10 (A02:2021 / A04:2025). Incluye la demostración práctica de explotación de firma nula/no verificada en JWT y la guía completa de controles defensivos para entornos de producción.

---

## 📋 Tabla de Contenidos
1. [Conceptos Fundamentales](#1-conceptos-fundamentales)
2. [Principios, Implementación o Metodología](#2-principios-implementación-o-metodología)
3. [Buenas Prácticas y Trade-offs](#3-buenas-prácticas-y-trade-offs)
4. [Cheat Sheet / Referencia Rápida](#4-cheat-sheet--referencia-rápida)

---

## 1. Conceptos Fundamentales

Los fallos criptográficos suceden cuando la información sensible no recibe la protección requerida o cuando la implementación de los mecanismos de seguridad es inadecuada, permitiendo la exposición o manipulación de datos en tránsito o en reposo.

### Primitivas Criptográficas

1. **Cifrado (Encryption):** Transformación bidireccional mediante algoritmos matemáticos y claves para garantizar la confidencialidad.
   - **Cifrado en tránsito:** Protección de datos mientras circulan por la red.
   - **Cifrado en reposo:** Protección de datos almacenados en bases de datos, discos o repositorios.
2. **Funciones Hash (Hashing):** Funciones unidireccionales deterministas que generan un mapa de tamaño fijo (*digest*) a partir de una entrada. Se utilizan para verificar la integridad (cambios en los datos alteran drásticamente el valor hash).
3. **Firmas Digitales (Digital Signatures):** Combinación de cifrado asimétrico y hashing para autenticar el origen/no repudio e integridad de la información emitida por una entidad.

---

## 2. Principios, Implementación o Metodología

### Vector de Vulnerabilidad: Mala Gestión y Algoritmos Débiles

Las fallas más comunes en las aplicaciones web devienen de:
- Almacenamiento y transmisión de datos sensibles en texto plano.
- Elección de algoritmos obsoletos o vulnerables (ej., MD5, SHA-1, DES) o modos de operación no seguros (ej., AES-ECB / PKCS#1 v1.5 padding).
- Gestión inadecuada de claves: claves quemadas en código (*hardcoded*), sin rotación, reutilizadas o con poca entropía.
- Vectores de Inicialización (IV) repetidos o predecibles y falta de aleatoriedad en PRNGs (*Pseudorandom Number Generators*).

---

### Caso Práctico: Bypassing de Firma en JSON Web Tokens (JWT)

#### Contexto de la Vulnerabilidad
Un servicio valida peticiones basándose en la información dentro de un token JWT. Sin embargo, el backend procesa el payload sin validar o verificar la firma criptográfica recibida en el token.

#### Anatomía del Token JWT
Un JWT consta de tres partes codificadas en Base64URL separadas por puntos (`.`):
```text
[Header].[Payload].[Signature]
```
- **Header:** Algoritmo y tipo de token (`{"alg": "HS256", "typ": "JWT"}`).
- **Payload:** Claims de la sesión (`{"sub": "1234567890", "name": "winner"}`).
- **Signature:** Hash cifrado generado a partir de `HMACSHA256(base64UrlEncode(Header) + "." + base64UrlEncode(Payload), secret)`.

#### Paso a Paso del Ataque (Demostración de Explotación)

1. **Autenticación e Intercepción:**
   - Iniciar sesión con credenciales de bajo privilegio (`winner:peter`).
   - Capturar el JWT emitido en la cookie o cabecera `Authorization` de la petición HTTP mediante Burp Suite.

2. **Análisis de Payload:**
   - Decodificar las secciones mediante herramientas o en jwt.io.
   - Se observa el payload con el rol/usuario de sesión:
     ```json
     {
       "iss": "portswigger",
       "sub": "winner"
     }
     ```

3. **Manipulación del Claim de Identidad (Tampering):**
   - Enviar la petición al módulo *Repeater* de Burp Suite (`Ctrl + R`).
   - Utilizar el panel *Inspector* para modificar el valor del objeto `sub` / `username` de `winner` a `administrator`.
   - Aplicar cambios re-codificando en Base64URL la sección modificada del payload.

4. **Explotación:**
   - Modificar el endpoint de la petición enviada a la ruta administrativa restringida: `GET /admin`
   - Enviar la petición (*Send*).
   - **Resultado:** El servidor procesa la petición como `administrator` debido a la falta de verificación del bloque de firma criptográfica en el backend, otorgando control total (posibilidad de eliminar cuentas como `carlos`).

> ⚠️ **Advertencia:** Modificar la cabecera o el payload altera la firma esperada. Si el servidor no valida la firma contra la clave secreta en cada request, cualquier token alterado será aceptado ciegamente.

---

## 3. Buenas Prácticas y Trade-offs

| Aspectos Positivos / Recomendados | Riesgos / Errores Comunes |
| :--- | :--- |
| **Clasificación estricta de datos:** Definir niveles de protección según requerimientos legales (GDPR, PCI-DSS) y del negocio. | Tratar todos los datos con la misma política o ignorar la retención/descarte de datos obsoletos. |
| **Cifrado en tránsito y reposo:** Implementar obligatoriamente TLS 1.2+ y cifrado de bases de datos con algoritmos robustos (ej., AES-256-GCM). | Permitir protocolos heredados no cifrados (FTP, HTTP) o almacenar credenciales con algoritmos reversibles. |
| **Hashes adaptativos para contraseñas:** Usar Argon2id, bcrypt o PBKDF2 con Salt único y factores de trabajo (*work factor*) ajustados. | Usar hashes rápidos de propósito general (MD5, SHA-256 sin salting) para hashes de contraseñas. |
| **Gestión centralizada de claves:** Usar HSMs o Key Vaults de proveedores cloud para custodiar la generación, almacenamiento y rotación de claves. | Hardcodear claves en repositorios de código, usar claves por defecto o no rotarlas periódicamente. |
| **Generación de aleatoriedad criptográfica:** Emplear CSAPRNG (*Cryptographically Secure Pseudo-Random Number Generators*) y vectores de inicialización (IV) únicos. | Definir IVs constantes, semillas predecibles o escribir algoritmos/librerías criptográficas propias. |
| **Protección HTTP/Headers:** Configurar la política HSTS (*HTTP Strict Transport Security*) y deshabilitar caché (*No-Store*) para respuestas sensibles. | Permitir el almacenamiento en caché de datos sensibles en CDNs o proxies intermedios. |

> 💡 **Punto Clave:** Nunca implementes tu propia criptografía (*"Don't roll your own crypto"*). Utiliza bibliotecas maduras, estándares internacionales probados y APIs del lenguaje optimizadas para entornos criptográficamente seguros.

---

## 4. Cheat Sheet / Referencia Rápida

### Checklist de Seguridad Criptográfica

- [ ] **Clasificación:** ¿Se identifican y categorizan todos los datos procesados, almacenados y transmitidos?
- [ ] **Almacenamiento de Contraseñas:** Utilizar Argon2id o bcrypt con Salt único por usuario.
- [ ] **Tránsito:** Deshabilitar versiones de TLS inferiores a 1.2. Implementar cabeceras HSTS:
  ```http
  Strict-Transport-Security: max-age=31536000; includeSubDomains
  ```
- [ ] **Control de Caché:** Evitar almacenar datos sensibles en CDNs y navegadores:
  ```http
  Cache-Control: no-store, private
  ```
- [ ] **Tokens JWT:**
  - Verificar obligatoriamente la firma en el lado del servidor antes de confiar en el payload.
  - Rechazar explícitamente tokens firmados con el algoritmo `"alg": "none"`.
- [ ] **Derivación de Claves:** Usar funciones KDF (*Key Derivation Function*) cuando la clave provenga de una contraseña/input de usuario.
- [ ] **Sistemas Críticos:** Planificar la agilidad criptográfica para la transición a Criptografía Post-Cuántica (PQC) objetivo pre-2030.
# OWASP A08: Fallos de Integridad del Software y de los Datos

> **Resumen Ejecutivo:** Documentación técnica sobre la categoría A08 del OWASP Top 10, enfocada en la identificación de vulnerabilidades por falta de verificación de integridad en datos y software. Se incluye un análisis de explotación práctica en cookies de sesión serializadas y la metodología defensiva para proteger el ciclo de vida del desarrollo y despliegue (CI/CD).

---

## 📋 Tabla de Contenidos
- [1. Conceptos Fundamentales](#1-conceptos-fundamentales)
- [2. Principios, Implementación o Metodología](#2-principios-implementación-o-metodología)
- [3. Buenas Prácticas y Trade-offs](#3-buenas-prácticas-y-trade-offs)
- [4. Cheat Sheet / Referencia Rápida](#4-cheat-sheet--referencia-rápida)

---

## 1. Conceptos Fundamentales

Los **fallos de integridad del software y de los datos** ocurren cuando una aplicación confía en código, objetos o datos de origen externo o interno sin antes validar su autenticidad e inalterabilidad. 

### Vectores de Riesgo Clave
1. **Datos del lado del cliente (Client-side Data):** Objetos serializados o cookies de sesión enviadas al cliente que no cuentan con firma criptográfica (ej. HMAC) o cifrado de integridad, lo que permite al usuario modificar atributos de control de acceso.
2. **Cadena de Suministro (Supply Chain):** Incorporación de paquetes y dependencias (NPM, Maven, etc.) provenientes de repositorios no verificados o sin comprobación de firmas/hashes.
3. **Pipeline de CI/CD e Infraestructura:** Ausencia de segregación de funciones (SoD) y falta de revisiones de código obligatorias, permitiendo que un único desarrollador o un atacante modifique el comportamiento del software antes o durante su despliegue en producción.

---

## 2. Principios, Implementación o Metodología

### A. Escenario Práctico: Escalada de Privilegios mediante Manipulación de Cookies Serializadas

En un entorno con vulnerabilidad de integridad en datos serializados:

1. **Autenticación con credenciales de bajo privilegio:**
   - Credenciales: `winner:peter`
   - La aplicación responde con una cookie de sesión serializada.

2. **Inspección de la Cookie:**
   - Mediante una herramienta de proxy (ej. Burp Suite Inspector), se analiza el objeto serializado subyacente:
     ```json
     {
       "username": "winner",
       "admin": 0
     }
     ```

3. **Explotación (Tampering):**
   - Se intercepta la solicitud HTTP y se reenvía al editor/repeater.
   - Se modifica la propiedad del objeto serializado: cambiar el flag `"admin": 0` por `"admin": 1`.
   - Se reenvía la petición modificada.

4. **Impacto:**
   - La aplicación desconfía pero procesa el objeto sin validar si fue alterado en el cliente, otorgando privilegios administrativos de forma inmediata.

---

### B. Estrategia Defensiva y Arquitectura de Mitigación

```
[ Fuente de Datos/Software ] 
           │
           ▼
[ Verificación de Integridad ] ──( Firma Válida? )──► [ Sí ] ──► [ Ejecutar / Procesar ]
           │                                            │
          [ No ]                                       [ No ]
           │                                            │
           ▼                                            ▼
[ Descartar / Bloquear ] ◄──────────────────────────────┘
```

1. **Firma y Cifrado de Objetos Serializados:**
   - No aceptar objetos serializados del cliente sin una firma criptográfica que garantice su inalterabilidad.
   - Usar JWT firmados (HS256/RS256) o tokens firmados con HMAC.

2. **Seguridad en la Cadena de Suministro:**
   - Verificar la firma digital de paquetes y actualizaciones antes de su instalación.
   - Utilizar un gestor de repositorios de código interno (Nexus, Artifactory) pre-aprobado y escaneado para reducir el riesgo de dependencias maliciosas.

3. **Protección del Pipeline de CI/CD:**
   - Implementar el principio de menor privilegio en entornos de build/deploy.
   - Requerir aprobación por pares (Pull Request Review) antes de integrar cambios al branch principal o activar despliegues automáticos.

---

## 3. Buenas Prácticas y Trade-offs

| Aspectos Positivos / Recomendados | Riesgos / Errores Comunes |
| :--- | :--- |
| **Firma Criptográfica:** Uso de HMAC o firmas digitales para verificar el origen e integridad de datos serializados. | **Confianza Ciega:** Procesar estados o roles provenientes del cliente sin validación en el backend. |
| **Repositorios Internos Curados:** Control centralizado de dependencias de terceros (NPM, Maven). | **Uso de Fuentes No Verificadas:** Descargar librerías directamente de la red sin validar *checksums* o firmas. |
| **Segregación de Funciones (SoD):** Separar permisos de desarrollo de los permisos de despliegue a producción. | **Acceso Total Monolítico:** Permitir que un solo usuario modifique el código fuente y despliegue a producción sin supervisión. |
| **Revisión Obligatoria de Código:** Proceso de PR/MR formal antes de mergear a ramas de producción. | **Bypass de Code Review:** Realizar *commits* directos a la rama principal en entornos productivos. |

> 💡 **Punto Clave:** Todo dato u objeto recibido de un cliente o sistema externo debe considerarse no confiable. Se debe validar de forma explícita su origen mediante firma criptográfica antes de procesar su contenido en la aplicación.

---

## 4. Cheat Sheet / Referencia Rápida

### Checklist de Seguridad de Integridad

- [ ] **Datos Serializados:** ¿Los datos en cookies/tokens están firmados criptográficamente con clave secreta del servidor?
- [ ] **Validación de Roles:** ¿La autorización de acceso se calcula en el servidor en lugar de confiar en *flags* del cliente (`admin=1`)?
- [ ] **Gestión de Dependencias:** ¿Se emplean repositorios privados/locales con paquetes aprobados previamente?
- [ ] **Verificación de Firma de Software:** ¿Se comprueba el *hash* o la firma digital antes de ejecutar scripts o actualizaciones?
- [ ] **Seguridad en CI/CD:** ¿Está deshabilitado el acceso directo a producción para los desarrolladores sin aprobación por pares?
- [ ] **Segregación de Permisos:** ¿El pipeline de producción requiere autenticación multifactor y roles diferenciados para su modificación?
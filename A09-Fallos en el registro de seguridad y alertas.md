# OWASP Top 10: Fallos en el Registro de Seguridad y Alertas (*Security Logging and Monitoring Failures*)

**Resumen Ejecutivo:** Análisis técnico sobre las vulnerabilidades derivadas del monitoreo insuficiente y el registro deficiente de eventos de seguridad. Explora cómo los flujos secundarios (como el cambio de contraseña) sufren de falta de telemetría y detalla las directrices esenciales de OWASP para implementar auditorías robustas, prevención de manipulación de logs y detección activa con *Honey Tokens*.

---

## 📋 Tabla de Contenidos
1. [Conceptos Fundamentales](#1-conceptos-fundamentales)
2. [Principios, Implementación o Metodología](#2-principios-implementaci%C3%B3n-o-metodolog%C3%ADa)
3. [Buenas Prácticas y Trade-offs](#3-buenas-pr%C3%A1cticas-y-trade-offs)
4. [Cheat Sheet / Referencia Rápida](#4-cheat-sheet--referencia-r%C3%A1pida)

---

## 1. Conceptos Fundamentales

Los fallos en los registros de seguridad y alertas ocurren cuando una aplicación no audita, almacena ni monitorea eventos críticos de seguridad de manera adecuada. Esto impide la visibilidad de las actividades maliciosas y retrasa la detección de intrusiones en tiempo real.

### El Problema de la Cobertura Asimétrica
* **Procesos Críticos vs. Flujos Secundarios:** Las organizaciones suelen centrar la auditoría, limitación de tasa (*rate limiting*) y alertas en puntos de entrada evidentes (p. ej., `/login`). Sin embargo, descuidan flujos menos frecuentes como la recuperación de credenciales o la actualización de contraseña (`/change-password`).
* **Vulnerabilidad al Ataque Automatizado:** Al no registrar fallos reiterados en endpoints secundarios, los atacantes pueden realizar ataques de fuerza bruta o de relleno de credenciales (*credential stuffing*) sin ser detectados ni bloqueados por el sistema.

---

## 2. Principios, Implementación o Metodología

### Caso Práctico: Fuerza Bruta mediante Canales Secundarios

Un escenario común de esta vulnerabilidad se demuestra al evadir los controles de seguridad en un portal web:

1. **Monitoreo en Login (`/login`):** La aplicación implementa bloqueo temporal e incrementa contadores al detectar múltiples intentos fallidos.
2. **Explotación en Cambio de Contraseña (`/change-password`):**
   * El atacante envía solicitudes `POST` parametrizadas enviando la credencial actual candidate, una nueva contraseña y su confirmación.
   * **Vector de Inferencia por Oráculo de Respuesta:**
     * Password actual incorrecto $\rightarrow$ Respuesta A (Longitud $X$).
     * Las nuevas contraseñas no coinciden $\rightarrow$ Respuesta B (Longitud $Y$).
   * Al no existir limitación de tasa ni registro de eventos en esta función, se pueden automatizar miles de intentos mediante herramientas como *Burp Suite Intruder*, difiriendo la respuesta correcta según la variación de la longitud de la respuesta HTTP ($Y$).

### Arquitectura Defensiva y Recomendaciones OWASP

```text
[ Evento de Aplicación ] 
       │
       ├── Log Estructurado (JSON) ──> [ Sanitización / Escape ]
       │                                         │
       │                                         ▼
       └──> [ BD / Storage Append-Only ] ──> [ SIEM / Wazuh / ELK Stack ]
                                                 │
                                                 ▼
                                        [ Casos de Uso / Playbooks ]
                                                 │
                                                 ▼
                                     [ Alertas al Blue Team / SOC ]
```

#### 1. Formato y Estructura de Registros
* **Eventos Estructurados:** Emplear formatos legibles por máquinas (p. ej., JSON) para facilitar la ingesta, indexación y correlación en plataformas SIEM (*Elasticsearch, Logstash, Kibana, Wazuh*).
* **Auditoría Simétrica:** Registrar tanto las operaciones fallidas como las exitosas en flujos sensibles. Si solo se auditan los fallos, se pierde la visibilidad cuando un ataque de fuerza bruta finalmente tiene éxito.
* **Contexto de Seguridad:** Cada entrada debe incluir marca temporal estandarizada (UTC), IP de origen, ID de usuario, tipo de evento y estado de ejecución.

#### 2. Sanitización y Protección de Logs
* **Escape de Caracteres (*Log Injection*):** Los datos introducidos por los usuarios deben codificarse antes de guardarse en los registros para evitar ataques como *Log Forge* o inyección de código interpretable en las herramientas de monitoreo/SIEM.
* **Inmutabilidad (*Append-Only*):** Las tablas o repositorios de auditoría deben diseñarse bajo un patrón de solo adición (*append-only*). No se deben sobrescribir ni borrar registros existentes durante reversiones o actualizaciones.
* **Manejo de Transacciones y Rollbacks:** En caso de fallos de lógica de negocio, se debe realizar un *rollback* de los datos de la aplicación, pero la trazabilidad del evento fallido debe persistir en la capa de auditoría.

#### 3. Detección Activa con Honey Tokens
Los *Honey Tokens* son recursos o datos señuelo (cuentas de usuario inexistentes, tablas ficticias, llaves API sintéticas) integrados en la aplicación:
* **Baja Tasa de Falsos Positivos:** Dado que ningún usuario legítimo debería interactuar con estos elementos, cualquier acceso o consulta activa una alerta inmediata de alta prioridad para el equipo de seguridad (*Blue Team*).

#### 4. Operación del SOC: Playbooks y Casos de Uso
* **Caso de Uso:** Define la condición lógica o regla de correlación a detectar (ejemplo: > 50 intentos de cambio de contraseña en menos de 1 minuto).
* **Playbook:** Guía procedimental detallada que prescribe las acciones exactas que debe ejecutar el equipo de SOC/Blue Team al recibir la alerta (aislamiento de IP, revocación de sesión, escalado).

---

## 3. Buenas Prácticas y Trade-offs

| Aspectos Positivos / Recomendados | Riesgos / Errores Comunes |
| :--- | :--- |
| **Auditoría Integral:** Cobertura de eventos en flujos primarios (login) y secundarios (cambio de clave, MFA). | **Puntos Ciegos:** Auditar únicamente el inicio de sesión dejando sin telemetría otras funciones críticas. |
| **Registros Inmutables (*Append-Only*):** Persistencia de auditoría independiente de las reversiones transaccionales. | **Sobresuscripción o Borrado:** Permitir actualización o eliminación de registros históricos en caso de error. |
| **Sanitización de Salida hacia Logs:** Codificar/escapar entradas para evitar inyección contra el SIEM. | **Log Injection:** Pasar directamente variables no confiables a los archivos de registro. |
| **Monitoreo Éxito/Fallo:** Trazabilidad completa de autenticaciones aceptadas y rechazadas. | **Falta de Contexto Post-Explotación:** Registrar solo errores e ignorar el evento de acceso concedido tras una fuerza bruta. |
| **Definición de Playbooks:** Guías estandarizadas de respuesta para incidentes. | **Alert Fatigue / Improvisación:** Alertas no correlacionadas o falta de procedimiento ante eventos anómalos. |

> 💡 **Punto Clave:** Ninguna solución de monitoreo o herramienta de IA es mágica. Su efectividad depende directamente de la calidad del diseño de la telemetría en la aplicación, el uso de formatos estructurados y la definición clara de casos de uso acompañados de playbooks de respuesta.

---

## 4. Cheat Sheet / Referencia Rápida

### Lista de Comprobación para Implementación de Telemetría

- [ ] **Cobertura Global:** Monitorear y registrar autenticación, control de acceso, cambios de contraseña y modificadores de perfil.
- [ ] **Telemetría Dual:** Registrar eventos `SUCCESS` y `FAILURE` con el contexto adecuado (IP, User-Agent, Timestamp UTC, User ID).
- [ ] **Estructuración:** Generar logs en formato estructurado (JSON) optimizado para correlación mediante SIEM (*Wazuh, ELK Stack*).
- [ ] **Sanitización de Logs:** Escapar caracteres especiales para prevenir inyección de registros en herramientas de análisis.
- [ ] **Persistencia Inmutable:** Configurar repositorios *append-only* donde las operaciones no modifiquen históricos.
- [ ] **Controles Activos:** Implementar *Honey Tokens* (cuentas señuelo) en la base de datos para detección temprana.
- [ ] **Respuesta Definida:** Vincular las alertas automáticas con un Playbook de respuesta a incidentes (ejemplo: basándose en estándares NIST).
# Sistema de Avisos Automáticos por WhatsApp para Gimnasios

Sistema de notificaciones automáticas de vencimiento de cuotas para un gimnasio, construido con n8n, la API oficial de WhatsApp Business (Meta) y Chatwoot, desplegado íntegramente en un VPS propio.

> **Nota de contexto:** este proyecto se desarrolló como propuesta real para un cliente (un gimnasio local). El sistema quedó completo y operativo en fase de pruebas; el acuerdo comercial finalmente no se cerró, pero el proyecto se documenta aquí como ejercicio de arquitectura e integración de sistemas.

---

## El problema

Un gimnasio pequeño gestiona el cobro de cuotas manualmente: revisar quién debe pagar, avisar por WhatsApp uno a uno, y hacer seguimiento de quién ha respondido. Es un proceso repetitivo, propenso a olvidos, y que no escala con el número de socios.

## La solución

Un flujo de automatización que:
1. Lee diariamente una base de datos de clientes (Google Sheets)
2. Calcula quién está próximo a vencer o ya en retraso
3. Envía un mensaje de WhatsApp personalizado usando plantillas aprobadas por Meta
4. Registra el envío para no repetir avisos
5. Permite al gimnasio responder a los clientes desde una bandeja de entrada centralizada (Chatwoot)

---

## Arquitectura

```
┌─────────────────┐     ┌──────────────┐     ┌─────────────────────┐
│  Google Sheets   │────▶│     n8n      │────▶│  WhatsApp Cloud API  │
│ (base de clientes)│    │ (automatización)│    │      (Meta)          │
└─────────────────┘     └──────────────┘     └──────────┬──────────┘
                                                          │
                                                          ▼
                                              ┌─────────────────────┐
                                              │      Cliente         │
                                              │  recibe el aviso      │
                                              └──────────┬──────────┘
                                                          │ responde
                                                          ▼
                                              ┌─────────────────────┐
                                              │      Chatwoot         │
                                              │ (bandeja de entrada)  │
                                              └──────────┬──────────┘
                                                          │
                                                          ▼
                                              ┌─────────────────────┐
                                              │  Dueño del gimnasio   │
                                              │  contesta desde el web│
                                              │   o app móvil          │
                                              └─────────────────────┘
```

Todo el stack corre en un único VPS (Hetzner CPX22, 4GB RAM) mediante Docker Compose, con Nginx como proxy inverso y certificados SSL de Let's Encrypt.

### Stack técnico

| Capa | Tecnología | Por qué |
|------|-----------|---------|
| Automatización | [n8n](https://n8n.io) self-hosted | No-code, control total de la lógica, sin coste por ejecución |
| Base de datos | Google Sheets | El cliente ya gestionaba los socios ahí; cero curva de aprendizaje |
| Mensajería | WhatsApp Cloud API (Meta) | API oficial, sin riesgo de baneo, sin coste de intermediario (BSP) |
| Bandeja de entrada | [Chatwoot](https://chatwoot.com) self-hosted | Alternativa gratuita a herramientas de pago (Respond.io, WATI) que cobran 80-150€/mes |
| Infraestructura | Docker + Docker Compose | Aislamiento de servicios, despliegue reproducible |
| Servidor | Hetzner VPS (Ubuntu 24.04) | Coste-rendimiento, control total sobre los datos |
| Proxy / SSL | Nginx + Certbot | HTTPS obligatorio para webhooks de Meta |
| DNS | DuckDNS | Dominio gratuito para entorno de pruebas |

---

## Decisiones de arquitectura

### ¿Por qué API oficial de Meta y no una librería no oficial (Baileys, WPPConnect)?

Las librerías no oficiales automatizan WhatsApp simulando un cliente web, lo cual viola los términos de servicio y puede resultar en el baneo del número sin previo aviso — un riesgo inaceptable para un negocio que depende de ese canal para cobrar. La API oficial es más rígida (requiere plantillas pre-aprobadas para iniciar conversaciones) pero es estable y sostenible a largo plazo.

### ¿Por qué Chatwoot self-hosted en lugar de una herramienta SaaS?

Se evaluaron Respond.io y WATI como bandejas de entrada visuales. Ambas tienen planes desde 80-150€/mes para el volumen de uso necesario — inviable para un gimnasio pequeño. Chatwoot Community Edition es open source (licencia MIT) y, autoalojado en el mismo VPS que ya se necesita para n8n, no añade coste recurrente.

### ¿Por qué un número de teléfono dedicado para la API?

Al vincular un número a la API de WhatsApp, ese número deja de funcionar en la app de WhatsApp normal o Business. La solución fue usar un número nuevo (SIM dedicada, coste mínimo) exclusivamente para la API, dejando intacto el WhatsApp Business habitual del negocio. Chatwoot permite responder igualmente desde ese mismo número dedicado, así que en la práctica el cliente final solo ve un único chat de WhatsApp.

### ¿Quién paga qué?

Se separó deliberadamente la facturación técnica de la facturación de servicio: el cliente final paga directamente a Meta (coste por mensaje) y a Hetzner (VPS) con su propia tarjeta, mientras que el desarrollador factura aparte por configuración y mantenimiento. Esto evita que impagos del cliente final se conviertan en una deuda técnica del desarrollador con terceros.

---

## Lógica del flujo (n8n)

1. **Schedule** — ejecución diaria (zona horaria configurada explícitamente; por defecto los contenedores Docker corren en UTC)
2. **Google Sheets (Get Many Rows)** — lee la hoja de clientes vía OAuth2
3. **Code (JavaScript)** — sustituye al nodo Filter estándar para controlar manualmente la conversión de tipos (evita errores con celdas vacías) y aplica dos condiciones independientes:
   - Aviso previo: 0-2 días antes del vencimiento, si no se ha enviado ya
   - Aviso de retraso: 1-5 días después del vencimiento, si no se ha enviado ya
4. **IF** — separa ambos casos en dos ramas
5. **HTTP Request** (por rama) — llama a la Graph API de Meta con la plantilla correspondiente, interpolando nombre y días desde los datos de la fila
6. **Google Sheets (Update Row)** — marca el envío en la columna de control correspondiente, preservando el valor de la otra columna de control (un primer intento sobrescribía toda la fila y borraba el histórico del otro aviso)

### Plantillas de mensaje

Meta requiere que cualquier mensaje que inicie una conversación use una plantilla pre-aprobada. Se diseñaron dos:

- **Aviso previo:** recuerda el vencimiento próximo, tono cordial
- **Aviso de retraso:** informa de un pago pendiente, sin tono acusatorio, invitando al contacto

Ambas pasaron el proceso de revisión de Meta (categoría *Utilidad*).

---

## Problemas encontrados y cómo se resolvieron

Esta sección documenta los obstáculos reales del despliegue, no solo el resultado final.

| Problema | Causa | Solución |
|----------|-------|----------|
| El nodo Code fallaba con `NaN` | Celdas vacías en Google Sheets no convertían a número de forma segura | Validación explícita con `isNaN()` antes de filtrar |
| El nombre de columna no coincidía en n8n | La hoja tenía una fila de título por encima de las cabeceras reales, desplazando la lectura | Se eliminó la fila de título (no afecta a las fórmulas, que usan referencias de celda, no nombres de columna) |
| El segundo aviso borraba el registro del primero | El nodo de actualización de Google Sheets sobrescribía todos los campos no especificados | Se referenció explícitamente el valor original de la otra columna de control en la expresión de actualización |
| Error 401 intermitente en la API de Meta | El token de acceso temporal de Meta caduca a las 24h | Generación de un token permanente mediante un *usuario del sistema* en Meta Business (caducidad: nunca) |
| Redirect URI rechazada por Google OAuth | Google no admite URIs de callback sobre IP directa, solo dominios | Se resolvió con un subdominio gratuito (DuckDNS) + certificado SSL, en lugar de comprar un dominio para el entorno de pruebas |
| Chatwoot en bucle de reinicio (varias causas sucesivas) | Nombre de base de datos incorrecto (`POSTGRES_DB` vs `POSTGRES_DATABASE`), imagen de Postgres sin la extensión `pgvector` requerida por versiones recientes de Chatwoot, y el comando de arranque saltaba la inicialización de la base de datos | Se separó el servicio en `chatwoot_web` + `chatwoot_worker` (Sidekiq), se cambió a la imagen `pgvector/pgvector:pg15`, y se inicializó la base de datos explícitamente con `rails db:chatwoot_prepare` antes del primer arranque |
| Build de Vite fallaba en el instalador oficial de Chatwoot | Incompatibilidad de versión de Node.js en el script de instalación automática | Se abandonó el instalador automático en favor de una configuración manual vía Docker Compose, con más control sobre cada variable de entorno |

---

## Costes de producción (estimados)

| Concepto | Coste mensual |
|----------|--------------|
| VPS (Hetzner CPX22, 4GB RAM) | ~10 € |
| Mensajes de WhatsApp (API Meta, categoría Utilidad) | ~5-8 € (depende del volumen) |
| Número SIM dedicado para la API | ~1 € |
| n8n self-hosted | 0 € |
| Chatwoot self-hosted | 0 € |
| **Total estimado** | **~16-19 €/mes** |

Mensajes entrantes (respuestas de clientes) son gratuitos según la política actual de Meta. Los costes de infraestructura y API quedan a cargo del cliente final, facturados directamente por cada proveedor.

---

## Qué añadiría en una siguiente iteración

- **Monitorización**: alerta automática (UptimeRobot o similar) si el VPS o el flujo de n8n dejan de responder
- **Workflow de errores en n8n**: notificación al desarrollador si el flujo falla silenciosamente (p. ej. Google Sheets inaccesible)
- **Backups automáticos** del VPS, no solo de los datos de Google Sheets
- **Tercera rama de morosidad**: clientes con más de 5 días de retraso, con un tono de mensaje distinto y marcado automático de estado
- **Migración de Google Sheets a una base de datos real** (PostgreSQL/Airtable) si el volumen de socios crece significativamente
- **Dominio propio** en lugar de DuckDNS para el entorno de producción final

---

## Stack resumido

`n8n` · `Google Sheets API` · `WhatsApp Cloud API (Meta)` · `Chatwoot` · `Docker` · `Docker Compose` · `Nginx` · `Let's Encrypt` · `PostgreSQL (pgvector)` · `Redis` · `JavaScript` · `Ubuntu 24.04`

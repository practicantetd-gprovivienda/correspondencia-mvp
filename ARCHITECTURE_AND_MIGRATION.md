# Correspondencia MVP · Arquitectura, despliegue y migración (Full-Stack + Drive)

> **Sistema de Trazabilidad y Gestión de Correspondencia — Grupo Provivienda (Panamá)**
> Guía técnica portable para **personas** y para **agentes de IA** que deban mantener, extender, exportar o desplegar el sistema desde una cuenta nueva.
> Fuente de verdad: `Code.gs` (backend) e `index.html` (frontend). Si este documento y el código difieren, manda el código.
> Por seguridad, los IDs reales (carpeta, URL `/exec`, ID de cliente OAuth) aparecen aquí como marcadores `<...>`; los valores vigentes están en los propios archivos.

---

## Índice

0. [Ficha rápida (legible por máquina)](#0-ficha-rápida-legible-por-máquina)
1. [Arquitectura](#1-arquitectura)
2. [Archivos y activos](#2-archivos-y-activos)
3. [Esquema de Google Sheets](#3-esquema-de-google-sheets)
4. [Carpeta de Evidencias en Google Drive (paso a paso)](#4-carpeta-de-evidencias-en-google-drive-paso-a-paso)
5. [Backend · `Code.gs`](#5-backend--codegs)
6. [Contrato de la API](#6-contrato-de-la-api)
7. [Reglas de negocio](#7-reglas-de-negocio)
8. [Frontend · `index.html`](#8-frontend--indexhtml)
9. [Google Cloud · OAuth y Orígenes autorizados](#9-google-cloud--oauth-y-orígenes-autorizados)
10. [GitHub Pages · publicación del frontend](#10-github-pages--publicación-del-frontend)
11. [Despliegue completo desde una cuenta nueva (checklist)](#11-despliegue-completo-desde-una-cuenta-nueva-checklist)
12. [Cómo cambiar la cuenta emisora de correos](#12-cómo-cambiar-la-cuenta-emisora-de-correos)
13. [Importación del histórico](#13-importación-del-histórico-historicocsvcsv)
14. [Seguridad y limitaciones conocidas](#14-seguridad-y-limitaciones-conocidas)
15. [Solución de problemas](#15-solución-de-problemas)
16. [Guía para agentes de IA](#16-guía-para-agentes-de-ia)

---

## 0. Ficha rápida (legible por máquina)

```yaml
proyecto: correspondencia-mvp
costo_operativo: 0 USD (Google Workspace + GitHub Pages)
frontend:
  archivo: index.html                 # archivo único: HTML + Tailwind CDN + JS vanilla (sin build)
  activos_requeridos: [index.html, logo-provivienda.jpg]
  hosting: GitHub Pages               # https://practicantetd-gprovivienda.github.io/correspondencia-mvp/
  origen_oauth: https://practicantetd-gprovivienda.github.io
  auth: Google Identity Services (JWT decodificado en el navegador)
  dominio_permitido: gprovivienda.com # constante ALLOWED_DOMAINS
  constantes_editables: [APPS_SCRIPT_URL, GOOGLE_CLIENT_ID, ADMIN_EMAILS, ALLOWED_DOMAINS]
  localStorage: [corr_session_v1, "corr_recientes_v1:<email>"]
  pestañas: [nuevo, panel (solo ADMIN_EMAILS), rastreo]
  paleta: brand rojo #fef2f2 → #7f1d1d + neutros slate
  camara: getUserMedia(facingMode environment) + canvas (máx. 800 px, JPEG 0.75) + "Subir archivo"
  gps: navigator.geolocation, límite duro 8 s, null si falla
  http: GET sin cabeceras; POST Content-Type text/plain;charset=utf-8
backend:
  archivo: Code.gs                    # Google Apps Script vinculado a la hoja, desplegado como Web App
  endpoints: [doGet, doPost]
  doPost_actions: [createShipment, updateStatus, syncContacts]
  implementacion: { ejecutar_como: "Yo (cuenta propietaria)", acceso: "Cualquier usuario" }
  servicios: [SpreadsheetApp, DriveApp, GmailApp, MailApp (cuota), People API (avanzado), LockService, PropertiesService, ScriptApp, Session]
  propiedades_script: [ULTIMA_REFERENCIA]
  trigger: sincronizarContactos diario 06:00 America/Panama
  remitente: { SENDER_ALIAS: kawak@gprovivienda.com, SENDER_NAME: "notificador Provivienda" }
datos:
  hoja: Google Sheets con pestañas [Envios (A–P, 16 col.), Directorio (A–C)]
  fotos: carpeta Drive "Evidencias_Correspondencia" (FOLDER_ID_FOTOS), compartida "Cualquier persona con el enlace · Lector"
dominio_negocio:
  estados: [Registrado, Recibido, En Camino, Entregado]   # avanzan, nunca retroceden
  tipos: ["Interna (Grupo Provivienda)", "Externa"]
  referencias: consecutivas 10000..99999, se muestran "#10000"
  zona_horaria: America/Panama
```

---

## 1. Arquitectura

```
┌───────────────────────────────────┐   GET  /exec  (sin cabeceras)        ┌───────────────────────────────────┐
│ index.html · GitHub Pages         │ ───────────────────────────────────▶ │ Code.gs · Apps Script Web App     │
│  · Login Google (GIS, @gprovivienda)                                     │  "Ejecutar como: Yo"              │
│  · Nuevo Envío (cámara + canvas)  │   POST /exec  (text/plain + JSON)    │  · doGet  → {directorio, envios}  │
│  · Panel Operador/Conductor (GPS) │ ───────────────────────────────────▶ │  · doPost → createShipment        │
│  · Mis Envíos (línea de tiempo)   │ ◀──── JSON {ok,...}  (CORS *) ────── │             updateStatus          │
└───────────────────────────────────┘                                      │             syncContacts          │
                                                                           └──────┬─────────┬─────────┬────────┘
                                                                                  │         │         │
                                                                   Google Sheets ◀┘         │         └▶ Gmail (GmailApp)
                                                                   Envios · Directorio      ▼            remitente kawak@…
                                                                         ▲          Drive: Evidencias_Correspondencia
                                                 Trigger diario 06:00 ───┘          (fotos, "cualquiera con el enlace")
                                                 People API → Directorio
```

### 1.1 Flujo de un envío

| Paso | Actor | Acción en la UI | Backend | Efecto en Sheets | Correo |
|---|---|---|---|---|---|
| 1 | Cualquier colaborador | Nuevo Envío → Registrar | `createShipment` | Nueva fila A–P, `Estado = Registrado`, código consecutivo en G, foto en L | Al destinatario |
| 2 | Archivo / conductor (`ADMIN_EMAILS`) | 📥 Recibido | `updateStatus` | H = Recibido, I = fecha/hora, P = GPS | — |
| 3 | Conductor | 🚚 En Camino | `updateStatus` | H = En Camino, P = GPS. **Si se saltó Recibido, I = misma fecha/hora** | Al destinatario |
| 4 | Conductor | ✅ Entregado | `updateStatus` | H = Entregado, J = fecha/hora, K = duración (I→J, o A→J si I vacío), P = GPS | Al emisor y al destinatario |

### 1.2 Responsabilidades

- **Frontend**: autenticación visual, pestañas por rol, selección del destinatario desde el Directorio, cámara y compresión de imagen, GPS con límite de tiempo, filtrado de "Mis Envíos", presentación (`#10000`, departamentos normalizados).
- **Backend**: validación, código consecutivo con bloqueo, escritura en Sheets, fotos en Drive, transiciones de estado, duración, normalización de departamentos, correos HTML, sincronización diaria de contactos.

---

## 2. Archivos y activos

| Archivo | Dónde vive | Rol | ¿Publicar en GitHub Pages? |
|---|---|---|---|
| `index.html` | Repositorio GitHub (raíz) | Aplicación web completa | **Sí (obligatorio)** |
| `logo-provivienda.jpg` | Repositorio GitHub (misma carpeta que `index.html`) | Logo en cabecera y login | **Sí (obligatorio)**. Nombre exacto, en minúsculas (GitHub Pages distingue mayúsculas) |
| `Code.gs` | Editor de Apps Script vinculado a la hoja | Backend | Opcional (copia de referencia; no se ejecuta en Pages) |
| `ARCHITECTURE_AND_MIGRATION.md` | Repositorio | Este documento | Opcional |
| `PRD.md.md` | Repositorio | Requisitos originales | Opcional |
| `historico.csv.csv` | Solo local | Exportación histórica (§13) | **No** (contiene datos personales) |
| Hoja de cálculo | Google Drive | Base de datos (`Envios`, `Directorio`) | — |
| Carpeta `Evidencias_Correspondencia` | Google Drive | Fotos de los paquetes (§4) | — |

---

## 3. Esquema de Google Sheets

> Nombres de pestañas y encabezados **exactos** (incluidos acentos). El orden A–O es heredado de reportes históricos; P (`GPS`) va al final. `setup()` crea las pestañas y encabezados si no existen y avisa si alguno no coincide.

### 3.1 Pestaña `Envios` (16 columnas)

| Col | Encabezado exacto | Tipo | Escrito por | Regla |
|---|---|---|---|---|
| A | `Marca temporal` | Fecha | `createShipment` | `new Date()` al registrar |
| B | `Dirección de correo electrónico` | Texto (minúsculas) | `createShipment` | Correo del emisor (`userEmail`) |
| C | `Correo del destinatario` | Texto (minúsculas) | `createShipment` | Obligatorio. En la UI viene siempre de la persona elegida en el Directorio |
| D | `Nombre del destinatario` | Texto ≤150 | `createShipment` | Obligatorio |
| E | `Departamento / Proyecto` | Texto ≤150 | `createShipment` | Departamento normalizado del Directorio (vacío si no tiene); no editable en la UI |
| F | `Descripción de la Correspondencia` | Texto ≤2000 | `createShipment` | Obligatorio |
| G | `Referencia` | **Número** | `createShipment` | Consecutivo 10000–99999, único (§7.1). Se muestra `#10000` |
| H | `Estado` | Texto | ambos | `Registrado` → `Recibido` → `En Camino` → `Entregado` |
| I | `Recibido` | Fecha | `updateStatus` | Al marcar Recibido, o autocompletado al saltar de Registrado a En Camino |
| J | `Entregado` | Fecha | `updateStatus` | Al marcar Entregado |
| K | `Duración` | Texto | `updateStatus` | `J − I` (o `J − A`): `<1m`, `25m`, `1h 25m`, `2d 0h 5m` |
| L | `Sube una foto de tu correspondencia` | URL | `createShipment` | `https://drive.google.com/thumbnail?id=<FILE_ID>&sz=w600` o vacío |
| M | `Tipo de Mensajería` | Texto | `createShipment` | Exactamente `Interna (Grupo Provivienda)` o `Externa` |
| N | `Nombre del proveedor` | Texto | `createShipment` | Solo si M = `Externa` (obligatorio) |
| O | `Correo del proveedor` | Texto | `createShipment` | Solo si M = `Externa` (opcional) |
| P | `GPS` | Texto | `updateStatus` | Último cambio: `lat,lng \| https://www.google.com/maps?q=lat,lng` |

- Textos que empiezan con `= + - @` se guardan con apóstrofo (anti-inyección de fórmulas).
- Al leer, cualquier URL de Drive en L (`open?id=`, `/file/d/`, `uc?id=`, `thumbnail?id=`) se convierte a miniatura `sz=w600`.
- Fechas históricas en texto `dd/MM/yyyy HH:mm[:ss]` se interpretan para calcular la duración.

### 3.2 Pestaña `Directorio`

| Col | Encabezado | Origen | Regla |
|---|---|---|---|
| A | `Nombre` | People API (`names.displayName`) | Obligatorio |
| B | `Correo` | People API (correo principal) | Obligatorio, minúsculas, único |
| C | `Departamento` | People API (`organizations.department` → `name` → `title`) | Normalizado (§7.4); `""` si no existe |

`sincronizarContactos()` borra las filas 2..N y reescribe todo ordenado por nombre. Fuentes: "Mis contactos" de la cuenta ejecutora y, con `INCLUDE_DOMAIN_DIRECTORY = true`, el directorio del dominio Workspace.

---

## 4. Carpeta de Evidencias en Google Drive (paso a paso)

Las fotos se guardan en una carpeta de Drive. Para que las miniaturas carguen en `index.html` (que se sirve desde GitHub Pages, sin sesión de Drive) y en los correos, **la carpeta y sus archivos deben ser legibles por "Cualquier persona con el enlace"**.

### 4.1 Crear la carpeta

1. Iniciar sesión en [drive.google.com](https://drive.google.com) con la **cuenta propietaria** del sistema (la que ejecutará Apps Script, p. ej. `kawak@gprovivienda.com`).
2. **Nuevo → Carpeta nueva** → nombre: `Evidencias_Correspondencia` → **Crear**.
3. *(Opcional)* Moverla junto a la hoja de cálculo para mantener todo el proyecto en un mismo lugar.

### 4.2 Configurar el acceso general

1. Clic derecho sobre la carpeta → **Compartir** → **Compartir**.
2. En **Acceso general**, cambiar *Restringido* por **"Cualquier persona con el enlace"**.
3. Rol a la derecha: **Lector**.
4. **Listo**.

Notas:
- Los archivos nuevos **heredan** el acceso de la carpeta. Además, `Code.gs` aplica a cada foto `setSharing(ANYONE_WITH_LINK, VIEW)` como refuerzo.
- Si en Google Workspace **no aparece** "Cualquier persona con el enlace", el administrador bloqueó el uso compartido externo. Opciones: pedir una excepción para esta carpeta/unidad organizativa, o aceptar el modo de respaldo. En ese modo el script comparte con "cualquiera **del dominio** con el enlace" y las miniaturas solo cargan para usuarios con sesión de Google de `@gprovivienda.com` abierta en el navegador.
- No comparta la carpeta como **Editor** con "cualquiera". Solo **Lector**.
- Para evitar que se borren evidencias, solo la cuenta propietaria (y, si se desea, Archivo) debe tener permiso de Editor.

### 4.3 Obtener el `FOLDER_ID`

1. Abrir la carpeta en Drive y mirar la barra de direcciones:
   ```
   https://drive.google.com/drive/folders/<ID_AQUI>
   https://drive.google.com/drive/u/0/folders/<ID_AQUI>?usp=sharing
   ```
2. El `FOLDER_ID` es el texto **entre `folders/` y el final de la URL o el primer `?`**. Suele tener unos 33 caracteres, con letras, números, `-` y `_`.
3. Alternativa: *Compartir → Copiar enlace* y extraer el mismo fragmento.

### 4.4 Dónde pegarlo en `Code.gs`

En el editor de Apps Script, sección **CONFIGURACIÓN** (al inicio, después de los comentarios), reemplazar el valor:

```js
/** ID de la carpeta de Google Drive donde se guardan las fotos.
 *  (Es la parte final de la URL: drive.google.com/drive/folders/ESTE_ID) */
const FOLDER_ID_FOTOS = '<ID_AQUI>';
```

- Solo el ID, entre comillas simples, sin `https://` ni `?usp=sharing`.
- `DRIVE_FOLDER_ID` es un alias automático (`= FOLDER_ID_FOTOS`); no se edita.
- Guardar (Ctrl+S) → ejecutar `setup()` → en el registro debe aparecer `Carpeta de fotos: Evidencias_Correspondencia`.
- Publicar una **Nueva versión** de la implementación (§11, paso C) para que la Web App use el nuevo valor.

### 4.5 Verificación

1. Registrar un envío con foto desde `index.html`.
2. En la carpeta debe aparecer `REF-<código>_<yyyyMMdd_HHmmss>.jpg`.
3. Abrir el enlace de la columna L en una ventana de **incógnito**: la imagen debe verse sin pedir inicio de sesión.

---

## 5. Backend · `Code.gs`

### 5.1 Estructura del archivo

1. Comentario de cabecera: instalación, **cambio de cuenta emisora** (§12), CORS y contrato.
2. **CONFIGURACIÓN** (constantes editables, §5.2).
3. **ESQUEMA** (encabezados, columnas, estados, tipos: no modificar).
4. Endpoints `doGet` / `doPost`.
5. Acciones: `createShipment_`, `updateStatus_`, `syncContacts_` / `sincronizarContactos`.
6. Referencias consecutivas, Drive, GPS, correos (GmailApp), utilidades.
7. Configuración inicial y pruebas: `setup()`, `testDoGet()`.

### 5.2 Constantes editables

| Constante | Valor por defecto | Descripción |
|---|---|---|
| `FOLDER_ID_FOTOS` | `<ID de Evidencias_Correspondencia>` | Carpeta de fotos (§4) |
| `SHEET_ID` | `''` | Vacío si el script está vinculado a la hoja (recomendado) |
| `SHEET_ENVIOS` / `SHEET_DIRECTORIO` | `Envios` / `Directorio` | Nombres de pestañas |
| `TIMEZONE` | `America/Panama` | Fechas, correos y trigger |
| `FOTO_THUMB_SIZE` | `w600` | Ancho de miniaturas |
| `MAX_FOTO_BYTES` | 5 MB | Límite de foto decodificada |
| `FRONTEND_URL` | `''` | **Recomendado:** `https://practicantetd-gprovivienda.github.io/correspondencia-mvp/` (agrega botón "Rastrear envío" en los correos) |
| `SEND_EMAILS` | `true` | Interruptor global de correos |
| `SENDER_ALIAS` | `kawak@gprovivienda.com` | Remitente preferido (§12) |
| `SENDER_NAME` | `notificador Provivienda` | Nombre visible del remitente |
| `REFERENCIA_INICIAL` / `REFERENCIA_MAXIMA` | `10000` / `99999` | Rango de códigos |
| `PROP_ULTIMA_REFERENCIA` | `ULTIMA_REFERENCIA` | Clave en Propiedades del script |
| `HORA_SYNC_DIARIA` | `6` | Hora del trigger diario |
| `ACRONIMOS_DEPARTAMENTO` | `['TI','IT','RRHH',…]` | Siglas en mayúsculas (igual que `DEPTO_ACRONIMOS` del frontend) |
| `ADMIN_EMAILS` | `[]` | Si tiene correos, solo ellos pueden `updateStatus`/`syncContacts`. Vacío = sin control en backend |
| `INCLUDE_DOMAIN_DIRECTORY` | `true` | Importa también el directorio del dominio Workspace |

### 5.3 Funciones públicas (ejecutables desde el editor)

| Función | Uso |
|---|---|
| `setup()` | Crea pestañas, valida carpeta, pide permisos y muestra el diagnóstico: hoja, carpeta, cuota de correo, **remitente y modo**, secuencia de códigos, trigger y People API |
| `configurarTriggerDiario()` | Crea el trigger diario de `sincronizarContactos` (no duplica) |
| `eliminarTriggerDiario()` | Elimina los triggers de sincronización **de la cuenta que lo ejecuta** |
| `sincronizarContactos()` | Importa contactos al Directorio (lo llama el trigger; también manual) |
| `reiniciarSecuenciaReferencias(n?)` | Fija el último código emitido; sin argumento, el próximo será `#10000` |
| `testDoGet()` | Prueba `doGet` y muestra conteos |
| `doGet` / `doPost` | Endpoints de la Web App |

Las funciones que terminan en `_` son privadas y no aparecen en el selector del editor.

### 5.4 Servicios, permisos y cuotas

| Servicio | Uso | Activación |
|---|---|---|
| SpreadsheetApp | `Envios` y `Directorio` | Automático |
| DriveApp | Crear y compartir fotos | Automático |
| GmailApp | Enviar correos con alias y nombre | Automático (permiso completo de Gmail) |
| MailApp | Solo `getRemainingDailyQuota()` | Automático |
| People API | `Connections.list`, `listDirectoryPeople` | **Manual:** Editor → Servicios (+) → People API (`People`) |
| LockService / PropertiesService | Exclusión mutua y secuencia de códigos | Automático |
| ScriptApp | Triggers | Automático |
| Session | Detectar la cuenta ejecutora | Automático |

Cada vez que se agrega un servicio, hay que ejecutar `setup()`, aceptar permisos y publicar una nueva versión. Cuotas (varían por cuenta): ≈1 500 destinatarios/día en Workspace, 6 min por ejecución, ~30 ejecuciones simultáneas.

### 5.5 Propiedades del script y triggers

| Elemento | Detalle |
|---|---|
| Propiedad `ULTIMA_REFERENCIA` | Último código emitido. Es del **proyecto**: se conserva al cambiar de propietario, pero **no** al copiar la hoja |
| Trigger `sincronizarContactos` | Diario ~06:00 `America/Panama`. Es del **usuario** que lo crea: no se copia ni se transfiere |

### 5.6 Configuración de la implementación (Web App)

- **Implementar → Nueva implementación → Aplicación web**
  - *Ejecutar como:* **Yo** (la cuenta propietaria; define el remitente y la cuota de correo).
  - *Quién tiene acceso:* **Cualquier usuario** (necesario para que GitHub Pages llame sin sesión).
- **Cambios de código:** *Gestionar implementaciones → ✏️ Editar → Versión: Nueva versión*. La URL `/exec` **se mantiene**.
- **Nueva implementación** (otra cuenta o una implementación adicional): genera una **URL `/exec` nueva**, que hay que pegar en `index.html`.

---

## 6. Contrato de la API

**URL base:** `APPS_SCRIPT_URL` (termina en `/exec`). Todas las respuestas son JSON con `ok`. Error: `{ "ok": false, "code": <n>, "error": "<mensaje>" }`.

### 6.1 Reglas CORS (obligatorias)

Apps Script no permite cabeceras CORS propias ni responde a `OPTIONS`. `ContentService` ya incluye `Access-Control-Allow-Origin: *` tras la redirección a `script.googleusercontent.com`. Por eso el frontend solo usa **peticiones simples**:

```js
// GET: sin cabeceras; el anti-caché va como parámetro
fetch(APPS_SCRIPT_URL + '?_t=' + Date.now(), { method: 'GET', redirect: 'follow' });

// POST: JSON como texto plano
fetch(APPS_SCRIPT_URL, {
  method: 'POST',
  redirect: 'follow',
  headers: { 'Content-Type': 'text/plain;charset=utf-8' },
  body: JSON.stringify(payload)
});
```

**Nunca** usar `application/json`, `Authorization`, `cache: 'no-cache'` ni otras cabeceras: disparan un *preflight* que falla. `doGet` acepta `?callback=fn` (JSONP); `doPost` acepta también form-urlencoded con un campo `payload`.

### 6.2 `GET /exec` → `doGet`

```json
{
  "ok": true,
  "directorio": [ { "nombre": "María Fernández", "correo": "mfernandez@gprovivienda.com", "departamento": "Transformación Digital" } ],
  "envios": [ {
    "fila": 12, "marcaTemporal": "2026-11-18T09:30:00-05:00",
    "emisor": "usuario@gprovivienda.com",
    "correoDestinatario": "mfernandez@gprovivienda.com", "nombreDestinatario": "María Fernández",
    "departamento": "Contabilidad", "descripcion": "Contrato / Adenda",
    "referencia": "10000", "estado": "En Camino",
    "recibido": "2026-11-18T10:05:00-05:00", "entregado": "", "duracion": "",
    "foto": "https://drive.google.com/thumbnail?id=<FILE_ID>&sz=w600",
    "tipoMensajeria": "Interna (Grupo Provivienda)", "nombreProveedor": "", "correoProveedor": "",
    "gps": "8.982412,-79.519876 | https://www.google.com/maps?q=8.982412,-79.519876",
    "gpsLat": 8.982412, "gpsLng": -79.519876,
    "mapsUrl": "https://www.google.com/maps?q=8.982412,-79.519876"
  } ],
  "generado": "2026-11-18T10:06:12-05:00"
}
```

- `envios` va del más reciente al más antiguo. `referencia` es **string sin `#`**. Las fechas van en ISO con `-05:00`; los textos históricos se devuelven tal cual.
- `directorio` sale sin duplicados por correo y con el departamento normalizado (la hoja no se modifica).

### 6.3 `POST /exec` → `doPost`

#### `createShipment`

```json
{
  "action": "createShipment", "userEmail": "usuario@gprovivienda.com",
  "tipoMensajeria": "Interna (Grupo Provivienda)",
  "correoDestinatario": "mfernandez@gprovivienda.com", "nombreDestinatario": "María Fernández",
  "departamento": "Transformación Digital", "descripcion": "Planos / Copias de planos",
  "nombreProveedor": "", "correoProveedor": "",
  "fotoBase64": "data:image/jpeg;base64,/9j/4AAQ...", "fotoMimeType": "image/jpeg"
}
```

| Campo | Obligatorio | Validación |
|---|---|---|
| `userEmail` | Sí | Email |
| `tipoMensajeria` | No | Contiene "extern" → `Externa`; si no → Interna |
| `correoDestinatario` | Sí | Email |
| `nombreDestinatario` | Sí | ≤150 |
| `departamento` | No | ≤150 |
| `descripcion` | Sí | ≤2000 |
| `nombreProveedor` | Si Externa | ≤150 |
| `correoProveedor` | No | Email si viene |
| `fotoBase64` | No | Data URL o Base64 de imagen, ≤5 MB |

Respuesta: `{ "ok": true, "referencia": "10000", "fila": 12, "estado": "Registrado", "marcaTemporal": "…", "fotoUrl": "…", "correo": { "enviado": true, "destinatarios": [...], "remitente": "kawak@gprovivienda.com", "modoRemitente": "cuenta" } }`

#### `updateStatus`

```json
{ "action": "updateStatus", "userEmail": "conductor@gprovivienda.com", "referencia": "10000", "estado": "En Camino", "gps": { "lat": 8.982412, "lng": -79.519876 } }
```

- `referencia`: 5 dígitos (acepta `"#10000"` o número). `estado`: `Recibido` | `En Camino` | `Entregado`. `gps`: `{lat,lng}`, `"lat,lng"` o `null`.
- Respuesta: `{ ok, referencia, estado, fecha, recibidoAutocompletado, duracion, gps, correo, envio }`.
- Mismo estado → `{ ok: true, unchanged: true, mensaje, envio }`.

#### `syncContacts`

`{ "action": "syncContacts", "userEmail": "…" }` → `{ "ok": true, "importados": 152 }`. Se conserva por compatibilidad: **el frontend no lo llama** (la sincronización es automática, §5.5).

### 6.4 Códigos de error

| `code` | Significado |
|---|---|
| 400 | Petición inválida (JSON, acción, email, referencia, imagen) |
| 403 | `userEmail` fuera de `ADMIN_EMAILS` del backend (si no está vacío) |
| 404 | Referencia inexistente |
| 409 | Envío ya entregado o intento de retroceder |
| 413 | Imagen mayor a `MAX_FOTO_BYTES` |
| 500 | Configuración (carpeta, People API) o códigos agotados |
| 503 | No se obtuvo el bloqueo en 30 s (reintentar) |

---

## 7. Reglas de negocio

### 7.1 Códigos consecutivos `#10000+`

- Se generan dentro de `LockService.getScriptLock()`, por lo que no hay duplicados aunque haya registros simultáneos.
- El último número sale de la propiedad `ULTIMA_REFERENCIA`. Si no existe, se lee **solo** la celda G del último registro: si es ≥ 10000 se continúa desde ahí; si no, se empieza en 10000.
- Antes de usar un número, se busca en la columna G con `TextFinder` y se salta si ya existe (por ejemplo, códigos históricos).
- La propiedad se actualiza **después** de escribir la fila. Si se pasa de 99999, se devuelve error.
- `reiniciarSecuenciaReferencias()` → el próximo será `#10000`; `reiniciarSecuenciaReferencias(12345)` → el próximo será `#12346`.

### 7.2 Estados

`Registrado(0) → Recibido(1) → En Camino(2) → Entregado(3)`.
- Se puede avanzar saltando pasos; retroceder devuelve 409.
- Repetir el mismo estado devuelve `unchanged`. `Entregado` es terminal.
- **Autocompletado:** de `Registrado` a `En Camino` con I vacía → I recibe la misma hora.
- La hora de "En Camino" no tiene columna propia.

### 7.3 Duración

`K = Entregado − (Recibido || Marca temporal)`, en formato `<1m` · `Xm` · `Xh Ym` · `Xd Yh Zm`.

### 7.4 Normalización de departamentos (solo lectura)

Recorta y colapsa espacios, aplica Title Case, deja en minúscula los conectores (`de, del, la, las, el, los, y, e, o, u, a, al, en, por, para, con`) salvo al inicio y respeta siglas (`TI`, `RRHH`, …). Si no hay departamento, queda `""`. La regla se aplica en `sincronizarContactos()`, en `doGet` (directorio) y en el frontend (`normDepto`, también al mostrar la col. E). Ejemplo: `"  TRANSFORMACIÓN   digital "` → `"Transformación Digital"`. **No existe endpoint para editar departamentos.**

### 7.5 Correos

| Evento | Para | Asunto |
|---|---|---|
| Registro | Destinatario | `Nueva correspondencia registrada · Ref. #10000` |
| En Camino | Destinatario | `Su correspondencia va en camino · Ref. #10000` |
| Entregado | Emisor + destinatario | `Correspondencia entregada · Ref. #10000` |

- Se envían como HTML con GmailApp (compatible con Outlook): código, línea de tiempo, foto, botón de Maps (si hay GPS) y "Rastrear envío" (si hay `FRONTEND_URL`).
- Remitente según `resolverRemitente_()`: `cuenta` · `alias` · `fallback` (§12.0). El nombre visible es siempre `SENDER_NAME`.
- Un fallo de correo nunca revierte la operación; se informa en `correo.enviado = false`.
- Las plantillas de correo usan sus propios colores por estado (azul/ámbar/verde), independientes de la paleta de la UI.

### 7.6 Fotos

- El navegador comprime a máx. 800 px de ancho, JPEG 0.75.
- El backend guarda la foto en `FOLDER_ID_FOTOS` como `REF-<código>_<fecha>.jpg` y aplica `ANYONE_WITH_LINK/VIEW`. Si la política del dominio lo impide, usa `DOMAIN_WITH_LINK`; si tampoco, la deja privada.
- Si falla la escritura en la hoja, la foto va a la papelera.

---

## 8. Frontend · `index.html`

### 8.1 Arquitectura de archivo único

- Un solo `index.html` sin compilación ni dependencias locales: HTML + `<style>` + `<script>` inline.
- Recursos externos (CDN):
  - Tailwind CSS Play CDN `https://cdn.tailwindcss.com/3.4.17`, con `tailwind.config` y componentes en `<style type="text/tailwindcss">`.
  - Google Fonts: Inter y JetBrains Mono.
  - Google Identity Services: `https://accounts.google.com/gsi/client`.
- Único activo local: `./logo-provivienda.jpg`.
- Orden interno del archivo:
  1. `<head>`: Tailwind y su configuración (paleta `brand`), clases de componentes (`.card`, `.input`, `.btn-primary`, `.chip`, `.seg`, `.tipo-card`, `.ro-label`, `.ro-value`), animaciones, `logoFallback()` y GIS.
  2. `<body>`: vistas `#view-login` y `#view-app` (cabecera, `<main>` con 3 secciones `data-tab-panel`, `#nav-mobile`), modales `#camara-modal` y `#lightbox`, y `#toasts`.
  3. `<script>`: configuración → constantes → `state` → utilidades → API → autenticación → navegación → datos → Nuevo Envío → cámara → panel → rastreo → eventos → `boot()`.

### 8.2 Dónde están las constantes

Al inicio del `<script>` principal, bajo el bloque **CONFIGURACIÓN · edite estas tres constantes**:

```js
const APPS_SCRIPT_URL  = 'https://script.google.com/macros/s/<ID_IMPLEMENTACION>/exec';
const GOOGLE_CLIENT_ID = '<NUMERO>-<HASH>.apps.googleusercontent.com';
const ADMIN_EMAILS     = ['practicantetd@gprovivienda.com', '…'];   // en minúsculas
// justo debajo:
const ALLOWED_DOMAINS  = ['gprovivienda.com'];                      // dominio permitido para iniciar sesión
```

Otras constantes (no suelen cambiar):

| Constante | Valor | Uso |
|---|---|---|
| `TIPOS`, `ESTADOS` | Iguales al backend | Contrato (no modificar) |
| `TZ` | `America/Panama` | Formato de fechas |
| `FOTO_MAX_W` / `FOTO_QUALITY` | `800` / `0.75` | Compresión |
| `GPS_TIMEOUT_MS` | `8000` | Límite del GPS |
| `MAX_RECIENTES` | `4` | Destinatarios recientes |
| `PAGE_SIZE` | `60` | Tarjetas por página en el panel |
| `AUTO_REFRESH_MS` | `120000` | Refresco silencioso |
| `DESC_CHIPS` | Top 5 del histórico (§8.6) | Chips de descripción |
| `DESC_CHIPS_GENERALES` | Lista general | Respaldo de los chips |
| `DEPTO_MINUSCULAS` / `DEPTO_ACRONIMOS` | Igual que en `Code.gs` | Normalización de departamentos |
| `TABS` | `nuevo`, `panel` (admin), `rastreo` | Navegación |
| `ESTADO_UI`, `ACCIONES` | Clases por estado y botones de un toque | Presentación |

### 8.3 Pestañas, roles y navegación

| Pestaña (`data-tab`) | Visible para | Contenido |
|---|---|---|
| `nuevo` · Nuevo Envío | Todos | Formulario de 3 pasos + lateral "Mis últimos registros" |
| `panel` · Panel Operador / Conductor | Solo `ADMIN_EMAILS` (no se renderiza para el resto) | Estadísticas, búsqueda, filtros, tarjetas con botones de un toque |
| `rastreo` · Mis Envíos | Todos | Envíos donde el usuario es emisor o destinatario |

- La navegación usa el hash (`#nuevo`, `#panel`, `#rastreo`) y respeta el botón Atrás.
- Pestaña inicial: la del hash. Si no hay, en pantallas < 1024 px los administradores abren `panel` y el resto `nuevo`.
- Desde 1024 px (`lg`) las pestañas van en la cabecera; por debajo, en una barra inferior fija con contadores de envíos activos.

### 8.4 Gestión de estado y `localStorage`

Objeto global `state`:

| Clave | Contenido |
|---|---|
| `user` / `isAdmin` | Perfil de la sesión y rol |
| `tab` | Pestaña activa |
| `loaded` / `loading` / `lastLoad` | Ciclo de carga del GET |
| `directorio` / `envios` | Datos del backend (con índice de búsqueda `_q`) |
| `tipo` | `Interna (Grupo Provivienda)` o `Externa` |
| `contacto` | Persona elegida `{ nombre, correo, departamento }` o `null` |
| `foto` | `{ dataUrl, w, h, kb }` o `null` |
| `resultados` / `activeIdx` | Autocompletado |
| `submitting` / `busy` | Antidoble envío y cambios de estado en curso (`Map` referencia → fase `gps`/`save`) |
| `panelFilter` / `panelLimit` / `rastreoFilter` | Filtros y paginación |
| `cam` | `{ stream, facing }` de la cámara |

`localStorage` (todas las lecturas y escrituras van en `try/catch`):

| Clave | Contenido |
|---|---|
| `corr_session_v1` | `{ email, name, picture, hd, loginAt }`. Evita repetir el login; se borra con "Cerrar sesión" |
| `corr_recientes_v1:<email>` | Últimos 4 destinatarios del usuario |

Los datos de envíos **no** se guardan en `localStorage`: se piden al backend al abrir la página, al volver a la pestaña (si pasó más de 1 min), cada 2 min y con "Actualizar".

### 8.5 Autenticación (Google Identity Services)

1. `initGis()` espera a que cargue GIS e inicializa con `client_id`, `ux_mode: 'popup'` y `hd: 'gprovivienda.com'` (sugerencia de dominio).
2. `handleCredential()` decodifica el JWT (base64url + UTF-8), exige `email_verified` y que el correo termine en `@gprovivienda.com` (`ALLOWED_DOMAINS`).
3. Guarda el perfil en `corr_session_v1` y llama a `startApp()`.
4. `isAdmin = ADMIN_EMAILS.includes(email)` controla la pestaña Panel.

> Esta restricción es **visual** (el JWT no se verifica en un servidor); ver §14.

### 8.6 Nuevo Envío

1. **Tipo:** tarjetas Interna / Externa. Externa muestra `nombreProveedor` (obligatorio) y `correoProveedor` (opcional).
2. **Destinatario:**
   - Chips de **recientes** y un **único campo editable "Nombre del destinatario"** con autocompletado en tiempo real sobre el Directorio (por nombre, correo o departamento; ↑ ↓ Enter Esc).
   - Al elegir a la persona, **Correo** y **Departamento / Proyecto** se muestran como texto de solo lectura (`#ro-correo`, `#ro-depto`). El departamento queda en blanco si la persona no lo tiene.
   - Si se edita el nombre, la selección se descarta. Si el texto coincide exactamente con un único nombre, se selecciona solo.
   - Sin persona elegida no se puede enviar.
3. **Descripción:** 5 chips de selección rápida (un toque agrega, otro toque quita).
   - `DESC_CHIPS` = **Top 5 de `historico.csv`** (541 registros clasificados por palabras clave el 18/11/2026): Planos / Copias de planos (265) · Permisos y aprobaciones (141) · Contrato / Adenda (118) · Memorial para firma (112) · Documentos legales (cédula, finca) (83).
   - Respaldo `DESC_CHIPS_GENERALES`: Contrato / Escritura · Cheque / Factura · Documentos firmados · Expediente de cliente · Sobre confidencial.
   - El CSV **no** se lee en el navegador ni se publica.
4. **Foto:** cámara híbrida (§8.7) o "Subir archivo".
5. **Envío:** botón bloqueado con spinner (`state.submitting`). Si sale bien, se muestra una tarjeta con el código gigante `#10000`, "Copiar código", "Ver en Mis Envíos" y "Registrar otro envío", y los datos se recargan en segundo plano.

### 8.7 Cámara híbrida (`getUserMedia` + `<canvas>`)

```
"Abrir Cámara" ─┬─ ¿navigator.mediaDevices.getUserMedia?
                │     ├─ Sí → modal #camara-modal
                │     │        getUserMedia({ video: { facingMode: 'environment', width: {ideal:1920}, height: {ideal:1080} }, audio: false })
                │     │        ├─ OverconstrainedError / NotFoundError → reintento { video: true }  (webcam de PC)
                │     │        ├─ vista previa <video playsinline muted autoplay> (espejo si es cámara frontal)
                │     │        ├─ "Cambiar cámara" si hay ≥ 2 videoinput
                │     │        └─ "Tomar Foto" → aJpegComprimido(video) → cierra modal y detiene tracks
                │     └─ No  → <input type="file" accept="image/*" capture="environment"> (cámara nativa del sistema)
"Subir archivo" ──── <input type="file" accept="image/*"> → comprimirArchivo() → aJpegComprimido(img)
```

- `aJpegComprimido(fuente, ancho, alto)` escala a **máx. 800 px de ancho** en `<canvas>`, rellena de blanco (por los PNG transparentes) y genera `toDataURL('image/jpeg', 0.75)`, de unos 50–200 KB.
- Errores con mensaje claro: `NotAllowedError` (permiso denegado), `NotFoundError` (sin cámara), `NotReadableError` (cámara en uso), `SecurityError`.
- La cámara se libera al cerrar el modal, con Esc o al ocultar la pestaña.
- **Requiere HTTPS** (GitHub Pages) o `localhost`. En iframes o vistas previas puede estar bloqueada.

### 8.8 Panel Operador / Conductor y geolocalización

- Estadísticas: por recibir, recibidos, en camino y entregados hoy. Búsqueda que acepta `#10000`, nombre o departamento; filtros Activos / Entregados / Todos; paginación de 60.
- Cada tarjeta muestra la foto (se amplía en `#lightbox` con `sz=w1600`), el código `#10000`, el estado y tres botones de un toque `Recibido → En Camino → Entregado`:
  - El siguiente paso va resaltado y los ya cumplidos, deshabilitados.
  - "Entregado" pide confirmación.
  - No hay botón de sincronizar contactos: solo **"Actualizar lista"**.
- `obtenerGps()` llama a `navigator.geolocation.getCurrentPosition` con `enableHighAccuracy`, `timeout: 8000` y `maximumAge: 60000`, más un **temporizador duro de 8 s** (el `timeout` nativo no cuenta el tiempo de espera del permiso). Si tarda, falla o se niega, resuelve `null` y el cambio de estado se envía igual con `gps: null`.
- Durante la operación el botón muestra "GPS…" y luego "Guardando…". La tarjeta se actualiza con el `envio` que devuelve el backend.

### 8.9 Mis Envíos

Secciones "Enviados por mí" (`emisor`) y "Para mí" (`correoDestinatario`), con búsqueda (código `#10000`, estado o palabra) y contadores. Cada tarjeta muestra la línea de tiempo de 4 pasos con fechas, la duración, la foto y "Ver ubicación en Google Maps" si hay `mapsUrl`.

### 8.10 Identidad visual

- **Paleta única** `brand` en `tailwind.config`: `50 #fef2f2 · 100 #fee2e2 · 200 #fecaca · 300 #fca5a5 · 400 #f87171 · 500 #ef4444 · 600 #dc2626 · 700 #b91c1c · 800 #991b1b · 900 #7f1d1d`, más neutros `slate`.
- Estados: Registrado gris · Recibido brand-50/700 · En Camino brand-100/800 · Entregado sólido brand-800.
- **Logo** `./logo-provivienda.jpg` sobre fondo blanco: recuadro blanco en la cabecera roja y tarjeta blanca en el login. Si no carga, `logoFallback()` oculta la imagen y muestra "Grupo Provivienda" / "GP" en rojo. Admite rutas alternativas con `data-alts="ruta1|ruta2"`.
- Responsive: probado a 375, 1024 y 1280 px sin desplazamiento horizontal.

### 8.11 Comunicación con el backend

| Función | Petición | Notas |
|---|---|---|
| `apiGet()` | `GET APPS_SCRIPT_URL?_t=<timestamp>` | Sin cabeceras |
| `apiPost(payload)` | `POST` con `Content-Type: text/plain;charset=utf-8` | El cuerpo es `JSON.stringify(payload)` |
| `parseJson()` | — | Si la respuesta no es JSON, indica revisar la URL `/exec` y el acceso "Cualquier usuario" |

Cualquier `ok: false` se convierte en `Error` con `code` (409/404 recargan los datos).

---

## 9. Google Cloud · OAuth y Orígenes autorizados

El `GOOGLE_CLIENT_ID` solo sirve para el botón "Iniciar sesión con Google". Es **independiente** de la cuenta que ejecuta Apps Script.

1. [console.cloud.google.com](https://console.cloud.google.com/) → crear o seleccionar un proyecto (de preferencia propiedad de una cuenta institucional).
2. **APIs y servicios → Pantalla de consentimiento de OAuth** (o *Google Auth Platform → Branding/Audience*):
   - Tipo de usuario: **Interno** (solo cuentas `@gprovivienda.com`).
   - Nombre de la app, correo de soporte y logo opcional.
   - Ámbitos: solo `openid`, `email`, `profile`.
3. **Credenciales → Crear credenciales → ID de cliente de OAuth → Aplicación web**:
   - **Orígenes autorizados de JavaScript:**
     - `https://practicantetd-gprovivienda.github.io` → sin ruta `/correspondencia-mvp/` y sin `/` final.
     - *(Pruebas locales, opcional)* `http://localhost:5500` (o el puerto usado).
   - **URI de redireccionamiento:** no se requieren (flujo popup).
4. Copiar el ID (`…apps.googleusercontent.com`) en `GOOGLE_CLIENT_ID` de `index.html`.
5. Los cambios pueden tardar de 5 a 10 min. Error `origin_mismatch` o botón ausente → revisar el paso 3.

---

## 10. GitHub Pages · publicación del frontend

1. Repositorio (p. ej. `practicantetd-gprovivienda/correspondencia-mvp`) con, en la **raíz**:
   - `index.html`
   - `logo-provivienda.jpg` (nombre exacto, en minúsculas)
2. **No** subir `historico.csv.csv`.
3. *Settings → Pages → Build and deployment → Source: Deploy from a branch → Branch: `main` / `(root)` → Save*.
4. URL resultante: `https://practicantetd-gprovivienda.github.io/correspondencia-mvp/`.
5. Cada cambio: commit + push. El despliegue tarda 1–2 min; los usuarios recargan con **Ctrl+F5**.
6. El repositorio de GitHub Pages gratuito es **público**: `APPS_SCRIPT_URL` y `GOOGLE_CLIENT_ID` quedan visibles. El ID de cliente no es secreto; las implicaciones de la URL `/exec` se explican en §14.

---

## 11. Despliegue completo desde una cuenta nueva (checklist)

> Para instalar el sistema desde cero o moverlo a una cuenta nueva (p. ej. `kawak@gprovivienda.com`). Si solo se quiere cambiar el remitente, ver §12.

### A. Datos (Google Drive)

- [ ] **Hoja de cálculo.** Elegir una:
  - **Proyecto existente:** recibirla compartida como Editor o con la propiedad transferida (§12.2). Conserva el script, la propiedad `ULTIMA_REFERENCIA` y las URLs de las fotos.
  - **Copia:** *Archivo → Hacer una copia* desde la cuenta nueva. Copia el script vinculado, pero **no** las propiedades ni los triggers.
  - **Desde cero:** hoja nueva; `setup()` crea `Envios` y `Directorio`.
- [ ] **Carpeta `Evidencias_Correspondencia`:** crearla o recibirla, configurar "Cualquier persona con el enlace · Lector" y copiar el `FOLDER_ID` (§4).

### B. Script (Apps Script)

- [ ] Abrir la hoja → **Extensiones → Apps Script**. Si está vacío, pegar `Code.gs`.
- [ ] **Servicios (+) → People API** (`People`), si no aparece.
- [ ] *Configuración del proyecto* → Zona horaria `America/Panama`.
- [ ] Editar en **CONFIGURACIÓN**: `FOLDER_ID_FOTOS`, `SHEET_ID` (`''`), `SENDER_ALIAS`, `SENDER_NAME`, `FRONTEND_URL` y, opcionalmente, `ADMIN_EMAILS`.
- [ ] Ejecutar **`setup()`** y aceptar todos los permisos. Verificar en el registro: carpeta OK, `Remitente: … · modo=cuenta|alias`, People API OK.
- [ ] Secuencia: si se copió la hoja o hay códigos de prueba, ejecutar `reiniciarSecuenciaReferencias(<último código real>)` o `reiniciarSecuenciaReferencias()` para empezar en `#10000`.
- [ ] Ejecutar **`configurarTriggerDiario()`** → `✅ Activador creado`.
- [ ] *(Opcional)* Ejecutar `sincronizarContactos()` y `testDoGet()`.

### C. Implementación

- [ ] **Implementar → Nueva implementación → Aplicación web** → *Ejecutar como: Yo* → *Acceso: Cualquier usuario* → **Implementar** → copiar la URL `/exec`.

### D. Frontend

- [ ] Google Cloud: ID de cliente OAuth con el origen `https://practicantetd-gprovivienda.github.io` (§9).
- [ ] `index.html`: `APPS_SCRIPT_URL`, `GOOGLE_CLIENT_ID`, `ADMIN_EMAILS` y `ALLOWED_DOMAINS`.
- [ ] GitHub: subir `index.html` y `logo-provivienda.jpg` y activar Pages (§10).

### E. Pruebas de aceptación

- [ ] El logo se ve en el login y en la cabecera.
- [ ] Iniciar sesión con una cuenta `@gprovivienda.com` funciona; otra cuenta es rechazada.
- [ ] Al escribir parte de un nombre y elegirlo, el correo y el departamento aparecen como solo lectura.
- [ ] "Abrir Cámara" funciona en PC (webcam) y en celular (cámara trasera); "Tomar Foto" deja una vista previa de ≤ 800 px.
- [ ] Al registrar un envío: código consecutivo `#10000…`, archivo en la carpeta de evidencias, miniatura visible y correo de **notificador Provivienda**.
- [ ] Panel: de Registrado directo a En Camino → I autocompletada y correo "en camino". Entregado → J, K y P llenos y correo a ambos.
- [ ] Mis Envíos: línea de tiempo, duración y enlace a Maps.
- [ ] Al día siguiente: *Ejecuciones* muestra `sincronizarContactos` a las ~06:00.

---

## 12. Cómo cambiar la cuenta emisora de correos

### 12.0 Cómo se decide el remitente

Los correos los envía la **cuenta que creó la implementación** (*Ejecutar como: Yo*), con el nombre `SENDER_NAME`. `resolverRemitente_()` decide así:

| Situación | Modo (`setup()`) | "De:" que ve el destinatario |
|---|---|---|
| La cuenta ejecutora **es** `SENDER_ALIAS` | `cuenta` | `notificador Provivienda <kawak@gprovivienda.com>` |
| `SENDER_ALIAS` es un alias **verificado** de la cuenta ejecutora | `alias` | `notificador Provivienda <kawak@gprovivienda.com>` |
| Ninguna de las anteriores | `fallback` | `notificador Provivienda <cuenta ejecutora>` (Responder a: `SENDER_ALIAS`) |

| | Opción 1 · Alias de Gmail | Opción 2 · Migración total |
|---|---|---|
| ¿Cambia la cuenta que ejecuta? | No | Sí |
| ¿Cambia la URL `/exec`? | **No** | **Sí** (actualizar `index.html`) |
| Dueño de hoja, carpeta y triggers | Cuenta actual | Cuenta nueva |
| Cuota diaria de correos | Cuenta actual | Cuenta nueva |
| Recomendado cuando… | Solo se quiere otra dirección visible | La cuenta actual es personal, temporal o se desactivará |

### 12.1 Opción 1 · Alias de Gmail (`SENDER_ALIAS`)

1. Iniciar sesión en Gmail con la **cuenta ejecutora** (la que creó la implementación), no con el alias.
2. ⚙ **Configuración → Ver todos los ajustes → Cuentas** (o *Cuentas e importación*).
3. **"Enviar mensaje como" → "Añadir otra dirección de correo electrónico"**:
   - Nombre: `notificador Provivienda`.
   - Dirección: `kawak@gprovivienda.com` (o el nuevo correo).
   - Dejar marcado **"Tratar como un alias"** → *Siguiente paso*.
4. **Verificar:** Gmail envía un enlace o código al nuevo correo y su titular debe confirmarlo.
   - Si es otro dominio, pedirá datos SMTP: los ingresa el titular, nunca van en el código.
   - En Workspace, el administrador puede tener que permitir el envío como otra dirección.
5. Confirmar que la dirección aparece sin la marca *"no verificada"*.
6. En `Code.gs` (CONFIGURACIÓN) editar y guardar:
   ```js
   const SENDER_ALIAS = 'kawak@gprovivienda.com';
   const SENDER_NAME  = 'notificador Provivienda';
   ```
7. Ejecutar `setup()` → debe mostrar `modo=alias`. Si muestra `modo=fallback`, el alias aún no está verificado en **esa** cuenta (volver al paso 4).
8. **Implementar → Gestionar implementaciones → ✏️ Editar → Versión: Nueva versión → Implementar.** La URL `/exec` no cambia y `index.html` no se toca.
9. Registrar un envío de prueba y revisar el remitente del correo recibido.

> Cambiar solo `SENDER_NAME` también requiere los pasos 6, 8 y 9.

### 12.2 Opción 2 · Migración total a una cuenta nueva propietaria

**A. Transferir permisos (desde la cuenta ACTUAL)**

1. **Hoja:** *Compartir* → agregar la cuenta nueva como **Editor**.
2. **Carpeta `Evidencias_Correspondencia`:** *Compartir* → cuenta nueva como **Editor**. Mantener el acceso general "Cualquier persona con el enlace · Lector" (§4.2).
3. *(Recomendado)* En ambos, elegir **"Transferir propiedad"**. La cuenta nueva debe aceptar la solicitud en Drive; en Workspace puede hacerlo el administrador.
4. Ejecutar **`eliminarTriggerDiario()`** desde la cuenta actual: los triggers son personales y la cuenta nueva no puede borrarlos.

**B. Configurar el script (desde la cuenta NUEVA)**

5. Abrir la hoja → **Extensiones → Apps Script**. Es el mismo proyecto vinculado: conserva el código, People API y `ULTIMA_REFERENCIA`, así que los códigos `#10000…` siguen sin saltos.
6. Ajustar `SENDER_ALIAS` (correo de la cuenta nueva o un alias suyo) y `SENDER_NAME`. Cambiar `FOLDER_ID_FOTOS` solo si se usa otra carpeta. Guardar.
7. Ejecutar **`setup()`**, aceptar permisos y verificar `modo=cuenta` (o `alias`), la carpeta y People API.
8. Ejecutar **`configurarTriggerDiario()`**.

**C. Nueva implementación (desde la cuenta NUEVA)**

9. **Implementar → Nueva implementación → Aplicación web** → *Ejecutar como: Yo (cuenta nueva)* → *Acceso: Cualquier usuario* → **Implementar** → copiar la **nueva URL `/exec`**.
   > No sirve editar la implementación anterior: seguiría ejecutándose como la cuenta antigua.

**D. Frontend**

10. En `index.html`: `const APPS_SCRIPT_URL = 'https://script.google.com/macros/s/<NUEVO_ID>/exec';`
11. Commit y push a GitHub Pages, esperar 1–2 min y recargar con Ctrl+F5. `GOOGLE_CLIENT_ID` **no cambia**.

**E. Verificar y cerrar**

12. Registrar un envío, avanzar estados y confirmar los correos con el nuevo remitente.
13. Al día siguiente, revisar en *Ejecuciones* que `sincronizarContactos` corrió bajo la cuenta nueva.
14. Desde la cuenta antigua: **Gestionar implementaciones → Archivar** la implementación anterior. Opcionalmente, quitarle el acceso a la hoja y a la carpeta.

**Reversión:** mientras la implementación antigua no se archive, basta con volver a poner su URL en `APPS_SCRIPT_URL` y recrear su trigger con `configurarTriggerDiario()` desde la cuenta antigua.

---

## 13. Importación del histórico (`historico.csv.csv`)

Archivo local con 541 registros:

- **Columnas desplazadas:** tiene una columna vacía sin encabezado en L, así que la foto, el tipo y el proveedor están corridos una posición. **Eliminar esa columna** antes de pegar en `Envios` (esquema A–P, §3.1).
- **Codificación:** revisar los acentos al importar (*Archivo → Importar*, eligiendo la codificación correcta).
- **Referencias:** van de 268 a 54 155; 99 tienen menos de 5 dígitos. No chocan con la secuencia nueva (`TextFinder`), pero las de menos de 5 dígitos no se pueden actualizar con `updateStatus`.
- **Estados:** la mayoría están vacíos y se verían como "activos" en el Panel. Conviene marcar `Entregado` en la hoja los que ya se entregaron.
- **Orden:** pegar el histórico **antes** de los registros nuevos (el sistema asume orden cronológico de filas). Después, si la última fila es histórica, ejecutar `reiniciarSecuenciaReferencias(<último código nuevo>)`.

---

## 14. Seguridad y limitaciones conocidas

- **Web App pública:** quien conozca la URL `/exec` puede leer todos los envíos (`doGet`) y llamar a `doPost`. El filtro de "Mis Envíos" ocurre en el navegador.
- **Identidad no verificada en el backend:** `userEmail` lo envía el frontend. Mejora posible: verificar el ID token (`https://oauth2.googleapis.com/tokeninfo?id_token=…`, comprobando `aud`, `hd` y `exp`) y llenar `ADMIN_EMAILS` en `Code.gs`.
- **Fotos públicas por enlace:** cualquiera con la URL puede verlas (es necesario para las miniaturas sin sesión).
- **La sesión local no caduca:** se cierra con "Cerrar sesión" o borrando `localStorage`.
- **Sin fecha de "En Camino"** (el esquema de 16 columnas no la tiene).
- **Destinatarios fuera del Directorio:** no se pueden registrar hasta la siguiente sincronización diaria.
- **Columna E histórica** sin normalizar en la hoja (solo se normaliza al mostrarla).
- **Límite de 90 000 códigos** (10000–99999).

---

## 15. Solución de problemas

| Síntoma | Causa probable | Acción |
|---|---|---|
| "El servidor no devolvió JSON" | URL `/exec` incorrecta, acceso distinto de "Cualquier usuario" o permisos sin aceptar | Revisar la implementación; ejecutar `setup()`; publicar nueva versión |
| Error CORS en la consola | Se envió `application/json` u otra cabecera | Usar `text/plain;charset=utf-8` (§6.1) |
| No aparece el botón de Google / `origin_mismatch` | Origen no registrado | Agregar `https://practicantetd-gprovivienda.github.io` (§9) |
| "La cuenta … no pertenece al dominio corporativo" | Cuenta fuera de `ALLOWED_DOMAINS` | Usar una cuenta `@gprovivienda.com` |
| No se ve la pestaña Panel | Correo fuera de `ADMIN_EMAILS` en `index.html` | Agregarlo en minúsculas y publicar |
| Se ve "Grupo Provivienda"/"GP" en texto en vez del logo | Falta `logo-provivienda.jpg` en la raíz del repositorio o el nombre difiere (mayúsculas) | Subir el archivo con el nombre exacto |
| Las miniaturas no cargan | Carpeta o archivos sin "Cualquier persona con el enlace" | §4.2; revisar la política de compartir del dominio |
| "Carpeta de fotos" con error en `setup()` | `FOLDER_ID_FOTOS` mal copiado o sin acceso | §4.3–4.4 |
| "Permiso de cámara denegado" | Permiso bloqueado en el navegador | Candado de la barra de direcciones → Cámara: Permitir; o "Subir archivo" |
| La cámara no abre | Página sin HTTPS o dentro de un iframe | Usar la URL de GitHub Pages |
| "Elija al destinatario en la lista del directorio" | Nombre escrito sin elegirlo, o persona no sincronizada | Elegirlo de la lista; esperar la sincronización o revisar People API |
| Directorio vacío | People API sin habilitar o trigger sin crear | Servicios → People API; `configurarTriggerDiario()`; `sincronizarContactos()` |
| Los correos salen desde otra cuenta | `modo=fallback` | §12 |
| El código no empieza en #10000 | Hay una propiedad o un último registro mayor | `reiniciarSecuenciaReferencias()` |
| Error 503 "sistema ocupado" | Concurrencia alta | Reintentar |
| Los cambios en `Code.gs` no surten efecto | No se publicó una nueva versión | Gestionar implementaciones → Editar → Nueva versión |

---

## 16. Guía para agentes de IA

**Invariantes que no se deben romper:**

1. Encabezados y orden A–P de `Envios`; pestañas `Envios` y `Directorio`.
2. Literales: `Interna (Grupo Provivienda)`, `Externa`, `Registrado`, `Recibido`, `En Camino`, `Entregado`.
3. Contrato JSON de §6: campos en español, `ok` en todas las respuestas, `referencia` como string sin `#` en el JSON y número en la hoja.
4. CORS: solo peticiones simples (GET sin cabeceras; POST `text/plain`).
5. Escrituras en `Envios`/`Directorio` dentro de `obtenerLock_()`; `ULTIMA_REFERENCIA` se actualiza solo después de escribir la fila.
6. Los correos nunca interrumpen la operación principal.
7. Sin endpoints de edición de departamentos; en la UI, correo y departamento del destinatario son de solo lectura y vienen del Directorio. `normDepto` (frontend) y `normalizarDepartamento_` (backend) deben aplicar la misma regla.
8. Sin botón manual de sincronización; el Directorio se actualiza solo con el trigger diario.
9. UI solo con la paleta `brand` roja y neutros `slate`; referencias siempre mostradas con `#` (`fmtRef`).
10. Toda foto pasa por `aJpegComprimido()` (máx. 800 px, JPEG 0.75).
11. Tras cambiar `Code.gs`: `setup()` si hay servicios nuevos y luego **Nueva versión** (la URL se mantiene). Cambiar de cuenta ejecutora implica una **Nueva implementación** y una nueva URL en `index.html`.

**Tareas frecuentes y dónde tocar:**

| Tarea | Archivo · ubicación |
|---|---|
| Agregar un administrador del panel | `index.html` → `ADMIN_EMAILS` (y opcionalmente `Code.gs` → `ADMIN_EMAILS`) |
| Cambiar los chips de descripción | `index.html` → `DESC_CHIPS` |
| Cambiar la carpeta de fotos | `Code.gs` → `FOLDER_ID_FOTOS` (§4) + Nueva versión |
| Cambiar el remitente | §12 |
| Cambiar la hora de sincronización | `Code.gs` → `HORA_SYNC_DIARIA`, luego `eliminarTriggerDiario()` + `configurarTriggerDiario()` |
| Agregar una sigla de departamento | `Code.gs` → `ACRONIMOS_DEPARTAMENTO` **y** `index.html` → `DEPTO_ACRONIMOS` |
| Cambiar colores | `index.html` → `tailwind.config.theme.extend.colors.brand` |

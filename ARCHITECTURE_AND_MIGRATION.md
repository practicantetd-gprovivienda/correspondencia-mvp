# Correspondencia MVP · Arquitectura, contrato y guía de migración

> **Sistema de Trazabilidad y Gestión de Correspondencia — Grupo Provivienda (Panamá)**
> Documento técnico portable para personas y para asistentes de IA que deban mantener, extender o migrar el sistema.
> Fuente de verdad del comportamiento: `Code.gs` (backend) e `index.html` (frontend). Si este documento y el código difieren, manda el código.

---

## 0. Ficha rápida (legible por máquina)

```yaml
proyecto: correspondencia-mvp
costo_operativo: 0 USD (Google Workspace + GitHub Pages)
frontend:
  archivo: index.html            # único archivo: HTML + Tailwind CDN + JS vanilla
  hosting: GitHub Pages          # https://practicantetd-gprovivienda.github.io/correspondencia-mvp/
  auth: Google Identity Services (ID token JWT decodificado en el navegador)
  persistencia_local: localStorage (corr_session_v1, corr_recientes_v1:<email>)
  identidad_visual: paleta roja "brand" #fef2f2 → #7f1d1d (Tailwind), logo ./logo-provivienda.png
  destinatario: un solo campo "Nombre" con autocompletado del Directorio; correo y departamento de solo lectura
  camara: getUserMedia (facingMode environment) en modal + "Subir archivo"; compresión canvas máx. 800 px, JPEG 0.75
  sincronizacion_contactos: solo automática (trigger diario); sin botón en la UI
backend:
  archivo: Code.gs               # Google Apps Script, desplegado como Web App
  endpoints: [doGet, doPost]
  doPost_actions: [createShipment, updateStatus, syncContacts]
  ejecutar_como: propietario de la implementación (recomendado kawak@gprovivienda.com)
  acceso: "Cualquier usuario"
datos:
  base: Google Sheets
  pestañas: [Envios (16 columnas A–P), Directorio (3 columnas A–C)]
  fotos: carpeta de Google Drive (FOLDER_ID_FOTOS), compartidas "cualquiera con el enlace: lector"
servicios_google: [SpreadsheetApp, DriveApp, GmailApp, MailApp (solo cuota), People API (avanzado), LockService, PropertiesService, ScriptApp]
zona_horaria: America/Panama
estados: [Registrado, Recibido, En Camino, Entregado]   # orden estricto, sin retroceso
referencias: consecutivas 10000..99999, mostradas como "#10000"
remitente_correo: { alias: kawak@gprovivienda.com, nombre: "notificador Provivienda" }
trigger: sincronizarContactos diario 06:00 America/Panama
cors: POST con Content-Type text/plain;charset=utf-8, sin cabeceras personalizadas
```

---

## 1. Arquitectura

```
┌──────────────────────────────┐        GET  /exec                 ┌──────────────────────────────┐
│  index.html (GitHub Pages)   │ ───────────────────────────────▶ │  Code.gs · Web App (Apps     │
│  · Login Google (GIS)        │                                   │  Script) "Ejecutar como: yo" │
│  · Nuevo Envío               │        POST /exec (text/plain)    │  · doGet  → JSON             │
│  · Panel Operador/Conductor  │ ───────────────────────────────▶ │  · doPost → createShipment   │
│  · Mis Envíos (rastreo)      │ ◀─── JSON {ok,...} (CORS *) ───── │             updateStatus     │
└──────────────────────────────┘                                   │             syncContacts     │
        │ geolocalización                                          └───────┬───────┬───────┬──────┘
        │ cámara + canvas (compresión)                                     │       │       │
        ▼                                                                   ▼       ▼       ▼
  Navegador del usuario                                       Google Sheets  Google Drive  Gmail
                                                              (Envios,        (fotos)       (notificaciones
                                                               Directorio)                   HTML)
                                                                   ▲
                                         Trigger diario 06:00 ─────┘  People API → Directorio
```

### 1.1 Flujo de un envío

| Paso | Actor | Acción en la UI | Backend | Efecto en Sheets | Correo |
|---|---|---|---|---|---|
| 1 | Cualquier colaborador | Nuevo Envío → Registrar | `createShipment` | Nueva fila A–P, `Estado = Registrado`, código consecutivo en G, foto en L | Al destinatario |
| 2 | Archivo / conductor (`ADMIN_EMAILS`) | 📥 Recibido | `updateStatus` | H = Recibido, I = fecha/hora, P = GPS | — |
| 3 | Conductor | 🚚 En Camino | `updateStatus` | H = En Camino, P = GPS. **Si se saltó Recibido, I = misma fecha/hora** | Al destinatario |
| 4 | Conductor | ✅ Entregado | `updateStatus` | H = Entregado, J = fecha/hora, K = duración (I→J, o A→J si I vacío), P = GPS | Al emisor y al destinatario |

### 1.2 Responsabilidades

- **Frontend**: autenticación visual, control de pestañas por rol (`ADMIN_EMAILS`), selección del destinatario desde el Directorio, filtrado "Mis Envíos" (emisor o destinatario = usuario), cámara y compresión de imagen, captura GPS (máx. 8 s, si falla envía `gps: null`).
- **Backend**: validación, generación de código, escritura atómica (ScriptLock), almacenamiento de fotos, reglas de transición de estado, cálculo de duración, normalización de departamentos, envío de correos, sincronización de contactos (trigger diario).

### 1.3 Interfaz (`index.html`)

**Identidad visual**
- Paleta única `brand` (rojos corporativos) definida en `tailwind.config`: `50 #fef2f2 · 100 #fee2e2 · 200 #fecaca · 300 #fca5a5 · 400 #f87171 · 500 #ef4444 · 600 #dc2626 · 700 #b91c1c · 800 #991b1b · 900 #7f1d1d`. Los neutros usan `slate`. No se usan otros colores (los estados se distinguen por intensidad de rojo e iconos).
- Estados: `Registrado` gris · `Recibido` brand-50/700 · `En Camino` brand-100/800 · `Entregado` sólido brand-800 con texto blanco.
- Logo `./logo-provivienda.png` en la cabecera (recuadro blanco) y en la pantalla de inicio de sesión. Si no carga, `logoFallback()` (definida en `<head>`) prueba las rutas de `data-alts` (`./logo-provivienda.png.jpg`) y, si tampoco existen, oculta la imagen y muestra el texto "Grupo Provivienda" / "GP" en rojo.

**Nuevo Envío**
1. Tipo: `Interna (Grupo Provivienda)` o `Externa` (esta última muestra Nombre del proveedor* y Correo del proveedor).
2. Destinatario: chips de "Destinatarios recientes" (4, por usuario) y **un único campo editable "Nombre del destinatario"** con autocompletado en tiempo real sobre el Directorio (busca por nombre, correo o departamento; flechas/Enter/Esc). Al elegir a la persona se muestran como **texto de solo lectura** el **Correo** y el **Departamento / Proyecto** (en blanco si no tiene). Si se edita el nombre, la selección se descarta. Si el texto coincide exactamente con un único nombre del Directorio se selecciona solo. No se puede registrar sin una persona del Directorio (el backend exige `correoDestinatario`). El mismo bloque aplica a envíos Externos (persona de la empresa asociada).
3. Descripción con 5 chips de selección rápida (toque = agrega; segundo toque = quita). `DESC_CHIPS` contiene el **Top 5 de `historico.csv`** (541 registros, clasificados por palabras clave): Planos / Copias de planos (265) · Permisos y aprobaciones (141) · Contrato / Adenda (118) · Memorial para firma (112) · Documentos legales (cédula, finca) (83). Si se vacía, se usa `DESC_CHIPS_GENERALES`: Contrato / Escritura · Cheque / Factura · Documentos firmados · Expediente de cliente · Sobre confidencial. El análisis es estático (el CSV **no** se publica ni se lee en el navegador).
4. Foto: botón **"Abrir Cámara"** → modal con `navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } })`, vista previa en vivo, **"Tomar Foto"**, cambio de cámara si hay más de una y respaldo `{ video: true }` para webcams de PC. Si el navegador no soporta `getUserMedia`, abre el selector nativo (`capture="environment"`). Botón secundario **"Subir archivo"**. Toda imagen (cámara o archivo) se redimensiona en `<canvas>` a máx. 800 px de ancho, JPEG 0.75, y se envía en Base64. La cámara se libera al cerrar el modal o al ocultar la pestaña. Requiere HTTPS (GitHub Pages) o `localhost`.
5. Confirmación con el código en grande (`#10000`), copiar código, ver en Mis Envíos y registrar otro.

**Panel Operador / Conductor** (solo `ADMIN_EMAILS`): estadísticas, búsqueda (acepta `#10000`), filtros Activos/Entregados/Todos, tarjetas con foto ampliable y tres botones de un toque `Recibido → En Camino → Entregado` (el siguiente paso resaltado, los cumplidos deshabilitados; Entregado pide confirmación). Solo existe el botón **"Actualizar lista"**: la sincronización de contactos es automática (§8).

**Mis Envíos**: secciones "Enviados por mí" / "Para mí", búsqueda (acepta `#10000`), línea de tiempo, duración, foto y enlace a Google Maps.

**Formato de referencias**: siempre `#` + número (`fmtRef()`), en tarjetas, confirmación, avisos y búsquedas. El JSON del backend sigue enviando `referencia` sin `#`.

**Departamentos en pantalla**: `normDepto()` replica la regla de `Code.gs` (§5.4) y además se deduplica el Directorio por correo, de modo que `Transformación Digital` y `transformación digital` nunca aparecen como valores distintos (también se normaliza la col. E al mostrar los envíos).

---

## 2. Archivos del repositorio

| Archivo | Rol | ¿Se publica en GitHub Pages? |
|---|---|---|
| `index.html` | Aplicación web completa (UI + lógica cliente) | Sí |
| `Code.gs` | Backend de Apps Script (se copia al editor de Apps Script) | Opcional (no se ejecuta en Pages) |
| `PRD.md.md` | Requisitos de producto originales | Opcional |
| `ARCHITECTURE_AND_MIGRATION.md` | Este documento | Opcional |
| `historico.csv.csv` | Exportación histórica de envíos (ver §12) | **No recomendado** (contiene datos personales) |
| `logo-provivienda.png.jpg` | Logo (JPEG 200×200). La página busca primero `logo-provivienda.png` y luego este nombre. **Recomendado: renombrarlo a `logo-provivienda.png`** | Sí |

---

## 3. Esquema de Google Sheets

> Los nombres de pestañas y encabezados deben coincidir **exactamente** (incluidos acentos). El orden A–O es heredado de los reportes históricos; P (`GPS`) se agregó al final.

### 3.1 Pestaña `Envios` (16 columnas)

| Col | Encabezado exacto | Tipo almacenado | Escrito por | Cuándo / regla |
|---|---|---|---|---|
| A | `Marca temporal` | Fecha (Date) | `createShipment` | `new Date()` al registrar |
| B | `Dirección de correo electrónico` | Texto (minúsculas) | `createShipment` | Correo del emisor (`userEmail`) |
| C | `Correo del destinatario` | Texto (minúsculas) | `createShipment` | Obligatorio, validado como email. En la UI proviene siempre de la persona elegida en el Directorio (solo lectura) |
| D | `Nombre del destinatario` | Texto ≤150 | `createShipment` | Obligatorio |
| E | `Departamento / Proyecto` | Texto ≤150 | `createShipment` | Opcional. La UI envía el departamento normalizado del Directorio de la persona elegida (vacío si no tiene); no es editable |
| F | `Descripción de la Correspondencia` | Texto ≤2000 | `createShipment` | Obligatorio |
| G | `Referencia` | **Número** | `createShipment` | Consecutivo 10000–99999, único (ver §5.1). La UI lo muestra como `#10000` |
| H | `Estado` | Texto | ambos | `Registrado` → `Recibido` → `En Camino` → `Entregado` |
| I | `Recibido` | Fecha | `updateStatus` | Al marcar Recibido, **o** autocompletado al saltar de Registrado a En Camino |
| J | `Entregado` | Fecha | `updateStatus` | Al marcar Entregado |
| K | `Duración` | Texto | `updateStatus` | `formatearDuracion_(J − I)` o `(J − A)` si I vacío. Formatos: `<1m`, `25m`, `1h 25m`, `2d 0h 5m` |
| L | `Sube una foto de tu correspondencia` | Texto (URL) | `createShipment` | `https://drive.google.com/thumbnail?id=<FILE_ID>&sz=w600` o vacío |
| M | `Tipo de Mensajería` | Texto | `createShipment` | Exactamente `Interna (Grupo Provivienda)` o `Externa` |
| N | `Nombre del proveedor` | Texto | `createShipment` | Solo si M = `Externa` (obligatorio en ese caso) |
| O | `Correo del proveedor` | Texto | `createShipment` | Solo si M = `Externa` (opcional, validado si viene) |
| P | `GPS` | Texto | `updateStatus` | Último cambio de estado: `lat,lng \| https://www.google.com/maps?q=lat,lng` (6 decimales) |

Reglas adicionales:
- Los textos que empiezan con `= + - @` se guardan con apóstrofo inicial (protección contra inyección de fórmulas).
- Al leer (`doGet`), cualquier URL de Drive en L (`open?id=`, `/file/d/`, `uc?id=`, `thumbnail?id=`) se convierte a `thumbnail?id=…&sz=w600`. Si hay varias separadas por comas se usa la primera.
- Fechas históricas en texto `dd/MM/yyyy HH:mm[:ss]` se interpretan correctamente para calcular duración.

### 3.2 Pestaña `Directorio` (3 columnas)

| Col | Encabezado | Origen | Regla |
|---|---|---|---|
| A | `Nombre` | People API (`names.displayName`) | Obligatorio |
| B | `Correo` | People API (`emailAddresses` principal) | Obligatorio, minúsculas, único |
| C | `Departamento` | People API (`organizations.department` → `name` → `title`) | **Normalizado** (§5.4); vacío `""` si no existe |

- `sincronizarContactos()` borra las filas 2..N (conserva encabezados) y reescribe todo, ordenado por nombre.
- Fuentes: "Mis contactos" de la cuenta ejecutora y, si `INCLUDE_DOMAIN_DIRECTORY = true`, el directorio del dominio de Google Workspace.

---

## 4. Contrato de la API

**URL base**: la URL `/exec` de la implementación (constante `APPS_SCRIPT_URL` en `index.html`).
**Formato**: todas las respuestas son JSON con `ok: boolean`. En error: `{ "ok": false, "code": <número>, "error": "<mensaje legible>" }`.

### 4.1 Reglas CORS (obligatorias)

Apps Script no permite cabeceras CORS personalizadas ni responde a `OPTIONS`. Las respuestas de `ContentService` ya incluyen `Access-Control-Allow-Origin: *` tras la redirección 302 a `script.googleusercontent.com`. Por eso:

```js
// GET: sin cabeceras. Agregar un parámetro anti-caché en vez de cabeceras Cache-Control.
fetch(APPS_SCRIPT_URL + '?_t=' + Date.now(), { method: 'GET', redirect: 'follow' });

// POST: "petición simple" → Content-Type text/plain, cuerpo JSON en texto.
fetch(APPS_SCRIPT_URL, {
  method: 'POST',
  redirect: 'follow',
  headers: { 'Content-Type': 'text/plain;charset=utf-8' },
  body: JSON.stringify(payload)
});
```

No usar `application/json`, `Authorization`, `cache: 'no-cache'` ni ninguna cabecera no "safelisted": provocan un preflight que Apps Script no responde.
`doGet` admite además `?callback=nombreFuncion` (JSONP) como respaldo. `doPost` acepta también `application/x-www-form-urlencoded` con un campo `payload` que contenga el JSON.

### 4.2 `GET /exec` → `doGet`

Respuesta:

```json
{
  "ok": true,
  "directorio": [
    { "nombre": "María Fernández", "correo": "mfernandez@gprovivienda.com", "departamento": "Transformación Digital" }
  ],
  "envios": [
    {
      "fila": 12,
      "marcaTemporal": "2026-11-18T09:30:00-05:00",
      "emisor": "usuario@gprovivienda.com",
      "correoDestinatario": "mfernandez@gprovivienda.com",
      "nombreDestinatario": "María Fernández",
      "departamento": "Contabilidad",
      "descripcion": "Contrato, Sobre cerrado",
      "referencia": "10000",
      "estado": "En Camino",
      "recibido": "2026-11-18T10:05:00-05:00",
      "entregado": "",
      "duracion": "",
      "foto": "https://drive.google.com/thumbnail?id=FILE_ID&sz=w600",
      "tipoMensajeria": "Interna (Grupo Provivienda)",
      "nombreProveedor": "",
      "correoProveedor": "",
      "gps": "8.982412,-79.519876 | https://www.google.com/maps?q=8.982412,-79.519876",
      "gpsLat": 8.982412,
      "gpsLng": -79.519876,
      "mapsUrl": "https://www.google.com/maps?q=8.982412,-79.519876"
    }
  ],
  "generado": "2026-11-18T10:06:12-05:00"
}
```

- `envios` va del más reciente al más antiguo (orden inverso de filas). Filas totalmente vacías se omiten.
- Fechas en ISO 8601 con zona `-05:00`; si la celda tiene texto (histórico), se devuelve el texto tal cual.
- `referencia` siempre es **string** (sin `#`). `estado` se normaliza a uno de los 4 valores; si la celda tiene otro texto se devuelve ese texto.
- `directorio`: solo filas con nombre y correo; correos duplicados se descartan; `departamento` normalizado en lectura (la hoja no se modifica).

### 4.3 `POST /exec` → `doPost`

Cuerpo JSON con `action`. Respuesta según acción.

#### `createShipment`

Petición:

```json
{
  "action": "createShipment",
  "userEmail": "usuario@gprovivienda.com",
  "tipoMensajeria": "Interna (Grupo Provivienda)",
  "correoDestinatario": "mfernandez@gprovivienda.com",
  "nombreDestinatario": "María Fernández",
  "departamento": "Contabilidad",
  "descripcion": "Contrato",
  "nombreProveedor": "",
  "correoProveedor": "",
  "fotoBase64": "data:image/jpeg;base64,/9j/4AAQ...",
  "fotoMimeType": "image/jpeg"
}
```

| Campo | Obligatorio | Validación |
|---|---|---|
| `userEmail` (alias `emisor`) | Sí | Email válido |
| `tipoMensajeria` | No | Cualquier texto que contenga "extern" → `Externa`; si no → `Interna (Grupo Provivienda)` |
| `correoDestinatario` | Sí | Email válido |
| `nombreDestinatario` | Sí | No vacío, ≤150 |
| `departamento` | No | ≤150 |
| `descripcion` | Sí | No vacío, ≤2000 |
| `nombreProveedor` | Si `Externa` | ≤150 |
| `correoProveedor` | No | Email válido si viene |
| `fotoBase64` | No | Data URL o Base64 puro de imagen; ≤5 MB decodificada (`MAX_FOTO_BYTES`) |
| `fotoMimeType` | No | Por defecto `image/jpeg`; debe ser `image/*` |

Respuesta:

```json
{
  "ok": true,
  "referencia": "10000",
  "fila": 12,
  "estado": "Registrado",
  "marcaTemporal": "2026-11-18T09:30:00-05:00",
  "fotoUrl": "https://drive.google.com/thumbnail?id=FILE_ID&sz=w600",
  "correo": { "enviado": true, "destinatarios": ["mfernandez@gprovivienda.com"], "remitente": "kawak@gprovivienda.com", "modoRemitente": "cuenta" }
}
```

#### `updateStatus`

Petición:

```json
{
  "action": "updateStatus",
  "userEmail": "conductor@gprovivienda.com",
  "referencia": "10000",
  "estado": "En Camino",
  "gps": { "lat": 8.982412, "lng": -79.519876 }
}
```

- `referencia`: 5 dígitos (se aceptan `"#10000"` o número).
- `estado`: `Recibido` | `En Camino` | `Entregado` (insensible a mayúsculas/acentos, acepta `en_camino`).
- `gps`: `{lat,lng}`, `{latitude,longitude}`, `"lat,lng"` o `null`. Coordenadas inválidas o `0,0` se ignoran (no bloquean).

Respuesta (cambio aplicado):

```json
{
  "ok": true,
  "referencia": "10000",
  "estado": "En Camino",
  "fecha": "2026-11-18T10:05:00-05:00",
  "recibidoAutocompletado": true,
  "duracion": "",
  "gps": { "lat": 8.982412, "lng": -79.519876, "mapsUrl": "https://www.google.com/maps?q=8.982412,-79.519876" },
  "correo": { "enviado": true, "destinatarios": ["mfernandez@gprovivienda.com"] },
  "envio": { "...": "mismo formato que un elemento de envios[] en doGet" }
}
```

Respuesta idempotente (el envío ya estaba en ese estado; no se modifica nada):

```json
{ "ok": true, "unchanged": true, "referencia": "10000", "estado": "En Camino", "mensaje": "El envío ya se encontraba en estado \"En Camino\".", "envio": { } }
```

#### `syncContacts`

Petición: `{ "action": "syncContacts", "userEmail": "admin@gprovivienda.com" }`
Respuesta: `{ "ok": true, "importados": 152 }`

> La sincronización es **automática y diaria** (§8). El frontend **ya no** tiene botón ni llama a esta acción; el endpoint se conserva en el backend por compatibilidad (scripts o pruebas manuales).

### 4.4 Códigos de error

| `code` | Significado | Ejemplos |
|---|---|---|
| 400 | Petición inválida | JSON mal formado, acción desconocida, email inválido, referencia no es de 5 dígitos, imagen no válida |
| 403 | No autorizado | `userEmail` fuera de `ADMIN_EMAILS` del backend (solo si esa lista no está vacía) |
| 404 | No encontrado | Referencia inexistente en la columna G |
| 409 | Conflicto de estado | Envío ya `Entregado`, o intento de retroceder de estado |
| 413 | Imagen muy grande | > `MAX_FOTO_BYTES` |
| 500 | Error interno / configuración | Falta `FOLDER_ID_FOTOS`, People API no habilitada, códigos agotados |
| 503 | Ocupado | No se obtuvo el ScriptLock en 30 s |

---

## 5. Reglas de negocio

### 5.1 Códigos de referencia consecutivos

- Rango: `REFERENCIA_INICIAL = 10000` … `REFERENCIA_MAXIMA = 99999`. Al agotarse → error 500.
- Todo ocurre dentro de `LockService.getScriptLock()` (máx. 30 s de espera): no hay duplicados aunque dos personas registren a la vez.
- Fuente del último número: **Propiedad del script** `ULTIMA_REFERENCIA` (lectura O(1), no recorre la hoja).
- Si la propiedad no existe (primer uso o proyecto copiado a otra cuenta): se lee **solo la celda G del último registro**. Si es un número entre 10000 y 99999 se continúa desde ahí; si no, se empieza en 10000.
- Salvaguarda anti-colisión: antes de usar un candidato se busca en la columna G con `TextFinder` (búsqueda nativa de Sheets, coincidencia exacta de celda, también `#N`). Si existe —p. ej. un código histórico importado— se prueba el siguiente (máx. 500 intentos).
- La propiedad se actualiza **después** de escribir la fila. Si la escritura falla, no se consume el número.
- Utilidad manual: `reiniciarSecuenciaReferencias()` → el próximo será `#10000`; `reiniciarSecuenciaReferencias(12345)` → el próximo será `#12346`.

### 5.2 Máquina de estados

```
Registrado ──▶ Recibido ──▶ En Camino ──▶ Entregado
     │                           ▲              ▲
     └──────── (salto) ──────────┘              │
     └──────────────────── (salto) ─────────────┘
```

- Rango: `Registrado=0, Recibido=1, En Camino=2, Entregado=3`. Estados desconocidos/vacíos (histórico) cuentan como 0.
- Se permite avanzar a cualquier estado de rango mayor (saltos permitidos). Retroceder → 409.
- Mismo estado → respuesta `unchanged: true` (doble toque seguro).
- `Entregado` es terminal → cualquier cambio posterior → 409.
- **Autocompletado de Recibido**: si el estado actual es `Registrado`, el nuevo es `En Camino` y la columna I está vacía, I recibe la misma marca de tiempo que el cambio a En Camino. La respuesta incluye `recibidoAutocompletado: true`.
- El salto directo `Registrado → Entregado` no autocompleta I; la duración se calcula desde A (`Marca temporal`).
- La fecha de "En Camino" no tiene columna propia (el esquema es fijo); solo queda el GPS del último cambio en P.

### 5.3 Duración (col. K)

`inicio = Recibido (I) || Marca temporal (A)`; `K = J − inicio` formateado como `<1m`, `Xm`, `Xh Ym` o `Xd Yh Zm`.

### 5.4 Normalización de departamentos (solo lectura)

Se aplica en `sincronizarContactos()` (al escribir el Directorio) y al devolver `directorio` en `doGet` (sin modificar la hoja). **No existe endpoint para editar departamentos.**

1. Recorta y colapsa espacios múltiples.
2. Pasa todo a minúsculas y aplica Title Case por palabra (separadores: espacio, `/`, `-`, `(`, `)`, `,`, `&`).
3. Conectores en minúscula salvo al inicio: `de, del, la, las, el, los, y, e, o, u, a, al, en, por, para, con`.
4. Siglas en mayúsculas según `ACRONIMOS_DEPARTAMENTO` (`TI`, `RRHH`, `QA`, `BIM`, `SAP`, …).
5. Sin departamento → `""`.

Ejemplos: `"  TRANSFORMACIÓN   digital "` → `"Transformación Digital"` · `"gerencia DE proyectos"` → `"Gerencia de Proyectos"` · `"rrhh"` → `"RRHH"`.
El resultado depende solo del texto en minúsculas, por lo que variantes de mayúsculas producen el mismo valor (se eliminan duplicados).
La columna E de `Envios` (texto libre del formulario) **no** se normaliza.

### 5.5 Correos

| Evento | Destinatarios | Asunto |
|---|---|---|
| `createShipment` | Destinatario (C) | `Nueva correspondencia registrada · Ref. #10000` |
| `updateStatus` → En Camino | Destinatario (C) | `Su correspondencia va en camino · Ref. #10000` |
| `updateStatus` → Entregado | Emisor (B) + Destinatario (C) | `Correspondencia entregada · Ref. #10000` |
| Recibido | — | (no envía) |

- Envío con **GmailApp**, HTML con tablas y estilos en línea (compatible con Outlook), incluye código, línea de tiempo, foto y botón a Google Maps si hay GPS; botón "Rastrear envío" si `FRONTEND_URL` está definido.
- Remitente (resuelto una vez por ejecución en `resolverRemitente_()`):
  - `cuenta`: la cuenta que ejecuta el script **es** `SENDER_ALIAS` → se envía normalmente.
  - `alias`: `SENDER_ALIAS` aparece en `GmailApp.getAliases()` → se usa la opción `from`.
  - `fallback`: no está disponible → sale desde la cuenta ejecutora con `replyTo = SENDER_ALIAS` y se registra una advertencia.
  - En todos los casos el nombre visible es `SENDER_NAME = "notificador Provivienda"`.
  - Para cambiar la cuenta o el alias remitente, ver §11.
- Un fallo de correo **nunca** revierte el registro o el cambio de estado; se informa en `correo.enviado = false` y `correo.error`/`motivo`.
- La cuota diaria se verifica con `MailApp.getRemainingDailyQuota()` (aplica también a GmailApp).

### 5.6 Fotos

- El navegador redimensiona a máx. 800 px de ancho, JPEG calidad 0.75, y envía Data URL Base64 (~100–200 KB).
- El backend la guarda en `FOLDER_ID_FOTOS`, la renombra `REF-<código>_<yyyyMMdd_HHmmss>.jpg` y la comparte como **"Cualquier usuario con el enlace: lector"**. Si la política del dominio lo impide, usa "dominio con enlace"; si tampoco, queda privada (la miniatura no cargará para otros).
- Si la escritura en la hoja falla, la foto se envía a la papelera.

---

## 6. Constantes de configuración

### 6.1 `Code.gs`

| Constante | Valor actual / ejemplo | Descripción |
|---|---|---|
| `FOLDER_ID_FOTOS` | `<ID de carpeta de Drive>` | Carpeta de fotos. **Cambia al migrar.** |
| `DRIVE_FOLDER_ID` | `= FOLDER_ID_FOTOS` | Alias con el nombre del PRD |
| `SHEET_ID` | `''` | Vacío si el script está vinculado a la hoja; si es independiente, ID de la hoja |
| `SHEET_ENVIOS` / `SHEET_DIRECTORIO` | `Envios` / `Directorio` | Nombres exactos de pestañas |
| `TIMEZONE` | `America/Panama` | Fechas en JSON/correos y zona del trigger |
| `FOTO_THUMB_SIZE` | `w600` | Ancho de miniaturas de Drive |
| `MAX_FOTO_BYTES` | `5 MB` | Tamaño máximo de foto decodificada |
| `FRONTEND_URL` | `''` (recomendado: URL de GitHub Pages) | Agrega botón "Rastrear envío" en correos |
| `SEND_EMAILS` | `true` | Interruptor global de correos |
| `SENDER_ALIAS` | `kawak@gprovivienda.com` | Remitente preferido |
| `SENDER_NAME` | `notificador Provivienda` | Nombre visible del remitente |
| `REFERENCIA_INICIAL` / `REFERENCIA_MAXIMA` | `10000` / `99999` | Rango de códigos |
| `PROP_ULTIMA_REFERENCIA` | `ULTIMA_REFERENCIA` | Clave en Propiedades del script |
| `HORA_SYNC_DIARIA` | `6` | Hora del trigger diario |
| `ACRONIMOS_DEPARTAMENTO` | `['TI','IT','RRHH',…]` | Siglas que se respetan en mayúsculas |
| `ADMIN_EMAILS` | `[]` | Si tiene correos, solo ellos pueden `updateStatus`/`syncContacts` (valida `userEmail`). Vacío = sin control en backend |
| `INCLUDE_DOMAIN_DIRECTORY` | `true` | Importa también el directorio del dominio Workspace |

**Propiedades del script** (Configuración del proyecto → Propiedades del script):

| Clave | Contenido |
|---|---|
| `ULTIMA_REFERENCIA` | Último código emitido (p. ej. `10457`). Gestionada automáticamente. |

### 6.2 `index.html`

| Constante | Descripción |
|---|---|
| `APPS_SCRIPT_URL` | URL `/exec` de la Web App. **Cambia al reimplementar desde otra cuenta.** |
| `GOOGLE_CLIENT_ID` | ID de cliente OAuth "Aplicación web" (termina en `.apps.googleusercontent.com`) |
| `ADMIN_EMAILS` | Correos (minúsculas) que ven el Panel Operador / Conductor |
| `ALLOWED_DOMAINS` | Dominios permitidos para iniciar sesión (p. ej. `['gprovivienda.com']`) |
| `TZ`, `FOTO_MAX_W` (800), `FOTO_QUALITY` (0.75), `GPS_TIMEOUT_MS` (8000), `MAX_RECIENTES` (4), `PAGE_SIZE` (60), `AUTO_REFRESH_MS` (120000) | Ajustes de UI |
| `DESC_CHIPS` | Top 5 de descripciones más enviadas (derivado de `historico.csv`, §1.3) |
| `DESC_CHIPS_GENERALES` | Respaldo si `DESC_CHIPS` está vacío |
| `DEPTO_MINUSCULAS`, `DEPTO_ACRONIMOS` | Regla de normalización de departamentos; mantener igual que `ACRONIMOS_DEPARTAMENTO` de `Code.gs` |
| `TIPOS`, `ESTADOS` | Deben coincidir con el backend (no modificar) |
| `tailwind.config.theme.extend.colors.brand` | Paleta corporativa roja (`#fef2f2` … `#7f1d1d`) |

Claves de `localStorage`: `corr_session_v1` (perfil `{email,name,picture,hd,loginAt}`) y `corr_recientes_v1:<email>` (últimos 4 destinatarios).

---

## 7. Servicios, permisos y cuotas

| Servicio | Uso | Activación |
|---|---|---|
| SpreadsheetApp | Leer/escribir `Envios` y `Directorio` | Automático |
| DriveApp | Crear y compartir fotos | Automático |
| GmailApp | Enviar correos con alias/nombre | Automático (permiso completo de Gmail) |
| MailApp | Solo `getRemainingDailyQuota()` | Automático |
| People API | `Connections.list`, `listDirectoryPeople` | **Manual**: Editor → Servicios (+) → People API (identificador `People`) |
| LockService / PropertiesService | Exclusión mutua y secuencia de códigos | Automático |
| ScriptApp | Triggers | Automático |
| Session | Detectar la cuenta ejecutora (remitente) | Automático |

Al ejecutar `setup()` Apps Script pide los permisos correspondientes. **Cada vez que se agrega un servicio nuevo (p. ej. GmailApp), hay que volver a ejecutar `setup()`, aceptar los permisos y publicar una nueva versión de la implementación.**

Cuotas relevantes (consultar la página oficial "Quotas for Google Services", varían por tipo de cuenta): destinatarios de correo por día (≈1 500 en Workspace), tiempo máximo por ejecución (6 min), tiempo total de triggers por día, ejecuciones simultáneas (~30).

---

## 8. Triggers

| Función | Tipo | Frecuencia | Creación |
|---|---|---|---|
| `sincronizarContactos` | Basado en tiempo | Diario, ~06:00 `America/Panama` | Ejecutar `configurarTriggerDiario()` una vez |

- `configurarTriggerDiario()` no crea duplicados: si ya existe uno lo conserva y borra los sobrantes.
- `eliminarTriggerDiario()` elimina todos los de `sincronizarContactos`.
- Los triggers pertenecen al usuario que los crea y **no se copian** al duplicar la hoja/proyecto: hay que recrearlos en la nueva cuenta.
- `setup()` informa si falta el trigger.

---

## 9. Google Cloud · OAuth (inicio de sesión del frontend)

El `GOOGLE_CLIENT_ID` solo sirve para el botón "Iniciar sesión con Google" del frontend. **Es independiente de la cuenta que ejecuta Apps Script.**

1. [Google Cloud Console](https://console.cloud.google.com/) → crear/seleccionar proyecto (de preferencia propiedad de una cuenta institucional, no personal).
2. **APIs y servicios → Pantalla de consentimiento de OAuth**:
   - Tipo de usuario: **Interno** (solo cuentas del dominio Workspace) — recomendado.
   - Nombre de la app, correo de soporte, dominio autorizado `github.io` si se solicita.
   - Ámbitos: solo `openid`, `email`, `profile` (no se requieren otros).
3. **Credenciales → Crear credenciales → ID de cliente de OAuth → Aplicación web**:
   - **Orígenes autorizados de JavaScript**: `https://practicantetd-gprovivienda.github.io` (sin ruta ni barra final). Para pruebas locales opcional: `http://localhost:5500`.
   - **URI de redireccionamiento**: no se necesitan (flujo popup de GIS).
4. Copiar el ID de cliente en `GOOGLE_CLIENT_ID` de `index.html`.
5. Los cambios de orígenes pueden tardar unos minutos en propagarse.

> Nota de seguridad: el frontend decodifica el JWT sin verificar su firma; la restricción por dominio y por `ADMIN_EMAILS` es **visual**. El backend confía en `userEmail`. Ver §13.

---

## 10. Checklist de migración a otra cuenta (p. ej. `kawak@gprovivienda.com`)

> Objetivo: que la app quede ejecutándose bajo `kawak@gprovivienda.com` (dueño de la hoja, la carpeta, el script, los triggers y remitente de los correos), sin perder datos ni la secuencia de códigos.
>
> Si solo necesita cambiar **desde qué dirección salen los correos**, vea primero §11 (la Opción 1 por alias no requiere migrar nada). Esta checklist detallada corresponde a la Opción 2 de §11 e incluye además la variante de copiar la hoja.

### 10.1 Antes de empezar

- [ ] Anotar el **último código emitido**: Apps Script (cuenta actual) → Configuración del proyecto → Propiedades del script → `ULTIMA_REFERENCIA` (o el mayor código de la columna G).
- [ ] Anotar `FOLDER_ID_FOTOS`, URL `/exec` actual y `GOOGLE_CLIENT_ID`.
- [ ] Avisar a los usuarios de una ventana de mantenimiento (unos minutos).

### 10.2 Datos (Sheets y Drive)

- [ ] **Opción A — transferir propiedad (recomendada, conserva IDs y fotos):** compartir la hoja y la carpeta de fotos con `kawak@` como Editor → Compartir → "Transferir propiedad". En Workspace puede requerir que el administrador lo haga desde la consola de administración.
- [ ] **Opción B — copiar:** desde `kawak@` abrir la hoja → Archivo → Hacer una copia (copia también el script vinculado, pero **no** las propiedades del script ni los triggers). Crear carpeta nueva de fotos y mover/copiar las fotos existentes (las URLs de la columna L apuntan a los IDs originales: si se copian archivos, cambian los IDs; preferir mover o transferir propiedad).
- [ ] Verificar que la carpeta de fotos permita "Cualquier usuario con el enlace" (política de compartir externo del dominio).

### 10.3 Script

- [ ] Abrir la hoja como `kawak@` → Extensiones → Apps Script (si es copia, ya contiene `Code.gs`; si no, pegarlo).
- [ ] Servicios (+) → **People API** (`People`).
- [ ] Configuración del proyecto → Zona horaria: `America/Panama`.
- [ ] Revisar constantes: `FOLDER_ID_FOTOS` (si cambió), `SHEET_ID` (`''` si es vinculado), `SENDER_ALIAS`, `SENDER_NAME`, `FRONTEND_URL`, `ADMIN_EMAILS`.
- [ ] Ejecutar `setup()` y aceptar todos los permisos. Revisar el registro:
  - `Remitente: kawak@gprovivienda.com · modo=cuenta` ✅
  - Carpeta de fotos y People API OK.
- [ ] Secuencia de códigos: si se usó la Opción B (propiedades no se copian), ejecutar `reiniciarSecuenciaReferencias(<último código anotado>)`. (Si no se hace, el sistema la reconstruye desde el último registro, que también es correcto si la última fila es un código ≥ 10000.)
- [ ] Ejecutar `configurarTriggerDiario()` → `✅ Activador creado`.
- [ ] (Opcional) Ejecutar `sincronizarContactos()` para poblar el Directorio inmediatamente.
- [ ] Ejecutar `testDoGet()` y revisar conteos.

### 10.4 Implementación

- [ ] Implementar → Nueva implementación → Aplicación web → **Ejecutar como: Yo (kawak@)** → **Acceso: Cualquier usuario** → Implementar.
- [ ] Copiar la nueva URL `/exec`.
- [ ] En la cuenta anterior: archivar/eliminar su implementación y ejecutar `eliminarTriggerDiario()` para evitar dos sincronizaciones y correos duplicados.

### 10.5 Frontend

- [ ] `index.html` → actualizar `APPS_SCRIPT_URL` (y `GOOGLE_CLIENT_ID` si se creó un nuevo cliente OAuth; si se reutiliza, no cambia).
- [ ] Commit y push a la rama de GitHub Pages; esperar el despliegue (1–2 min).
- [ ] Si el cliente OAuth es nuevo, registrar el origen `https://practicantetd-gprovivienda.github.io`.

### 10.6 Pruebas de aceptación

- [ ] Iniciar sesión con una cuenta `@gprovivienda.com` (y comprobar que otra cuenta es rechazada).
- [ ] Logo visible en el inicio de sesión y en la cabecera.
- [ ] Escribir parte de un nombre en "Nombre del destinatario" → elegirlo → correo y departamento aparecen como solo lectura.
- [ ] "Abrir Cámara" en PC (webcam) y en celular (cámara trasera) → "Tomar Foto" → vista previa ≤ 800 px.
- [ ] Registrar un envío con foto → código consecutivo correcto (`#10000`…), foto visible, correo recibido desde **notificador Provivienda <kawak@gprovivienda.com>**.
- [ ] Panel: Registrado → En Camino directamente → columna I autocompletada, correo "en camino".
- [ ] Entregado → J y K llenos, correo a emisor y destinatario, GPS en P.
- [ ] Mis Envíos muestra línea de tiempo, duración y enlace a Maps.
- [ ] Al día siguiente: Ejecuciones (Apps Script) muestra `sincronizarContactos` exitoso a las ~06:00.

### 10.7 Reversión

Volver a poner la URL `/exec` anterior en `index.html` (si la implementación antigua no se eliminó) y recrear su trigger con `configurarTriggerDiario()`.

---

## 11. Cómo cambiar la cuenta emisora de correos

### 11.0 Cómo se decide hoy el remitente

Los correos los envía **la cuenta que ejecuta la Web App**: la que creó la implementación con *Ejecutar como: Yo*. Esa cuenta no siempre es la que edita el código. Al enviar, `resolverRemitente_()` aplica estas reglas:

| Situación | Modo (`setup()`) | Dirección "De:" que ve el destinatario |
|---|---|---|
| La cuenta ejecutora **es** `SENDER_ALIAS` | `cuenta` | `SENDER_NAME <SENDER_ALIAS>` |
| `SENDER_ALIAS` es un alias **verificado** de la cuenta ejecutora | `alias` | `SENDER_NAME <SENDER_ALIAS>` |
| Ninguna de las anteriores | `fallback` | `SENDER_NAME <cuenta ejecutora>`, con *Responder a* = `SENDER_ALIAS` |

Elija el procedimiento:

| | Opción 1 · Alias de Gmail | Opción 2 · Migración total |
|---|---|---|
| ¿Cambia la cuenta que ejecuta el script? | No | Sí |
| ¿Cambia la URL `/exec`? | **No** | **Sí** (hay que actualizar `index.html`) |
| ¿Quién es dueño de la hoja, la carpeta y los triggers? | La cuenta actual | La cuenta nueva |
| Cuota diaria de correos | La de la cuenta actual | La de la cuenta nueva |
| Esfuerzo | ~10 minutos | ~30 minutos + prueba completa |
| Recomendado cuando… | Solo se quiere otra dirección visible | La cuenta actual es personal, temporal o va a desactivarse |

### 11.1 Opción 1 · Cambio por alias de Gmail (`SENDER_ALIAS`)

**Requisitos:** acceso a la cuenta ejecutora actual y alguien con acceso al buzón del nuevo correo para confirmar la verificación.

1. **Iniciar sesión en Gmail con la cuenta ejecutora** (la que creó la implementación), no con el nuevo correo.
2. Ir a ⚙ **Configuración → Ver todos los ajustes → Cuentas** (en algunas cuentas se llama *Cuentas e importación*).
3. En **"Enviar mensaje como"** pulsar **"Añadir otra dirección de correo electrónico"**.
4. Completar:
   - **Nombre:** el que verán los destinatarios (p. ej. `notificador Provivienda`).
   - **Dirección de correo electrónico:** el nuevo remitente (p. ej. `kawak@gprovivienda.com`).
   - Dejar marcado **"Tratar como un alias"** → **Siguiente paso**.
5. **Verificar la dirección:** Gmail envía un enlace o código al nuevo correo, y quien administre ese buzón debe confirmarlo.
   - Si es una dirección de **otro dominio**, Gmail pedirá datos de un servidor SMTP. Los ingresa el titular de esa cuenta; nunca van en el código.
   - En Google Workspace, el administrador puede tener restringido el envío como otras direcciones. En ese caso debe habilitarlo desde la consola de administración.
6. Confirmar que la dirección aparece en **"Enviar mensaje como"** sin la marca *"no verificada"*.
7. En el editor de Apps Script, abrir `Code.gs` y editar:
   ```js
   const SENDER_ALIAS = 'nuevo.correo@gprovivienda.com';
   const SENDER_NAME  = 'notificador Provivienda';
   ```
   Guardar (Ctrl+S).
8. Ejecutar **`setup()`** y revisar el *Registro de ejecución*. Debe decir:
   ```
   Remitente: nuevo.correo@gprovivienda.com · modo=alias · nombre="notificador Provivienda"
   ```
   Si dice `modo=fallback`, el alias todavía no está verificado en **esa** cuenta (volver al paso 5).
9. Publicar los cambios: **Implementar → Gestionar implementaciones → ✏️ Editar → Versión: "Nueva versión" → Implementar.**
   La URL `/exec` **no cambia**, así que `index.html` no se toca.
10. **Prueba:** registrar un envío de prueba y revisar en el correo recibido que el remitente sea `notificador Provivienda <nuevo.correo@…>`.

> Solo cambiar el nombre visible (`SENDER_NAME`) también requiere los pasos 7, 9 y 10.

### 11.2 Opción 2 · Migración total a una cuenta nueva propietaria

**Requisitos:** acceso simultáneo (o coordinado) a la cuenta actual y a la cuenta nueva. Ventana de mantenimiento de unos minutos.

**A. Transferir permisos (desde la cuenta ACTUAL)**

1. **Hoja de cálculo:** *Compartir* → agregar el correo de la cuenta nueva → rol **Editor** → *Enviar*.
2. **Carpeta de evidencias de Drive** (la de `FOLDER_ID_FOTOS`): clic derecho → *Compartir* → agregar la cuenta nueva → **Editor**.
3. *(Recomendado)* En ambos elementos, abrir *Compartir* de nuevo y cambiar el rol de la cuenta nueva a **"Transferir propiedad"**. La cuenta nueva debe **aceptar** la solicitud desde Drive. En Workspace, si la opción no aparece, la transferencia la hace el administrador.
4. **Eliminar el activador de la cuenta actual:** abrir Apps Script y ejecutar `eliminarTriggerDiario()`. Los activadores son personales: la cuenta nueva no puede ver ni borrar los de la cuenta actual, y si quedan ambos, el Directorio se sincronizaría dos veces.

**B. Configurar el script (desde la cuenta NUEVA)**

5. Iniciar sesión con la cuenta nueva → abrir la hoja → **Extensiones → Apps Script**.
   Es **el mismo proyecto vinculado**: conserva el código, el servicio *People API* y la propiedad del script `ULTIMA_REFERENCIA`, así que la secuencia `#10000…` continúa sin saltos. Comprobar en *Servicios* que aparece **People**.
6. Revisar las constantes en `Code.gs` y guardar:
   - `SENDER_ALIAS` = correo de la cuenta nueva (queda en `modo=cuenta`) o un alias verificado suyo (ver §11.1).
   - `SENDER_NAME` = `notificador Provivienda`.
   - `FOLDER_ID_FOTOS`: **solo** si se usa otra carpeta. Si se compartió o transfirió la misma, no cambia.
7. Ejecutar **`setup()`** y **aceptar todos los permisos** con la cuenta nueva. Verificar en el registro:
   - `Remitente: <cuenta nueva> · modo=cuenta` (o `modo=alias`).
   - Carpeta de fotos y People API OK.
   - `⚠️ Sin activador diario` (es normal todavía).
8. Ejecutar **`configurarTriggerDiario()`** → `✅ Activador creado: sincronizarContactos todos los días ~6:00`.

**C. Nueva implementación (desde la cuenta NUEVA)**

9. **Implementar → Nueva implementación** → ⚙ *Seleccionar tipo* → **Aplicación web**:
   - **Ejecutar como:** *Yo (cuenta nueva)*
   - **Quién tiene acceso:** *Cualquier usuario*
   → **Implementar** → autorizar → **copiar la nueva URL** (termina en `/exec`).
   > No sirve editar la implementación anterior: seguiría ejecutándose como la cuenta antigua y enviando desde ella.

**D. Actualizar el frontend**

10. En `index.html`, reemplazar:
    ```js
    const APPS_SCRIPT_URL = 'https://script.google.com/macros/s/<NUEVO_ID>/exec';
    ```
11. Commit y push a la rama de GitHub Pages. Esperar 1–2 minutos y recargar con **Ctrl+F5** (los usuarios con la página abierta deben recargarla).
    `GOOGLE_CLIENT_ID` **no cambia**: el inicio de sesión del frontend es independiente de la cuenta que ejecuta el script (§9).

**E. Verificar y cerrar**

12. Pruebas: iniciar sesión, registrar un envío con foto (código consecutivo correcto), avanzar estados en el Panel y confirmar los correos con el nuevo remitente.
13. Al día siguiente, en Apps Script → *Ejecuciones*, comprobar que `sincronizarContactos` corrió a las ~06:00 bajo la cuenta nueva.
14. Desde la cuenta **antigua**: **Implementar → Gestionar implementaciones → Archivar** la implementación anterior, para que no queden dos backends activos. Opcionalmente, retirar el acceso de la cuenta antigua a la hoja y a la carpeta.

**Reversión:** mientras la implementación antigua no se archive, basta con volver a poner su URL en `APPS_SCRIPT_URL` y recrear su activador con `configurarTriggerDiario()` desde la cuenta antigua.

---

## 12. Importación del histórico (`historico.csv.csv`)

Observaciones del archivo actual (541 registros):

- **Columnas desplazadas**: tiene una columna vacía sin encabezado en la posición L; la foto está en M, el tipo en N, el proveedor en O y P. Antes de pegarlo en `Envios` hay que **eliminar esa columna vacía** para respetar el esquema A–P de §3.1 (si no, `foto`, `tipoMensajeria` y proveedor quedarán corridos).
- **Codificación**: no es UTF-8 (acentos dañados). Importar con Archivo → Importar en Google Sheets eligiendo la codificación correcta, o convertir a UTF-8 antes.
- **Referencias**: valores entre 268 y 54 155; 99 tienen menos de 5 dígitos. No colisionan con la secuencia nueva gracias a la verificación con TextFinder (§5.1), pero los de menos de 5 dígitos no se pueden actualizar con `updateStatus` (exige 5 dígitos).
- **Estados**: la mayoría vacíos (se interpretan como `Registrado` y aparecerán como "activos" en el Panel). Conviene marcarlos como `Entregado` en la hoja si ya fueron entregados.
- **Orden**: el sistema asume que las filas están en orden cronológico (el JSON se devuelve en orden inverso de filas). Pegar el histórico **antes** de los registros nuevos.
- Tras importar, si la última fila de la hoja es histórica, ejecutar `reiniciarSecuenciaReferencias(<último código nuevo emitido>)` para no depender de la reconstrucción automática.

---

## 13. Seguridad y limitaciones conocidas

- **Web App pública**: con acceso "Cualquier usuario", quien conozca la URL `/exec` puede leer todos los envíos (`doGet`) y llamar a `doPost`. El filtrado de "Mis Envíos" ocurre en el navegador.
- **Identidad no verificada en el backend**: `userEmail` lo envía el frontend. Mejoras posibles: verificar el ID token de Google en el backend (`UrlFetchApp` a `https://oauth2.googleapis.com/tokeninfo?id_token=…` y comprobar `aud`, `hd`, `exp`) y llenar `ADMIN_EMAILS` en `Code.gs`.
- **Fotos públicas por enlace**: cualquier persona con la URL puede verlas.
- **Sesión local sin caducidad**: se cierra solo con "Cerrar sesión" o borrando `localStorage`.
- **Sin fecha de "En Camino"**: el esquema de 16 columnas no la almacena.
- **Departamento en `Envios` (col. E)** no se normaliza en la hoja (los históricos quedan como se escribieron); la UI lo normaliza solo al mostrarlo.
- **Destinatarios fuera del Directorio**: no se pueden registrar desde la UI hasta que aparezcan en el Directorio (sincronización diaria a las 6:00).
- **Límite de códigos**: 90 000 (10000–99999).

---

## 14. Solución de problemas

| Síntoma | Causa probable | Acción |
|---|---|---|
| "El servidor no devolvió JSON" | URL `/exec` incorrecta, implementación sin acceso "Cualquier usuario", o error de permisos sin aceptar | Revisar implementación; ejecutar `setup()` y publicar nueva versión |
| Error CORS en consola | Se envió `Content-Type: application/json` u otra cabecera | Usar `text/plain;charset=utf-8` (§4.1) |
| Botón de Google no aparece / `origin_mismatch` | Origen no registrado en el cliente OAuth | Agregar `https://practicantetd-gprovivienda.github.io` (§9) |
| Correos salen desde otra cuenta | `SENDER_ALIAS` no es la cuenta ejecutora ni alias verificado | Implementar como `kawak@` o configurar "Enviar como" en Gmail de la cuenta ejecutora; revisar `setup()` (ver §11) |
| Miniaturas no cargan | Foto no compartida públicamente (política del dominio) | Permitir compartir con enlace en la carpeta / consola de administración |
| Directorio vacío | People API no habilitada o trigger no creado | Servicios → People API; `configurarTriggerDiario()`; `sincronizarContactos()` |
| Error 503 "sistema ocupado" | Muchas escrituras simultáneas | Reintentar; es esperado bajo alta concurrencia |
| Código no empieza en #10000 | La propiedad o el último registro tienen un número mayor | `reiniciarSecuenciaReferencias()` |
| Aparece "Grupo Provivienda"/"GP" en texto en vez del logo | No existe `logo-provivienda.png` (ni `.png.jpg`) junto a `index.html` en el repositorio | Subir/renombrar el archivo a `logo-provivienda.png` |
| "Permiso de cámara denegado" | El usuario o el navegador bloqueó la cámara | Permitir la cámara para el sitio (icono del candado) o usar "Subir archivo" |
| La cámara no abre en un iframe / vista previa | `getUserMedia` requiere HTTPS y permiso del contexto | Probar en la URL de GitHub Pages |
| "Elija al destinatario en la lista del directorio" | Se escribió un nombre sin elegirlo, o la persona no está en el Directorio | Elegirlo de la lista; si no aparece, esperar la sincronización diaria o revisar People API |
| Cambios en `Code.gs` no surten efecto | No se publicó una nueva versión | Implementar → Gestionar implementaciones → Editar → Nueva versión |

---

## 15. Guía para asistentes de IA

Invariantes que **no** deben romperse al modificar el sistema:

1. Encabezados y orden A–P de `Envios`; nombres de pestañas `Envios` y `Directorio`.
2. Valores literales: `Interna (Grupo Provivienda)`, `Externa`, `Registrado`, `Recibido`, `En Camino`, `Entregado`.
3. Contrato JSON de §4 (nombres de campos en español, `ok` en todas las respuestas, `referencia` como string en JSON y número en la hoja).
4. CORS: el frontend solo puede usar peticiones simples; no introducir cabeceras ni `application/json`.
5. Toda escritura en `Envios`/`Directorio` dentro de `obtenerLock_()`; la propiedad `ULTIMA_REFERENCIA` solo se actualiza después de escribir la fila.
6. Los correos no deben interrumpir la operación principal (siempre `try/catch`).
7. No agregar endpoints de edición de departamentos (decisión de producto: normalización solo en lectura/sincronización). En la UI, correo y departamento del destinatario son de solo lectura y vienen del Directorio.
8. No reintroducir el botón manual de sincronización: el Directorio se actualiza solo con el trigger diario.
9. UI solo con la paleta `brand` roja + neutros `slate`; referencias siempre mostradas con `#` (`fmtRef`).
10. Toda foto (cámara o archivo) pasa por `aJpegComprimido()` (máx. 800 px, JPEG 0.75) antes de enviarse.
11. Tras cambiar `Code.gs`: ejecutar `setup()` si hay servicios nuevos y publicar **nueva versión** de la implementación; la URL `/exec` se mantiene al editar una implementación existente.

Funciones públicas (ejecutables desde el editor o triggers): `doGet`, `doPost`, `sincronizarContactos`, `configurarTriggerDiario`, `eliminarTriggerDiario`, `reiniciarSecuenciaReferencias`, `setup`, `testDoGet`. Las funciones que terminan en `_` son privadas (no aparecen en el selector del editor).

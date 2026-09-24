# Panel de Gestión de Cobranza

Reportería de cobranza en un solo archivo HTML: clasifica llamadas de campañas de cobranza a partir de un CSV, agrupa por cliente y genera KPIs, embudo de resultados, desgloses, rendimiento por hora y una tabla de detalle con ficha por cliente — todo corriendo **100% en el navegador**, sin backend.

---

## Tabla de contenido

- [Características](#características)
- [Inicio rápido](#inicio-rápido)
- [Despliegue](#despliegue)
- [Arquitectura](#arquitectura)
- [Esquema de datos (CSV)](#esquema-de-datos-csv)
- [Reglas de clasificación](#reglas-de-clasificación)
- [Estructura del código](#estructura-del-código)
- [Personalización](#personalización)
- [Limitaciones conocidas](#limitaciones-conocidas)
- [Compatibilidad](#compatibilidad)
- [Changelog](#changelog)

---

## Características

- 📂 Carga de CSV por arrastre o selector de archivo, sin subir nada a ningún servidor
- 🏷️ Clasificación automática de cada llamada en 5 categorías (Efectivo, Directo, Indirecto, No contacto, Sin gestión)
- 👤 Agrupación por cliente (`unique_user_id`): cada cliente se representa por su **mejor resultado logrado**
- 📊 Resumen ejecutivo narrado automáticamente, KPIs, embudo de resultados y desgloses — todos con clic-para-filtrar
- 🕐 Rendimiento por hora del día
- 🔍 Tabla de detalle ordenable/buscable con panel lateral de ficha por cliente (identificadores, historial cronológico de gestiones, pago acordado)
- 🔗 SID de cada llamada enlazado al historial de Vozy
- 🌗 Modo claro/oscuro
- 🖨️ Exportación a CSV filtrado e impresión a PDF
- ⚡ Sin dependencias externas en tiempo de ejecución (PapaParse va incluido inline en el archivo)

## Inicio rápido

No hay build, no hay `npm install`, no hay servidor. Es un único archivo HTML.

```bash
# Clonar el repo
git clone <url-del-repo>
cd <repo>

# Abrir directamente en el navegador
open index.html        # macOS
xdg-open index.html     # Linux
start index.html        # Windows
```

O simplemente hacer doble clic en el archivo `.html`.

## Despliegue

Al ser un archivo estático, se puede alojar en cualquier hosting estático:

**Vercel**
```bash
npm i -g vercel
vercel --prod
```
(el proyecto solo necesita contener `index.html` en la raíz)

**Cualquier otro hosting estático:** Netlify, GitHub Pages, Cloudflare Pages, S3 + CloudFront, etc. — basta con subir el archivo `.html` tal cual.

## Arquitectura

```
CSV del usuario
      │
      ▼
 PapaParse (inline, sin Web Worker)
      │
      ▼
 DATA[]  ──────────────► filas normalizadas (headers trimeados, sin BOM)
      │
      ▼
 classify(row)  ────────► clasifica CADA llamada individual
      │
      ▼
 buildUnits()  ──────────► agrupa por unique_user_id, calcula:
      │                     • estado consolidado (mejor resultado)
      │                     • gestiones, última gestión, días sin gestión
      │                     • fecha/monto de promesa de pago
      │                     • __rows ordenado cronológicamente
      ▼
 UNITS[]  (una fila = un cliente)
      │
      ▼
 render()  ──────────────► pinta KPIs, resumen ejecutivo, embudo,
                            desgloses, rendimiento por hora, tabla
```

Todo el estado vive en variables JS de módulo (`DATA`, `FIELDS`, `C`, `UNITS`, `UNIT_BY_I`). No hay framework, no hay virtual DOM: el render es `innerHTML` directo por sección.

## Esquema de datos (CSV)

Las columnas se detectan por nombre mediante la función `col()` (normaliza a minúsculas y sin símbolos, así que "Contact ID", "ContactID" y "contact_id" son equivalentes). Ninguna columna es estrictamente obligatoria — si falta una, la sección que depende de ella se omite o queda vacía.

| Columna CSV | Variable interna (`C.*`) | Uso |
|---|---|---|
| `unique_user_id` | `C.uid` | Clave de agrupación por cliente |
| `lr_fullName` / `lr_name` | `C.name` | Nombre del cliente |
| `Phone` | `C.phone` | Teléfono |
| `lr_agentCompany` | `C.company` | Empresa/marca |
| `lr_product` | `C.product` | Producto / cuenta |
| `lr_debtAmount` | `C.debt` | Monto de deuda |
| `lr_limitDate` | `C.limitDate` | Fecha límite |
| `lr_dueDate` | `C.dueDate` | Fecha de vencimiento |
| `Contacted` | `C.contacted` | Booleano de contacto |
| `Success` | `C.success` | Booleano de acuerdo de pago |
| `contactability_type` | `C.contactType` | `contacted_yes`, `contact_family`, etc. |
| `HangUPCause` | `C.hangup` | Causa técnica de cierre de llamada |
| `LastNode` | `C.node` | Último nodo del flujo |
| `duration` | `C.duration` | Duración en segundos |
| `date` | `C.date` | Fecha de la llamada (`MM/DD/YYYY`) |
| `hour` | `C.hour` | Hora de la llamada (`H:MM am/pm`) |
| `rpt_payment_date` | `C.payDate` | Fecha de pago prometida |
| `rpt_payment_amount` | `C.payAmount` | Monto prometido |
| `rpt_reason` | `C.reason` | Motivo de no pago |
| `rpt_summary` | `C.summary` | Resumen de la llamada |
| `CampaignName` | `C.campaign` | Nombre de campaña |
| `voicemail` | `C.voicemail` | Booleano de buzón de voz |
| `Answered` | `C.answered` | Booleano de si la llamada fue contestada |
| `session_id` / `ContactID` | `C.sessionId` / `C.contactId` | SID de la llamada |

## Reglas de clasificación

Definidas en `classify(row)` (≈línea 483). Se evalúan **en orden**; la primera condición verdadera gana:

```js
function classify(r){
  // Regla dura: buzón de voz o llamada no contestada => siempre No contacto,
  // aunque otras columnas (Success, contactability_type) digan lo contrario.
  if (voicemail === true
      || Answered === false           // solo si la columna trae valor explícito
      || HangUPCause === "VOICEMAIL")                 return "nocontacto";

  if (Success === true)                              return "efectivo";
  if (contactability_type === "contacted_yes"
      && Success === false)                          return "directo";
  if (contactability_type === "contact_family")       return "indirecto";
  if (HangUPCause === "PENDING_EXECUTION")            return "singestion";
  /* cualquier otro caso: EXPIRED_EXECUTION, NO_ANSWER,
     SENT_TO_CHANNEL, etc. */                          return "nocontacto";
}
```

| Categoría | Significado de negocio |
|---|---|
| **Efectivo** | Se llegó a un acuerdo de pago |
| **Directo** | Se habló con el cuentahabiente, sin acuerdo |
| **Indirecto** | Contestó alguien que no es el cuentahabiente (familiar) |
| **No contacto** | El teléfono sonó (o se intentó) pero no se logró contacto real |
| **Sin gestión** | La llamada nunca llegó a marcarse (quedó pendiente en cola) |

**Estado consolidado por cliente** (`buildUnits()`, ≈línea 556): cada cliente toma el resultado de más alta prioridad entre todas sus llamadas, con este orden (`CAT_RANK`):

```
Efectivo (5) > Directo (4) > Indirecto (3) > No contacto (2) > Sin gestión (1)
```

Esta prioridad también gobierna el orden por defecto de la tabla y el ordenamiento de la columna "Estado" (no es alfabético).

## Estructura del código

Archivo único `index.html` (~1,100 líneas), dividido en:

```
<style>                      # design tokens (:root / [data-theme="dark"]), componentes
<body>
  .hero / .masthead          # encabezado, toggle de tema, botones
  #dropzone                  # carga de archivo
  #report                    # contenedor del reporte (oculto hasta cargar datos)
    #execSummary             # resumen ejecutivo narrado
    #kpis                    # tarjetas KPI
    #funnel                  # embudo de 5 categorías
    .grid2 > #bkCompany      # desglose por empresa
    #hourPanel                # rendimiento por hora
    #thead / #tbody          # tabla de detalle por cliente
  #overlay > .drawer         # ficha lateral del cliente
  #printOverlay               # overlay de generación de PDF
<script>
  PapaParse 5.4.1 minificado (inline)
  --- lógica de la app ---
  col(), truthy(), norm()     # helpers de parseo
  classify()                  # clasificación por llamada
  buildUnits()                # agrupación y agregación por cliente
  render()                    # orquesta el pintado de todas las secciones
  renderHours()                # rendimiento por hora
  breakdownField()            # desglose genérico (usado por Empresas)
  buildTable() / refresh() / drawMore()  # tabla paginada
  setFilter() / *FilterObj()  # filtros clic-para-filtrar
  openDrawer()                # ficha de detalle del cliente
  handleFile()                # entrada de datos (Papa.parse)
</script>
```

## Personalización

| Qué cambiar | Dónde |
|---|---|
| Reglas de clasificación | función `classify()` |
| Prioridad entre categorías | objeto `CAT_RANK` |
| Colores por categoría | objeto `CAT_COLOR` + variables CSS `--signal`, `--amber`, `--slate`, `--alert`, `--neutral` |
| Traducción de causas de finalización | objeto `HANGUP_LABELS` |
| Columnas de la tabla de detalle | array `TCOLS` |
| URL del historial de llamadas (SID) | función `vozyHistoryLink()` |
| Paleta de colores / modo oscuro | bloque `:root { }` y `[data-theme="dark"] { }` en `<style>` |
| Tamaño de página de la tabla | constante `PAGE` (por defecto 250) |
| Límite de filas al imprimir/PDF | constante `PRINT_CAP` (por defecto 500) |

## Limitaciones conocidas

- El parseo de CSV corre en el **hilo principal** (Web Workers deshabilitados a propósito, ver Changelog) — con archivos muy grandes (decenas de miles de filas) puede notarse una breve congelación de la UI durante la carga.
- El formato de fecha/hora está fijado a `MM/DD/YYYY` y `H:MM am/pm`; otros formatos no se interpretan correctamente para los cálculos de "última gestión" y "días sin gestión" (aunque sí se muestran tal cual como texto).
- No hay persistencia entre sesiones: cada carga de archivo empieza de cero, no hay `localStorage`.
- La URL del historial de Vozy asume el parámetro `session_id`; si la instancia real usa otro nombre de parámetro, ajustar `vozyHistoryLink()`.

## Compatibilidad

Navegadores modernos con soporte de ES6+ (`const`/`let`, arrow functions, template literals, `Map`, `IntersectionObserver`, `Blob`/`URL.createObjectURL`). Probado en Chrome/Edge/Firefox recientes. No requiere polyfills para su uso previsto.

## Changelog

| Versión | Cambios |
|---|---|
| v1 | Carga de CSV, clasificación Efectivo/Directo/Indirecto/No contacto, métricas por cliente |
| v2 | Rediseño visual (estética de sellos de tinta / ledger) |
| v3 | Rediseño completo: paleta lavanda/morado, modo oscuro, embudo, metas y semáforo, tabla con panel de detalle |
| v3.1 | Fix: PapaParse sin Web Workers (`SecurityError` en sandbox de artifacts) |
| v3.2 | Se quitan KPIs de duración y sección de metas/semáforo; embudo pasa a mostrar directamente las 5 categorías; se agrega "Sin gestión"; rendimiento por hora agrupado por hora completa |
| v3.3 | Tabla prioriza por defecto el mejor resultado (Efectivo primero) |
| v3.4 | Dropzone se oculta tras cargar archivo; botón "Subir otro CSV" en el encabezado, reemplaza datos sin recargar la página |
| v3.5 | Gestiones de cada cliente ordenadas cronológicamente en la ficha de detalle |
| v3.6 | SID (`session_id`/`ContactID`) visible y enlazado al historial de Vozy |
| v3.7 | Fix: ordenamiento de la columna "Estado" por prioridad de resultado, no alfabético |
| v3.8 | Se agregan columnas "Cuenta" (producto) y "Última gestión" a la tabla |
| v3.9 | "Indirecto" redefinido: exclusivamente `contactability_type = contact_family` |
| v3.10 | Traducción de causas de finalización (`HangUPCause`) a español |
| v3.11 | "Sin contacto" renombrado a "No contacto" |
| v3.12 | Fix: "Sin gestión" limitado a `PENDING_EXECUTION`; `EXPIRED_EXECUTION` reclasificado a "No contacto" |
| v3.13 | Fix: `voicemail=true`, `Answered=false` o `HangUPCause=VOICEMAIL` fuerzan "No contacto" por encima de cualquier otra clasificación |

---

## Licencia

_Agregar la licencia correspondiente al repositorio (MIT, propietaria, interna, etc.)._

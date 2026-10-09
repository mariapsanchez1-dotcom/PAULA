# Sistema integral — Materiales de Construcción en Seco y Steel Frame

> Documento de arquitectura v0.2 — propuesta inicial para validación.

## 1. Arquitectura técnica recomendada

### 1.1 Principios
- **Un solo lenguaje (TypeScript)** en frontend, backend y agente: menos curva de mantenimiento para un equipo chico.
- **Monorepo**: la lógica de cálculo de materiales y los tipos se comparten entre la web y el agente de email (el bot cotiza con el mismo motor que la web).
- **Servicios gestionados** (Supabase, Vercel) para no administrar servidores.
- **Revisión humana obligatoria**: ninguna respuesta de email ni presupuesto formal sale sin que una persona de la empresa lo apruebe. La IA y el calculador solo preparan borradores.
- **El sistema de cobro manda en los precios**: es la única fuente de verdad; el resto del sistema se sincroniza desde ahí.

### 1.2 Stack

| Capa | Tecnología | Por qué |
|---|---|---|
| Web pública + panel interno | **Next.js 15 (App Router) + React + TypeScript** | SSR/SEO para el catálogo, Server Actions, un solo deploy |
| UI | **TailwindCSS + shadcn/ui** | Componentes accesibles, rápido de iterar |
| Mobile | **PWA** (instalable) en la fase inicial | Evita mantener apps nativas; React Native/Expo queda como opción futura |
| Base de datos | **PostgreSQL en Supabase** | Auth, Storage (fichas técnicas, PDFs), Row Level Security, backups |
| ORM / migraciones | **Drizzle ORM** | Tipado fuerte, SQL explícito, migraciones versionadas |
| Validación | **Zod** | Esquemas compartidos (formularios, API, salidas del LLM) |
| Agente de email | **Worker Node.js (TypeScript)** | Comparte paquetes con la web |
| Cola de trabajos | **pg-boss** (cola sobre Postgres) | Sin infraestructura extra (no Redis) |
| Correo | **Gmail API** (o Microsoft Graph si usan Outlook) — fallback IMAP/SMTP | Lectura por push/polling, creación de borradores nativos |
| IA | **Claude API** — modelo rápido para clasificar/extraer, modelo principal para redactar; *tool use* para consultar stock/precios | Salidas estructuradas validadas con Zod |
| PDF | **@react-pdf/renderer** | Presupuestos con la misma estética que la web |
| Sincronización de precios | Importador desde el sistema de cobro (Excel/CSV o API) con vista previa de cambios | Ver sección 1.6 |
| Pagos (fase posterior) | **Mercado Pago** Checkout Pro | Estándar en Argentina |
| Deploy | **Vercel** (web) + **Railway/Fly.io** (worker) + Supabase | Costos bajos al inicio |
| Observabilidad | Sentry + logs estructurados (pino) | Trazabilidad de cada email procesado |

### 1.3 Diagrama de componentes

```
                ┌──────────────────────────┐
  Clientes ───► │  apps/web (Next.js)      │◄─── Personal de la empresa
  (web/PWA)     │  - Catálogo y precios    │     (panel /admin)
                │  - Calculadoras          │
                │  - PDF de presupuestos   │
                │  - Estado de consultas   │
                └──────────┬───────────────┘
                           │ Drizzle / Supabase client
                ┌──────────▼───────────────┐
                │ PostgreSQL (Supabase)    │◄──── Storage: fichas técnicas, PDFs
                │ + cola pg-boss           │
                └──────────▲───────────────┘
                           │
                ┌──────────┴───────────────┐      ┌──────────────┐
  Gmail/Outlook◄┤ apps/email-agent (Node)  ├─────►│  Claude API  │
                │ 1. Ingesta               │      └──────────────┘
                │ 2. Clasificación/extrac. │
                │ 3. Cotización (calc)     │
                │ 4. Borrador → revisión   │
                └──────────────────────────┘
```

### 1.4 Flujo del agente de email
1. **Ingesta**: nuevo mensaje → se guarda crudo en `emails` y se encola `procesar_email`.
2. **Identificación**: se busca o crea el `cliente` por dirección de correo; se agrupa por hilo en `consultas`.
3. **Clasificación + extracción** (LLM, salida JSON validada): intención (`precio`, `stock`, `ficha_tecnica`, `demora_entrega`, `cotizacion`, `reclamo`, `otro`), productos mencionados, cantidades, superficies (m²), localidad, urgencia, confianza.
4. **Resolución con herramientas**: el LLM llama a `buscar_producto`, `consultar_precio`, `consultar_stock`, `obtener_ficha_tecnica`, `calcular_materiales` (mismo motor que la web).
5. **Redacción** de la respuesta con el tono de la empresa.
6. **Revisión humana (siempre)**: el borrador queda en la bandeja de `/admin` como `pendiente_revision`. Una persona lo lee, puede editarlo y recién ahí lo envía o lo descarta. No existe envío automático.
   - La bandeja ordena por prioridad (reclamos, cotizaciones grandes, baja confianza de la IA arriba) y marca las alertas: precio cambiado desde que se redactó, producto sin stock, adjuntos con planos.
   - Si el borrador incluye un presupuesto, se aprueban juntos: el PDF se emite al aprobar.
7. **Registro y auditoría**: todo queda en `respuestas` y `eventos_consulta`; el cliente ve el estado en la web.

### 1.5 Motor de cálculo (`packages/calc`)
Funciones puras y testeadas. Las **recetas** (rendimientos por m²) viven en la base de datos para que la empresa las ajuste sin programar.

Ejemplo de receta "Tabique simple — placa 12,5 mm ambas caras, montante 70 mm c/40 cm" (valores de referencia, **a validar por la empresa**):

| Insumo | Cantidad por m² | Unidad |
|---|---|---|
| Placa de yeso 12,5 mm (1,20 × 2,40) | 2,0 + desperdicio 10 % | m² |
| Solera 70 mm | 0,9 | ml |
| Montante 70 mm | 2,5 | ml |
| Tornillo T1 (perfil-perfil) | 4 | u |
| Tornillo T2 (placa-perfil) | 30 | u |
| Masilla | 0,9 | kg |
| Cinta de papel | 3,0 | ml |
| Lana de vidrio (opcional) | 1,0 | m² |

El resultado se redondea a **unidades de venta** (placas enteras, perfiles de 2,60 m, cajas de 1000 tornillos, baldes, rollos) y se valoriza con la lista de precios vigente.

**Flujo del presupuesto web**: el cliente usa el calculador y ve en pantalla una **estimación** de materiales y precios. Al pedir el presupuesto formal, se crea una solicitud `pendiente_revision`; un vendedor la revisa y ajusta, y recién al aprobarla se genera el PDF numerado que recibe el cliente.

**Steel Frame**: estimador paramétrico por m² cubierto según tipología (vivienda 1 planta, 2 plantas, ampliación, galpón), con kg de acero/m², placas OSB, barreras, aislación y un rango de precio (mín–máx). Se presenta siempre como **estimación orientativa** que requiere visita técnica.

### 1.6 Actualización de precios desde el sistema de cobro

**Regla de oro**: los precios se cargan en un solo lugar (el sistema de cobro) y fluyen hacia la web, el calculador y el agente. Nadie edita precios a mano en este sistema.

**Cómo llegan los precios** (de menor a mayor automatización; se elige según el sistema de cobro que usen):
1. **Exportación Excel/CSV + importador** (punto de partida, sirve con cualquier sistema): alguien exporta la lista desde el sistema de cobro y la sube en `/admin/precios/importar`.
2. **Carpeta compartida o correo**: el sistema de cobro deja el archivo en Drive o lo envía a una casilla, y el importador lo levanta solo cada X horas.
3. **Integración directa por API o base de datos**, si el sistema de cobro lo permite.

**Cómo se importa sin romper nada**:
1. **Emparejamiento por código (SKU)**: cada producto se identifica por el mismo código que usa el sistema de cobro. Los códigos nuevos o desconocidos quedan aparte para revisarlos; nunca se crean productos solos.
2. **Vista previa**: antes de aplicar, se muestra qué cambia (precio anterior → nuevo, % de variación). Las variaciones fuera de rango (por ejemplo, más de 30 % o precio en cero) se marcan en rojo.
3. **Aplicación en un solo paso**: o se aplica todo el lote o nada. Se crea una nueva versión en `precios`; las anteriores quedan como historial, así se puede **volver atrás** al lote previo con un clic.
4. **Propagación inmediata**: catálogo web, calculadoras y agente leen siempre el precio vigente, así que se actualizan solos (con un refresco de caché del sitio al terminar la importación).

**Qué pasa con lo que ya existe** (para no comprometer documentos emitidos):

| Elemento | Qué pasa al cambiar los precios |
|---|---|
| Presupuesto **emitido** (PDF enviado) | **No cambia.** Cada ítem guarda una copia del precio del momento; el PDF es inmutable y se respeta hasta su fecha de validez |
| Presupuesto **vencido** que el cliente quiere retomar | Se genera una **nueva versión** a precios actuales (el original queda archivado) |
| Presupuesto **pendiente de revisión** | Se recalcula y se marca "precios actualizados" para que el vendedor lo vea antes de aprobar |
| Borrador de email **pendiente** que menciona precios | Se marca con alerta "precio cambió desde la redacción" y se puede regenerar con un clic |
| Recetas del calculador | No se tocan: guardan **cantidades**, no precios |
| Estimaciones en pantalla del cliente | Siempre muestran el precio vigente al momento de consultar |

**Material que conviene que la empresa facilite**:
- **Lista de precios en Excel/CSV exportada del sistema de cobro** (preferible a un PDF). Columnas mínimas: código/SKU, descripción, unidad de venta, precio, si incluye IVA o no. Un PDF también sirve para la carga inicial, pero hay que convertirlo y revisarlo; para las actualizaciones periódicas conviene el Excel.
- Nombre del sistema de cobro o facturación, para evaluar si se puede conectar directo.
- Fichas técnicas de los productos (PDF de los fabricantes) e imágenes, si las tienen.
- Rendimientos por m² que usan hoy para presupuestar (o un presupuesto real de tabique, cielorraso y Steel Frame como ejemplo).
- 20 a 50 correos reales de clientes (se anonimizan) para probar al agente.
- Texto de condiciones del presupuesto: validez, formas de pago, entrega, logo y datos fiscales.

---

## 2. Estructura del repositorio

```
/
├── apps/
│   ├── web/                         # Next.js: sitio público + panel admin
│   │   ├── src/app/
│   │   │   ├── (public)/
│   │   │   │   ├── catalogo/        # listado, filtros, [slug] del producto
│   │   │   │   ├── presupuestar/
│   │   │   │   │   ├── seco/        # tabiques, cielorrasos, revestimientos
│   │   │   │   │   └── steel-frame/
│   │   │   │   ├── presupuestos/[id]/   # ver/descargar PDF
│   │   │   │   └── mi-cuenta/       # estado de consultas y presupuestos
│   │   │   ├── admin/               # productos, precios, recetas, bandeja de emails
│   │   │   └── api/                 # webhooks (Gmail push, Mercado Pago), PDF
│   │   ├── src/components/
│   │   ├── src/features/            # lógica por dominio (catalogo, presupuesto, consultas)
│   │   └── public/
│   └── email-agent/                 # Worker Node.js
│       ├── src/
│       │   ├── ingest/              # conectores Gmail / Graph / IMAP
│       │   ├── pipeline/            # clasificar, extraer, resolver, redactar
│       │   ├── tools/               # herramientas expuestas al LLM
│       │   ├── prompts/             # prompts versionados
│       │   ├── rules/               # reglas de aprobación
│       │   └── jobs/                # handlers de pg-boss
│       └── test/fixtures/           # emails reales anonimizados para pruebas
├── packages/
│   ├── db/                          # Drizzle: schema, migraciones, seeds
│   │   ├── src/schema/
│   │   ├── migrations/
│   │   └── seed/                    # catálogo inicial, recetas
│   ├── calc/                        # motor de cálculo de materiales (puro + tests)
│   ├── pdf/                         # plantillas de presupuesto
│   ├── shared/                      # tipos, esquemas Zod, utilidades (moneda, IVA)
│   └── ui/                          # componentes compartidos (opcional)
├── docs/                            # arquitectura, decisiones (ADR), manual de uso
├── .github/workflows/               # CI: lint, typecheck, tests
├── .env.example
├── package.json                     # pnpm workspaces
├── pnpm-workspace.yaml
└── turbo.json                       # Turborepo
```

---

## 3. Esquema preliminar de base de datos

Convenciones: `id uuid` PK, `created_at`/`updated_at timestamptz`, importes en `numeric(14,2)` en **ARS**, IVA explícito.

### Catálogo y precios
```sql
categorias (id, nombre, slug, padre_id → categorias, orden)

productos (
  id, sku UNIQUE,          -- mismo código que el sistema de cobro nombre, slug UNIQUE, descripcion,
  categoria_id → categorias, marca,
  unidad_venta,            -- 'unidad' | 'caja' | 'rollo' | 'balde' | 'kg' | 'ml' | 'm2'
  contenido_por_unidad,    -- ej. 1000 tornillos/caja, 2.6 ml/perfil, 2.88 m2/placa
  medidas jsonb,           -- espesor, largo, ancho, calibre...
  ficha_tecnica_url, imagen_url,
  activo bool, publicado_web bool
)

listas_precios (id, nombre, tipo,     -- 'publico' | 'mayorista' | 'obra'
  vigente_desde, vigente_hasta, activa bool)

precios (id, producto_id, lista_id, precio_neto, alicuota_iva default 21,
  vigente_desde, importacion_id → importaciones_precios, cargado_por → usuarios)
  -- se insertan filas nuevas (historial), nunca se pisan

importaciones_precios (id, origen,   -- 'excel' | 'csv' | 'api' | 'drive'
  archivo_url, estado,               -- 'vista_previa' | 'aplicada' | 'revertida' | 'fallida'
  total_filas, cambios, nuevos_sin_emparejar, alertas jsonb,
  aplicada_por → usuarios, aplicada_en)

stock (producto_id PK, cantidad, reservado, punto_reposicion,
  demora_reposicion_dias, actualizado_en)
```

### Recetas de cálculo
```sql
sistemas_constructivos (id, codigo, nombre,
  tipo,                    -- 'tabique' | 'cielorraso' | 'revestimiento' | 'steel_frame'
  parametros jsonb,        -- espesor perfil, separación, placas por cara...
  desperdicio_pct, activo)

receta_items (id, sistema_id, producto_id, cantidad_por_m2, unidad,
  opcional bool, condicion jsonb)    -- ej. "solo si aislacion = true"

steel_frame_tipologias (id, nombre, kg_acero_m2,
  precio_m2_min, precio_m2_max, incluye jsonb)
```

### Clientes
```sql
clientes (id, tipo,        -- 'particular' | 'profesional' | 'empresa'
  nombre, razon_social, cuit, email UNIQUE, telefono,
  localidad, provincia, origen,      -- 'web' | 'email' | 'mostrador'
  auth_user_id → auth.users NULL, notas, acepta_marketing bool)
```

### Presupuestos
```sql
presupuestos (id, numero SERIAL UNIQUE, cliente_id, origen,  -- 'web' | 'email' | 'admin'
  estado,                  -- 'estimacion' | 'pendiente_revision' | 'emitido'
                           -- | 'aceptado' | 'vencido' | 'rechazado' | 'reemplazado'
  version, presupuesto_original_id NULL,   -- versiones al re-cotizar
  revisado_por → usuarios, emitido_en,
  precios_desactualizados bool,             -- alerta si cambió la lista antes de aprobar
  lista_id, subtotal, iva, total, validez_hasta,
  datos_calculo jsonb,     -- entradas del cliente (m2, sistema, opciones)
  pdf_url, consulta_id NULL, creado_por NULL)

presupuesto_items (id, presupuesto_id, producto_id, descripcion,
  cantidad, unidad,
  precio_unitario, alicuota_iva,   -- COPIA del precio al emitir (no cambia luego)
  precio_id → precios,             -- de qué versión de precio salió
  subtotal, orden)
```

### Consultas y emails
```sql
consultas (id, cliente_id, canal,     -- 'email' | 'web' | 'whatsapp'
  asunto, intencion, estado,          -- 'nueva' | 'en_proceso' | 'pendiente_revision'
                                      -- | 'respondida' | 'cerrada'
  prioridad, asignada_a → usuarios, thread_id_externo)

emails (id, consulta_id, direccion,   -- 'entrante' | 'saliente'
  message_id_externo UNIQUE, de, para, cc, asunto,
  cuerpo_texto, cuerpo_html, adjuntos jsonb, recibido_en)

extracciones_ia (id, email_id, modelo, intencion, confianza,
  datos jsonb,             -- productos, cantidades, m2, localidad...
  tokens_entrada, tokens_salida, costo_usd)

respuestas (id, consulta_id, email_id_respuesta NULL,
  borrador_ia,             -- texto original de la IA
  texto_final,             -- texto que aprobó/editó la persona
  estado,                  -- 'pendiente_revision' | 'aprobada' | 'enviada' | 'descartada'
  alertas jsonb,           -- precio cambiado, sin stock, adjuntos...
  aprobada_por → usuarios, -- obligatorio para poder enviar
  enviada_en, presupuesto_id NULL)

eventos_consulta (id, consulta_id, tipo, detalle jsonb, actor, creado_en)
```

### Usuarios internos
```sql
usuarios (id → auth.users, nombre, rol)   -- 'admin' | 'ventas' | 'deposito'
```

**Seguridad (RLS)**: el cliente autenticado solo lee sus propios `presupuestos` y `consultas`; catálogo y precios de lista pública son de lectura anónima; el resto exige rol interno.

---

## 4. Plan de desarrollo por fases

Sprints de 2 semanas (estimación para 1–2 desarrolladores).

### Fase 1 — Base y datos (Sprints 1–2)
- Monorepo (pnpm + Turborepo), lint, formato, CI en GitHub Actions.
- Proyecto Supabase (dev/prod), variables de entorno, `.env.example`.
- `packages/db`: esquema Drizzle completo, migraciones, RLS.
- Seed: categorías, ~50 productos reales, lista de precios, 3–4 recetas de seco.
- **Importador de precios desde el sistema de cobro** (Excel/CSV) con vista previa, alertas y reversión.
- Auth: usuarios internos (email + contraseña) y clientes (magic link).
- Panel admin mínimo: ABM de productos, carga de precios, stock.
- ✅ Entregable: base de datos operativa y catálogo cargable por la empresa.

### Fase 2 — App de clientes (Sprints 3–6)
- Sprint 3: catálogo público (filtros, búsqueda, ficha de producto, PDF técnico).
- Sprint 4: `packages/calc` con tests + calculadora de construcción en seco (tabiques, cielorrasos, revestimientos), redondeo a unidades de venta.
- Sprint 5: cotizador Steel Frame estimativo + solicitud de presupuesto formal + bandeja de revisión para vendedores + PDF al aprobar.
- Sprint 6: "Mi cuenta" (estado de consultas/presupuestos), PWA, SEO, analítica; Mercado Pago opcional.
- ✅ Entregable: sitio público en producción.

### Fase 3 — Agente de email (Sprints 7–10)
- Sprint 7: conector Gmail/Outlook, ingesta, hilos → `consultas`, cola pg-boss.
- Sprint 8: clasificación y extracción con Claude (JSON validado), set de ~50 emails reales anonimizados como pruebas de regresión.
- Sprint 9: herramientas (precio, stock, fichas, cálculo), redacción de borradores, bandeja de revisión en `/admin` (editar, aprobar, descartar).
- Sprint 10: alertas (precio cambiado, sin stock), métricas (tiempo de respuesta, % de borradores aprobados sin editar, costo IA), ajuste de prompts.
- ✅ Entregable: agente que redacta borradores; **todas las respuestas se envían solo tras aprobación humana**.

### Fase 4 — Evolución (opcional)
WhatsApp Business, app nativa (Expo), integración con sistema contable/facturación (AFIP/ARCA), CRM de seguimiento de obras.

---

## 5. Decisiones

**Definidas**
- Todas las respuestas de email y todos los presupuestos formales pasan por revisión humana.
- El sistema de cobro es la fuente de verdad de los precios; los presupuestos emitidos conservan su precio.

**Pendientes (a confirmar con la empresa)**
1. Nombre del sistema de cobro y formato de exportación disponible (Excel, CSV, API).
2. Proveedor de correo actual (Gmail/Google Workspace, Outlook/Microsoft 365, otro).
3. ¿Precios con o sin IVA en el catálogo público? ¿Listas diferenciadas para profesionales?
4. Rendimientos por m² y desperdicios que usa hoy la empresa.
5. ¿Compra directa con pago online desde el inicio o solo "solicitud de contacto"?

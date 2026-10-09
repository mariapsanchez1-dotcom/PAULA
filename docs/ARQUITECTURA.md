# Sistema integral — Materiales de Construcción en Seco y Steel Frame

> Documento de arquitectura v0.1 — propuesta inicial para validación.

## 1. Arquitectura técnica recomendada

### 1.1 Principios
- **Un solo lenguaje (TypeScript)** en frontend, backend y agente: menos curva de mantenimiento para un equipo chico.
- **Monorepo**: la lógica de cálculo de materiales y los tipos se comparten entre la web y el agente de email (el bot cotiza con el mismo motor que la web).
- **Servicios gestionados** (Supabase, Vercel) para no administrar servidores.
- **Humano en el circuito**: el agente redacta; solo envía solo en casos de bajo riesgo y alta confianza.

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
                │ 4. Borrador / envío      │
                └──────────────────────────┘
```

### 1.4 Flujo del agente de email
1. **Ingesta**: nuevo mensaje → se guarda crudo en `emails` y se encola `procesar_email`.
2. **Identificación**: se busca o crea el `cliente` por dirección de correo; se agrupa por hilo en `consultas`.
3. **Clasificación + extracción** (LLM, salida JSON validada): intención (`precio`, `stock`, `ficha_tecnica`, `demora_entrega`, `cotizacion`, `reclamo`, `otro`), productos mencionados, cantidades, superficies (m²), localidad, urgencia, confianza.
4. **Resolución con herramientas**: el LLM llama a `buscar_producto`, `consultar_precio`, `consultar_stock`, `obtener_ficha_tecnica`, `calcular_materiales` (mismo motor que la web).
5. **Redacción** de la respuesta con el tono de la empresa.
6. **Reglas de aprobación** (configurables en la tabla `reglas_aprobacion`):
   - *Auto-envío*: intención de bajo riesgo (ficha técnica, horarios, stock disponible) **y** confianza ≥ umbral **y** cliente no marcado como sensible.
   - *Revisión humana obligatoria*: cotizaciones con importes, reclamos, montos > X, adjuntos (planos), baja confianza.
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

**Steel Frame**: estimador paramétrico por m² cubierto según tipología (vivienda 1 planta, 2 plantas, ampliación, galpón), con kg de acero/m², placas OSB, barreras, aislación y un rango de precio (mín–máx). Se presenta siempre como **estimación orientativa** que requiere visita técnica.

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
  id, sku UNIQUE, nombre, slug UNIQUE, descripcion,
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
  vigente_desde, cargado_por → usuarios)
  -- se insertan filas nuevas (historial), nunca se pisan

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
  estado,                  -- 'borrador' | 'emitido' | 'aceptado' | 'vencido' | 'rechazado'
  lista_id, subtotal, iva, total, validez_hasta,
  datos_calculo jsonb,     -- entradas del cliente (m2, sistema, opciones)
  pdf_url, consulta_id NULL, creado_por NULL)

presupuesto_items (id, presupuesto_id, producto_id, descripcion,
  cantidad, unidad, precio_unitario, alicuota_iva, subtotal, orden)
```

### Consultas y emails
```sql
consultas (id, cliente_id, canal,     -- 'email' | 'web' | 'whatsapp'
  asunto, intencion, estado,          -- 'nueva' | 'en_proceso' | 'esperando_aprobacion'
                                      -- | 'respondida' | 'cerrada'
  prioridad, asignada_a → usuarios, thread_id_externo)

emails (id, consulta_id, direccion,   -- 'entrante' | 'saliente'
  message_id_externo UNIQUE, de, para, cc, asunto,
  cuerpo_texto, cuerpo_html, adjuntos jsonb, recibido_en)

extracciones_ia (id, email_id, modelo, intencion, confianza,
  datos jsonb,             -- productos, cantidades, m2, localidad...
  tokens_entrada, tokens_salida, costo_usd)

respuestas (id, consulta_id, email_id_respuesta NULL, borrador_texto,
  estado,                  -- 'borrador' | 'aprobada' | 'enviada' | 'descartada'
  modo,                    -- 'auto' | 'revision'
  aprobada_por, enviada_en, presupuesto_id NULL)

reglas_aprobacion (id, intencion, confianza_minima, monto_maximo,
  auto_enviar bool, activa)

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
- **Importador de precios desde Excel/CSV** (la actualización de precios es frecuente).
- Auth: usuarios internos (email + contraseña) y clientes (magic link).
- Panel admin mínimo: ABM de productos, carga de precios, stock.
- ✅ Entregable: base de datos operativa y catálogo cargable por la empresa.

### Fase 2 — App de clientes (Sprints 3–6)
- Sprint 3: catálogo público (filtros, búsqueda, ficha de producto, PDF técnico).
- Sprint 4: `packages/calc` con tests + calculadora de construcción en seco (tabiques, cielorrasos, revestimientos), redondeo a unidades de venta.
- Sprint 5: cotizador Steel Frame estimativo + generación de PDF + envío por email + solicitud de contacto.
- Sprint 6: "Mi cuenta" (estado de consultas/presupuestos), PWA, SEO, analítica; Mercado Pago opcional.
- ✅ Entregable: sitio público en producción.

### Fase 3 — Agente de email (Sprints 7–10)
- Sprint 7: conector Gmail/Outlook, ingesta, hilos → `consultas`, cola pg-boss.
- Sprint 8: clasificación y extracción con Claude (JSON validado), set de ~50 emails reales anonimizados como pruebas de regresión.
- Sprint 9: herramientas (precio, stock, fichas, cálculo), redacción de borradores, bandeja de aprobación en `/admin`.
- Sprint 10: reglas de auto-envío, métricas (tiempo de respuesta, % auto-resuelto, costo IA), ajuste de prompts.
- ✅ Entregable: agente en **modo sombra** (solo borradores) 2 semanas → luego auto-envío gradual.

### Fase 4 — Evolución (opcional)
WhatsApp Business, app nativa (Expo), integración con sistema contable/facturación (AFIP/ARCA), CRM de seguimiento de obras.

---

## 5. Decisiones pendientes (a confirmar con la empresa)
1. Proveedor de correo actual (Gmail/Google Workspace, Outlook/Microsoft 365, otro).
2. ¿Precios con o sin IVA en el catálogo público? ¿Se muestran listas diferenciadas a profesionales?
3. ¿Hay sistema de stock/facturación existente con el que integrarse?
4. Rendimientos por m² y desperdicios que usa hoy la empresa.
5. ¿Compra directa con pago online desde el inicio o solo "solicitud de contacto"?

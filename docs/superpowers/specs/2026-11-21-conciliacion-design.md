# Conciliación — Albaranes, facturas y pagos de proveedores

**Fecha:** 2026-11-21
**Estado:** Borrador para revisión de Guido
**Origen funcional:** "261121_Natif_Especificación funcional Conciliación_In" (claude.ai)

## 1. Objetivo y alcance

Conciliación es una app web que cruza **cada albarán con su factura y cada factura con su pago**, y muestra las diferencias en euros por proveedor y por local. El primer cliente es Natif Coffee SL (Madrid). La única tarea del cliente es fotografiar los albaranes desde una tablet. Basal opera el resto: revisa lo dudoso, completa maestros, resuelve incidencias y prepara reclamos.

Se construye **como app independiente para probarla**. Después se integra a `basal-erp` como un módulo más. Por eso usa **el mismo stack y las mismas reglas que basal-erp** (§3): integrar es mover código y mapear tablas, no reescribir.

**Dentro de la v1:**

- Captura de albaranes, facturas y tickets desde la tablet de cada local.
- Lectura con IA (API de Claude), validación y bandeja de revisión.
- Importación del extracto de Santander (Excel).
- Cruce factura ↔ pago y cruce albarán ↔ factura línea a línea.
- Incidencias y reclamos con aprobación del Dueño.
- Vistas: inicio, documentos, revisión, cruces, pagos, incidencias, proveedores.
- Alerta de subida de precio contra la compra anterior.
- Lectura automática de facturas del buzón info@natif.es (GoDaddy, IMAP, solo lectura).

**Fuera de la v1 (documentado en §12):** WhatsApp, conexión directa con el banco, cuenta PGC por factura, albaranes internos entre locales, escandallos, proyección de caja, pedidos y P&G por local.

## 2. Principios

1. **Ninguna factura con diferencia figura como conciliada.** Toda diferencia es una incidencia con importe hasta que se cierra con abono, corrección o decisión registrada.
2. **No se inventan datos.** Lo que la IA no lee con confianza va a revisión; nunca se completa una cantidad ni un precio adivinado.
3. **Los originales no se modifican.** Cada archivo se guarda tal como llegó, identificado por su huella SHA-256, y cada dato conserva el vínculo a su archivo.
4. **El cliente no carga datos.** Las pantallas del local solo capturan; la edición es de Basal.
5. **Lógica pura y testeable.** Los cruces y validaciones son funciones puras, sin Prisma, como `computeItemCost` y `planFifoConsumption` en el ERP.

## 3. Stack y lo que se copia de basal-erp

| Pieza | Decisión |
|---|---|
| Framework | Next.js 14 (App Router) + TypeScript, igual que el ERP |
| Base de datos | Postgres en Supabase (proyecto `Conciliacion`, región eu-west-3), con Prisma 5 y migraciones escritas a mano, igual que el ERP (nunca `prisma migrate dev`) |
| Aislamiento | Multi-empresa con RLS nativo de Postgres: `withTenant` / `withUser` como únicas entradas a los datos, el rol `app_user` sin BYPASSRLS y las políticas `<tabla>_isolation` |
| Login | NextAuth 4 con credenciales, igual que el ERP (contraseña temporal, cambio obligatorio en el primer ingreso) |
| UI | Tailwind 4 + shadcn/base-ui, componentes de `components/ui` y `components/shell`, y la marca Basal |
| Auditoría | `recordAudit` en la misma transacción de cada alta, modificación o baja |
| Archivos | Supabase Storage, en un bucket privado `documentos`. El servidor sube y descarga con la clave de servicio; el navegador ve las fotos por URL firmada de corta duración |
| IA | API de Claude con visión, solo desde el servidor. El modelo se configura con `ANTHROPIC_MODEL` y la salida es un JSON estructurado definido por esquema (tool use) |
| Tareas programadas | Rutas `/api/cron/*` protegidas con `CRON_SECRET`, llamadas cada minuto por `pg_cron` + `pg_net` desde Supabase, así no dependemos del plan de Vercel |
| Deploy | Vercel conectado a GitHub: `main` va a producción y las ramas a previews. Nunca se usa el CLI de Vercel |
| Tests | Vitest. Las funciones puras se prueban sin base; las integraciones, contra un Postgres real con fixtures de varias empresas para verificar RLS |

**Se copia del ERP casi sin cambios:** `lib/db` (withTenant, prisma extendido, audit), `lib/auth`, `lib/rbac` (con un catálogo propio), `lib/format.ts`, `lib/utils.ts`, `components/ui`, `components/shell`, `components/brand`, `middleware.ts`, la estructura de `app/(main)` y `app/login`, los scripts `createTenant` / `seedProd` y las migraciones de core (tenants, users, memberships, roles, permisos, módulos, audit_logs). En el código copiado se deja anotado de qué archivo y commit del ERP sale cada uno.

**Diferencia deliberada con el ERP: los importes van en `Decimal`, no en `Float`.** Una app de conciliación compara céntimos, y `Float` acumula errores de redondeo que aparecen como diferencias falsas. Los importes van en `Decimal(14,2)`, y las cantidades, precios unitarios y factores en `Decimal(14,4)`. Las funciones puras trabajan en céntimos y milésimas enteras. La conversión al `Float` del ERP se resuelve en la integración (§11).

## 4. Localización España

| Tema | Regla |
|---|---|
| Identificación fiscal | CIF/NIF/NIE validados con su dígito de control (`lib/es/nif.ts`, pura). Se guarda normalizado, sin guiones ni espacios y en mayúsculas |
| IVA | Tipos 0, 4, 10 y 21%, con recargo de equivalencia opcional (0,5 / 1,4 / 5,2%). Base, cuota y recargo se guardan por tipo |
| Comprobantes | `albaran`, `factura` (completa), `factura_simplificada` (ticket), `abono` (factura rectificativa) |
| Moneda | EUR, una sola moneda por empresa |
| Formato visible | Punto de miles y coma decimal (`1.234,56 €`), con helpers en `lib/format.ts`. Es el mismo separador que la regla es-AR del ERP; se agrega `formatEuro` |
| Fechas | DD/MM/AAAA en pantalla e ISO en la base. Zona horaria de la empresa: `Europe/Madrid` |

## 5. Perfiles y permisos

Perfiles fijos en código (`lib/rbac/roleProfiles.ts`), como en el ERP:

| Perfil | Quién en Natif | Permisos |
|---|---|---|
| `dueno` | Robbie | Ve todo y aprueba reclamos y diferencias aceptadas. No edita documentos |
| `consulta` | Juampo | Ve todo y no edita |
| `operador` | Basal | Todo: revisar, editar, cruzar, gestionar incidencias, importar extractos, maestros, usuarios y configuración |
| `local` | La tablet de cada local | Solo captura, y ve lo subido por su local en los últimos 7 días |

Catálogo de permisos (`moduleKey = conciliacion`): `view`, `capture`, `review`, `match`, `incidents`, `incidents_approve`, `bank_import`, `masters`, `settings`, `sensitive` (ver CIF completos y datos bancarios). Cada pantalla y cada acción del servidor verifican el permiso.

**Usuario de tablet:** un usuario por local, con el local fijo en su membresía (`tenant_memberships.location_id`). Basal inicia la sesión una sola vez en la tablet, y la sesión dura 180 días.

## 6. Modelo de datos

Todas las tablas son de una empresa: llevan `tenant_id`, RLS estándar, `created_at` / `updated_at` / `created_by` / `updated_by` y auditoría, salvo donde se aclara. Se indica la tabla equivalente de basal-erp para la integración.

### Maestros

**`locations`** (ERP: `locations`): `name`, `kind` (`local` | `central`), `delivery_address`, `active`. En Natif: Kitchen, Bakery, Natif 3 y Estructura central.

**`suppliers`** (ERP: `suppliers`): `legal_name`, `trade_name`, `tax_id` (NIF validado, único por empresa cuando no es null, con índice parcial), `bank_aliases` (text[]: cómo aparece en el extracto, por ejemplo "EUROPATATAS CEN"), `payment_method` (`transferencia` | `domiciliacion` | `tarjeta` | `efectivo` | `mixto`), `payment_term_days`, `has_prepaid_balance` (bool, para el caso Euceco), `category`, `email`, `notes`, `active`.

**`supply_items`**, los insumos de Natif (ERP: `items` de tipo insumo): `name`, `base_unit` (clave del catálogo de unidades del ERP: g, kg, ml, l, ud), `category`, `active`. Se cargan del Maestro de Insumos de los escandallos.

**`product_equivalences`** (ERP: `item_purchase_units`, más el código del proveedor): `supplier_id`, `supplier_code` (nullable), `supplier_description` (normalizada), `supply_item_id`, `purchase_unit_label` (por ejemplo "caja 6x1 L"), `factor_to_base` (Decimal 14,4: 6000 para 6 x 1 L expresado en ml), `active`. Es única por (`supplier_id`, `supplier_code` o `supplier_description`).

### Captura y lectura

**`source_files`**: `sha256` (único por empresa), `storage_path`, `mime_type`, `size_bytes`, `page_count`, `origin` (`tablet` | `mail` | `bank_import` | `manual` | `whatsapp`), `location_id` (nullable), `uploaded_by`, `uploaded_at`, `mail_message_id` (nullable), `local_note` (la nota "Llegó con diferencias"), `reported_difference` (bool), `status` (`pendiente_lectura` | `leyendo` | `leido` | `rechazado` | `duplicado`), `reject_reason`, `attempts`. Un archivo puede tener varias páginas y dar uno o más documentos.

**`documents`** (ERP: `purchases`): `source_file_id`, `doc_type` (`albaran` | `factura` | `factura_simplificada` | `abono` | `otro`), `supplier_id` (nullable hasta que se identifica), `supplier_tax_id_read`, `supplier_name_read`, `doc_number`, `doc_date`, `due_date`, `location_id`, `referenced_delivery_numbers` (text[]: los albaranes que cita la factura), `rectifies_document_id` (en un abono), `net_total`, `vat_total`, `surcharge_total`, `total` (Decimal 14,2), `read_confidence` (`alta` | `media` | `baja`), `read_status` (`leido` | `a_revisar` | `revisado` | `rechazado`), `match_status` (`pendiente` | `conciliado` | `con_diferencia` | `falta_albaran` | `revision` | `aceptado_con_diferencia` | `no_aplica`), `payment_status` (`pendiente` | `parcial` | `pagada` | `vencida` | `no_aplica`), `extraction_json` (jsonb: la salida cruda de la IA y la versión del prompt), `prompt_version`. Tiene un índice único parcial en (`tenant_id`, `supplier_id`, `doc_type`, `doc_number`) WHERE `doc_number` IS NOT NULL AND `read_status` <> 'rechazado'.

**`document_vat_breakdown`**: `document_id`, `vat_rate`, `base`, `vat_amount`, `surcharge_rate`, `surcharge_amount`.

**`document_lines`** (ERP: `purchase_lines`): `document_id`, `line_no`, `supplier_code`, `description`, `quantity`, `unit_label`, `unit_price`, `discount_pct`, `vat_rate`, `line_net` (Decimal), `equivalence_id` (nullable), `supply_item_id` (nullable), `base_quantity` y `unit_price_base` (calculados con la equivalencia), `field_flags` (jsonb: los campos dudosos de la línea).

**`review_items`**, la bandeja de Basal: `kind` (`lectura` | `proveedor_nuevo` | `equivalencia_nueva` | `cruce_propuesto` | `pago_propuesto`), `document_id` / `bank_movement_id` / `match_id` (los que apliquen), `reason`, `status` (`abierto` | `resuelto` | `descartado`), `resolved_by`, `resolved_at`, `review_seconds` (para la métrica de tiempo de revisión).

### Banco

**`bank_accounts`**: `bank` (`santander`), `name`, `last4`, `active`. El IBAN completo solo con el permiso `sensitive`.

**`bank_imports`**: `bank_account_id`, `source_file_id`, `period_from`, `period_to`, `rows_total`, `rows_new`, `rows_duplicate`.

**`bank_movements`** (ERP: `treasury_movements`): `bank_account_id`, `bank_import_id`, `op_date`, `value_date`, `concept`, `amount` (Decimal; negativo = salida), `balance_after`, `bank_code` (071, 072, 136, 174…), `row_hash` (único por cuenta: fecha + concepto + importe + saldo), `counterparty_read` (contraparte extraída del concepto), `supplier_id` (detectado), `kind` (`transferencia` | `recibo` | `tarjeta` | `impuesto` | `efectivo` | `ingreso` | `otro`), `classification` (`proveedor` | `no_proveedor`), `match_status` (`pendiente` | `conciliado` | `propuesto` | `sin_factura` | `excluido`).

### Cruces e incidencias

**`matches`**: `kind` (`albaran_factura` | `factura_pago`), `status` (`propuesto` | `confirmado` | `descartado`), `confidence` (0–1), `rule` (la regla que lo generó), `difference_amount`, `confirmed_by`, `confirmed_at`.

**`match_links`**, los vínculos n a n: `match_id`, `document_id` (nullable), `bank_movement_id` (nullable), `role` (`albaran` | `factura` | `abono` | `pago`), `amount_applied`.

**`match_line_results`**, el resultado línea a línea de un cruce albarán ↔ factura: `match_id`, `supply_item_id`, `description`, `qty_received`, `qty_invoiced`, `price_delivery`, `price_invoice`, `diff_qty`, `diff_amount`, `result` (`ok` | `dif_cantidad` | `dif_precio` | `facturado_no_recibido` | `recibido_no_facturado`).

**`incidents`**: `number` (INC-000123), `kind` (`dif_cantidad` | `dif_precio` | `dif_total` | `facturado_no_recibido` | `falta_albaran` | `falta_factura` | `pago_sin_factura` | `pago_doble` | `importe_pagado_distinto` | `subida_precio`), `supplier_id`, `location_id`, `amount`, `vat_at_stake` (para pagos sin factura), `status` (`abierta` | `en_revision` | `reclamo_preparado` | `aprobada` | `enviada` | `cerrada` | `descartada`), `claim_text`, `approved_by`, `approved_at`, `sent_by`, `sent_at`, `closed_reason` (`abono` | `factura_corregida` | `diferencia_aceptada` | `error_lectura` | `otro`), `closing_document_id`, `opened_at`, `closed_at`.

**`incident_links`**: `incident_id`, `document_id` / `bank_movement_id` / `match_id`.

**`price_history`**, que se registra al confirmar una factura (ERP: `item_cost_history`): `supply_item_id`, `supplier_id`, `document_id`, `doc_date`, `unit_price_base`, `variation_pct` (contra la compra anterior del mismo insumo y proveedor).

### Configuración

**`tenant_settings`**: `tolerance_qty` (por defecto 0), `tolerance_price_pct` (1), `tolerance_total_eur` (0,50), `tolerance_sum_eur` (0,05), `missing_invoice_days` (35), `price_alert_pct` (5), `payment_window_days` (10), `timezone` (`Europe/Madrid`).

**`mailboxes`**: `host`, `port`, `username`, `secret_ref` (nombre de la variable de entorno con la contraseña; nunca la contraseña en la base), `folder`, `last_uid`, `last_polled_at`, `active`.

## 7. Flujos y reglas

### 7.1 Captura (tablet)

1. El usuario `local` toca **Subir documento**, saca una o varias fotos, opcionalmente marca "Llegó con diferencias" con una nota, y confirma.
2. El servidor recibe las imágenes, las une en un PDF si son varias páginas, calcula el SHA-256 y, si ya existe, marca `duplicado` y avisa. Si no existe, lo sube a Storage en `{tenant}/{AAAA}/{MM}/{sha256}.{ext}` y crea el `source_file` en `pendiente_lectura` con el local de la membresía.
3. Antes de subir, el navegador hace un control de calidad básico (resolución mínima y nitidez) y pide repetir la foto si no pasa.

### 7.2 Lectura (`/api/cron/read`, cada minuto, hasta 5 archivos por corrida)

1. Toma los archivos `pendiente_lectura` y los marca `leyendo`. Después de 3 intentos fallidos, el archivo va a revisión.
2. Llama a Claude con las páginas y un esquema JSON: tipo, proveedor (nombre y NIF), número, fechas, albaranes citados, dirección de entrega, líneas, desglose de IVA, totales, y una confianza por campo. El prompt está versionado en `lib/reading/prompt.ts`.
3. Valida con `validateExtraction` (pura): suma de líneas = base (±`tolerance_sum_eur`), base + IVA + recargo = total, tipos de IVA válidos, NIF válido, fecha entre hoy −400 días y hoy +5, y duplicado por (proveedor, tipo, número).
4. Normaliza: proveedor por NIF y, si no hay, por nombre aproximado contra `legal_name`/`trade_name`. Las líneas se resuelven con las equivalencias y el local sale de la tablet o de la dirección de entrega.
5. Decide el destino: con todo validado y confianza alta, `read_status = leido` y pasa a cruces. Si no, `a_revisar` y se crea un `review_item`. Un archivo ilegible queda `rechazado` con su motivo (`borroso`, `cortado`, `faltan_paginas`, `mezclados`, `no_es_compra`, `pdf_danado`).
6. Un proveedor o una equivalencia desconocidos crean `review_items` (`proveedor_nuevo`, `equivalencia_nueva`). El documento no entra al cruce línea a línea hasta resolverlos, pero sí al cruce de pagos por total.

### 7.3 Importación del extracto (`operador`)

1. Basal sube el Excel de Santander. El parser (`lib/bank/santander.ts`, puro) busca la fila de encabezados ("Fecha Operación", "Concepto", "Importe", "Saldo") y no depende de una fila fija.
2. Cada fila genera un `row_hash`, y las que ya existen se cuentan como duplicadas y no se cargan.
3. `classifyMovement` (pura) deduce el tipo por el código y el texto ("Recibo …", "Transferencia A Favor De …", "Transaccion Contactless En …", "Compra …"), extrae la contraparte y la clasifica como `no_proveedor` si es sueldo, TGSS, impuesto, efectivo, Square u otro movimiento interno, con una lista configurable. Lo demás se busca contra `suppliers.bank_aliases`.

### 7.4 Cruce factura ↔ pago (`matchPayments`, pura)

Entradas: facturas y abonos con saldo pendiente, movimientos de proveedor sin conciliar y la configuración. Para cada movimiento:

1. **Exacto:** mismo proveedor, importe igual al total de una factura (±0,01 €) y fecha dentro de [`doc_date` − 3, `due_date` + `payment_window_days`]. Confianza 0,95.
2. **Agrupado:** mismo proveedor, y una combinación de 2 a 6 facturas pendientes que suma el importe (subset-sum acotado, priorizando las más antiguas). Confianza 0,85.
3. **Parcial:** el importe es menor que el de una única factura pendiente del proveedor. Se propone como pago parcial, con confianza 0,6.
4. **Tarjeta:** se cruza con `factura_simplificada` o `factura` del mismo proveedor y la misma fecha (±1 día) por importe exacto.
5. **Saldo a cuenta** (`has_prepaid_balance`): los pagos suman saldo y las facturas lo consumen. No se cruzan uno a uno.

Con confianza ≥ 0,9 el cruce se confirma solo; entre 0,6 y 0,9 queda `propuesto` y va a revisión; por debajo de 0,6, el movimiento queda `sin_factura`. Un movimiento de proveedor sin factura a los 7 días genera la incidencia `pago_sin_factura`, con `vat_at_stake` estimado al 10% si no se conoce el tipo. Un movimiento que coincide con una factura ya pagada genera `pago_doble`.

`payment_status` se recalcula desde los vínculos. `vencida` se calcula al consultar, nunca se guarda.

### 7.5 Cruce albarán ↔ factura (`matchDeliveries`, pura)

1. **Candidatos:** los albaranes citados en `referenced_delivery_numbers`. Si la factura no cita ninguno, los albaranes del mismo proveedor sin factura entre la fecha del albarán más antiguo probable y `doc_date`, sin límite fijo de días. Si un número citado no existe, se crea la incidencia `falta_albaran` y se cruza con los que sí están.
2. **Agregación:** se suman las líneas de los albaranes por `supply_item_id` (o por `supplier_code` o descripción normalizada si no hay equivalencia), en unidad base.
3. **Comparación por insumo:** cantidad (±`tolerance_qty`) y precio unitario base (±`tolerance_price_pct`, solo si el albarán trae precio). Insumo facturado sin albarán → `facturado_no_recibido`. Insumo recibido no facturado → `recibido_no_facturado`, que se informa pero no genera reclamo.
4. **Valorización:** las diferencias de cantidad se valorizan con el precio de la factura.
5. **Total:** la factura contra la suma de los albaranes valorizados (±`tolerance_total_eur`).
6. **Resultado:** sin diferencias, `conciliado`. Si hay alguna, `con_diferencia` y una incidencia por cada tipo, con su importe. Si no hay ningún albarán candidato, `falta_albaran`.
7. Las notas del local ("Llegó con diferencias") se muestran junto al cruce como evidencia.

Un albarán sin factura a los `missing_invoice_days` días genera `falta_factura` (cron diario).

### 7.6 Alerta de precio

Al confirmar el cruce de una factura (o al leerla, si el proveedor no manda albaranes), se registra `price_history` por línea con equivalencia. Una variación mayor que `price_alert_pct` contra la compra anterior del mismo insumo y proveedor crea la incidencia `subida_precio`, de tipo aviso.

### 7.7 Incidencias

Ciclo: `abierta` → `en_revision` (Basal confirma o descarta como `error_lectura`) → `reclamo_preparado` → `aprobada` (solo `dueno`) → `enviada` (registra quién y cuándo) → `cerrada`, por abono vinculado, factura corregida o diferencia aceptada por el Dueño. Un abono leído se propone automáticamente para cerrar las incidencias abiertas del mismo proveedor con importe compatible. La v1 no envía mails: el texto del reclamo se copia y se manda por fuera.

### 7.8 Buzón (`/api/cron/mailbox`, cada 30 minutos)

Se conecta por IMAP con `imapflow`, abre la carpeta en modo solo lectura (EXAMINE), trae los mensajes con UID mayor que `last_uid`, descarta los remitentes internos y crea un `source_file` con `origin = mail` por cada adjunto PDF o imagen. Nunca marca, mueve ni borra mails.

## 8. Pantallas

Rutas bajo `app/(main)/`:

| Ruta | Perfil | Contenido |
|---|---|---|
| `/capturar` | local (pantalla de inicio de la tablet) | Botón "Subir documento", cámara con varias páginas, "Llegó con diferencias" con nota, y la lista de lo subido hoy con su estado |
| `/` | dueno, consulta, operador | Tarjetas de diferencias abiertas, pagos sin factura con el IVA en juego, facturas que vencen en 7 días y pendientes de revisión, con filtros por local y período |
| `/documentos` y `/documentos/[id]` | todos menos local | Lista con filtros, y el detalle con la foto a la izquierda y datos, líneas, vínculos, incidencias e historial a la derecha |
| `/revision` | operador | La bandeja de `review_items`, con el campo dudoso resaltado sobre la foto |
| `/cruces` | todos menos local | Pestañas Pendientes / Con diferencia / Conciliadas, con la tabla línea a línea por factura |
| `/pagos` | todos menos local | Movimientos con su factura y estado, "Facturas a pedir" por proveedor, pendientes y vencidas |
| `/incidencias` | operador, dueno | Lista y detalle. El Dueño aprueba desde acá |
| `/proveedores` | todos menos local | Por proveedor y mes: albaranes, facturado, diferencia, pagado y pendiente, más la evolución de precios |
| `/configuracion/*` | operador | Locales, proveedores, insumos, equivalencias, cuentas bancarias, tolerancias, buzón y usuarios |

Todas las tablas exportan a Excel con `exceljs`, igual que el ERP.

## 9. Variables de entorno

Solo se documentan con valores de ejemplo en `.env.example`; nunca van al repositorio. Son `DATABASE_URL`, `DIRECT_URL`, `RAW_DATABASE_URL` (solo tests), `NEXTAUTH_SECRET`, `NEXTAUTH_URL`, `CRON_SECRET`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL`, `MAILBOX_NATIF_PASSWORD`, `ADMIN_PROVISION_SECRET`.

## 10. Testing

- **Funciones puras con casos exhaustivos:** `validateExtraction`, `nif`, `santander` (con un extracto real anonimizado), `classifyMovement`, `matchPayments` (exacto, agrupado, parcial, tarjeta, saldo a cuenta, pago doble), `matchDeliveries` (con y sin referencias, sin precio, unidades distintas, facturado no recibido, tolerancias en el límite) y `priceVariation`.
- **Integración contra Postgres real:** RLS entre dos empresas en cada tabla nueva, captura con duplicado, lectura simulada (el cliente de Claude se inyecta como proveedor), importación de extracto con duplicados y el ciclo completo de una incidencia.
- **Precisión de lectura:** un script que corre la lectura sobre la muestra de 20 documentos de octubre y la compara campo por campo contra un JSON verificado a mano. Mide precisión de cabecera, de líneas, el porcentaje sin revisión y el costo por documento.

## 11. Integración futura con basal-erp

| Conciliación | basal-erp |
|---|---|
| `locations` | `locations` (sucursales) |
| `suppliers` | `suppliers` (agregar `tax_id` genérico por país y `bank_aliases`) |
| `supply_items` | `items` de tipo insumo |
| `product_equivalences` | `item_purchase_units` (agregar proveedor y código de proveedor) |
| `documents` de tipo `factura` conciliada | `purchases` confirmada. El modo "solo control" no mueve stock mientras la empresa no use inventario |
| `documents` de tipo `albaran` | Recepción vinculada a la compra (nuevo en el ERP: "remito y factura separados", que la especificación de compras dejó como futuro) |
| `abono` | `supplier_credit_notes` |
| `bank_movements` + `matches` de pago | `treasury_movements` + `supplier_payments` / `supplier_allocations` |
| `price_history` | `item_cost_history` |
| `incidents` | Tabla nueva en el ERP |

Pendientes para la integración: la configuración de país por empresa en el ERP (España/Argentina), la conversión `Decimal` ↔ `Float` y el tipo de comprobante español en `DOCUMENT_TYPES`.

## 12. Entregas

| # | Entrega | Qué se puede probar al terminar |
|---|---|---|
| 1 | Base copiada del ERP, Natif como empresa, usuarios y perfiles, locales, deploy en Vercel | Iniciar sesión, la tablet entra directo a `/capturar` |
| 2 | Captura, Storage y lista de documentos | Subir fotos desde la tablet y verlas en `/documentos` |
| 3 | Lectura con IA, validación, bandeja de revisión, maestros de proveedores, insumos y equivalencias | Ver los datos leídos y corregir en la bandeja; medir la precisión con la muestra de octubre |
| 4 | Importación del extracto de Santander y cruce factura ↔ pago | Cargar septiembre y octubre y ver los pagos sin factura |
| 5 | Cruce albarán ↔ factura e incidencias con aprobación | El ciclo completo de una diferencia, de la detección al cierre |
| 6 | Inicio, proveedores, alertas de precio y exportaciones | El tablero para Robbie y Juampo |
| 7 | Buzón IMAP de info@natif.es | Las facturas por mail entran solas |

**Fuera de la v1:** WhatsApp, conexión bancaria directa (PSD2), cuenta PGC por factura para el cierre mensual, albaranes internos entre locales, escandallos con precio real, proyección de caja, pedidos y P&G por local.

## 13. Preguntas abiertas

1. Quién envía el reclamo al proveedor: el Dueño, o Basal en nombre de Natif.
2. La credencial IMAP de solo lectura de info@natif.es (GoDaddy).
3. Confirmar que las tablets no son la caja de Square.
4. Cómo funciona el saldo a cuenta con Euceco.
5. Validar las tolerancias iniciales (§6, `tenant_settings`).

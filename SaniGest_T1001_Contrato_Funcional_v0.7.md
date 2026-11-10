# SaniGest — Contrato funcional y mapa de APIs (v0.7)

**Proyecto:** SaniGest Backend  
**Ticket Jira:** `S2-293` (referencia académica `T1001`)  
**Estado:** REVISADO Y ALINEADO CON EL BACKLOG. Reglas funcionales aprobadas; contratos REST preliminares sujetos a OpenAPI durante la implementación.  
**Fuente:** 12 pantallas del prototipo, contrato v0.6 y refinamiento del backlog del 9 de noviembre de 2026.  
**Alcance:** backend; el frontend de las capturas es referencia funcional y se desarrolla aparte.  
**Horizonte:** 6 sprints de 2 semanas y estabilización.

> Este documento define comportamiento y contratos objetivo; no afirma que los endpoints estén implementados.

## 1. Decisiones aprobadas

1. Un usuario regular pertenece como máximo a una empresa; una empresa puede tener varios usuarios.
2. Un asesor de SaniGest usa el rol global `ADVISOR`. No es empleado ni miembro de las empresas y solo accede con autorización explícita, revocable y auditada.
3. Los roles empresariales son `OWNER`, `ADMIN`, `OPERATOR` y `VIEWER`.
4. La factura es un documento interno, no fiscal, ligado a una venta. Se genera manualmente y no usa NCF, e-CF ni servicios de la DGII.
5. Registrar una venta no registra dinero. `CASH` significa vencimiento inmediato; la venta solo queda pagada después de un cobro verificable.
6. Un cobro genera un ingreso de caja y un pago genera un egreso. También existen movimientos manuales independientes.
7. Cada microservicio posee sus datos. Ningún servicio comparte tablas o entidades JPA con otro.
8. Finance obtiene datos externos mediante composición HTTP temporal o proyecciones locales por eventos; nunca consulta bases de datos ajenas.
9. El despliegue AWS del MVP usa ECR, ECS/Fargate y RDS PostgreSQL. Kubernetes con kind es una ampliación local; EKS no forma parte del MVP.

## 2. Mapa de pantallas

| Pantalla | Servicio principal | Recursos y operaciones |
| --- | --- | --- |
| Inicio | Finance | resumen, actividad mensual, facturas recientes proyectadas |
| Empresas y usuarios | Company | empresa propia, miembros, roles, invitaciones, acceso ADVISOR |
| Clientes | Catalog | alta, detalle, búsqueda, actualización y desactivación |
| Proveedores | Catalog | alta, detalle, búsqueda, actualización y desactivación |
| Productos y servicios | Catalog | catálogo PRODUCT/SERVICE, precio y estado |
| Ventas | Sales | venta, líneas, búsqueda, detalle y cancelación elegible |
| Facturación | Sales | ventas elegibles, emisión manual, listado y detalle interno |
| Compras y gastos | Expense | PURCHASE/EXPENSE, búsqueda, detalle, corrección y cancelación |
| Cuentas por cobrar | Sales | saldos, vencimientos, abonos y saldo inicial |
| Cuentas por pagar | Expense | obligaciones, vencimientos, pagos y saldo inicial |
| Ingresos y egresos | Finance | movimientos manuales y automáticos trazables |
| Reportes financieros | Finance | resumen, tendencias, operaciones y saldos a fecha de corte |

## 3. Propiedad de datos y comunicación

| Servicio | Fuente de verdad | Dependencias permitidas |
| --- | --- | --- |
| Identity | usuarios, credenciales, sesiones, refresh tokens, rol global `ADVISOR` | Ninguna tabla externa |
| Company | empresas, membresías, roles, invitaciones, permisos empresariales y autorizaciones ADVISOR | Identidad por `userId`; no almacena contraseñas |
| Catalog | clientes, proveedores, productos y servicios | Contexto empresarial autorizado |
| Sales | ventas, facturas internas, cuentas por cobrar y cobros | Catalog mediante API; eventos hacia Finance |
| Expense | compras/gastos, cuentas por pagar y pagos | Catalog mediante API; eventos hacia Finance |
| Finance | movimientos de caja y proyecciones de dashboard/reportes | Eventos o composición HTTP temporal |

### 3.1 Reglas de integración

- Cada servicio usa su propia base o esquema aislado y migraciones Flyway propias.
- No se comparten entidades JPA, repositorios ni reglas de dominio.
- Las llamadas HTTP tienen timeout, fallan cerrado y aplican circuit breaker cuando corresponde.
- La creación de empresa y membresía `OWNER` se resuelve como transacción local en Company. Identity aporta el `userId`; no participa en una transacción distribuida.
- Las respuestas agregadas indican período y fecha de actualización cuando usan consistencia eventual.
- Una autorización no se obtiene de un `companyId` enviado por el cliente; se verifica identidad, membresía, rol y permiso.

## 4. Reglas funcionales

### 4.1 Empresas, usuarios e invitaciones

- El RNC es opcional, se normaliza a dígitos y, cuando se informa, debe ser único conforme a la política dominicana definida para el MVP. No se consulta la DGII.
- `OWNER` administra miembros y acceso ADVISOR. `ADMIN` administra miembros salvo OWNER; `OPERATOR` opera módulos permitidos; `VIEWER` solo consulta.
- Una invitación se crea por email, expira, es de un solo uso y puede revocarse o reenviarse. Un token anterior no vuelve a ser válido.
- El invitado puede completar su registro antes de aceptar, pero no puede pertenecer ya a otra empresa.
- El acceso ADVISOR es de lectura por defecto, configurable por módulo y con escritura únicamente mediante concesión explícita.

### 4.2 Catálogos

- Clientes y proveedores se crean, consultan, buscan, actualizan y desactivan; no se borran si tienen historial.
- Productos y servicios comparten catálogo con tipo `PRODUCT` o `SERVICE`, precio DOP y estado.
- No hay inventario avanzado, múltiples monedas, impuestos ni descuentos en el MVP.

### 4.3 Ventas, facturas y cobros

- Sales valida cliente e ítems con Catalog y conserva snapshot de nombre, descripción y precio.
- El servidor calcula cantidades y totales con `BigDecimal`.
- Una venta `CREDIT` crea cuenta por cobrar; una venta `CASH` tiene vencimiento inmediato, pero no queda pagada sin cobro.
- La venta y el cobro son operaciones separadas e idempotentes.
- La factura se emite manualmente desde una venta elegible; existe una sola factura vigente por venta.
- El número `FAC-AAAA-NNN` es único por empresa/año y se asigna transaccionalmente bajo concurrencia.
- Emitir factura no genera cobro, ingreso ni movimiento de caja.
- El estado `PENDING`, `PARTIALLY_PAID` o `PAID` se deriva de cobros verificados; una reversión recalcula el estado.
- Un saldo inicial por cobrar usa origen `OPENING_BALANCE`; no crea venta, factura ni ingreso.

### 4.4 Compras, gastos, pagos y caja

- Expense registra `PURCHASE` o `EXPENSE`, proveedor opcional, importe, fecha y condición de pago.
- Una operación a crédito crea cuenta por pagar.
- Una obligación inicial usa origen `OPENING_BALANCE`; no crea compra/gasto ni egreso.
- Pagos y cobros parciales no pueden superar el saldo y usan control de concurrencia.
- Un cobro/pago confirmado publica un evento mediante outbox. Finance crea exactamente un movimiento lógico mediante clave `(companyId, origin, sourceId)`.
- Un movimiento manual `INCOME` o `OUTFLOW` no modifica ventas, cuentas por cobrar ni cuentas por pagar.
- Operaciones económicas no se borran físicamente; se cancelan o revierten con motivo, actor y auditoría.

### 4.5 Dashboard y reportes

- Las tarjetas muestran ventas registradas, compras/gastos, saldo por cobrar y saldo por pagar.
- Las gráficas muestran ingresos y egresos de caja, no ventas y gastos devengados.
- Finance no lee tablas de Sales ni Expense. Usa proyecciones por empresa o, temporalmente, composición HTTP documentada.
- Reprocesar eventos no duplica importes. Debe existir reconciliación y reconstrucción de proyecciones.
- Los reportes son administrativos, no estados financieros certificados.
- Las notificaciones in-app son una ampliación; no se requieren correos automáticos en el MVP.

## 5. Contratos REST propuestos

Todas las rutas empresariales requieren JWT y autorización en el Gateway y en el servicio propietario. Los listados aceptan `page`, `size`, `sort` y filtros contextuales.

### 5.1 Identity Service

| Método | Ruta | Resultado |
| --- | --- | --- |
| POST | `/api/v1/auth/register` | Crear cuenta regular sin empresa inicial |
| POST | `/api/v1/auth/login` | Autenticar y emitir sesión/tokens |
| POST | `/api/v1/auth/refresh` | Rotar refresh token y renovar acceso |
| POST | `/api/v1/auth/logout` | Revocar la sesión actual |
| GET | `/api/v1/auth/me` | Identidad y rol global |

Los refresh tokens se almacenan mediante hash, rotan en cada uso, detectan reutilización y se invalidan al cerrar sesión o desactivar al usuario.

### 5.2 Company Service

| Método | Ruta | Resultado |
| --- | --- | --- |
| POST | `/api/v1/companies` | Crear empresa y OWNER autenticado |
| GET/PATCH | `/api/v1/companies/me` | Consultar o modificar empresa propia |
| GET | `/api/v1/companies/me/users` | Listar miembros |
| POST | `/api/v1/companies/me/invitations` | Crear invitación |
| POST | `/api/v1/invitations/{token}/accept` | Aceptar invitación |
| PATCH | `/api/v1/companies/me/users/{userId}/role` | Cambiar rol permitido |
| PATCH | `/api/v1/companies/me/users/{userId}/status` | Activar/desactivar miembro |
| GET/PUT/DELETE | `/api/v1/companies/me/advisor-access` | Consultar, conceder o revocar acceso ADVISOR |
| GET | `/api/v1/advisor/companies` | Empresas autorizadas al asesor |

### 5.3 Catalog Service

| Recurso | Rutas principales |
| --- | --- |
| Clientes | `POST/GET /api/v1/clients`, `GET/PATCH /api/v1/clients/{id}`, `PATCH /api/v1/clients/{id}/status` |
| Proveedores | `POST/GET /api/v1/suppliers`, `GET/PATCH /api/v1/suppliers/{id}`, `PATCH /api/v1/suppliers/{id}/status` |
| Catálogo | `POST/GET /api/v1/catalog-items`, `GET/PATCH /api/v1/catalog-items/{id}`, `PATCH /api/v1/catalog-items/{id}/status` |

### 5.4 Sales Service

| Método | Ruta | Resultado |
| --- | --- | --- |
| POST/GET | `/api/v1/sales` | Crear o listar ventas |
| GET | `/api/v1/sales/{id}` | Detalle y cobros |
| POST | `/api/v1/sales/{id}/cancel` | Cancelar venta elegible |
| GET | `/api/v1/sales?invoiceEligible=true` | Ventas facturables |
| POST | `/api/v1/sales/{saleId}/invoices` | Emitir factura interna idempotente |
| GET | `/api/v1/invoices` | Buscar facturas |
| GET | `/api/v1/invoices/{id}` | Detalle interno no fiscal |
| POST | `/api/v1/receivables` | Crear saldo inicial |
| GET | `/api/v1/receivables` | Consultar cartera |
| GET | `/api/v1/receivables/{id}` | Detalle e historial |
| POST | `/api/v1/receivables/{id}/payments` | Registrar abono idempotente |

### 5.5 Expense Service

| Método | Ruta | Resultado |
| --- | --- | --- |
| POST/GET | `/api/v1/expenses` | Crear o listar compras/gastos |
| GET/PATCH | `/api/v1/expenses/{id}` | Consultar o corregir operación elegible |
| POST | `/api/v1/expenses/{id}/cancel` | Cancelar operación elegible |
| POST | `/api/v1/payables` | Crear obligación inicial |
| GET | `/api/v1/payables` | Consultar obligaciones |
| GET | `/api/v1/payables/{id}` | Detalle e historial |
| POST | `/api/v1/payables/{id}/payments` | Registrar pago idempotente |

### 5.6 Finance Service

| Método | Ruta | Resultado |
| --- | --- | --- |
| POST/GET | `/api/v1/cash-movements` | Crear movimiento manual o listar movimientos |
| GET | `/api/v1/cash-movements/{id}` | Detalle y origen |
| GET | `/api/v1/dashboard/summary?from=&to=` | Tarjetas con período/corte |
| GET | `/api/v1/dashboard/activity?from=&to=&groupBy=MONTH` | Ingresos y egresos por período |
| GET | `/api/v1/dashboard/recent-invoices?limit=5` | Proyección explícita de facturas recientes |
| GET | `/api/v1/reports/summary?from=&to=` | Operaciones, caja y saldos diferenciados |
| GET | `/api/v1/reports/monthly-activity?from=&to=` | Tendencia de caja |
| GET | `/api/v1/reports/receivables-payables?asOf=` | Saldos a fecha de corte |
| GET | `/api/v1/reports/operations?from=&to=&type=` | Indicadores operativos |

## 6. Convenciones transversales

- **Identificadores:** UUID.
- **Importes:** `BigDecimal`, PostgreSQL `NUMERIC` y cadenas decimales en JSON; moneda `DOP`.
- **Tiempo:** `LocalDate` para fechas empresariales sin hora; `Instant` UTC para eventos, sesiones y auditoría. Los cortes usan `America/Santo_Domingo`.
- **Estados:** catálogos `ACTIVE/INACTIVE`; obligaciones `PENDING/PARTIALLY_PAID/PAID`. `OVERDUE` se deriva de saldo y vencimiento, no se edita manualmente.
- **Errores:** `application/problem+json` RFC 9457; 400, 401, 403, 404, 409 y 422 según el caso.
- **Paginación:** `page=0`, `size=20`, `sort=campo,dirección`, límite máximo configurable y metadatos de total.
- **Idempotencia:** `Idempotency-Key` en ventas, facturas, saldos iniciales, cobros y pagos. Misma clave/cuerpo devuelve el mismo resultado; misma clave con otro cuerpo devuelve 409.
- **Auditoría:** empresas, membresías, permisos ADVISOR, ventas, facturas, cobros, pagos, movimientos, cancelaciones y reversiones.
- **Seguridad:** validación de issuer, audience, firma y expiración; controles de método y tenant en cada servicio; fallar cerrado.

## 7. Eventos y confiabilidad

Eventos iniciales:

- `ReceivablePaymentRegisteredV1`
- `PayablePaymentRegisteredV1`
- Eventos V1 de reversión cuando se implementen esos flujos

Cada evento contiene como mínimo `eventId`, `eventType`, `version`, `occurredAt`, `companyId`, `sourceId`, `amount` y `currency`. No incluye entidades JPA, credenciales ni datos personales innecesarios.

- Productores: transacción local + transactional outbox.
- Consumidores: deduplicación persistente e idempotencia.
- Operación: reintentos limitados con backoff, DLQ, replay controlado, métricas de lag y runbook.
- Reconciliación: detectar cobros/pagos sin movimiento y movimientos huérfanos.
- No se promete exactamente-una-vez entre PostgreSQL y Kafka; se garantiza un único efecto lógico mediante outbox e idempotencia.

## 8. Ejemplos esenciales

### 8.1 Emitir factura interna

```http
POST /api/v1/sales/{saleId}/invoices
Authorization: Bearer <token>
Idempotency-Key: <uuid>
```

Resultado: `201 Created`; un reintento idéntico devuelve la misma factura. La emisión no crea cobro ni movimiento de caja.

### 8.2 Registrar abono

```http
POST /api/v1/receivables/{id}/payments
Authorization: Bearer <token>
Idempotency-Key: <uuid>
Content-Type: application/json

{
  "amount": "8000.00",
  "paidAt": "2026-11-09",
  "method": "TRANSFER",
  "reference": "TRX-100"
}
```

Resultado: el saldo disminuye una vez; Sales publica un evento por outbox y Finance registra un único ingreso lógico. La proyección puede actualizarse con retardo breve.

### 8.3 Movimiento manual

```json
{
  "type": "INCOME",
  "category": "CAPITAL_CONTRIBUTION",
  "amount": "20000.00",
  "occurredAt": "2026-11-09",
  "description": "Aporte del propietario"
}
```

No modifica ventas, cobros, cuentas por cobrar ni cuentas por pagar.

## 9. Plan de implementación actualizado

| Sprint | Resultado demostrable |
| --- | --- |
| 1 | Contrato y ownership; monorepo; Company ejecutable; PostgreSQL/Flyway; Testcontainers; CI. Ocho tickets, 24 puntos. |
| 2 | Identity, sesiones, empresa única, miembros, roles y permisos básicos. |
| 3 | Catalog, Gateway, primera API protegida y despliegue AWS inicial. |
| 4 | Sales, factura interna, cuentas por cobrar, cobros e idempotencia. |
| 5 | Expense, cuentas por pagar, pagos y movimientos manuales. |
| 6 | Proyecciones, dashboard/reportes reales, Kafka, resiliencia y observabilidad según capacidad. |

Swagger/OpenAPI se incorpora con las primeras APIs de negocio; ArchUnit y Kubernetes/kind permanecen en backlog como ampliaciones. Pruebas, seguridad multiempresa, documentación y CI se trabajan transversalmente.

## 10. Criterios de aceptación de S2-293 / T1001

- [x] Las 12 pantallas tienen recursos, endpoints y servicio propietario.
- [x] Se distinguen operaciones registradas, documentos, obligaciones y movimientos de caja.
- [x] Se documentan usuario de empresa única, ADVISOR, roles e invitaciones.
- [x] Se define ownership de Identity, Company, Catalog, Sales, Expense y Finance.
- [x] Se define composición/proyección de dashboard sin compartir bases de datos.
- [x] Se aclara venta CASH, factura manual y estado derivado de cobros.
- [x] Se definen idempotencia, outbox, DLQ, replay y reconciliación.
- [x] Se alinea el plan del Sprint 1 con Jira.
- [x] Se documenta la equivalencia `T1001 = S2-293` y `T1002 = S2-294`.

## 11. Resultado

El contrato funcional v0.7 está listo para revisión final del equipo. `S2-293` puede cerrarse cuando este archivo se incorpore al repositorio y se confirme que no quedan decisiones de negocio pendientes.

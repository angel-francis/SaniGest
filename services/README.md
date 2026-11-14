# Servicios

Cada subdirectorio representa un microservicio desplegable y dueño exclusivo de
su modelo, persistencia y migraciones.

Topología prevista según el contrato funcional y ADR-001:

| Directorio | Responsabilidad | Ticket de implementación |
| --- | --- | --- |
| `api-gateway` | Entrada pública y enrutamiento | `S2-316` |
| `identity-service` | Identidad, autenticación y sesiones | `S2-304` |
| `company-service` | Empresas, membresías y permisos | `S2-295` |
| `catalog-service` | Clientes, proveedores y catálogo | `S2-318` |
| `sales-service` | Ventas, facturación y cuentas por cobrar | `S2-331` |
| `expense-service` | Compras, gastos y cuentas por pagar | `S2-342` |
| `finance-service` | Caja, dashboard y reportes | `S2-349` |

`S2-294` reserva la estructura completa, pero solamente `company-service` es un
módulo Maven compilable en esta etapa. Cada directorio reservado se incorpora al
reactor raíz cuando su ticket entregue al menos un esqueleto ejecutable y sus
pruebas básicas.

Notificaciones no constituyen un microservicio separado en el MVP. La auditoría
se conserva junto al servicio propietario de cada operación; cualquier cambio a
estos límites requiere actualizar el ADR.

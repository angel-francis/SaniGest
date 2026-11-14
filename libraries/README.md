# Librerías compartidas

Este directorio permanece sin módulos hasta que una necesidad esté demostrada
por al menos dos consumidores.

Se permiten contratos técnicos versionados y utilidades pequeñas, por ejemplo
`common-web`, `common-security` o `common-events`. Se prohíben entidades JPA,
repositorios, DTO de dominio y reglas de negocio compartidas entre servicios,
conforme al ADR-001 de `S2-303`.

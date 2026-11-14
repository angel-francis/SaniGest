# Workflows

Los workflows de GitHub Actions se agregarán en sus tickets específicos:

- `S2-301`: compilación y pruebas con `mvn -B verify`;
- `S2-327`: autenticación AWS mediante OIDC;
- `S2-364`: publicación y despliegue de servicios;
- `S2-393`: `terraform plan` y `terraform apply` protegidos.

No se agrega un workflow vacío en `S2-294` para evitar una falsa señal de CI.

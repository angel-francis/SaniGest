# SaniGest

Backend de SaniGest basado en microservicios con Java, Spring Boot y PostgreSQL.

## Documentación arquitectónica

Los siguientes documentos se están incorporando mediante los PRs de `S2-293` y
`S2-303`; sus rutas se consideran parte estable de la estructura del monorepo:

- [Contrato funcional y mapa de APIs v0.7](SaniGest_T1001_Contrato_Funcional_v0.7.md)
- [ADR-001: propietarios de datos y comunicación entre servicios](docs/adr/ADR-001-propiedad-datos-comunicacion-servicios.md)

El ADR-001 es la referencia obligatoria para definir nuevas historias, tablas,
integraciones síncronas y eventos entre servicios. Ningún módulo puede consultar
las tablas de otro servicio ni compartir sus entidades JPA.

## Requisitos

- Java 21
- Maven 3.6.3 o posterior
- Docker 24 o posterior para dependencias locales y pruebas de integración

Comprueba las versiones instaladas:

```bash
java -version
mvn -version
docker --version
```

## Estructura del monorepo

```text
.
├── services/               Microservicios desplegables e independientes
│   └── company-service/    Módulo inicial de Company
├── libraries/              Contratos técnicos compartidos ya demostrados
├── infrastructure/         Terraform, Docker y despliegue
├── docs/                   ADRs y documentación técnica
└── .github/workflows/      Automatización de CI/CD
```

Cada microservicio mantiene su modelo de dominio, persistencia, migraciones y
configuración. `libraries` no admite entidades JPA, repositorios ni reglas de
negocio compartidas.

## Compilar y probar

Desde la raíz:

```bash
mvn -B verify
```

Para reconstruir desde cero:

```bash
mvn -B clean verify
```

Si Maven no está instalado, se puede ejecutar la misma verificación con Docker:

```bash
docker run --rm \
  --user "$(id -u):$(id -g)" \
  -v "$PWD:/workspace" \
  -v "$HOME/.m2:/var/maven/.m2" \
  -w /workspace \
  -e MAVEN_CONFIG=/var/maven/.m2 \
  maven:3.9.11-eclipse-temurin-21 mvn -B verify
```

## Validar el arranque de Company Service

El módulo inicial carga el contexto Spring Boot sin depender de otros servicios:

```bash
mvn -pl services/company-service -am spring-boot:run
```

Spring Web, el puerto HTTP, los perfiles, Spring Data JPA, PostgreSQL y Actuator
se incorporan en `S2-295` y tickets posteriores.

## Convenciones

- Nombres de módulos y artefactos en `kebab-case`.
- Paquetes Java bajo `com.sanigest`.
- Java compilado con `--release 21`.
- Dependencias Spring administradas por los BOM de Spring Boot y Spring Cloud.
- Versiones `SNAPSHOT` únicamente para módulos propios durante el desarrollo.
- Una librería común se extrae solo después de existir al menos dos consumidores
  y nunca contiene lógica de dominio de un servicio.

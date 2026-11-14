# Infraestructura

Este directorio contendrá la infraestructura reproducible de SaniGest:

- Docker y Docker Compose para desarrollo local;
- Terraform para AWS;
- manifiestos Kubernetes con `kind` como ampliación local.

El MVP se despliega en AWS con ECR, ECS/Fargate y RDS PostgreSQL. EKS y MSK no se
crean por defecto. Los módulos Terraform se implementan en los tickets
`S2-390` a `S2-394`.

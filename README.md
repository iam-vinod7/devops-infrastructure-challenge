# DevOps Engineer Infrastructure Challenge

Minimal production-style Kubernetes application demonstrating:

- Docker containerization
- Kubernetes deployment
- PostgreSQL dependency
- Readiness and liveness probes
- CI/CD automation
- Intentional failure simulation
- Kubernetes debugging

## Architecture

GitHub → CI/CD → Docker Image → Kubernetes

Application:

Flask Backend → PostgreSQL

## Kubernetes

Namespace:

devops-demo

Services:

- backend: NodePort 30080
- postgres: ClusterIP 5432

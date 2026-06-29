# InnoTech Devops — Despliegue en AWS con CI/CD

## Descripción

Sistema distribuido compuesto por dos microservicios Spring Boot y un frontend React, desplegado en AWS ECS con Fargate mediante un pipeline CI/CD automatizado con GitHub Actions.

## Arquitectura

    Internet → ALB (puerto 80/8080/8081)
                  ├── frontend-service  (puerto 80)  → ECR innovatech-frontend
                  ├── ventas-service    (puerto 8080) → ECR innovatech-ventas
                  └── despachos-service (puerto 8081) → ECR innovatech-despachos
                            ↓
                      RDS MySQL (innovatech-db)

## Servicios

| Servicio | Stack | Puerto |
|----------|-------|--------|
| API Ventas | Spring Boot 3.4.4 + JPA | 8080 |
| API Despachos | Spring Boot 3.4.4 + JPA | 8081 |
| Frontend | React 18 + Vite + Tailwind | 80 |

## Infraestructura AWS

### Red

- **VPC**: `innovatech-vpc` (10.0.0.0/16)
- **Subnets públicas**: 2 (us-east-1a, us-east-1b) — para el ALB
- **Subnets privadas**: 2 (us-east-1a, us-east-1b) — para los contenedores ECS
- **NAT Gateway**: `innovatech-nat` (Public, Zonal) ubicado en `innovatech-subnet-public1-us-east-1a`. Permite que los contenedores ECS en subnets privadas salgan a internet para descargar imágenes desde ECR y enviar logs a CloudWatch.

### Security Groups

| Nombre | Permite |
|--------|---------|
| `innovatech-alb-sg` | HTTP 80, 8080, 8081 desde Internet |
| `innovatech-ecs-sg` | 80, 8080, 8081 desde el ALB |
| `innovatech-rds-sg` | MySQL 3306 desde ECS |

### Roles IAM

Se utiliza el rol `LabRole` de AWS Academy tanto como Task Role y Task Execution Role. Este rol incluye permisos para ECR, ECS, CloudWatch y RDS.

### Base de datos

- **Motor**: MySQL 8.4.8 en Amazon RDS
- **Instancia**: db.t3.micro
- **Endpoint**: `innovatech-db.coqjn4hvzksx.us-east-1.rds.amazonaws.com`
- Las credenciales se inyectan como variables de entorno en las Task Definitions de ECS.

### ECS Cluster

- **Cluster**: `innovatech-cluster` (Fargate)
- **Task Definitions**: `ventas-task`, `despachos-task`, `frontend-task`
- **Servicios**: `ventas-service`, `despachos-service`, `frontend-service`

### Autoscaling

Configurado con Target Tracking en cada servicio ECS:

| Parámetro | Valor |
|-----------|-------|
| Métrica | ECSServiceAverageCPUUtilization |
| Valor objetivo | 50% |
| Mínimo de tareas | 1 |
| Máximo de tareas | 3 |

**Justificación**: Al superar el 50% de CPU se escala horizontalmente para mantener disponibilidad sin desperdiciar recursos en períodos de baja carga.

### Load Balancer

- **Tipo**: Application Load Balancer (internet-facing)
- **DNS**: `innovatech-alb-127900156.us-east-1.elb.amazonaws.com`

| Listener | Target Group | Servicio |
|----------|-------------|---------|
| HTTP:80 | innovatech-frontend-tg | Frontend |
| HTTP:8080 | innovatech-ventas-tg | API Ventas |
| HTTP:8081 | innovatech-despachos-tg | API Despachos |

### Logs

Los logs de cada contenedor se envían automáticamente a **Amazon CloudWatch Logs** mediante el log driver `awslogs` configurado en cada Task Definition.

## Pipeline CI/CD

### Flujo

    git push → GitHub Actions → Build → Test → Docker Build → Push ECR → Deploy ECS

### Workflows

| Archivo | Trigger | Servicio |
|---------|---------|---------|
| `ci-cd-ventas.yml` | Push en `back-ventas-springboot/**` | API Ventas |
| `ci-cd-despachos.yml` | Push en `back-bespachos-springboot/**` | API Despachos |
| `ci-cd-frontend.yml` | Push en `front-despacho/**` | Frontend |

### Secrets requeridos en GitHub

| Secret | Descripción |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | Credencial AWS Academy |
| `AWS_SECRET_ACCESS_KEY` | Credencial AWS Academy |
| `AWS_SESSION_TOKEN` | Token de sesión AWS Academy |
| `VITE_API_VENTAS` | URL pública API Ventas |
| `VITE_API_DESPACHOS` | URL pública API Despachos |

## Acceso público

| Recurso | URL |
|---------|-----|
| Frontend | http://innovatech-alb-127900156.us-east-1.elb.amazonaws.com |
| API Ventas | http://innovatech-alb-127900156.us-east-1.elb.amazonaws.com:8080/api/v1/ventas |
| API Despachos | http://innovatech-alb-127900156.us-east-1.elb.amazonaws.com:8081/api/v1/despachos |
| Swagger Ventas | http://innovatech-alb-127900156.us-east-1.elb.amazonaws.com:8080/swagger-ui.html |
| Swagger Despachos | http://innovatech-alb-127900156.us-east-1.elb.amazonaws.com:8081/swagger-ui.html |

## Problemas encontrados y soluciones

### 1. URLs hardcodeadas en el frontend

Las IPs LAN en los componentes JSX fueron reemplazadas por variables de entorno `VITE_API_VENTAS` y `VITE_API_DESPACHOS` inyectadas en tiempo de build desde GitHub Actions.

### 2. Permisos IAM en AWS Academy

AWS Academy no permite crear roles IAM personalizados. Se resolvió usando el rol `LabRole` preconfigurado por la plataforma, que incluye los permisos necesarios para ECS, ECR y CloudWatch.

### 3. Health checks fallando con 404

Los target groups del ALB apuntaban a `/` por defecto. Se corrigió configurando las rutas `/api/v1/ventas` y `/api/v1/despachos` como rutas de health check en cada target group.

### 4. Servicios ECS en estado pendiente

Los servicios mostraban tareas pendientes porque no había imágenes en ECR antes de correr el pipeline por primera vez. Se resolvió ejecutando el pipeline para subir las imágenes iniciales a ECR.

### 5. Clúster ECS no visible en consola

Durante la configuración el clúster no aparecía en la lista. Se resolvió recreándolo, ya que la primera creación falló silenciosamente.

### 6. Contenedores ECS sin acceso a internet

Los contenedores en subnets privadas no podían descargar imágenes desde ECR ni enviar logs a CloudWatch. Se resolvió creando un NAT Gateway público (`innovatech-nat`) en la subnet pública `us-east-1a` y configurando las tablas de enrutamiento de las subnets privadas para apuntar el tráfico `0.0.0.0/0` hacia el NAT Gateway.

## Métricas del pipeline

Tiempos aproximados observados en GitHub Actions:

| Etapa | API Ventas | API Despachos | Frontend |
|-------|-----------|--------------|---------|
| Build | ~2 min | ~2 min | ~1 min |
| Push ECR | ~1 min | ~1 min | ~1 min |
| Deploy ECS | ~2 min | ~2 min | ~2 min |
| **Total** | **~5 min** | **~5 min** | **~4 min** |

## Estructura del repositorio

    innovatech-devops/
    ├── .github/
    │   └── workflows/
    │       ├── ci-cd-ventas.yml
    │       ├── ci-cd-despachos.yml
    │       └── ci-cd-frontend.yml
    ├── back-ventas-springboot/
    │   └── api-rest-ventas/
    │       ├── Dockerfile
    │       └── src/
    ├── back-bespachos-springboot/
    │   └── api-rest-despacho/
    │       ├── Dockerfile
    │       └── src/
    └── front-despacho/
        ├── Dockerfile
        ├── nginx.conf
        └── src/
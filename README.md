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
- **Frontend**: `http://innovatech-alb-127900156.us-east-1.elb.amazonaws.com`
- **API Ventas**: `http://innovatech-alb-127900156.us-east-1.elb.amazonaws.com:8080/api/v1/ventas`
- **API Despachos**: `http://innovatech-alb-127900156.us-east-1.elb.amazonaws.com:8081/api/v1/despachos`
- **Swagger Ventas**: `http://innovatech-alb-127900156.us-east-1.elb.amazonaws.com:8080/swagger-ui.html`
- **Swagger Despachos**: `http://innovatech-alb-127900156.us-east-1.elb.amazonaws.com:8081/swagger-ui.html`

## Problemas encontrados y soluciones

### 1. URLs hardcodeadas en el frontend
Las IPs LAN en los componentes JSX fueron reemplazadas por variables de entorno `VITE_API_VENTAS` y `VITE_API_DESPACHOS` inyectadas en tiempo de build.

### 2. Permisos IAM en AWS Academy
AWS Academy no permite crear roles IAM personalizados. Se resolvió usando el rol `LabRole` preconfigurado por la plataforma.

### 3. Health checks fallando con 404
Los target groups del ALB apuntaban a `/` por defecto. Se corrigió configurando las rutas `/api/v1/ventas` y `/api/v1/despachos` como rutas de health check.

### 4. Servicios ECS en estado pendiente
Los servicios mostraban tareas pendientes porque no había imágenes en ECR antes de correr el pipeline. Se resolvió ejecutando el pipeline por primera vez para subir las imágenes.

## Estructura del repositorio
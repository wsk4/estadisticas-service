# estadisticas-service

Microservicio de **estadísticas y KPIs** del casino (FastAPI, solo lectura).
Agrega datos de las tablas compartidas (`transacciones`, `usuarios`, `apuestas`) y
los expone para el dashboard del frontend. Comparte la base de datos PostgreSQL y el
`JWT_SECRET` con `casino-backend` — no tiene login propio, valida el JWT que emite el backend.

- **Prefijo de rutas:** `/api/estadisticas`
- **Puerto:** `8006`
- **Docs interactivos:** `/docs` (Swagger UI)

***

## Estructura del repositorio

```
estadisticas-service/
├── app/
│   ├── __init__.py
│   ├── main.py          # rutas FastAPI + health probes
│   ├── auth.py          # validación JWT
│   └── db.py            # conexión PostgreSQL
├── k8s/
│   ├── estadisticas-deployment.yaml
│   ├── estadisticas-service.yaml
│   └── estadisticas-hpa.yaml
├── tests/
├── .github/
│   └── workflows/
│       └── deploy.yml   # pipeline CI/CD
├── .dockerignore
├── .env.example         # plantilla de variables de entorno
├── .gitignore
├── Dockerfile
├── requirements.txt
└── README.md
```

***

## Variables de entorno

Copia `.env.example` a `.env` y ajusta los valores. **Nunca commitees `.env`.**

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `PORT` | Puerto del servicio | `8006` |
| `JWT_SECRET` | Debe ser idéntico al de `casino-backend` | `lab-secret-compartido-cambiar` |
| `DB_HOST` | Host de PostgreSQL | `localhost` |
| `DB_PORT` | Puerto de PostgreSQL | `5432` |
| `DB_USER` | Usuario de la BD | `casino` |
| `DB_PASSWORD` | Contraseña de la BD | `casino` |
| `DB_NAME` | Nombre de la BD | `casino_db` |
| `CORS_ORIGIN` | Orígenes CORS permitidos (coma) | `http://localhost:4200` |
| `READY_MAX_MEM_PERCENT` | Umbral de memoria para readiness probe | `90` |

***

## Cómo construir

### Local (sin Docker)

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env    # editar con valores reales
uvicorn app.main:app --reload --port 8006
```

### Con Docker

```bash
# Build
docker build -t estadisticas-service:local .

# Correr
docker run --env-file .env -p 8006:8006 estadisticas-service:local

# Verificar health probes
curl http://localhost:8006/livez
curl http://localhost:8006/readyz
```

***

## Endpoints

| Método | Ruta | Auth | Descripción |
|--------|------|------|-------------|
| GET | `/api/estadisticas/mias` | JWT | KPIs, desglose y evolución de saldo del usuario autenticado |
| GET | `/api/estadisticas/globales` | JWT | KPIs globales: GGR, top jugadores, win rate |
| GET | `/livez` | No | Liveness probe (Kubernetes) |
| GET | `/readyz` | No | Readiness probe (Kubernetes) |

***

## Health Probes

### `GET /livez` — Liveness
Verifica que el proceso FastAPI está vivo. No depende de la base de datos.
Kubernetes reinicia el pod si este endpoint falla.

```json
{ "status": "ok", "uptime_segundos": 42.3 }
```

### `GET /readyz` — Readiness
Verifica la conexión a PostgreSQL y el uso de memoria del pod.
Kubernetes saca el pod del balanceo sin reiniciarlo si este endpoint falla.

```json
// 200 OK — listo para recibir tráfico
{ "ready": true, "cpu_%": 8.2, "memoria_%": 41.5 }

// 503 Service Unavailable — fuera del balanceo
{ "ready": false, "motivo": "BD no disponible: ...", "cpu_%": 5.0, "memoria_%": 92.1 }
```

***

## Cómo desplegar

### Pipeline automático (recomendado)

```bash
# Despliegue a rama deploy (imagen taggeada con SHA del commit)
git push origin deploy

# Release con versión semántica (imagen taggeada como v1.2.3 + latest + SHA)
git tag v1.2.3
git push origin v1.2.3
```

El workflow `.github/workflows/deploy.yml` ejecuta automáticamente:
build → push a ECR → deploy en EKS.

### Manual en EKS

```bash
# 1. Configurar kubeconfig
aws eks update-kubeconfig --name <CLUSTER_NAME> --region <AWS_REGION>

# 2. Aplicar manifiestos
kubectl apply -f k8s/estadisticas-deployment.yaml
kubectl apply -f k8s/estadisticas-service.yaml
kubectl apply -f k8s/estadisticas-hpa.yaml

# 3. Verificar
kubectl get pods -l app=estadisticas-service
kubectl get hpa estadisticas-hpa
```

> El Secret `casino-secrets` es compartido entre todos los servicios. Se crea
> una sola vez en el clúster desde `apuestas-service/k8s/casino-secrets.yaml`.

***

## Pipeline CI/CD

### GitHub Secrets requeridos

| Secret | Descripción |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | Credencial temporal AWS Academy |
| `AWS_SECRET_ACCESS_KEY` | Credencial temporal AWS Academy |
| `AWS_SESSION_TOKEN` | Token de sesión temporal AWS Academy |
| `AWS_REGION` | Región AWS (ej. `us-east-1`) |
| `ECR_REPOSITORY` | `estadisticas-service` |
| `EKS_CLUSTER` | Nombre del clúster EKS |

### Tags de imagen generados

| Tag | Cuándo | Propósito |
|-----|--------|-----------|
| `v1.2.3` | Push de Git tag | Versión rastreable y reproducible |
| `latest` | Siempre | Referencia rápida al build más reciente |
| `abc1234` (SHA) | Siempre | Trazabilidad exacta del commit |

***

## Autoescalado (HPA)

El HPA escala entre **2 y 6 réplicas** según el uso de CPU, con umbral del **50%**.

```bash
# Ver estado del HPA en tiempo real
kubectl get hpa estadisticas-hpa -w

# Generar carga de prueba (requiere JWT válido en <TOKEN>)
kubectl run carga --image=busybox --rm -it --restart=Never -- \
  sh -c "while true; do \
    wget -q -O- --header='Authorization: Bearer <TOKEN>' \
    http://estadisticas-service:8006/api/estadisticas/globales; \
  done"
```

***

## Comandos útiles

```bash
# Ver pods del servicio
kubectl get pods -l app=estadisticas-service

# Ver logs en tiempo real
kubectl logs -f deployment/estadisticas-service

# Ver logs de un pod específico
kubectl logs <nombre-del-pod>

# Describir el deployment (eventos, probes, recursos)
kubectl describe deployment estadisticas-service

# Ver uso de recursos
kubectl top pods -l app=estadisticas-service

# Ejecutar shell dentro del pod
kubectl exec -it <nombre-del-pod> -- bash

# Forzar rollout (p. ej. para recargar un Secret)
kubectl rollout restart deployment/estadisticas-service

# Ver historial de rollouts
kubectl rollout history deployment/estadisticas-service

# Demostrar autorecuperación (borrar un pod y ver cómo se recrea)
kubectl delete pod <nombre-del-pod>
kubectl get pods -l app=estadisticas-service -w
```

***

## Troubleshooting

### Pod en estado `CrashLoopBackOff`
```bash
kubectl logs <nombre-del-pod> --previous
kubectl describe pod <nombre-del-pod>
```
Causas frecuentes: `JWT_SECRET` incorrecto o credenciales de BD inválidas en `casino-secrets`.

### Readiness probe fallando (`0/1 READY`)
```bash
# Verificar que la BD está accesible desde el pod
kubectl exec -it <nombre-del-pod> -- python -c \
  "import psycopg2, os; psycopg2.connect(host=os.getenv('DB_HOST')); print('OK')"
```

### Error `401 Unauthorized` en `/api/estadisticas/*`
El `JWT_SECRET` en `casino-secrets` no coincide con el de `casino-backend`.
Verificar con `kubectl get secret casino-secrets -o yaml`.

### KPIs muestran `0` o datos vacíos
Es comportamiento esperado si no hay transacciones en la BD.
Crear una apuesta desde el frontend y volver a consultar.

### HPA sin métricas (`<unknown>/50%`)
```bash
# Verificar que metrics-server está corriendo
kubectl get deployment metrics-server -n kube-system

# Verificar que el Deployment tiene resources.requests.cpu definido
kubectl describe deployment estadisticas-service | grep -A4 Requests
```

### Imagen no encontrada en ECR
```bash
# Las credenciales Academy expiran cada ~4h
aws sts get-caller-identity

# Re-autenticar en ECR
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.$AWS_REGION.amazonaws.com
```

***

## Convención de commits

```
feat:  nueva funcionalidad
fix:   corrección de bug
chore: mantenimiento (dependencias, configs)
ci:    cambios en pipeline o workflows
docs:  cambios en documentación
test:  agregar o corregir tests
```

**Ejemplos:**
```
feat: implementar health probes /livez y /readyz con psutil
ci: agregar workflow deploy.yml con push a ECR y EKS
chore: agregar psutil a requirements.txt
docs: documentar pipeline CI/CD en README
fix: corregir umbral de memoria en readiness probe
```
# Deal Engine — Reto Técnico DevOps/SRE

Diseño de arquitectura de despliegue y operación de una aplicación web sobre Kubernetes en AWS, orientada a manejar información sensible de tickets de viaje para agencias y aerolíneas.

---

## Índice

1. [Supuestos](#supuestos)
2. [Arquitectura](#arquitectura)
3. [Diseño de Kubernetes](#diseño-de-kubernetes)
4. [CI/CD](#cicd)
5. [Red y Seguridad](#red-y-seguridad)
6. [Observabilidad](#observabilidad)

---

## Supuestos

Antes de cualquier decisión de diseño, se definieron los siguientes supuestos explícitos:

| Supuesto | Decisión |
|---|---|
| Proveedor cloud | AWS |
| Kubernetes | Amazon EKS (Control Plane gestionado por AWS) |
| Tipo de nodos | EC2 gestionados por Karpenter |
| Base de datos | Amazon RDS PostgreSQL Multi-AZ |
| Registry de imágenes | Amazon ECR |
| Gestión de secretos | AWS Secrets Manager |
| Certificados TLS | AWS Certificate Manager (ACM) |
| CI/CD | GitHub Actions + AWS CodePipeline |
| Tamaño del equipo | 3-5 desarrolladores |
| Tráfico base | ~500 req/s con picos estacionales del doble (~1000 req/s) |

**¿Por qué AWS y no GCP?**
El equipo tiene experiencia previa con AWS. Usar el mismo proveedor reduce la curva de aprendizaje operativa y permite integración nativa entre EKS, ECR, IAM y Secrets Manager sin configuración adicional.

**¿Por qué EKS y no Fargate?**
Fargate es serverless y elimina la gestión de nodos, pero a la escala de 500-1000 req/s, EC2 con Karpenter ofrece más control sobre el tipo de instancia, mejor relación costo/rendimiento y aprovisionamiento de nodos en segundos ante picos estacionales.

---

## Arquitectura

### Diagrama de alto nivel

<img width="818" height="108" alt="image" src="https://github.com/user-attachments/assets/00498bb9-810b-4e74-9e27-a5bb0fbee761" />

### Flujo de tráfico

```
Usuario (HTTPS)
    ↓
Internet Gateway  →  AWS WAF  →  ALB (TLS termination con ACM)
    ↓ HTTP :80
Ingress Controller (enruta por path/host)
    ├── app.dealengine.com /      →  Frontend pods (2 réplicas)
    └── api.dealengine.com /api   →  Backend pods  (3 réplicas)
                                          ↓ SQL
                                  RDS PostgreSQL Primary (Multi-AZ)
                                          ↓ replicación
                                  RDS Read Replica
```

### Componentes principales

**Internet Gateway** — único punto de entrada a la VPC desde internet. Solo existe en la subred pública, lo que garantiza que el backend y la base de datos no sean accesibles directamente desde el exterior.

**AWS WAF** — filtra tráfico malicioso antes de que llegue a la aplicación. Protege contra ataques comunes (SQL injection, XSS) críticos dado que se manejan datos sensibles de pasajeros.

**ALB (Application Load Balancer)** — termina TLS usando certificados de AWS Certificate Manager. A partir de aquí el tráfico interno viaja por HTTP, simplificando la configuración dentro del clúster.

**Ingress Controller** — enruta el tráfico interno hacia el Service correcto según el host y el path. Centraliza el punto de entrada al clúster evitando exponer múltiples LoadBalancers.

**HPA (Horizontal Pod Autoscaler)** — escala los pods automáticamente entre 3 y 10 réplicas según CPU y memoria. Maneja los picos estacionales del doble de tráfico sin intervención manual.

**Karpenter** — aprovisiona y termina nodos EC2 automáticamente cuando HPA necesita más pods de los que caben en los nodos actuales. Más rápido y eficiente que el Cluster Autoscaler tradicional.

**RDS PostgreSQL Multi-AZ** — base de datos principal con réplica en otra zona de disponibilidad. Si la instancia primaria falla, el failover es automático en menos de 60 segundos.

---

## Diseño de Kubernetes

### Proveedor y tipo de clúster

**Amazon EKS** — Kubernetes gestionado por AWS. AWS administra el Control Plane (API Server, etcd, Scheduler), el equipo solo gestiona los Worker Nodes y los recursos del clúster.

### Estructura de namespaces

```
cluster EKS
├── production      → aplicación en vivo (frontend, backend)
├── staging         → pruebas previas a producción
├── monitoring      → Prometheus, Grafana, Alertmanager, Fluent Bit
└── ingress-nginx   → Ingress Controller
```

**¿Por qué separar en namespaces?**
El aislamiento por namespace garantiza que un pod en `staging` no pueda comunicarse con pods en `production` a menos que se permita explícitamente. También permite aplicar límites de recursos y permisos RBAC por ambiente, evitando que staging consuma recursos de producción.

### Pseudo-manifiestos

Los manifiestos se encuentran en la carpeta `/k8s-manifests`:

| Archivo | Descripción |
|---|---|
| `01-deployment.yaml` | Deployment del backend con 3 réplicas, probes y variables de entorno |
| `02-service.yaml` | Service ClusterIP para backend y frontend |
| `03-ingress.yaml` | Ingress con TLS y enrutamiento por host |
| `04-configmap.yaml` | Configuración no sensible (URLs, ambiente, DB host) |
| `05-secret.yaml` | Credenciales sincronizadas desde AWS Secrets Manager |
| `06-hpa.yaml` | HPA que escala pods de 3 a 10 según CPU y memoria |

### Estrategia de despliegue

Se usa una combinación de dos estrategias según el tipo de cambio:

**Rolling Update — para deploys del día a día**
K8s reemplaza los pods de uno en uno garantizando zero downtime. Siempre hay al menos 3 pods disponibles durante el deploy (`maxUnavailable: 0`).

**Canary — para cambios críticos**
Cuando se toca lógica sensible (módulo de pagos, datos de pasajeros, cambios en la API), se envía un 5% del tráfico a la nueva versión. Si las métricas son estables después de 15 minutos, se aumenta gradualmente al 100%.

**¿Por qué no Blue/Green como estrategia principal?**
Blue/Green requiere el doble de recursos de forma permanente. El costo operativo no se justifica para deploys rutinarios cuando Rolling Update ya garantiza zero downtime. Blue/Green se reservaría para migraciones de base de datos o cambios de infraestructura mayores.

**Rollback**
```bash
# Ver historial de versiones
kubectl rollout history deployment/backend -n production

# Rollback a la versión anterior
kubectl rollout undo deployment/backend -n production

# Rollback a una versión específica
kubectl rollout undo deployment/backend -n production --to-revision=2
```

---

## CI/CD

### Herramienta

**GitHub Actions** para integración continua (build, pruebas, escaneo de seguridad, push a ECR).
**AWS CodePipeline** para orquestar el despliegue a los ambientes de AWS.

### Pipeline

El archivo se encuentra en `.github/workflows/pipeline.yml`.

```
push a GitHub
      ↓
JOB 1: Checkout y validación
  - Valida manifiestos YAML de K8s con kubeval
  - Lint del Dockerfile con hadolint
      ↓
JOB 2: Build y pruebas unitarias
  - npm install + npm test
  - docker build
      ↓
JOB 3: Escaneo de seguridad
  - Trivy: busca vulnerabilidades CRITICAL/HIGH en la imagen
  - Gitleaks: detecta secretos hardcodeados en el código
      ↓
JOB 4: Publicación a ECR
  - docker push con tag del commit SHA (trazabilidad)
  - docker push con tag latest
      ↓
JOB 5: Deploy a Staging        (rama: staging)
  - kubectl set image + rollout status
      ↓
JOB 6: Gate de aprobación manual
  - Reviewer aprueba en GitHub antes de continuar
      ↓
JOB 7: Deploy a Producción     (rama: main)
  - kubectl set image + rollout status
  - Rollback automático si falla (kubectl rollout undo)
```

### Decisiones clave

**Tag con `github.sha`** — cada imagen tiene el hash único del commit como tag. Esto garantiza trazabilidad completa: si hay un bug en producción, se sabe exactamente qué commit lo causó.

**Trivy + Gitleaks** — dos herramientas con propósitos distintos. Trivy escanea la imagen buscando vulnerabilidades en dependencias. Gitleaks escanea el código buscando credenciales que alguien pudo haber commiteado accidentalmente.

**Gate manual** — ningún deploy llega a producción sin aprobación explícita de un reviewer. Se implementa con el campo `environment: production` de GitHub Actions, que pausa el pipeline y notifica al equipo.

**Rollback automático** — si el deploy a producción falla, `kubectl rollout undo` se ejecuta automáticamente minimizando el tiempo de downtime.

---

## Red y Seguridad

### Diseño de VPC

```
VPC — 10.0.0.0/16
├── Subred pública  10.0.1.0/24   → Internet Gateway, WAF, ALB
├── Subred privada  10.0.2.0/24   → Clúster EKS, Worker Nodes
└── Subred privada  10.0.3.0/24   → RDS PostgreSQL
```

La base de datos y los pods nunca tienen acceso directo desde internet. Todo el tráfico externo entra por la subred pública y es filtrado antes de llegar al clúster.

### Security Groups

| Recurso | Permite entrada | Permite salida |
|---|---|---|
| ALB | 443 desde internet | 80 hacia Ingress |
| Worker Nodes EKS | 80 desde ALB | 5432 hacia RDS |
| RDS PostgreSQL | 5432 desde Worker Nodes | ninguna |

### Gestión de secretos

Los Secrets nativos de Kubernetes almacenan valores en base64, lo cual **no es encriptación**. Para datos sensibles de pasajeros se usa **AWS Secrets Manager** como fuente de verdad:

```
AWS Secrets Manager (encriptado con KMS)
        ↓
Secrets Store CSI Driver
        ↓
Secret de K8s (sincronizado automáticamente)
        ↓
Pod (consume como variable de entorno)
```

Ventajas sobre Secrets nativos: encriptación real con KMS, rotación automática de credenciales, auditoría de accesos en CloudTrail.

### Acceso administrativo al clúster

El acceso al clúster EKS se gestiona mediante **IAM + RBAC**:
- Los desarrolladores tienen acceso de solo lectura a `production`
- Solo el pipeline CI/CD tiene permisos para hacer deploy a `production`
- Acceso administrativo completo restringido a un rol IAM específico con MFA obligatorio

---

## Observabilidad

### Logs

**Fluent Bit** recolecta logs de todos los pods y los envía a **Amazon CloudWatch**. Centralizar los logs permite buscar errores en todos los servicios desde un solo lugar.

### Métricas

**Prometheus** recolecta métricas de pods y nodos cada 15 segundos. **Grafana** las visualiza en dashboards. Las métricas más importantes para este sistema:

- Latencia de requests (p50, p95, p99)
- Tasa de errores (5xx)
- CPU y memoria por pod
- Pods en estado Pending (indica que Karpenter necesita aprovisionar nodos)

### Health checks

Todos los pods tienen dos probes configurados:

- **Readiness probe** — K8s no envía tráfico al pod hasta que responda `/ready`. Evita enviar requests a pods que todavía están iniciando.
- **Liveness probe** — K8s reinicia el pod si `/health` deja de responder. Recuperación automática ante estados corruptos.

### SLOs mínimos

| Métrica | Objetivo |
|---|---|
| Disponibilidad | 99.9% uptime mensual (máx. ~43 min de downtime/mes) |
| Latencia p99 | < 500ms para el 99% de las requests |
| Tasa de errores | < 0.1% de requests con error 5xx |

**Alertmanager** notifica al equipo vía Slack cuando algún SLO está en riesgo antes de que se rompa.

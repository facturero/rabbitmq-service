# rabbitmq-basic

Proyecto de **infraestructura básica** (no es un microservicio) para desplegar y versionar **RabbitMQ 3.13 (con plugin management)** en tu cluster **k3s**, usando GitOps simple: cualquier cambio en `k8s/` que se suba a `main` se aplica automáticamente vía el runner self-hosted de GitHub Actions.

## ¿Qué hace?

- Despliega **RabbitMQ 3.13-management** (`rabbitmq-basic`) con almacenamiento persistente (`PersistentVolumeClaim`, storage class `local-path` de k3s).
- Lo expone vía `NodePort` (AMQP + Management UI) para conectarte desde fuera del cluster.

## Puertos expuestos

| Servicio | Puerto interno | NodePort | Acceso |
|---|---|---|---|
| RabbitMQ (AMQP) | 5672 | **30672** | `amqp://<ip-servidor>:30672` |
| RabbitMQ (Management UI) | 15672 | **31672** | `http://<ip-servidor>:31672` |

## Configuración inicial (una sola vez)

1. Crea el repo en GitHub y conéctalo:

   ```bash
   cd C:\Users\sansh\cmr-proyect\backend\rabbitmq-basic
   git init
   git branch -M main
   git remote add origin <URL_DE_TU_REPO>
   ```

2. Agrega estos **Secrets** en GitHub (Settings → Secrets and variables → Actions):

   | Secret | Ejemplo |
   |---|---|
   | `RABBITMQ_USER` | `admin` |
   | `RABBITMQ_PASSWORD` | contraseña admin |

3. Confirma que tu runner self-hosted tiene `kubectl` configurado apuntando a tu cluster k3s.

4. Sube el proyecto:

   ```bash
   git add .
   git commit -m "infra: rabbitmq-basic"
   git push -u origin main
   ```

## Flujo día a día

```bash
git add k8s/
git commit -m "infra: ajuste de recursos rabbitmq-basic"
git push
```

El runner aplica los cambios automáticamente. También puedes disparar el deploy manualmente desde la pestaña **Actions** de GitHub (`workflow_dispatch`).

## Verificar estado manualmente

```bash
kubectl get pods -l app=rabbitmq-basic
kubectl logs deployment/rabbitmq-basic
```

## Notas

- **Recursos:** límites de memoria/CPU ajustados pensando en un servidor con ~7GB de RAM (ver `resources` en `k8s/rabbitmq-basic-deployment.yaml`).
- **Namespace:** se despliega en `default`.

# rabbitmq-basic

Proyecto de **infraestructura básica** (no es un microservicio) para desplegar y versionar **RabbitMQ 3.13 (con plugin management)** en tu cluster **k3s**, usando GitOps simple: cualquier cambio en `k8s/` que se suba a `master` se aplica automáticamente vía el runner self-hosted de GitHub Actions.

## ¿Qué hace?

- Despliega **RabbitMQ 3.13-management** (`rabbitmq-basic`) con almacenamiento persistente (`PersistentVolumeClaim`, storage class `local-path` de k3s).
- Lo expone vía `NodePort` (AMQP + Management UI) para conectarte desde fuera del cluster.

## Puertos expuestos

| Servicio | Puerto interno | NodePort | Acceso |
|---|---|---|---|
| RabbitMQ (AMQP) | 5672 | **30672** | `amqp://<ip-servidor>:30672` |
| RabbitMQ (Management UI) | 15672 | **31672** | `http://<ip-servidor>:31672` |

## Configuración inicial (una sola vez)

1. Agrega estos **Secrets** en GitHub (Settings → Secrets and variables → Actions) del repo `rabbitmq-basic`:

   | Secret | Ejemplo |
   |---|---|
   | `RABBITMQ_USER` | `admin` |
   | `RABBITMQ_PASSWORD` | contraseña admin |

2. Confirma que tu runner self-hosted tiene `kubectl` configurado apuntando a tu cluster k3s.

3. Sube el proyecto:

   ```bash
   git add .
   git commit -m "infra: rabbitmq-basic"
   git push -u origin master
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

## Si el auto-despliegue no se dispara

- Confirma en GitHub → pestaña **Actions** si aparece el workflow "Deploy rabbitmq-basic to k3s" corriendo o con error.
- Confirma que tu runner self-hosted está **online** (Settings → Actions → Runners, debe verse verde/"Idle").
- Confirma que el push fue a la rama `master` y que tocó archivos dentro de `k8s/`.
- Si el runner está online pero el job falla en el paso de `kubectl`, revisa que el usuario que corre el runner tenga acceso al `kubeconfig` de k3s.

## Notas

- **Recursos:** límites de memoria/CPU ajustados pensando en un servidor con ~7GB de RAM (ver `resources` en `k8s/rabbitmq-basic-deployment.yaml`).
- **Namespace:** se despliega en `default`.

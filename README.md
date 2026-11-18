# rabbitmq-basic

Proyecto de **infraestructura básica** (no es un microservicio) para desplegar y versionar **RabbitMQ 3.13 (con plugin management)** en tu cluster **k3s**, usando GitOps simple: cualquier cambio en `k8s/` que se suba a `master` se aplica automáticamente vía el runner self-hosted de GitHub Actions.

## ¿Qué hace?

- Despliega **RabbitMQ 3.13-management** (`rabbitmq-basic`) con almacenamiento persistente (`PersistentVolumeClaim`, storage class `local-path` de k3s).
- Lo expone vía `NodePort` (AMQP + Management UI) para conectarte desde fuera del cluster.

## Puertos expuestos

El Service interno se llama **`rabbitmq`** (ClusterIP) y es por donde entran los demás
servicios: `amqp://rabbitmq:5672`. La Management UI no se expone fuera del clúster; se mira
con un port-forward:

```bash
kubectl port-forward svc/rabbitmq 15672:15672   # http://localhost:15672  (guest/guest)
```

Las credenciales por defecto (`guest`/`guest`) viven en el ConfigMap `rabbitmq-config`
(`k8s/rabbitmq-config.yaml`), que además trae `loopback_users = none` para que `guest` pueda
conectar desde otros pods. Los env `RABBITMQ_DEFAULT_USER`/`RABBITMQ_DEFAULT_PASS` **no** los
lee la imagen 3.13 (solo `RABBITMQ_DEFAULT_ALLOC_ARGS`): no sirven para esto.

## Configuración inicial

1. Confirma que tu runner self-hosted tiene `kubectl` configurado apuntando a tu cluster k3s.

2. Sube el proyecto:

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

- **Recursos:** solo `requests` (reserva) de memoria/CPU, pensando en un servidor con ~7GB de RAM (ver `resources` en `k8s/rabbitmq-basic-deployment.yaml`). Sin `limits` a propósito: un tope de memoria baja el `vm_memory_high_watermark` de RabbitMQ por debajo de su uso normal y bloquea a los publicadores.
- **`runAsUser: 999`:** el contenedor corre directamente como el usuario `rabbitmq` para esquivar `gosu`. Con el kernel `7.0.0-43` del nodo (reinicio del 2026-11-18) `gosu` gira al 100% de CPU y nunca ejecuta el entrypoint, dejando RabbitMQ en `0/1`. El entrypoint oficial, al ver que no es root, arranca el broker sin `gosu`.
- **livenessProbe:** `rabbitmq-diagnostics -q ping` reinicia el contenedor si se queda trabado. Sin ella, el pod del 2026-11-18 estuvo `0/1` durante horas sin que kubelet lo reiniciara.
- **Namespace:** se despliega en `default`.

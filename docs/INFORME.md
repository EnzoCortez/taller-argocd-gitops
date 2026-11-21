# Taller: Despliegue de una aplicación con Argo CD y Kubernetes (GitOps)

**Repositorio (fuente de verdad):** https://github.com/EnzoCortez/taller-argocd-gitops
**Ruta de los manifiestos:** `k8s/`

```
k8s/
├── deployment.yml   # Deployment: 2 réplicas, labels/selectors, requests/limits, puerto 80
├── service.yml      # Service NodePort 30080 -> targetPort 80
└── hpa.yml          # HPA: min 2, max 5, CPU 50 %
argocd/
└── application.yml  # Application de Argo CD con Automated Sync
```

> `argocd/application.yml` está **fuera** de `k8s/` a propósito: Argo CD sincroniza solo
> la carpeta `k8s/`, y la Application se crea una sola vez a mano.

---

## 1. Objetivo

Desplegar una aplicación web en Kubernetes con Argo CD, usando un repositorio Git como
fuente de verdad. Cualquier cambio en `k8s/` se aplica solo en el clúster (Automated Sync).

## 2. Descripción de los manifiestos

### 2.1 Deployment (`k8s/deployment.yml`)

| Requisito | Implementación |
|---|---|
| 2 réplicas iniciales | `spec.replicas: 2` |
| Labels y selectors | `selector.matchLabels.app: web-app` = `template.metadata.labels.app: web-app` |
| Requests / limits | requests `cpu: 200m`, `memory: 64Mi` · limits `cpu: 500m`, `memory: 128Mi` |
| Puerto expuesto | `containerPort: 80` |

Imagen: `registry.k8s.io/hpa-example`, la imagen oficial de Kubernetes para demostrar el
HPA. Es un Apache + PHP que hace un cálculo intensivo de CPU en cada petición, así el
escalado se ve fácilmente. Además lleva `readinessProbe` y `livenessProbe` HTTP.

### 2.2 Service (`k8s/service.yml`)

- `type: NodePort`
- `port: 80` (dentro del clúster) → `targetPort: 80` (contenedor) → `nodePort: 30080` (externo)
- `selector: app: web-app` → apunta a los Pods del Deployment

### 2.3 HPA (`k8s/hpa.yml`)

- `scaleTargetRef` → `Deployment/web-app`
- `minReplicas: 2`, `maxReplicas: 5`
- `averageUtilization: 50` de CPU (porcentaje sobre el `request` de 200m, es decir ~100m por Pod)

### 2.4 Application de Argo CD (`argocd/application.yml`)

- `repoURL`: este repositorio · `targetRevision: main` · `path: k8s`
- `syncPolicy.automated` con `prune: true` y `selfHeal: true` (Automated Sync)
- `CreateNamespace=true` → crea el namespace `taller-argocd`
- `ignoreDifferences` en `/spec/replicas` del Deployment: el HPA cambia el número de
  réplicas y, sin esto, Argo CD (con selfHeal) lo devolvería a 2 en cada sincronización.

---

## 3. Procedimiento de ejecución

Puede usarse **Minikube**, **kind**, **k3d/k3s** o **Docker Desktop**. Los ejemplos usan Minikube.

### Paso 0 — Clúster y Metrics Server

```bash
minikube start --cpus=4 --memory=4096
```

```bash
minikube addons enable metrics-server
```

En otros clústeres:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

(En kind/Docker Desktop puede hacer falta añadir `--kubelet-insecure-tls` a los args del metrics-server.)

```bash
kubectl top nodes
```

📸 **Captura 1:** `kubectl get nodes` y `kubectl top nodes` funcionando (Metrics Server operativo).

### Paso 1 — Instalar Argo CD

```bash
kubectl create namespace argocd
```

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

```bash
kubectl get pods -n argocd -w
```

Contraseña inicial del usuario `admin`:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Acceso a la interfaz web (dejar corriendo en otra terminal):

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Abrir https://localhost:8080 e iniciar sesión con `admin` / contraseña anterior.

📸 **Captura 2:** Pods de Argo CD en `Running` (`kubectl get pods -n argocd`).
📸 **Captura 3:** Pantalla de inicio de Argo CD tras el login.

### Paso 2 — Crear la Application en Argo CD

**Opción A (declarativa, recomendada):**

```bash
kubectl apply -f argocd/application.yml
```

**Opción B (desde la interfaz web):** *+ NEW APP* y rellenar:

| Campo | Valor |
|---|---|
| Application Name | `web-app` |
| Project | `default` |
| Sync Policy | **Automatic** ✔ Prune Resources ✔ Self Heal |
| Sync Options | ✔ Auto-Create Namespace |
| Repository URL | `https://github.com/EnzoCortez/taller-argocd-gitops.git` |
| Revision | `main` |
| Path | `k8s` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `taller-argocd` |

📸 **Captura 4:** Formulario de creación (o el `kubectl apply` de la Application).
📸 **Captura 5:** Application `web-app` en estado **Synced** y **Healthy**, con el árbol de
recursos (Deployment → ReplicaSet → Pods, Service, HPA).
📸 **Captura 6:** Detalle *App Details* mostrando `SYNC POLICY: Automated` y el path `k8s`.

### Paso 3 — Verificar Deployment y Service

```bash
kubectl get all -n taller-argocd
```

```bash
kubectl get pods -n taller-argocd --show-labels
```

```bash
kubectl get endpoints web-app-service -n taller-argocd
```

Acceso a la aplicación:

```bash
minikube service web-app-service -n taller-argocd --url
```

```bash
curl http://$(minikube ip):30080
```

La respuesta esperada es `OK!`.

📸 **Captura 7:** `kubectl get all -n taller-argocd` con 2 Pods `Running` y el Service `NodePort 80:30080`.
📸 **Captura 8:** La aplicación respondiendo (`curl` o navegador → `OK!`).

### Paso 4 — Verificar el HPA (escalado hacia arriba y hacia abajo)

```bash
kubectl get hpa -n taller-argocd
```

Al principio debe verse algo como `cpu: 1%/50%   MINPODS 2   MAXPODS 5   REPLICAS 2`.

Generar carga (en una terminal aparte):

```bash
kubectl run load-generator -n taller-argocd --rm -it --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://web-app-service; done"
```

Observar el escalado:

```bash
kubectl get hpa web-app-hpa -n taller-argocd --watch
```

```bash
kubectl get pods -n taller-argocd -w
```

El % de CPU sube por encima del 50 % y las réplicas aumentan hasta un máximo de 5.
Al detener la carga (Ctrl+C), tras unos **5 minutos** (ventana de estabilización) vuelven a 2.

```bash
kubectl describe hpa web-app-hpa -n taller-argocd
```

📸 **Captura 9:** HPA en reposo (2 réplicas, CPU baja).
📸 **Captura 10:** HPA bajo carga (CPU > 50 %, réplicas 3–5) y Pods nuevos creándose.
📸 **Captura 11:** `kubectl describe hpa` con los eventos `SuccessfulRescale` (subida y bajada).
📸 **Captura 12:** Vista de Argo CD con más Pods durante la carga (la app sigue **Synced** gracias a `ignoreDifferences`).

### Paso 5 — Demostrar GitOps (Automated Sync)

1. Editar un manifiesto en el repositorio, por ejemplo agregar una label en `k8s/deployment.yml`
   (`version: "v2"` en `metadata.labels`) o cambiar `maxReplicas: 5` → `6` en `k8s/hpa.yml`.
2. Hacer commit y push a `main`:

```bash
git add k8s/ && git commit -m "Cambio para probar Automated Sync" && git push origin main
```

3. Argo CD revisa el repositorio cada ~3 minutos (o pulsar **Refresh**). La aplicación pasa
   brevemente a *OutOfSync* y se sincroniza sola.

```bash
kubectl get hpa web-app-hpa -n taller-argocd
```

4. **Self Heal:** borrar un recurso a mano y ver cómo Argo CD lo recrea:

```bash
kubectl delete svc web-app-service -n taller-argocd
```

```bash
kubectl get svc -n taller-argocd
```

📸 **Captura 13:** Commit en GitHub con el cambio.
📸 **Captura 14:** Argo CD mostrando la nueva revisión sincronizada automáticamente (historial *History and Rollback*).
📸 **Captura 15:** El cambio reflejado en el clúster (y el Service recreado por Self Heal).

---

## 4. Resultados

| Requisito | Evidencia |
|---|---|
| Deployment con 2 réplicas, labels, recursos y puerto | Capturas 5, 7 |
| Service NodePort accesible | Capturas 7, 8 |
| HPA 2–5 réplicas, CPU 50 %, escala arriba/abajo | Capturas 9, 10, 11 |
| Metrics Server en el clúster | Captura 1 |
| Application de Argo CD con path `k8s` y Automated Sync | Capturas 4, 5, 6 |
| Sincronización automática tras cambios en Git | Capturas 13, 14, 15 |

## 5. Conclusiones

- Con GitOps, el repositorio Git es la única fuente de verdad: el estado del clúster se
  declara en YAML y Argo CD se encarga de reconciliarlo, sin `kubectl apply` manuales.
- `selfHeal` corrige desviaciones hechas a mano y `prune` elimina recursos que se quitan del repo.
- El HPA necesita **Metrics Server** y **requests de CPU** definidos, porque el porcentaje de
  utilización se calcula sobre el `request`.
- Cuando un HPA gestiona las réplicas, hay que excluir `spec.replicas` de la comparación de
  Argo CD (`ignoreDifferences`) para evitar que ambos controladores compitan.

## 6. Limpieza

```bash
kubectl delete -f argocd/application.yml
```

```bash
minikube delete
```

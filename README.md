# Taller GitOps: Argo CD + Kubernetes

Despliegue automatizado de una aplicación web en Kubernetes con **Argo CD**, usando este
repositorio como fuente de verdad. Incluye Deployment, Service (NodePort) y Horizontal Pod
Autoscaler, sincronizados automáticamente (Automated Sync).

## Estructura

```
k8s/                    # ← ruta que sincroniza Argo CD
├── deployment.yml      # 2 réplicas, labels/selectors, requests/limits, puerto 80
├── service.yml         # NodePort 30080 → targetPort 80
└── hpa.yml             # min 2, max 5 réplicas, CPU 50 %
argocd/
└── application.yml     # Application de Argo CD (path k8s, Automated Sync)
docs/
├── INFORME.md          # Informe del taller con el procedimiento y las capturas
└── capturas/           # Capturas de pantalla de la ejecución
```

## Requisitos

- Clúster Kubernetes (Minikube, kind, k3d o Docker Desktop)
- **Metrics Server** instalado (necesario para el HPA)
- **Argo CD** instalado en el namespace `argocd`

## Uso rápido

```bash
kubectl apply -f argocd/application.yml
```

Argo CD creará el namespace `taller-argocd` y desplegará los recursos de `k8s/`.
Cada `git push` a `main` que modifique `k8s/` se aplica automáticamente en el clúster.

```bash
kubectl get all,hpa -n taller-argocd
```

La aplicación queda accesible en `http://<IP-del-nodo>:30080` (responde `OK!`).

El procedimiento completo y las evidencias están en [docs/INFORME.md](docs/INFORME.md).

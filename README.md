# Despliegue continuo de una aplicación web con Argo CD y Kubernetes (GitOps)

**Universidad de las Américas — UDLA**
**Asignatura:** Procesos de Software — ISWZ3205
**Actividad:** Automatización de la configuración continua 10 % P2
**Modalidad:** Individual
**Autor:** Justin Gomezcoello

---

## 1. Descripción

Este repositorio implementa un flujo **GitOps** completo: el repositorio Git es la
única fuente de verdad y **Argo CD** se encarga de sincronizar el estado declarado
con un clúster de Kubernetes. Cualquier cambio en los manifiestos de la carpeta
`k8s/` es detectado por Argo CD y aplicado automáticamente al clúster, sin ejecutar
`kubectl apply` de forma manual.

La aplicación desplegada es el **frontend de la tienda streetwear**
(`e-commerce`), un proyecto web desarrollado previamente en React 18 (Create React
App), servido por **nginx**. Es una aplicación 100 % estática, sin estado ni base de
datos, por lo que puede ejecutarse con varias réplicas idénticas detrás de un
Service de Kubernetes.

## 2. Objetivos

- Desplegar una aplicación web en Kubernetes mediante un Deployment con 2 réplicas.
- Exponer la aplicación con un Service de tipo NodePort.
- Configurar un Horizontal Pod Autoscaler entre 2 y 5 réplicas con objetivo de CPU del 50 %.
- Instalar Argo CD y registrar la aplicación apuntando a este repositorio.
- Habilitar la sincronización automática (Automated Sync) y demostrar que los cambios
  en Git se reflejan solos en el clúster.

## 3. Arquitectura

```
 Desarrollador
      │  git push
      ▼
 ┌──────────────────────────────┐
 │ GitHub (fuente de verdad)    │
 │  k8s/deployment.yml          │
 │  k8s/service.yml             │
 │  k8s/hpa.yml                 │
 └──────────────┬───────────────┘
                │  sondeo cada 3 min / webhook
                ▼
        ┌───────────────┐
        │   Argo CD     │  OutOfSync → Synced
        └───────┬───────┘
                │  aplica el estado declarado
                ▼
 ┌────────────────────────────────────────────┐
 │ Clúster Kubernetes (Minikube)              │
 │ namespace: streetwear-lab                  │
 │                                            │
 │  Service NodePort 30080 ──► Deployment     │
 │                              │             │
 │                              ├─ Pod 1      │
 │                              ├─ Pod 2      │
 │                              └─ … (hasta 5)│
 │                                            │
 │  HPA: 2 → 5 réplicas, CPU 50 %             │
 │  Metrics Server: provee las métricas       │
 └────────────────────────────────────────────┘
```

## 4. Estructura del repositorio

```
udla-argocd-gitops/
├── app/                    # Aplicación web (frontend React de e-commerce)
│   ├── Dockerfile          # Build multi-stage: node:18-alpine → nginx:alpine
│   ├── nginx.conf          # Servidor estático con fallback de SPA
│   ├── public/
│   └── src/
├── k8s/                    # Manifiestos gestionados por Argo CD
│   ├── deployment.yml      # 2 réplicas, labels, requests/limits, puerto 80
│   ├── service.yml         # NodePort 80 → targetPort 80
│   └── hpa.yml             # min 2, max 5, CPU 50 %
├── argocd/
│   └── application.yml     # Application de Argo CD (auto-sync)
├── evidencias/             # Capturas numeradas del proceso
└── README.md
```

### Recursos declarados

| Recurso | Configuración |
|---|---|
| Deployment | 2 réplicas, imagen `udla-web:1.0`, puerto 80 |
| requests | CPU `100m`, memoria `128Mi` |
| limits | CPU `500m`, memoria `320Mi` |
| Service | `NodePort`, puerto 80 → targetPort 80, nodePort 30080 |
| HPA | mínimo 2, máximo 5, objetivo de CPU 50 % |

## 5. Requisitos

| Herramienta | Versión usada |
|---|---|
| Docker Desktop | 29.6.2 |
| Minikube | v1.39.0 |
| kubectl | v1.36.1 |
| Kubernetes | v1.37.0 (runtime containerd) |
| Argo CD | manifiestos `stable` |

Se recomienda asignar al menos **4 CPU y 6 GB de RAM** a Minikube, porque Argo CD
agrega varios Pods al clúster.

## 6. Construcción de la imagen

La imagen se construye en el equipo y se carga en el runtime del clúster local
(containerd), sin necesidad de un registro externo:

```bash
docker build -t udla-web:1.0 -f app/Dockerfile app
minikube image load udla-web:1.0
minikube image ls | findstr udla-web
```

El Deployment usa `imagePullPolicy: IfNotPresent` para tomar esa imagen local.

## 7. Instalación de Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd get pods -w
```

Obtener la contraseña inicial del usuario `admin`:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Abrir la interfaz:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
# https://localhost:8080  →  usuario: admin
```

## 8. Registro de la aplicación en Argo CD

```bash
kubectl apply -f argocd/application.yml
kubectl -n argocd get applications
```

La `Application` declara:

- `repoURL`: este repositorio público.
- `path: k8s` → ruta donde están los manifiestos.
- `targetRevision: main`.
- `syncPolicy.automated` con `prune` y `selfHeal` → sincronización automática y
  corrección de desviaciones manuales.

### Nota: Argo CD y el HPA sobre `spec.replicas`

El HPA modifica `spec.replicas` del Deployment en tiempo de ejecución. Si Argo CD
considerara ese campo, revertiría cada escalado del HPA y se produciría un bucle de
reconciliación (Argo pone 2, el HPA pone 5, Argo vuelve a 2…). Por eso la
`Application` excluye ese campo:

```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
      - /spec/replicas
```

Con esta exclusión, Argo CD administra la plantilla del Deployment (imagen, recursos,
sondas, etiquetas) y el HPA administra el número de réplicas. Es la práctica
recomendada cuando ambos controladores conviven.

## 9. Verificación en el clúster

```bash
kubectl get applications -n argocd
kubectl get deployments,replicasets,pods -n streetwear-lab
kubectl get svc -n streetwear-lab
kubectl get hpa -n streetwear-lab
kubectl top pods -n streetwear-lab
minikube service streetwear-web -n streetwear-lab
```

## 10. Demostración del flujo GitOps

1. Editar un campo de la plantilla en `k8s/deployment.yml`, por ejemplo
   `limits.memory: 256Mi → 320Mi`.
2. Hacer `git commit` y `git push` a `main`.
3. En Argo CD, la aplicación pasa a `OutOfSync` y ejecuta la sincronización
   automática: crea un nuevo ReplicaSet y reemplaza los Pods.
4. `kubectl get pods -n streetwear-lab` muestra los Pods nuevos sin haber usado
   `kubectl apply`.

También se puede verificar la **autorreparación** (`selfHeal`): si se elimina un
objeto del clúster con `kubectl delete` (por ejemplo el Service), Argo CD lo vuelve a
crear para restablecer el estado declarado en Git.

## 11. Evidencias

| # | Archivo | Qué demuestra |
|---|---|---|
| 01 | `01-argocd-instalado.png` | Argo CD instalado y aplicación registrada (`Synced`) |
| 02 | `02-argocd-ui-sincronizado.jpeg` | Interfaz de Argo CD: `Healthy`, `Synced`, auto-sync activo |
| 03 | `03-deployment-replicaset-pods.png` | Deployment 2/2, ReplicaSet y Pods |
| 04 | `04-service-nodeport.png` | Service NodePort 30080, selector y endpoints |
| 05 | `05-aplicacion-en-navegador.jpeg` | Aplicación web accesible desde el navegador |
| 06 | `06-metricas-pods.png` | Métricas de CPU y memoria (`kubectl top`) |
| 07 | `07-hpa-configurado.png` | HPA: mínimo 2, máximo 5, objetivo 50 % |
| 08 | `08-cambio-en-el-repositorio.jpeg` | Commit del cambio en `main` |
| 09 | `09-sincronizacion-automatica.png` | Sincronización automática y nuevo ReplicaSet |
| 10 | `10-escalado-a-5-replicas.png` | Escalado automático 2 → 5 con carga (482 % de CPU) |
| 11 | `11-reduccion-a-2-replicas.png` | Reducción automática a 2 réplicas |
| 12 | `12-eliminacion-manual-del-service.png` | Eliminación manual del Service |
| 13 | `13-autorreparacion-selfheal.png` | Argo CD recrea el Service (selfHeal) |

Las capturas se describen en detalle en el informe de la actividad.

## 12. Conclusiones

El uso de Argo CD transforma la entrega en un proceso declarativo: el clúster deja de
configurarse con comandos manuales y pasa a reconciliarse continuamente contra Git.
Esto aporta trazabilidad (todo cambio queda en el historial de commits),
reproducibilidad (el mismo repositorio reconstruye el entorno) y recuperación ante
desviaciones (selfHeal).

## 13. Repositorio

<https://github.com/JustinGomezcoello/udla-argocd-gitops>

# rd-fluxcd-lesson

## Етап 1. Підготовка репозиторію та Flux CD

Було створено публічний GitHub репозиторій:

```text
rd-fluxcd-lesson
```

Перед bootstrap було створено GitHub Personal Access Token та виконано:

```bash
export GITHUB_TOKEN=XXXXXXXXXXX
```

Flux CD було ініціалізовано в Kubernetes кластері:

```bash
curl -s https://fluxcd.io/install.sh | sudo bash

flux bootstrap github \
  --owner=YevhenMarholin \
  --repository=rd-fluxcd-lesson \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal

► connecting to github.com
► cloning branch "main" from Git repository "https://github.com/YevhenMarholin/rd-fluxcd-lesson.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ committed component manifests to "main" ("09c1077aade847c66a749c05f8164c8b7dcd43f6")
► pushing component manifests to "https://github.com/YevhenMarholin/rd-fluxcd-lesson.git"
► installing components in "flux-system" namespace
✔ installed components
✔ reconciled components
► determining if source secret "flux-system/flux-system" exists
► generating source secret
✔ public key: ecdsa-sha2-nistp384 AAAAE2VjZHNhLXNoYTItbmlzdHAzODQAAAAIbmlzdHAzODQAAABhBKSyLbsTpKesXvHz1lbyJpL9liKumFgG5dty1qtIXgKWG9foAHifHEqVL4R3UEvt4AtSxcMHquJia6vOZS8qft43NYYkxOXlr85BopKnQp3KceOhdmAX1c09VeNAOGmuvQ==
✔ configured deploy key "flux-system-main-flux-system-./clusters/my-cluster" for "https://github.com/YevhenMarholin/rd-fluxcd-lesson"
► applying source secret "flux-system/flux-system"
✔ reconciled source secret
► generating sync manifests
✔ generated sync manifests
✔ committed sync manifests to "main" ("d4046e839e282b323e45209caf07efbaaca270c3")
► pushing sync manifests to "https://github.com/YevhenMarholin/rd-fluxcd-lesson.git"
► applying sync manifests
✔ reconciled sync configuration
◎ waiting for GitRepository "flux-system/flux-system" to be reconciled
✔ GitRepository reconciled successfully
◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
✔ Kustomization reconciled successfully
► confirming components are healthy
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```

Після bootstrap у репозиторії автоматично зʼявилась директорія:

```text
clusters/my-cluster/flux-system
```

Перевірка Flux:

```bash
flux check

► checking prerequisites
✗ Kubernetes version v1.30.0 does not match >=1.33.0-0
► checking version in cluster
✔ distribution: flux-v2.8.7
✔ bootstrapped: true
► checking controllers
✔ helm-controller: deployment ready
► ghcr.io/fluxcd/helm-controller:v1.5.4
✔ kustomize-controller: deployment ready
► ghcr.io/fluxcd/kustomize-controller:v1.8.5
✔ notification-controller: deployment ready
► ghcr.io/fluxcd/notification-controller:v1.8.4
✔ source-controller: deployment ready
► ghcr.io/fluxcd/source-controller:v1.8.4
► checking crds
✔ alerts.notification.toolkit.fluxcd.io/v1beta3
✔ buckets.source.toolkit.fluxcd.io/v1
✔ externalartifacts.source.toolkit.fluxcd.io/v1
✔ gitrepositories.source.toolkit.fluxcd.io/v1
✔ helmcharts.source.toolkit.fluxcd.io/v1
✔ helmreleases.helm.toolkit.fluxcd.io/v2
✔ helmrepositories.source.toolkit.fluxcd.io/v1
✔ kustomizations.kustomize.toolkit.fluxcd.io/v1
✔ ocirepositories.source.toolkit.fluxcd.io/v1
✔ providers.notification.toolkit.fluxcd.io/v1beta3
✔ receivers.notification.toolkit.fluxcd.io/v1
```

```bash
flux get sources git
flux get kustomizations
```

---

## Етап 2. Структура репозиторію

У репозиторії було створено структуру:

```text
rd-fluxcd-lesson/
├── apps/
│   └── course-app/
│       ├── base/
│       └── overlays/
│           ├── development/
│           └── production/
└── clusters/
    └── my-cluster/
        ├── flux-system/
        ├── app-dev.yaml
        └── app-prod.yaml
```

---

## Етап 3. Kustomize Base

Було створено базові маніфести для `course-app`.

### `apps/course-app/base/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: course-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: course-app
  template:
    metadata:
      labels:
        app: course-app
    spec:
      containers:
        - name: course-app
          image: djmen12/course-app:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          env:
            - name: APP_STORE
              value: redis
            - name: APP_REDIS_URL
              value: redis://dragonfly:6379
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

### `apps/course-app/base/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: course-app
spec:
  type: ClusterIP
  selector:
    app: course-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

### `apps/course-app/base/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: course-app
spec:
  ingressClassName: nginx
  rules:
    - host: course-app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: course-app
                port:
                  number: 8080
```

### `apps/course-app/base/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

---

## Етап 4. Development overlay

Для development створено окремий namespace та налаштовано 1 репліку застосунку.

### `apps/course-app/overlays/development/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
```

### `apps/course-app/overlays/development/dragonfly.yaml`

```yaml
apiVersion: dragonflydb.io/v1alpha1
kind: Dragonfly
metadata:
  name: dragonfly
  namespace: development
spec:
  replicas: 1
  image: docker.dragonflydb.io/dragonflydb/dragonfly:latest
```

### `apps/course-app/overlays/development/patch-replicas.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: course-app
spec:
  replicas: 1
```

### `apps/course-app/overlays/development/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: development

resources:
  - ../../base
  - namespace.yaml
  - dragonfly.yaml

patches:
  - path: patch-replicas.yaml
```

---

## Етап 5. Production overlay

Для production створено окремий namespace, 3 репліки застосунку, Dragonfly з 2 репліками та HPA.

### `apps/course-app/overlays/production/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

### `apps/course-app/overlays/production/dragonfly.yaml`

```yaml
apiVersion: dragonflydb.io/v1alpha1
kind: Dragonfly
metadata:
  name: dragonfly
  namespace: production
spec:
  replicas: 2
  image: docker.dragonflydb.io/dragonflydb/dragonfly:latest
```

### `apps/course-app/overlays/production/patch-replicas.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: course-app
spec:
  replicas: 3
```

### `apps/course-app/overlays/production/patch-resources.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: course-app
spec:
  template:
    spec:
      containers:
        - name: course-app
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

### `apps/course-app/overlays/production/hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: course-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: course-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### `apps/course-app/overlays/production/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base
  - namespace.yaml
  - dragonfly.yaml
  - hpa.yaml

patches:
  - path: patch-replicas.yaml
  - path: patch-resources.yaml
```

---

## Етап 6. Підключення overlays до Flux

У папці `clusters/my-cluster` було створено два Flux Kustomization ресурси.

### `clusters/my-cluster/app-dev.yaml`

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: app-dev
  namespace: flux-system
spec:
  interval: 1m
  path: ./apps/course-app/overlays/development
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  targetNamespace: development
  wait: true
  timeout: 2m
```

### `clusters/my-cluster/app-prod.yaml`

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: app-prod
  namespace: flux-system
spec:
  interval: 1m
  path: ./apps/course-app/overlays/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  targetNamespace: production
  wait: true
  timeout: 2m
```

Після цього зміни було додано в Git:

```bash
git add .
git commit -m "Add Flux Kustomize apps"
git push origin main
```

Flux автоматично підтягнув зміни з репозиторію.

---

## Етап 7. Перевірка Flux

Перевірка GitRepository:

```bash
flux get sources git
```

Перевірка Kustomization:

```bash
flux get kustomizations
```

Очікувано:

```bash
NAME          READY
flux-system   True
app-dev       True
app-prod      True
```

---

## Етап 8. Перевірка namespace

```bash
kubectl get ns
```

Очікувано:

```bash
development
production
```

---

## Етап 9. Перевірка development

```bash
kubectl get pods -n development
```

Очікувано:

```bash
course-app-...   1/1   Running
dragonfly-...    1/1   Running
```

---

## Етап 10. Перевірка production

```bash
kubectl get pods -n production
```

Очікувано:

```bash
course-app-...   1/1   Running
course-app-...   1/1   Running
course-app-...   1/1   Running
dragonfly-...    1/1   Running
dragonfly-...    1/1   Running
```

Перевірка HPA:

```bash
kubectl get hpa -n production
```

Очікувано:

```bash
NAME         REFERENCE               MINPODS   MAXPODS
course-app   Deployment/course-app   3         10
```

---

## Етап 11. Drift Check

Для перевірки GitOps reconciliation було вручну видалено Service у production:

```bash
kubectl delete svc course-app -n production
```

Після цього Flux автоматично відновив ресурс.

Перевірка:

```bash
kubectl get svc -n production
```

Очікувано:

```bash
course-app   ClusterIP   8080/TCP
```

---

## Висновок

У межах домашнього завдання було:

- створено публічний GitHub репозиторій `rd-fluxcd-lesson`
- ініціалізовано Flux CD через `flux bootstrap github`
- створено Kustomize base для `course-app`
- створено overlays для `development` та `production`
- у development налаштовано 1 репліку застосунку
- у production налаштовано 3 репліки застосунку
- у production додано resources та HPA
- додано Dragonfly через Custom Resource
- підключено overlays до Flux через Kustomization resources
- перевірено, що Flux автоматично застосовує зміни
- виконано drift check: видалений Service було автоматично відновлено Flux CD

Посилання на репозиторій:

```text
https://github.com/<YOUR_GITHUB_USERNAME>/rd-fluxcd-lesson
```
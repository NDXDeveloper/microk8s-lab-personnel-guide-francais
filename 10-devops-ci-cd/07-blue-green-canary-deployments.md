🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.7 Blue-Green et Canary Deployments

## Introduction aux stratégies de déploiement avancées

Lorsque vous mettez à jour une application en production, le risque principal est d'introduire un bug qui affecte tous vos utilisateurs simultanément. Les stratégies Blue-Green et Canary sont des techniques qui minimisent ce risque en contrôlant comment les nouvelles versions sont déployées et exposées aux utilisateurs.

### Le problème des déploiements traditionnels

Dans un déploiement classique (appelé "recreate" ou "rolling update"), vous remplacez progressivement l'ancienne version par la nouvelle. Si un problème survient, tous les utilisateurs sont potentiellement impactés et le retour en arrière peut prendre du temps.

### Les solutions modernes

- **Blue-Green** : Vous maintenez deux environnements identiques et basculez instantanément entre eux
- **Canary** : Vous déployez la nouvelle version pour un petit pourcentage d'utilisateurs d'abord, puis augmentez progressivement

Ces stratégies permettent de détecter les problèmes rapidement et de revenir en arrière instantanément si nécessaire.

## Blue-Green Deployment

### Concept

Le déploiement Blue-Green fonctionne avec deux environnements complets :
- **Blue** : L'environnement actuellement en production
- **Green** : Le nouvel environnement avec la version mise à jour

Vous testez Green en isolation, puis basculez tout le trafic de Blue vers Green d'un coup. Si un problème survient, vous rebasculez immédiatement vers Blue.

### Architecture Blue-Green dans Kubernetes

```
                    Ingress/Service
                          |
                    [Selector: version=blue]
                    /            \
            Deployment-Blue    Deployment-Green
            (v1.0 - Actif)     (v2.0 - En attente)
```

Après le basculement :
```
                    Ingress/Service
                          |
                    [Selector: version=green]
                    /            \
            Deployment-Blue    Deployment-Green
            (v1.0 - En attente)  (v2.0 - Actif)
```

### Implémentation manuelle Blue-Green

#### Étape 1 : Déploiement Blue (version actuelle)

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-blue
  labels:
    app: mon-app
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-app
      version: blue
  template:
    metadata:
      labels:
        app: mon-app
        version: blue
    spec:
      containers:
      - name: mon-app
        image: localhost:32000/mon-app:v1.0
        ports:
        - containerPort: 8080
        env:
        - name: VERSION
          value: "1.0-blue"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

#### Étape 2 : Service pointant vers Blue

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mon-app-service
spec:
  selector:
    app: mon-app
    version: blue  # Pointe vers Blue initialement
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
# Ingress pour accès externe
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: mon-app.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app-service
            port:
              number: 80
```

#### Étape 3 : Déploiement Green (nouvelle version)

```yaml
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-green
  labels:
    app: mon-app
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mon-app
      version: green
  template:
    metadata:
      labels:
        app: mon-app
        version: green
    spec:
      containers:
      - name: mon-app
        image: localhost:32000/mon-app:v2.0  # Nouvelle version
        ports:
        - containerPort: 8080
        env:
        - name: VERSION
          value: "2.0-green"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

#### Étape 4 : Test de Green avant basculement

```yaml
# test-service-green.yaml
# Service temporaire pour tester Green
apiVersion: v1
kind: Service
metadata:
  name: mon-app-test-green
spec:
  selector:
    app: mon-app
    version: green
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

```bash
# Créer un pod de test pour accéder au service Green
kubectl run test-pod --image=curlimages/curl:latest -it --rm -- sh

# Dans le pod, tester Green
curl http://mon-app-test-green/health
curl http://mon-app-test-green/api/version
```

#### Étape 5 : Basculement de Blue vers Green

```bash
# Méthode 1 : Patch du service
kubectl patch service mon-app-service -p '{"spec":{"selector":{"version":"green"}}}'

# Méthode 2 : Appliquer un nouveau manifeste
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: mon-app-service
spec:
  selector:
    app: mon-app
    version: green  # Changé de blue à green
  ports:
  - port: 80
    targetPort: 8080
EOF
```

#### Étape 6 : Rollback si nécessaire

```bash
# Retour immédiat vers Blue
kubectl patch service mon-app-service -p '{"spec":{"selector":{"version":"blue"}}}'

# Vérification
kubectl get endpoints mon-app-service
```

### Script d'automatisation Blue-Green

```bash
#!/bin/bash
# blue-green-deploy.sh

APP_NAME="mon-app"
NAMESPACE="default"
NEW_VERSION=$1

if [ -z "$NEW_VERSION" ]; then
  echo "Usage: $0 <new-version>"
  exit 1
fi

# Déterminer la couleur actuelle
CURRENT_COLOR=$(kubectl get service $APP_NAME-service -o jsonpath='{.spec.selector.version}')
if [ "$CURRENT_COLOR" == "blue" ]; then
  NEW_COLOR="green"
else
  NEW_COLOR="blue"
fi

echo "Déploiement Blue-Green"
echo "Version actuelle: $CURRENT_COLOR"
echo "Nouvelle version: $NEW_COLOR ($NEW_VERSION)"

# Déployer la nouvelle version
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $APP_NAME-$NEW_COLOR
spec:
  replicas: 3
  selector:
    matchLabels:
      app: $APP_NAME
      version: $NEW_COLOR
  template:
    metadata:
      labels:
        app: $APP_NAME
        version: $NEW_COLOR
    spec:
      containers:
      - name: $APP_NAME
        image: localhost:32000/$APP_NAME:$NEW_VERSION
        ports:
        - containerPort: 8080
EOF

# Attendre que le déploiement soit prêt
echo "Attente du déploiement $NEW_COLOR..."
kubectl wait --for=condition=available --timeout=300s deployment/$APP_NAME-$NEW_COLOR

# Tests de santé
echo "Tests de santé sur $NEW_COLOR..."
kubectl run test-$RANDOM --image=curlimages/curl:latest --rm -it --restart=Never -- \
  curl http://$APP_NAME-$NEW_COLOR:8080/health

read -p "Basculer vers $NEW_COLOR? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
  kubectl patch service $APP_NAME-service -p "{\"spec\":{\"selector\":{\"version\":\"$NEW_COLOR\"}}}"
  echo "Basculement effectué vers $NEW_COLOR"

  # Optionnel : supprimer l'ancien déploiement après stabilisation
  read -p "Supprimer l'ancien déploiement $CURRENT_COLOR? (y/n) " -n 1 -r
  echo
  if [[ $REPLY =~ ^[Yy]$ ]]; then
    kubectl delete deployment $APP_NAME-$CURRENT_COLOR
  fi
else
  echo "Basculement annulé"
  kubectl delete deployment $APP_NAME-$NEW_COLOR
fi
```

## Canary Deployment

### Concept

Le déploiement Canary tire son nom de la pratique historique des mineurs qui utilisaient des canaris pour détecter les gaz toxiques. De la même façon, vous déployez la nouvelle version pour un petit groupe d'utilisateurs "canaris" qui servent de test avant le déploiement complet.

### Architecture Canary dans Kubernetes

```
                    Ingress/Service
                          |
                  [Load Balancing]
                    /          \
                  90%          10%
                   |            |
            Deployment-Stable  Deployment-Canary
              (v1.0 - 9 pods)   (v2.0 - 1 pod)
```

### Implémentation avec répartition simple

#### Méthode 1 : Canary avec un seul Service

```yaml
# stable-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-stable
spec:
  replicas: 9  # 90% du trafic
  selector:
    matchLabels:
      app: mon-app
      track: stable
  template:
    metadata:
      labels:
        app: mon-app
        track: stable
        version: v1.0
    spec:
      containers:
      - name: mon-app
        image: localhost:32000/mon-app:v1.0
        ports:
        - containerPort: 8080
        env:
        - name: VERSION
          value: "1.0-stable"
---
# canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app-canary
spec:
  replicas: 1  # 10% du trafic
  selector:
    matchLabels:
      app: mon-app
      track: canary
  template:
    metadata:
      labels:
        app: mon-app
        track: canary
        version: v2.0
    spec:
      containers:
      - name: mon-app
        image: localhost:32000/mon-app:v2.0
        ports:
        - containerPort: 8080
        env:
        - name: VERSION
          value: "2.0-canary"
---
# Service qui route vers les deux
apiVersion: v1
kind: Service
metadata:
  name: mon-app-service
spec:
  selector:
    app: mon-app  # Sélectionne stable ET canary
  ports:
  - port: 80
    targetPort: 8080
```

#### Méthode 2 : Canary avec Ingress NGINX

```yaml
# ingress-canary.yaml
# Ingress principal (stable)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app-ingress-stable
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: mon-app.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app-stable
            port:
              number: 80
---
# Ingress canary (10% du trafic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app-ingress-canary
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% du trafic
spec:
  ingressClassName: nginx
  rules:
  - host: mon-app.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app-canary
            port:
              number: 80
```

### Canary progressif avec automatisation

```yaml
# canary-rollout.yaml
# Script pour augmenter progressivement le trafic canary
apiVersion: v1
kind: ConfigMap
metadata:
  name: canary-rollout-script
data:
  rollout.sh: |
    #!/bin/bash
    APP_NAME="mon-app"
    CANARY_STEPS="10 25 50 75 100"
    STABILITY_WAIT=300  # 5 minutes entre chaque étape

    for PERCENTAGE in $CANARY_STEPS; do
      echo "Augmentation du trafic canary à ${PERCENTAGE}%"

      # Mettre à jour le poids canary
      kubectl annotate ingress $APP_NAME-ingress-canary \
        nginx.ingress.kubernetes.io/canary-weight="${PERCENTAGE}" \
        --overwrite

      # Vérifier les métriques
      echo "Surveillance des métriques pendant ${STABILITY_WAIT} secondes..."

      # Vérifier le taux d'erreur
      ERROR_RATE=$(kubectl exec deployment/prometheus -- \
        promtool query instant \
        'rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])' | \
        grep -o '[0-9.]*' | head -1)

      if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
        echo "Taux d'erreur trop élevé: ${ERROR_RATE}"
        echo "Rollback automatique..."
        kubectl annotate ingress $APP_NAME-ingress-canary \
          nginx.ingress.kubernetes.io/canary-weight="0" \
          --overwrite
        exit 1
      fi

      sleep $STABILITY_WAIT
    done

    echo "Déploiement canary terminé avec succès!"
```

### Canary basé sur les headers ou cookies

```yaml
# canary-by-header.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app-ingress-canary-header
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "x-canary"
    nginx.ingress.kubernetes.io/canary-by-header-value: "always"
spec:
  ingressClassName: nginx
  rules:
  - host: mon-app.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app-canary
            port:
              number: 80
---
# canary-by-cookie.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app-ingress-canary-cookie
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-cookie: "canary"
spec:
  ingressClassName: nginx
  rules:
  - host: mon-app.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-app-canary
            port:
              number: 80
```

## Utilisation avec Flagger

Flagger est un outil qui automatise les déploiements progressifs (canary, blue-green, A/B testing).

### Installation de Flagger sur MicroK8s

```bash
# Installer Flagger
kubectl apply -k github.com/fluxcd/flagger/kustomize/kubernetes

# Installer Flagger pour NGINX Ingress
kubectl apply -k github.com/fluxcd/flagger/kustomize/ingress-nginx

# Vérifier l'installation
kubectl -n flagger-system get pods
```

### Configuration d'un Canary avec Flagger

```yaml
# flagger-canary.yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: mon-app
spec:
  # Deployment cible
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app

  # Configuration du service
  service:
    port: 80
    targetPort: 8080
    gateways:
    - public-gateway.istio-system.svc.cluster.local
    hosts:
    - mon-app.monlab.local

  # Stratégie de déploiement progressif
  analysis:
    # Intervalle entre les analyses
    interval: 1m

    # Seuil avant de considérer le canary comme réussi
    threshold: 10

    # Pourcentage maximum de trafic canary
    maxWeight: 50

    # Incrément du trafic à chaque étape
    stepWeight: 10

    # Métriques pour valider le canary
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m

    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m

    # Webhooks pour notifications
    webhooks:
    - name: acceptance-test
      url: http://test-runner.test/
      timeout: 30s
      metadata:
        type: pre-rollout
        cmd: "curl -sd 'test=all' http://mon-app-canary/test"

    - name: load-test
      url: http://test-runner.test/
      metadata:
        cmd: "k6 run -< /scripts/load-test.js"
```

### Déclenchement d'un déploiement avec Flagger

```bash
# Mettre à jour l'image du deployment
kubectl set image deployment/mon-app \
  mon-app=localhost:32000/mon-app:v2.0

# Surveiller le déploiement progressif
kubectl get canary mon-app -w

# Voir les événements
kubectl describe canary mon-app

# Métriques Flagger
kubectl -n flagger-system logs deployment/flagger -f | jq .
```

## ArgoCD pour Blue-Green et Canary

### Configuration Blue-Green avec ArgoCD

```yaml
# argo-rollout-bluegreen.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: mon-app-rollout
spec:
  replicas: 3
  strategy:
    blueGreen:
      # Service actif (production)
      activeService: mon-app-active

      # Service de preview (test)
      previewService: mon-app-preview

      # Promotion automatique après 10 minutes
      autoPromotionEnabled: true
      autoPromotionSeconds: 600

      # Garder l'ancienne version pour rollback rapide
      scaleDownDelaySeconds: 30
      scaleDownDelayRevisionLimit: 2

      # Tests de validation
      prePromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: mon-app-preview

      # Tests post-promotion
      postPromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: mon-app-active

  selector:
    matchLabels:
      app: mon-app

  template:
    metadata:
      labels:
        app: mon-app
    spec:
      containers:
      - name: mon-app
        image: localhost:32000/mon-app:v1.0
        ports:
        - containerPort: 8080
```

### Configuration Canary avec ArgoCD

```yaml
# argo-rollout-canary.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: mon-app-rollout
spec:
  replicas: 10
  strategy:
    canary:
      # Étapes du déploiement progressif
      steps:
      - setWeight: 10
      - pause: {duration: 5m}
      - setWeight: 20
      - pause: {duration: 5m}
      - setWeight: 40
      - pause: {duration: 10m}
      - setWeight: 60
      - pause: {duration: 10m}
      - setWeight: 80
      - pause: {duration: 10m}

      # Configuration du trafic
      trafficRouting:
        nginx:
          stableIngress: mon-app-stable
          additionalIngressAnnotations:
            canary-by-header: X-Canary

      # Analyse continue
      analysis:
        templates:
        - templateName: success-rate
        startingStep: 2  # Commence l'analyse à 20%
        args:
        - name: service-name
          value: mon-app-canary

      # Limites de rollback
      maxSurge: "25%"
      maxUnavailable: 0

      # Anti-affinity pour distribution
      antiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution: {}

  selector:
    matchLabels:
      app: mon-app

  template:
    metadata:
      labels:
        app: mon-app
    spec:
      containers:
      - name: mon-app
        image: localhost:32000/mon-app:v1.0
        ports:
        - containerPort: 8080
```

### Template d'analyse pour validation

```yaml
# analysis-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 5m
    successCondition: result[0] >= 0.95
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(
            http_requests_total{job="{{args.service-name}}",status=~"2.."}[5m]
          )) /
          sum(rate(
            http_requests_total{job="{{args.service-name}}"}[5m]
          ))
  - name: latency
    interval: 5m
    successCondition: result[0] < 1000
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket{job="{{args.service-name}}"}[5m]))
            by (le)
          ) * 1000
```

## Monitoring et observabilité

### Dashboard Grafana pour Blue-Green/Canary

```json
{
  "dashboard": {
    "title": "Blue-Green & Canary Deployments",
    "panels": [
      {
        "title": "Traffic Distribution",
        "targets": [
          {
            "expr": "sum(rate(nginx_ingress_controller_requests[5m])) by (ingress)",
            "legendFormat": "{{ ingress }}"
          }
        ]
      },
      {
        "title": "Success Rate by Version",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"2..\"}[5m])) by (version) / sum(rate(http_requests_total[5m])) by (version)",
            "legendFormat": "{{ version }}"
          }
        ]
      },
      {
        "title": "Response Time by Version",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (version, le))",
            "legendFormat": "p95 {{ version }}"
          }
        ]
      },
      {
        "title": "Error Rate Comparison",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) by (version)",
            "legendFormat": "{{ version }}"
          }
        ]
      }
    ]
  }
}
```

### Alertes Prometheus

```yaml
# prometheus-alerts.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: deployment-alerts
data:
  alerts.yaml: |
    groups:
    - name: deployment
      rules:
      - alert: CanaryHighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{version="canary",status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total{version="canary"}[5m]))
          ) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate in canary deployment"
          description: "Canary version has {{ $value | humanizePercentage }} error rate"

      - alert: CanaryHighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket{version="canary"}[5m])) by (le)
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency in canary deployment"
          description: "Canary p95 latency is {{ $value }}s"

      - alert: BlueGreenTrafficImbalance
        expr: |
          abs(
            sum(rate(nginx_ingress_controller_requests{service="mon-app-blue"}[5m])) -
            sum(rate(nginx_ingress_controller_requests{service="mon-app-green"}[5m]))
          ) > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Traffic imbalance between Blue and Green"
          description: "Unexpected traffic distribution during Blue-Green deployment"
```

## Scripts utilitaires

### Validation avant basculement

```bash
#!/bin/bash
# validate-deployment.sh

DEPLOYMENT=$1
CHECKS_PASSED=0
CHECKS_FAILED=0

echo "Validation du déploiement: $DEPLOYMENT"

# Check 1: Tous les pods sont ready
READY=$(kubectl get deployment $DEPLOYMENT -o jsonpath='{.status.readyReplicas}')
DESIRED=$(kubectl get deployment $DEPLOYMENT -o jsonpath='{.spec.replicas}')

if [ "$READY" == "$DESIRED" ]; then
  echo "✓ Tous les pods sont prêts ($READY/$DESIRED)"
  ((CHECKS_PASSED++))
else
  echo "✗ Pods non prêts ($READY/$DESIRED)"
  ((CHECKS_FAILED++))
fi

# Check 2: Pas de redémarrages récents
RESTARTS=$(kubectl get pods -l deployment=$DEPLOYMENT -o jsonpath='{.items[*].status.containerStatuses[*].restartCount}' | tr ' ' '+' | bc)

if [ "$RESTARTS" -eq 0 ]; then
  echo "✓ Aucun redémarrage détecté"
  ((CHECKS_PASSED++))
else
  echo "✗ $RESTARTS redémarrages détectés"
  ((CHECKS_FAILED++))
fi

# Check 3: Test de santé HTTP
POD=$(kubectl get pod -l deployment=$DEPLOYMENT -o jsonpath='{.items[0].metadata.name}')
HTTP_STATUS=$(kubectl exec $POD -- curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health)

if [ "$HTTP_STATUS" == "200" ]; then
  echo "✓ Endpoint de santé répond correctement"
  ((CHECKS_PASSED++))
else
  echo "✗ Endpoint de santé en erreur (HTTP $HTTP_STATUS)"
  ((CHECKS_FAILED++))
fi

# Check 4: Consommation des ressources
CPU=$(kubectl top pod -l deployment=$DEPLOYMENT --no-headers | awk '{sum+=$2} END {print sum}')
MEMORY=$(kubectl top pod -l deployment=$DEPLOYMENT --no-headers | awk '{sum+=$3} END {print sum}')

echo "ℹ Consommation: CPU=${CPU}m, Memory=${MEMORY}Mi"

# Résultat final
echo ""
echo "Résultat: $CHECKS_PASSED validations réussies, $CHECKS_FAILED échecs"

if [ $CHECKS_FAILED -eq 0 ]; then
  echo "✅ Déploiement validé avec succès"
  exit 0
else
  echo "❌ Validation échouée"
  exit 1
fi
```

### Comparaison de performances

```bash
#!/bin/bash
# compare-versions.sh

OLD_VERSION=$1
NEW_VERSION=$2
DURATION=${3:-60}  # Durée du test en secondes

echo "Comparaison des performances"
echo "Version actuelle: $OLD_VERSION"
echo "Nouvelle version: $NEW_VERSION"
echo "Durée du test: ${DURATION}s"

# Test de l'ancienne version
echo ""
echo "Test de $OLD_VERSION..."
kubectl port-forward service/$OLD_VERSION-service 8081:80 &
PF_PID=$!
sleep 5

OLD_RESULTS=$(ab -n 1000 -c 10 -t $DURATION http://localhost:8081/ | grep -E "Requests per second|Time per request|Transfer rate")
kill $PF_PID 2>/dev/null

# Test de la nouvelle version
echo ""
echo "Test de $NEW_VERSION..."
kubectl port-forward service/$NEW_VERSION-service 8082:80 &
PF_PID=$!
sleep 5

NEW_RESULTS=$(ab -n 1000 -c 10 -t $DURATION http://localhost:8082/ | grep -E "Requests per second|Time per request|Transfer rate")
kill $PF_PID 2>/dev/null

# Affichage des résultats
echo ""
echo "=== Résultats ==="
echo "Version $OLD_VERSION:"
echo "$OLD_RESULTS"
echo ""
echo "Version $NEW_VERSION:"
echo "$NEW_RESULTS"

# Analyse basique
OLD_RPS=$(echo "$OLD_RESULTS" | grep "Requests per second" | awk '{print $4}')
NEW_RPS=$(echo "$NEW_RESULTS" | grep "Requests per second" | awk '{print $4}')

if (( $(echo "$NEW_RPS > $OLD_RPS" | bc -l) )); then
  IMPROVEMENT=$(echo "scale=2; (($NEW_RPS - $OLD_RPS) / $OLD_RPS) * 100" | bc)
  echo ""
  echo "✅ Amélioration des performances: +${IMPROVEMENT}%"
else
  DEGRADATION=$(echo "scale=2; (($OLD_RPS - $NEW_RPS) / $OLD_RPS) * 100" | bc)
  echo ""
  echo "⚠️ Dégradation des performances: -${DEGRADATION}%"
fi
```

## Bonnes pratiques

### 1. Tests automatisés avant basculement

Toujours valider la nouvelle version avant de router du trafic :

```yaml
# pre-deployment-tests.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pre-deployment-validation
spec:
  template:
    spec:
      containers:
      - name: validator
        image: localhost:32000/deployment-validator:latest
        command: ["/validate.sh"]
        env:
        - name: TARGET_DEPLOYMENT
          value: "mon-app-green"
        - name: REQUIRED_TESTS
          value: "health,api,database,performance"
      restartPolicy: Never
  backoffLimit: 3
```

### 2. Stratégie de rollback claire

Définissez des critères de rollback automatique :

```yaml
# rollback-policy.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rollback-policy
data:
  policy.yaml: |
    triggers:
      - metric: error_rate
        threshold: 0.05  # 5% d'erreurs
        duration: 2m
        action: immediate_rollback

      - metric: p95_latency
        threshold: 1000  # 1 seconde
        duration: 5m
        action: gradual_rollback

      - metric: cpu_usage
        threshold: 80  # 80%
        duration: 10m
        action: alert_then_rollback

      - metric: memory_usage
        threshold: 90  # 90%
        duration: 5m
        action: immediate_rollback
```

### 3. Labels et annotations cohérents

Utilisez des conventions de nommage claires :

```yaml
metadata:
  labels:
    app: mon-app                    # Application
    version: v2.0.0                  # Version sémantique
    track: canary                    # Type de déploiement
    environment: production          # Environnement
  annotations:
    deployment-strategy: canary      # Stratégie utilisée
    rollout-weight: "10"            # Pourcentage de trafic
    promotion-time: "2024-01-15T10:00:00Z"  # Heure de promotion
    deployed-by: "CI/CD"            # Source du déploiement
```

### 4. Monitoring granulaire

Instrumentez votre code pour distinguer les versions :

```javascript
// Exemple Node.js avec Prometheus
const prometheus = require('prom-client');

const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status', 'version'],
  buckets: [0.1, 0.5, 1, 2, 5]
});

app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestDuration.observe({
      method: req.method,
      route: req.route?.path || 'unknown',
      status: res.statusCode,
      version: process.env.VERSION || 'unknown'
    }, duration);
  });

  next();
});
```

### 5. Communication et documentation

Documentez vos déploiements :

```yaml
# deployment-record.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: deployment-history
data:
  deployments.json: |
    [
      {
        "date": "2024-01-15T10:00:00Z",
        "version": "v2.0.0",
        "strategy": "canary",
        "duration": "45m",
        "rollback": false,
        "issues": [],
        "metrics": {
          "error_rate_before": 0.001,
          "error_rate_after": 0.0008,
          "latency_p95_before": 250,
          "latency_p95_after": 230
        }
      }
    ]
```

### 6. Tests de chaos pendant Canary

Testez la résilience pendant le déploiement :

```yaml
# chaos-test-canary.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: canary-network-delay
spec:
  action: delay
  mode: one
  selector:
    labelSelectors:
      "track": "canary"
  delay:
    latency: "100ms"
    jitter: "10ms"
  duration: "5m"
  scheduler:
    cron: "*/10 * * * *"  # Toutes les 10 minutes
```

## Choix de la stratégie

### Quand utiliser Blue-Green ?

✅ **Utilisez Blue-Green quand :**
- Vous avez besoin de rollbacks instantanés
- Votre application ne supporte pas plusieurs versions simultanées
- Vous avez des migrations de base de données complexes
- Vous voulez tester complètement avant de basculer
- Vous avez suffisamment de ressources pour dupliquer l'environnement

❌ **Évitez Blue-Green quand :**
- Les ressources sont limitées
- Vous avez des sessions utilisateur stateful
- Le coût de duplication est prohibitif

### Quand utiliser Canary ?

✅ **Utilisez Canary quand :**
- Vous voulez minimiser l'impact des bugs
- Vous avez besoin de feedback progressif
- Votre application supporte plusieurs versions
- Vous voulez collecter des métriques comparatives
- Vous avez une base d'utilisateurs large

❌ **Évitez Canary quand :**
- Les changements de schéma de base de données sont incompatibles
- Vous ne pouvez pas distinguer le trafic par version
- Le monitoring n'est pas suffisamment mature

## Exemple complet : Pipeline Blue-Green

```yaml
# complete-blue-green-pipeline.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: blue-green-pipeline
data:
  deploy.sh: |
    #!/bin/bash
    set -e

    # Configuration
    APP_NAME="mon-app"
    NEW_VERSION=$1
    NAMESPACE=${2:-default}
    HEALTH_CHECK_RETRIES=30
    HEALTH_CHECK_INTERVAL=10

    # Fonctions utilitaires
    log() { echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"; }
    error() { log "ERROR: $1" >&2; exit 1; }

    # Déterminer les couleurs
    CURRENT_COLOR=$(kubectl get service ${APP_NAME}-service -n $NAMESPACE \
      -o jsonpath='{.spec.selector.version}' 2>/dev/null || echo "none")

    if [ "$CURRENT_COLOR" == "blue" ]; then
      NEW_COLOR="green"
    else
      NEW_COLOR="blue"
    fi

    log "Configuration du déploiement"
    log "  Application: $APP_NAME"
    log "  Version: $NEW_VERSION"
    log "  Environnement actuel: $CURRENT_COLOR"
    log "  Nouvel environnement: $NEW_COLOR"

    # Phase 1: Déploiement
    log "Phase 1: Déploiement de $NEW_COLOR"
    kubectl apply -f - <<EOF
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ${APP_NAME}-${NEW_COLOR}
      namespace: $NAMESPACE
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: $APP_NAME
          version: $NEW_COLOR
      template:
        metadata:
          labels:
            app: $APP_NAME
            version: $NEW_COLOR
        spec:
          containers:
          - name: $APP_NAME
            image: localhost:32000/${APP_NAME}:${NEW_VERSION}
            ports:
            - containerPort: 8080
            env:
            - name: VERSION
              value: "${NEW_VERSION}-${NEW_COLOR}"
            readinessProbe:
              httpGet:
                path: /health
                port: 8080
              initialDelaySeconds: 10
              periodSeconds: 5
            livenessProbe:
              httpGet:
                path: /health
                port: 8080
              initialDelaySeconds: 30
              periodSeconds: 10
            resources:
              requests:
                memory: "128Mi"
                cpu: "100m"
              limits:
                memory: "256Mi"
                cpu: "200m"
    EOF

    # Phase 2: Attente de la disponibilité
    log "Phase 2: Attente de la disponibilité de $NEW_COLOR"
    kubectl wait --for=condition=available --timeout=600s \
      deployment/${APP_NAME}-${NEW_COLOR} -n $NAMESPACE || \
      error "Le déploiement n'est pas disponible après 10 minutes"

    # Phase 3: Tests de santé
    log "Phase 3: Tests de santé sur $NEW_COLOR"

    # Créer un service temporaire pour les tests
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: Service
    metadata:
      name: ${APP_NAME}-${NEW_COLOR}-test
      namespace: $NAMESPACE
    spec:
      selector:
        app: $APP_NAME
        version: $NEW_COLOR
      ports:
      - port: 80
        targetPort: 8080
    EOF

    # Exécuter les tests
    for i in $(seq 1 $HEALTH_CHECK_RETRIES); do
      log "  Test $i/$HEALTH_CHECK_RETRIES"

      if kubectl run test-${RANDOM} \
        --image=curlimages/curl:latest \
        --rm -i --restart=Never -n $NAMESPACE -- \
        curl -f http://${APP_NAME}-${NEW_COLOR}-test/health; then
        log "  ✓ Test de santé réussi"
        break
      else
        if [ $i -eq $HEALTH_CHECK_RETRIES ]; then
          error "Les tests de santé ont échoué après $HEALTH_CHECK_RETRIES tentatives"
        fi
        log "  ✗ Test échoué, nouvelle tentative dans ${HEALTH_CHECK_INTERVAL}s..."
        sleep $HEALTH_CHECK_INTERVAL
      fi
    done

    # Phase 4: Tests de performance
    log "Phase 4: Tests de performance rapides"
    kubectl run perf-test-${RANDOM} \
      --image=jordi/ab \
      --rm -i --restart=Never -n $NAMESPACE -- \
      ab -n 100 -c 10 http://${APP_NAME}-${NEW_COLOR}-test/ || \
      log "  ⚠ Tests de performance non critiques échoués"

    # Phase 5: Basculement
    log "Phase 5: Basculement du trafic vers $NEW_COLOR"
    read -p "Confirmer le basculement vers $NEW_COLOR? (y/n) " -n 1 -r
    echo

    if [[ $REPLY =~ ^[Yy]$ ]]; then
      kubectl patch service ${APP_NAME}-service -n $NAMESPACE \
        -p "{\"spec\":{\"selector\":{\"version\":\"$NEW_COLOR\"}}}"
      log "✅ Basculement effectué avec succès"

      # Phase 6: Vérification post-basculement
      log "Phase 6: Vérification post-basculement"
      sleep 10

      RESPONSE=$(kubectl run verify-${RANDOM} \
        --image=curlimages/curl:latest \
        --rm -i --restart=Never -n $NAMESPACE -- \
        curl -s http://${APP_NAME}-service/version)

      if [[ $RESPONSE == *"$NEW_VERSION"* ]]; then
        log "✅ Nouvelle version confirmée en production"
      else
        log "⚠ Impossible de confirmer la nouvelle version"
      fi

      # Phase 7: Nettoyage optionnel
      log "Phase 7: Nettoyage"
      read -p "Supprimer l'ancien déploiement $CURRENT_COLOR? (y/n) " -n 1 -r
      echo

      if [[ $REPLY =~ ^[Yy]$ ]]; then
        kubectl delete deployment ${APP_NAME}-${CURRENT_COLOR} -n $NAMESPACE || true
        kubectl delete service ${APP_NAME}-${NEW_COLOR}-test -n $NAMESPACE || true
        log "Ancien déploiement supprimé"
      else
        log "Ancien déploiement conservé pour rollback rapide"
      fi
    else
      log "Basculement annulé"
      kubectl delete deployment ${APP_NAME}-${NEW_COLOR} -n $NAMESPACE
      kubectl delete service ${APP_NAME}-${NEW_COLOR}-test -n $NAMESPACE
      error "Déploiement annulé par l'utilisateur"
    fi

    log "✅ Pipeline Blue-Green terminé avec succès"
```

## Ressources et outils complémentaires

### Outils recommandés

- **Flagger** : Automatisation des déploiements progressifs
- **Argo Rollouts** : Stratégies de déploiement avancées
- **Istio** : Service mesh avec capacités de traffic management
- **Linkerd** : Service mesh léger avec support canary
- **Contour** : Ingress controller avec support blue-green natif

### Commandes utiles

```bash
# Surveiller un déploiement
watch -n 2 "kubectl get pods -l app=mon-app --show-labels"

# Vérifier la distribution du trafic
kubectl exec -it deployment/nginx-ingress-controller -- \
  cat /etc/nginx/nginx.conf | grep -A5 "upstream"

# Simuler du trafic pour les tests
while true; do curl http://mon-app.local/; sleep 0.5; done

# Analyser les logs par version
kubectl logs -l app=mon-app,version=canary --tail=100

# Rollback rapide
kubectl rollout undo deployment/mon-app

# Voir l'historique des déploiements
kubectl rollout history deployment/mon-app
```

## Conclusion

Les stratégies Blue-Green et Canary transforment les déploiements risqués en processus contrôlés et mesurables. Dans votre lab MicroK8s, ces techniques vous permettent de :

- **Minimiser les risques** : Détectez les problèmes avant qu'ils n'impactent tous les utilisateurs
- **Rollback instantané** : Revenez en arrière en quelques secondes
- **Tests en production** : Validez avec du trafic réel sur un échantillon
- **Confiance accrue** : Déployez sereinement grâce aux validations progressives
- **Apprentissage continu** : Collectez des métriques comparatives entre versions

Commencez par implémenter manuellement ces stratégies pour bien comprendre les concepts, puis automatisez progressivement avec des outils comme Flagger ou Argo Rollouts. L'investissement initial en configuration sera rapidement rentabilisé par la réduction des incidents en production et l'amélioration de la vélocité de déploiement.

⏭️

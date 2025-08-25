🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.7 Tests de charge

## Introduction aux Tests de Charge

Les tests de charge permettent de valider que votre cluster MicroK8s et vos applications peuvent gérer la charge attendue et identifier leurs limites. C'est comme faire un test d'effort médical : vous poussez progressivement le système à ses limites pour comprendre sa capacité maximale et détecter les faiblesses avant qu'elles ne deviennent critiques. Dans un lab personnel, ces tests sont essentiels pour valider vos configurations de scaling, optimiser l'utilisation des ressources, et apprendre comment votre infrastructure se comporte sous pression.

## Comprendre les Types de Tests

### Les Différents Types de Tests de Performance

**Load Testing (Test de charge)** : Simule la charge normale attendue pour vérifier que le système fonctionne correctement dans des conditions d'utilisation réalistes.

**Stress Testing (Test de stress)** : Pousse le système au-delà de sa capacité normale pour identifier le point de rupture et comprendre comment il échoue.

**Spike Testing (Test de pic)** : Simule des augmentations soudaines et importantes du trafic pour tester la réactivité du scaling automatique.

**Soak Testing (Test d'endurance)** : Maintient une charge constante sur une longue période pour détecter les fuites mémoire et les dégradations progressives.

**Volume Testing** : Teste avec de grandes quantités de données pour identifier les limites de stockage et de traitement.

### Métriques à Surveiller

Pendant les tests de charge, surveillez ces métriques clés :

```yaml
# Métriques de performance
- Latence (P50, P90, P95, P99)
- Throughput (requêtes par seconde)
- Taux d'erreur
- Temps de réponse

# Métriques système
- Utilisation CPU
- Consommation mémoire
- I/O disque
- Bande passante réseau

# Métriques Kubernetes
- Nombre de pods
- Temps de scaling
- Pod restarts
- Evictions
```

## Outils de Test de Charge

### K6 - Outil Moderne et Scriptable

K6 est un outil de test de charge moderne, facile à utiliser et parfait pour un lab :

```javascript
// test-basic.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 10 },   // Montée à 10 users
    { duration: '5m', target: 10 },   // Maintien à 10 users
    { duration: '2m', target: 20 },   // Montée à 20 users
    { duration: '5m', target: 20 },   // Maintien à 20 users
    { duration: '2m', target: 0 },    // Descente à 0
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% des requêtes < 500ms
    http_req_failed: ['rate<0.1'],    // Taux d'erreur < 10%
  },
};

export default function() {
  let response = http.get('http://my-app.default.svc.cluster.local');

  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  sleep(1);
}
```

Déploiement de K6 dans le cluster :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: k6-test
spec:
  template:
    spec:
      containers:
      - name: k6
        image: grafana/k6:latest
        command: ["k6", "run", "/scripts/test.js"]
        volumeMounts:
        - name: test-script
          mountPath: /scripts
      volumes:
      - name: test-script
        configMap:
          name: k6-test-script
      restartPolicy: Never
```

### Locust - Interface Web Intuitive

Locust offre une interface web pour visualiser les tests en temps réel :

```python
# locustfile.py
from locust import HttpUser, task, between

class WebsiteUser(HttpUser):
    wait_time = between(1, 3)

    @task(3)
    def index_page(self):
        self.client.get("/")

    @task(1)
    def view_item(self):
        item_id = random.randint(1, 1000)
        self.client.get(f"/item/{item_id}")

    def on_start(self):
        # Login ou initialisation
        self.client.post("/login", json={
            "username": "test",
            "password": "test"
        })
```

Déploiement de Locust :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: locust-master
spec:
  replicas: 1
  selector:
    matchLabels:
      app: locust-master
  template:
    metadata:
      labels:
        app: locust-master
    spec:
      containers:
      - name: locust
        image: locustio/locust
        ports:
        - containerPort: 8089
        - containerPort: 5557
        command: ["locust"]
        args: ["--master", "--host=http://my-app.default.svc.cluster.local"]
        volumeMounts:
        - name: locust-scripts
          mountPath: /home/locust
      volumes:
      - name: locust-scripts
        configMap:
          name: locust-scripts
---
apiVersion: v1
kind: Service
metadata:
  name: locust-master
spec:
  type: NodePort
  ports:
  - port: 8089
    targetPort: 8089
    nodePort: 30089
    name: web
  - port: 5557
    targetPort: 5557
    name: communication
  selector:
    app: locust-master
```

### Apache Bench (ab) - Simple et Rapide

Pour des tests simples et rapides :

```bash
# Test basique
ab -n 1000 -c 10 http://my-app.local/

# Test avec keepalive
ab -k -n 10000 -c 50 http://my-app.local/

# Test POST avec données
ab -p data.json -T application/json -n 1000 -c 10 http://my-app.local/api/
```

### Hey - Alternative Moderne à AB

```bash
# Installation
go install github.com/rakyll/hey@latest

# Test simple
hey -n 1000 -c 50 http://my-app.local/

# Test avec durée
hey -z 30s -c 100 http://my-app.local/

# Test avec headers custom
hey -H "Authorization: Bearer token" -n 1000 http://my-app.local/api/
```

## Scénarios de Test pour un Lab

### Test de Montée Progressive

Simule une augmentation graduelle du trafic :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: progressive-load-test
data:
  test.js: |
    import http from 'k6/http';
    import { check } from 'k6';

    export let options = {
      stages: [
        { duration: '5m', target: 50 },
        { duration: '10m', target: 100 },
        { duration: '5m', target: 150 },
        { duration: '10m', target: 200 },
        { duration: '5m', target: 0 },
      ],
    };

    export default function() {
      let res = http.get('http://app-service:8080/');
      check(res, {
        'status 200': (r) => r.status === 200,
      });
    }
```

### Test de Pic de Trafic

Teste la réaction à un pic soudain :

```javascript
// spike-test.js
export let options = {
  stages: [
    { duration: '2m', target: 10 },   // Charge normale
    { duration: '30s', target: 200 }, // Pic soudain
    { duration: '30s', target: 200 }, // Maintien du pic
    { duration: '30s', target: 10 },  // Retour à la normale
    { duration: '2m', target: 10 },   // Stabilisation
    { duration: '30s', target: 0 },
  ],
};
```

### Test de Stress Limite

Trouve le point de rupture :

```javascript
// breakpoint-test.js
export let options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '2m', target: 200 },
    { duration: '2m', target: 300 },
    { duration: '2m', target: 400 },
    { duration: '2m', target: 500 },
    // Continue jusqu'à la rupture
  ],
  thresholds: {
    http_req_failed: [{
      threshold: 'rate<0.01',
      abortOnFail: true,
    }],
  },
};
```

## Configuration de l'Environnement de Test

### Application de Test

Déployez une application simple pour les tests :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: app
        image: nginxdemos/hello
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: test-app
spec:
  selector:
    app: test-app
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: test-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: test-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### Générateur de Charge dans le Cluster

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-generator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: load-generator
  template:
    metadata:
      labels:
        app: load-generator
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh"]
        args:
        - -c
        - |
          while true; do
            for i in $(seq 1 100); do
              wget -q -O- http://test-app/
            done
            sleep 1
          done
```

### Monitoring pendant les Tests

```yaml
# Configuration Prometheus pour les tests
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-metrics-queries
data:
  queries.yaml: |
    # Requêtes par seconde
    rate(nginx_ingress_controller_requests[1m])

    # Latence P95
    histogram_quantile(0.95,
      rate(nginx_ingress_controller_request_duration_seconds_bucket[1m]))

    # Taux d'erreur
    rate(nginx_ingress_controller_requests{status=~"5.."}[1m])

    # Utilisation CPU des pods
    rate(container_cpu_usage_seconds_total[1m])

    # Nombre de pods actifs
    kube_deployment_status_replicas
```

## Automatisation des Tests

### Pipeline de Test CI/CD

```yaml
# gitlab-ci.yml
stages:
  - deploy
  - test
  - analyze

deploy-test-env:
  stage: deploy
  script:
    - kubectl apply -f test-app.yaml
    - kubectl wait --for=condition=available deployment/test-app

load-test:
  stage: test
  image: grafana/k6:latest
  script:
    - k6 run --out json=results.json test.js
  artifacts:
    paths:
      - results.json

analyze-results:
  stage: analyze
  script:
    - python analyze_results.py results.json
    - |
      if [ "$ERROR_RATE" -gt "5" ]; then
        echo "Test failed: Error rate too high"
        exit 1
      fi
```

### Script de Test Automatisé

```bash
#!/bin/bash
# automated-load-test.sh

set -e

NAMESPACE="load-test"
APP_NAME="test-app"
DURATION="10m"

echo "=== Starting Load Test ==="
echo "Namespace: $NAMESPACE"
echo "Target: $APP_NAME"
echo "Duration: $DURATION"

# 1. Préparer l'environnement
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

# 2. Déployer l'application
kubectl apply -n $NAMESPACE -f test-app.yaml

# 3. Attendre que l'app soit prête
kubectl wait --for=condition=available \
  -n $NAMESPACE deployment/$APP_NAME \
  --timeout=300s

# 4. Obtenir l'état initial
echo "=== Initial State ==="
kubectl top pods -n $NAMESPACE
kubectl get hpa -n $NAMESPACE

# 5. Démarrer le monitoring
kubectl top pods -n $NAMESPACE --watch &
MONITOR_PID=$!

# 6. Lancer le test de charge
k6 run \
  --duration $DURATION \
  --vus 50 \
  --out json=results.json \
  test-script.js

# 7. Arrêter le monitoring
kill $MONITOR_PID

# 8. Analyser les résultats
echo "=== Test Results ==="
jq '.metrics.http_req_duration' results.json
jq '.metrics.http_reqs' results.json

# 9. Vérifier le scaling
echo "=== Scaling Results ==="
kubectl describe hpa -n $NAMESPACE $APP_NAME

# 10. Nettoyer
read -p "Clean up test environment? (y/n) " -n 1 -r
if [[ $REPLY =~ ^[Yy]$ ]]; then
  kubectl delete namespace $NAMESPACE
fi
```

## Analyse des Résultats

### Métriques de Performance

```python
# analyze_results.py
import json
import statistics

def analyze_k6_results(filename):
    with open(filename, 'r') as f:
        data = json.load(f)

    # Extraire les métriques
    durations = data['metrics']['http_req_duration']['values']
    errors = data['metrics']['http_req_failed']['values']

    # Calculer les statistiques
    p50 = statistics.median(durations)
    p95 = statistics.quantiles(durations, n=20)[18]
    p99 = statistics.quantiles(durations, n=100)[98]
    error_rate = sum(errors) / len(errors) * 100

    print(f"=== Performance Report ===")
    print(f"P50 Latency: {p50:.2f}ms")
    print(f"P95 Latency: {p95:.2f}ms")
    print(f"P99 Latency: {p99:.2f}ms")
    print(f"Error Rate: {error_rate:.2f}%")

    # Vérifier les SLOs
    if p95 > 500:
        print("⚠️ WARNING: P95 latency exceeds 500ms SLO")
    if error_rate > 1:
        print("⚠️ WARNING: Error rate exceeds 1% SLO")

    return {
        'p50': p50,
        'p95': p95,
        'p99': p99,
        'error_rate': error_rate
    }
```

### Dashboard Grafana pour Tests

```json
{
  "dashboard": {
    "title": "Load Test Monitoring",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [{
          "expr": "rate(http_requests_total[1m])"
        }]
      },
      {
        "title": "Response Time Percentiles",
        "targets": [
          {"expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[1m]))"},
          {"expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[1m]))"},
          {"expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[1m]))"}
        ]
      },
      {
        "title": "Error Rate",
        "targets": [{
          "expr": "rate(http_requests_total{status=~\"5..\"}[1m])"
        }]
      },
      {
        "title": "Pod Count",
        "targets": [{
          "expr": "kube_deployment_status_replicas_available"
        }]
      },
      {
        "title": "CPU Usage",
        "targets": [{
          "expr": "rate(container_cpu_usage_seconds_total[1m])"
        }]
      },
      {
        "title": "Memory Usage",
        "targets": [{
          "expr": "container_memory_working_set_bytes"
        }]
      }
    ]
  }
}
```

### Rapport de Test

```markdown
# Template de Rapport de Test de Charge

## Informations du Test
- **Date**: 2024-01-15
- **Durée**: 30 minutes
- **Charge Max**: 200 utilisateurs virtuels
- **Environnement**: MicroK8s Lab

## Résultats Clés
| Métrique | Valeur | SLO | Status |
|----------|--------|-----|--------|
| P50 Latency | 45ms | <100ms | ✅ |
| P95 Latency | 230ms | <500ms | ✅ |
| P99 Latency | 580ms | <1000ms | ✅ |
| Error Rate | 0.3% | <1% | ✅ |
| Max Pods | 12 | 20 | ✅ |

## Observations
- HPA a réagi en 30 secondes au pic de charge
- Scaling de 3 à 12 pods en 2 minutes
- Pas d'OOM kills détectés
- CPU throttling minimal

## Recommandations
1. Augmenter les CPU requests de 100m à 150m
2. Réduire le seuil HPA de 50% à 40%
3. Implémenter du cache pour réduire la latence P99
```

## Problèmes Courants et Solutions

### Pods ne Scalent pas

```bash
# Diagnostic
kubectl describe hpa test-app-hpa

# Vérifications
# 1. Metrics server fonctionne ?
kubectl top nodes

# 2. Requests CPU définies ?
kubectl get deployment test-app -o yaml | grep -A 5 resources

# 3. Métriques disponibles ?
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods
```

### Performances Dégradées sous Charge

```yaml
# Optimisations possibles
apiVersion: apps/v1
kind: Deployment
metadata:
  name: optimized-app
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            cpu: 200m      # Augmenté
            memory: 256Mi
          limits:
            cpu: 1000m     # Plus de burst
            memory: 1Gi
        # Tuning application
        env:
        - name: WORKER_PROCESSES
          value: "4"
        - name: WORKER_CONNECTIONS
          value: "1024"
        - name: KEEPALIVE_TIMEOUT
          value: "65"
```

### Erreurs de Connexion

```yaml
# Augmenter les limites de connexion
apiVersion: v1
kind: Service
metadata:
  name: test-app
  annotations:
    # Pour NGINX Ingress
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

## Bonnes Pratiques

### 1. Commencer Petit

Toujours commencer avec une charge faible et augmenter progressivement :

```javascript
// Progression recommandée
export let options = {
  stages: [
    { duration: '1m', target: 5 },    // Warmup
    { duration: '2m', target: 10 },   // Test léger
    { duration: '2m', target: 20 },   // Charge normale
    { duration: '2m', target: 50 },   // Charge élevée
    // Augmenter seulement si stable
  ],
};
```

### 2. Isoler les Tests

Créer un namespace dédié pour les tests :

```bash
# Créer environnement isolé
kubectl create namespace load-test
kubectl label namespace load-test purpose=testing

# Limiter les ressources
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: load-test-quota
  namespace: load-test
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    pods: "50"
EOF
```

### 3. Monitorer Activement

Surveiller en temps réel pendant les tests :

```bash
# Terminal 1: Pods
watch kubectl top pods -n load-test

# Terminal 2: Nodes
watch kubectl top nodes

# Terminal 3: HPA
watch kubectl get hpa -n load-test

# Terminal 4: Events
kubectl get events -n load-test --watch
```

### 4. Sauvegarder les Résultats

```bash
# Script de sauvegarde des résultats
#!/bin/bash
TEST_ID=$(date +%Y%m%d-%H%M%S)
RESULTS_DIR="test-results/$TEST_ID"

mkdir -p $RESULTS_DIR

# Sauvegarder les métriques
k6 run --out json=$RESULTS_DIR/k6-results.json test.js

# Sauvegarder l'état du cluster
kubectl get all -n load-test > $RESULTS_DIR/cluster-state.txt
kubectl top pods -n load-test > $RESULTS_DIR/pod-metrics.txt
kubectl describe hpa -n load-test > $RESULTS_DIR/hpa-status.txt

# Créer un rapport
echo "Test ID: $TEST_ID" > $RESULTS_DIR/report.md
echo "Date: $(date)" >> $RESULTS_DIR/report.md
# ... ajouter plus d'infos
```

### 5. Tests Reproductibles

Utiliser des configurations versionnées :

```yaml
# test-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: load-test-config
data:
  test-params.json: |
    {
      "duration": "10m",
      "vus": 50,
      "rampup": "2m",
      "target_rps": 1000,
      "thresholds": {
        "p95": 500,
        "error_rate": 0.01
      }
    }
```

## Conclusion

Les tests de charge dans un lab MicroK8s sont essentiels pour comprendre les limites de vos applications et valider vos stratégies de scaling. Même avec des ressources limitées, vous pouvez simuler des conditions réalistes et identifier les problèmes avant qu'ils n'affectent la production.

Les points clés à retenir :
- Commencez avec des tests simples et augmentez progressivement la complexité
- Utilisez différents types de tests selon vos objectifs
- Surveillez à la fois les métriques applicatives et système
- Automatisez vos tests pour garantir la reproductibilité
- Documentez et analysez systématiquement les résultats
- Utilisez les tests pour valider vos configurations HPA, VPA et limites de ressources

Les compétences acquises en testant dans votre lab vous prépareront à gérer efficacement la performance et la scalabilité en production, où les enjeux sont plus importants mais les principes restent les mêmes.

⏭️

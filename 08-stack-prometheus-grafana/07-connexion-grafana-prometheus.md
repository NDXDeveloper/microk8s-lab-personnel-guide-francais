🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.07 - Connexion Grafana-Prometheus

## Comprendre la connexion Grafana-Prometheus

### Architecture de la connexion

La connexion entre Grafana et Prometheus fonctionne sur un modèle client-serveur où Grafana agit comme client qui interroge Prometheus via son API HTTP.

```
┌──────────────────────────────────────────────────────────┐
│                     Flux de données                      │
│                                                          │
│  [Utilisateur] ──1.Requête──▶ [Grafana]                  │
│                                    │                     │
│                              2.PromQL Query              │
│                                    ▼                     │
│                              [Prometheus]                │
│                                    │                     │
│                              3.Résultats JSON            │
│                                    ▼                     │
│  [Utilisateur] ◀──4.Graphique── [Grafana]                │
│                                                          │
└──────────────────────────────────────────────────────────┘

Détail des étapes :
1. L'utilisateur ouvre un dashboard ou crée un panel
2. Grafana traduit la configuration en requête PromQL
3. Prometheus exécute la requête et retourne les données
4. Grafana transforme les données en visualisation
```

### Types de connexion

| Type | Description | Utilisation | Sécurité |
|------|-------------|-------------|----------|
| **Server (Proxy)** | Grafana fait proxy vers Prometheus | Recommandé | ✅ Sécurisé |
| **Browser (Direct)** | Le navigateur contacte directement Prometheus | Tests locaux | ⚠️ Expose Prometheus |

### Protocole de communication

```yaml
# Exemple de requête HTTP de Grafana vers Prometheus
POST /api/v1/query_range
Content-Type: application/x-www-form-urlencoded

query=rate(http_requests_total[5m])&
start=2024-01-01T00:00:00Z&
end=2024-01-01T01:00:00Z&
step=15s

# Réponse Prometheus
{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [...]
  }
}
```

## Configuration de la datasource Prometheus

### Méthode 1 : Via l'interface web Grafana

#### Étape 1 : Accéder aux datasources

1. Se connecter à Grafana
2. Cliquer sur l'icône ⚙️ (Configuration) dans le menu latéral
3. Sélectionner "Data sources"
4. Cliquer sur "Add data source"
5. Rechercher et sélectionner "Prometheus"

#### Étape 2 : Configuration de base

```yaml
# Paramètres essentiels
Name: Prometheus              # Nom de la datasource
Default: ✓                    # Datasource par défaut

# HTTP
URL: http://prometheus:9090   # URL de Prometheus
Access: Server (Default)      # Type d'accès

# Auth
Aucune authentification      # Si Prometheus n'est pas sécurisé
```

#### Configuration détaillée dans l'interface

**Section HTTP :**
```
URL: http://prometheus:9090
  # Pour un service dans le même namespace :
  # http://prometheus:9090

  # Pour un service dans un autre namespace :
  # http://prometheus.monitoring.svc.cluster.local:9090

  # Pour un Prometheus externe :
  # https://prometheus.example.com

Access: Server (Default)
  # Server : Grafana backend fait les requêtes (recommandé)
  # Browser : Le navigateur fait les requêtes directement

Timeout: 30                   # Timeout en secondes
```

**Section Auth (si nécessaire) :**
```
# Basic Auth
User: prometheus_user
Password: ********

# TLS Client Auth
Client cert: [Télécharger le certificat]
Client key: [Télécharger la clé]

# With CA Cert
CA cert: [Télécharger le CA]

# Skip TLS Verify (pour certificats auto-signés)
☑ Skip TLS certificate validation
```

**Section Misc :**
```
Scrape interval: 15s          # Doit correspondre à Prometheus
Query timeout: 60s            # Timeout pour les requêtes
HTTP Method: POST             # Recommandé pour requêtes longues
```

#### Étape 3 : Test de connexion

Cliquer sur **"Save & Test"** pour vérifier la connexion.

Messages possibles :
- ✅ **"Data source is working"** : Connexion réussie
- ❌ **"HTTP Error Bad Gateway"** : Prometheus inaccessible
- ❌ **"Network Error"** : Problème réseau/DNS
- ❌ **"401 Unauthorized"** : Authentification requise

### Méthode 2 : Configuration automatique via YAML

#### Provisioning avec ConfigMap

```yaml
# grafana-datasource-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  prometheus.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      access: proxy
      orgId: 1
      url: http://prometheus:9090
      basicAuth: false
      isDefault: true
      jsonData:
        httpMethod: POST
        keepCookies: []
        timeInterval: 15s
        queryTimeout: 60s
        incrementalQuerying: true
        incrementalQueryOverlapWindow: 10m
      editable: true
      version: 1
```

#### Montage dans Grafana

```yaml
# Dans le deployment Grafana
spec:
  containers:
  - name: grafana
    volumeMounts:
    - name: datasources
      mountPath: /etc/grafana/provisioning/datasources
  volumes:
  - name: datasources
    configMap:
      name: grafana-datasources
```

### Méthode 3 : Configuration via API REST

```bash
# Créer une datasource via API
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  http://grafana:3000/api/datasources \
  -d '{
    "name": "Prometheus",
    "type": "prometheus",
    "url": "http://prometheus:9090",
    "access": "proxy",
    "isDefault": true,
    "jsonData": {
      "httpMethod": "POST",
      "timeInterval": "15s",
      "queryTimeout": "60s"
    }
  }'

# Vérifier les datasources existantes
curl -H "Authorization: Bearer YOUR_API_KEY" \
  http://grafana:3000/api/datasources

# Tester une datasource
curl -X POST \
  -H "Authorization: Bearer YOUR_API_KEY" \
  http://grafana:3000/api/datasources/1/health
```

## Résolution DNS dans Kubernetes

### Formats d'URL selon la configuration

```yaml
# Service dans le même namespace
http://prometheus:9090

# Service dans un autre namespace
http://prometheus.monitoring:9090
http://prometheus.monitoring.svc:9090
http://prometheus.monitoring.svc.cluster.local:9090

# Service avec port nommé
http://prometheus:http-metrics

# Service headless (StatefulSet)
http://prometheus-0.prometheus:9090
```

### Vérification de la résolution DNS

```bash
# Depuis le pod Grafana
kubectl exec -n monitoring grafana-xxx -- nslookup prometheus
kubectl exec -n monitoring grafana-xxx -- nc -zv prometheus 9090

# Test de connectivité HTTP
kubectl exec -n monitoring grafana-xxx -- wget -O- http://prometheus:9090/api/v1/query?query=up
```

## Configuration avancée

### Paramètres de performance

```yaml
jsonData:
  # Requêtes incrémentales (pour dashboards avec beaucoup de panels)
  incrementalQuerying: true
  incrementalQueryOverlapWindow: 10m

  # Cache
  cacheLevel: 'High'
  cacheDurationSeconds: 300

  # Limites
  maxDataPoints: 1000      # Points max par série
  intervalFactor: 2        # Facteur d'intervalle

  # Métriques custom
  customQueryParameters: "timeout=60s&max_samples=1000000"
```

### Configuration pour haute disponibilité

```yaml
# Pour Prometheus avec plusieurs replicas
datasources:
- name: Prometheus-Primary
  url: http://prometheus-0:9090
  jsonData:
    httpMethod: POST

- name: Prometheus-Secondary
  url: http://prometheus-1:9090
  jsonData:
    httpMethod: POST

# Avec load balancer
- name: Prometheus-HA
  url: http://prometheus-lb:9090
  jsonData:
    httpMethod: POST
    customHttpHeaders:
      X-Prometheus-Replica: "any"
```

### Authentification avancée

#### Basic Auth
```yaml
datasources:
- name: Prometheus-Secure
  url: https://prometheus:9090
  basicAuth: true
  basicAuthUser: admin
  secureJsonData:
    basicAuthPassword: "password123"
```

#### Bearer Token
```yaml
datasources:
- name: Prometheus-Token
  url: https://prometheus:9090
  jsonData:
    httpHeaderName1: "Authorization"
  secureJsonData:
    httpHeaderValue1: "Bearer YOUR_TOKEN_HERE"
```

#### TLS/mTLS
```yaml
datasources:
- name: Prometheus-TLS
  url: https://prometheus:9090
  jsonData:
    tlsAuth: true
    tlsAuthWithCACert: true
    serverName: prometheus.example.com
  secureJsonData:
    tlsCACert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    tlsClientCert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    tlsClientKey: |
      -----BEGIN PRIVATE KEY-----
      ...
      -----END PRIVATE KEY-----
```

## Test et validation de la connexion

### Tests depuis l'interface Grafana

1. **Test basique dans les settings :**
   - Configuration → Data Sources → Prometheus
   - Cliquer sur "Save & Test"

2. **Test avec Explore :**
   - Aller dans Explore (icône compass)
   - Sélectionner Prometheus
   - Taper `up` et exécuter
   - Vérifier que des résultats apparaissent

3. **Création d'un panel de test :**
```promql
# Requête simple pour tester
up{job="prometheus"}

# Devrait retourner 1 si Prometheus est accessible
```

### Tests via ligne de commande

```bash
# Test direct de l'API Prometheus
curl http://prometheus:9090/api/v1/query?query=up

# Test via Grafana API
curl -H "Authorization: Bearer API_KEY" \
  http://grafana:3000/api/datasources/proxy/1/api/v1/query?query=up

# Test de santé de la datasource
curl -X GET -H "Authorization: Bearer API_KEY" \
  http://grafana:3000/api/datasources/1/health
```

### Métriques de diagnostic

```promql
# Vérifier que Prometheus est up
up{job="prometheus"}

# Nombre de séries temporelles
prometheus_tsdb_symbol_table_size_bytes

# Temps de scrape
scrape_duration_seconds

# Métriques Grafana sur l'utilisation de la datasource
grafana_datasource_request_total{datasource="prometheus"}
grafana_datasource_request_duration_seconds{datasource="prometheus"}
```

## Troubleshooting des problèmes de connexion

### Problème : "Bad Gateway" ou "502 Error"

**Causes possibles :**
- Prometheus est down
- Mauvaise URL
- Problème réseau

**Solutions :**
```bash
# Vérifier que Prometheus est running
kubectl get pods -n monitoring | grep prometheus

# Vérifier les services
kubectl get svc -n monitoring

# Test de connectivité depuis Grafana
kubectl exec -n monitoring grafana-xxx -- curl http://prometheus:9090/api/v1/query?query=up
```

### Problème : "Network Error" ou timeout

**Causes possibles :**
- DNS ne résout pas
- Firewall/NetworkPolicy bloque
- Timeout trop court

**Solutions :**
```bash
# Vérifier la résolution DNS
kubectl exec -n monitoring grafana-xxx -- nslookup prometheus

# Vérifier les NetworkPolicies
kubectl get networkpolicies -n monitoring

# Augmenter le timeout dans la config
# Query timeout: 60s → 120s
```

### Problème : "401 Unauthorized"

**Causes possibles :**
- Prometheus nécessite une authentification
- Token/credentials incorrects

**Solutions :**
```yaml
# Ajouter l'authentification dans Grafana
basicAuth: true
basicAuthUser: prometheus_user
secureJsonData:
  basicAuthPassword: "correct_password"
```

### Problème : Pas de données dans les dashboards

**Causes possibles :**
- Mauvaise sélection de datasource
- Time range incorrect
- Métriques n'existent pas
- Requête PromQL incorrecte

**Solutions :**
```bash
# Vérifier que les métriques existent dans Prometheus
curl http://prometheus:9090/api/v1/label/__name__/values

# Tester la requête directement dans Prometheus
curl -G http://prometheus:9090/api/v1/query --data-urlencode 'query=up'

# Vérifier le time range dans Grafana
# Ajuster à "Last 5 minutes" pour données récentes

# Dans Grafana Explore, tester une requête simple
up{job="prometheus"}
```

## Optimisation de la connexion

### Configuration pour grandes requêtes

```yaml
# Pour dashboards avec beaucoup de panels
jsonData:
  # Activer les requêtes incrémentales
  incrementalQuerying: true
  incrementalQueryOverlapWindow: 10m

  # Augmenter les limites
  maxDataPoints: 5000
  intervalFactor: 2

  # Timeout plus long
  queryTimeout: 300s

  # Cache agressif
  cacheLevel: 'High'
  cacheDurationSeconds: 600
```

### Réduction de la charge sur Prometheus

```yaml
# Utiliser le bon intervalle
jsonData:
  timeInterval: 30s  # Minimum interval entre points

  # Activer le downsampling
  interval: "30s"

  # Limiter les requêtes parallèles
  maxConcurrentQueries: 10
```

### Configuration pour lab avec ressources limitées

```yaml
datasources:
- name: Prometheus-Lab
  type: prometheus
  url: http://prometheus:9090
  jsonData:
    httpMethod: POST
    timeInterval: 30s        # Intervalle plus long
    queryTimeout: 30s         # Timeout court
    maxDataPoints: 500        # Moins de points
    incrementalQuerying: false # Désactiver pour économiser
    cacheLevel: 'Low'         # Cache minimal
```

## Monitoring de la connexion

### Métriques Grafana sur Prometheus

```promql
# Requêtes vers Prometheus
grafana_datasource_request_total{datasource="prometheus"}

# Durée des requêtes
grafana_datasource_request_duration_seconds{datasource="prometheus"}

# Erreurs de datasource
grafana_datasource_response_total{datasource="prometheus",code!="200"}

# Cache hits
grafana_datasource_cache_hits_total{datasource="prometheus"}

# Requêtes en cours
grafana_datasource_request_in_flight{datasource="prometheus"}
```

### Dashboard de monitoring de la connexion

```json
{
  "dashboard": {
    "title": "Grafana-Prometheus Connection Health",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(grafana_datasource_request_total{datasource=\"prometheus\"}[5m])",
            "legendFormat": "Requests/sec"
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 }
      },
      {
        "id": 2,
        "title": "Request Duration (95th percentile)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(grafana_datasource_request_duration_seconds_bucket{datasource=\"prometheus\"}[5m]))",
            "legendFormat": "p95 latency"
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 }
      },
      {
        "id": 3,
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(grafana_datasource_response_total{datasource=\"prometheus\",code!=\"200\"}[5m])",
            "legendFormat": "Errors/sec"
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 8 }
      },
      {
        "id": 4,
        "title": "Cache Hit Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(grafana_datasource_cache_hits_total{datasource=\"prometheus\"}[5m]) / rate(grafana_datasource_request_total{datasource=\"prometheus\"}[5m]) * 100",
            "legendFormat": "Cache Hit %"
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 8 }
      }
    ]
  }
}
```

### Alertes sur la connexion

```yaml
# alerts-connection.yaml
groups:
- name: grafana_prometheus_connection
  interval: 30s
  rules:
  - alert: PrometheusDataSourceDown
    expr: up{job="prometheus"} == 0
    for: 5m
    labels:
      severity: critical
      component: grafana
    annotations:
      summary: "Prometheus datasource is down"
      description: "Grafana cannot reach Prometheus for more than 5 minutes"

  - alert: HighDataSourceErrorRate
    expr: |
      rate(grafana_datasource_response_total{datasource="prometheus",code!="200"}[5m]) > 0.1
    for: 10m
    labels:
      severity: warning
      component: grafana
    annotations:
      summary: "High error rate on Prometheus datasource"
      description: "Error rate is {{ $value | humanizePercentage }} in the last 5 minutes"

  - alert: SlowPrometheusQueries
    expr: |
      histogram_quantile(0.95, rate(grafana_datasource_request_duration_seconds_bucket{datasource="prometheus"}[5m])) > 10
    for: 15m
    labels:
      severity: warning
      component: performance
    annotations:
      summary: "Slow Prometheus queries detected"
      description: "95th percentile query duration is {{ $value }}s"
```

## Configuration multi-environnements

### Plusieurs instances Prometheus

```yaml
# datasources-multi.yaml
apiVersion: 1
datasources:
# Développement
- name: Prometheus-Dev
  type: prometheus
  url: http://prometheus-dev:9090
  access: proxy
  orgId: 1
  jsonData:
    httpMethod: POST
    timeInterval: 30s

# Staging
- name: Prometheus-Staging
  type: prometheus
  url: http://prometheus-staging:9090
  access: proxy
  orgId: 1
  basicAuth: true
  basicAuthUser: staging_user
  secureJsonData:
    basicAuthPassword: "staging_password"

# Production
- name: Prometheus-Prod
  type: prometheus
  url: https://prometheus-prod:9090
  access: proxy
  orgId: 1
  basicAuth: true
  basicAuthUser: prod_user
  secureJsonData:
    basicAuthPassword: "prod_password"
  jsonData:
    tlsSkipVerify: false
    httpMethod: POST
    timeInterval: 15s
```

### Variables de dashboard pour switch entre environnements

```json
{
  "templating": {
    "list": [
      {
        "name": "datasource",
        "label": "Environment",
        "type": "datasource",
        "query": "prometheus",
        "current": {
          "text": "Prometheus-Dev",
          "value": "Prometheus-Dev"
        },
        "regex": "Prometheus.*",
        "options": [],
        "refresh": 1,
        "includeAll": false,
        "multi": false
      },
      {
        "name": "namespace",
        "label": "Namespace",
        "type": "query",
        "datasource": "$datasource",
        "query": "label_values(kube_pod_info, namespace)",
        "refresh": 2,
        "sort": 1,
        "includeAll": true,
        "multi": true
      }
    ]
  }
}
```

### Fédération Prometheus

```yaml
# Pour requêtes cross-cluster avec Thanos
- name: Prometheus-Federation
  type: prometheus
  url: http://thanos-query:9090
  jsonData:
    httpMethod: POST
    customQueryParameters: "partial_response=true&dedup=true"
    timeInterval: 30s

# Pour Cortex
- name: Prometheus-Cortex
  type: prometheus
  url: http://cortex-query-frontend:9090
  jsonData:
    httpMethod: POST
```

## Scripts d'automatisation

### Script de configuration automatique

```bash
#!/bin/bash
# setup-prometheus-datasource.sh

set -e

# Configuration
GRAFANA_URL="${GRAFANA_URL:-http://localhost:3000}"
GRAFANA_USER="${GRAFANA_USER:-admin}"
GRAFANA_PASSWORD="${GRAFANA_PASSWORD:-admin}"
PROMETHEUS_URL="${PROMETHEUS_URL:-http://prometheus:9090}"

echo "Setting up Prometheus datasource..."

# Créer la datasource
RESPONSE=$(curl -s -X POST \
  -H "Content-Type: application/json" \
  -u ${GRAFANA_USER}:${GRAFANA_PASSWORD} \
  ${GRAFANA_URL}/api/datasources \
  -d @- <<EOF
{
  "name": "Prometheus",
  "type": "prometheus",
  "url": "${PROMETHEUS_URL}",
  "access": "proxy",
  "isDefault": true,
  "jsonData": {
    "httpMethod": "POST",
    "timeInterval": "15s",
    "queryTimeout": "60s",
    "incrementalQuerying": true
  }
}
EOF
)

# Vérifier le résultat
if echo "$RESPONSE" | grep -q "Datasource added"; then
    echo "✅ Datasource created successfully"
else
    echo "❌ Failed to create datasource: $RESPONSE"
    exit 1
fi

# Tester la connexion
echo "Testing connection..."
TEST_RESPONSE=$(curl -s -X POST \
  -u ${GRAFANA_USER}:${GRAFANA_PASSWORD} \
  ${GRAFANA_URL}/api/datasources/name/Prometheus/health)

if echo "$TEST_RESPONSE" | grep -q "OK"; then
    echo "✅ Connection test successful"
else
    echo "❌ Connection test failed: $TEST_RESPONSE"
    exit 1
fi

echo "✅ Datasource configured and tested successfully"
```

### Script de validation complète

```bash
#!/bin/bash
# validate-connection.sh

set -e

PROMETHEUS_URL="${PROMETHEUS_URL:-http://prometheus:9090}"
GRAFANA_URL="${GRAFANA_URL:-http://localhost:3000}"
GRAFANA_USER="${GRAFANA_USER:-admin}"
GRAFANA_PASSWORD="${GRAFANA_PASSWORD:-admin}"

echo "=== Grafana-Prometheus Connection Validation ==="

# Test 1: Prometheus direct
echo -n "1. Testing Prometheus directly... "
if curl -s ${PROMETHEUS_URL}/api/v1/query?query=up > /dev/null 2>&1; then
    echo "✅ OK"
else
    echo "❌ FAILED - Cannot reach Prometheus"
    exit 1
fi

# Test 2: DNS resolution
echo -n "2. Testing DNS resolution... "
if nslookup prometheus > /dev/null 2>&1; then
    echo "✅ OK"
else
    echo "⚠️  WARNING - DNS resolution failed"
fi

# Test 3: Grafana API
echo -n "3. Testing Grafana API... "
if curl -s -u ${GRAFANA_USER}:${GRAFANA_PASSWORD} ${GRAFANA_URL}/api/health | grep -q "ok"; then
    echo "✅ OK"
else
    echo "❌ FAILED - Grafana API not accessible"
    exit 1
fi

# Test 4: Datasource exists
echo -n "4. Checking datasource exists... "
DS_COUNT=$(curl -s -u ${GRAFANA_USER}:${GRAFANA_PASSWORD} \
    ${GRAFANA_URL}/api/datasources | jq '[.[] | select(.type=="prometheus")] | length')
if [ "$DS_COUNT" -gt 0 ]; then
    echo "✅ OK - Found $DS_COUNT Prometheus datasource(s)"
else
    echo "❌ FAILED - No Prometheus datasource found"
    exit 1
fi

# Test 5: Query via Grafana
echo -n "5. Testing query via Grafana... "
DS_ID=$(curl -s -u ${GRAFANA_USER}:${GRAFANA_PASSWORD} \
    ${GRAFANA_URL}/api/datasources | jq -r '.[] | select(.type=="prometheus") | .id' | head -1)

RESULT=$(curl -s -X POST \
  -u ${GRAFANA_USER}:${GRAFANA_PASSWORD} \
  ${GRAFANA_URL}/api/datasources/proxy/${DS_ID}/api/v1/query \
  -d 'query=up')

if echo "$RESULT" | grep -q "success"; then
    echo "✅ OK"
else
    echo "❌ FAILED - Query failed"
    exit 1
fi

echo ""
echo "=== All tests passed! ==="
echo "✅ Grafana-Prometheus connection is working correctly"
```

## Bonnes pratiques

### 1. Naming conventions

```yaml
# Utiliser des noms descriptifs et cohérents
datasources:
- name: Prometheus-Prod-EU     # ✅ Clair et précis
- name: prom1                   # ❌ Ambigu

# Pattern recommandé : [Type]-[Env]-[Region]
- name: Prometheus-Dev-Local
- name: Prometheus-Staging-AWS
- name: Prometheus-Prod-EU
- name: Prometheus-Prod-US
```

### 2. Sécurité

```yaml
# Toujours utiliser HTTPS en production
- name: Prometheus-Prod
  url: https://prometheus:9090  # ✅
  jsonData:
    tlsSkipVerify: false       # ✅ Vérifier les certificats

# Ne jamais exposer Prometheus directement
  access: proxy                 # ✅ Toujours proxy, jamais browser

# Utiliser l'authentification
  basicAuth: true
  secureJsonData:
    basicAuthPassword: "${PROMETHEUS_PASSWORD}"  # Variable d'environnement

# Rotation des credentials
# Utiliser Kubernetes secrets et rotation automatique
```

### 3. Performance

```yaml
# Adapter les paramètres selon l'usage
datasources:
# Pour dashboards temps réel
- name: Prometheus-Realtime
  jsonData:
    timeInterval: 5s
    cacheLevel: 'None'
    maxDataPoints: 1000

# Pour analyses historiques
- name: Prometheus-Historical
  jsonData:
    timeInterval: 60s
    cacheLevel: 'High'
    cacheDurationSeconds: 300
    maxDataPoints: 500
```

### 4. Maintenance

```yaml
# Configuration maintenable
datasources:
- name: Prometheus
  # Documentation inline
  description: |
    Production Prometheus - EU Region
    Contact: ops-team@company.com
    SLA: 99.9%

  # Garder éditable pour debug
  editable: true

  # Versionner
  version: 2

  # Tags pour organisation
  jsonData:
    tags: ["production", "eu", "critical"]
```

## Cas d'usage avancés

### Connexion via SSH tunnel

```bash
# Créer un tunnel SSH vers Prometheus distant
ssh -L 9090:prometheus:9090 -N -f user@bastion-host

# Configuration Grafana
datasources:
- name: Prometheus-Remote
  url: http://localhost:9090  # Via tunnel local
```

### Prometheus derrière Ingress avec auth

```yaml
datasources:
- name: Prometheus-Ingress
  type: prometheus
  url: https://prometheus.example.com
  jsonData:
    httpHeaderName1: "Authorization"
    httpHeaderName2: "X-Scope-OrgID"
    httpHeaderName3: "X-Custom-Header"
  secureJsonData:
    httpHeaderValue1: "Bearer ${PROMETHEUS_TOKEN}"
    httpHeaderValue2: "tenant-1"
    httpHeaderValue3: "custom-value"
```

### Load balancing multiple Prometheus

```yaml
# Avec HAProxy ou nginx en front
datasources:
- name: Prometheus-LB
  url: http://prometheus-lb:9090
  jsonData:
    httpMethod: POST
    customHttpHeaders:
      X-Prometheus-Replica: "any"
      X-Preferred-Instance: "primary"
```

### Recording rules pour optimisation

```yaml
# Dans Prometheus, créer des recording rules
groups:
- name: grafana_optimized
  interval: 30s
  rules:
  # Pre-calculer les métriques lourdes
  - record: instance:node_cpu:rate5m
    expr: 100 * (1 - avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])))

  - record: namespace:pod_memory:sum
    expr: sum by(namespace)(container_memory_working_set_bytes)

# Dans Grafana, utiliser les métriques pré-calculées
# Au lieu de : 100 * (1 - avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])))
# Utiliser : instance:node_cpu:rate5m
```

## Checklist de validation

- [ ] **Datasource créée** : Prometheus apparaît dans la liste des datasources
- [ ] **Test réussi** : "Save & Test" retourne "Data source is working"
- [ ] **URL correcte** : Format adapté (namespace, service, port)
- [ ] **Access mode** : Server/Proxy configuré (pas Browser)
- [ ] **Authentification** : Configurée si Prometheus est sécurisé
- [ ] **Timeouts** : Adaptés à l'environnement (30s minimum)
- [ ] **Scrape interval** : Aligné avec Prometheus (généralement 15s ou 30s)
- [ ] **Requête simple** : `up` fonctionne dans Explore
- [ ] **Dashboard test** : Au moins un panel affiche des données
- [ ] **Monitoring** : Métriques de connexion disponibles
- [ ] **Documentation** : Configuration documentée
- [ ] **Backup** : Export de la configuration sauvegardé

## Migration et évolution

### Export/Import de datasources

```bash
# Export datasource existante
curl -s -u admin:admin \
  http://old-grafana:3000/api/datasources/name/Prometheus \
  > prometheus-datasource.json

# Import dans nouveau Grafana
curl -X POST \
  -H "Content-Type: application/json" \
  -u admin:admin \
  http://new-grafana:3000/api/datasources \
  -d @prometheus-datasource.json
```

### Passage à Prometheus HA (Thanos)

```yaml
# Migration vers Thanos pour stockage long terme
datasources:
- name: Prometheus-Thanos
  type: prometheus
  url: http://thanos-query:9090
  jsonData:
    httpMethod: POST
    customQueryParameters: "partial_response=true&dedup=true&max_source_resolution=5m"
    timeInterval: 30s
```

### Migration vers Mimir/Cortex

```yaml
# Pour scalabilité horizontale
datasources:
- name: Prometheus-Mimir
  type: prometheus
  url: http://mimir-query-frontend:9090
  jsonData:
    httpMethod: POST
    httpHeaderName1: "X-Scope-OrgID"
  secureJsonData:
    httpHeaderValue1: "tenant-1"
```

## Préparation pour la suite

Avec la connexion Grafana-Prometheus établie et optimisée, vous êtes maintenant prêt à :

1. **Section 8.08** : Importer des dashboards Kubernetes pré-configurés
2. **Section 8.09** : Créer vos propres dashboards personnalisés
3. **Section 8.10** : Maîtriser les variables et le templating avancé

La connexion datasource est le pont critique entre vos métriques et vos visualisations. Une configuration solide garantit des dashboards performants et fiables. Prenez le temps de bien tester et optimiser cette connexion avant de créer des dashboards complexes.

⏭️

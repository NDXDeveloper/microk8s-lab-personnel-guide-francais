🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.1 Métriques custom dans vos applications

## Introduction aux métriques personnalisées

Les métriques par défaut de Kubernetes (CPU, mémoire, réseau) vous donnent une vue de l'infrastructure, mais elles ne racontent qu'une partie de l'histoire. Pour vraiment comprendre si votre application fonctionne correctement, vous avez besoin de métriques spécifiques à votre logique métier. Ces métriques custom sont la fenêtre sur ce qui compte vraiment pour votre application.

Imaginez que vous développez une boutique en ligne dans votre lab MicroK8s. Les métriques système vous diront si les pods consomment trop de CPU, mais elles ne vous diront pas combien de paniers sont abandonnés, quel est le temps moyen de traitement d'une commande, ou combien d'erreurs de paiement se produisent. Ce sont ces métriques métier qui transforment des données techniques en insights actionnables.

## Comprendre le format Prometheus

Avant de créer nos propres métriques, il est essentiel de comprendre comment Prometheus les consomme. Prometheus utilise un format texte simple et lisible par l'humain pour exposer les métriques.

### Structure d'une métrique Prometheus

Une métrique Prometheus suit cette structure basique :
```
nom_de_la_metrique{label1="valeur1",label2="valeur2"} valeur timestamp
```

Le nom identifie ce que vous mesurez. Les labels permettent de filtrer et agréger les données. La valeur est le nombre mesuré. Le timestamp est optionnel (Prometheus l'ajoute lors du scraping).

### Exemple concret
```
http_requests_total{method="GET",endpoint="/api/users",status="200"} 1547
```
Cette ligne indique que l'endpoint `/api/users` a reçu 1547 requêtes GET réussies.

## Les quatre types de métriques Prometheus

Prometheus définit quatre types de métriques, chacun adapté à des cas d'usage spécifiques. Comprendre quand utiliser chaque type est crucial pour une observabilité efficace.

### Counter : le compteur qui ne descend jamais

Un Counter est une métrique cumulative qui ne peut qu'augmenter ou être remise à zéro. C'est parfait pour compter des événements : nombre de requêtes, erreurs, tâches complétées.

**Cas d'usage typiques :**
- Nombre total de requêtes HTTP reçues
- Nombre d'erreurs rencontrées
- Nombre de messages traités dans une queue
- Nombre de connexions à la base de données

**Convention de nommage :** Les counters se terminent généralement par `_total`.

**Exemple d'implémentation conceptuelle :**
```python
# Pseudo-code pour illustrer le concept
requests_total = Counter('http_requests_total', 'Total HTTP requests')
errors_total = Counter('application_errors_total', 'Total application errors')

# Dans votre code applicatif
def handle_request():
    requests_total.increment()
    try:
        # Traitement de la requête
        process_request()
    except Exception:
        errors_total.increment()
```

### Gauge : la jauge qui monte et descend

Un Gauge représente une valeur qui peut augmenter ou diminuer. C'est idéal pour mesurer l'état actuel de quelque chose.

**Cas d'usage typiques :**
- Nombre de connexions actives
- Température d'un capteur
- Nombre d'éléments dans une queue
- Utilisation mémoire custom
- Nombre de tâches en cours

**Exemple d'implémentation conceptuelle :**
```python
# Pseudo-code
active_connections = Gauge('websocket_connections_active', 'Active WebSocket connections')
queue_size = Gauge('task_queue_size', 'Number of tasks in queue')

# Dans votre code
def on_connection_open():
    active_connections.increment()

def on_connection_close():
    active_connections.decrement()
```

### Histogram : la distribution statistique

Un Histogram échantillonne les observations et les compte dans des buckets configurables. Il calcule automatiquement la somme et le nombre d'observations, permettant de calculer des moyennes et des percentiles.

**Cas d'usage typiques :**
- Temps de réponse des requêtes
- Taille des réponses HTTP
- Durée des opérations de base de données
- Temps d'attente dans une queue

**Structure d'un histogram :**
Un histogram génère plusieurs séries temporelles :
- `_bucket{le="x"}` : nombre d'observations ≤ x
- `_sum` : somme de toutes les observations
- `_count` : nombre total d'observations

**Exemple d'implémentation conceptuelle :**
```python
# Pseudo-code
request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    buckets=[0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

# Dans votre code
def handle_request():
    start_time = time.now()
    process_request()
    duration = time.now() - start_time
    request_duration.observe(duration)
```

### Summary : les percentiles précis

Un Summary est similaire à un Histogram mais calcule les quantiles directement côté client. Il est plus précis pour les percentiles mais plus coûteux en ressources.

**Cas d'usage typiques :**
- Quand vous avez besoin de percentiles précis
- Quand vous ne connaissez pas la distribution à l'avance
- Pour des métriques critiques nécessitant une grande précision

**Différence avec Histogram :**
- Summary : calcul exact des percentiles côté application
- Histogram : approximation des percentiles côté Prometheus

## Instrumentation de vos applications

L'instrumentation est l'art d'ajouter des métriques à votre code. Il existe plusieurs approches, de la plus manuelle à la plus automatique.

### Bibliothèques client Prometheus

Prometheus fournit des bibliothèques officielles pour les langages populaires. Chaque bibliothèque offre une API similaire pour créer et exposer des métriques.

**Langages supportés officiellement :**
- Go : la bibliothèque de référence
- Java/Scala : avec support Spring Boot
- Python : intégration Flask/Django disponible
- Ruby : compatible avec Rails
- Node.js : pour applications JavaScript

### Pattern d'exposition des métriques

Toutes les applications suivent le même pattern pour exposer leurs métriques :

1. **Créer un endpoint HTTP** (généralement `/metrics`)
2. **Enregistrer les métriques** lors de l'initialisation
3. **Mettre à jour les métriques** dans votre logique métier
4. **Exposer les métriques** au format Prometheus

### Exemple Python avec Flask

Voici un exemple complet mais simple d'une application Flask instrumentée :

```python
from flask import Flask, Response
from prometheus_client import Counter, Histogram, Gauge, generate_latest
import time
import random

app = Flask(__name__)

# Définition des métriques
request_count = Counter(
    'app_requests_total',
    'Total number of requests',
    ['method', 'endpoint', 'status']
)

request_duration = Histogram(
    'app_request_duration_seconds',
    'Request duration',
    ['method', 'endpoint']
)

active_users = Gauge(
    'app_active_users',
    'Number of active users'
)

# Decorator pour mesurer automatiquement les requêtes
def track_metrics(f):
    def wrapper(*args, **kwargs):
        start = time.time()
        status = "500"
        try:
            result = f(*args, **kwargs)
            status = "200"
            return result
        except Exception as e:
            status = "500"
            raise e
        finally:
            duration = time.time() - start
            request_count.labels(
                method='GET',
                endpoint=f.__name__,
                status=status
            ).inc()
            request_duration.labels(
                method='GET',
                endpoint=f.__name__
            ).observe(duration)

    wrapper.__name__ = f.__name__
    return wrapper

@app.route('/')
@track_metrics
def home():
    # Simuler du travail
    time.sleep(random.uniform(0.1, 0.5))
    return "Hello from instrumented app!"

@app.route('/users')
@track_metrics
def users():
    # Simuler des utilisateurs actifs
    active_users.set(random.randint(10, 100))
    time.sleep(random.uniform(0.2, 0.8))
    return "Users endpoint"

@app.route('/metrics')
def metrics():
    # Endpoint pour Prometheus
    return Response(generate_latest(), mimetype='text/plain')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### Exemple Node.js avec Express

Pour les développeurs JavaScript, voici l'équivalent en Node.js :

```javascript
const express = require('express');
const promClient = require('prom-client');

const app = express();
const register = new promClient.Registry();

// Métriques par défaut (CPU, mémoire, etc.)
promClient.collectDefaultMetrics({ register });

// Métriques custom
const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status'],
  registers: [register]
});

const httpDuration = new promClient.Histogram({
  name: 'http_request_duration_ms',
  help: 'Duration of HTTP requests in ms',
  labelNames: ['method', 'route'],
  buckets: [0.1, 5, 15, 50, 100, 500],
  registers: [register]
});

const connectedUsers = new promClient.Gauge({
  name: 'connected_users',
  help: 'Number of connected users',
  registers: [register]
});

// Middleware pour tracker toutes les requêtes
app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    httpRequestsTotal.labels(req.method, req.route?.path || req.path, res.statusCode).inc();
    httpDuration.labels(req.method, req.route?.path || req.path).observe(duration);
  });

  next();
});

// Routes application
app.get('/', (req, res) => {
  res.send('Hello from Node.js instrumented app!');
});

app.get('/users', (req, res) => {
  // Simuler des utilisateurs connectés
  connectedUsers.set(Math.floor(Math.random() * 50) + 10);
  res.json({ users: 'data' });
});

// Endpoint pour Prometheus
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

## Déploiement sur MicroK8s

Une fois votre application instrumentée, il faut la déployer sur MicroK8s et configurer Prometheus pour collecter les métriques.

### Manifeste Kubernetes pour l'application

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: instrumented-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: instrumented-app
  template:
    metadata:
      labels:
        app: instrumented-app
      annotations:
        # Annotations pour que Prometheus découvre automatiquement l'app
        prometheus.io/scrape: "true"
        prometheus.io/port: "5000"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: your-registry/instrumented-app:latest
        ports:
        - containerPort: 5000
          name: http
        - containerPort: 5000
          name: metrics
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: instrumented-app
  labels:
    app: instrumented-app
spec:
  selector:
    app: instrumented-app
  ports:
  - name: http
    port: 80
    targetPort: 5000
  - name: metrics
    port: 5000
    targetPort: 5000
```

### Configuration du ServiceMonitor

Pour que Prometheus découvre automatiquement votre application, créez un ServiceMonitor :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: instrumented-app
  namespace: default
spec:
  selector:
    matchLabels:
      app: instrumented-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

## Bonnes pratiques pour les métriques custom

### Nommage des métriques

Suivez ces conventions pour des métriques claires et cohérentes :

1. **Utilisez des underscores** pour séparer les mots : `http_requests_total`
2. **Incluez l'unité** dans le nom : `_seconds`, `_bytes`, `_total`
3. **Soyez descriptif** mais concis : `database_connection_pool_size`
4. **Groupez par préfixe** : `myapp_feature_metric`

### Labels : puissance et dangers

Les labels permettent de filtrer et agréger vos métriques, mais attention à la cardinalité !

**Bonnes pratiques pour les labels :**
- Utilisez des labels pour les dimensions importantes (method, endpoint, status)
- Évitez les valeurs uniques (user_id, session_id)
- Limitez le nombre de valeurs possibles par label
- Préférez des labels génériques (status="4xx") aux spécifiques (status="404")

**Exemple de mauvaise pratique :**
```
# ÉVITEZ : cardinalité explosive
http_requests_total{user_id="12345", session_id="abc-def-ghi", timestamp="2024-01-01-10:23:45"}

# PRÉFÉREZ : cardinalité contrôlée
http_requests_total{method="GET", endpoint="/api/users", status="2xx"}
```

### Fréquence de mise à jour

Différentes métriques nécessitent différentes fréquences de mise à jour :

- **Counters** : mettez à jour à chaque événement
- **Gauges** : mettez à jour quand la valeur change ou périodiquement
- **Histograms/Summaries** : observez chaque occurrence

### Gestion de la performance

L'instrumentation a un coût. Voici comment le minimiser :

1. **Évitez les calculs complexes** dans le hot path
2. **Utilisez l'échantillonnage** pour les métriques haute fréquence
3. **Agrégez côté application** quand possible
4. **Limitez la cardinalité** des labels

## Métriques métier vs métriques techniques

### Métriques techniques

Ces métriques concernent le fonctionnement technique de votre application :
- Latence des requêtes
- Taux d'erreur
- Utilisation du pool de connexions
- Taille des queues internes
- Performance du garbage collector

### Métriques métier

Ces métriques reflètent la valeur business de votre application :
- Nombre de commandes passées
- Valeur moyenne du panier
- Taux de conversion
- Nombre d'utilisateurs actifs
- Revenue par minute

### Exemple complet : E-commerce

Voici comment instrumenter une application e-commerce avec les deux types :

```python
# Métriques techniques
request_latency = Histogram('http_request_duration_seconds', 'Request latency')
db_connections = Gauge('database_connections_active', 'Active DB connections')
cache_hits = Counter('cache_hits_total', 'Cache hits')
cache_misses = Counter('cache_misses_total', 'Cache misses')

# Métriques métier
orders_created = Counter('shop_orders_created_total', 'Orders created')
order_value = Histogram('shop_order_value_euros', 'Order value in euros',
                       buckets=[10, 25, 50, 100, 250, 500, 1000])
cart_abandonment = Counter('shop_carts_abandoned_total', 'Abandoned carts')
payment_failures = Counter('shop_payment_failures_total', 'Payment failures',
                         ['reason'])
active_shopping_sessions = Gauge('shop_active_sessions', 'Active shopping sessions')
```

## RED Method et USE Method

Deux méthodologies populaires guident le choix des métriques à implémenter.

### RED Method (pour les services)

RED signifie Rate, Errors, Duration. Pour chaque service, mesurez :

- **Rate** : nombre de requêtes par seconde
- **Errors** : nombre de requêtes échouées
- **Duration** : temps de traitement des requêtes

```python
# Implémentation RED
rate = Counter('service_requests_total', 'Request rate')
errors = Counter('service_errors_total', 'Error rate')
duration = Histogram('service_duration_seconds', 'Request duration')
```

### USE Method (pour les ressources)

USE signifie Utilization, Saturation, Errors. Pour chaque ressource, mesurez :

- **Utilization** : pourcentage de temps occupé
- **Saturation** : quantité de travail en attente
- **Errors** : nombre d'erreurs

```python
# Implémentation USE pour un pool de workers
utilization = Gauge('worker_pool_utilization_ratio', 'Pool utilization')
saturation = Gauge('worker_pool_queue_length', 'Tasks waiting')
errors = Counter('worker_pool_errors_total', 'Processing errors')
```

## Intégration avec les dashboards Grafana

Une fois vos métriques custom exposées, créez des dashboards Grafana pertinents.

### Requêtes PromQL pour métriques custom

Quelques exemples de requêtes utiles :

```promql
# Taux de requêtes par seconde
rate(app_requests_total[5m])

# Latence p95
histogram_quantile(0.95, rate(app_request_duration_seconds_bucket[5m]))

# Taux d'erreur en pourcentage
100 * sum(rate(app_requests_total{status=~"5.."}[5m]))
    / sum(rate(app_requests_total[5m]))

# Évolution du nombre d'utilisateurs actifs
app_active_users

# Top 5 des endpoints les plus lents
topk(5, histogram_quantile(0.99, rate(app_request_duration_seconds_bucket[5m])))
```

### Organisation des dashboards

Structurez vos dashboards par niveau d'abstraction :

1. **Vue d'ensemble** : métriques business critiques
2. **Santé des services** : RED metrics pour chaque service
3. **Détails techniques** : métriques d'infrastructure et de performance
4. **Debugging** : métriques détaillées pour investigation

## Debugging avec les métriques custom

Les métriques custom sont particulièrement utiles pour le debugging en production.

### Métriques de debug temporaires

Parfois, vous avez besoin de métriques supplémentaires pour investiguer un problème :

```python
# Feature flag pour activer les métriques de debug
if DEBUG_METRICS_ENABLED:
    detailed_processing = Histogram(
        'debug_processing_steps_duration',
        'Detailed processing time',
        ['step']
    )

    # Dans le code
    with timer(detailed_processing.labels(step='validation')):
        validate_data()
    with timer(detailed_processing.labels(step='transformation')):
        transform_data()
```

### Correlation avec les logs

Enrichissez vos logs avec les valeurs de métriques pour faciliter la corrélation :

```python
logger.info("Request processed", extra={
    'duration': duration,
    'endpoint': endpoint,
    'active_connections': active_connections.get(),
    'queue_size': queue_size.get()
})
```

## Alerting sur métriques custom

Définissez des alertes Prometheus basées sur vos métriques métier :

```yaml
groups:
- name: business_alerts
  rules:
  - alert: HighCartAbandonment
    expr: rate(shop_carts_abandoned_total[1h]) > 10
    for: 5m
    annotations:
      summary: "High cart abandonment rate"
      description: "{{ $value }} carts/hour abandoned"

  - alert: PaymentSystemFailure
    expr: rate(shop_payment_failures_total[5m]) > 0.1
    for: 2m
    annotations:
      summary: "Payment system experiencing failures"
      description: "{{ $value }} failures per second"
```

## Migration vers OpenTelemetry

OpenTelemetry est le futur de l'instrumentation. Voici comment préparer la migration :

### Approche hybride

Commencez par utiliser OpenTelemetry pour les nouvelles applications tout en gardant Prometheus pour l'existant :

```python
from opentelemetry import metrics
from opentelemetry.exporter.prometheus import PrometheusMetricReader

# Configuration OpenTelemetry avec export Prometheus
meter = metrics.get_meter(__name__)

# Création de métriques OpenTelemetry
request_counter = meter.create_counter(
    "requests",
    description="Number of requests",
    unit="1"
)

# Utilisation identique
request_counter.add(1, {"method": "GET", "endpoint": "/api"})
```

### Avantages d'OpenTelemetry

- **Unified SDK** : même API pour métriques, traces et logs
- **Auto-instrumentation** : support automatique des frameworks populaires
- **Vendor-agnostic** : changez de backend sans modifier le code
- **Context propagation** : correlation automatique entre signaux

## Conclusion et prochaines étapes

Les métriques custom transforment votre capacité à comprendre et optimiser vos applications. Dans votre lab MicroK8s, elles vous permettent d'expérimenter avec l'observabilité professionnelle sans les contraintes de la production.

Commencez simple avec quelques métriques RED/USE, puis enrichissez progressivement avec des métriques métier spécifiques. Rappelez-vous que chaque métrique a un coût : choisissez celles qui apportent vraiment de la valeur.

Dans la prochaine section, nous explorerons la centralisation des logs, le deuxième pilier de l'observabilité, qui complètera parfaitement vos métriques custom en fournissant le contexte détaillé derrière les chiffres.

⏭️

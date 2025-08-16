🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.4 Tracing distribué avec Jaeger

## Introduction : Le troisième pilier de l'observabilité

Imaginez que vous êtes détective et qu'un utilisateur se plaint que votre application est lente. Les métriques Prometheus vous disent que la latence moyenne a augmenté de 200ms à 2 secondes. Les logs Loki montrent quelques timeouts de base de données. Mais quelle est la vraie histoire ? Où ces 1.8 secondes supplémentaires sont-elles passées ? C'est là qu'intervient le tracing distribué.

Le tracing distribué est comme un GPS pour vos requêtes. Il suit chaque requête à travers tous les services, enregistrant exactement combien de temps est passé dans chaque composant. Au lieu de deviner où est le goulot d'étranglement, vous le voyez visuellement dans une timeline détaillée.

### Pourquoi le tracing est indispensable dans Kubernetes

Dans une architecture monolithique, une requête reste dans un seul processus. Le profiling traditionnel suffit pour comprendre les performances. Mais dans Kubernetes avec des microservices, une simple requête utilisateur peut :

- Traverser 10+ services différents
- Être traitée par des pods sur différents nodes
- Utiliser plusieurs bases de données et caches
- Déclencher des appels asynchrones et des queues

Sans tracing distribué, comprendre le chemin exact et les timings d'une requête spécifique est pratiquement impossible. C'est comme essayer de suivre un colis sans numéro de suivi - vous savez qu'il est parti et arrivé, mais le voyage entre les deux reste un mystère.

### Concepts fondamentaux du tracing

**Trace** : L'enregistrement complet du parcours d'une requête à travers votre système. Une trace est composée de plusieurs spans.

**Span** : Une unité de travail individuelle dans une trace. Par exemple, un appel HTTP, une requête database, ou un calcul interne. Chaque span a un début, une fin, et peut avoir des spans enfants.

**Trace Context** : L'identifiant unique qui lie tous les spans d'une même trace. Il est propagé de service en service, permettant de reconstituer le parcours complet.

**Exemple concret** : Quand un utilisateur charge une page produit :
```
[Trace: abc123]
├── [Span: Frontend] durée: 2000ms
│   ├── [Span: API Gateway] durée: 1950ms
│   │   ├── [Span: Product Service] durée: 500ms
│   │   │   └── [Span: Database Query] durée: 450ms
│   │   ├── [Span: Inventory Service] durée: 300ms
│   │   │   └── [Span: Cache Lookup] durée: 50ms
│   │   └── [Span: Review Service] durée: 1100ms
│   │       ├── [Span: Database Query] durée: 900ms
│   │       └── [Span: ML Scoring] durée: 150ms
```

D'un coup d'œil, vous voyez que le Review Service avec sa requête database est le goulot d'étranglement.

## Jaeger : L'outil de tracing pour Kubernetes

### Qu'est-ce que Jaeger ?

Jaeger (prononcé "yay-gur", mot allemand pour "chasseur") est un système de tracing distribué open source créé par Uber et maintenant partie de la CNCF. Il est spécifiquement conçu pour les environnements cloud-native comme Kubernetes.

### Architecture de Jaeger

```
[Applications Instrumentées]
           ↓
    [Jaeger Agent]  (sidecar ou DaemonSet)
           ↓
    [Jaeger Collector]
           ↓
    [Storage Backend]  (Elasticsearch, Cassandra, ou mémoire)
           ↓
    [Jaeger Query]
           ↓
    [Jaeger UI]
```

**Jaeger Agent** : Un daemon léger qui reçoit les spans des applications. Il peut s'exécuter comme sidecar dans chaque pod ou comme DaemonSet sur chaque node.

**Jaeger Collector** : Reçoit les traces des agents, les valide, les enrichit et les stocke.

**Storage Backend** : Stocke les traces. Pour un lab, nous utiliserons le stockage en mémoire ou Elasticsearch.

**Jaeger Query** : Service qui expose l'API pour rechercher les traces.

**Jaeger UI** : Interface web intuitive pour visualiser et analyser les traces.

## Installation de Jaeger sur MicroK8s

### Option 1 : Installation All-in-One (parfait pour un lab)

La version all-in-one combine tous les composants dans un seul déploiement, idéal pour un lab :

```yaml
# jaeger-all-in-one.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tracing
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: tracing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:1.52
        ports:
        - containerPort: 5775
          protocol: UDP
          name: zk-compact-trft
        - containerPort: 6831
          protocol: UDP
          name: jaeger-compact
        - containerPort: 6832
          protocol: UDP
          name: jaeger-binary
        - containerPort: 5778
          protocol: TCP
          name: config-rest
        - containerPort: 16686
          protocol: TCP
          name: jaeger-ui
        - containerPort: 14268
          protocol: TCP
          name: jaeger-coll
        - containerPort: 14250
          protocol: TCP
          name: grpc
        - containerPort: 9411
          protocol: TCP
          name: zipkin
        env:
        - name: COLLECTOR_ZIPKIN_HOST_PORT
          value: ":9411"
        - name: COLLECTOR_OTLP_ENABLED
          value: "true"
        - name: SPAN_STORAGE_TYPE
          value: memory
        - name: MEMORY_MAX_TRACES
          value: "10000"
        resources:
          requests:
            memory: 256Mi
            cpu: 100m
          limits:
            memory: 512Mi
            cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-collector
  namespace: tracing
spec:
  selector:
    app: jaeger
  ports:
  - name: jaeger-collector-tchannel
    port: 14267
    protocol: TCP
  - name: jaeger-collector-http
    port: 14268
    protocol: TCP
  - name: jaeger-collector-grpc
    port: 14250
    protocol: TCP
  - name: zipkin
    port: 9411
    protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-query
  namespace: tracing
spec:
  selector:
    app: jaeger
  ports:
  - name: jaeger-query
    port: 16686
    protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-agent
  namespace: tracing
spec:
  selector:
    app: jaeger
  clusterIP: None
  ports:
  - name: agent-zipkin-thrift
    port: 5775
    protocol: UDP
  - name: agent-compact
    port: 6831
    protocol: UDP
  - name: agent-binary
    port: 6832
    protocol: UDP
  - name: agent-configs
    port: 5778
    protocol: TCP
```

Déployez avec :
```bash
kubectl apply -f jaeger-all-in-one.yaml
```

### Option 2 : Installation avec Jaeger Operator

Pour une installation plus robuste et configurable :

```bash
# Installer l'operator
kubectl create namespace observability
kubectl create -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.52.0/jaeger-operator.yaml -n observability

# Attendre que l'operator soit prêt
kubectl wait --for=condition=available deployment/jaeger-operator -n observability --timeout=300s
```

Puis créez une instance Jaeger :

```yaml
# jaeger-instance.yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-lab
  namespace: tracing
spec:
  strategy: allInOne
  allInOne:
    image: jaegertracing/all-in-one:1.52
    options:
      memory.max-traces: 10000
      query.max-clock-skew-adjustment: 30s
  storage:
    type: memory
    options:
      memory:
        max-traces: 10000
  ingress:
    enabled: false
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 256Mi
```

### Option 3 : Jaeger avec stockage Elasticsearch

Pour persister les traces au-delà des redémarrages :

```yaml
# jaeger-with-elasticsearch.yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-es
  namespace: tracing
spec:
  strategy: production
  collector:
    replicas: 1
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch.logging.svc.cluster.local:9200
        index-prefix: jaeger
        username: elastic
        password: changeme
    esIndexCleaner:
      enabled: true
      numberOfDays: 7
      schedule: "55 23 * * *"
  query:
    replicas: 1
    resources:
      limits:
        cpu: 300m
        memory: 256Mi
```

### Exposition de l'interface Jaeger

Créez un Ingress pour accéder à Jaeger UI :

```yaml
# jaeger-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jaeger-ui
  namespace: tracing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: jaeger.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jaeger-query
            port:
              number: 16686
```

Ou utilisez port-forward pour un accès rapide :
```bash
kubectl port-forward -n tracing svc/jaeger-query 16686:16686
```

## Instrumentation des applications

### OpenTelemetry : Le standard unifié

OpenTelemetry est le standard de facto pour l'instrumentation. Il unifie OpenTracing et OpenCensus, supportant métriques, logs et traces.

#### Python avec OpenTelemetry

```python
# requirements.txt
opentelemetry-api==1.21.0
opentelemetry-sdk==1.21.0
opentelemetry-instrumentation-flask==0.42b0
opentelemetry-instrumentation-requests==0.42b0
opentelemetry-exporter-jaeger==1.21.0

# app.py
from flask import Flask, request
import requests
from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.trace import Status, StatusCode
import time

# Configuration du tracing
resource = Resource.create({
    "service.name": "python-api",
    "service.version": "1.0.0",
    "deployment.environment": "lab"
})

provider = TracerProvider(resource=resource)
trace.set_tracer_provider(provider)

# Exporter vers Jaeger
jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger-agent.tracing.svc.cluster.local",
    agent_port=6831,
)

# Processeur de spans
span_processor = BatchSpanProcessor(jaeger_exporter)
provider.add_span_processor(span_processor)

# Tracer pour l'application
tracer = trace.get_tracer(__name__)

# Application Flask
app = Flask(__name__)

# Auto-instrumentation
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

@app.route('/api/process')
def process_request():
    """Endpoint avec tracing manuel détaillé"""
    with tracer.start_as_current_span("process_request") as span:
        # Ajouter des attributs au span
        span.set_attribute("http.method", request.method)
        span.set_attribute("http.url", request.url)
        span.set_attribute("user.id", request.headers.get("X-User-ID", "anonymous"))

        try:
            # Étape 1: Validation
            with tracer.start_as_current_span("validate_input") as validate_span:
                time.sleep(0.05)  # Simulation validation
                validate_span.set_attribute("validation.passed", True)

            # Étape 2: Appel base de données
            with tracer.start_as_current_span("database_query") as db_span:
                db_span.set_attribute("db.system", "postgresql")
                db_span.set_attribute("db.statement", "SELECT * FROM products WHERE id = ?")
                time.sleep(0.2)  # Simulation query
                db_span.set_attribute("db.rows_affected", 1)

            # Étape 3: Appel service externe
            with tracer.start_as_current_span("external_api_call") as api_span:
                api_span.set_attribute("http.url", "http://inventory-service/check")
                response = requests.get("http://inventory-service:8080/check")
                api_span.set_attribute("http.status_code", response.status_code)

                if response.status_code != 200:
                    api_span.set_status(Status(StatusCode.ERROR, "External API failed"))

            # Étape 4: Processing
            with tracer.start_as_current_span("business_logic") as logic_span:
                result = complex_calculation()
                logic_span.set_attribute("result.size", len(result))

            return {"status": "success", "data": result}

        except Exception as e:
            span.record_exception(e)
            span.set_status(Status(StatusCode.ERROR, str(e)))
            raise

def complex_calculation():
    """Fonction avec son propre span"""
    with tracer.start_as_current_span("complex_calculation") as span:
        span.set_attribute("calculation.type", "fibonacci")
        time.sleep(0.1)
        return [1, 1, 2, 3, 5, 8, 13]

@app.route('/api/chain')
def chain_request():
    """Démontre le tracing à travers plusieurs services"""
    with tracer.start_as_current_span("chain_orchestrator") as span:
        # Propager le contexte de trace
        headers = {}
        trace.get_current_span().context

        # Appel service A
        with tracer.start_as_current_span("call_service_a"):
            response_a = requests.get(
                "http://service-a:8080/process",
                headers=headers
            )

        # Appel service B avec le résultat de A
        with tracer.start_as_current_span("call_service_b"):
            response_b = requests.post(
                "http://service-b:8080/transform",
                json=response_a.json(),
                headers=headers
            )

        return response_b.json()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

#### Node.js avec OpenTelemetry

```javascript
// package.json dependencies
{
  "dependencies": {
    "@opentelemetry/api": "^1.7.0",
    "@opentelemetry/sdk-node": "^0.45.0",
    "@opentelemetry/auto-instrumentations-node": "^0.40.0",
    "@opentelemetry/exporter-jaeger": "^1.18.0",
    "express": "^4.18.0",
    "axios": "^1.6.0"
  }
}

// tracing.js - Configuration du tracing
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

// Configuration de l'exporter Jaeger
const jaegerExporter = new JaegerExporter({
  endpoint: 'http://jaeger-collector.tracing.svc.cluster.local:14268/api/traces',
});

// Configuration de la resource
const resource = Resource.default().merge(
  new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'node-api',
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: 'lab',
  })
);

// Initialisation du SDK
const sdk = new NodeSDK({
  resource,
  traceExporter: jaegerExporter,
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': {
        enabled: false, // Désactiver fs pour éviter le bruit
      },
    }),
  ],
});

// Démarrer le SDK
sdk.start()
  .then(() => console.log('Tracing initialized'))
  .catch((error) => console.log('Error initializing tracing', error));

// Graceful shutdown
process.on('SIGTERM', () => {
  sdk.shutdown()
    .then(() => console.log('Tracing terminated'))
    .catch((error) => console.log('Error terminating tracing', error))
    .finally(() => process.exit(0));
});

module.exports = sdk;

// app.js - Application principale
require('./tracing'); // Doit être importé en premier!

const express = require('express');
const axios = require('axios');
const { trace, context, SpanStatusCode } = require('@opentelemetry/api');

const app = express();
app.use(express.json());

// Obtenir le tracer
const tracer = trace.getTracer('node-api', '1.0.0');

// Middleware pour ajouter des attributs à tous les spans
app.use((req, res, next) => {
  const span = trace.getActiveSpan();
  if (span) {
    span.setAttributes({
      'http.method': req.method,
      'http.target': req.path,
      'http.host': req.hostname,
      'user.id': req.headers['x-user-id'] || 'anonymous',
    });
  }
  next();
});

// Route avec tracing manuel
app.get('/api/products/:id', async (req, res) => {
  // Créer un span parent
  return tracer.startActiveSpan('get_product', async (span) => {
    try {
      span.setAttributes({
        'product.id': req.params.id,
        'request.type': 'product_detail',
      });

      // Étape 1: Récupérer depuis la base de données
      const product = await tracer.startActiveSpan('database_query', async (dbSpan) => {
        dbSpan.setAttributes({
          'db.system': 'postgresql',
          'db.name': 'products',
          'db.operation': 'SELECT',
          'db.statement': `SELECT * FROM products WHERE id = ${req.params.id}`,
        });

        // Simuler une requête DB
        await new Promise(resolve => setTimeout(resolve, 100));

        const result = {
          id: req.params.id,
          name: 'Sample Product',
          price: 99.99
        };

        dbSpan.setAttributes({
          'db.rows_affected': 1,
        });

        dbSpan.end();
        return result;
      });

      // Étape 2: Enrichir avec l'inventaire
      const inventory = await tracer.startActiveSpan('inventory_check', async (invSpan) => {
        invSpan.setAttributes({
          'service.name': 'inventory-service',
          'product.id': req.params.id,
        });

        try {
          const response = await axios.get(`http://inventory-service:8080/stock/${req.params.id}`);
          invSpan.setAttributes({
            'http.status_code': response.status,
            'inventory.available': response.data.available,
          });
          invSpan.end();
          return response.data;
        } catch (error) {
          invSpan.recordException(error);
          invSpan.setStatus({
            code: SpanStatusCode.ERROR,
            message: error.message,
          });
          invSpan.end();
          return { available: 0 };
        }
      });

      // Étape 3: Calculer le prix final
      const finalPrice = await tracer.startActiveSpan('calculate_price', async (priceSpan) => {
        priceSpan.setAttributes({
          'price.base': product.price,
          'price.currency': 'USD',
        });

        // Logique de calcul
        const discount = 0.1;
        const tax = 0.08;
        const final = product.price * (1 - discount) * (1 + tax);

        priceSpan.setAttributes({
          'price.discount': discount,
          'price.tax': tax,
          'price.final': final,
        });

        priceSpan.end();
        return final;
      });

      // Retourner le résultat
      const response = {
        ...product,
        inventory: inventory.available,
        finalPrice
      };

      span.setAttributes({
        'response.size': JSON.stringify(response).length,
      });

      span.end();
      res.json(response);

    } catch (error) {
      span.recordException(error);
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error.message,
      });
      span.end();
      res.status(500).json({ error: 'Internal server error' });
    }
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### Configuration Kubernetes pour le tracing

Déployez vos applications instrumentées :

```yaml
# deployment-with-tracing.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: default
  labels:
    app: api-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
      annotations:
        sidecar.jaegertracing.io/inject: "true"  # Injection automatique du sidecar
    spec:
      containers:
      - name: api
        image: myregistry/api-server:latest
        env:
        - name: OTEL_SERVICE_NAME
          value: "api-server"
        - name: OTEL_EXPORTER_JAEGER_ENDPOINT
          value: "http://jaeger-collector.tracing.svc.cluster.local:14268/api/traces"
        - name: OTEL_TRACES_EXPORTER
          value: "jaeger"
        - name: OTEL_METRICS_EXPORTER
          value: "prometheus"
        - name: OTEL_PROPAGATORS
          value: "tracecontext,baggage,jaeger"
        - name: OTEL_RESOURCE_ATTRIBUTES
          value: "deployment.environment=lab,service.namespace=default"
        ports:
        - containerPort: 8080
          name: http
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: api-server
spec:
  selector:
    app: api-server
  ports:
  - port: 8080
    targetPort: 8080
```

## Utilisation de Jaeger UI

### Navigation dans l'interface

L'interface Jaeger est intuitive et puissante. Voici les principales sections :

**Search** : Recherchez des traces par service, opération, tags, ou durée.

**Trace Timeline** : Visualisez le waterfall de tous les spans d'une trace.

**Service Dependencies** : Graphique des dépendances entre services basé sur les traces.

**Performance Monitoring** : Analyse des latences et taux d'erreur par endpoint.

### Recherche de traces

Exemples de requêtes de recherche :

```
# Par service
service="api-server"

# Par opération
service="api-server" operation="GET /api/products"

# Par durée (traces lentes)
service="api-server" minDuration="1s"

# Par tags
service="api-server" tags="error=true"
service="api-server" tags="http.status_code=500"
service="api-server" tags="user.id=user-123"

# Combinaisons
service="api-server" operation="database_query" minDuration="500ms"
```

### Analyse d'une trace

Une fois une trace sélectionnée, vous voyez :

1. **Timeline View** : Waterfall montrant la durée de chaque span
2. **Span Details** : Tags, logs, et process info pour chaque span
3. **Critical Path** : Le chemin le plus long dans la trace
4. **Service Map** : Relations entre services dans cette trace

### Comparaison de traces

Jaeger permet de comparer deux traces pour identifier les différences :

1. Sélectionnez une trace de référence (performante)
2. Comparez avec une trace problématique
3. Les différences de timing sont mises en évidence
4. Identifiez rapidement les régressions de performance

## Patterns d'instrumentation avancés

### Baggage Propagation

Le baggage permet de propager des données contextuelles à travers toute la trace :

```python
from opentelemetry import baggage

# Définir du baggage au début
ctx = baggage.set_baggage("user.id", "user-123")
ctx = baggage.set_baggage("tenant.id", "tenant-456")
ctx = baggage.set_baggage("feature.flags", "new-ui=true")

# Le baggage est automatiquement propagé
with tracer.start_as_current_span("operation", context=ctx) as span:
    # Récupérer le baggage n'importe où
    user_id = baggage.get_baggage("user.id")
    span.set_attribute("user.id", user_id)
```

### Sampling intelligent

Pour réduire le volume de traces sans perdre les traces importantes :

```python
from opentelemetry.sdk.trace.sampling import (
    TraceIdRatioBased,
    ParentBased,
    AlwaysOn,
    AlwaysOff,
    Sampler,
    SamplingResult,
    Decision
)

# Sampler personnalisé
class SmartSampler(Sampler):
    def should_sample(self, context, trace_id, name, kind, attributes, links):
        # Toujours échantillonner les erreurs
        if attributes and attributes.get("error") == True:
            return SamplingResult(Decision.RECORD_AND_SAMPLE)

        # Toujours échantillonner les requêtes lentes
        if attributes and attributes.get("duration.ms", 0) > 1000:
            return SamplingResult(Decision.RECORD_AND_SAMPLE)

        # Échantillonner 10% du reste
        if trace_id % 10 == 0:
            return SamplingResult(Decision.RECORD_AND_SAMPLE)

        return SamplingResult(Decision.DROP)

# Configuration
provider = TracerProvider(
    resource=resource,
    sampler=ParentBased(root=SmartSampler())
)
```

### Correlation avec logs et métriques

```python
import logging
from opentelemetry import trace
from prometheus_client import Histogram

# Configuration du logger avec trace context
class TraceContextFilter(logging.Filter):
    def filter(self, record):
        span = trace.get_current_span()
        if span:
            span_context = span.get_span_context()
            record.trace_id = format(span_context.trace_id, '032x')
            record.span_id = format(span_context.span_id, '016x')
        else:
            record.trace_id = '00000000000000000000000000000000'
            record.span_id = '0000000000000000'
        return True

# Configuration du logging
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - [trace_id=%(trace_id)s span_id=%(span_id)s] - %(message)s'
)
logger = logging.getLogger(__name__)
logger.addFilter(TraceContextFilter())

# Métriques avec labels de trace
request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint', 'status', 'traced']
)

@app.route('/api/data')
def get_data():
    with tracer.start_as_current_span("get_data") as span:
        start_time = time.time()

        # Log avec contexte de trace
        logger.info("Processing data request")

        try:
            result = process_data()
            duration = time.time() - start_time

            # Métrique avec indication de trace
            request_duration.labels(
                method='GET',
                endpoint='/api/data',
                status='200',
                traced='true'
            ).observe(duration)

            # Ajouter le trace ID à la réponse
            span_context = span.get_span_context()
            return {
                'data': result,
                'trace_id': format(span_context.trace_id, '032x')
            }

        except Exception as e:
            logger.error(f"Error processing data: {e}")
            span.record_exception(e)
            span.set_status(Status(StatusCode.ERROR))
            raise
```

## Intégration avec Grafana

### Configuration de Jaeger comme datasource

Ajoutez Jaeger à Grafana pour une vue unifiée :

```yaml
# grafana-datasource-jaeger.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasource-jaeger
  namespace: monitoring
data:
  jaeger.yaml: |
    apiVersion: 1
    datasources:
    - name: Jaeger
      type: jaeger
      access: proxy
      url: http://jaeger-query.tracing.svc.cluster.local:16686
      jsonData:
        tracesToLogs:
          datasourceUid: loki-uid
          tags: ['trace_id']
          mappedTags:
            - key: 'service.name'
              value: 'service'
          spanStartTimeShift: '-1h'
          spanEndTimeShift: '1h'
          filterByTraceID: true
          filterBySpanID: false
        tracesToMetrics:
          datasourceUid: prometheus-uid
          tags:
            - key: 'service.name'
              value: 'service'
            - key: 'operation'
              value: 'operation'
          queries:
            - name: 'Request Rate'
              query: 'rate(http_requests_total{service="${service}"}[5m])'
            - name: 'Error Rate'
              query: 'rate(http_requests_total{service="${service}",status=~"5.."}[5m])'
```

### Dashboard unifié avec traces

Créez un dashboard combinant métriques, logs et traces :

```json
{
  "dashboard": {
    "title": "Full Observability Dashboard",
    "panels": [
      {
        "id": 1,
        "title": "Service Latency (Prometheus)",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [{
          "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))"
        }],
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 }
      },
      {
        "id": 2,
        "title": "Error Logs (Loki)",
        "type": "logs",
        "datasource": "Loki",
        "targets": [{
          "expr": "{service=\"$service\", level=\"ERROR\"}"
        }],
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 }
      },
      {
        "id": 3,
        "title": "Trace Search (Jaeger)",
        "type": "traces",
        "datasource": "Jaeger",
        "targets": [{
          "query": "service=\"$service\" operation=\"$operation\""
        }],
        "gridPos": { "h": 10, "w": 24, "x": 0, "y": 8 }
      },
      {
        "id": 4,
        "title": "Service Dependencies",
        "type": "nodeGraph",
        "datasource": "Jaeger",
        "targets": [{
          "query": "service=\"$service\""
        }],
        "gridPos": { "h": 10, "w": 24, "x": 0, "y": 18 }
      }
    ],
    "templating": {
      "list": [
        {
          "name": "service",
          "type": "query",
          "datasource": "Prometheus",
          "query": "label_values(service)",
          "current": { "value": "api-server" }
        },
        {
          "name": "operation",
          "type": "query",
          "datasource": "Jaeger",
          "query": "operations($service)",
          "current": { "value": "GET /api/products" }
        }
      ]
    }
  }
}
```

### Liens contextuels entre panels

Configurez des data links pour naviguer entre traces, logs et métriques :

```javascript
// Panel de traces avec lien vers logs
{
  "fieldConfig": {
    "defaults": {
      "links": [{
        "title": "View Logs for this Trace",
        "url": "/explore?left={\"datasource\":\"Loki\",\"queries\":[{\"expr\":\"{trace_id=\\\"${__data.fields.traceID}\\\"}\"}],\"range\":{\"from\":\"${__data.fields.startTime}\",\"to\":\"${__data.fields.endTime}\"}}"
      }]
    }
  }
}

// Panel de logs avec lien vers trace
{
  "fieldConfig": {
    "defaults": {
      "links": [{
        "title": "View Trace",
        "url": "/explore?left={\"datasource\":\"Jaeger\",\"queries\":[{\"query\":\"${__data.fields.trace_id}\"}]}"
      }]
    }
  }
}

// Panel de métriques avec lien vers traces
{
  "fieldConfig": {
    "defaults": {
      "links": [{
        "title": "View Traces",
        "url": "/explore?left={\"datasource\":\"Jaeger\",\"queries\":[{\"query\":\"service=\\\"${__field.labels.service}\\\" operation=\\\"${__field.labels.operation}\\\" minDuration=\\\"${__value}ms\\\"\"}]}"
      }]
    }
  }
}
```

## Cas d'usage pratiques

### Debugging d'une requête lente

Workflow d'investigation complet :

```python
# 1. Identifier la requête lente dans les métriques
# Prometheus query:
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 2

# 2. Rechercher les traces correspondantes dans Jaeger
service="api-server" operation="GET /api/products" minDuration="2s"

# 3. Analyser la trace pour identifier le bottleneck
# - Span le plus long : database_query (1.8s)
# - Tags du span : db.statement="SELECT * FROM products JOIN..."

# 4. Vérifier les logs de la base de données
{service="postgres"} |= "slow query" |= "products"

# 5. Optimiser et vérifier l'amélioration
```

Script d'analyse automatisée :

```python
#!/usr/bin/env python3
# analyze_slow_traces.py

import requests
import json
from datetime import datetime, timedelta

JAEGER_URL = "http://localhost:16686"
PROMETHEUS_URL = "http://localhost:9090"
LOKI_URL = "http://localhost:3100"

def find_slow_traces(service, lookback_hours=1):
    """Trouve les traces lentes pour un service"""
    end_time = datetime.now()
    start_time = end_time - timedelta(hours=lookback_hours)

    # Requête Jaeger pour les traces lentes
    params = {
        'service': service,
        'start': int(start_time.timestamp() * 1000000),
        'end': int(end_time.timestamp() * 1000000),
        'minDuration': '1s',
        'limit': 100
    }

    response = requests.get(f"{JAEGER_URL}/api/traces", params=params)
    traces = response.json()['data']

    slow_operations = {}
    for trace in traces:
        for span in trace['spans']:
            operation = span['operationName']
            duration = span['duration'] / 1000  # Convert to ms

            if operation not in slow_operations or duration > slow_operations[operation]['duration']:
                slow_operations[operation] = {
                    'duration': duration,
                    'trace_id': trace['traceID'],
                    'span_id': span['spanID'],
                    'tags': {tag['key']: tag['value'] for tag in span['tags']}
                }

    return slow_operations

def get_related_logs(trace_id):
    """Récupère les logs associés à une trace"""
    query = f'{{trace_id="{trace_id}"}}'
    params = {
        'query': query,
        'limit': 100
    }

    response = requests.get(f"{LOKI_URL}/loki/api/v1/query", params=params)
    return response.json()['data']['result']

def analyze_performance(service):
    """Analyse complète des performances d'un service"""
    print(f"Analyzing performance for service: {service}")
    print("=" * 50)

    # Trouver les opérations lentes
    slow_ops = find_slow_traces(service)

    for operation, details in slow_ops.items():
        print(f"\nSlow Operation: {operation}")
        print(f"  Duration: {details['duration']:.2f}ms")
        print(f"  Trace ID: {details['trace_id']}")

        # Analyser les tags pour comprendre le contexte
        if 'db.statement' in details['tags']:
            print(f"  DB Query: {details['tags']['db.statement'][:100]}...")

        if 'http.status_code' in details['tags']:
            print(f"  HTTP Status: {details['tags']['http.status_code']}")

        # Récupérer les logs associés
        logs = get_related_logs(details['trace_id'])
        if logs:
            print(f"  Related logs found: {len(logs)}")
            for log_stream in logs[:3]:  # Afficher les 3 premiers
                for entry in log_stream['values'][:2]:  # 2 premières entrées
                    print(f"    - {entry[1][:100]}...")

if __name__ == "__main__":
    analyze_performance("api-server")
```

### Analyse des dépendances de services

```python
# Script pour analyser et visualiser les dépendances
import requests
from collections import defaultdict
import networkx as nx
import matplotlib.pyplot as plt

def analyze_dependencies():
    """Analyse les dépendances entre services depuis Jaeger"""
    # Récupérer les dépendances depuis Jaeger API
    response = requests.get(f'{JAEGER_URL}/api/dependencies',
                           params={'endTs': int(time.time() * 1000)})
    dependencies = response.json()

    # Construire le graphe
    graph = defaultdict(list)
    call_counts = {}

    for dep in dependencies:
        parent = dep['parent']
        child = dep['child']
        call_count = dep['callCount']

        graph[parent].append(child)
        call_counts[f"{parent}->{child}"] = call_count

    # Identifier les services critiques (beaucoup de dépendances)
    critical_services = []
    for service, deps in graph.items():
        if len(deps) > 3:  # Seuil arbitraire
            critical_services.append({
                'service': service,
                'dependencies': len(deps),
                'total_calls': sum(call_counts[f"{service}->{d}"] for d in deps)
            })

    # Détecter les cycles potentiels
    G = nx.DiGraph()
    for parent, children in graph.items():
        for child in children:
            G.add_edge(parent, child, weight=call_counts[f"{parent}->{child}"])

    cycles = list(nx.simple_cycles(G))

    # Visualiser le graphe
    plt.figure(figsize=(12, 8))
    pos = nx.spring_layout(G)

    # Colorer les nœuds critiques
    node_colors = ['red' if node in [c['service'] for c in critical_services] else 'lightblue'
                   for node in G.nodes()]

    nx.draw(G, pos, node_color=node_colors, with_labels=True,
            node_size=3000, font_size=10, font_weight='bold')

    # Ajouter les poids des arêtes (call counts)
    edge_labels = nx.get_edge_attributes(G, 'weight')
    nx.draw_networkx_edge_labels(G, pos, edge_labels)

    plt.title("Service Dependencies Graph")
    plt.savefig('service_dependencies.png')

    return {
        'graph': dict(graph),
        'critical_services': critical_services,
        'cycles': cycles,
        'total_services': len(G.nodes()),
        'total_dependencies': len(G.edges())
    }

# Utilisation
result = analyze_dependencies()
print(f"Services critiques: {result['critical_services']}")
print(f"Cycles détectés: {result['cycles']}")
```

### Monitoring de SLO avec traces

```python
from collections import defaultdict
from prometheus_client import Gauge, Counter
import time

# Métriques pour SLO
slo_latency_gauge = Gauge('slo_latency_seconds', 'Current latency', ['service', 'operation'])
slo_error_rate = Gauge('slo_error_rate', 'Current error rate', ['service'])
slo_availability = Gauge('slo_availability', 'Service availability', ['service'])
slo_violations = Counter('slo_violations_total', 'SLO violations', ['service', 'type'])

class SLOMonitor:
    def __init__(self, latency_slo=1.0, error_rate_slo=0.01):
        """
        latency_slo: Latence maximale acceptable en secondes
        error_rate_slo: Taux d'erreur maximal acceptable (1%)
        """
        self.latency_slo = latency_slo
        self.error_rate_slo = error_rate_slo
        self.metrics = defaultdict(lambda: {
            'requests': 0,
            'errors': 0,
            'latencies': []
        })

    def process_trace(self, trace):
        """Traite une trace et met à jour les métriques SLO"""
        service = trace['processes'][trace['spans'][0]['processID']]['serviceName']
        root_span = trace['spans'][0]

        # Calculer la latence totale
        duration = root_span['duration'] / 1000000  # Convertir en secondes

        # Vérifier si c'est une erreur
        is_error = any(tag['key'] == 'error' and tag['value'] == True
                      for tag in root_span['tags'])

        # Mettre à jour les métriques
        self.metrics[service]['requests'] += 1
        if is_error:
            self.metrics[service]['errors'] += 1
        self.metrics[service]['latencies'].append(duration)

        # Calculer et publier les métriques SLO
        self._update_slo_metrics(service, root_span['operationName'], duration, is_error)

    def _update_slo_metrics(self, service, operation, duration, is_error):
        """Met à jour les métriques Prometheus pour les SLO"""
        # Latence
        slo_latency_gauge.labels(service=service, operation=operation).set(duration)

        if duration > self.latency_slo:
            slo_violations.labels(service=service, type='latency').inc()

        # Taux d'erreur
        metrics = self.metrics[service]
        error_rate = metrics['errors'] / metrics['requests'] if metrics['requests'] > 0 else 0
        slo_error_rate.labels(service=service).set(error_rate)

        if error_rate > self.error_rate_slo:
            slo_violations.labels(service=service, type='error_rate').inc()

        # Disponibilité (inverse du taux d'erreur)
        availability = 1 - error_rate
        slo_availability.labels(service=service).set(availability)

    def get_slo_report(self, service):
        """Génère un rapport SLO pour un service"""
        if service not in self.metrics:
            return None

        metrics = self.metrics[service]
        latencies = metrics['latencies']

        if not latencies:
            return None

        # Calculer les percentiles
        latencies_sorted = sorted(latencies)
        p50 = latencies_sorted[len(latencies_sorted) // 2]
        p95 = latencies_sorted[int(len(latencies_sorted) * 0.95)]
        p99 = latencies_sorted[int(len(latencies_sorted) * 0.99)]

        error_rate = metrics['errors'] / metrics['requests'] if metrics['requests'] > 0 else 0

        return {
            'service': service,
            'total_requests': metrics['requests'],
            'total_errors': metrics['errors'],
            'error_rate': error_rate,
            'availability': 1 - error_rate,
            'latency_p50': p50,
            'latency_p95': p95,
            'latency_p99': p99,
            'slo_latency_compliance': sum(1 for l in latencies if l <= self.latency_slo) / len(latencies),
            'slo_error_rate_compliance': error_rate <= self.error_rate_slo
        }

# Script de monitoring continu
def monitor_slos():
    monitor = SLOMonitor(latency_slo=1.0, error_rate_slo=0.01)

    while True:
        # Récupérer les traces récentes
        end_time = int(time.time() * 1000000)
        start_time = end_time - 60000000  # Dernière minute

        response = requests.get(f"{JAEGER_URL}/api/traces", params={
            'start': start_time,
            'end': end_time,
            'limit': 1000
        })

        traces = response.json()['data']

        for trace in traces:
            monitor.process_trace(trace)

        # Générer les rapports
        for service in monitor.metrics.keys():
            report = monitor.get_slo_report(service)
            if report:
                print(f"SLO Report for {service}:")
                print(f"  Availability: {report['availability']*100:.2f}%")
                print(f"  P95 Latency: {report['latency_p95']*1000:.2f}ms")
                print(f"  Error Rate: {report['error_rate']*100:.2f}%")
                print(f"  SLO Compliance: {'✓' if report['slo_error_rate_compliance'] else '✗'}")
                print()

        time.sleep(60)  # Attendre 1 minute avant la prochaine vérification
```

## Optimisations pour un lab

### Configuration de sampling adaptatif

```yaml
# jaeger-sampling-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jaeger-sampling
  namespace: tracing
data:
  sampling.json: |
    {
      "service_strategies": [
        {
          "service": "api-server",
          "type": "adaptive",
          "max_traces_per_second": 100,
          "sampling_rate": 0.1
        },
        {
          "service": "database-service",
          "type": "probabilistic",
          "sampling_rate": 0.05
        },
        {
          "service": "critical-service",
          "type": "probabilistic",
          "sampling_rate": 1.0
        }
      ],
      "default_strategy": {
        "type": "probabilistic",
        "sampling_rate": 0.01,
        "operation_strategies": [
          {
            "operation": "GET /health",
            "type": "probabilistic",
            "sampling_rate": 0.001
          },
          {
            "operation": "POST /api/order",
            "type": "probabilistic",
            "sampling_rate": 1.0
          }
        ]
      }
    }
```

Application du sampling config :

```bash
kubectl create configmap jaeger-sampling --from-file=sampling.json -n tracing
kubectl set env deployment/jaeger -n tracing SAMPLING_CONFIG_TYPE=file
kubectl set env deployment/jaeger -n tracing SAMPLING_CONFIG=/etc/jaeger/sampling.json
kubectl set volume deployment/jaeger -n tracing --add \
  --name=sampling-config \
  --type=configmap \
  --configmap-name=jaeger-sampling \
  --mount-path=/etc/jaeger
```

### Rétention et nettoyage

```yaml
# jaeger-cleanup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: jaeger-cleanup
  namespace: tracing
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h du matin
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: jaeger-es-cleaner
            image: jaegertracing/jaeger-es-index-cleaner:1.52
            env:
            - name: ROLLOVER
              value: "true"
            - name: ES_SERVER_URLS
              value: "http://elasticsearch.logging:9200"
            - name: ES_USERNAME
              value: "elastic"
            - name: ES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: elasticsearch-credentials
                  key: password
            - name: NUMBER_OF_DAYS
              value: "3"  # Garder seulement 3 jours
            args:
            - "3"
            - "http://elasticsearch.logging:9200"
          restartPolicy: OnFailure

          # Script de nettoyage pour stockage en mémoire
          - name: memory-cleanup
            image: curlimages/curl:latest
            command:
            - /bin/sh
            - -c
            - |
              # Pour Jaeger all-in-one avec stockage mémoire
              # Redémarrer le pod pour libérer la mémoire
              curl -X DELETE http://jaeger-query:16686/api/traces/old
```

### Optimisation mémoire

```yaml
# jaeger-memory-optimized.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jaeger-config
  namespace: tracing
data:
  collector.yaml: |
    collector:
      queue_size: 1000         # Réduire la taille de queue
      num_workers: 5           # Réduire le nombre de workers

    processor:
      jaeger-compact:
        server-host-port: 0.0.0.0:6831
        server-max-packet-size: 65000
        workers: 5             # Limiter les workers

    # Limites de stockage
    span-storage:
      type: memory
      memory:
        max-traces: 5000       # Limiter les traces en mémoire

    # Configuration de l'ingestion
    ingestion:
      max-batch-size: 100
      max-queue-size: 1000
      batch-timeout: 1s
```

### Monitoring de Jaeger lui-même

```yaml
# jaeger-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: jaeger-metrics
  namespace: tracing
spec:
  selector:
    matchLabels:
      app: jaeger
  endpoints:
  - port: admin-http
    path: /metrics
    interval: 30s
```

Métriques importantes à surveiller :

```promql
# Traces reçues par seconde
rate(jaeger_collector_traces_received_total[5m])

# Traces rejetées (sampling)
rate(jaeger_collector_traces_rejected_total[5m])

# Latence du collector
histogram_quantile(0.99, rate(jaeger_collector_save_latency_bucket[5m]))

# Utilisation mémoire
process_resident_memory_bytes{job="jaeger"}

# Spans par trace
jaeger_collector_spans_per_trace

# Queue size
jaeger_collector_queue_length
```

## Troubleshooting

### Traces manquantes

```bash
# Vérifier que l'agent reçoit les spans
kubectl logs -n tracing deployment/jaeger -c jaeger-agent | grep "received"

# Vérifier la configuration du collector
kubectl logs -n tracing deployment/jaeger -c jaeger-collector | grep -i error

# Tester la connectivité depuis un pod
kubectl run test-jaeger --image=curlimages/curl -it --rm -- \
  curl -X POST http://jaeger-collector.tracing:14268/api/traces

# Vérifier le sampling
curl http://jaeger-agent.tracing:5778/sampling?service=api-server
```

### Performance dégradée

```bash
# Métriques Jaeger
curl http://jaeger:14269/metrics | grep jaeger

# Vérifier la charge CPU/mémoire
kubectl top pod -n tracing

# Analyser les traces lentes
curl "http://jaeger:16686/api/traces?service=jaeger-collector&minDuration=1s"

# Réduire le sampling si nécessaire
kubectl edit configmap jaeger-sampling -n tracing
```

### Problèmes de propagation du contexte

```python
# Debug de la propagation du contexte
import logging
from opentelemetry.propagate import inject, extract

logging.basicConfig(level=logging.DEBUG)

def debug_propagation():
    # Vérifier l'injection
    carrier = {}
    inject(carrier)
    print(f"Headers injectés: {carrier}")

    # Vérifier l'extraction
    test_headers = {
        'traceparent': '00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01',
        'tracestate': 'rojo=00f067aa0ba902b7'
    }
    context = extract(test_headers)
    print(f"Context extrait: {context}")

    # Vérifier la propagation entre services
    with tracer.start_as_current_span("test_propagation") as span:
        headers = {}
        inject(headers)
        print(f"Headers pour propagation: {headers}")

        # Simuler un appel HTTP
        response = requests.get("http://other-service", headers=headers)
        print(f"Réponse avec trace_id: {response.headers.get('X-Trace-Id')}")

debug_propagation()
```

## Scripts utilitaires

### Générateur de charge avec traces

```python
#!/usr/bin/env python3
# generate-traced-load.py

import time
import random
import requests
from concurrent.futures import ThreadPoolExecutor
from opentelemetry import trace
from opentelemetry.exporter.jaeger import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.trace import Status, StatusCode

# Configuration
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer("load-generator")

jaeger_exporter = JaegerExporter(
    agent_host_name="localhost",
    agent_port=6831,
)
span_processor = BatchSpanProcessor(jaeger_exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

def make_request(endpoint, delay=None, error_rate=0.1):
    """Génère une requête tracée avec possibilité d'erreur"""
    with tracer.start_as_current_span(f"load_test_{endpoint}") as span:
        span.set_attribute("test.type", "load")
        span.set_attribute("test.endpoint", endpoint)
        span.set_attribute("test.simulated", True)

        # Simuler une latence variable
        if delay:
            time.sleep(delay)
        else:
            time.sleep(random.uniform(0.01, 0.5))

        # Simuler des erreurs aléatoires
        if random.random() < error_rate:
            span.set_attribute("error", True)
            span.set_attribute("error.message", "Simulated error")
            span.set_status(Status(StatusCode.ERROR, "Simulated error"))
            return 500

        try:
            response = requests.get(f"http://localhost:8080{endpoint}")
            span.set_attribute("http.status_code", response.status_code)

            if response.status_code >= 400:
                span.set_attribute("error", True)
                span.set_status(Status(StatusCode.ERROR, f"HTTP {response.status_code}"))

            return response.status_code
        except Exception as e:
            span.record_exception(e)
            span.set_attribute("error", True)
            span.set_status(Status(StatusCode.ERROR, str(e)))
            return 500

def simulate_user_journey():
    """Simule un parcours utilisateur complet"""
    with tracer.start_as_current_span("user_journey") as journey_span:
        user_id = f"user_{random.randint(1000, 9999)}"
        journey_span.set_attribute("user.id", user_id)
        journey_span.set_attribute("journey.type", "shopping")

        # Étape 1: Page d'accueil
        with tracer.start_as_current_span("visit_homepage"):
            make_request("/", delay=0.1)

        # Étape 2: Recherche produit
        with tracer.start_as_current_span("search_products") as search_span:
            search_term = random.choice(["laptop", "phone", "tablet"])
            search_span.set_attribute("search.term", search_term)
            make_request(f"/api/search?q={search_term}", delay=0.3)

        # Étape 3: Voir détails produit
        with tracer.start_as_current_span("view_product") as product_span:
            product_id = random.randint(1, 100)
            product_span.set_attribute("product.id", product_id)
            make_request(f"/api/products/{product_id}", delay=0.2)

        # Étape 4: Ajouter au panier (50% de chance)
        if random.random() > 0.5:
            with tracer.start_as_current_span("add_to_cart") as cart_span:
                cart_span.set_attribute("product.id", product_id)
                cart_span.set_attribute("quantity", random.randint(1, 3))
                make_request(f"/api/cart/add", delay=0.15)

            # Étape 5: Checkout (30% de chance)
            if random.random() > 0.7:
                with tracer.start_as_current_span("checkout") as checkout_span:
                    checkout_span.set_attribute("payment.method", "credit_card")
                    status = make_request("/api/checkout", delay=0.5, error_rate=0.05)
                    if status == 200:
                        journey_span.set_attribute("journey.completed", True)
                        journey_span.set_attribute("journey.revenue", random.uniform(50, 500))

def generate_load(duration=60, rps=10, enable_journeys=True):
    """
    Génère de la charge pendant duration secondes

    Args:
        duration: Durée en secondes
        rps: Requêtes par seconde
        enable_journeys: Active la simulation de parcours utilisateur
    """
    endpoints = [
        '/api/products',
        '/api/users',
        '/api/orders',
        '/api/inventory',
        '/health',
        '/metrics'
    ]

    # Définir différents profils de charge
    load_profiles = {
        'normal': {'error_rate': 0.01, 'slow_rate': 0.05},
        'degraded': {'error_rate': 0.05, 'slow_rate': 0.20},
        'failure': {'error_rate': 0.30, 'slow_rate': 0.50}
    }

    start_time = time.time()
    request_count = 0
    error_count = 0
    journey_count = 0

    print(f"Starting load test: {rps} rps for {duration}s")
    print(f"User journeys: {'Enabled' if enable_journeys else 'Disabled'}")

    with ThreadPoolExecutor(max_workers=20) as executor:
        while time.time() - start_time < duration:
            # Changer le profil de charge dynamiquement
            elapsed = time.time() - start_time
            if elapsed < duration * 0.7:
                profile = load_profiles['normal']
            elif elapsed < duration * 0.9:
                profile = load_profiles['degraded']
            else:
                profile = load_profiles['failure']

            # Générer des requêtes individuelles
            futures = []
            for _ in range(int(rps * 0.8)):  # 80% requêtes simples
                endpoint = random.choice(endpoints)
                delay = random.uniform(0.5, 2.0) if random.random() < profile['slow_rate'] else None
                future = executor.submit(make_request, endpoint, delay, profile['error_rate'])
                futures.append(future)

            # Générer des parcours utilisateur
            if enable_journeys and random.random() < 0.2:  # 20% parcours
                future = executor.submit(simulate_user_journey)
                futures.append(future)
                journey_count += 1

            # Collecter les résultats
            for future in futures:
                try:
                    status = future.result(timeout=5)
                    request_count += 1
                    if status >= 400:
                        error_count += 1
                except Exception:
                    error_count += 1
                    request_count += 1

            # Afficher les statistiques périodiquement
            if request_count % 100 == 0:
                error_rate = (error_count / request_count * 100) if request_count > 0 else 0
                print(f"Progress: {request_count} requests, {error_rate:.2f}% errors, {journey_count} journeys")

            time.sleep(1.0 / rps)

    # Résumé final
    print("\n" + "="*50)
    print("Load Test Summary")
    print("="*50)
    print(f"Total requests: {request_count}")
    print(f"Total errors: {error_count} ({error_count/request_count*100:.2f}%)")
    print(f"User journeys: {journey_count}")
    print(f"Duration: {time.time() - start_time:.2f}s")
    print(f"Actual RPS: {request_count / (time.time() - start_time):.2f}")

if __name__ == "__main__":
    import argparse

    parser = argparse.ArgumentParser(description='Generate traced load for testing')
    parser.add_argument('--duration', type=int, default=300, help='Test duration in seconds')
    parser.add_argument('--rps', type=int, default=20, help='Requests per second')
    parser.add_argument('--no-journeys', action='store_true', help='Disable user journey simulation')

    args = parser.parse_args()

    generate_load(
        duration=args.duration,
        rps=args.rps,
        enable_journeys=not args.no_journeys
    )
```

### Export et analyse de traces

```bash
#!/bin/bash
# export-traces.sh

# Configuration
JAEGER_URL="${JAEGER_URL:-http://localhost:16686}"
SERVICE="${1:-all}"
LOOKBACK="${2:-1h}"
LIMIT="${3:-1000}"
OUTPUT_DIR="traces-export-$(date +%Y%m%d-%H%M%S)"

# Créer le répertoire de sortie
mkdir -p "$OUTPUT_DIR"

echo "Exporting traces from Jaeger..."
echo "Service: $SERVICE"
echo "Lookback: $LOOKBACK"
echo "Limit: $LIMIT"
echo "Output: $OUTPUT_DIR"
echo "="*50

# Fonction pour exporter les traces d'un service
export_service_traces() {
    local service=$1
    local output_file="$OUTPUT_DIR/${service}-traces.json"

    echo "Exporting traces for service: $service"

    # Récupérer les traces
    if [ "$service" == "all" ]; then
        curl -s "${JAEGER_URL}/api/traces?lookback=${LOOKBACK}&limit=${LIMIT}" > "$output_file"
    else
        curl -s "${JAEGER_URL}/api/traces?service=${service}&lookback=${LOOKBACK}&limit=${LIMIT}" > "$output_file"
    fi

    # Extraire les statistiques
    local trace_count=$(jq '.data | length' "$output_file")
    echo "  Traces exported: $trace_count"

    # Analyser les traces
    jq -r '.data[] | {
        traceID: .traceID,
        duration: (.spans[0].duration / 1000),
        spanCount: (.spans | length),
        services: ([.processes[].serviceName] | unique | join(",")),
        errors: ([.spans[] | select(.tags[]? | select(.key == "error" and .value == true))] | length),
        startTime: (.spans[0].startTime / 1000000 | strftime("%Y-%m-%d %H:%M:%S"))
    }' "$output_file" > "$OUTPUT_DIR/${service}-analysis.json"

    # Créer un résumé CSV
    echo "traceID,duration_ms,span_count,services,error_count,start_time" > "$OUTPUT_DIR/${service}-summary.csv"
    jq -r '.data[] |
        (.traceID) + "," +
        (.spans[0].duration / 1000 | tostring) + "," +
        (.spans | length | tostring) + "," +
        ([.processes[].serviceName] | unique | join("|")) + "," +
        ([.spans[] | select(.tags[]? | select(.key == "error" and .value == true))] | length | tostring) + "," +
        (.spans[0].startTime / 1000000 | strftime("%Y-%m-%d %H:%M:%S"))
    ' "$output_file" >> "$OUTPUT_DIR/${service}-summary.csv"
}

# Exporter les traces
if [ "$SERVICE" == "all" ]; then
    # Récupérer la liste des services
    services=$(curl -s "${JAEGER_URL}/api/services" | jq -r '.data[]')
    for svc in $services; do
        export_service_traces "$svc"
    done
else
    export_service_traces "$SERVICE"
fi

# Générer un rapport global
echo -e "\nGenerating global analysis report..."

cat > "$OUTPUT_DIR/analysis-report.md" << EOF
# Jaeger Traces Analysis Report
Generated: $(date)

## Configuration
- Jaeger URL: $JAEGER_URL
- Service: $SERVICE
- Lookback: $LOOKBACK
- Limit: $LIMIT

## Summary Statistics
EOF

# Analyser toutes les traces exportées
for json_file in "$OUTPUT_DIR"/*-traces.json; do
    if [ -f "$json_file" ]; then
        service_name=$(basename "$json_file" -traces.json)

        # Calculer les statistiques
        stats=$(jq -s '
            . as $traces |
            {
                total_traces: ($traces[0].data | length),
                avg_duration: ([$traces[0].data[].spans[0].duration] | add / length / 1000),
                max_duration: ([$traces[0].data[].spans[0].duration] | max / 1000),
                min_duration: ([$traces[0].data[].spans[0].duration] | min / 1000),
                avg_spans: ([$traces[0].data[].spans | length] | add / length),
                error_traces: ([$traces[0].data[] | select(.spans[].tags[]? | select(.key == "error" and .value == true))] | length),
                unique_operations: ([$traces[0].data[].spans[].operationName] | unique | length)
            }
        ' "$json_file")

        cat >> "$OUTPUT_DIR/analysis-report.md" << EOF

### Service: $service_name
- Total Traces: $(echo $stats | jq '.total_traces')
- Average Duration: $(echo $stats | jq '.avg_duration')ms
- Max Duration: $(echo $stats | jq '.max_duration')ms
- Min Duration: $(echo $stats | jq '.min_duration')ms
- Average Spans per Trace: $(echo $stats | jq '.avg_spans')
- Traces with Errors: $(echo $stats | jq '.error_traces')
- Unique Operations: $(echo $stats | jq '.unique_operations')

EOF
    fi
done

# Créer un script Python pour analyse avancée
cat > "$OUTPUT_DIR/analyze_traces.py" << 'EOF'
#!/usr/bin/env python3
import json
import pandas as pd
import matplotlib.pyplot as plt
from pathlib import Path
import sys

def analyze_traces(traces_dir):
    """Analyse avancée des traces exportées"""
    traces_dir = Path(traces_dir)

    # Charger tous les fichiers de traces
    all_traces = []
    for trace_file in traces_dir.glob("*-traces.json"):
        with open(trace_file) as f:
            data = json.load(f)
            for trace in data.get('data', []):
                all_traces.append({
                    'trace_id': trace['traceID'],
                    'duration_ms': trace['spans'][0]['duration'] / 1000,
                    'span_count': len(trace['spans']),
                    'service': list(trace['processes'].values())[0]['serviceName'],
                    'has_error': any(
                        tag.get('key') == 'error' and tag.get('value') == True
                        for span in trace['spans']
                        for tag in span.get('tags', [])
                    )
                })

    if not all_traces:
        print("No traces found!")
        return

    # Créer un DataFrame
    df = pd.DataFrame(all_traces)

    # Statistiques par service
    print("\n=== Statistics by Service ===")
    service_stats = df.groupby('service').agg({
        'duration_ms': ['mean', 'median', 'std', 'min', 'max'],
        'span_count': ['mean', 'median'],
        'has_error': 'sum',
        'trace_id': 'count'
    }).round(2)
    print(service_stats)

    # Identifier les traces lentes (>p95)
    p95_duration = df['duration_ms'].quantile(0.95)
    slow_traces = df[df['duration_ms'] > p95_duration]
    print(f"\n=== Slow Traces (>{p95_duration:.2f}ms) ===")
    print(slow_traces[['trace_id', 'service', 'duration_ms', 'span_count']].head(10))

    # Créer des visualisations
    fig, axes = plt.subplots(2, 2, figsize=(12, 10))

    # Distribution des durées
    axes[0, 0].hist(df['duration_ms'], bins=50, edgecolor='black')
    axes[0, 0].axvline(df['duration_ms'].median(), color='red', linestyle='--', label='Median')
    axes[0, 0].axvline(p95_duration, color='orange', linestyle='--', label='P95')
    axes[0, 0].set_xlabel('Duration (ms)')
    axes[0, 0].set_ylabel('Count')
    axes[0, 0].set_title('Duration Distribution')
    axes[0, 0].legend()

    # Durée par service
    df.boxplot(column='duration_ms', by='service', ax=axes[0, 1])
    axes[0, 1].set_xlabel('Service')
    axes[0, 1].set_ylabel('Duration (ms)')
    axes[0, 1].set_title('Duration by Service')

    # Taux d'erreur par service
    error_rate = df.groupby('service')['has_error'].mean() * 100
    error_rate.plot(kind='bar', ax=axes[1, 0])
    axes[1, 0].set_xlabel('Service')
    axes[1, 0].set_ylabel('Error Rate (%)')
    axes[1, 0].set_title('Error Rate by Service')
    axes[1, 0].tick_params(axis='x', rotation=45)

    # Corrélation durée vs nombre de spans
    axes[1, 1].scatter(df['span_count'], df['duration_ms'], alpha=0.5)
    axes[1, 1].set_xlabel('Span Count')
    axes[1, 1].set_ylabel('Duration (ms)')
    axes[1, 1].set_title('Duration vs Span Count')

    plt.tight_layout()
    plt.savefig(traces_dir / 'trace_analysis.png')
    print(f"\nVisualization saved to {traces_dir / 'trace_analysis.png'}")

    # Sauvegarder les statistiques
    df.describe().to_csv(traces_dir / 'statistics.csv')
    print(f"Statistics saved to {traces_dir / 'statistics.csv'}")

if __name__ == "__main__":
    if len(sys.argv) > 1:
        analyze_traces(sys.argv[1])
    else:
        analyze_traces(".")
EOF

chmod +x "$OUTPUT_DIR/analyze_traces.py"

echo -e "\n=== Export Complete ==="
echo "Output directory: $OUTPUT_DIR"
echo "Files generated:"
ls -la "$OUTPUT_DIR"
echo -e "\nRun the following for advanced analysis:"
echo "  cd $OUTPUT_DIR && python3 analyze_traces.py"
```

### Diagnostic et validation du tracing

```python
#!/usr/bin/env python3
# validate-tracing.py

import requests
import time
import json
from datetime import datetime, timedelta
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.jaeger import JaegerExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor

class TracingValidator:
    """Valide que le tracing fonctionne correctement"""

    def __init__(self, jaeger_url="http://localhost:16686", jaeger_agent="localhost:6831"):
        self.jaeger_url = jaeger_url
        self.jaeger_agent = jaeger_agent
        self.test_service = "tracing-validator"
        self.test_trace_id = None

        # Configurer le tracer pour les tests
        trace.set_tracer_provider(TracerProvider())
        self.tracer = trace.get_tracer(self.test_service)

        host, port = jaeger_agent.split(':')
        jaeger_exporter = JaegerExporter(
            agent_host_name=host,
            agent_port=int(port),
        )
        span_processor = BatchSpanProcessor(jaeger_exporter)
        trace.get_tracer_provider().add_span_processor(span_processor)

    def test_span_creation(self):
        """Test 1: Création et envoi de spans"""
        print("Test 1: Creating and sending test spans...")

        with self.tracer.start_as_current_span("test_root_span") as span:
            span.set_attribute("test.type", "validation")
            span.set_attribute("test.timestamp", datetime.now().isoformat())
            self.test_trace_id = format(span.get_span_context().trace_id, '032x')

            # Créer des spans enfants
            with self.tracer.start_as_current_span("test_child_1"):
                time.sleep(0.1)

            with self.tracer.start_as_current_span("test_child_2"):
                time.sleep(0.05)

                with self.tracer.start_as_current_span("test_grandchild"):
                    time.sleep(0.02)

        print(f"  ✓ Test trace created with ID: {self.test_trace_id}")
        return True

    def test_jaeger_api(self):
        """Test 2: API Jaeger accessible"""
        print("Test 2: Checking Jaeger API...")

        try:
            # Test health endpoint
            response = requests.get(f"{self.jaeger_url}/")
            if response.status_code != 200:
                print(f"  ✗ Jaeger UI not accessible: {response.status_code}")
                return False

            # Test services endpoint
            response = requests.get(f"{self.jaeger_url}/api/services")
            if response.status_code != 200:
                print(f"  ✗ Jaeger API not accessible: {response.status_code}")
                return False

            services = response.json().get('data', [])
            print(f"  ✓ Jaeger API accessible. Found {len(services)} services")
            return True

        except Exception as e:
            print(f"  ✗ Error accessing Jaeger: {e}")
            return False

    def test_trace_retrieval(self):
        """Test 3: Récupération de la trace de test"""
        print("Test 3: Retrieving test trace...")

        # Attendre que la trace soit indexée
        time.sleep(5)

        try:
            # Rechercher la trace par service
            end_time = int(time.time() * 1000000)
            start_time = end_time - (60 * 1000000)  # 1 minute

            params = {
                'service': self.test_service,
                'start': start_time,
                'end': end_time,
                'limit': 10
            }

            response = requests.get(f"{self.jaeger_url}/api/traces", params=params)
            traces = response.json().get('data', [])

            # Chercher notre trace de test
            test_trace = None
            for trace in traces:
                if trace['traceID'] == self.test_trace_id:
                    test_trace = trace
                    break

            if test_trace:
                span_count = len(test_trace['spans'])
                print(f"  ✓ Test trace found with {span_count} spans")
                return True
            else:
                print(f"  ✗ Test trace not found (ID: {self.test_trace_id})")
                return False

        except Exception as e:
            print(f"  ✗ Error retrieving trace: {e}")
            return False

    def test_service_dependencies(self):
        """Test 4: Dépendances entre services"""
        print("Test 4: Checking service dependencies...")

        try:
            response = requests.get(f"{self.jaeger_url}/api/dependencies",
                                   params={'endTs': int(time.time() * 1000)})

            if response.status_code != 200:
                print(f"  ✗ Dependencies endpoint error: {response.status_code}")
                return False

            dependencies = response.json()
            print(f"  ✓ Found {len(dependencies)} service dependencies")
            return True

        except Exception as e:
            print(f"  ✗ Error checking dependencies: {e}")
            return False

    def test_sampling(self):
        """Test 5: Configuration du sampling"""
        print("Test 5: Checking sampling configuration...")

        try:
            # Envoyer 100 traces rapides
            trace_ids = []
            for i in range(100):
                with self.tracer.start_as_current_span(f"sampling_test_{i}") as span:
                    span.set_attribute("test.sampling", True)
                    trace_ids.append(format(span.get_span_context().trace_id, '032x'))

            # Attendre l'indexation
            time.sleep(5)

            # Vérifier combien ont été échantillonnées
            params = {
                'service': self.test_service,
                'operation': 'sampling_test',
                'limit': 200
            }

            response = requests.get(f"{self.jaeger_url}/api/traces", params=params)
            traces = response.json().get('data', [])

            sampled_count = len([t for t in traces if t['traceID'] in trace_ids])
            sampling_rate = sampled_count / 100

            print(f"  ✓ Sampling rate: {sampling_rate*100:.1f}% ({sampled_count}/100 traces)")
            return True

        except Exception as e:
            print(f"  ✗ Error testing sampling: {e}")
            return False

    def run_all_tests(self):
        """Exécute tous les tests de validation"""
        print("="*50)
        print("Jaeger Tracing Validation")
        print("="*50)
        print(f"Jaeger URL: {self.jaeger_url}")
        print(f"Jaeger Agent: {self.jaeger_agent}")
        print()

        tests = [
            self.test_jaeger_api,
            self.test_span_creation,
            self.test_trace_retrieval,
            self.test_service_dependencies,
            self.test_sampling
        ]

        results = []
        for test in tests:
            try:
                result = test()
                results.append(result)
            except Exception as e:
                print(f"  ✗ Test failed with exception: {e}")
                results.append(False)
            print()

        # Résumé
        print("="*50)
        print("Validation Summary")
        print("="*50)
        passed = sum(results)
        total = len(results)

        if passed == total:
            print(f"✅ All tests passed ({passed}/{total})")
        else:
            print(f"⚠️  Some tests failed ({passed}/{total} passed)")

        return passed == total

if __name__ == "__main__":
    import argparse

    parser = argparse.ArgumentParser(description='Validate Jaeger tracing setup')
    parser.add_argument('--jaeger-url', default='http://localhost:16686',
                       help='Jaeger UI URL')
    parser.add_argument('--jaeger-agent', default='localhost:6831',
                       help='Jaeger agent host:port')

    args = parser.parse_args()

    validator = TracingValidator(
        jaeger_url=args.jaeger_url,
        jaeger_agent=args.jaeger_agent
    )

    success = validator.run_all_tests()
    exit(0 if success else 1)
```

## Bonnes pratiques

### Nommage des spans

```python
# ❌ MAUVAIS : Noms génériques ou non descriptifs
with tracer.start_as_current_span("process"):
    with tracer.start_as_current_span("query"):
        with tracer.start_as_current_span("transform"):
            pass

# ✅ BON : Noms spécifiques et hiérarchiques
with tracer.start_as_current_span("order_processing"):
    with tracer.start_as_current_span("validate_order_items"):
        pass
    with tracer.start_as_current_span("check_inventory_availability"):
        pass
    with tracer.start_as_current_span("calculate_total_price"):
        with tracer.start_as_current_span("apply_discounts"):
            pass
        with tracer.start_as_current_span("calculate_tax"):
            pass

# Convention de nommage recommandée
# Format: <resource>.<action>
# Exemples:
# - database.query
# - cache.get
# - http.request
# - queue.publish
# - payment.process
```

### Attributs essentiels

```python
# Attributs standards (semantic conventions)
ESSENTIAL_ATTRIBUTES = {
    # Service
    "service.name": "api-server",
    "service.version": "1.2.3",
    "service.namespace": "production",
    "deployment.environment": "lab",

    # HTTP
    "http.method": "GET",
    "http.url": "https://api.example.com/users/123",
    "http.target": "/users/123",
    "http.host": "api.example.com",
    "http.scheme": "https",
    "http.status_code": 200,
    "http.response_content_length": 1024,
    "http.user_agent": "Mozilla/5.0...",

    # Database
    "db.system": "postgresql",
    "db.connection_string": "postgresql://localhost:5432/mydb",
    "db.name": "users",
    "db.statement": "SELECT * FROM users WHERE id = ?",
    "db.operation": "SELECT",
    "db.user": "app_user",

    # Messaging
    "messaging.system": "rabbitmq",
    "messaging.destination": "orders",
    "messaging.destination_kind": "queue",
    "messaging.operation": "publish",
    "messaging.message_id": "msg-123",

    # RPC
    "rpc.system": "grpc",
    "rpc.service": "UserService",
    "rpc.method": "GetUser",
    "rpc.grpc.status_code": 0,

    # Erreur
    "error": True,
    "error.type": "ValueError",
    "error.message": "Invalid user ID",
    "error.stack": "Traceback...",

    # Custom business
    "user.id": "user-123",
    "order.id": "order-456",
    "product.sku": "SKU-789",
    "payment.amount": 99.99,
    "payment.currency": "USD"
}

# Fonction helper pour ajouter les attributs standards
def add_standard_attributes(span, request=None, response=None, error=None):
    """Ajoute les attributs standards à un span"""

    # Attributs de base
    span.set_attribute("service.name", os.getenv("SERVICE_NAME", "unknown"))
    span.set_attribute("service.version", os.getenv("SERVICE_VERSION", "unknown"))
    span.set_attribute("deployment.environment", os.getenv("ENVIRONMENT", "development"))
    span.set_attribute("host.name", socket.gethostname())

    # Attributs HTTP si disponibles
    if request:
        span.set_attribute("http.method", request.method)
        span.set_attribute("http.url", request.url)
        span.set_attribute("http.target", request.path)
        span.set_attribute("http.host", request.host)
        span.set_attribute("http.scheme", request.scheme)
        span.set_attribute("http.user_agent", request.headers.get("User-Agent", ""))

        # Headers personnalisés importants
        if "X-Request-ID" in request.headers:
            span.set_attribute("http.request_id", request.headers["X-Request-ID"])
        if "X-User-ID" in request.headers:
            span.set_attribute("user.id", request.headers["X-User-ID"])

    # Attributs de réponse
    if response:
        span.set_attribute("http.status_code", response.status_code)
        if hasattr(response, 'content_length'):
            span.set_attribute("http.response_content_length", response.content_length)

    # Gestion des erreurs
    if error:
        span.set_attribute("error", True)
        span.set_attribute("error.type", type(error).__name__)
        span.set_attribute("error.message", str(error))
        span.record_exception(error)
        span.set_status(Status(StatusCode.ERROR, str(error)))
```

### Gestion des erreurs et exceptions

```python
from opentelemetry.trace import Status, StatusCode
import traceback

class TracedError(Exception):
    """Exception personnalisée avec contexte de trace"""
    def __init__(self, message, span=None, **attributes):
        super().__init__(message)
        self.span = span
        self.attributes = attributes

        if span:
            span.record_exception(self)
            span.set_status(Status(StatusCode.ERROR, message))
            for key, value in attributes.items():
                span.set_attribute(f"error.context.{key}", value)

# Pattern de gestion d'erreur avec retry
def traced_operation_with_retry(operation_name, max_retries=3):
    """Décorateur pour tracer une opération avec retry"""
    def decorator(func):
        def wrapper(*args, **kwargs):
            with tracer.start_as_current_span(operation_name) as span:
                span.set_attribute("retry.max_attempts", max_retries)

                for attempt in range(max_retries):
                    try:
                        with tracer.start_as_current_span(f"attempt_{attempt + 1}") as attempt_span:
                            attempt_span.set_attribute("retry.attempt", attempt + 1)
                            result = func(*args, **kwargs)

                            if attempt > 0:
                                span.set_attribute("retry.succeeded_after", attempt + 1)

                            return result

                    except Exception as e:
                        attempt_span.record_exception(e)
                        attempt_span.set_status(Status(StatusCode.ERROR))

                        if attempt == max_retries - 1:
                            # Dernière tentative échouée
                            span.set_attribute("retry.exhausted", True)
                            span.set_attribute("error", True)
                            span.record_exception(e)
                            span.set_status(Status(StatusCode.ERROR, f"Failed after {max_retries} attempts"))
                            raise TracedError(f"Operation failed after {max_retries} attempts", span=span)
                        else:
                            # Attendre avant de réessayer
                            wait_time = (2 ** attempt) * 0.1  # Backoff exponentiel
                            span.add_event(f"Retry {attempt + 1} failed, waiting {wait_time}s")
                            time.sleep(wait_time)

        return wrapper
    return decorator

# Utilisation
@traced_operation_with_retry("database_query", max_retries=3)
def query_database(query):
    # Simulation d'une requête qui peut échouer
    if random.random() < 0.3:  # 30% de chance d'échec
        raise ConnectionError("Database connection lost")
    return "query_result"
```

### Patterns de tracing pour différents scénarios

```python
# 1. Pattern pour API REST
class TracedAPIHandler:
    def __init__(self, tracer):
        self.tracer = tracer

    def handle_request(self, request):
        # Extraire le contexte de trace depuis les headers
        context = extract(request.headers)

        with self.tracer.start_as_current_span(
            f"{request.method} {request.path}",
            context=context
        ) as span:
            try:
                # Validation
                with self.tracer.start_as_current_span("validate_request"):
                    self.validate(request)

                # Authentification
                with self.tracer.start_as_current_span("authenticate") as auth_span:
                    user = self.authenticate(request)
                    auth_span.set_attribute("user.id", user.id)
                    auth_span.set_attribute("user.role", user.role)

                # Autorisation
                with self.tracer.start_as_current_span("authorize"):
                    self.authorize(user, request.resource)

                # Traitement métier
                with self.tracer.start_as_current_span("process_business_logic"):
                    result = self.process(request, user)

                # Réponse
                response = self.create_response(result)
                span.set_attribute("http.status_code", response.status_code)

                return response

            except ValidationError as e:
                span.set_attribute("error.type", "validation")
                return self.error_response(400, str(e))
            except AuthenticationError as e:
                span.set_attribute("error.type", "authentication")
                return self.error_response(401, str(e))
            except AuthorizationError as e:
                span.set_attribute("error.type", "authorization")
                return self.error_response(403, str(e))
            except Exception as e:
                span.record_exception(e)
                span.set_status(Status(StatusCode.ERROR))
                return self.error_response(500, "Internal server error")

# 2. Pattern pour traitement asynchrone
async def traced_async_processing(items):
    """Pattern pour tracer le traitement asynchrone par batch"""
    with tracer.start_as_current_span("batch_processing") as batch_span:
        batch_span.set_attribute("batch.size", len(items))
        batch_span.set_attribute("batch.id", str(uuid.uuid4()))

        results = []
        errors = []

        # Traitement parallèle avec tracing
        async def process_item(item, index):
            with tracer.start_as_current_span(f"process_item_{index}") as item_span:
                item_span.set_attribute("item.id", item.id)
                item_span.set_attribute("item.type", item.type)

                try:
                    result = await async_process(item)
                    item_span.set_attribute("item.status", "success")
                    return result
                except Exception as e:
                    item_span.record_exception(e)
                    item_span.set_attribute("item.status", "failed")
                    errors.append((item.id, str(e)))
                    return None

        # Exécuter en parallèle
        tasks = [process_item(item, i) for i, item in enumerate(items)]
        results = await asyncio.gather(*tasks, return_exceptions=True)

        # Résumé du batch
        successful = sum(1 for r in results if r is not None and not isinstance(r, Exception))
        batch_span.set_attribute("batch.successful", successful)
        batch_span.set_attribute("batch.failed", len(errors))
        batch_span.set_attribute("batch.success_rate", successful / len(items))

        if errors:
            batch_span.set_attribute("error", True)
            batch_span.add_event("Batch completed with errors", {
                "error_count": len(errors),
                "error_ids": [e[0] for e in errors[:10]]  # Limiter à 10 pour éviter spam
            })

        return results, errors

# 3. Pattern pour circuit breaker
class TracedCircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60, tracer=None):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = "closed"  # closed, open, half-open
        self.tracer = tracer or trace.get_tracer(__name__)

    def call(self, func, *args, **kwargs):
        with self.tracer.start_as_current_span("circuit_breaker") as span:
            span.set_attribute("circuit_breaker.state", self.state)
            span.set_attribute("circuit_breaker.failures", self.failures)

            # Circuit ouvert
            if self.state == "open":
                if time.time() - self.last_failure_time > self.timeout:
                    self.state = "half-open"
                    span.add_event("Circuit breaker entering half-open state")
                else:
                    span.set_attribute("circuit_breaker.rejected", True)
                    raise CircuitOpenError("Circuit breaker is open")

            # Tentative d'appel
            try:
                with self.tracer.start_as_current_span("protected_call"):
                    result = func(*args, **kwargs)

                # Succès - réinitialiser si nécessaire
                if self.state == "half-open":
                    self.state = "closed"
                    self.failures = 0
                    span.add_event("Circuit breaker reset to closed")

                return result

            except Exception as e:
                self.failures += 1
                self.last_failure_time = time.time()

                span.record_exception(e)
                span.set_attribute("circuit_breaker.failure_count", self.failures)

                if self.failures >= self.failure_threshold:
                    self.state = "open"
                    span.add_event("Circuit breaker opened", {
                        "threshold": self.failure_threshold,
                        "failures": self.failures
                    })

                raise
```

### Optimisation des performances du tracing

```python
# Sampling dynamique basé sur les conditions
class DynamicSampler:
    """Sampler qui ajuste le taux d'échantillonnage dynamiquement"""

    def __init__(self, base_rate=0.1):
        self.base_rate = base_rate
        self.error_rate = 1.0  # Toujours échantillonner les erreurs
        self.slow_threshold = 1.0  # secondes
        self.vip_users = set()  # Users à toujours tracer

    def should_sample(self, span_context, attributes):
        # Toujours échantillonner les erreurs
        if attributes.get("error") == True:
            return True

        # Toujours échantillonner les VIP users
        if attributes.get("user.id") in self.vip_users:
            return True

        # Échantillonner les requêtes lentes
        if attributes.get("duration.estimated", 0) > self.slow_threshold:
            return True

        # Échantillonner les opérations critiques
        if attributes.get("operation.critical") == True:
            return True

        # Sampling de base probabiliste
        return random.random() < self.base_rate

# Réduction de la cardinalité des attributs
def sanitize_attributes(attributes):
    """Nettoie les attributs pour réduire la cardinalité"""
    sanitized = {}

    for key, value in attributes.items():
        # Limiter la longueur des strings
        if isinstance(value, str) and len(value) > 100:
            sanitized[key] = value[:100] + "..."

        # Masquer les données sensibles
        elif key in ["password", "token", "api_key", "secret"]:
            sanitized[key] = "***REDACTED***"

        # Arrondir les timestamps pour réduire la cardinalité
        elif key.endswith("_timestamp"):
            sanitized[key] = int(value / 60) * 60  # Arrondir à la minute

        # Grouper les IDs utilisateur
        elif key == "user.id":
            # Hasher l'ID pour la privacy tout en gardant la cohérence
            sanitized[key] = hashlib.md5(str(value).encode()).hexdigest()[:8]

        else:
            sanitized[key] = value

    return sanitized

# Batch processing des spans
class BatchSpanProcessor:
    """Processeur qui groupe les spans avant l'export"""

    def __init__(self, exporter, max_batch_size=100, max_delay=5.0):
        self.exporter = exporter
        self.max_batch_size = max_batch_size
        self.max_delay = max_delay
        self.batch = []
        self.last_export = time.time()
        self.lock = threading.Lock()

        # Démarrer le thread d'export
        self.export_thread = threading.Thread(target=self._export_loop, daemon=True)
        self.export_thread.start()

    def on_end(self, span):
        """Appelé quand un span se termine"""
        with self.lock:
            self.batch.append(span)

            if len(self.batch) >= self.max_batch_size:
                self._export_batch()

    def _export_loop(self):
        """Thread qui exporte périodiquement les spans"""
        while True:
            time.sleep(1)
            with self.lock:
                if self.batch and time.time() - self.last_export > self.max_delay:
                    self._export_batch()

    def _export_batch(self):
        """Exporte le batch actuel"""
        if not self.batch:
            return

        try:
            self.exporter.export(self.batch)
            self.batch = []
            self.last_export = time.time()
        except Exception as e:
            print(f"Failed to export spans: {e}")
```

## Intégration complète dans une application

### Exemple d'application complète avec tracing

```python
# app.py - Application Flask complète avec tracing
from flask import Flask, request, jsonify
import redis
import psycopg2
from opentelemetry import trace, baggage, metrics
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor
from opentelemetry.instrumentation.psycopg2 import Psycopg2Instrumentor
from opentelemetry.exporter.jaeger import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource
from prometheus_client import Counter, Histogram, generate_latest
import logging
import time

# Configuration du logging avec trace context
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - [%(trace_id)s] - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# Configuration OpenTelemetry
resource = Resource.create({
    "service.name": "complete-app",
    "service.version": "1.0.0",
    "deployment.environment": "lab"
})

provider = TracerProvider(resource=resource)
trace.set_tracer_provider(provider)

# Configuration Jaeger
jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger-agent.tracing.svc.cluster.local",
    agent_port=6831,
)
span_processor = BatchSpanProcessor(jaeger_exporter)
provider.add_span_processor(span_processor)

# Tracer pour l'application
tracer = trace.get_tracer(__name__)

# Métriques Prometheus
request_count = Counter('app_requests_total', 'Total requests', ['method', 'endpoint', 'status'])
request_duration = Histogram('app_request_duration_seconds', 'Request duration', ['method', 'endpoint'])
cache_hits = Counter('app_cache_hits_total', 'Cache hits')
cache_misses = Counter('app_cache_misses_total', 'Cache misses')
db_queries = Counter('app_db_queries_total', 'Database queries', ['operation'])

# Application Flask
app = Flask(__name__)

# Auto-instrumentation
FlaskInstrumentor().instrument_app(app)
RedisInstrumentor().instrument()
Psycopg2Instrumentor().instrument()

# Connexions
redis_client = redis.Redis(host='redis', port=6379, decode_responses=True)
db_conn = psycopg2.connect(
    host='postgres',
    database='appdb',
    user='appuser',
    password='apppass'
)

# Middleware pour ajouter le trace context aux logs
@app.before_request
def before_request():
    span = trace.get_current_span()
    if span:
        span_context = span.get_span_context()
        logger = logging.LoggerAdapter(
            logging.getLogger(__name__),
            {'trace_id': format(span_context.trace_id, '032x')}
        )
        request.logger = logger
    else:
        request.logger = logger

# Routes de l'application
@app.route('/api/users/<int:user_id>')
def get_user(user_id):
    """Récupère un utilisateur avec cache et DB"""
    with tracer.start_as_current_span("get_user") as span:
        span.set_attribute("user.id", user_id)

        # Vérifier le cache
        with tracer.start_as_current_span("cache_lookup") as cache_span:
            cache_key = f"user:{user_id}"
            cached_user = redis_client.get(cache_key)

            if cached_user:
                cache_hits.inc()
                cache_span.set_attribute("cache.hit", True)
                request.logger.info(f"Cache hit for user {user_id}")
                return jsonify(json.loads(cached_user))
            else:
                cache_misses.inc()
                cache_span.set_attribute("cache.hit", False)

        # Requête DB
        with tracer.start_as_current_span("database_query") as db_span:
            db_span.set_attribute("db.statement", "SELECT * FROM users WHERE id = %s")

            cursor = db_conn.cursor()
            cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
            user = cursor.fetchone()
            cursor.close()

            db_queries.labels(operation='SELECT').inc()

            if not user:
                span.set_attribute("user.found", False)
                return jsonify({"error": "User not found"}), 404

            user_dict = {
                "id": user[0],
                "name": user[1],
                "email": user[2]
            }

        # Mettre en cache
        with tracer.start_as_current_span("cache_set"):
            redis_client.setex(cache_key, 300, json.dumps(user_dict))

        span.set_attribute("user.found", True)
        request.logger.info(f"User {user_id} fetched from database")
        return jsonify(user_dict)

@app.route('/api/orders', methods=['POST'])
def create_order():
    """Crée une commande avec workflow complet"""
    with tracer.start_as_current_span("create_order") as span:
        order_data = request.json
        span.set_attribute("order.items", len(order_data.get('items', [])))

        try:
            # Validation
            with tracer.start_as_current_span("validate_order"):
                if not order_data.get('items'):
                    raise ValueError("Order must contain items")

            # Vérifier l'inventaire
            with tracer.start_as_current_span("check_inventory") as inv_span:
                for item in order_data['items']:
                    available = check_inventory(item['product_id'], item['quantity'])
                    if not available:
                        inv_span.set_attribute("inventory.available", False)
                        raise ValueError(f"Product {item['product_id']} not available")

            # Calculer le prix
            with tracer.start_as_current_span("calculate_price") as price_span:
                total = calculate_order_total(order_data['items'])
                price_span.set_attribute("order.total", total)

            # Traiter le paiement
            with tracer.start_as_current_span("process_payment") as payment_span:
                payment_result = process_payment(order_data['payment'], total)
                payment_span.set_attribute("payment.success", payment_result['success'])

                if not payment_result['success']:
                    raise ValueError("Payment failed")

            # Créer la commande en DB
            with tracer.start_as_current_span("save_order"):
                order_id = save_order_to_db(order_data, total, payment_result)

            span.set_attribute("order.id", order_id)
            span.set_attribute("order.success", True)

            return jsonify({
                "order_id": order_id,
                "total": total,
                "status": "created"
            }), 201

        except Exception as e:
            span.record_exception(e)
            span.set_attribute("order.success", False)
            request.logger.error(f"Order creation failed: {e}")
            return jsonify({"error": str(e)}), 400

@app.route('/metrics')
def metrics():
    """Endpoint pour Prometheus"""
    return generate_latest()

@app.route('/health')
def health():
    """Health check endpoint"""
    with tracer.start_as_current_span("health_check") as span:
        checks = {
            "database": check_database(),
            "redis": check_redis(),
            "disk_space": check_disk_space()
        }

        healthy = all(checks.values())
        span.set_attribute("health.status", "healthy" if healthy else "unhealthy")

        for check, status in checks.items():
            span.set_attribute(f"health.{check}", status)

        if healthy:
            return jsonify({"status": "healthy", "checks": checks})
        else:
            return jsonify({"status": "unhealthy", "checks": checks}), 503

# Fonctions helper
def check_inventory(product_id, quantity):
    """Vérifie la disponibilité en stock"""
    # Simulation
    return random.random() > 0.1

def calculate_order_total(items):
    """Calcule le total de la commande"""
    # Simulation
    return sum(item.get('price', 10) * item.get('quantity', 1) for item in items)

def process_payment(payment_info, amount):
    """Traite le paiement"""
    # Simulation
    return {"success": random.random() > 0.05, "transaction_id": str(uuid.uuid4())}

def save_order_to_db(order_data, total, payment_result):
    """Sauvegarde la commande en base"""
    # Simulation
    return str(uuid.uuid4())

def check_database():
    """Vérifie la connexion DB"""
    try:
        cursor = db_conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        return True
    except:
        return False

def check_redis():
    """Vérifie la connexion Redis"""
    try:
        redis_client.ping()
        return True
    except:
        return False

def check_disk_space():
    """Vérifie l'espace disque"""
    import shutil
    usage = shutil.disk_usage("/")
    return (usage.free / usage.total) > 0.1

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

## Conclusion

Le tracing distribué avec Jaeger complète votre stack d'observabilité dans MicroK8s. Vous disposez maintenant d'une vision complète de vos applications distribuées avec :

### Capacités acquises

1. **Visualisation end-to-end** : Chaque requête est tracée à travers tous les services
2. **Identification précise des bottlenecks** : Les spans montrent exactement où le temps est dépensé
3. **Corrélation complète** : Traces, logs et métriques travaillent ensemble
4. **Debugging efficace** : Du symptôme à la cause racine en quelques clics
5. **Monitoring proactif** : SLO basés sur des données réelles de performance

### Points clés à retenir

- **OpenTelemetry est le standard** : Utilisez-le pour une instrumentation pérenne
- **La propagation du contexte est cruciale** : Sans elle, pas de trace distribuée
- **Le sampling intelligent est nécessaire** : Équilibrez visibilité et performance
- **Les attributs font la différence** : Plus de contexte = debugging plus facile
- **L'intégration est la clé** : Traces + Logs + Métriques = Observabilité complète

### Architecture finale d'observabilité

Votre lab MicroK8s dispose maintenant de :

```
┌─────────────────────────────────────────────────┐
│                 Applications                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │   API    │  │ Frontend │  │ Workers  │      │
│  └──────────┘  └──────────┘  └──────────┘      │
│       │              │              │            │
│       └──────────────┴──────────────┘            │
│                      │                           │
│            OpenTelemetry SDK                     │
└─────────────────────┬───────────────────────────┘
                      │
        ┌─────────────┴─────────────┐
        │                           │
        ▼                           ▼
┌──────────────┐          ┌──────────────┐
│  Prometheus  │          │    Jaeger    │
│  (Métriques) │          │   (Traces)   │
└──────────────┘          └──────────────┘
        │                           │
        │      ┌──────────────┐     │
        │      │     Loki     │     │
        │      │    (Logs)    │     │
        │      └──────────────┘     │
        │              │            │
        └──────────────┴────────────┘
                      │
                      ▼
              ┌──────────────┐
              │   Grafana    │
              │ (Dashboards) │
              └──────────────┘
```

Avec cette infrastructure complète, vous êtes équipé pour comprendre, optimiser et dépanner n'importe quelle application distribuée dans votre lab MicroK8s. La vraie puissance de l'observabilité émerge quand vous utilisez ces trois piliers ensemble, naviguant naturellement entre métriques, logs et traces pour résoudre les problèmes complexes.

Dans la prochaine section, nous explorerons le monitoring synthétique avec Blackbox Exporter, ajoutant une dimension proactive à votre observabilité en simulant l'expérience utilisateur.

⏭️

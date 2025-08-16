🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.3 Correlation logs-métriques avec Loki

## Introduction : Pourquoi Loki change la donne

Imaginez cette situation : vous observez un pic de latence dans Grafana sur vos métriques Prometheus. Vous voulez immédiatement voir les logs correspondants pour comprendre ce qui s'est passé. Avec la stack ELK traditionnelle, vous devez ouvrir Kibana dans un autre onglet, reconfigurer la période temporelle, rechercher le bon service, et espérer retrouver les logs pertinents. Avec Loki, vous cliquez simplement sur le graphique dans Grafana et les logs apparaissent instantanément en dessous. Cette intégration native transforme radicalement votre workflow d'investigation.

### Loki vs Elasticsearch : une philosophie différente

Elasticsearch indexe chaque mot de vos logs, créant une base de données de recherche massive. C'est puissant mais gourmand en ressources. Loki adopte l'approche inverse : il n'indexe que les métadonnées (labels Kubernetes, timestamps) et compresse les logs eux-mêmes. Cette simplicité a des implications profondes :

**Stockage réduit de 10x** : Un gigaoctet de logs dans Elasticsearch devient 100MB dans Loki.

**RAM minimale** : Loki fonctionne confortablement avec 100MB de RAM contre plusieurs gigaoctets pour Elasticsearch.

**Coût opérationnel négligeable** : Pas de shards à gérer, pas de mapping complexe, pas de ré-indexation.

**Modèle de labels Prometheus** : Si vous connaissez Prometheus, vous connaissez déjà Loki.

### Le concept révolutionnaire : "Like Prometheus, but for logs"

Loki traite les logs comme Prometheus traite les métriques. Au lieu d'avoir une métrique `http_requests_total{service="api", status="200"}`, vous avez un flux de logs `{service="api", namespace="production"}`. Les mêmes labels, la même philosophie, le même langage de requête (LogQL est basé sur PromQL). Cette cohérence conceptuelle simplifie énormément l'apprentissage et l'utilisation.

## Architecture de Loki dans MicroK8s

### Composants principaux

**Promtail** : L'agent de collecte, équivalent à Fluent Bit mais optimisé pour Loki. Il s'exécute comme DaemonSet sur chaque node, découvre automatiquement les pods, extrait les labels Kubernetes, et envoie les logs à Loki.

**Loki** : Le serveur central qui reçoit, compresse, stocke et sert les logs. Il expose une API compatible avec Grafana et peut fonctionner en mode single-binary pour un lab.

**Grafana** : L'interface unifiée pour visualiser métriques (de Prometheus) et logs (de Loki) dans les mêmes dashboards.

### Flux de données

```
[Pods] → [stdout/stderr] → [/var/log/containers/*.log]
                                    ↓
                            [Promtail DaemonSet]
                                    ↓
                        [Extraction labels K8s]
                                    ↓
                            [Compression logs]
                                    ↓
                                [Loki]
                                    ↓
                    [Stockage chunks + index]
                                    ↓
                        [API Query (LogQL)]
                                    ↓
                              [Grafana]
```

### Stockage dans Loki

Loki sépare les données en deux parties :

**Index** : Petite base de données (BoltDB pour un lab) contenant les labels et timestamps. C'est la seule partie qui nécessite des performances de lecture rapide.

**Chunks** : Les logs compressés (gzip/snappy) stockés comme objets. Peuvent être sur filesystem local, S3, ou tout object storage compatible.

Cette séparation permet à Loki d'être si efficace : l'index reste petit et rapide, tandis que les chunks sont simplement des fichiers compressés.

## Installation de Loki sur MicroK8s

### Méthode 1 : Installation avec Helm (recommandée)

```bash
# Ajouter le repository Grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Créer le namespace
kubectl create namespace loki-stack

# Installer Loki avec Promtail
helm install loki grafana/loki-stack \
  --namespace loki-stack \
  --set loki.isDefault=false \
  --set promtail.enabled=true \
  --set grafana.enabled=false \
  --set prometheus.enabled=false \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=10Gi \
  --values loki-values.yaml
```

Créez un fichier `loki-values.yaml` optimisé pour un lab :

```yaml
# loki-values.yaml
loki:
  # Configuration pour single-node
  auth_enabled: false

  # Configuration serveur
  server:
    http_listen_port: 3100
    grpc_listen_port: 9096
    log_level: info

  # Limites adaptées au lab
  limits_config:
    enforce_metric_name: false
    reject_old_samples: true
    reject_old_samples_max_age: 168h
    ingestion_rate_mb: 10
    ingestion_burst_size_mb: 20
    max_entries_limit_per_query: 5000
    max_streams_per_user: 10000
    max_global_streams_per_user: 10000

  # Configuration du schéma
  schema_config:
    configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

  # Stockage local pour un lab
  storage_config:
    boltdb_shipper:
      active_index_directory: /loki/boltdb-shipper-active
      cache_location: /loki/boltdb-shipper-cache
      cache_ttl: 24h
      shared_store: filesystem
    filesystem:
      directory: /loki/chunks

  # Compacteur pour nettoyer les vieux logs
  compactor:
    working_directory: /loki/compactor
    shared_store: filesystem
    compaction_interval: 10m
    retention_enabled: true
    retention_delete_delay: 2h
    retention_delete_worker_count: 150

  # Rétention des logs (7 jours pour un lab)
  chunk_store_config:
    max_look_back_period: 168h

  table_manager:
    retention_deletes_enabled: true
    retention_period: 168h

promtail:
  # Configuration Promtail
  config:
    server:
      http_listen_port: 3101
      grpc_listen_port: 0

    positions:
      filename: /tmp/positions.yaml

    clients:
    - url: http://loki:3100/loki/api/v1/push

    scrape_configs:
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod

      relabel_configs:
      # Garder seulement les pods avec annotations de scraping
      - source_labels: [__meta_kubernetes_pod_annotation_promtail_io_scrape]
        action: keep
        regex: true

      # Extraire le namespace
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace

      # Extraire le nom du pod
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod

      # Extraire les labels du pod
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app

      # Extraire le container name
      - source_labels: [__meta_kubernetes_pod_container_name]
        target_label: container

      pipeline_stages:
      # Parser pour logs Docker/containerd
      - docker: {}

      # Parser pour logs JSON
      - json:
          expressions:
            level: level
            message: message
            timestamp: timestamp

      # Extraire le level si présent
      - labels:
          level:

      # Filtrer les logs de santé
      - drop:
          expression: '.*GET /health.*'
```

### Méthode 2 : Installation manuelle (contrôle total)

Pour un contrôle complet, déployez manuellement :

```yaml
# loki-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-config
  namespace: loki-stack
data:
  loki.yaml: |
    auth_enabled: false

    server:
      http_listen_port: 3100
      grpc_listen_port: 9096

    ingester:
      lifecycler:
        address: 127.0.0.1
        ring:
          kvstore:
            store: inmemory
          replication_factor: 1
      chunk_idle_period: 5m
      chunk_retain_period: 30s
      max_transfer_retries: 0
      wal:
        enabled: true
        dir: /loki/wal

    schema_config:
      configs:
      - from: 2024-01-01
        store: boltdb-shipper
        object_store: filesystem
        schema: v11
        index:
          prefix: index_
          period: 24h

    storage_config:
      boltdb_shipper:
        active_index_directory: /loki/index
        cache_location: /loki/cache
        shared_store: filesystem
      filesystem:
        directory: /loki/chunks

    limits_config:
      enforce_metric_name: false
      reject_old_samples: true
      reject_old_samples_max_age: 168h
      ingestion_rate_mb: 10
      ingestion_burst_size_mb: 20

    chunk_store_config:
      max_look_back_period: 168h

    table_manager:
      retention_deletes_enabled: true
      retention_period: 168h
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: loki
  namespace: loki-stack
spec:
  serviceName: loki
  replicas: 1
  selector:
    matchLabels:
      app: loki
  template:
    metadata:
      labels:
        app: loki
    spec:
      containers:
      - name: loki
        image: grafana/loki:2.9.3
        args:
        - -config.file=/etc/loki/loki.yaml
        ports:
        - containerPort: 3100
          name: http
        - containerPort: 9096
          name: grpc
        volumeMounts:
        - name: config
          mountPath: /etc/loki
        - name: storage
          mountPath: /loki
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
      volumes:
      - name: config
        configMap:
          name: loki-config
  volumeClaimTemplates:
  - metadata:
      name: storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: loki
  namespace: loki-stack
spec:
  selector:
    app: loki
  ports:
  - name: http
    port: 3100
    targetPort: 3100
  - name: grpc
    port: 9096
    targetPort: 9096
```

### Configuration de Promtail

Déployez Promtail comme DaemonSet :

```yaml
# promtail-daemonset.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: loki-stack
data:
  promtail.yaml: |
    server:
      http_listen_port: 9080
      grpc_listen_port: 0

    positions:
      filename: /tmp/positions.yaml

    clients:
    - url: http://loki:3100/loki/api/v1/push

    scrape_configs:
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod

      relabel_configs:
      # Filtre pour les pods à scraper
      - source_labels: [__meta_kubernetes_pod_node_name]
        target_label: __host__

      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)

      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace

      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod

      - source_labels: [__meta_kubernetes_container_name]
        target_label: container

      - replacement: /var/log/pods/*$1/*.log
        separator: /
        source_labels:
        - __meta_kubernetes_pod_uid
        - __meta_kubernetes_pod_container_name
        target_label: __path__

      pipeline_stages:
      - docker: {}

      - multiline:
          firstline: '^\d{4}-\d{2}-\d{2}'
          max_wait_time: 3s

      - regex:
          expression: '^(?P<timestamp>\S+)\s+(?P<level>\S+)\s+(?P<message>.*)$'

      - labels:
          level:

      - timestamp:
          source: timestamp
          format: RFC3339Nano
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
  namespace: loki-stack
spec:
  selector:
    matchLabels:
      app: promtail
  template:
    metadata:
      labels:
        app: promtail
    spec:
      serviceAccountName: promtail
      containers:
      - name: promtail
        image: grafana/promtail:2.9.3
        args:
        - -config.file=/etc/promtail/promtail.yaml
        volumeMounts:
        - name: config
          mountPath: /etc/promtail
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: positions
          mountPath: /tmp
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
      volumes:
      - name: config
        configMap:
          name: promtail-config
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: positions
        emptyDir: {}
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: promtail
  namespace: loki-stack
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: promtail
rules:
- apiGroups: [""]
  resources: ["nodes", "pods", "services"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: promtail
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: promtail
subjects:
- kind: ServiceAccount
  name: promtail
  namespace: loki-stack
```

## Configuration de Grafana pour Loki

### Ajout de la source de données Loki

Dans Grafana, ajoutez Loki comme source de données :

```yaml
# grafana-datasource-loki.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasource-loki
  namespace: monitoring
data:
  loki.yaml: |
    apiVersion: 1
    datasources:
    - name: Loki
      type: loki
      access: proxy
      url: http://loki.loki-stack.svc.cluster.local:3100
      jsonData:
        maxLines: 1000
        derivedFields:
          - datasourceUid: prometheus-uid
            matcherRegex: "traceID=(\\w+)"
            name: TraceID
            url: "$${__value.raw}"
            urlDisplayLabel: "View Trace"
```

Ou configurez manuellement via l'interface :

1. **Configuration** → **Data Sources** → **Add data source**
2. Sélectionnez **Loki**
3. URL : `http://loki.loki-stack.svc.cluster.local:3100`
4. Laissez les autres options par défaut
5. **Save & Test**

### Configuration des derived fields

Les derived fields permettent de créer des liens entre logs et traces :

```json
{
  "derivedFields": [
    {
      "matcherRegex": "traceID=(\\w+)",
      "name": "TraceID",
      "url": "${__value.raw}",
      "datasourceUid": "jaeger-uid",
      "urlDisplayLabel": "View in Jaeger"
    },
    {
      "matcherRegex": "userId=(\\w+)",
      "name": "UserID",
      "url": "/d/user-dashboard?var-user=${__value.raw}",
      "urlDisplayLabel": "View User Dashboard"
    }
  ]
}
```

## LogQL : Le langage de requête

### Syntaxe de base

LogQL ressemble à PromQL mais pour les logs :

```logql
# Sélecteur de flux simple
{app="nginx"}

# Multiple labels
{namespace="production", app="api-server"}

# Regex sur labels
{app=~"api.*", namespace!="kube-system"}

# Filtrage de texte
{app="nginx"} |= "error"

# Exclusion de texte
{app="nginx"} != "health"

# Regex sur le contenu
{app="nginx"} |~ "status=[4-5].."

# Parsing JSON
{app="api"} | json | level="ERROR"

# Parsing logfmt
{app="app"} | logfmt | duration > 1s
```

### Requêtes d'agrégation

LogQL permet des agrégations similaires à PromQL :

```logql
# Compter les logs par minute
rate({app="nginx"}[5m])

# Somme des erreurs par namespace
sum by (namespace) (rate({level="ERROR"}[5m]))

# Top 10 des pods avec le plus de logs
topk(10, sum by (pod) (rate({namespace="production"}[5m])))

# Bytes de logs par service
bytes_rate({namespace="production"}[5m])

# Percentile de la durée (après parsing)
{app="api"}
  | json
  | duration > 0
  | quantile_over_time(0.95, duration[5m])
```

### Exemples pratiques

```logql
# Erreurs de l'API dans la dernière heure
{app="api-server", namespace="production"}
  |= "ERROR"
  | json
  | line_format "{{.timestamp}} {{.level}} {{.message}}"

# Requêtes lentes (>1s)
{app="nginx"}
  | regexp `(?P<duration>\d+\.\d+)ms`
  | duration > 1000

# Correlation avec un request ID
{namespace="production"} |= "req-12345-abcd"

# Logs d'un déploiement spécifique
{app="myapp", version="v2.1.0"}
  | json
  | level=~"ERROR|WARNING"

# Pattern matching pour extraction
{app="postgres"}
  | pattern `<_> <level> <_> <message>`
  | level="ERROR"
```

## Création de dashboards unifiés

### Dashboard métriques + logs

Créez un dashboard Grafana combinant Prometheus et Loki :

```json
{
  "dashboard": {
    "title": "Application Overview with Logs",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "datasource": "Prometheus",
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{status}}"
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 }
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "datasource": "Prometheus",
            "expr": "rate(http_requests_total{status=~\"5..\"}[5m])"
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 }
      },
      {
        "title": "Application Logs",
        "type": "logs",
        "targets": [
          {
            "datasource": "Loki",
            "expr": "{app=\"$app\", namespace=\"$namespace\"}"
          }
        ],
        "options": {
          "showTime": true,
          "showLabels": true,
          "showCommonLabels": false,
          "wrapLogMessage": false,
          "sortOrder": "Descending",
          "dedupStrategy": "none"
        },
        "gridPos": { "h": 10, "w": 24, "x": 0, "y": 8 }
      }
    ],
    "templating": {
      "list": [
        {
          "name": "namespace",
          "type": "query",
          "datasource": "Prometheus",
          "query": "label_values(namespace)",
          "current": { "value": "production" }
        },
        {
          "name": "app",
          "type": "query",
          "datasource": "Prometheus",
          "query": "label_values(app)",
          "current": { "value": "api-server" }
        }
      ]
    }
  }
}
```

### Panel de corrélation automatique

Configurez des panels qui se mettent à jour mutuellement :

```javascript
// Panel de métriques avec data link vers logs
{
  "fieldConfig": {
    "defaults": {
      "links": [
        {
          "title": "View logs",
          "url": "/explore?left={\"datasource\":\"Loki\",\"queries\":[{\"expr\":\"{pod=\\\"${__field.labels.pod}\\\"}\",\"refId\":\"A\"}],\"range\":{\"from\":\"${__from}\",\"to\":\"${__to}\"}}"
        }
      ]
    }
  }
}
```

### Mixed datasources dans un panel

Grafana permet de mixer les sources dans un même panel :

```json
{
  "targets": [
    {
      "datasource": "Prometheus",
      "expr": "rate(app_errors_total[5m])",
      "refId": "A",
      "legendFormat": "Error Rate"
    },
    {
      "datasource": "Loki",
      "expr": "sum(rate({app=\"myapp\"} |= \"ERROR\" [5m]))",
      "refId": "B",
      "legendFormat": "Log Errors"
    }
  ]
}
```

## Optimisation des performances

### Labels vs recherche full-text

**Utilisez les labels pour le filtrage principal** :
```logql
# ✅ BON : Filtre par labels d'abord
{namespace="production", app="api"} |= "error"

# ❌ MAUVAIS : Recherche full-text seule
{} |= "error" |= "production"
```

### Indexation des labels importants

Configurez Promtail pour extraire les labels critiques :

```yaml
pipeline_stages:
- json:
    expressions:
      level: level
      service: service
      user_id: user_id

- labels:
    level:
    service:
    # user_id NON - trop de cardinalité!

- metrics:
    log_lines_total:
      type: Counter
      description: "Total log lines"
      prefix: promtail_
      max_idle_duration: 24h
      config:
        match_all: true
        action: inc
```

### Stratégies de rétention

```yaml
# Configuration de rétention par flux
schema_config:
  configs:
  - from: 2024-01-01
    store: boltdb-shipper
    object_store: filesystem
    schema: v11
    index:
      prefix: index_
      period: 24h
    row_shards: 16

limits_config:
  retention_period: 168h  # 7 jours par défaut

  per_stream_rate_limit: 3MB
  per_stream_rate_limit_burst: 15MB

  # Rétention spécifique par tenant/label
  retention_stream:
  - selector: '{namespace="production"}'
    priority: 1
    period: 336h  # 14 jours pour production

  - selector: '{level="DEBUG"}'
    priority: 2
    period: 24h  # 1 jour pour debug
```

## Cas d'usage de corrélation

### Debugging d'un incident de production

Workflow typique d'investigation :

```logql
# 1. Identifier le pic d'erreurs dans Grafana
rate(http_requests_total{status="500"}[5m])

# 2. Cliquer sur le pic pour voir les logs correspondants
{namespace="production", app="api"}
  |= "ERROR"
  | json
  | __timestamp__ >= 1704000000
  | __timestamp__ <= 1704003600

# 3. Trouver un pattern d'erreur
{namespace="production"}
  |= "database connection failed"
  | json
  | line_format "{{.pod}} {{.message}}"

# 4. Vérifier les logs du service database
{app="postgres", namespace="production"}
  | json
  | level="ERROR"

# 5. Corréler avec les métriques de connexion
postgres_connections_active{namespace="production"}
```

### Analyse de performance end-to-end

```logql
# Extraire les durées depuis les logs
{app="api"}
  | json
  | duration > 0
  | __error__=""

# Calculer le p95 sur 5 minutes
quantile_over_time(0.95,
  {app="api"}
    | json
    | unwrap duration [5m]
) by (endpoint)

# Comparer avec les métriques Prometheus
histogram_quantile(0.95,
  rate(http_request_duration_seconds_bucket[5m])
) by (endpoint)
```

### Audit trail complet

```logql
# Suivre un utilisateur à travers le système
{namespace="production"}
  | json
  | user_id="user-123"
  | line_format "{{.timestamp}} [{{.app}}] {{.action}} {{.details}}"

# Créer une timeline d'activité
sum by (action) (
  rate(
    {namespace="production"}
      | json
      | user_id="user-123" [1h]
  )
)
```

## Alerting avec Loki

### Configuration des règles d'alerte

```yaml
# loki-rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-rules
  namespace: loki-stack
data:
  rules.yaml: |
    groups:
    - name: logs_alerts
      interval: 1m
      rules:

      # Alerte sur taux d'erreur élevé
      - alert: HighErrorRate
        expr: |
          sum(rate({namespace="production"} |= "ERROR" [5m])) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate in production logs"
          description: "{{ $value }} errors per second in the last 5 minutes"

      # Alerte sur pattern spécifique
      - alert: DatabaseConnectionErrors
        expr: |
          sum(rate({app="api"} |= "database connection failed" [5m])) > 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Database connection failures detected"

      # Alerte sur absence de logs
      - alert: NoLogsReceived
        expr: |
          sum(rate({namespace="production", app="api"}[5m])) == 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "No logs received from API service"
```

### Intégration avec AlertManager

```yaml
# Configuration Loki pour les alertes
ruler:
  enable_api: true
  enable_alertmanager_v2: true
  alertmanager_url: http://alertmanager:9093
  storage:
    type: local
    local:
      directory: /loki/rules
  rule_path: /tmp/rules
  ring:
    kvstore:
      store: inmemory
```

## Instrumentation des applications

### Logs structurés optimisés pour Loki

#### Python avec structlog

```python
import structlog
import time
from opentelemetry import trace

# Configuration pour Loki
structlog.configure(
    processors=[
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.add_log_level,
        structlog.processors.format_exc_info,
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
)

logger = structlog.get_logger()

class LokiLogger:
    def __init__(self, service_name, namespace):
        self.logger = structlog.get_logger()
        self.service_name = service_name
        self.namespace = namespace
        self.tracer = trace.get_tracer(__name__)

    def log_with_trace(self, level, message, **kwargs):
        """Log avec contexte de trace pour corrélation"""
        span = trace.get_current_span()
        span_context = span.get_span_context()

        log_data = {
            'service': self.service_name,
            'namespace': self.namespace,
            'level': level,
            'message': message,
            'timestamp': time.time(),
            'trace_id': format(span_context.trace_id, '032x'),
            'span_id': format(span_context.span_id, '016x'),
            **kwargs
        }

        getattr(self.logger, level.lower())(message, **log_data)
        return log_data

    def info(self, message, **kwargs):
        return self.log_with_trace('INFO', message, **kwargs)

    def error(self, message, **kwargs):
        return self.log_with_trace('ERROR', message, **kwargs)

    def warning(self, message, **kwargs):
        return self.log_with_trace('WARNING', message, **kwargs)

# Utilisation
app_logger = LokiLogger('api-server', 'production')

@app.route('/api/process')
def process_request():
    with tracer.start_as_current_span("process_request") as span:
        app_logger.info("Processing request",
                       endpoint="/api/process",
                       user_id=request.user_id,
                       request_id=request.id)

        try:
            result = complex_processing()
            app_logger.info("Request processed successfully",
                          duration=span.duration,
                          result_size=len(result))
            return result
        except Exception as e:
            app_logger.error("Processing failed",
                           error=str(e),
                           stack_trace=traceback.format_exc())
            raise

⏭️

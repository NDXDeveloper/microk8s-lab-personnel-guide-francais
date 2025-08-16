🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.2 Logs centralisés (ELK/EFK stack)

## Introduction : Pourquoi centraliser les logs ?

Dans un environnement Kubernetes, vos applications s'exécutent dans des conteneurs éphémères répartis sur potentiellement plusieurs nodes. Quand un pod crashe ou est redéployé, ses logs disparaissent avec lui. Imaginez devoir débugger un problème intermittent en parcourant manuellement les logs de dizaines de pods qui apparaissent et disparaissent constamment. C'est un cauchemar opérationnel que la centralisation des logs résout élégamment.

### Les défis des logs dans Kubernetes

Kubernetes présente plusieurs défis uniques pour la gestion des logs :

**Éparpillement des sources** : Une simple requête utilisateur peut traverser plusieurs microservices, chacun générant ses propres logs. Sans centralisation, reconstituer le parcours d'une requête nécessite de consulter plusieurs sources.

**Nature éphémère** : Les pods sont créés et détruits dynamiquement. Les logs locaux disparaissent avec le conteneur, rendant l'analyse post-mortem impossible.

**Volume et vélocité** : Même un petit cluster peut générer des gigaoctets de logs par jour. Les parcourir manuellement est impraticable.

**Formats hétérogènes** : Chaque application peut logger dans un format différent (JSON, texte brut, XML), compliquant l'analyse croisée.

### La solution : un pipeline de logs centralisé

Un système de logs centralisé collecte, traite, stocke et permet de rechercher tous les logs de votre cluster depuis une interface unique. C'est comme avoir un moteur de recherche Google pour vos logs, capable de retrouver instantanément une erreur spécifique parmi des millions de lignes.

## Comprendre les stacks ELK et EFK

Deux stacks dominent l'écosystème de la centralisation des logs : ELK et EFK. Elles partagent le même objectif mais diffèrent dans leur approche de la collecte.

### La stack ELK : Elasticsearch, Logstash, Kibana

**Elasticsearch** est le cœur de stockage et de recherche. C'est une base de données distribuée optimisée pour la recherche full-text. Pensez-y comme un index géant où chaque mot de vos logs est catalogué pour une recherche ultra-rapide.

**Logstash** est le pipeline de traitement. Il ingère les logs depuis diverses sources, les transforme (parsing, enrichissement, filtrage) et les envoie vers Elasticsearch. C'est le couteau suisse de la transformation de données.

**Kibana** est l'interface de visualisation. C'est votre fenêtre sur les données, offrant recherche, tableaux de bord, alertes et analyses.

### La stack EFK : Elasticsearch, Fluentd/Fluent Bit, Kibana

La stack EFK remplace Logstash par Fluentd ou sa version allégée Fluent Bit :

**Fluentd** est un collecteur de logs écrit en Ruby, conçu spécifiquement pour les environnements cloud-native. Il est plus léger que Logstash et s'intègre naturellement avec Kubernetes.

**Fluent Bit** est la version ultra-légère de Fluentd, écrite en C. Avec une empreinte mémoire de seulement 450KB, c'est le choix idéal pour un lab MicroK8s aux ressources limitées.

### Comparaison pour votre lab MicroK8s

| Aspect | ELK (Logstash) | EFK (Fluent Bit) |
|--------|----------------|-------------------|
| **Empreinte mémoire** | ~500MB+ | ~5-10MB |
| **CPU** | Intensif (JVM) | Minimal |
| **Configuration** | Complexe mais puissante | Simple et directe |
| **Plugins** | Très nombreux | Essentiels disponibles |
| **Intégration K8s** | Via plugins | Native |
| **Recommandé pour lab** | Non | **Oui** |

Pour un lab MicroK8s, nous privilégierons Fluent Bit pour sa légèreté, tout en gardant la puissance d'Elasticsearch et Kibana.

## Architecture de la solution

### Vue d'ensemble du flux de données

```
[Pods Application] → [Stdout/Stderr] → [Node Docker/Containerd]
                                              ↓
                                    [Fluent Bit DaemonSet]
                                              ↓
                                    [Parsing & Enrichissement]
                                              ↓
                                        [Elasticsearch]
                                              ↓
                                          [Kibana UI]
```

### Composants détaillés

**Logs sources** : Vos applications écrivent sur stdout/stderr. Kubernetes capture automatiquement ces flux et les rend disponibles via l'API.

**Fluent Bit DaemonSet** : Un pod Fluent Bit s'exécute sur chaque node, collectant les logs de tous les conteneurs locaux. Le DaemonSet garantit qu'un collecteur est toujours présent sur chaque node.

**Pipeline de traitement** : Fluent Bit parse les logs, ajoute des métadonnées (namespace, pod name, labels), et peut filtrer ou transformer le contenu.

**Elasticsearch** : Stocke les logs indexés, permettant des recherches complexes en millisecondes.

**Kibana** : Interface web pour rechercher, visualiser et analyser vos logs.

## Installation sur MicroK8s

### Préparation du cluster

Avant d'installer la stack, vérifions les ressources disponibles :

```bash
# Vérifier les ressources du cluster
kubectl top nodes

# Créer un namespace dédié
kubectl create namespace logging

# Activer les addons nécessaires
microk8s enable dns storage
```

### Option 1 : Installation avec Elastic Cloud on Kubernetes (ECK)

ECK est l'opérateur officiel d'Elastic pour Kubernetes, simplifiant grandement le déploiement.

```bash
# Installer l'opérateur ECK
kubectl create -f https://download.elastic.co/downloads/eck/2.10.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.10.0/operator.yaml
```

Créez ensuite le manifeste Elasticsearch optimisé pour un lab :

```yaml
# elasticsearch-lab.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-lab
  namespace: logging
spec:
  version: 8.11.0
  nodeSets:
  - name: default
    count: 1  # Single node pour un lab
    config:
      # Optimisations pour environnement contraint
      node.store.allow_mmap: false
      xpack.security.enabled: false  # Désactiver la sécurité pour simplifier
      xpack.security.http.ssl.enabled: false
      discovery.type: single-node  # Mode single node
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 1Gi
              cpu: 500m
            limits:
              memory: 2Gi
              cpu: 1000m
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
```

Déployez Kibana :

```yaml
# kibana-lab.yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana-lab
  namespace: logging
spec:
  version: 8.11.0
  count: 1
  elasticsearchRef:
    name: elasticsearch-lab
  config:
    server.publicBaseUrl: "http://kibana.lab.local"
    elasticsearch.ssl.verificationMode: none
  podTemplate:
    spec:
      containers:
      - name: kibana
        resources:
          requests:
            memory: 512Mi
            cpu: 200m
          limits:
            memory: 1Gi
            cpu: 500m
```

### Option 2 : Installation manuelle légère

Pour un contrôle total ou des ressources très limitées, déployez manuellement :

```yaml
# elasticsearch-minimal.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: logging
spec:
  serviceName: elasticsearch
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
        env:
        - name: discovery.type
          value: single-node
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"  # Limite heap Java
        - name: xpack.security.enabled
          value: "false"
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        resources:
          requests:
            memory: 1Gi
            cpu: 500m
          limits:
            memory: 2Gi
            cpu: 1000m
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: logging
spec:
  selector:
    app: elasticsearch
  ports:
  - name: http
    port: 9200
  - name: transport
    port: 9300
  clusterIP: None
```

## Configuration de Fluent Bit

Fluent Bit est le collecteur qui va récupérer tous les logs de votre cluster.

### Comprendre la configuration Fluent Bit

Fluent Bit utilise un pipeline en quatre étapes :

1. **INPUT** : D'où viennent les logs
2. **PARSER** : Comment les interpréter
3. **FILTER** : Comment les enrichir ou transformer
4. **OUTPUT** : Où les envoyer

### ConfigMap de configuration

```yaml
# fluent-bit-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Daemon Off
        Flush 5
        Log_Level info
        Parsers_File parsers.conf

    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        Parser docker
        Tag kube.*
        Refresh_Interval 5
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On

    [FILTER]
        Name kubernetes
        Match kube.*
        Kube_URL https://kubernetes.default.svc:443
        Kube_CA_File /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix kube.var.log.containers.
        Merge_Log On
        Keep_Log Off
        K8S-Logging.Parser On
        K8S-Logging.Exclude On

    [FILTER]
        Name record_modifier
        Match *
        Record cluster_name microk8s-lab
        Record environment development

    [OUTPUT]
        Name es
        Match *
        Host elasticsearch.logging.svc.cluster.local
        Port 9200
        Logstash_Format On
        Logstash_Prefix k8s-logs
        Retry_Limit 5
        Type _doc

  parsers.conf: |
    [PARSER]
        Name docker
        Format json
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%LZ

    [PARSER]
        Name syslog
        Format regex
        Regex ^<(?<priority>[0-9]+)>(?<time>[^ ]* {1,2}[^ ]* [^ ]*) (?<host>[^ ]*) (?<ident>[^ \[]*)(?:\[(?<pid>[0-9]+)\])?(?:[^\:]*\:)? *(?<message>.*)$
        Time_Key time
        Time_Format %b %d %H:%M:%S
```

### DaemonSet Fluent Bit

```yaml
# fluent-bit-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.2
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
        resources:
          requests:
            memory: 100Mi
            cpu: 100m
          limits:
            memory: 200Mi
            cpu: 200m
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
      tolerations:
      - effect: NoSchedule
        operator: Exists
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit-read
rules:
- apiGroups: [""]
  resources:
  - namespaces
  - pods
  - nodes
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit-read
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit-read
subjects:
- kind: ServiceAccount
  name: fluent-bit
  namespace: logging
```

## Formats de logs et parsing

### Logs structurés vs non structurés

**Logs non structurés** (texte brut) :
```
2024-01-15 10:23:45 ERROR Failed to connect to database server
2024-01-15 10:23:46 INFO Retrying connection attempt 1/3
```

**Logs structurés** (JSON) :
```json
{"timestamp":"2024-01-15T10:23:45Z","level":"ERROR","message":"Failed to connect to database","server":"db-01","retry":0}
{"timestamp":"2024-01-15T10:23:46Z","level":"INFO","message":"Retrying connection","server":"db-01","retry":1}
```

Les logs structurés sont beaucoup plus faciles à parser et analyser. Privilégiez toujours JSON pour vos applications.

### Configuration du logging dans vos applications

#### Python avec structlog

```python
import structlog
import logging

# Configuration structlog
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

# Utilisation
logger.info("Application started",
            version="1.2.3",
            environment="production")

try:
    result = process_order(order_id)
    logger.info("Order processed successfully",
                order_id=order_id,
                amount=result.amount,
                customer_id=result.customer_id)
except Exception as e:
    logger.error("Order processing failed",
                 order_id=order_id,
                 error=str(e),
                 exc_info=True)
```

#### Node.js avec Winston

```javascript
const winston = require('winston');

// Configuration Winston pour JSON
const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'user-service',
    version: process.env.APP_VERSION || '1.0.0'
  },
  transports: [
    new winston.transports.Console()
  ]
});

// Utilisation
logger.info('Server started', {
  port: 3000,
  environment: process.env.NODE_ENV
});

// Log avec contexte
function processRequest(req, res) {
  const requestLogger = logger.child({
    requestId: req.id,
    userId: req.user?.id
  });

  requestLogger.info('Processing request', {
    method: req.method,
    path: req.path
  });

  try {
    // Traitement...
    requestLogger.info('Request successful', {
      statusCode: 200
    });
  } catch (error) {
    requestLogger.error('Request failed', {
      error: error.message,
      stack: error.stack
    });
  }
}
```

### Parsers Fluent Bit personnalisés

Pour des formats de logs spécifiques, créez des parsers personnalisés :

```ini
# Parser pour logs Nginx
[PARSER]
    Name nginx
    Format regex
    Regex ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
    Time_Key time
    Time_Format %d/%b/%Y:%H:%M:%S %z

# Parser pour logs d'application custom
[PARSER]
    Name app_custom
    Format regex
    Regex ^\[(?<time>[^\]]*)\] (?<level>[^ ]*) (?<component>[^ ]*) - (?<message>.*)$
    Time_Key time
    Time_Format %Y-%m-%d %H:%M:%S
```

## Enrichissement et filtrage des logs

### Ajout de métadonnées Kubernetes

Fluent Bit enrichit automatiquement les logs avec les métadonnées Kubernetes :

```json
{
  "log": "Order processed successfully",
  "kubernetes": {
    "pod_name": "api-server-7d9c9d6b5-x2ktv",
    "namespace_name": "production",
    "pod_id": "b8f7c818-22e9-11ee-be56-0242ac120002",
    "labels": {
      "app": "api-server",
      "version": "v1.2.3"
    },
    "annotations": {
      "prometheus.io/scrape": "true"
    },
    "host": "node-01",
    "container_name": "api",
    "docker_id": "abc123def456"
  }
}
```

### Filtrage des logs sensibles

Protégez les données sensibles avec des filtres :

```ini
# Masquer les emails
[FILTER]
    Name modify
    Match *
    Condition Key_value_matches email *@*
    Set email [REDACTED]

# Masquer les numéros de carte
[FILTER]
    Name lua
    Match *
    script mask_credit_cards.lua
    call mask_cards
```

Script Lua pour masquage avancé :
```lua
-- mask_credit_cards.lua
function mask_cards(tag, timestamp, record)
    local pattern = "%d%d%d%d%s?%d%d%d%d%s?%d%d%d%d%s?%d%d%d%d"
    if record["message"] then
        record["message"] = string.gsub(record["message"], pattern, "XXXX-XXXX-XXXX-XXXX")
    end
    return 1, timestamp, record
end
```

### Multiline logs

Gérez les logs multi-lignes (stack traces, logs SQL) :

```ini
[INPUT]
    Name tail
    Path /var/log/containers/*app*.log
    Parser docker
    Tag app.*
    Multiline On
    Parser_Firstline app_multiline_start

[PARSER]
    Name app_multiline_start
    Format regex
    Regex /^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})/
    Time_Key time
    Time_Format %Y-%m-%d %H:%M:%S
```

## Kibana : exploration et visualisation

### Première connexion à Kibana

Accédez à Kibana via port-forward ou Ingress :

```bash
# Port-forward pour accès local
kubectl port-forward -n logging svc/kibana-lab-kb-http 5601:5601

# Ou créez un Ingress
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana
  namespace: logging
spec:
  rules:
  - host: kibana.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kibana-lab-kb-http
            port:
              number: 5601
```

### Configuration des index patterns

Dans Kibana, configurez les patterns d'index :

1. **Navigation** : Stack Management → Index Patterns
2. **Création** : Create index pattern
3. **Pattern** : `k8s-logs-*`
4. **Time field** : `@timestamp`

### Requêtes de recherche Kibana (KQL)

Kibana Query Language (KQL) permet des recherches puissantes :

```kql
# Logs d'erreur du namespace production
kubernetes.namespace_name: "production" AND level: "ERROR"

# Logs d'un pod spécifique
kubernetes.pod_name: "api-server-7d9c9d6b5-x2ktv"

# Erreurs de connexion database
message: *database* AND (level: "ERROR" OR level: "CRITICAL")

# Logs des 15 dernières minutes avec status code 500
status_code: 500 AND @timestamp > now-15m

# Logs contenant une IP spécifique
message: "192.168.1.100"

# Exclusion de certains pods
NOT kubernetes.pod_name: *fluent-bit*
```

### Création de dashboards

Créez un dashboard de monitoring des erreurs :

1. **Visualisation 1 : Timeline des erreurs**
   - Type : Line chart
   - Y-axis : Count
   - X-axis : @timestamp
   - Split series : kubernetes.namespace_name
   - Filter : level: "ERROR"

2. **Visualisation 2 : Top 10 pods avec erreurs**
   - Type : Data table
   - Metric : Count
   - Bucket : Terms on kubernetes.pod_name
   - Filter : level: "ERROR"

3. **Visualisation 3 : Distribution des log levels**
   - Type : Pie chart
   - Metric : Count
   - Bucket : Terms on level

4. **Visualisation 4 : Logs stream**
   - Type : Saved search
   - Columns : Time, Level, Namespace, Pod, Message
   - Sort : @timestamp DESC

### Alertes dans Kibana

Configurez des alertes sur les patterns de logs :

```json
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["k8s-logs-*"],
        "body": {
          "query": {
            "bool": {
              "must": [
                {"range": {"@timestamp": {"gte": "now-5m"}}},
                {"term": {"level": "ERROR"}},
                {"term": {"kubernetes.namespace_name": "production"}}
              ]
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.hits.total": {
        "gt": 10
      }
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "to": "admin@lab.local",
        "subject": "High error rate detected",
        "body": "{{ctx.payload.hits.total}} errors in the last 5 minutes"
      }
    }
  }
}
```

## Optimisations pour un lab

### Gestion de la rétention

Configurez des politiques de rétention adaptées :

```json
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "1GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "1d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "delete": {
        "min_age": "7d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

### Sampling des logs

Pour réduire le volume, échantillonnez les logs non critiques :

```ini
[FILTER]
    Name lua
    Match *
    script sample_logs.lua
    call sample

# sample_logs.lua
function sample(tag, timestamp, record)
    -- Garder tous les logs d'erreur
    if record["level"] == "ERROR" or record["level"] == "CRITICAL" then
        return 1, timestamp, record
    end

    -- Échantillonner 10% des logs INFO
    if record["level"] == "INFO" then
        if math.random() < 0.1 then
            return 1, timestamp, record
        else
            return -1, timestamp, record
        end
    end

    return 1, timestamp, record
end
```

### Réduction de l'empreinte Elasticsearch

Optimisations pour environnement contraint :

```yaml
# Désactiver les replicas (single node)
PUT /k8s-logs-*/_settings
{
  "number_of_replicas": 0
}

# Réduire le refresh interval
PUT /k8s-logs-*/_settings
{
  "refresh_interval": "30s"
}

# Limiter les shards
PUT _template/logs_template
{
  "index_patterns": ["k8s-logs-*"],
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  }
}
```

## Cas d'usage pratiques

### Debugging d'une erreur intermittente

Workflow typique pour investiguer une erreur :

1. **Identifier le pattern d'erreur** dans Kibana
2. **Corréler avec les métriques** Prometheus (pic CPU, latence)
3. **Isoler le pod problématique** via les labels Kubernetes
4. **Analyser le contexte** : logs avant/après l'erreur
5. **Vérifier les logs des services dépendants**

### Audit de sécurité

Recherchez les activités suspectes :

```kql
# Tentatives de connexion échouées
message: "authentication failed" OR message: "login failed"

# Accès aux ressources sensibles
path: */admin/* OR path: */api/internal/*

# Changements de configuration
message: "configuration changed" OR message: "settings updated"
```

### Analyse de performance

Identifiez les goulots d'étranglement :

```kql
# Requêtes lentes (>1s)
response_time: >1000

# Timeouts database
message: "connection timeout" AND component: "database"

# Saturation de queue
queue_size: >100
```

## Troubleshooting courant

### Fluent Bit ne collecte pas les logs

Vérifications à effectuer :

```bash
# Vérifier le statut du DaemonSet
kubectl get daemonset -n logging fluent-bit

# Vérifier les logs de Fluent Bit
kubectl logs -n logging -l app=fluent-bit

# Vérifier les permissions
kubectl get clusterrolebinding fluent-bit-read

# Vérifier le montage des volumes
kubectl describe pod -n logging -l app=fluent-bit
```

### Elasticsearch disk watermark

Si Elasticsearch arrête d'indexer :

```bash
# Vérifier l'espace disque
kubectl exec -n logging elasticsearch-0 -- df -h

# Ajuster les watermarks
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "85%",
    "cluster.routing.allocation.disk.watermark.high": "90%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
  }
}'

# Forcer la suppression des vieux index
curl -X DELETE "localhost:9200/k8s-logs-2024.01.*"
```

### Performance dégradée de Kibana

Optimisations possibles :

```yaml
# Augmenter les ressources
kubectl edit kibana -n logging kibana-lab

# Désactiver les fonctionnalités non utilisées
kibana.yml: |
  xpack.apm.enabled: false
  xpack.ml.enabled: false
  xpack.infra.enabled: false
```

## Migration vers des solutions alternatives

### Loki : l'alternative légère

Si les ressources sont vraiment limitées, considérez Loki :

```yaml
# Installation simple avec Helm
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack -n logging \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=5Gi \
  --set promtail.enabled=true
```

**Avantages de Loki pour un lab :**
- Empreinte mémoire 10x plus petite qu'Elasticsearch
- Stockage optimisé (n'indexe que les métadonnées)
- Intégration native avec Grafana
- Syntaxe de requête similaire à Prometheus (LogQL)

### OpenSearch : le fork open source

Suite aux changements de licence Elastic, OpenSearch offre une alternative :

```bash
# Installation OpenSearch
helm repo add opensearch https://opensearch-project.github.io/helm-charts/
helm install opensearch opensearch/opensearch \
  --set replicas=1 \
  --set resources.requests.memory=1Gi \
  --set singleNode=true \
  -n logging
```

### Cloud providers

Pour un lab de test, les tiers gratuits peuvent suffire :
- **Elastic Cloud** : 14 jours d'essai gratuit
- **AWS CloudWatch** : Logs gratuits jusqu'à 5GB
- **Google Cloud Logging** : 50GB/mois gratuits
- **Azure Monitor** : 5GB/mois gratuits

## Intégration avec l'écosystème d'observabilité

### Corrélation logs-métriques

Liez vos logs avec les métriques Prometheus :

```python
# Ajouter des labels communs
import logging
import time
from prometheus_client import Counter, Histogram

# Métriques Prometheus
request_counter = Counter('requests_total', 'Total requests', ['endpoint', 'status'])
request_duration = Histogram('request_duration_seconds', 'Request duration', ['endpoint'])

# Logger structuré
logger = logging.getLogger(__name__)

def handle_request(endpoint, request_id):
    start_time = time.time()

    # Log avec métadonnées pour corrélation
    logger.info("Request started", extra={
        'request_id': request_id,
        'endpoint': endpoint,
        'timestamp': start_time
    })

    try:
        # Traitement...
        result = process(endpoint)
        status = 'success'
        status_code = 200
    except Exception as e:
        logger.error("Request failed", extra={
            'request_id': request_id,
            'endpoint': endpoint,
            'error': str(e)
        }, exc_info=True)
        status = 'error'
        status_code = 500

    duration = time.time() - start_time

    # Métriques
    request_counter.labels(endpoint=endpoint, status=status).inc()
    request_duration.labels(endpoint=endpoint).observe(duration)

    # Log final avec durée
    logger.info("Request completed", extra={
        'request_id': request_id,
        'endpoint': endpoint,
        'duration': duration,
        'status': status_code
    })
```

### Liens depuis Grafana

Configurez des liens contextuels vers Kibana depuis vos dashboards Grafana :

```json
{
  "links": [
    {
      "title": "View logs in Kibana",
      "url": "http://kibana.lab.local/app/discover#/?_g=(time:(from:'${__from}',to:'${__to}'))&_a=(query:(match:(kubernetes.pod_name:'${__series.labels.pod}')))",
      "targetBlank": true
    }
  ]
}
```

### Enrichissement avec traces

Ajoutez des trace IDs à vos logs pour la corrélation avec Jaeger :

```javascript
const { trace, context } = require('@opentelemetry/api');

function processRequest(req, res) {
  const span = trace.getActiveSpan();
  const traceId = span?.spanContext().traceId;

  // Ajouter le trace ID aux logs
  logger.info('Processing request', {
    traceId: traceId,
    spanId: span?.spanContext().spanId,
    method: req.method,
    path: req.path
  });

  // Le trace ID permet de retrouver les logs dans Kibana
  // et la trace complète dans Jaeger
}
```

## Patterns et anti-patterns

### Bonnes pratiques

**DO - À faire :**

1. **Logs structurés JSON** : Toujours préférer JSON au texte brut
2. **Niveaux de log appropriés** : DEBUG pour dev, INFO pour prod
3. **Contexte riche** : Inclure request ID, user ID, transaction ID
4. **Timestamps ISO 8601** : Format standard pour l'horodatage
5. **Échantillonnage intelligent** : Garder 100% des erreurs, sample le reste
6. **Rotation des index** : Suppression automatique des vieux logs
7. **Monitoring du pipeline** : Surveiller Fluent Bit et Elasticsearch

**DON'T - À éviter :**

1. **Logs sensibles** : Ne jamais logger mots de passe, tokens, PII
2. **Logs verbeux en production** : Éviter DEBUG/TRACE en prod
3. **Parsing complexe** : Préférer JSON natif au regex parsing
4. **Rétention infinie** : Définir des politiques de suppression
5. **Logs synchrones** : Utiliser des buffers asynchrones
6. **Format inconsistant** : Standardiser à travers toutes les apps

### Exemples d'anti-patterns

```python
# ❌ MAUVAIS : Log non structuré avec données sensibles
print(f"User {user.email} logged in with password {password}")

# ✅ BON : Log structuré sans données sensibles
logger.info("User authentication", {
    'event': 'login',
    'user_id': user.id,
    'success': True,
    'ip_address': request.remote_addr
})

# ❌ MAUVAIS : Trop de détails en production
logger.debug(f"Full request object: {request.__dict__}")

# ✅ BON : Informations pertinentes seulement
logger.debug("Request received", {
    'method': request.method,
    'path': request.path,
    'size': len(request.data)
})
```

## Monitoring de la stack de logging

### Métriques à surveiller

Créez un dashboard Grafana pour monitorer votre pipeline de logs :

```promql
# Taux de logs collectés par Fluent Bit
rate(fluentbit_input_records_total[5m])

# Erreurs de parsing
rate(fluentbit_filter_drop_records_total[5m])

# Latence d'envoi vers Elasticsearch
histogram_quantile(0.99, fluentbit_output_proc_time_seconds_bucket)

# Santé Elasticsearch
elasticsearch_cluster_health_status

# Utilisation disque des index
elasticsearch_filesystem_data_size_bytes / elasticsearch_filesystem_data_free_bytes

# Taux d'indexation
rate(elasticsearch_indices_indexing_index_total[5m])
```

### Alertes critiques

```yaml
# Alerte si Fluent Bit ne peut pas envoyer les logs
- alert: FluentBitOutputError
  expr: rate(fluentbit_output_errors_total[5m]) > 0
  for: 5m
  annotations:
    summary: "Fluent Bit cannot send logs to Elasticsearch"

# Alerte si Elasticsearch manque d'espace
- alert: ElasticsearchDiskSpace
  expr: elasticsearch_filesystem_data_available_bytes < 1073741824  # 1GB
  for: 5m
  annotations:
    summary: "Elasticsearch running out of disk space"

# Alerte si le pipeline de logs est en retard
- alert: LogPipelineLag
  expr: time() - fluentbit_input_tail_files_rotated_total > 300
  for: 10m
  annotations:
    summary: "Log pipeline is lagging behind"
```

## Scripts d'automatisation

### Script de déploiement complet

```bash
#!/bin/bash
# deploy-logging-stack.sh

set -e

NAMESPACE="logging"

echo "📦 Creating namespace..."
kubectl create namespace $NAMESPACE || true

echo "🔧 Installing ECK operator..."
kubectl create -f https://download.elastic.co/downloads/eck/2.10.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.10.0/operator.yaml

echo "⏳ Waiting for operator..."
kubectl wait --for=condition=Ready pods -l control-plane=elastic-operator -n elastic-system --timeout=300s

echo "🗄️ Deploying Elasticsearch..."
kubectl apply -f elasticsearch-lab.yaml

echo "📊 Deploying Kibana..."
kubectl apply -f kibana-lab.yaml

echo "🔍 Configuring Fluent Bit..."
kubectl apply -f fluent-bit-config.yaml
kubectl apply -f fluent-bit-daemonset.yaml

echo "⏳ Waiting for Elasticsearch..."
kubectl wait --for=condition=Ready elasticsearch -n $NAMESPACE elasticsearch-lab --timeout=600s

echo "⏳ Waiting for Kibana..."
kubectl wait --for=condition=Ready kibana -n $NAMESPACE kibana-lab --timeout=300s

echo "✅ Logging stack deployed!"
echo "Access Kibana at: http://localhost:5601"
kubectl port-forward -n $NAMESPACE svc/kibana-lab-kb-http 5601:5601
```

### Script de nettoyage des logs

```bash
#!/bin/bash
# cleanup-old-logs.sh

# Supprimer les index de plus de 7 jours
CUTOFF_DATE=$(date -d '7 days ago' +%Y.%m.%d)
ES_URL="http://localhost:9200"

echo "🗑️ Cleaning indices older than $CUTOFF_DATE..."

# Lister tous les index
indices=$(curl -s "$ES_URL/_cat/indices/k8s-logs-*" | awk '{print $3}')

for index in $indices; do
  index_date=$(echo $index | grep -oE '[0-9]{4}\.[0-9]{2}\.[0-9]{2}')
  if [[ ! -z "$index_date" && "$index_date" < "$CUTOFF_DATE" ]]; then
    echo "Deleting $index..."
    curl -X DELETE "$ES_URL/$index"
  fi
done

echo "✅ Cleanup complete!"
```

### Script de backup des configurations

```bash
#!/bin/bash
# backup-logging-config.sh

BACKUP_DIR="logging-backup-$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

echo "💾 Backing up configurations..."

# Backup Kibana saved objects
kubectl port-forward -n logging svc/kibana-lab-kb-http 5601:5601 &
PF_PID=$!
sleep 5

curl -X POST "localhost:5601/api/saved_objects/_export" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{"type": ["dashboard", "visualization", "search", "index-pattern"]}' \
  > "$BACKUP_DIR/kibana-objects.ndjson"

kill $PF_PID

# Backup Kubernetes manifests
kubectl get -n logging -o yaml \
  elasticsearch,kibana,configmap,daemonset,service,ingress \
  > "$BACKUP_DIR/k8s-manifests.yaml"

# Backup Elasticsearch templates
kubectl exec -n logging elasticsearch-0 -- \
  curl -s localhost:9200/_template \
  > "$BACKUP_DIR/es-templates.json"

echo "✅ Backup saved to $BACKUP_DIR/"
```

## Conclusion et prochaines étapes

Félicitations ! Vous avez maintenant une stack de logging centralisée opérationnelle dans votre lab MicroK8s. Cette infrastructure vous permet de :

- **Collecter** tous les logs de votre cluster automatiquement
- **Enrichir** les logs avec les métadonnées Kubernetes
- **Rechercher** instantanément dans des millions de lignes
- **Visualiser** les patterns et tendances
- **Alerter** sur les anomalies

Cette base solide de logging centralisé est le deuxième pilier de votre observabilité. Combinée avec les métriques Prometheus du chapitre précédent, vous avez maintenant une vision profonde de votre infrastructure.

### Points clés à retenir

1. **Fluent Bit** est idéal pour les environnements contraints en ressources
2. **Les logs structurés JSON** simplifient grandement le parsing et l'analyse
3. **L'enrichissement avec les métadonnées Kubernetes** est automatique et précieux
4. **Kibana** offre des capacités de recherche et visualisation puissantes
5. **La gestion de la rétention** est cruciale pour contrôler l'utilisation du stockage

### Prochaine étape : Correlation avec Loki

Dans la section suivante, nous explorerons Loki comme alternative légère à Elasticsearch, particulièrement son intégration native avec Grafana qui permet une corrélation naturelle entre métriques et logs dans une interface unifiée.

La vraie puissance de l'observabilité émerge quand vous pouvez naviguer fluidement entre métriques, logs et traces. Préparez-vous à découvrir comment Loki simplifie cette corrélation tout en réduisant drastiquement les besoins en ressources.

⏭️

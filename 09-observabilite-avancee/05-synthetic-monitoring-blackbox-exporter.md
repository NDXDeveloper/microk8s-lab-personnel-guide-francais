🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.5 Synthetic monitoring avec Blackbox exporter

## Introduction : La surveillance proactive

Jusqu'à présent, nous avons construit une observabilité réactive : nous collectons des métriques, logs et traces générés par de vraies interactions utilisateur. Mais que se passe-t-il quand personne n'utilise votre application ? Comment savoir si elle fonctionne toujours ? C'est là qu'intervient le monitoring synthétique.

Le monitoring synthétique, c'est comme avoir un robot qui teste continuellement votre application en simulant des actions utilisateur. Au lieu d'attendre qu'un vrai utilisateur découvre que votre site est down, votre robot vous alerte immédiatement. C'est la différence entre découvrir un problème quand un client se plaint et le détecter avant que quiconque ne s'en aperçoive.

### Le concept de "synthetic monitoring"

Imaginez que vous gérez une boutique en ligne. Le monitoring traditionnel vous dit combien de clients sont venus, ce qu'ils ont acheté, s'ils ont rencontré des erreurs. Le monitoring synthétique, c'est comme envoyer un employé toutes les 5 minutes vérifier que :
- La porte s'ouvre bien
- Les lumières fonctionnent
- La caisse enregistreuse répond
- Les produits sont accessibles
- Le terminal de paiement fonctionne

Cette approche proactive vous permet de détecter les problèmes même à 3h du matin quand aucun vrai client n'est présent.

### Blackbox vs Whitebox monitoring

**Whitebox monitoring** : Vous avez accès à l'intérieur du système. Vous mesurez l'utilisation CPU, la mémoire, les requêtes par seconde. C'est ce que nous faisons avec Prometheus et les métriques custom.

**Blackbox monitoring** : Vous testez le système de l'extérieur, comme le ferait un utilisateur. Vous ne savez pas ce qui se passe à l'intérieur, vous vérifiez juste que ça marche du point de vue externe.

Les deux approches sont complémentaires :
- Whitebox vous dit **pourquoi** ça ne marche pas
- Blackbox vous dit **si** ça marche du point de vue utilisateur

## Qu'est-ce que Blackbox Exporter ?

Blackbox Exporter est un outil de l'écosystème Prometheus qui effectue des sondes (probes) sur vos endpoints depuis l'extérieur. Il peut :

- **Vérifier la disponibilité HTTP/HTTPS** : Le site répond-il ? Avec quel code status ?
- **Valider les certificats SSL** : Sont-ils valides ? Vont-ils expirer bientôt ?
- **Tester la connectivité TCP** : Le port est-il ouvert ? La connexion s'établit-elle ?
- **Effectuer des checks ICMP** : Le serveur est-il pingable ?
- **Vérifier les serveurs DNS** : Les résolutions fonctionnent-elles ?
- **Mesurer les temps de réponse** : Combien de temps prend chaque étape ?

Blackbox Exporter transforme ces vérifications en métriques Prometheus, permettant d'alerter sur les problèmes et de visualiser la disponibilité dans Grafana.

## Architecture dans MicroK8s

### Vue d'ensemble

```
                Internet
                    │
                    ▼
            [Ingress Controller]
                    │
    ┌───────────────┼───────────────┐
    │               │               │
    ▼               ▼               ▼
[Service A]    [Service B]    [Service C]
    ▲               ▲               ▲
    │               │               │
    └───────────────┴───────────────┘
                    │
            [Blackbox Exporter]
                    │
                    ▼
              [Prometheus]
                    │
                    ▼
               [Grafana]
```

### Flux de monitoring

1. **Prometheus** configure les cibles à monitorer
2. **Blackbox Exporter** effectue les probes selon la configuration
3. Les résultats sont exposés comme métriques Prometheus
4. **Prometheus** scrape ces métriques régulièrement
5. **Grafana** visualise la disponibilité et les performances
6. **AlertManager** envoie des alertes si nécessaire

## Installation de Blackbox Exporter

### Déploiement sur MicroK8s

```yaml
# blackbox-exporter-deployment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: blackbox-config
  namespace: monitoring
data:
  blackbox.yml: |
    modules:
      # Module HTTP basique
      http_2xx:
        prober: http
        timeout: 5s
        http:
          preferred_ip_protocol: "ip4"
          valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
          valid_status_codes: [200, 201, 202, 203, 204, 301, 302, 303, 307, 308]
          method: GET
          follow_redirects: true
          fail_if_ssl: false
          fail_if_not_ssl: false

      # Module HTTPS avec validation SSL
      https_2xx:
        prober: http
        timeout: 5s
        http:
          preferred_ip_protocol: "ip4"
          valid_status_codes: [200, 201, 202, 203, 204]
          method: GET
          fail_if_not_ssl: true
          tls_config:
            insecure_skip_verify: false

      # Module pour API avec authentification
      http_api_check:
        prober: http
        timeout: 10s
        http:
          valid_status_codes: [200, 401]
          method: GET
          headers:
            Accept: "application/json"
            User-Agent: "Blackbox-Exporter-Monitor"
          basic_auth:
            username: "monitoring"
            password: "secret"

      # Module POST avec body
      http_post_2xx:
        prober: http
        timeout: 5s
        http:
          method: POST
          headers:
            Content-Type: "application/json"
          body: '{"test": "probe"}'
          valid_status_codes: [200, 201, 202]

      # Module TCP
      tcp_connect:
        prober: tcp
        timeout: 5s
        tcp:
          preferred_ip_protocol: "ip4"

      # Module TCP avec TLS
      tcp_tls_connect:
        prober: tcp
        timeout: 5s
        tcp:
          tls: true
          tls_config:
            insecure_skip_verify: false

      # Module ICMP (ping)
      icmp:
        prober: icmp
        timeout: 5s
        icmp:
          preferred_ip_protocol: "ip4"

      # Module DNS
      dns_probe:
        prober: dns
        timeout: 5s
        dns:
          preferred_ip_protocol: "ip4"
          valid_rcodes:
          - NOERROR
          query_name: "example.com"
          query_type: "A"

      # Module pour vérifier le contenu
      http_content_check:
        prober: http
        timeout: 10s
        http:
          valid_status_codes: [200]
          fail_if_body_not_matches_regexp:
          - "Welcome to our site"
          fail_if_body_matches_regexp:
          - "Error"
          - "Exception"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blackbox-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blackbox-exporter
  template:
    metadata:
      labels:
        app: blackbox-exporter
    spec:
      containers:
      - name: blackbox-exporter
        image: prom/blackbox-exporter:v0.24.0
        args:
        - --config.file=/config/blackbox.yml
        - --log.level=info
        - --web.listen-address=:9115
        ports:
        - containerPort: 9115
          name: http
        resources:
          requests:
            cpu: 50m
            memory: 50Mi
          limits:
            cpu: 200m
            memory: 200Mi
        volumeMounts:
        - name: config
          mountPath: /config
        livenessProbe:
          httpGet:
            path: /health
            port: 9115
          initialDelaySeconds: 30
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 9115
          initialDelaySeconds: 5
          periodSeconds: 10
      volumes:
      - name: config
        configMap:
          name: blackbox-config
---
apiVersion: v1
kind: Service
metadata:
  name: blackbox-exporter
  namespace: monitoring
  labels:
    app: blackbox-exporter
spec:
  selector:
    app: blackbox-exporter
  ports:
  - name: http
    port: 9115
    targetPort: 9115
```

### Configuration pour privilèges ICMP

Pour utiliser les probes ICMP (ping), Blackbox Exporter a besoin de privilèges supplémentaires :

```yaml
# blackbox-exporter-privileged.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blackbox-exporter
  namespace: monitoring
spec:
  template:
    spec:
      containers:
      - name: blackbox-exporter
        securityContext:
          capabilities:
            add:
            - NET_RAW  # Nécessaire pour ICMP
          allowPrivilegeEscalation: false
          runAsNonRoot: false  # ICMP nécessite root
          runAsUser: 0
```

## Configuration de Prometheus

### Ajout des jobs de scraping

```yaml
# prometheus-blackbox-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    scrape_configs:
    # Monitoring de Blackbox Exporter lui-même
    - job_name: 'blackbox-exporter'
      static_configs:
      - targets: ['blackbox-exporter:9115']

    # Probes HTTP/HTTPS externes
    - job_name: 'blackbox-http-external'
      metrics_path: /probe
      params:
        module: [http_2xx]
      static_configs:
      - targets:
        - https://google.com
        - https://github.com
        - https://prometheus.io
        labels:
          environment: external
          probe_type: http
      relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

    # Probes des services internes
    - job_name: 'blackbox-http-internal'
      metrics_path: /probe
      params:
        module: [http_2xx]
      static_configs:
      - targets:
        - http://api-server.default.svc.cluster.local:8080/health
        - http://frontend.default.svc.cluster.local:3000
        - http://grafana.monitoring.svc.cluster.local:3000/api/health
        labels:
          environment: internal
          probe_type: http
      relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

    # Probes TCP
    - job_name: 'blackbox-tcp'
      metrics_path: /probe
      params:
        module: [tcp_connect]
      static_configs:
      - targets:
        - postgresql.database.svc.cluster.local:5432
        - redis.cache.svc.cluster.local:6379
        - rabbitmq.messaging.svc.cluster.local:5672
        labels:
          environment: internal
          probe_type: tcp
      relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

    # Probes ICMP
    - job_name: 'blackbox-icmp'
      metrics_path: /probe
      params:
        module: [icmp]
      static_configs:
      - targets:
        - 8.8.8.8
        - 1.1.1.1
        - gateway.local
        labels:
          probe_type: icmp
      relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

    # Probes DNS
    - job_name: 'blackbox-dns'
      metrics_path: /probe
      params:
        module: [dns_probe]
      static_configs:
      - targets:
        - 8.8.8.8
        - 1.1.1.1
        labels:
          probe_type: dns
      relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

### Service Discovery dynamique

Pour découvrir automatiquement les services Kubernetes à monitorer :

```yaml
# prometheus-kubernetes-discovery.yaml
scrape_configs:
- job_name: 'blackbox-kubernetes-services'
  metrics_path: /probe
  params:
    module: [http_2xx]
  kubernetes_sd_configs:
  - role: service
    namespaces:
      names:
      - default
      - production
  relabel_configs:
  # Seulement les services avec l'annotation
  - source_labels: [__meta_kubernetes_service_annotation_blackbox_probe]
    action: keep
    regex: true
  # Construire l'URL à tester
  - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_service_port_number]
    target_label: __param_target
    regex: (.+);(.+);(.+)
    replacement: http://$2.$1.svc.cluster.local:$3/health
  - target_label: __address__
    replacement: blackbox-exporter:9115
  - source_labels: [__param_target]
    target_label: instance
  # Ajouter les labels Kubernetes
  - source_labels: [__meta_kubernetes_namespace]
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_service_name]
    target_label: kubernetes_service
```

Annotez vos services pour activer le monitoring :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  annotations:
    blackbox.probe: "true"
spec:
  selector:
    app: my-app
  ports:
  - port: 8080
```

## Métriques importantes

### Métriques de base

```promql
# Disponibilité (1 = up, 0 = down)
probe_success

# Durée totale de la probe en secondes
probe_duration_seconds

# Statut HTTP retourné
probe_http_status_code

# Version HTTP utilisée
probe_http_version

# Taille de la réponse en bytes
probe_http_content_length

# Temps de résolution DNS
probe_dns_lookup_time_seconds

# Temps d'établissement de connexion TCP
probe_tcp_connect_duration_seconds

# Information sur le certificat SSL
probe_ssl_earliest_cert_expiry

# RTT pour ICMP
probe_icmp_duration_seconds
```

### Phases détaillées d'une requête HTTP

```promql
# Résolution DNS
probe_http_duration_seconds{phase="resolve"}

# Connexion TCP
probe_http_duration_seconds{phase="connect"}

# Handshake TLS
probe_http_duration_seconds{phase="tls"}

# Envoi de la requête
probe_http_duration_seconds{phase="processing"}

# Transfert de la réponse
probe_http_duration_seconds{phase="transfer"}
```

## Dashboards Grafana

### Dashboard de disponibilité

```json
{
  "dashboard": {
    "title": "Blackbox Monitoring - Service Availability",
    "panels": [
      {
        "title": "Overall Availability",
        "type": "stat",
        "targets": [{
          "expr": "(sum(rate(probe_success[5m])) / count(probe_success)) * 100"
        }],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {"color": "red", "value": null},
                {"color": "yellow", "value": 95},
                {"color": "green", "value": 99}
              ]
            }
          }
        }
      },
      {
        "title": "Service Status",
        "type": "table",
        "targets": [{
          "expr": "probe_success",
          "format": "table",
          "instant": true
        }],
        "transformations": [{
          "id": "organize",
          "options": {
            "excludeByName": {"Time": true},
            "renameByName": {
              "instance": "Service",
              "Value": "Status"
            }
          }
        }]
      },
      {
        "title": "Response Time by Service",
        "type": "graph",
        "targets": [{
          "expr": "probe_duration_seconds",
          "legendFormat": "{{instance}}"
        }],
        "yaxes": [{
          "format": "s",
          "label": "Response Time"
        }]
      },
      {
        "title": "HTTP Status Codes",
        "type": "graph",
        "targets": [{
          "expr": "probe_http_status_code",
          "legendFormat": "{{instance}} - {{status_code}}"
        }]
      },
      {
        "title": "SSL Certificate Expiry",
        "type": "table",
        "targets": [{
          "expr": "(probe_ssl_earliest_cert_expiry - time()) / 86400",
          "format": "table",
          "instant": true
        }],
        "fieldConfig": {
          "defaults": {
            "unit": "days",
            "custom": {
              "displayMode": "color-background",
              "coloring": {
                "mode": "thresholds"
              }
            },
            "thresholds": {
              "steps": [
                {"color": "red", "value": null},
                {"color": "yellow", "value": 7},
                {"color": "green", "value": 30}
              ]
            }
          }
        }
      }
    ]
  }
}
```

### Dashboard de performance

```json
{
  "panels": [
    {
      "title": "Response Time Breakdown",
      "type": "graph",
      "targets": [
        {
          "expr": "probe_http_duration_seconds{phase=\"resolve\"}",
          "legendFormat": "DNS - {{instance}}"
        },
        {
          "expr": "probe_http_duration_seconds{phase=\"connect\"}",
          "legendFormat": "Connect - {{instance}}"
        },
        {
          "expr": "probe_http_duration_seconds{phase=\"tls\"}",
          "legendFormat": "TLS - {{instance}}"
        },
        {
          "expr": "probe_http_duration_seconds{phase=\"processing\"}",
          "legendFormat": "Processing - {{instance}}"
        },
        {
          "expr": "probe_http_duration_seconds{phase=\"transfer\"}",
          "legendFormat": "Transfer - {{instance}}"
        }
      ],
      "stack": true
    },
    {
      "title": "Latency Heatmap",
      "type": "heatmap",
      "targets": [{
        "expr": "sum(rate(probe_duration_seconds[5m])) by (instance)",
        "format": "heatmap"
      }]
    },
    {
      "title": "Availability Over Time",
      "type": "graph",
      "targets": [{
        "expr": "avg_over_time(probe_success[1h]) * 100",
        "legendFormat": "{{instance}}"
      }],
      "yaxes": [{
        "format": "percent",
        "min": 0,
        "max": 100
      }]
    }
  ]
}
```

## Alertes

### Règles d'alerte essentielles

```yaml
# prometheus-alerts.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-alerts
  namespace: monitoring
data:
  alerts.yml: |
    groups:
    - name: blackbox_alerts
      interval: 30s
      rules:

      # Service down
      - alert: ServiceDown
        expr: probe_success == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} has been down for more than 2 minutes"

      # Haute latence
      - alert: HighLatency
        expr: probe_duration_seconds > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency for {{ $labels.instance }}"
          description: "{{ $labels.instance }} has latency of {{ $value }}s (threshold: 2s)"

      # Certificat SSL expire bientôt
      - alert: SSLCertificateExpiringSoon
        expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "SSL certificate expiring soon for {{ $labels.instance }}"
          description: "SSL certificate for {{ $labels.instance }} expires in {{ $value }} days"

      - alert: SSLCertificateExpiryCritical
        expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 7
        for: 1h
        labels:
          severity: critical
        annotations:
          summary: "SSL certificate expiring very soon for {{ $labels.instance }}"
          description: "SSL certificate for {{ $labels.instance }} expires in {{ $value }} days"

      # Erreurs HTTP
      - alert: HTTPErrors
        expr: probe_http_status_code >= 400
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "HTTP errors for {{ $labels.instance }}"
          description: "{{ $labels.instance }} returns HTTP status {{ $value }}"

      # Disponibilité faible
      - alert: LowAvailability
        expr: (sum by (instance) (rate(probe_success[1h])) * 100) < 95
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Low availability for {{ $labels.instance }}"
          description: "{{ $labels.instance }} availability is {{ $value }}% (SLA: 95%)"

      # DNS lent
      - alert: SlowDNSResolution
        expr: probe_dns_lookup_time_seconds > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Slow DNS resolution for {{ $labels.instance }}"
          description: "DNS resolution for {{ $labels.instance }} takes {{ $value }}s"

      # Perte de paquets ICMP
      - alert: ICMPPacketLoss
        expr: (rate(probe_icmp_duration_seconds_count[5m]) - rate(probe_icmp_duration_seconds_sum[5m])) > 0.1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "ICMP packet loss to {{ $labels.instance }}"
          description: "Packet loss detected when pinging {{ $labels.instance }}"
```

## Cas d'usage avancés

### Multi-step monitoring

Pour tester des workflows complets :

```yaml
# blackbox-multi-step.yaml
modules:
  user_login_flow:
    prober: http
    timeout: 30s
    http:
      # Step 1: Page de login
      valid_status_codes: [200]
      method: GET
      fail_if_body_not_matches_regexp:
      - '<form.*login.*</form>'

  user_login_post:
    prober: http
    timeout: 30s
    http:
      # Step 2: Soumission du login
      method: POST
      headers:
        Content-Type: "application/x-www-form-urlencoded"
      body: "username=test&password=test123"
      valid_status_codes: [302]  # Redirection après login

  user_dashboard:
    prober: http
    timeout: 30s
    http:
      # Step 3: Vérifier le dashboard
      method: GET
      valid_status_codes: [200]
      fail_if_body_not_matches_regexp:
      - 'Welcome.*Dashboard'
```

Script pour orchestrer les tests :

```bash
#!/bin/bash
# test-user-flow.sh

BLACKBOX_URL="http://blackbox-exporter:9115"
TARGET="https://myapp.example.com"

# Step 1: Login page
curl -s "${BLACKBOX_URL}/probe?target=${TARGET}/login&module=user_login_flow" | \
  grep "probe_success 1" || exit 1

# Step 2: Submit login
curl -s "${BLACKBOX_URL}/probe?target=${TARGET}/login&module=user_login_post" | \
  grep "probe_success 1" || exit 1

# Step 3: Dashboard
curl -s "${BLACKBOX_URL}/probe?target=${TARGET}/dashboard&module=user_dashboard" | \
  grep "probe_success 1" || exit 1

echo "User flow test passed!"
```

### Monitoring géographiquement distribué

Déployez plusieurs instances de Blackbox Exporter dans différentes zones :

```yaml
# blackbox-multi-region.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blackbox-exporter-eu
  namespace: monitoring
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: blackbox-exporter
        region: eu-west
    spec:
      nodeSelector:
        region: eu-west
      # ... reste de la config
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blackbox-exporter-us
  namespace: monitoring
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: blackbox-exporter
        region: us-east
    spec:
      nodeSelector:
        region: us-east
      # ... reste de la config
```

Requêtes Prometheus pour comparer les régions :

```promql
# Latence par région
avg by (region) (probe_duration_seconds)

# Disponibilité par région
avg by (region, instance) (probe_success)

# Différence de latence entre régions
max(probe_duration_seconds{region="us-east"}) - max(probe_duration_seconds{region="eu-west"})
```

### Monitoring de contenu et SEO

```yaml
modules:
  seo_check:
    prober: http
    timeout: 30s
    http:
      valid_status_codes: [200]
      # Vérifier les meta tags
      fail_if_body_not_matches_regexp:
      - '<title>.*</title>'
      - '<meta name="description"'
      - '<meta property="og:title"'
      # Vérifier l'absence d'erreurs
      fail_if_body_matches_regexp:
      - '404 Not Found'
      - 'Error'
      - 'Exception'
      # Vérifier la taille de page
      fail_if_header_not_matches:
      - header: Content-Length
        regexp: '^[0-9]{4,}$'  # Au moins 1000 bytes
```

## Optimisations pour un lab

### Configuration légère pour ressources limitées

```yaml
# blackbox-light-config.yaml
modules:
  # Module simplifié pour lab
  http_basic:
    prober: http
    timeout: 3s  # Timeout court
    http:
      preferred_ip_protocol: "ip4"
      valid_status_codes: [200]
      method: GET
      no_follow_redirects: true  # Éviter les redirections

# Prometheus config optimisée
scrape_configs:
- job_name: 'blackbox-minimal'
  scrape_interval: 60s  # Intervalle plus long
  scrape_timeout: 10s
  metrics_path: /probe
  params:
    module: [http_basic]
  static_configs:
  - targets:
    - http://app.local/health  # Seulement les endpoints critiques
```

### Cache des résultats

Pour réduire la charge, implémentez un cache :

```python
#!/usr/bin/env python3
# cached-prober.py

import time
import requests
from flask import Flask, request, jsonify
from functools import lru_cache
from threading import Lock

app = Flask(__name__)
cache_lock = Lock()
cache = {}

@app.route('/probe')
def probe():
    target = request.args.get('target')
    module = request.args.get('module', 'http_2xx')

    # Clé de cache
    cache_key = f"{target}:{module}"
    current_time = time.time()

    with cache_lock:
        # Vérifier le cache (30 secondes de TTL)
        if cache_key in cache:
            cached_result, cached_time = cache[cache_key]
            if current_time - cached_time < 30:
                return cached_result

        # Effectuer la probe
        response = requests.get(
            f"http://blackbox-exporter:9115/probe",
            params={'target': target, 'module': module}
        )

        # Mettre en cache
        cache[cache_key] = (response.text, current_time)

        # Nettoyer le cache (garder max 100 entrées)
        if len(cache) > 100:
            oldest_key = min(cache.keys(), key=lambda k: cache[k][1])
            del cache[oldest_key]

        return response.text

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=9116)
```

### Réduction du nombre de probes

Groupez les vérifications similaires :

```yaml
# prometheus-grouped-probes.yaml
scrape_configs:
- job_name: 'blackbox-critical-only'
  scrape_interval: 30s
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
  - targets:
    # Seulement les endpoints vraiment critiques
    - http://api.local/health
    - http://frontend.local/
    labels:
      priority: critical

- job_name: 'blackbox-non-critical'
  scrape_interval: 5m  # Intervalle beaucoup plus long
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
  - targets:
    - http://admin.local/
    - http://metrics.local/
    labels:
      priority: low
```

## Troubleshooting

### Problèmes courants et solutions

#### Probe failures inexpliqués

```bash
# Vérifier les logs de Blackbox Exporter
kubectl logs -n monitoring deployment/blackbox-exporter

# Tester manuellement une probe
curl "http://blackbox-exporter:9115/probe?target=https://example.com&module=http_2xx&debug=true"

# Vérifier la résolution DNS depuis le pod
kubectl exec -n monitoring deployment/blackbox-exporter -- nslookup example.com

# Vérifier la connectivité réseau
kubectl exec -n monitoring deployment/blackbox-exporter -- wget -O- https://example.com
```

#### Timeouts fréquents

```yaml
# Augmenter les timeouts
modules:
  http_longer_timeout:
    prober: http
    timeout: 30s  # Augmenter le timeout
    http:
      valid_status_codes: [200]
      # Timeout spécifique pour certaines phases
      timeout: 25s
```

#### Certificats SSL auto-signés

```yaml
# Module pour certificats auto-signés
modules:
  https_self_signed:
    prober: http
    http:
      valid_status_codes: [200]
      tls_config:
        insecure_skip_verify: true  # Ignorer la validation SSL
```

### Scripts de diagnostic

```bash
#!/bin/bash
# diagnose-blackbox.sh

echo "=== Blackbox Exporter Diagnostics ==="

# Vérifier que Blackbox est running
echo "Checking Blackbox Exporter status..."
kubectl get pods -n monitoring -l app=blackbox-exporter

# Tester la santé de Blackbox
echo "Testing Blackbox health..."
curl -s http://blackbox-exporter:9115/health || echo "Health check failed"

# Lister les modules disponibles
echo "Available modules:"
curl -s http://blackbox-exporter:9115/config | grep "prober:"

# Tester une probe simple
echo "Testing a simple probe..."
curl -s "http://blackbox-exporter:9115/probe?target=http://google.com&module=http_2xx" | grep probe_success

# Vérifier les métriques dans Prometheus
echo "Checking Prometheus for Blackbox metrics..."
curl -s http://prometheus:9090/api/v1/query?query=up{job=\"blackbox-exporter\"} | jq '.data.result[0].value[1]'

echo "=== Diagnostics complete ==="
```

## Intégration avec l'écosystème d'observabilité

### Corrélation avec les métriques applicatives

```promql
# Comparer la disponibilité externe vs interne
probe_success{environment="external"}
  vs
up{job="api-server"}

# Corréler latence externe avec charge interne
rate(probe_duration_seconds[5m])
  vs
rate(http_requests_total[5m])

# Impact des déploiements sur la disponibilité
probe_success
  and on()
changes(kube_deployment_spec_replicas[5m]) > 0
```

### Enrichissement des logs avec le contexte de probe

```python
# logger-with-probe-context.py
import logging
import requests
from datetime import datetime

class ProbeLogger:
    def __init__(self):
        self.logger = logging.getLogger(__name__)

    def log_probe_result(self, target, success, duration, details=None):
        """Log les résultats de probe avec contexte"""
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'type': 'synthetic_probe',
            'target': target,
            'success': success,
            'duration_ms': duration * 1000,
            'details': details or {}
        }

        if success:
            self.logger.info(f"Probe successful: {target}", extra=log_entry)
        else:
            self.logger.error(f"Probe failed: {target}", extra=log_entry)

        # Envoyer aussi à Loki
        self._send_to_loki(log_entry)

    def _send_to_loki(self, log_entry):
        """Envoie les logs de probe à Loki"""
        loki_url = "http://loki:3100/loki/api/v1/push"
        payload = {
            "streams": [{
                "stream": {
                    "job": "blackbox-probe-logs",
                    "target": log_entry['target']
                },
                "values": [[
                    str(int(datetime.utcnow().timestamp() * 1e9)),
                    str(log_entry)
                ]]
            }]
        }
        requests.post(loki_url, json=payload)
```

### Dashboard unifié dans Grafana

```json
{
  "dashboard": {
    "title": "Unified Observability with Synthetic Monitoring",
    "panels": [
      {
        "title": "Service Health Overview",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "probe_success",
            "datasource": "Prometheus"
          }
        ]
      },
      {
        "title": "Recent Probe Logs",
        "type": "logs",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 4},
        "targets": [
          {
            "expr": "{job=\"blackbox-probe-logs\"}",
            "datasource": "Loki"
          }
        ]
      },
      {
        "title": "Trace Analysis for Failed Probes",
        "type": "traces",
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 4},
        "targets": [
          {
            "query": "service=\"synthetic-monitor\" error=true",
            "datasource": "Jaeger"
          }
        ]
      },
      {
        "title": "Correlation: External vs Internal Latency",
        "type": "graph",
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 12},
        "targets": [
          {
            "expr": "probe_duration_seconds",
            "legendFormat": "External - {{instance}}",
            "datasource": "Prometheus"
          },
          {
            "expr": "histogram_quantile(0.95, http_request_duration_seconds_bucket)",
            "legendFormat": "Internal P95",
            "datasource": "Prometheus"
          }
        ]
      }
    ]
  }
}
```

## Patterns avancés de monitoring synthétique

### API Testing avec validation de contenu

```yaml
# Module pour tester une API REST complète
modules:
  api_health_check:
    prober: http
    timeout: 10s
    http:
      valid_status_codes: [200]
      headers:
        Accept: application/json
      fail_if_body_not_matches_regexp:
      - '"status":\s*"healthy"'
      - '"database":\s*"connected"'
      - '"cache":\s*"connected"'

  api_data_validation:
    prober: http
    timeout: 10s
    http:
      valid_status_codes: [200]
      headers:
        Accept: application/json
      # Valider la structure JSON
      fail_if_body_not_matches_regexp:
      - '"data":\s*\['
      - '"total":\s*[0-9]+'
      - '"page":\s*[0-9]+'
```

### Monitoring de SLA/SLO

```promql
# Calcul de disponibilité sur 30 jours
(
  sum(increase(probe_success[30d]))
  /
  sum(increase(probe_success[30d])) + sum(increase(probe_success{probe_success="0"}[30d]))
) * 100

# SLO: 99.9% de disponibilité
(avg_over_time(probe_success[30d]) * 100) >= 99.9

# Budget d'erreur restant
(1 - 0.999) * 30 * 24 * 60 -
(count(probe_success == 0) * scrape_interval / 60)

# Temps de réponse P95 < 2 secondes
histogram_quantile(0.95, rate(probe_duration_seconds[5m])) < 2
```

### Chaos Engineering avec Blackbox

Script pour injecter des pannes et vérifier la détection :

```python
#!/usr/bin/env python3
# chaos-testing.py

import time
import random
import subprocess
import requests

class ChaosTest:
    def __init__(self, blackbox_url="http://blackbox-exporter:9115"):
        self.blackbox_url = blackbox_url
        self.targets = [
            "http://app.local/health",
            "http://api.local/status"
        ]

    def inject_network_delay(self, delay_ms=1000):
        """Injecte un délai réseau"""
        print(f"Injecting {delay_ms}ms network delay...")
        subprocess.run([
            "kubectl", "exec", "-n", "default", "deployment/app",
            "--", "tc", "qdisc", "add", "dev", "eth0", "root",
            "netem", "delay", f"{delay_ms}ms"
        ])

    def remove_network_delay(self):
        """Supprime le délai réseau"""
        print("Removing network delay...")
        subprocess.run([
            "kubectl", "exec", "-n", "default", "deployment/app",
            "--", "tc", "qdisc", "del", "dev", "eth0", "root"
        ])

    def kill_pod(self, deployment):
        """Tue un pod aléatoire"""
        print(f"Killing random pod from {deployment}...")
        subprocess.run([
            "kubectl", "delete", "pod", "-n", "default",
            "-l", f"app={deployment}",
            "--grace-period=0", "--force"
        ])

    def verify_detection(self, expected_failure=True):
        """Vérifie que Blackbox détecte le problème"""
        time.sleep(10)  # Attendre que Blackbox probe

        for target in self.targets:
            response = requests.get(
                f"{self.blackbox_url}/probe",
                params={"target": target, "module": "http_2xx"}
            )

            # Parser la métrique probe_success
            for line in response.text.split('\n'):
                if line.startswith('probe_success'):
                    success = float(line.split()[1])

                    if expected_failure and success == 1:
                        print(f"❌ Failed to detect issue for {target}")
                        return False
                    elif not expected_failure and success == 0:
                        print(f"❌ False positive for {target}")
                        return False

        print("✅ Detection working correctly")
        return True

    def run_chaos_test(self):
        """Exécute une série de tests chaos"""
        tests = [
            ("Network Delay", self.inject_network_delay, self.remove_network_delay),
            ("Pod Failure", lambda: self.kill_pod("app"), lambda: time.sleep(30)),
        ]

        for test_name, inject_func, cleanup_func in tests:
            print(f"\n=== Testing: {test_name} ===")

            # Vérifier l'état initial
            if not self.verify_detection(expected_failure=False):
                print("Initial state check failed")
                continue

            # Injecter le chaos
            inject_func()

            # Vérifier la détection
            if not self.verify_detection(expected_failure=True):
                print(f"Failed to detect {test_name}")

            # Nettoyer
            cleanup_func()

            # Vérifier le retour à la normale
            time.sleep(30)
            if not self.verify_detection(expected_failure=False):
                print(f"Failed to recover from {test_name}")

if __name__ == "__main__":
    chaos = ChaosTest()
    chaos.run_chaos_test()
```

## Scripts d'automatisation

### Déploiement automatisé

```bash
#!/bin/bash
# deploy-blackbox-monitoring.sh

set -e

NAMESPACE="monitoring"

echo "🚀 Deploying Blackbox Exporter monitoring stack..."

# Créer le namespace
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

# Déployer Blackbox Exporter
echo "📦 Deploying Blackbox Exporter..."
kubectl apply -f blackbox-exporter-deployment.yaml

# Attendre que Blackbox soit prêt
echo "⏳ Waiting for Blackbox Exporter to be ready..."
kubectl wait --for=condition=available --timeout=300s \
  deployment/blackbox-exporter -n $NAMESPACE

# Configurer Prometheus
echo "⚙️ Configuring Prometheus..."
kubectl apply -f prometheus-blackbox-config.yaml

# Recharger Prometheus
echo "🔄 Reloading Prometheus configuration..."
kubectl rollout restart deployment/prometheus -n $NAMESPACE

# Importer les dashboards Grafana
echo "📊 Importing Grafana dashboards..."
for dashboard in dashboards/*.json; do
  curl -X POST \
    -H "Content-Type: application/json" \
    -d @"$dashboard" \
    http://admin:admin@grafana.monitoring.svc.cluster.local:3000/api/dashboards/db
done

# Configurer les alertes
echo "🚨 Setting up alerts..."
kubectl apply -f prometheus-alerts.yaml

echo "✅ Blackbox monitoring stack deployed successfully!"

# Afficher les URLs
echo ""
echo "📋 Access URLs:"
echo "  Blackbox Exporter: http://localhost:9115"
echo "  Prometheus: http://localhost:9090"
echo "  Grafana: http://localhost:3000"

# Test de smoke
echo ""
echo "🧪 Running smoke test..."
curl -s "http://localhost:9115/probe?target=https://google.com&module=http_2xx" | \
  grep "probe_success 1" && echo "✅ Smoke test passed!" || echo "❌ Smoke test failed!"
```

### Rapport automatique de disponibilité

```python
#!/usr/bin/env python3
# availability-report.py

import requests
from datetime import datetime, timedelta
import json
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

class AvailabilityReporter:
    def __init__(self, prometheus_url="http://prometheus:9090"):
        self.prometheus_url = prometheus_url

    def query_prometheus(self, query, start, end):
        """Exécute une requête Prometheus"""
        response = requests.get(
            f"{self.prometheus_url}/api/v1/query_range",
            params={
                'query': query,
                'start': start.timestamp(),
                'end': end.timestamp(),
                'step': '1h'
            }
        )
        return response.json()['data']['result']

    def calculate_availability(self, service, days=7):
        """Calcule la disponibilité d'un service"""
        end = datetime.now()
        start = end - timedelta(days=days)

        query = f'avg_over_time(probe_success{{instance="{service}"}}[{days}d])'
        result = self.query_prometheus(query, start, end)

        if result:
            availability = float(result[0]['value'][1]) * 100
            return availability
        return 0

    def generate_report(self, days=7):
        """Génère un rapport de disponibilité"""
        services = self.get_monitored_services()
        report = {
            'period': f"Last {days} days",
            'generated': datetime.now().isoformat(),
            'services': {}
        }

        for service in services:
            availability = self.calculate_availability(service, days)

            # Récupérer les incidents
            incidents_query = f'count(probe_success{{instance="{service}"}} == 0)'
            incidents = self.query_prometheus(
                incidents_query,
                datetime.now() - timedelta(days=days),
                datetime.now()
            )

            report['services'][service] = {
                'availability': f"{availability:.2f}%",
                'sla_met': availability >= 99.9,
                'incidents': len(incidents) if incidents else 0,
                'status': 'Healthy' if availability >= 99.9 else 'Degraded'
            }

        return report

    def get_monitored_services(self):
        """Récupère la liste des services monitorés"""
        query = 'group by (instance) (probe_success)'
        response = requests.get(
            f"{self.prometheus_url}/api/v1/query",
            params={'query': query}
        )

        services = []
        for result in response.json()['data']['result']:
            services.append(result['metric']['instance'])

        return services

    def send_email_report(self, report, recipient):
        """Envoie le rapport par email"""
        html = self.format_html_report(report)

        msg = MIMEMultipart('alternative')
        msg['Subject'] = f"Availability Report - {report['period']}"
        msg['From'] = "monitoring@lab.local"
        msg['To'] = recipient

        msg.attach(MIMEText(html, 'html'))

        # Envoyer l'email
        with smtplib.SMTP('localhost', 25) as server:
            server.send_message(msg)

    def format_html_report(self, report):
        """Formate le rapport en HTML"""
        html = f"""
        <html>
        <head>
            <style>
                table {{ border-collapse: collapse; width: 100%; }}
                th, td {{ border: 1px solid #ddd; padding: 8px; text-align: left; }}
                th {{ background-color: #4CAF50; color: white; }}
                .healthy {{ color: green; }}
                .degraded {{ color: red; }}
            </style>
        </head>
        <body>
            <h2>Availability Report</h2>
            <p>Period: {report['period']}</p>
            <p>Generated: {report['generated']}</p>

            <table>
                <tr>
                    <th>Service</th>
                    <th>Availability</th>
                    <th>SLA Met</th>
                    <th>Incidents</th>
                    <th>Status</th>
                </tr>
        """

        for service, data in report['services'].items():
            status_class = 'healthy' if data['sla_met'] else 'degraded'
            html += f"""
                <tr>
                    <td>{service}</td>
                    <td>{data['availability']}</td>
                    <td>{'✅' if data['sla_met'] else '❌'}</td>
                    <td>{data['incidents']}</td>
                    <td class="{status_class}">{data['status']}</td>
                </tr>
            """

        html += """
            </table>
        </body>
        </html>
        """

        return html

if __name__ == "__main__":
    reporter = AvailabilityReporter()
    report = reporter.generate_report(days=7)

    # Afficher le rapport
    print(json.dumps(report, indent=2))

    # Optionnel : envoyer par email
    # reporter.send_email_report(report, "admin@lab.local")
```

## Conclusion

Le monitoring synthétique avec Blackbox Exporter complète votre observabilité en ajoutant une dimension proactive. Vous ne découvrez plus les problèmes quand les utilisateurs se plaignent, vous les détectez avant qu'ils n'impactent quiconque.

### Points clés à retenir

1. **Le monitoring synthétique est complémentaire** : Il ne remplace pas les métriques internes mais les complète avec une vue externe
2. **La simplicité est clé** : Commencez avec des checks HTTP simples avant d'ajouter de la complexité
3. **L'automatisation est essentielle** : Les probes doivent tourner 24/7 sans intervention manuelle
4. **La corrélation multiplie la valeur** : Combinez les données de Blackbox avec vos métriques, logs et traces
5. **Les alertes doivent être actionnables** : Une alerte de Blackbox devrait déclencher une action claire

### Architecture complète d'observabilité

Avec Blackbox Exporter, votre stack d'observabilité est maintenant complète :

- **Prometheus** : Métriques internes (whitebox)
- **Loki** : Logs centralisés
- **Jaeger** : Tracing distribué
- **Blackbox Exporter** : Monitoring externe (blackbox)
- **Grafana** : Visualisation unifiée

Cette combinaison vous donne une vue à 360° de votre infrastructure, de l'intérieur comme de l'extérieur, en temps réel comme historiquement, proactivement comme réactivement.

Dans la prochaine section, nous explorerons les SLI/SLO et dashboards de fiabilité, utilisant toutes ces sources de données pour définir et mesurer objectivement la fiabilité de vos services.

⏭️

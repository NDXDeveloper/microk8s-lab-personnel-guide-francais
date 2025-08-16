🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.6 SLI/SLO et dashboards de fiabilité

## Introduction : Mesurer objectivement la fiabilité

Imaginez que vous gérez un restaurant. Vos clients veulent trois choses simples : être servis rapidement, recevoir la bonne commande, et que le restaurant soit ouvert quand ils arrivent. Comment mesurez-vous si vous répondez à ces attentes ? C'est exactement ce que font les SLI et SLO pour vos services numériques.

### La pyramide de la fiabilité

```
        SLA (Accord légal)
           "99.9% uptime"
              ▲
             ╱ ╲
            ╱   ╲
           ╱     ╲
          ╱  SLO  ╲
         ╱ Objectif ╲
        ╱  "99.95%"  ╲
       ╱──────────────╲
      ╱                ╲
     ╱       SLI        ╲
    ╱    Indicateurs     ╲
   ╱  "Latence < 100ms"   ╲
  ╱────────────────────────╲
 ╱     Métriques brutes     ╲
╱  (Prometheus, Loki, etc.)  ╲
```

**SLI (Service Level Indicator)** : Ce sont les mesures concrètes. "Le temps de réponse de notre API", "Le pourcentage de requêtes réussies". C'est le thermomètre de votre service.

**SLO (Service Level Objective)** : C'est votre objectif interne. "Notre API doit répondre en moins de 100ms pour 99% des requêtes". C'est votre standard de qualité.

**SLA (Service Level Agreement)** : C'est votre promesse contractuelle aux clients. "Nous garantissons 99.9% de disponibilité, sinon remboursement". C'est votre engagement légal.

### Pourquoi c'est crucial dans Kubernetes

Dans un environnement Kubernetes dynamique où les pods vont et viennent, où les déploiements sont fréquents, et où la complexité est élevée, avoir des objectifs clairs et mesurables devient vital. Sans SLI/SLO, vous naviguez à l'aveugle. Avec eux, vous savez exactement où vous en êtes et quand agir.

## Les concepts fondamentaux

### Error Budget : Votre droit à l'erreur

L'Error Budget est révolutionnaire : au lieu de viser 100% de disponibilité (impossible et coûteux), vous acceptez un certain niveau d'erreur. C'est votre "budget" pour innover et prendre des risques.

**Exemple concret** :
- SLO : 99.9% de disponibilité sur 30 jours
- Temps total : 30 jours × 24 heures × 60 minutes = 43,200 minutes
- Error Budget : 0.1% × 43,200 = 43.2 minutes de downtime acceptable
- Si vous avez déjà consommé 40 minutes, vous devez être très prudent pour le reste du mois

### Les Golden Signals de Google

Google SRE définit quatre signaux essentiels pour mesurer la santé d'un service :

1. **Latency** : Temps de réponse des requêtes
2. **Traffic** : Volume de requêtes
3. **Errors** : Taux d'échec
4. **Saturation** : Utilisation des ressources

Ces quatre métriques suffisent pour comprendre 90% des problèmes de production.

### La méthode RED et USE

**RED Method** (pour les services) :
- **Rate** : Requêtes par seconde
- **Errors** : Requêtes échouées par seconde
- **Duration** : Temps de traitement

**USE Method** (pour les ressources) :
- **Utilization** : % de temps occupé
- **Saturation** : Quantité de travail en attente
- **Errors** : Nombre d'erreurs

## Définir vos SLI

### Types de SLI courants

```yaml
# Configuration des SLI types
sli_definitions:
  # Disponibilité
  availability:
    description: "Service répond avec succès"
    formula: "successful_requests / total_requests"
    threshold: 0.999  # 99.9%

  # Latence
  latency:
    description: "Temps de réponse acceptable"
    formula: "requests_under_100ms / total_requests"
    threshold: 0.95  # 95% des requêtes < 100ms

  # Throughput
  throughput:
    description: "Capacité de traitement"
    formula: "processed_requests / time_period"
    threshold: 1000  # 1000 req/s minimum

  # Qualité
  quality:
    description: "Réponses correctes"
    formula: "correct_responses / total_responses"
    threshold: 0.999  # 99.9% de réponses correctes

  # Fraîcheur
  freshness:
    description: "Données à jour"
    formula: "up_to_date_records / total_records"
    threshold: 0.99  # 99% des données < 5 min
```

### Implémentation des SLI avec Prometheus

```promql
# SLI: Disponibilité
# Ratio de requêtes réussies sur 5 minutes
sum(rate(http_requests_total{status!~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# SLI: Latence (95e percentile < 100ms)
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
) < 0.1

# SLI: Taux d'erreur
1 - (
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
)

# SLI: Saturation (utilisation CPU < 80%)
1 - avg(rate(container_cpu_usage_seconds_total[5m])) / 0.8

# SLI: Fraîcheur des données
(time() - max(last_successful_sync_timestamp)) < 300
```

## Définir vos SLO

### Méthodologie pour choisir les bons SLO

```python
# slo_calculator.py
class SLOCalculator:
    """Aide à définir des SLO réalistes basés sur l'historique"""

    def __init__(self, prometheus_url="http://prometheus:9090"):
        self.prometheus = prometheus_url

    def analyze_historical_performance(self, metric_query, days=30):
        """Analyse les performances historiques pour suggérer un SLO"""
        # Récupérer les données historiques
        data = self.query_prometheus(metric_query, days)

        # Calculer les percentiles
        percentiles = {
            'p50': np.percentile(data, 50),
            'p90': np.percentile(data, 90),
            'p95': np.percentile(data, 95),
            'p99': np.percentile(data, 99),
            'p99.9': np.percentile(data, 99.9)
        }

        # Suggérer un SLO basé sur les percentiles actuels
        suggestions = {
            'aggressive': percentiles['p99.9'],  # Top 0.1%
            'balanced': percentiles['p99'],      # Top 1%
            'conservative': percentiles['p95']   # Top 5%
        }

        return {
            'current_performance': percentiles,
            'suggested_slos': suggestions,
            'recommendation': self.recommend_slo(percentiles)
        }

    def recommend_slo(self, percentiles):
        """Recommande un SLO basé sur les patterns observés"""
        # Si p99 et p99.9 sont proches, on peut être agressif
        if (percentiles['p99.9'] - percentiles['p99']) / percentiles['p99'] < 0.1:
            return {
                'target': percentiles['p99.5'],
                'confidence': 'high',
                'rationale': 'Performance stable au 99e percentile'
            }
        else:
            return {
                'target': percentiles['p99'],
                'confidence': 'medium',
                'rationale': 'Variabilité observée au-delà du 99e percentile'
            }
```

### Configuration des SLO

```yaml
# slo-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: slo-definitions
  namespace: monitoring
data:
  slos.yaml: |
    # SLO pour l'API principale
    api_availability:
      service: api-server
      sli: availability
      objective: 99.95  # 99.95% de disponibilité
      window: 30d       # Sur 30 jours glissants
      alert_burn_rate:
        - 1h: 14.4      # Alerte si burn rate > 14.4x sur 1h
        - 6h: 6         # Alerte si burn rate > 6x sur 6h
        - 3d: 1         # Alerte si burn rate > 1x sur 3j

    api_latency:
      service: api-server
      sli: latency_p99
      objective: 100    # 99% des requêtes < 100ms
      window: 7d        # Sur 7 jours glissants

    database_availability:
      service: postgresql
      sli: availability
      objective: 99.99  # 99.99% de disponibilité (HA)
      window: 30d

    cache_hit_rate:
      service: redis
      sli: cache_hit_ratio
      objective: 95     # 95% de cache hit
      window: 1d        # Sur 1 jour glissant
```

## Calcul de l'Error Budget

### Implémentation du calcul

```python
# error_budget.py
from datetime import datetime, timedelta
import requests

class ErrorBudget:
    def __init__(self, slo_target, window_days=30):
        self.slo_target = slo_target  # Ex: 0.999 pour 99.9%
        self.window_days = window_days
        self.window_seconds = window_days * 24 * 3600

    def calculate_budget(self):
        """Calcule le budget d'erreur total"""
        allowed_downtime = (1 - self.slo_target) * self.window_seconds
        return {
            'total_seconds': allowed_downtime,
            'total_minutes': allowed_downtime / 60,
            'total_hours': allowed_downtime / 3600,
            'percentage': (1 - self.slo_target) * 100
        }

    def calculate_remaining(self, consumed_seconds):
        """Calcule le budget restant"""
        total_budget = self.calculate_budget()['total_seconds']
        remaining = total_budget - consumed_seconds

        return {
            'remaining_seconds': remaining,
            'remaining_minutes': remaining / 60,
            'remaining_percentage': (remaining / total_budget) * 100,
            'burn_rate': consumed_seconds / total_budget,
            'status': self.get_status(remaining / total_budget)
        }

    def get_status(self, remaining_ratio):
        """Détermine le statut basé sur le budget restant"""
        if remaining_ratio > 0.5:
            return "🟢 Healthy"
        elif remaining_ratio > 0.2:
            return "🟡 Warning"
        elif remaining_ratio > 0:
            return "🟠 Critical"
        else:
            return "🔴 Exhausted"

    def calculate_burn_rate(self, errors_in_window, window_minutes):
        """Calcule le burn rate actuel"""
        # Burn rate = vitesse de consommation / vitesse normale
        normal_rate = (1 - self.slo_target) * window_minutes
        actual_rate = errors_in_window

        if normal_rate > 0:
            return actual_rate / normal_rate
        return 0
```

### Requêtes Prometheus pour Error Budget

```promql
# Budget d'erreur total pour 30 jours (en secondes)
(1 - 0.999) * 30 * 24 * 3600

# Temps d'indisponibilité consommé sur 30 jours
sum(increase(probe_success{job="blackbox"}[30d] == 0)) * 15  # Si scrape interval = 15s

# Budget restant en pourcentage
100 * (1 -
  sum(increase(probe_success{job="blackbox"}[30d] == 0)) * 15
  /
  ((1 - 0.999) * 30 * 24 * 3600)
)

# Burn rate sur la dernière heure
(
  sum(rate(http_requests_total{status=~"5.."}[1h]))
  /
  sum(rate(http_requests_total[1h]))
) / (1 - 0.999)

# Temps avant épuisement du budget au rythme actuel
((1 - 0.999) * 30 * 24 * 3600 - sum(increase(probe_success{job="blackbox"}[30d] == 0)) * 15)
/
(sum(rate(probe_success{job="blackbox"}[1h] == 0)) * 3600)
```

## Dashboards de fiabilité dans Grafana

### Dashboard SLO Overview

```json
{
  "dashboard": {
    "title": "SLO Overview - Service Reliability",
    "panels": [
      {
        "id": 1,
        "title": "SLO Compliance Status",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
        "targets": [{
          "expr": "avg(slo_compliance_ratio)",
          "legendFormat": "Overall Compliance"
        }],
        "fieldConfig": {
          "defaults": {
            "unit": "percentunit",
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {"color": "red", "value": null},
                {"color": "yellow", "value": 0.95},
                {"color": "green", "value": 0.999}
              ]
            },
            "decimals": 3
          }
        }
      },
      {
        "id": 2,
        "title": "Error Budget Status",
        "type": "gauge",
        "gridPos": {"h": 8, "w": 6, "x": 6, "y": 0},
        "targets": [{
          "expr": "error_budget_remaining_percentage"
        }],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "max": 100,
            "thresholds": {
              "mode": "percentage",
              "steps": [
                {"color": "red", "value": null},
                {"color": "orange", "value": 20},
                {"color": "yellow", "value": 50},
                {"color": "green", "value": 80}
              ]
            }
          }
        }
      },
      {
        "id": 3,
        "title": "Service Availability (30d)",
        "type": "table",
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0},
        "targets": [{
          "expr": "avg_over_time(up[30d]) * 100",
          "format": "table",
          "instant": true
        }],
        "transformations": [
          {
            "id": "organize",
            "options": {
              "excludeByName": {"Time": true},
              "renameByName": {
                "job": "Service",
                "Value": "Availability %"
              }
            }
          },
          {
            "id": "sortBy",
            "options": {
              "fields": {},
              "sort": [{"field": "Availability %", "desc": false}]
            }
          }
        ]
      },
      {
        "id": 4,
        "title": "SLI Trends",
        "type": "graph",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8},
        "targets": [
          {
            "expr": "sli_availability",
            "legendFormat": "Availability"
          },
          {
            "expr": "sli_latency_compliance",
            "legendFormat": "Latency Compliance"
          },
          {
            "expr": "sli_error_rate",
            "legendFormat": "Success Rate"
          }
        ],
        "yaxes": [{
          "format": "percentunit",
          "min": 0.9,
          "max": 1
        }],
        "thresholds": [{
          "value": 0.999,
          "op": "gt",
          "fill": true,
          "line": true,
          "colorMode": "critical"
        }]
      },
      {
        "id": 5,
        "title": "Burn Rate Alert Status",
        "type": "heatmap",
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 8},
        "targets": [{
          "expr": "burn_rate_by_window",
          "format": "heatmap"
        }],
        "dataFormat": "timeseries",
        "cards": {"cardPadding": 2, "cardRound": 2},
        "color": {
          "mode": "spectrum",
          "scheme": "RdYlGn",
          "reverse": true
        }
      }
    ]
  }
}
```

### Dashboard Error Budget

```json
{
  "dashboard": {
    "title": "Error Budget Management",
    "panels": [
      {
        "title": "Error Budget Consumption Rate",
        "type": "graph",
        "targets": [{
          "expr": "rate(error_budget_consumed[1h])",
          "legendFormat": "{{service}} - Hourly Burn"
        }],
        "alert": {
          "conditions": [{
            "evaluator": {"params": [1], "type": "gt"},
            "operator": {"type": "and"},
            "query": {"params": ["A", "5m", "now"]},
            "reducer": {"params": [], "type": "avg"},
            "type": "query"
          }]
        }
      },
      {
        "title": "Time to Budget Exhaustion",
        "type": "stat",
        "targets": [{
          "expr": "error_budget_remaining_seconds / rate(error_budget_consumed[1h])",
          "legendFormat": "Time Remaining"
        }],
        "fieldConfig": {
          "defaults": {
            "unit": "dtdurations",
            "thresholds": {
              "steps": [
                {"color": "red", "value": null},
                {"color": "yellow", "value": 86400},
                {"color": "green", "value": 604800}
              ]
            }
          }
        }
      },
      {
        "title": "Budget Usage by Service",
        "type": "piechart",
        "targets": [{
          "expr": "sum by (service) (error_budget_consumed)",
          "legendFormat": "{{service}}"
        }]
      },
      {
        "title": "Historical Budget Consumption",
        "type": "graph",
        "targets": [
          {
            "expr": "error_budget_consumed",
            "legendFormat": "{{service}} - Consumed"
          },
          {
            "expr": "error_budget_total",
            "legendFormat": "Total Budget"
          }
        ],
        "fill": 1,
        "stack": false,
        "percentage": false
      }
    ]
  }
}
```

### Dashboard Multi-Window Multi-Burn-Rate

```json
{
  "dashboard": {
    "title": "Multi-Window Multi-Burn-Rate Alerts",
    "description": "Based on Google SRE Workbook",
    "panels": [
      {
        "title": "Burn Rate Matrix",
        "type": "table",
        "targets": [{
          "expr": "max by (service, window) (burn_rate)",
          "format": "table"
        }],
        "transformations": [{
          "id": "pivot",
          "options": {
            "pivotField": "window",
            "valueField": "Value",
            "groupByField": "service"
          }
        }],
        "fieldConfig": {
          "defaults": {
            "custom": {
              "displayMode": "color-background",
              "coloring": {
                "mode": "thresholds"
              }
            },
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 2},
                {"color": "orange", "value": 6},
                {"color": "red", "value": 14.4}
              ]
            }
          }
        }
      },
      {
        "title": "Alert Conditions",
        "type": "state-timeline",
        "targets": [
          {
            "expr": "burn_rate{window=\"1h\"} > 14.4",
            "legendFormat": "1h > 14.4x (2% budget/1h)"
          },
          {
            "expr": "burn_rate{window=\"6h\"} > 6",
            "legendFormat": "6h > 6x (5% budget/6h)"
          },
          {
            "expr": "burn_rate{window=\"3d\"} > 1",
            "legendFormat": "3d > 1x (10% budget/3d)"
          }
        ]
      }
    ]
  }
}
```

## Implémentation des SLO dans votre code

### Recording Rules Prometheus

```yaml
# prometheus-recording-rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-recording-rules
  namespace: monitoring
data:
  recording_rules.yml: |
    groups:
    - name: sli_recordings
      interval: 30s
      rules:
      # SLI: Disponibilité sur 5 minutes
      - record: sli:availability:5m
        expr: |
          sum(rate(http_requests_total{status!~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)

      # SLI: Latence P99 sur 5 minutes
      - record: sli:latency_p99:5m
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le)
          )

      # SLI: Taux d'erreur sur 5 minutes
      - record: sli:error_rate:5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)

      # SLO: Compliance de disponibilité
      - record: slo:availability:compliance
        expr: |
          avg_over_time(sli:availability:5m[30d]) >= 0.999

      # Error Budget: Consommé
      - record: error_budget:consumed:ratio
        expr: |
          1 - (
            (1 - avg_over_time(sli:availability:5m[30d]))
            /
            (1 - 0.999)
          )

      # Error Budget: Restant en secondes
      - record: error_budget:remaining:seconds
        expr: |
          (1 - 0.999) * 30 * 24 * 3600
          -
          sum(increase(http_requests_total{status=~"5.."}[30d])) by (service)
          /
          sum(increase(http_requests_total[30d])) by (service)
          * 30 * 24 * 3600

      # Burn Rate: Taux de consommation
      - record: burn_rate:1h
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[1h])) by (service)
            /
            sum(rate(http_requests_total[1h])) by (service)
          )
          /
          (1 - 0.999)

      - record: burn_rate:6h
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[6h])) by (service)
            /
            sum(rate(http_requests_total[6h])) by (service)
          )
          /
          (1 - 0.999)

      - record: burn_rate:3d
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[3d])) by (service)
            /
            sum(rate(http_requests_total[3d])) by (service)
          )
          /
          (1 - 0.999)
```

### Alertes basées sur les SLO

```yaml
# prometheus-slo-alerts.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-slo-alerts
  namespace: monitoring
data:
  alerts.yml: |
    groups:
    - name: slo_alerts
      rules:
      # Alerte Multi-Window Multi-Burn-Rate
      - alert: HighErrorBudgetBurn1h
        expr: |
          burn_rate:1h > 14.4
          and
          burn_rate:5m > 14.4
        for: 2m
        labels:
          severity: critical
          window: 1h
        annotations:
          summary: "High error budget burn rate (1h window)"
          description: "{{ $labels.service }} is consuming error budget 14.4x faster than normal"
          runbook: "https://wiki.lab.local/runbooks/high-burn-rate"

      - alert: HighErrorBudgetBurn6h
        expr: |
          burn_rate:6h > 6
          and
          burn_rate:30m > 6
        for: 15m
        labels:
          severity: warning
          window: 6h
        annotations:
          summary: "Sustained error budget burn (6h window)"
          description: "{{ $labels.service }} burning budget 6x normal rate for 6 hours"

      - alert: ErrorBudgetLow
        expr: error_budget:remaining:seconds < 3600
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Error budget nearly exhausted"
          description: "{{ $labels.service }} has less than 1 hour of error budget remaining"

      - alert: SLOViolation
        expr: slo:availability:compliance == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "SLO violation detected"
          description: "{{ $labels.service }} is not meeting its availability SLO"
```

## Automatisation et reporting

### Générateur de rapports SLO

```python
#!/usr/bin/env python3
# slo_reporter.py

import requests
from datetime import datetime, timedelta
import pandas as pd
import matplotlib.pyplot as plt
from jinja2 import Template

class SLOReporter:
    def __init__(self, prometheus_url="http://prometheus:9090"):
        self.prometheus_url = prometheus_url
        self.report_data = {}

    def query_prometheus(self, query, start, end, step='1h'):
        """Execute une requête Prometheus"""
        response = requests.get(
            f"{self.prometheus_url}/api/v1/query_range",
            params={
                'query': query,
                'start': start.timestamp(),
                'end': end.timestamp(),
                'step': step
            }
        )
        return response.json()['data']['result']

    def calculate_slo_compliance(self, service, slo_target, days=30):
        """Calcule la conformité SLO pour un service"""
        end = datetime.now()
        start = end - timedelta(days=days)

        # Requête pour la disponibilité
        query = f'''
            avg_over_time(
                (sum(rate(http_requests_total{{service="{service}",status!~"5.."}}[5m]))
                /
                sum(rate(http_requests_total{{service="{service}"}}[5m])))
                [{days}d]
            )
        '''

        result = self.query_prometheus(query, start, end)
        if result:
            actual_availability = float(result[0]['value'][1])
            compliance = actual_availability >= slo_target

            return {
                'service': service,
                'slo_target': slo_target,
                'actual': actual_availability,
                'compliance': compliance,
                'gap': actual_availability - slo_target
            }
        return None

    def calculate_error_budget_status(self, service, slo_target, days=30):
        """Calcule le statut du budget d'erreur"""
        end = datetime.now()
        start = end - timedelta(days=days)

        # Total time in seconds
        total_seconds = days * 24 * 3600

        # Allowed downtime
        allowed_downtime = (1 - slo_target) * total_seconds

        # Actual downtime
        query = f'''
            sum(increase(
                http_requests_total{{service="{service}",status=~"5.."}}[{days}d]
            ))
            /
            sum(increase(
                http_requests_total{{service="{service}"}}[{days}d]
            ))
            * {total_seconds}
        '''

        result = self.query_prometheus(query, start, end)
        if result:
            actual_downtime = float(result[0]['value'][1])
            remaining = allowed_downtime - actual_downtime

            return {
                'total_budget': allowed_downtime,
                'consumed': actual_downtime,
                'remaining': remaining,
                'remaining_percentage': (remaining / allowed_downtime) * 100,
                'burn_rate': actual_downtime / allowed_downtime
            }
        return None

    def generate_report(self, services_config):
        """Génère un rapport complet SLO"""
        report = {
            'generated_at': datetime.now().isoformat(),
            'period': '30 days',
            'services': {}
        }

        for service, config in services_config.items():
            slo_target = config['slo_target']

            # Compliance SLO
            compliance = self.calculate_slo_compliance(service, slo_target)

            # Error Budget
            budget = self.calculate_error_budget_status(service, slo_target)

            # Incidents
            incidents = self.get_incidents(service, days=30)

            report['services'][service] = {
                'slo_compliance': compliance,
                'error_budget': budget,
                'incidents': incidents,
                'risk_level': self.assess_risk_level(compliance, budget)
            }

        return report

    def get_incidents(self, service, days=30):
        """Récupère les incidents pour un service"""
        end = datetime.now()
        start = end - timedelta(days=days)

        query = f'''
            count_over_time(
                (sum(rate(http_requests_total{{service="{service}",status=~"5.."}}[5m]))
                /
                sum(rate(http_requests_total{{service="{service}"}}[5m])) > 0.01)
                [{days}d:1h]
            )
        '''

        result = self.query_prometheus(query, start, end)
        incidents = []

        if result:
            for point in result[0]['values']:
                if float(point[1]) > 0:
                    incidents.append({
                        'timestamp': datetime.fromtimestamp(point[0]),
                        'severity': 'high' if float(point[1]) > 5 else 'medium'
                    })

        return {
            'count': len(incidents),
            'details': incidents[:10]  # Limiter aux 10 derniers
        }

    def assess_risk_level(self, compliance, budget):
        """Évalue le niveau de risque"""
        if not compliance or not budget:
            return 'unknown'

        if not compliance['compliance']:
            return 'critical'

        if budget['remaining_percentage'] < 10:
            return 'high'
        elif budget['remaining_percentage'] < 30:
            return 'medium'
        else:
            return 'low'

    def create_visualizations(self, report):
        """Crée des graphiques pour le rapport"""
        fig, axes = plt.subplots(2, 2, figsize=(12, 8))

        # Graphique 1: SLO Compliance
        services = list(report['services'].keys())
        compliances = [s['slo_compliance']['actual'] * 100 for s in report['services'].values()]

        axes[0, 0].bar(services, compliances)
        axes[0, 0].axhline(y=99.9, color='r', linestyle='--', label='SLO Target')
        axes[0, 0].set_title('SLO Compliance (%)')
        axes[0, 0].set_ylim([99, 100])
        axes[0, 0].legend()

        # Graphique 2: Error Budget Remaining
        budgets = [s['error_budget']['remaining_percentage'] for s in report['services'].values()]
        colors = ['green' if b > 50 else 'yellow' if b > 20 else 'red' for b in budgets]

        axes[0, 1].bar(services, budgets, color=colors)
        axes[0, 1].set_title('Error Budget Remaining (%)')
        axes[0, 1].set_ylim([0, 100])

        # Graphique 3: Incidents Count
        incidents = [s['incidents']['count'] for s in report['services'].values()]

        axes[1, 0].bar(services, incidents, color='orange')
        axes[1, 0].set_title('Incidents (30 days)')

        # Graphique 4: Risk Matrix
        risk_levels = [s['risk_level'] for s in report['services'].values()]
        risk_colors = {
            'low': 'green',
            'medium': 'yellow',
            'high': 'orange',
            'critical': 'red',
            'unknown': 'gray'
        }
        colors = [risk_colors[r] for r in risk_levels]

        axes[1, 1].pie([1] * len(services), labels=services, colors=colors, autopct='')
        axes[1, 1].set_title('Risk Assessment')

        plt.tight_layout()
        plt.savefig('slo_report.png')

        return 'slo_report.png'

    def generate_html_report(self, report):
        """Génère un rapport HTML"""
        template = Template('''
        <!DOCTYPE html>
        <html>
        <head>
            <title>SLO Report</title>
            <style>
                body { font-family: Arial, sans-serif; margin: 20px; }
                table { border-collapse: collapse; width: 100%; margin: 20px 0; }
                th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
                th { background-color: #4CAF50; color: white; }
                .compliant { color: green; }
                .non-compliant { color: red; }
                .low { background-color: #d4edda; }
                .medium { background-color: #fff3cd; }
                .high { background-color: #f8d7da; }
                .critical { background-color: #dc3545; color: white; }
            </style>
        </head>
        <body>
            <h1>SLO Compliance Report</h1>
            <p>Generated: {{ report.generated_at }}</p>
            <p>Period: {{ report.period }}</p>

            <h2>Service Status</h2>
            <table>
                <tr>
                    <th>Service</th>
                    <th>SLO Target</th>
                    <th>Actual</th>
                    <th>Compliance</th>
                    <th>Error Budget</th>
                    <th>Incidents</th>
                    <th>Risk</th>
                </tr>
                {% for service, data in report.services.items() %}
                <tr class="{{ data.risk_level }}">
                    <td>{{ service }}</td>
                    <td>{{ "%.3f%%" % (data.slo_compliance.slo_target * 100) }}</td>
                    <td>{{ "%.3f%%" % (data.slo_compliance.actual * 100) }}</td>
                    <td class="{{ 'compliant' if data.slo_compliance.compliance else 'non-compliant' }}">
                        {{ '✓' if data.slo_compliance.compliance else '✗' }}
                    </td>
                    <td>{{ "%.1f%%" % data.error_budget.remaining_percentage }}</td>
                    <td>{{ data.incidents.count }}</td>
                    <td>{{ data.risk_level.upper() }}</td>
                </tr>
                {% endfor %}
            </table>

            <h2>Recommendations</h2>
            <ul>
                {% for service, data in report.services.items() %}
                    {% if data.risk_level in ['high', 'critical'] %}
                    <li><strong>{{ service }}</strong>:
                        {% if data.risk_level == 'critical' %}
                        Immediate action required. SLO violation detected.
                        {% else %}
                        Close monitoring required. Error budget nearly exhausted.
                        {% endif %}
                    </li>
                    {% endif %}
                {% endfor %}
            </ul>

            <img src="slo_report.png" alt="SLO Visualizations" style="max-width: 100%;">
        </body>
        </html>
        ''')

        html = template.render(report=report)

        with open('slo_report.html', 'w') as f:
            f.write(html)

        return 'slo_report.html'

# Utilisation
if __name__ == "__main__":
    reporter = SLOReporter()

    services_config = {
        'api-server': {'slo_target': 0.999},
        'database': {'slo_target': 0.9999},
        'cache': {'slo_target': 0.99}
    }

    report = reporter.generate_report(services_config)
    reporter.create_visualizations(report)
    reporter.generate_html_report(report)

    print("Report generated: slo_report.html")
```

## Runbooks et documentation

### Template de runbook pour alertes SLO

```markdown
# Runbook: High Error Budget Burn Rate

## Alert Name
`HighErrorBudgetBurn1h`

## Description
Le service consomme son budget d'erreur 14.4x plus vite que la normale sur la dernière heure.

## Impact
- **User Impact**: Les utilisateurs peuvent rencontrer des erreurs ou des lenteurs
- **Business Impact**: Risque de violation SLO si la situation persiste
- **Technical Impact**: Épuisement du budget d'erreur en moins de 2 heures au rythme actuel

## Dashboard Links
- [SLO Overview](http://grafana.lab.local/d/slo-overview)
- [Error Budget Status](http://grafana.lab.local/d/error-budget)
- [Service Details](http://grafana.lab.local/d/service-details)

## Investigation Steps

1. **Vérifier l'ampleur du problème**
   ```promql
   # Taux d'erreur actuel
   sum(rate(http_requests_total{service="$service",status=~"5.."}[5m]))
   /
   sum(rate(http_requests_total{service="$service"}[5m]))
   ```

2. **Identifier la source des erreurs**
   ```promql
   # Top endpoints en erreur
   topk(5,
     sum by (endpoint) (rate(http_requests_total{service="$service",status=~"5.."}[5m]))
   )
   ```

3. **Vérifier les logs d'erreur**
   ```logql
   {service="$service",level="ERROR"} |= "500" | json | line_format "{{.timestamp}} {{.message}}"
   ```

4. **Analyser les traces des requêtes échouées**
   - Ouvrir Jaeger: http://jaeger.lab.local
   - Filtrer par service et erreurs
   - Identifier le span causant l'échec

## Mitigation

### Immediate Actions
1. **Si déploiement récent**: Rollback immédiat
   ```bash
   kubectl rollout undo deployment/$service -n production
   ```

2. **Si surcharge**: Scaler horizontalement
   ```bash
   kubectl scale deployment/$service --replicas=5 -n production
   ```

3. **Si problème de dépendance**: Activer le circuit breaker
   ```bash
   kubectl set env deployment/$service CIRCUIT_BREAKER_ENABLED=true -n production
   ```

### Long-term Actions
- Review post-mortem
- Améliorer les tests de charge
- Implémenter des feature flags
- Renforcer le monitoring

## Escalation
- **L1 (0-15min)**: SRE on-call
- **L2 (15-30min)**: Team Lead
- **L3 (30min+)**: Engineering Manager

## Prevention
- Tests de charge réguliers
- Canary deployments
- Chaos engineering
- Review des dépendances
```

### Automatisation des runbooks

```python
#!/usr/bin/env python3
# runbook_executor.py

import subprocess
import time
from datetime import datetime

class RunbookExecutor:
    def __init__(self):
        self.log = []

    def log_action(self, action, result):
        """Log chaque action du runbook"""
        entry = {
            'timestamp': datetime.now().isoformat(),
            'action': action,
            'result': result
        }
        self.log.append(entry)
        print(f"[{entry['timestamp']}] {action}: {result}")

    def check_service_health(self, service):
        """Vérifie la santé du service"""
        cmd = f"kubectl get deployment {service} -n production -o json"
        result = subprocess.run(cmd, shell=True, capture_output=True, text=True)

        if result.returncode == 0:
            import json
            deployment = json.loads(result.stdout)
            ready = deployment['status'].get('readyReplicas', 0)
            desired = deployment['spec']['replicas']

            health = {
                'healthy': ready == desired,
                'ready': ready,
                'desired': desired
            }

            self.log_action(f"Health check {service}", health)
            return health

        return {'healthy': False, 'error': result.stderr}

    def get_error_rate(self, service):
        """Récupère le taux d'erreur depuis Prometheus"""
        query = f'''
            sum(rate(http_requests_total{{service="{service}",status=~"5.."}}[5m]))
            /
            sum(rate(http_requests_total{{service="{service}"}}[5m]))
        '''

        # Simulé pour l'exemple
        error_rate = 0.15  # 15% d'erreurs
        self.log_action(f"Error rate check {service}", f"{error_rate*100:.2f}%")
        return error_rate

    def execute_mitigation(self, service, strategy='auto'):
        """Exécute la stratégie de mitigation"""
        error_rate = self.get_error_rate(service)

        if error_rate > 0.1:  # Plus de 10% d'erreurs
            if strategy == 'auto':
                # Déterminer la meilleure stratégie
                health = self.check_service_health(service)

                if not health['healthy']:
                    strategy = 'restart'
                elif health['ready'] < 3:
                    strategy = 'scale'
                else:
                    strategy = 'rollback'

            self.log_action("Mitigation strategy selected", strategy)

            if strategy == 'rollback':
                self.rollback_deployment(service)
            elif strategy == 'scale':
                self.scale_deployment(service, replicas=5)
            elif strategy == 'restart':
                self.restart_deployment(service)

            # Attendre et vérifier
            time.sleep(30)
            new_error_rate = self.get_error_rate(service)

            success = new_error_rate < error_rate * 0.5
            self.log_action("Mitigation result",
                          f"Success: {success}, Error rate: {new_error_rate*100:.2f}%")

            return success

        return True

    def rollback_deployment(self, service):
        """Rollback le déploiement"""
        cmd = f"kubectl rollout undo deployment/{service} -n production"
        result = subprocess.run(cmd, shell=True, capture_output=True)
        self.log_action(f"Rollback {service}",
                       "Success" if result.returncode == 0 else "Failed")
        return result.returncode == 0

    def scale_deployment(self, service, replicas):
        """Scale le déploiement"""
        cmd = f"kubectl scale deployment/{service} --replicas={replicas} -n production"
        result = subprocess.run(cmd, shell=True, capture_output=True)
        self.log_action(f"Scale {service} to {replicas}",
                       "Success" if result.returncode == 0 else "Failed")
        return result.returncode == 0

    def restart_deployment(self, service):
        """Redémarre le déploiement"""
        cmd = f"kubectl rollout restart deployment/{service} -n production"
        result = subprocess.run(cmd, shell=True, capture_output=True)
        self.log_action(f"Restart {service}",
                       "Success" if result.returncode == 0 else "Failed")
        return result.returncode == 0

    def generate_report(self):
        """Génère un rapport des actions"""
        report = {
            'start_time': self.log[0]['timestamp'] if self.log else None,
            'end_time': self.log[-1]['timestamp'] if self.log else None,
            'actions': self.log,
            'success': all(
                'Success' in action.get('result', '') or
                action.get('result', {}).get('healthy', False)
                for action in self.log
            )
        }

        return report

# Utilisation
if __name__ == "__main__":
    executor = RunbookExecutor()

    # Exécuter le runbook pour un service en erreur
    service = "api-server"
    print(f"Executing runbook for {service}...")

    success = executor.execute_mitigation(service, strategy='auto')

    if success:
        print("✅ Mitigation successful")
    else:
        print("❌ Mitigation failed, escalating...")
        # Logique d'escalation

    # Générer le rapport
    report = executor.generate_report()
    print("\nReport:", report)
```

## Best Practices et recommandations

### Choisir les bons SLI

```yaml
# Guide de sélection des SLI
sli_selection_guide:
  user_facing_services:
    required:
      - availability  # Le service répond-il ?
      - latency      # Est-il assez rapide ?
      - quality      # Les réponses sont-elles correctes ?
    optional:
      - throughput   # Peut-il gérer la charge ?

  backend_services:
    required:
      - availability
      - latency
      - error_rate
    optional:
      - queue_depth
      - processing_time

  batch_jobs:
    required:
      - completion_rate  # Les jobs se terminent-ils ?
      - duration        # Dans un temps acceptable ?
    optional:
      - throughput
      - resource_usage

  data_pipelines:
    required:
      - freshness      # Les données sont-elles à jour ?
      - completeness   # Toutes les données sont-elles là ?
      - correctness    # Les données sont-elles correctes ?
    optional:
      - latency
      - throughput
```

### Éviter les pièges courants

```python
# Anti-patterns à éviter

# ❌ MAUVAIS : SLO trop ambitieux
slo_config = {
    'availability': 99.999,  # 5 minutes de downtime par an !
    'latency_p99': 10        # 10ms pour tout !
}

# ✅ BON : SLO réaliste basé sur l'historique
slo_config = {
    'availability': 99.9,    # 43 minutes par mois
    'latency_p99': 200,      # Basé sur le P99 actuel + marge
    'latency_p95': 100       # Objectif plus strict pour la majorité
}

# ❌ MAUVAIS : Moyennes trompeuses
sli = "avg(response_time)"  # Cache les outliers

# ✅ BON : Percentiles significatifs
sli = "histogram_quantile(0.95, response_time)"

# ❌ MAUVAIS : Fenêtre trop courte
error_budget_window = "1d"  # Trop volatile

# ✅ BON : Fenêtre stable
error_budget_window = "30d"  # Lisse les variations

# ❌ MAUVAIS : Ignorer les dépendances
slo_availability = 99.99  # Mais dépend d'un service à 99.9% !

# ✅ BON : SLO réaliste compte tenu des dépendances
# Si vous dépendez de N services à 99.9%
# Votre max théorique = 0.999^N
dependencies = 3
max_availability = 0.999 ** dependencies  # 99.7%
slo_availability = 99.5  # Avec marge
```

## Conclusion

Les SLI/SLO transforment la fiabilité d'un art subjectif en science objective. Avec les dashboards appropriés, vous savez exactement où vous en êtes, quand agir, et comment prioriser vos efforts.

### Points clés à retenir

1. **Commencez simple** : Un SLO de disponibilité vaut mieux que pas de SLO
2. **Basez-vous sur l'historique** : Vos SLO doivent être atteignables
3. **L'Error Budget est votre ami** : Il légitime l'innovation et le risque calculé
4. **Multi-window multi-burn-rate** : Détecte les problèmes rapidement sans faux positifs
5. **Automatisez** : Rapports, alertes et runbooks doivent être automatiques

### Architecture complète de fiabilité

Votre lab MicroK8s dispose maintenant d'un système complet de gestion de la fiabilité :

- **SLI** : Métriques objectives de performance
- **SLO** : Objectifs clairs et mesurables
- **Error Budget** : Gestion du risque quantifiée
- **Dashboards** : Visualisation temps réel
- **Alertes** : Détection proactive des problèmes
- **Runbooks** : Réponse standardisée aux incidents

Cette approche data-driven vous permet de prendre des décisions éclairées sur l'équilibre entre vélocité et fiabilité, transformant votre lab en un environnement professionnel de gestion de la fiabilité.

Dans la prochaine section, nous explorerons les runbooks et la documentation des alertes, complétant ainsi votre système d'observabilité avec des procédures opérationnelles claires et actionnables.

⏭️

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.04 - PromQL : langage de requête Prometheus

## Introduction à PromQL

### Qu'est-ce que PromQL ?

PromQL (Prometheus Query Language) est le langage de requête utilisé pour interroger les métriques stockées dans Prometheus. Si vous connaissez SQL pour les bases de données relationnelles, PromQL est l'équivalent pour les séries temporelles.

**Comparaison simple :**
```sql
-- SQL
SELECT count(*) FROM requests WHERE status = 200

-- PromQL équivalent
http_requests_total{status="200"}
```

### Où utiliser PromQL ?

1. **Interface web Prometheus** : Pour explorer les métriques
2. **Grafana** : Pour créer des graphiques
3. **Alertes** : Pour définir des conditions d'alerte
4. **API Prometheus** : Pour des requêtes programmatiques

### Accéder à l'interface de requête

```bash
# Port-forward vers Prometheus
kubectl port-forward -n monitoring svc/prometheus 9090:9090

# Ouvrir dans le navigateur
# http://localhost:9090/graph
```

## Types de données dans PromQL

### 1. Instant Vector (Vecteur instantané)
Une valeur unique par série temporelle à un instant T.

```promql
# Exemple : Utilisation CPU actuelle de tous les pods
container_cpu_usage_seconds_total
```

Résultat :
```
container_cpu_usage_seconds_total{pod="nginx-1", container="nginx"} 142.5
container_cpu_usage_seconds_total{pod="nginx-2", container="nginx"} 98.3
container_cpu_usage_seconds_total{pod="app-1", container="app"} 201.7
```

### 2. Range Vector (Vecteur de plage)
Plusieurs valeurs par série temporelle sur une période.

```promql
# Exemple : Utilisation CPU sur les 5 dernières minutes
container_cpu_usage_seconds_total[5m]
```

Résultat :
```
container_cpu_usage_seconds_total{pod="nginx-1"}
  100.5 @1234567890
  102.3 @1234567920
  104.1 @1234567950
  ...
```

### 3. Scalar (Scalaire)
Une simple valeur numérique.

```promql
# Exemple : Le nombre 100
100

# Ou le résultat d'une agrégation
count(up)  # Résultat: 42
```

### 4. String (Chaîne)
Rarement utilisé, principalement pour les labels.

## Sélecteurs de base

### Sélection simple

```promql
# Toutes les métriques avec ce nom
up

# Résultat exemple :
up{job="prometheus", instance="localhost:9090"} 1
up{job="node-exporter", instance="node1:9100"} 1
up{job="kubernetes-pods", instance="10.1.2.3:8080"} 0
```

### Sélection avec labels

```promql
# Syntaxe : metric_name{label="value"}

# Sélection exacte
up{job="prometheus"}

# Plusieurs labels (ET logique)
http_requests_total{method="GET", status="200"}

# Négation
http_requests_total{status!="200"}

# Expression régulière
http_requests_total{status=~"2.."}  # Tous les codes 2xx

# Négation d'expression régulière
http_requests_total{status!~"4.."}  # Pas de codes 4xx
```

### Opérateurs de comparaison pour les labels

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `=` | Égalité exacte | `{job="prometheus"}` |
| `!=` | Différent de | `{status!="200"}` |
| `=~` | Regex match | `{pod=~"nginx-.*"}` |
| `!~` | Regex ne match pas | `{namespace!~"kube-.*"}` |

## Opérateurs arithmétiques et de comparaison

### Opérations mathématiques

```promql
# Addition
node_memory_MemTotal_bytes + node_memory_MemFree_bytes

# Soustraction
node_memory_MemTotal_bytes - node_memory_MemFree_bytes

# Multiplication
rate(http_requests_total[5m]) * 60  # Requêtes par minute

# Division
node_memory_MemFree_bytes / node_memory_MemTotal_bytes * 100  # Pourcentage libre

# Modulo
timestamp(up) % 3600  # Secondes dans l'heure courante

# Puissance
2 ^ 10  # 1024
```

### Comparaisons avec filtrage

```promql
# Garder seulement les valeurs > 100
http_requests_total > 100

# Mémoire utilisée > 80%
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 0.8

# Comparaisons disponibles : ==, !=, >, <, >=, <=
```

### Opérateurs logiques

```promql
# AND - Intersection des séries
up{job="prometheus"} and up{instance="localhost:9090"}

# OR - Union des séries
up{job="prometheus"} or up{job="node-exporter"}

# UNLESS - Soustraction de séries
up{job="prometheus"} unless up{instance="localhost:9090"}
```

## Fonctions essentielles

### Fonctions de taux et dérivées

#### rate() - Taux de variation par seconde
```promql
# Taux de requêtes HTTP par seconde sur 5 minutes
rate(http_requests_total[5m])

# Taux d'erreurs
rate(http_requests_total{status=~"5.."}[5m])
```

#### irate() - Taux instantané
```promql
# Taux instantané (basé sur les 2 derniers points)
# Plus réactif mais plus volatile
irate(http_requests_total[5m])
```

#### increase() - Augmentation totale
```promql
# Nombre total de requêtes sur 1 heure
increase(http_requests_total[1h])

# Équivalent à : rate(metric[1h]) * 3600
```

### Fonctions d'agrégation

#### sum() - Somme
```promql
# Somme totale de toutes les requêtes
sum(rate(http_requests_total[5m]))

# Somme par label (GROUP BY en SQL)
sum by (method) (rate(http_requests_total[5m]))

# Somme en gardant certains labels
sum without (instance, pod) (rate(http_requests_total[5m]))
```

#### avg() - Moyenne
```promql
# Utilisation CPU moyenne de tous les pods
avg(container_cpu_usage_seconds_total)

# Moyenne par namespace
avg by (namespace) (container_cpu_usage_seconds_total)
```

#### count() - Comptage
```promql
# Nombre de pods qui sont UP
count(up == 1)

# Nombre de services par namespace
count by (namespace) (kube_service_info)
```

#### min() et max()
```promql
# Utilisation mémoire minimale parmi tous les nodes
min(node_memory_MemAvailable_bytes)

# Utilisation CPU maximale par pod
max by (pod) (rate(container_cpu_usage_seconds_total[5m]))
```

#### stddev() et stdvar() - Écart-type et variance
```promql
# Écart-type de la latence
stddev(http_request_duration_seconds)

# Variance de l'utilisation CPU
stdvar(rate(container_cpu_usage_seconds_total[5m]))
```

### Fonctions de transformation

#### abs() - Valeur absolue
```promql
abs(node_load1 - 4)
```

#### ceil() et floor() - Arrondi
```promql
# Arrondi supérieur
ceil(node_memory_MemAvailable_bytes / 1024 / 1024 / 1024)  # GB

# Arrondi inférieur
floor(node_filesystem_avail_bytes{mountpoint="/"} / 1073741824)
```

#### round() - Arrondi au plus proche
```promql
# Arrondi à 2 décimales
round(node_load1, 0.01)
```

#### clamp() - Limiter les valeurs
```promql
# Limiter entre 0 et 100
clamp(cpu_usage_percent, 0, 100)

# Minimum seulement
clamp_min(memory_available_bytes, 1073741824)  # Au moins 1GB

# Maximum seulement
clamp_max(rate(http_requests_total[5m]), 1000)  # Max 1000 req/s
```

### Fonctions temporelles

#### time() - Timestamp actuel
```promql
# Temps Unix actuel
time()
```

#### timestamp() - Timestamp de la métrique
```promql
# Quand la métrique a été scrapée
timestamp(up)
```

#### day_of_week() et hour()
```promql
# Jour de la semaine (0=dimanche, 6=samedi)
day_of_week()

# Heure actuelle (0-23)
hour()
```

### Fonctions pour histogrammes

#### histogram_quantile() - Calcul de quantiles
```promql
# 95e percentile de la latence
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# 50e percentile (médiane) par endpoint
histogram_quantile(0.5,
  sum by (endpoint, le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

## Requêtes avancées

### Labels dynamiques avec label_replace()

```promql
# Créer un nouveau label basé sur un existant
label_replace(
  up,
  "environment",  # Nouveau label
  "$1",           # Valeur (capture du regex)
  "namespace",    # Label source
  "(.*)-.*"       # Regex avec capture
)
```

### Jointures avec vector matching

```promql
# Multiplication avec correspondance de labels
node_memory_MemTotal_bytes * on(instance) group_left() up

# Division avec ignoring
rate(http_requests_total[5m])
  / ignoring(status, method)
  group_left()
  up
```

### Sous-requêtes

```promql
# Taux de variation du taux (accélération)
rate(rate(http_requests_total[5m])[10m:])

# Moyenne sur 1 heure du rate sur 5 minutes
avg_over_time(rate(http_requests_total[5m])[1h:])
```

## Patterns de requêtes courantes

### 1. Utilisation CPU en pourcentage

```promql
# CPU par container
100 * (
  rate(container_cpu_usage_seconds_total[5m])
)

# CPU par node
100 * (1 - avg by (instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
))
```

### 2. Utilisation mémoire

```promql
# Mémoire utilisée en pourcentage par node
100 * (1 -
  node_memory_MemAvailable_bytes /
  node_memory_MemTotal_bytes
)

# Mémoire par pod
sum by (pod) (container_memory_usage_bytes) / 1024 / 1024 / 1024  # En GB
```

### 3. Espace disque

```promql
# Espace disque disponible en pourcentage
100 * (
  node_filesystem_avail_bytes{mountpoint="/"} /
  node_filesystem_size_bytes{mountpoint="/"}
)

# Prédiction : disque plein dans combien de temps (en heures)
predict_linear(
  node_filesystem_avail_bytes{mountpoint="/"}[1h],
  4 * 3600
) < 0
```

### 4. Taux de requêtes et erreurs

```promql
# Requêtes par seconde total
sum(rate(http_requests_total[5m]))

# Taux d'erreur en pourcentage
100 * (
  sum(rate(http_requests_total{status=~"5.."}[5m])) /
  sum(rate(http_requests_total[5m]))
)

# Top 5 des endpoints avec le plus d'erreurs
topk(5,
  sum by (endpoint) (
    rate(http_requests_total{status=~"5.."}[5m])
  )
)
```

### 5. Latence et percentiles

```promql
# 95e percentile de latence global
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# Percentiles multiples
label_join(
  histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)),
  "percentile", "", "p50"
)
or
label_join(
  histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)),
  "percentile", "", "p95"
)
or
label_join(
  histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)),
  "percentile", "", "p99"
)
```

### 6. Disponibilité et SLO

```promql
# Disponibilité sur 24h (en %)
100 * (
  avg_over_time(up[24h])
)

# SLO : 99.9% de requêtes réussies sur 30 jours
sum(increase(http_requests_total{status!~"5.."}[30d])) /
sum(increase(http_requests_total[30d])) > 0.999
```

### 7. Saturation et limites

```promql
# Pods proches de leur limite mémoire
container_memory_usage_bytes /
container_spec_memory_limit_bytes > 0.8

# Pods throttled CPU
rate(container_cpu_throttled_periods_total[5m]) > 0
```

## Optimisation des requêtes

### Bonnes pratiques

1. **Utiliser rate() avant sum()**
```promql
# ✅ BON - Calcule le rate d'abord
sum(rate(http_requests_total[5m]))

# ❌ MAUVAIS - Somme puis rate (incorrecte)
rate(sum(http_requests_total)[5m])
```

2. **Spécifier les labels pour réduire la cardinalité**
```promql
# ✅ BON - Filtre tôt
rate(http_requests_total{job="api", method="GET"}[5m])

# ❌ MAUVAIS - Filtre tard
rate(http_requests_total[5m]){job="api", method="GET"}
```

3. **Utiliser les bonnes fenêtres temporelles**
```promql
# Pour rate(), utiliser au moins 2x le scrape_interval
# Si scrape_interval = 15s, utiliser au moins [30s]
rate(metric[1m])  # ✅ Sûr

# Pour les alertes, utiliser des fenêtres plus larges
rate(metric[5m])  # ✅ Stable pour les alertes
```

### Recording rules pour les requêtes complexes

Si une requête est utilisée souvent, créez une recording rule :

```yaml
# prometheus-rules.yaml
groups:
- name: shortcuts
  interval: 30s
  rules:
  - record: instance:node_cpu:rate5m
    expr: |
      100 * (1 - avg by (instance) (
        rate(node_cpu_seconds_total{mode="idle"}[5m])
      ))

  - record: job:http_requests:rate5m
    expr: |
      sum by (job, method, status) (
        rate(http_requests_total[5m])
      )
```

Puis utiliser simplement :
```promql
instance:node_cpu:rate5m{instance="node1"}
```

## Interface et outils

### Console Prometheus

**Onglet Graph :**
- Zone de requête avec autocomplétion
- Bouton "Execute" pour lancer
- Switch Table/Graph pour la vue
- Time range selector

**Onglet Table :**
- Affichage tabulaire des résultats
- Tri par colonne
- Export CSV possible

### Raccourcis utiles

| Action | Raccourci |
|--------|-----------|
| Autocomplétion | `Ctrl + Space` |
| Exécuter requête | `Ctrl + Enter` |
| Format requête | `Ctrl + Shift + F` |
| Historique | Flèches haut/bas |

### API pour requêtes programmatiques

```bash
# Requête instantanée
curl -G http://localhost:9090/api/v1/query \
  --data-urlencode 'query=up'

# Requête sur une plage
curl -G http://localhost:9090/api/v1/query_range \
  --data-urlencode 'query=rate(http_requests_total[5m])' \
  --data-urlencode 'start=2024-01-01T00:00:00Z' \
  --data-urlencode 'end=2024-01-01T01:00:00Z' \
  --data-urlencode 'step=15s'

# Avec jq pour formater
curl -sG http://localhost:9090/api/v1/query \
  --data-urlencode 'query=up' | jq '.data.result'
```

## Debugging et erreurs courantes

### Erreurs fréquentes

**1. "No data points"**
```promql
# Vérifier que la métrique existe
{__name__=~"http_request.*"}

# Vérifier la fenêtre temporelle
# Augmenter si nécessaire
rate(metric[10m])  # Au lieu de [1m]
```

**2. "Invalid expression type"**
```promql
# rate() a besoin d'un range vector
rate(metric)       # ❌ Erreur
rate(metric[5m])   # ✅ Correct
```

**3. "Many-to-many matching not allowed"**
```promql
# Utiliser on() ou ignoring() pour préciser
metric1 * on(instance) metric2

# Ou group_left/group_right pour cardinalité
metric1 * on(instance) group_left() metric2
```

### Requêtes de diagnostic

```promql
# Vérifier les targets UP
up

# Métriques les plus volumineuses
topk(10, count by (__name__)({__name__=~".+"}))

# Scrape duration
scrape_duration_seconds

# Métriques par job
count by (job) (up)

# Dernière fois qu'un target a été scrapé
time() - timestamp(up)
```

## Exemples pour cas d'usage spécifiques

### Pour les développeurs

```promql
# Performance de mon API
histogram_quantile(0.95,
  sum by (endpoint, le) (
    rate(http_request_duration_seconds_bucket{job="my-api"}[5m])
  )
)

# Erreurs par endpoint
sum by (endpoint, status) (
  rate(http_requests_total{job="my-api", status=~"5.."}[5m])
)

# Goroutines (pour Go)
go_goroutines{job="my-api"}
```

### Pour les ops

```promql
# Nodes avec peu de mémoire disponible
node_memory_MemAvailable_bytes < 1073741824  # < 1GB

# Pods redémarrés récemment
rate(kube_pod_container_status_restarts_total[15m]) > 0

# Services sans endpoints
kube_service_info unless on(namespace, service)
  kube_endpoint_address_available
```

### Pour le capacity planning

```promql
# Prédiction utilisation CPU dans 4h
predict_linear(node_cpu_seconds_total[1h], 4 * 3600)

# Tendance mémoire sur 7 jours
deriv(node_memory_MemAvailable_bytes[7d])

# Croissance du nombre de pods
increase(kube_pod_info[30d])
```

## Checklist PromQL

- [ ] Je comprends les 4 types de données (instant/range vector, scalar, string)
- [ ] Je sais utiliser les sélecteurs avec labels
- [ ] Je maîtrise rate(), increase() et irate()
- [ ] Je sais utiliser les agrégations (sum, avg, max, etc.)
- [ ] Je comprends by() et without() pour grouper
- [ ] Je sais calculer des percentiles avec histogram_quantile()
- [ ] Je peux écrire des requêtes pour CPU, mémoire, disque
- [ ] Je sais calculer des taux d'erreur et disponibilité
- [ ] Je comprends les bonnes pratiques d'optimisation
- [ ] Je sais débugger les erreurs courantes

## Préparation pour la suite

Maintenant que vous maîtrisez PromQL, vous êtes prêt à :
- Configurer des exporters pour enrichir vos métriques (section 8.05)
- Créer des dashboards Grafana avec vos requêtes (section 8.06)
- Définir des alertes basées sur vos requêtes PromQL (section 8.11)

PromQL est le cœur de l'écosystème Prometheus. Plus vous le maîtriserez, plus vous pourrez extraire d'insights de vos métriques.

⏭️

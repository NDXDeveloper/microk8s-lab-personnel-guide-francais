🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.01 - Introduction à l'écosystème Prometheus

## Qu'est-ce que Prometheus ?

Prometheus est un système de monitoring et d'alerting open source créé par SoundCloud en 2012 et adopté par la Cloud Native Computing Foundation (CNCF) en 2016. Il est devenu le standard de facto pour le monitoring dans Kubernetes grâce à sa conception adaptée aux environnements dynamiques et conteneurisés.

### Concepts fondamentaux pour les débutants

Avant de plonger dans Prometheus, clarifions quelques termes essentiels :

- **Métrique** : Une mesure numérique qui change dans le temps (exemple : utilisation CPU à 45%)
- **Série temporelle** : Une suite de valeurs d'une métrique enregistrées au fil du temps
- **Label** : Une étiquette qui ajoute du contexte à une métrique (exemple : `pod="nginx"`, `namespace="production"`)
- **Target** : Une source de métriques que Prometheus va interroger (un pod, un service, un node)
- **Scraping** : L'action de collecter les métriques depuis les targets

## Architecture de Prometheus

### Composants principaux

```
┌─────────────────────────────────────────────────────────────┐
│                     PROMETHEUS SERVER                       │
│                                                             │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐     │
│  │   Retrieval  │   │     TSDB     │   │   PromQL     │     │
│  │   (Scraper)  │──▶│   Storage    │◀──│    Engine    │     │
│  └──────────────┘   └──────────────┘   └──────────────┘     │
│         ▲                                      │            │
└─────────┼──────────────────────────────────────┼────────────┘
          │                                      │
          │ Pull                                 │ Query
          │ (HTTP)                               │
          │                                      ▼
   ┌──────────────┐                      ┌──────────────┐
   │   Targets    │                      │   Clients    │
   │              │                      │              │
   │ - Apps       │                      │ - Grafana    │
   │ - Exporters  │                      │ - API calls  │
   │ - Pushgateway│                      │ - Alerts     │
   └──────────────┘                      └──────────────┘
```

**1. Prometheus Server**
- **Retrieval** : Module qui "scrape" (récupère) les métriques des targets
- **TSDB** : Base de données optimisée pour les séries temporelles
- **PromQL Engine** : Moteur de requêtes pour analyser les données

**2. Targets (Cibles)**
- Applications qui exposent leurs métriques
- Exporters qui traduisent des métriques système
- Pushgateway pour les jobs de courte durée

**3. Clients**
- Interfaces de visualisation (Grafana)
- Systèmes d'alerting
- Applications utilisant l'API

### Le modèle Pull vs Push

Contrairement à beaucoup de systèmes de monitoring qui utilisent un modèle "push" (les agents envoient les données), Prometheus utilise principalement un modèle "pull" :

**Modèle Pull de Prometheus :**
```
Prometheus ──────[HTTP GET /metrics]──────▶ Target
           ◀─────[Métriques au format texte]─────
```

**Avantages du modèle Pull :**
- **Simplicité** : Les targets exposent simplement leurs métriques sur un endpoint HTTP
- **Fiabilité** : Prometheus sait quand un target est down
- **Sécurité** : Pas besoin d'ouvrir des ports entrants sur Prometheus
- **Découverte** : Service discovery automatique dans Kubernetes

## Types de métriques

Prometheus supporte quatre types de métriques, chacun adapté à des cas d'usage spécifiques :

### 1. Counter (Compteur)
Une valeur qui ne peut qu'augmenter ou être remise à zéro.

**Exemples :**
- Nombre total de requêtes HTTP reçues
- Nombre d'erreurs depuis le démarrage
- Bytes transmis sur le réseau

**Format :**
```
http_requests_total{method="GET", status="200"} 1254
http_requests_total{method="POST", status="500"} 23
```

### 2. Gauge (Jauge)
Une valeur qui peut monter ou descendre.

**Exemples :**
- Utilisation CPU actuelle
- Mémoire disponible
- Nombre de connexions actives
- Température

**Format :**
```
memory_usage_bytes{host="node1"} 2147483648
active_connections{service="database"} 42
temperature_celsius{location="datacenter"} 21.5
```

### 3. Histogram (Histogramme)
Échantillonne les observations et les compte dans des buckets configurables.

**Utilisation :**
- Temps de réponse des requêtes
- Tailles des requêtes/réponses
- Durées d'opérations

**Format :**
```
http_request_duration_seconds_bucket{le="0.1"} 1000
http_request_duration_seconds_bucket{le="0.5"} 1500
http_request_duration_seconds_bucket{le="1.0"} 1700
http_request_duration_seconds_bucket{le="+Inf"} 1800
http_request_duration_seconds_sum 543.21
http_request_duration_seconds_count 1800
```

### 4. Summary (Résumé)
Similaire à l'histogram mais calcule des quantiles côté client.

**Format :**
```
http_request_duration_seconds{quantile="0.5"} 0.052
http_request_duration_seconds{quantile="0.9"} 0.210
http_request_duration_seconds{quantile="0.99"} 0.950
```

## Format d'exposition des métriques

Les métriques sont exposées dans un format texte simple et lisible :

```
# HELP http_requests_total Total des requêtes HTTP reçues
# TYPE http_requests_total counter
http_requests_total{method="GET",handler="/api/users",status="200"} 1027
http_requests_total{method="GET",handler="/api/users",status="404"} 3
http_requests_total{method="POST",handler="/api/users",status="201"} 251

# HELP memory_usage_ratio Ratio d'utilisation mémoire
# TYPE memory_usage_ratio gauge
memory_usage_ratio 0.65

# HELP go_gc_duration_seconds Durée du garbage collector Go
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0.000125
go_gc_duration_seconds{quantile="0.5"} 0.000300
go_gc_duration_seconds{quantile="1"} 0.002100
```

Structure d'une ligne de métrique :
```
nom_métrique{label1="valeur1",label2="valeur2"} valeur_numérique timestamp_optionnel
```

## Service Discovery dans Kubernetes

L'une des forces de Prometheus est sa capacité à découvrir automatiquement les targets dans Kubernetes.

### Mécanismes de découverte

**1. Pod Discovery**
```yaml
- job_name: 'kubernetes-pods'
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
```

**2. Service Discovery**
```yaml
- job_name: 'kubernetes-services'
  kubernetes_sd_configs:
  - role: service
```

**3. Node Discovery**
```yaml
- job_name: 'kubernetes-nodes'
  kubernetes_sd_configs:
  - role: node
```

### Annotations Kubernetes

Les pods et services peuvent utiliser des annotations pour contrôler le scraping :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-application
  annotations:
    prometheus.io/scrape: "true"      # Active le scraping
    prometheus.io/port: "8080"        # Port des métriques
    prometheus.io/path: "/metrics"    # Chemin des métriques
spec:
  containers:
  - name: app
    image: mon-app:latest
```

## Exporters : Les traducteurs de métriques

Les exporters sont des composants qui exposent des métriques de systèmes qui ne supportent pas nativement Prometheus.

### Exporters essentiels pour Kubernetes

**1. Node Exporter**
Expose les métriques hardware et OS des nodes :
- CPU, mémoire, disque, réseau
- Processus, load average
- Système de fichiers

**2. kube-state-metrics**
Expose l'état des objets Kubernetes :
- Deployments, ReplicaSets, Pods
- Services, Ingress
- ConfigMaps, Secrets
- Jobs, CronJobs

**3. cAdvisor (Container Advisor)**
Intégré au kubelet, expose les métriques des containers :
- Utilisation CPU/mémoire par container
- I/O réseau et disque
- Throttling

### Architecture avec exporters

```
┌────────────────────────────────────────────────┐
│                  Kubernetes Node               │
│                                                │
│  ┌──────────────┐        ┌──────────────┐      │
│  │     Pod      │        │     Pod      │      │
│  │  [App:8080]  │        │  [App:8080]  │      │
│  └──────────────┘        └──────────────┘      │
│                                                │
│  ┌──────────────────────────────────────┐      │
│  │         Node Exporter [:9100]        │      │
│  └──────────────────────────────────────┘      │
│                                                │
│  ┌──────────────────────────────────────┐      │
│  │    kube-state-metrics [:8080]        │      │
│  └──────────────────────────────────────┘      │
│                                                │
│  ┌──────────────────────────────────────┐      │
│  │         cAdvisor [:10250]            │      │
│  └──────────────────────────────────────┘      │
└────────────────────────────────────────────────┘
                       │
                       │ Scraping
                       ▼
               ┌──────────────┐
               │  Prometheus  │
               └──────────────┘
```

## Labels et bonnes pratiques

### Importance des labels

Les labels sont la clé de la puissance de Prometheus. Ils permettent de :
- **Filtrer** : Sélectionner des métriques spécifiques
- **Agréger** : Grouper des métriques similaires
- **Analyser** : Créer des dimensions d'analyse

### Bonnes pratiques pour les labels

**✅ À FAIRE :**
```
# Bon : labels avec cardinalité contrôlée
http_requests_total{method="GET", status="200", service="api"}
cpu_usage_percent{node="worker-1", cluster="production"}
```

**❌ À ÉVITER :**
```
# Mauvais : labels avec haute cardinalité
http_requests_total{user_id="12345", session_id="abc-def-ghi"}
# Problème : Peut créer des millions de séries temporelles
```

### Labels automatiques dans Kubernetes

Prometheus ajoute automatiquement des labels utiles :
- `namespace` : Namespace Kubernetes
- `pod` : Nom du pod
- `service` : Service associé
- `node` : Node hébergeant le pod
- `job` : Job Prometheus

## Configuration de base

### Structure d'un fichier prometheus.yml

```yaml
# Configuration globale
global:
  scrape_interval: 15s        # Fréquence de scraping par défaut
  evaluation_interval: 15s    # Fréquence d'évaluation des règles
  external_labels:
    cluster: 'microk8s-lab'   # Labels ajoutés à toutes les métriques

# Configuration des alertes (optionnel)
alerting:
  alertmanagers:
  - static_configs:
    - targets: ['alertmanager:9093']

# Fichiers de règles (optionnel)
rule_files:
  - 'alerts/*.yml'
  - 'rules/*.yml'

# Jobs de scraping
scrape_configs:
  # Prometheus lui-même
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']

  # Découverte automatique des pods
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
    - role: pod
    relabel_configs:
    # Ne scraper que les pods avec l'annotation prometheus.io/scrape: "true"
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    # Utiliser le port depuis l'annotation
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      target_label: __address__
      regex: ([^:]+)(?::\d+)?
      replacement: $1:${1}
```

## Stockage et rétention

### TSDB (Time Series Database)

Prometheus utilise sa propre base de données optimisée :

**Caractéristiques :**
- **Compression** : Données compressées efficacement
- **Blocks** : Données organisées en blocks de 2 heures
- **WAL** : Write-Ahead Log pour la durabilité
- **Compaction** : Fusion automatique des anciens blocks

### Estimation du stockage

Formule approximative :
```
Stockage = nb_séries × nb_échantillons × 2 bytes

Où :
- nb_séries = nb_métriques × nb_combinaisons_labels
- nb_échantillons = (durée_rétention / intervalle_scraping)
```

**Exemple pour un petit lab :**
- 1000 séries temporelles
- Scraping toutes les 15 secondes
- Rétention de 15 jours

```
Échantillons par série = (15 jours × 86400 sec) / 15 sec = 86,400
Stockage = 1000 × 86,400 × 2 bytes ≈ 165 MB
```

## Écosystème et intégrations

### Projets officiels

1. **Alertmanager** : Gestion des alertes (déduplication, routage, notifications)
2. **Pushgateway** : Pour les jobs batch qui ne peuvent pas être scrapés
3. **Blackbox Exporter** : Monitoring de endpoints (HTTP, DNS, TCP, ICMP)
4. **SNMP Exporter** : Pour équipements réseau

### Intégrations populaires

- **Grafana** : Visualisation de référence
- **Thanos** : Stockage long terme et requêtes globales
- **Cortex** : Multi-tenancy et scalabilité horizontale
- **VictoriaMetrics** : Alternative compatible avec stockage optimisé

## Points clés à retenir

1. **Prometheus est pull-based** : Il va chercher les métriques plutôt que de les recevoir
2. **Les labels sont essentiels** : Ils permettent la flexibilité des requêtes
3. **Service discovery automatique** : Parfait pour les environnements dynamiques
4. **Format simple** : Les métriques sont exposées en texte lisible
5. **Écosystème riche** : Des dizaines d'exporters disponibles
6. **Stockage efficace** : TSDB optimisée pour les séries temporelles

## Préparation pour la suite

Maintenant que vous comprenez l'architecture et les concepts de Prometheus, vous êtes prêt pour :
- Installer Prometheus dans votre cluster MicroK8s
- Configurer la découverte automatique des services
- Comprendre les métriques collectées
- Écrire vos premières requêtes PromQL

La prochaine section couvrira l'installation pratique de Prometheus sur MicroK8s avec les différentes options disponibles.

⏭️

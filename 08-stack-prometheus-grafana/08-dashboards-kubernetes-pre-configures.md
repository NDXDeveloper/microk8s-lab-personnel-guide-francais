🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.08 - Dashboards Kubernetes pré-configurés

## Introduction aux dashboards pré-configurés

### Pourquoi utiliser des dashboards pré-configurés ?

Les dashboards pré-configurés sont des tableaux de bord créés par la communauté ou des experts qui vous font gagner un temps précieux :

- **Gain de temps** : Pas besoin de créer from scratch
- **Bonnes pratiques** : Conçus par des experts Kubernetes
- **Métriques essentielles** : Incluent les KPIs importants
- **Design professionnel** : Layouts optimisés et esthétiques
- **Évolutifs** : Base solide pour personnalisation

### Sources de dashboards

```
┌────────────────────────────────────────────────────────┐
│              Sources de Dashboards                     │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌──────────────────────────────────────────────┐      │
│  │         Grafana.com/dashboards               │      │
│  │    • Plus de 15,000 dashboards               │      │
│  │    • Recherche par tags (kubernetes, k8s)    │      │
│  │    • Notes et popularité                     │      │
│  └──────────────────────────────────────────────┘      │
│                                                        │
│  ┌──────────────────────────────────────────────┐      │
│  │           GitHub Repositories                │      │
│  │    • kubernetes-monitoring                   │      │
│  │    • awesome-grafana                         │      │
│  │    • prometheus-operator dashboards          │      │
│  └──────────────────────────────────────────────┘      │
│                                                        │
│  ┌──────────────────────────────────────────────┐      │
│  │            Vendors officiels                 │      │
│  │    • NGINX Ingress Controller                │      │
│  │    • CoreDNS                                 │      │
│  │    • Istio Service Mesh                      │      │
│  └──────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────┘
```

## Top 10 des dashboards Kubernetes essentiels

### 1. Kubernetes Cluster Monitoring (ID: 8588)

**Description :** Vue d'ensemble complète du cluster
**Métriques clés :**
- Santé globale du cluster
- Utilisation CPU/RAM par node
- Pods par namespace
- Network I/O

```bash
# Import rapide
Dashboard ID: 8588
Revision: Latest
Data Source: Prometheus
```

### 2. Node Exporter Full (ID: 1860)

**Description :** Métriques détaillées des nodes
**Métriques clés :**
- CPU, Memory, Disk, Network
- System load
- Filesystem usage
- Hardware temperatures

```bash
Dashboard ID: 1860
Revision: 31
Data Source: Prometheus
```

### 3. Kubernetes Pods Overview (ID: 15760)

**Description :** Monitoring détaillé des pods
**Métriques clés :**
- État des pods
- Ressources par pod
- Restarts
- Age des pods

```bash
Dashboard ID: 15760
Revision: Latest
Data Source: Prometheus
```

### 4. NGINX Ingress Controller (ID: 9614)

**Description :** Métriques de l'Ingress Controller
**Métriques clés :**
- Requêtes par seconde
- Latence
- Taux d'erreur
- Connexions actives

```bash
Dashboard ID: 9614
Revision: 1
Data Source: Prometheus
```

### 5. Kubernetes Cluster Autoscaler (ID: 3831)

**Description :** Monitoring de l'autoscaling
**Métriques clés :**
- Scaling events
- Pending pods
- Node utilization
- Scale up/down decisions

```bash
Dashboard ID: 3831
Revision: Latest
Data Source: Prometheus
```

### 6. CoreDNS Monitoring (ID: 14981)

**Description :** DNS cluster metrics
**Métriques clés :**
- Query rate
- Response time
- Cache hit ratio
- Errors

```bash
Dashboard ID: 14981
Revision: Latest
Data Source: Prometheus
```

### 7. Kubernetes StatefulSets (ID: 14001)

**Description :** Monitoring des StatefulSets
**Métriques clés :**
- Replicas status
- PVC usage
- Pod ordering
- Update status

```bash
Dashboard ID: 14001
Revision: Latest
Data Source: Prometheus
```

### 8. Container Resources (ID: 11600)

**Description :** Ressources par container
**Métriques clés :**
- CPU throttling
- Memory limits vs usage
- Container restarts
- Network metrics

```bash
Dashboard ID: 11600
Revision: Latest
Data Source: Prometheus
```

### 9. Kubernetes Deployments (ID: 8309)

**Description :** État des deployments
**Métriques clés :**
- Deployment status
- Rollout history
- Replica availability
- Update strategy metrics

```bash
Dashboard ID: 8309
Revision: Latest
Data Source: Prometheus
```

### 10. Persistent Volumes (ID: 13646)

**Description :** Monitoring du stockage
**Métriques clés :**
- PV/PVC usage
- Storage capacity
- I/O metrics
- Volume health

```bash
Dashboard ID: 13646
Revision: Latest
Data Source: Prometheus
```

## Import de dashboards via l'interface Grafana

### Méthode 1 : Import par ID

1. **Accéder à l'import :**
   - Cliquer sur l'icône "+" dans le menu latéral
   - Sélectionner "Import"

2. **Entrer l'ID du dashboard :**
   ```
   Import via grafana.com: [1860]
   Click "Load"
   ```

3. **Configuration :**
   - Nom : Personnaliser si souhaité
   - Folder : Choisir ou créer un dossier
   - UID : Laisser auto-généré
   - Data source : Sélectionner "Prometheus"

4. **Import :**
   - Cliquer sur "Import"
   - Le dashboard s'ouvre automatiquement

### Méthode 2 : Import par JSON

```json
{
  "dashboard": {
    "title": "Custom Kubernetes Dashboard",
    "panels": [...],
    "templating": {...},
    "time": {
      "from": "now-6h",
      "to": "now"
    }
  },
  "inputs": [
    {
      "name": "DS_PROMETHEUS",
      "label": "Prometheus",
      "description": "",
      "type": "datasource",
      "pluginId": "prometheus",
      "pluginName": "Prometheus"
    }
  ]
}
```

### Méthode 3 : Import via URL

```bash
# Télécharger le JSON
wget https://grafana.com/api/dashboards/1860/revisions/31/download -O node-exporter.json

# Puis importer le fichier JSON dans Grafana
```

## Import automatique via provisioning

### Configuration avec ConfigMap

```yaml
# grafana-dashboards-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-providers
  namespace: monitoring
data:
  dashboards.yaml: |
    apiVersion: 1
    providers:
    - name: 'kubernetes'
      orgId: 1
      folder: 'Kubernetes'
      type: file
      updateIntervalSeconds: 30
      allowUiUpdates: true
      options:
        path: /var/lib/grafana/dashboards/kubernetes
    - name: 'infrastructure'
      orgId: 1
      folder: 'Infrastructure'
      type: file
      updateIntervalSeconds: 30
      options:
        path: /var/lib/grafana/dashboards/infrastructure
```

### Téléchargement automatique de dashboards

```yaml
# init-container pour télécharger les dashboards
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  template:
    spec:
      initContainers:
      - name: download-dashboards
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          mkdir -p /dashboards/kubernetes
          mkdir -p /dashboards/infrastructure

          # Kubernetes Cluster
          curl -L https://grafana.com/api/dashboards/8588/revisions/1/download \
            -o /dashboards/kubernetes/cluster.json

          # Node Exporter
          curl -L https://grafana.com/api/dashboards/1860/revisions/31/download \
            -o /dashboards/infrastructure/node-exporter.json

          # Pods Overview
          curl -L https://grafana.com/api/dashboards/15760/revisions/1/download \
            -o /dashboards/kubernetes/pods.json

          echo "Dashboards downloaded successfully"
        volumeMounts:
        - name: dashboards
          mountPath: /dashboards

      containers:
      - name: grafana
        volumeMounts:
        - name: dashboards
          mountPath: /var/lib/grafana/dashboards

      volumes:
      - name: dashboards
        emptyDir: {}
```

### Script d'import en masse

```bash
#!/bin/bash
# import-dashboards.sh

GRAFANA_URL="http://localhost:3000"
GRAFANA_USER="admin"
GRAFANA_PASSWORD="admin"

# Liste des dashboards à importer
declare -A DASHBOARDS=(
  ["Kubernetes Cluster"]="8588"
  ["Node Exporter Full"]="1860"
  ["Kubernetes Pods"]="15760"
  ["NGINX Ingress"]="9614"
  ["CoreDNS"]="14981"
)

echo "Importing Kubernetes dashboards..."

for name in "${!DASHBOARDS[@]}"; do
  ID="${DASHBOARDS[$name]}"
  echo "Importing $name (ID: $ID)..."

  # Télécharger le dashboard
  DASHBOARD_JSON=$(curl -s "https://grafana.com/api/dashboards/${ID}/revisions/1/download")

  # Préparer le payload
  IMPORT_PAYLOAD=$(cat <<EOF
{
  "dashboard": ${DASHBOARD_JSON},
  "overwrite": true,
  "inputs": [
    {
      "name": "DS_PROMETHEUS",
      "type": "datasource",
      "pluginId": "prometheus",
      "value": "Prometheus"
    }
  ],
  "folderId": 0
}
EOF
)

  # Importer dans Grafana
  RESULT=$(curl -s -X POST \
    -H "Content-Type: application/json" \
    -u ${GRAFANA_USER}:${GRAFANA_PASSWORD} \
    ${GRAFANA_URL}/api/dashboards/import \
    -d "${IMPORT_PAYLOAD}")

  if echo "$RESULT" | grep -q "success"; then
    echo "✅ $name imported successfully"
  else
    echo "❌ Failed to import $name"
  fi
done

echo "Import complete!"
```

## Personnalisation des dashboards importés

### Adaptation aux labels de votre environnement

```json
// Variables à adapter dans les dashboards
{
  "templating": {
    "list": [
      {
        "name": "namespace",
        "query": "label_values(kube_pod_info, namespace)",
        "datasource": "Prometheus"
      },
      {
        "name": "pod",
        "query": "label_values(kube_pod_info{namespace=\"$namespace\"}, pod)",
        "datasource": "Prometheus"
      },
      {
        "name": "node",
        "query": "label_values(kube_node_info, node)",
        "datasource": "Prometheus"
      }
    ]
  }
}
```

### Ajustement des seuils d'alerte

```json
{
  "fieldConfig": {
    "defaults": {
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {
            "color": "green",
            "value": null
          },
          {
            "color": "yellow",
            "value": 70  // Ajuster selon vos besoins
          },
          {
            "color": "red",
            "value": 90  // Seuil critique
          }
        ]
      }
    }
  }
}
```

### Modification des time ranges

```json
{
  "time": {
    "from": "now-3h",  // Ajuster selon vos besoins
    "to": "now"
  },
  "refresh": "30s"     // Fréquence de rafraîchissement
}
```

## Organisation des dashboards

### Structure recommandée par dossiers

```
Grafana Dashboards/
├── 🏠 Home/
│   └── Overview Dashboard
├── 📊 Kubernetes/
│   ├── Cluster Overview
│   ├── Nodes
│   ├── Pods
│   ├── Deployments
│   ├── StatefulSets
│   └── Services
├── 🔧 Infrastructure/
│   ├── Node Exporter
│   ├── Network
│   ├── Storage
│   └── DNS
├── 🌐 Ingress/
│   ├── NGINX Controller
│   ├── Traffic Overview
│   └── SSL Certificates
├── 📱 Applications/
│   ├── App Dashboard Template
│   └── Custom Apps
└── 🚨 Alerts/
    └── Alert Overview
```

### Création de dossiers via API

```bash
# Créer des dossiers pour organiser les dashboards
curl -X POST \
  -H "Content-Type: application/json" \
  -u admin:admin \
  http://localhost:3000/api/folders \
  -d '{
    "title": "Kubernetes",
    "uid": "kubernetes"
  }'

curl -X POST \
  -H "Content-Type: application/json" \
  -u admin:admin \
  http://localhost:3000/api/folders \
  -d '{
    "title": "Infrastructure",
    "uid": "infrastructure"
  }'
```

## Dashboards spécialisés par composant

### Dashboard pour MicroK8s spécifique

```json
{
  "dashboard": {
    "title": "MicroK8s Overview",
    "panels": [
      {
        "title": "MicroK8s Status",
        "targets": [
          {
            "expr": "up{job=\"kube-state-metrics\"}"
          }
        ]
      },
      {
        "title": "Addons Status",
        "targets": [
          {
            "expr": "kube_deployment_status_replicas_available{namespace=~\"kube-system|ingress|monitoring\"}"
          }
        ]
      },
      {
        "title": "Snap Services",
        "targets": [
          {
            "expr": "node_systemd_unit_state{name=~\"snap.microk8s.*\"}"
          }
        ]
      }
    ]
  }
}
```

### Dashboard pour Ingress Controller

```json
{
  "panels": [
    {
      "title": "Request Rate",
      "targets": [
        {
          "expr": "rate(nginx_ingress_controller_requests[5m])"
        }
      ]
    },
    {
      "title": "Response Time P95",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, rate(nginx_ingress_controller_request_duration_seconds_bucket[5m]))"
        }
      ]
    },
    {
      "title": "Error Rate",
      "targets": [
        {
          "expr": "rate(nginx_ingress_controller_requests{status=~\"5..\"}[5m])"
        }
      ]
    }
  ]
}
```

### Dashboard pour StatefulSets

```json
{
  "panels": [
    {
      "title": "StatefulSet Replicas",
      "targets": [
        {
          "expr": "kube_statefulset_status_replicas"
        }
      ]
    },
    {
      "title": "PVC Usage",
      "targets": [
        {
          "expr": "kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100"
        }
      ]
    }
  ]
}
```

## Optimisation des dashboards importés

### Réduction de la charge sur Prometheus

```json
// Utiliser des variables pour limiter les requêtes
{
  "templating": {
    "list": [
      {
        "name": "resolution",
        "options": [
          {"text": "30s", "value": "30s"},
          {"text": "1m", "value": "1m"},
          {"text": "5m", "value": "5m"}
        ],
        "current": {
          "value": "1m"
        }
      }
    ]
  },
  "panels": [
    {
      "targets": [
        {
          // Utiliser la variable resolution
          "expr": "rate(metric[$resolution])"
        }
      ]
    }
  ]
}
```

### Cache et performance

```json
{
  "panels": [
    {
      "cacheTimeout": 60,  // Cache de 60 secondes
      "interval": "30s",    // Intervalle minimum
      "maxDataPoints": 300  // Limiter les points
    }
  ]
}
```

### Queries optimisées

```promql
# Au lieu de :
rate(container_cpu_usage_seconds_total[5m])

# Utiliser si disponible :
container_cpu_usage_seconds:rate5m  # Recording rule
```

## Backup et versioning des dashboards

### Export automatique

```bash
#!/bin/bash
# backup-dashboards.sh

GRAFANA_URL="http://localhost:3000"
GRAFANA_USER="admin"
GRAFANA_PASSWORD="admin"
BACKUP_DIR="./dashboard-backups/$(date +%Y%m%d)"

mkdir -p "$BACKUP_DIR"

# Lister tous les dashboards
DASHBOARDS=$(curl -s -u ${GRAFANA_USER}:${GRAFANA_PASSWORD} \
  ${GRAFANA_URL}/api/search?type=dash-db | jq -r '.[] | .uid')

for uid in $DASHBOARDS; do
  echo "Backing up dashboard: $uid"

  # Exporter le dashboard
  curl -s -u ${GRAFANA_USER}:${GRAFANA_PASSWORD} \
    ${GRAFANA_URL}/api/dashboards/uid/${uid} | \
    jq '.dashboard' > "${BACKUP_DIR}/${uid}.json"
done

echo "Backup complete in $BACKUP_DIR"

# Git commit
cd dashboard-backups
git add .
git commit -m "Dashboard backup $(date +%Y-%m-%d)"
```

### Restauration

```bash
#!/bin/bash
# restore-dashboard.sh

DASHBOARD_FILE=$1
GRAFANA_URL="http://localhost:3000"
GRAFANA_USER="admin"
GRAFANA_PASSWORD="admin"

if [ -z "$DASHBOARD_FILE" ]; then
  echo "Usage: $0 <dashboard.json>"
  exit 1
fi

# Préparer le payload
IMPORT_PAYLOAD=$(cat <<EOF
{
  "dashboard": $(cat $DASHBOARD_FILE),
  "overwrite": true,
  "folderId": 0
}
EOF
)

# Importer
curl -X POST \
  -H "Content-Type: application/json" \
  -u ${GRAFANA_USER}:${GRAFANA_PASSWORD} \
  ${GRAFANA_URL}/api/dashboards/db \
  -d "${IMPORT_PAYLOAD}"
```

## Troubleshooting des dashboards importés

### Problème : "No data" sur tous les panels

**Causes possibles :**
- Mauvaise datasource
- Labels différents
- Métriques manquantes

**Solutions :**
```bash
# Vérifier que les métriques existent
curl http://prometheus:9090/api/v1/label/__name__/values | grep "kube_"

# Vérifier les labels
curl http://prometheus:9090/api/v1/label/job/values

# Adapter les requêtes
# Original : {job="kubernetes-nodes"}
# Adapter : {job="node-exporter"}
```

### Problème : Panels vides après import

**Solution : Mapper correctement la datasource**
```json
{
  "inputs": [
    {
      "name": "DS_PROMETHEUS",
      "type": "datasource",
      "pluginId": "prometheus",
      "value": "Prometheus"  // Nom exact de votre datasource
    }
  ]
}
```

### Problème : Variables ne fonctionnent pas

**Solution : Vérifier et adapter les queries**
```json
{
  "templating": {
    "list": [
      {
        "name": "namespace",
        // Vérifier que cette métrique existe
        "query": "label_values(kube_pod_info, namespace)",
        // Spécifier la datasource
        "datasource": "Prometheus"
      }
    ]
  }
}
```

## Dashboards recommandés par use case

### Pour le monitoring de production

```yaml
Essentiels:
- Kubernetes Cluster Overview (8588)
- Node Exporter Full (1860)
- Kubernetes Pods (15760)
- Alert Manager (9578)

Recommandés:
- NGINX Ingress (9614)
- Persistent Volumes (13646)
- Network Traffic (15757)
```

### Pour le développement

```yaml
Essentiels:
- Kubernetes Pods Simple (6417)
- Container Overview (10000)
- Deployments (8349)

Utiles:
- ConfigMaps & Secrets (15758)
- Jobs & CronJobs (15759)
```

### Pour le capacity planning

```yaml
Essentiels:
- Cluster Autoscaler (3831)
- Resource Quotas (15761)
- Namespace Resources (15141)

Analytics:
- Cost Analysis (15762)
- Trend Analysis (15763)
```

## Maintenance des dashboards

### Mise à jour régulière

```bash
#!/bin/bash
# update-dashboards.sh

# Vérifier les nouvelles versions
for dashboard_id in 8588 1860 15760; do
  LATEST=$(curl -s https://grafana.com/api/dashboards/${dashboard_id} | \
    jq -r '.revision')
  echo "Dashboard $dashboard_id - Latest revision: $LATEST"
done
```

### Nettoyage des dashboards inutilisés

```bash
# Identifier les dashboards non utilisés (pas vus depuis 30 jours)
curl -s -u admin:admin \
  http://localhost:3000/api/search?type=dash-db | \
  jq '.[] | select(.lastSeen < (now - 2592000000)) | .title'
```

## Checklist d'import de dashboards

- [ ] Datasource Prometheus configurée et testée
- [ ] Dashboard importé avec succès
- [ ] Tous les panels affichent des données
- [ ] Variables de dashboard fonctionnelles
- [ ] Time range approprié configuré
- [ ] Seuils d'alerte ajustés si nécessaire
- [ ] Dashboard organisé dans le bon dossier
- [ ] Permissions configurées si multi-utilisateurs
- [ ] Backup du dashboard effectué
- [ ] Documentation des modifications

## Préparation pour la suite

Avec vos dashboards Kubernetes pré-configurés importés et fonctionnels, vous êtes prêt à :
- Créer vos propres dashboards personnalisés (section 8.09)
- Maîtriser les variables et le templating (section 8.10)
- Configurer l'alerting basé sur ces dashboards (section 8.11)

Les dashboards pré-configurés sont un excellent point de départ. Utilisez-les comme base, apprenez de leur construction, puis personnalisez-les selon vos besoins spécifiques.

⏭️

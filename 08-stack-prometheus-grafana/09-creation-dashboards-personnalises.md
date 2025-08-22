🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.09 - Création de dashboards personnalisés

## Introduction à la création de dashboards

### Qu'est-ce qu'un dashboard personnalisé ?

Un dashboard personnalisé est un tableau de bord que vous créez de zéro ou adaptez selon vos besoins spécifiques. Contrairement aux dashboards pré-configurés, vous avez un contrôle total sur :

- **Les métriques affichées** : Exactement ce qui vous intéresse
- **La disposition** : Organisation selon votre logique
- **Le design** : Couleurs, tailles, types de graphiques
- **Les interactions** : Liens, drill-down, variables

### Anatomie d'un dashboard Grafana

```
┌────────────────────────────────────────────────────────┐
│                    Dashboard                           │
├────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────┐  │
│  │                  Header/Title                    │  │
│  │         Variables    Time Range    Refresh      │  │
│  └─────────────────────────────────────────────────┘  │
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │    Panel 1   │  │    Panel 2   │  │  Panel 3   │  │
│  │   (Graph)    │  │   (Gauge)    │  │   (Stat)   │  │
│  └──────────────┘  └──────────────┘  └────────────┘  │
│                                                        │
│  ┌─────────────────────────┐  ┌────────────────────┐  │
│  │        Panel 4          │  │      Panel 5       │  │
│  │        (Table)          │  │      (Heatmap)     │  │
│  └─────────────────────────┘  └────────────────────┘  │
│                                                        │
│  ┌─────────────────────────────────────────────────┐  │
│  │                    Panel 6                       │  │
│  │                  (Time series)                   │  │
│  └─────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

## Création d'un nouveau dashboard

### Étape 1 : Créer un dashboard vide

1. **Via l'interface :**
   - Cliquer sur "+" dans le menu latéral
   - Sélectionner "Dashboard"
   - Cliquer sur "Add new panel"

2. **Via API :**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -u admin:admin \
  http://localhost:3000/api/dashboards/db \
  -d '{
    "dashboard": {
      "title": "Mon Dashboard Custom",
      "panels": [],
      "schemaVersion": 27,
      "version": 0
    },
    "overwrite": false
  }'
```

### Étape 2 : Configuration générale du dashboard

```json
{
  "dashboard": {
    "title": "Monitoring Application Lab",
    "description": "Dashboard personnalisé pour mon lab Kubernetes",
    "tags": ["kubernetes", "custom", "lab"],
    "timezone": "browser",
    "editable": true,
    "graphTooltip": 1,  // 0=none, 1=shared crosshair, 2=shared tooltip
    "time": {
      "from": "now-3h",
      "to": "now"
    },
    "refresh": "30s"
  }
}
```

## Types de panels disponibles

### 1. Time Series (Graphique temporel)

**Utilisation :** Évolution de métriques dans le temps
**Idéal pour :** CPU, mémoire, requêtes/sec, latence

```json
{
  "type": "timeseries",
  "title": "CPU Usage Over Time",
  "targets": [
    {
      "expr": "rate(container_cpu_usage_seconds_total[5m])",
      "legendFormat": "{{pod}}"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "percentunit",
      "min": 0,
      "max": 1
    }
  }
}
```

### 2. Gauge (Jauge)

**Utilisation :** Valeur actuelle avec seuils visuels
**Idéal pour :** Pourcentages, utilisation, saturation

```json
{
  "type": "gauge",
  "title": "Memory Usage",
  "targets": [
    {
      "expr": "(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "percent",
      "thresholds": {
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 70 },
          { "color": "red", "value": 90 }
        ]
      }
    }
  }
}
```

### 3. Stat (Statistique)

**Utilisation :** Valeur unique mise en évidence
**Idéal pour :** KPIs, compteurs, valeurs importantes

```json
{
  "type": "stat",
  "title": "Total Pods Running",
  "targets": [
    {
      "expr": "count(kube_pod_status_phase{phase=\"Running\"})"
    }
  ],
  "options": {
    "colorMode": "background",
    "graphMode": "area",
    "textMode": "value_and_name"
  }
}
```

### 4. Bar Gauge (Barre de progression)

**Utilisation :** Comparaison de valeurs multiples
**Idéal pour :** Top N, comparaisons, rankings

```json
{
  "type": "bargauge",
  "title": "Namespace Memory Usage",
  "targets": [
    {
      "expr": "sum by(namespace)(container_memory_usage_bytes) / 1024 / 1024 / 1024"
    }
  ],
  "options": {
    "orientation": "horizontal",
    "displayMode": "gradient",
    "showUnfilled": true
  }
}
```

### 5. Table (Tableau)

**Utilisation :** Données tabulaires détaillées
**Idéal pour :** Listes, inventaires, détails

```json
{
  "type": "table",
  "title": "Pod Status",
  "targets": [
    {
      "expr": "kube_pod_info",
      "format": "table",
      "instant": true
    }
  ],
  "transformations": [
    {
      "id": "organize",
      "options": {
        "excludeByName": {
          "Time": true
        },
        "renameByName": {
          "pod": "Pod Name",
          "namespace": "Namespace"
        }
      }
    }
  ]
}
```

### 6. Pie Chart (Camembert)

**Utilisation :** Répartition proportionnelle
**Idéal pour :** Distribution, parts de marché, allocation

```json
{
  "type": "piechart",
  "title": "Resources by Namespace",
  "targets": [
    {
      "expr": "sum by(namespace)(kube_pod_container_resource_requests{resource=\"cpu\"})"
    }
  ],
  "options": {
    "pieType": "donut",
    "legendDisplayMode": "table",
    "legendPlacement": "right"
  }
}
```

### 7. Heatmap (Carte de chaleur)

**Utilisation :** Distribution de valeurs dans le temps
**Idéal pour :** Latences, distributions, patterns

```json
{
  "type": "heatmap",
  "title": "Request Duration Distribution",
  "targets": [
    {
      "expr": "sum(increase(http_request_duration_seconds_bucket[1m])) by (le)",
      "format": "heatmap"
    }
  ],
  "options": {
    "calculate": true,
    "calculation": {
      "yBuckets": {
        "mode": "count",
        "value": "10"
      }
    }
  }
}
```

## Création d'un dashboard complet : Exemple pratique

### Dashboard "Santé du Cluster Kubernetes"

```json
{
  "dashboard": {
    "title": "Kubernetes Cluster Health - Custom",
    "panels": [
      {
        "id": 1,
        "gridPos": { "h": 4, "w": 6, "x": 0, "y": 0 },
        "type": "stat",
        "title": "Cluster Status",
        "targets": [
          {
            "expr": "sum(up{job=\"kube-state-metrics\"})",
            "legendFormat": "Healthy"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "mappings": [
              { "type": "value", "value": "1", "text": "Healthy" },
              { "type": "value", "value": "0", "text": "Unhealthy" }
            ],
            "thresholds": {
              "steps": [
                { "color": "red", "value": null },
                { "color": "green", "value": 1 }
              ]
            }
          }
        }
      },
      {
        "id": 2,
        "gridPos": { "h": 4, "w": 6, "x": 6, "y": 0 },
        "type": "stat",
        "title": "Total Nodes",
        "targets": [
          {
            "expr": "count(kube_node_info)"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "short",
            "color": { "mode": "thresholds" }
          }
        }
      },
      {
        "id": 3,
        "gridPos": { "h": 4, "w": 6, "x": 12, "y": 0 },
        "type": "stat",
        "title": "Running Pods",
        "targets": [
          {
            "expr": "count(kube_pod_status_phase{phase=\"Running\"})"
          }
        ]
      },
      {
        "id": 4,
        "gridPos": { "h": 4, "w": 6, "x": 18, "y": 0 },
        "type": "stat",
        "title": "Failed Pods",
        "targets": [
          {
            "expr": "count(kube_pod_status_phase{phase=\"Failed\"})"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "thresholds": {
              "steps": [
                { "color": "green", "value": null },
                { "color": "red", "value": 1 }
              ]
            }
          }
        }
      },
      {
        "id": 5,
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 4 },
        "type": "timeseries",
        "title": "CPU Usage by Node",
        "targets": [
          {
            "expr": "100 - (avg by(instance)(rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "{{instance}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "max": 100
          }
        }
      },
      {
        "id": 6,
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 4 },
        "type": "timeseries",
        "title": "Memory Usage by Node",
        "targets": [
          {
            "expr": "(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100",
            "legendFormat": "{{instance}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "max": 100
          }
        }
      },
      {
        "id": 7,
        "gridPos": { "h": 8, "w": 8, "x": 0, "y": 12 },
        "type": "piechart",
        "title": "Pods by Namespace",
        "targets": [
          {
            "expr": "count by(namespace)(kube_pod_info)",
            "instant": true
          }
        ]
      },
      {
        "id": 8,
        "gridPos": { "h": 8, "w": 8, "x": 8, "y": 12 },
        "type": "bargauge",
        "title": "Top 5 CPU Consumers",
        "targets": [
          {
            "expr": "topk(5, sum by(pod)(rate(container_cpu_usage_seconds_total[5m])))"
          }
        ],
        "options": {
          "orientation": "horizontal",
          "displayMode": "basic"
        }
      },
      {
        "id": 9,
        "gridPos": { "h": 8, "w": 8, "x": 16, "y": 12 },
        "type": "table",
        "title": "Pod Restarts",
        "targets": [
          {
            "expr": "kube_pod_container_status_restarts_total > 0",
            "format": "table",
            "instant": true
          }
        ]
      }
    ]
  }
}
```

## Configuration des panels

### Options communes à tous les panels

```json
{
  "panel": {
    // Identification
    "id": 1,
    "title": "Mon Panel",
    "description": "Description détaillée",

    // Position et taille
    "gridPos": {
      "h": 8,    // Hauteur (unités de grille)
      "w": 12,   // Largeur (24 = pleine largeur)
      "x": 0,    // Position X
      "y": 0     // Position Y
    },

    // Options générales
    "transparent": false,
    "repeat": null,  // Variable pour répétition
    "repeatDirection": "h",  // h=horizontal, v=vertical

    // Liens
    "links": [
      {
        "title": "Documentation",
        "url": "https://docs.example.com",
        "targetBlank": true
      }
    ]
  }
}
```

### Configuration des axes et légendes

```json
{
  "fieldConfig": {
    "defaults": {
      // Unités
      "unit": "percent",  // bytes, ops, percent, etc.
      "decimals": 2,

      // Limites
      "min": 0,
      "max": 100,

      // Couleurs
      "color": {
        "mode": "palette-classic"  // ou "thresholds"
      },

      // Seuils
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 60 },
          { "color": "red", "value": 80 }
        ]
      },

      // Mappings
      "mappings": [
        { "type": "value", "value": "null", "text": "N/A" },
        { "type": "value", "value": "0", "text": "Down" },
        { "type": "value", "value": "1", "text": "Up" }
      ]
    },

    // Overrides par série
    "overrides": [
      {
        "matcher": { "id": "byName", "options": "CPU" },
        "properties": [
          { "id": "color", "value": { "fixedColor": "blue" } }
        ]
      }
    ]
  }
}
```

### Transformations de données

```json
{
  "transformations": [
    // Organiser les colonnes
    {
      "id": "organize",
      "options": {
        "excludeByName": {
          "Time": true,
          "__name__": true
        },
        "renameByName": {
          "Value": "Current",
          "pod": "Pod Name"
        }
      }
    },

    // Calculer des champs
    {
      "id": "calculateField",
      "options": {
        "mode": "reduceRow",
        "reduce": {
          "reducer": "sum"
        },
        "alias": "Total"
      }
    },

    // Filtrer les résultats
    {
      "id": "filterByValue",
      "options": {
        "filters": [
          {
            "fieldName": "Value",
            "config": {
              "id": "greater",
              "options": { "value": 0 }
            }
          }
        ]
      }
    }
  ]
}
```

## Variables et templating

### Création de variables

```json
{
  "templating": {
    "list": [
      // Variable de requête
      {
        "name": "namespace",
        "label": "Namespace",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(kube_pod_info, namespace)",
        "refresh": 2,  // On time range change
        "sort": 1,     // Alphabétique
        "multi": true,
        "includeAll": true
      },

      // Variable custom
      {
        "name": "interval",
        "label": "Interval",
        "type": "custom",
        "options": [
          { "text": "1m", "value": "1m" },
          { "text": "5m", "value": "5m" },
          { "text": "10m", "value": "10m" }
        ],
        "current": {
          "text": "5m",
          "value": "5m"
        }
      },

      // Variable constante
      {
        "name": "cluster",
        "type": "constant",
        "value": "production"
      },

      // Variable de datasource
      {
        "name": "datasource",
        "type": "datasource",
        "query": "prometheus",
        "current": {
          "text": "Prometheus",
          "value": "Prometheus"
        }
      }
    ]
  }
}
```

### Utilisation des variables dans les requêtes

```promql
# Dans les requêtes PromQL
kube_pod_info{namespace=~"$namespace"}

# Avec multiple values
kube_pod_info{namespace=~"${namespace:regex}"}

# Avec interval
rate(http_requests_total[$interval])

# All format
kube_pod_info{namespace=~"${namespace:pipe}"}
```

## Row et organisation des panels

### Création de rows (lignes)

```json
{
  "panels": [
    // Row collapsed
    {
      "type": "row",
      "title": "Node Metrics",
      "gridPos": { "h": 1, "w": 24, "x": 0, "y": 0 },
      "collapsed": true,
      "panels": [
        // Panels dans la row...
      ]
    },

    // Row expanded
    {
      "type": "row",
      "title": "Pod Metrics",
      "gridPos": { "h": 1, "w": 24, "x": 0, "y": 10 },
      "collapsed": false
    }
    // Panels suivants font partie de cette row
  ]
}
```

### Layout responsive

```json
// Disposition pour différentes tailles d'écran
{
  "panels": [
    // Mobile: 1 colonne
    { "gridPos": { "h": 8, "w": 24, "x": 0, "y": 0 } },

    // Tablet: 2 colonnes
    { "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 } },
    { "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 } },

    // Desktop: 3-4 colonnes
    { "gridPos": { "h": 8, "w": 6, "x": 0, "y": 0 } },
    { "gridPos": { "h": 8, "w": 6, "x": 6, "y": 0 } },
    { "gridPos": { "h": 8, "w": 6, "x": 12, "y": 0 } },
    { "gridPos": { "h": 8, "w": 6, "x": 18, "y": 0 } }
  ]
}
```

## Annotations et événements

### Configuration des annotations

```json
{
  "annotations": {
    "list": [
      {
        "name": "Deployments",
        "datasource": "Prometheus",
        "enable": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "query": "changes(kube_deployment_created[5m])",
        "tagKeys": "deployment,namespace",
        "textFormat": "Deployment {{deployment}} updated"
      },
      {
        "name": "Alerts",
        "datasource": "Prometheus",
        "enable": true,
        "iconColor": "rgba(255, 96, 96, 1)",
        "query": "ALERTS{alertstate=\"firing\"}",
        "tagKeys": "alertname,severity"
      }
    ]
  }
}
```

## Liens et navigation

### Liens entre dashboards

```json
{
  "links": [
    // Lien vers autre dashboard
    {
      "title": "Detailed View",
      "type": "dashboard",
      "dashboard": "kubernetes-pods",
      "params": "var-namespace=$namespace"
    },

    // Lien externe
    {
      "title": "Documentation",
      "type": "link",
      "url": "https://docs.example.com",
      "targetBlank": true
    }
  ]
}
```

### Data links dans les panels

```json
{
  "fieldConfig": {
    "defaults": {
      "links": [
        {
          "title": "View Pod Details",
          "url": "/d/pod-details?var-pod=${__field.labels.pod}&var-namespace=${__field.labels.namespace}",
          "targetBlank": false
        }
      ]
    }
  }
}
```

## Bonnes pratiques de design

### 1. Organisation logique

```yaml
Structure recommandée:
- Row 1: KPIs principaux (stats)
- Row 2: Tendances (time series)
- Row 3: Détails (tables, listes)
- Row 4: Diagnostics (logs, events)
```

### 2. Couleurs cohérentes

```json
// Palette de couleurs cohérente
{
  "fieldConfig": {
    "defaults": {
      "thresholds": {
        "steps": [
          { "color": "#73BF69", "value": null },  // Vert
          { "color": "#F2CC0C", "value": 70 },    // Jaune
          { "color": "#FF9830", "value": 85 },    // Orange
          { "color": "#E02F44", "value": 95 }     // Rouge
        ]
      }
    }
  }
}
```

### 3. Titres descriptifs

```json
// ✅ Bon
"title": "CPU Usage by Pod (5min avg)"

// ❌ Mauvais
"title": "CPU"
```

### 4. Unités appropriées

```json
{
  "fieldConfig": {
    "defaults": {
      "unit": "percent",     // Pour pourcentages
      "unit": "bytes",       // Pour mémoire
      "unit": "ops",         // Pour operations/sec
      "unit": "ms",          // Pour latence
      "unit": "short"        // Pour compteurs
    }
  }
}
```

## Export et partage

### Export JSON du dashboard

```bash
# Via API
curl -u admin:admin \
  http://localhost:3000/api/dashboards/uid/my-dashboard \
  | jq '.dashboard' > my-dashboard.json
```

### Partage via snapshot

```bash
# Créer un snapshot
curl -X POST \
  -H "Content-Type: application/json" \
  -u admin:admin \
  http://localhost:3000/api/snapshots \
  -d '{
    "dashboard": { ... },
    "expires": 3600
  }'
```

## Templates de dashboards custom

### Template 1 : Application Monitoring

```json
{
  "dashboard": {
    "title": "Application ${app_name}",
    "panels": [
      // Santé
      { "title": "Health Status", "type": "stat" },
      { "title": "Uptime", "type": "stat" },

      // Performance
      { "title": "Request Rate", "type": "timeseries" },
      { "title": "Response Time", "type": "timeseries" },
      { "title": "Error Rate", "type": "timeseries" },

      // Ressources
      { "title": "CPU Usage", "type": "gauge" },
      { "title": "Memory Usage", "type": "gauge" },

      // Business metrics
      { "title": "Active Users", "type": "stat" },
      { "title": "Transactions/min", "type": "stat" }
    ]
  }
}
```

### Template 2 : Infrastructure Overview

```json
{
  "dashboard": {
    "title": "Infrastructure Overview",
    "panels": [
      // Nodes
      { "title": "Node Status", "type": "stat" },
      { "title": "Node Resources", "type": "bargauge" },

      // Workloads
      { "title": "Deployments", "type": "table" },
      { "title": "StatefulSets", "type": "table" },

      // Storage
      { "title": "PV Usage", "type": "piechart" },
      { "title": "Storage Trends", "type": "timeseries" }
    ]
  }
}
```

## Troubleshooting

### Problème : Panel sans données

```bash
# Vérifier la requête dans Prometheus
curl -G http://prometheus:9090/api/v1/query \
  --data-urlencode 'query=YOUR_QUERY'

# Vérifier les variables
echo "Variable value: $namespace"
```

### Problème : Performance lente

Solutions :
- Réduire le nombre de séries (utiliser aggregations)
- Augmenter l'intervalle minimum
- Utiliser le cache
- Limiter le time range

### Problème : Layout cassé

```json
// Réinitialiser les positions
{
  "panels": [
    { "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 } },
    { "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 } },
    { "gridPos": { "h": 8, "w": 24, "x": 0, "y": 8 } }
  ]
}
```

## Checklist de création

- [ ] Dashboard créé avec titre descriptif
- [ ] Panels organisés logiquement
- [ ] Variables définies si nécessaire
- [ ] Time range approprié configuré
- [ ] Refresh rate adapté (pas trop fréquent)
- [ ] Unités correctes sur tous les panels
- [ ] Seuils configurés pour les alertes visuelles
- [ ] Légendes claires et concises
- [ ] Description du dashboard ajoutée
- [ ] Tags pour faciliter la recherche
- [ ] Permissions configurées si multi-users
- [ ] Export JSON sauvegardé

## Préparation pour la suite

Avec vos compétences en création de dashboards personnalisés, vous êtes prêt à :
- Maîtriser les variables et le templating avancé (section 8.10)
- Configurer l'alerting basé sur vos dashboards (section 8.11)
- Optimiser les performances avec les recording rules (section 8.13)

La création de dashboards personnalisés est un art qui s'améliore avec la pratique. Commencez simple, puis enrichissez progressivement vos dashboards selon vos besoins.

⏭️

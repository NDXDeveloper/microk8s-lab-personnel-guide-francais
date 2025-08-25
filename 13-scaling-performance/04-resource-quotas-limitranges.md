🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.4 Resource Quotas et LimitRanges

## Introduction aux Contraintes de Ressources

Resource Quotas et LimitRanges sont les gardiens de vos ressources Kubernetes. Imaginez que votre cluster est comme un immeuble d'appartements : Resource Quotas définit combien d'espace total chaque locataire (namespace) peut utiliser, tandis que LimitRanges définit la taille minimale et maximale de chaque pièce (pod/container). Ces mécanismes empêchent qu'une seule application gourmande consomme toutes les ressources, laissant les autres applications sans rien.

Dans un lab MicroK8s où chaque CPU et chaque Mo de RAM compte, ces outils deviennent essentiels pour faire coexister plusieurs projets, équipes simulées, ou environnements (dev, test, prod) sur le même hardware limité.

## Qu'est-ce qu'un Resource Quota ?

### Concept Fondamental

Un Resource Quota est un objet Kubernetes qui limite la consommation totale de ressources dans un namespace. C'est comme avoir un budget : peu importe comment vous le dépensez, vous ne pouvez pas dépasser le montant alloué.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev-team
spec:
  hard:
    requests.cpu: "10"      # 10 cores maximum en requests
    requests.memory: 20Gi   # 20 GiB maximum en requests
    limits.cpu: "20"        # 20 cores maximum en limits
    limits.memory: 40Gi     # 40 GiB maximum en limits
    persistentvolumeclaims: "5"  # Maximum 5 PVC
    pods: "50"              # Maximum 50 pods
```

### Types de Ressources Contrôlables

Resource Quotas peut limiter :

**Ressources de calcul** :
- `requests.cpu` : Total des CPU requests
- `requests.memory` : Total des memory requests
- `limits.cpu` : Total des CPU limits
- `limits.memory` : Total des memory limits

**Ressources de stockage** :
- `requests.storage` : Total des demandes de stockage
- `persistentvolumeclaims` : Nombre de PVC
- `<storage-class>.storageclass.storage.k8s.io/requests.storage` : Par storage class

**Objets Kubernetes** :
- `pods` : Nombre de pods
- `services` : Nombre de services
- `configmaps` : Nombre de ConfigMaps
- `secrets` : Nombre de secrets
- `replicationcontrollers` : Nombre de RC
- `services.loadbalancers` : Services de type LoadBalancer
- `services.nodeports` : Services de type NodePort

### Scopes et Priorités

Les quotas peuvent être scopés pour cibler des pods spécifiques :

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: high-priority-quota
  namespace: production
spec:
  hard:
    pods: "10"
    requests.cpu: "1000"
    requests.memory: "200Gi"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high"]
```

Scopes disponibles :
- `Terminating` : Pods avec durée de vie limitée
- `NotTerminating` : Pods permanents
- `BestEffort` : Pods sans requests/limits
- `NotBestEffort` : Pods avec requests/limits
- `PriorityClass` : Pods avec une priorité spécifique

## Qu'est-ce qu'un LimitRange ?

### Concept Fondamental

Un LimitRange définit les contraintes de ressources au niveau des pods et containers individuels. Si Resource Quota est le budget total, LimitRange définit les règles de dépense individuelle : montant minimum par transaction, maximum autorisé, et ratio acceptable.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-memory-limits
  namespace: dev-team
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    min:
      cpu: "50m"
      memory: "64Mi"
    max:
      cpu: "2"
      memory: "2Gi"
```

### Types de Limites

LimitRange peut s'appliquer à :

**Container** : Limites par container
```yaml
- type: Container
  max:
    cpu: "2"
  min:
    cpu: "100m"
  default:
    cpu: "500m"
  defaultRequest:
    cpu: "200m"
  maxLimitRequestRatio:
    cpu: "10"  # Limit peut être max 10x la request
```

**Pod** : Limites pour le pod entier
```yaml
- type: Pod
  max:
    cpu: "4"
    memory: "8Gi"
  min:
    cpu: "200m"
    memory: "256Mi"
```

**PersistentVolumeClaim** : Limites de stockage
```yaml
- type: PersistentVolumeClaim
  max:
    storage: "10Gi"
  min:
    storage: "1Gi"
```

## Implementation dans MicroK8s

### Activation et Vérification

Resource Quotas et LimitRanges sont natifs à Kubernetes, donc disponibles immédiatement dans MicroK8s :

```bash
# Vérifier que l'API est disponible
kubectl api-resources | grep -E "resourcequotas|limitranges"

# Créer un namespace de test
kubectl create namespace quota-demo

# Voir les quotas existants
kubectl get resourcequota --all-namespaces

# Voir les limitranges existants
kubectl get limitrange --all-namespaces
```

### Configuration Basique pour un Lab

Voici une configuration type pour un lab personnel :

```yaml
# namespace-with-limits.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "4"        # 4 cores pour l'équipe
    requests.memory: "8Gi"   # 8 GiB de RAM
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
    persistentvolumeclaims: "5"
    services: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: team-alpha-limits
  namespace: team-alpha
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    min:
      cpu: "10m"
      memory: "32Mi"
    max:
      cpu: "2"
      memory: "4Gi"
  - type: Pod
    max:
      cpu: "4"
      memory: "8Gi"
```

## Stratégies pour un Lab Personnel

### Stratégie 1 : Environnements Isolés

Créez des namespaces avec quotas pour simuler différents environnements :

```yaml
# dev-environment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    environment: development
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "2"       # Dev a moins de ressources
    requests.memory: "4Gi"
    pods: "30"              # Mais peut avoir plus de pods pour tester

---
# prod-environment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    environment: production
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: prod
spec:
  hard:
    requests.cpu: "6"       # Prod a plus de ressources
    requests.memory: "12Gi"
    pods: "15"              # Mais moins de pods (plus stables)
```

### Stratégie 2 : Quotas par Type d'Application

Différenciez les quotas selon le type de workload :

```yaml
# batch-jobs-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: batch-quota
  namespace: batch-processing
spec:
  hard:
    requests.cpu: "8"       # Beaucoup de CPU pour le calcul
    requests.memory: "4Gi"  # Moins de mémoire
    pods: "100"             # Beaucoup de jobs parallèles
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: Terminating  # Seulement pour les jobs

---
# web-apps-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: web-quota
  namespace: web-apps
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "16Gi"  # Plus de mémoire pour les apps web
    pods: "20"
    services: "20"           # Plus de services
    services.loadbalancers: "2"
```

### Stratégie 3 : Quotas Dynamiques par Période

Adaptez les quotas selon l'utilisation :

```bash
#!/bin/bash
# dynamic-quotas.sh

# Quotas de jour (9h-18h)
apply_day_quotas() {
    kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dynamic-quota
  namespace: shared
spec:
  hard:
    requests.cpu: "8"
    requests.memory: "16Gi"
EOF
}

# Quotas de nuit (18h-9h)
apply_night_quotas() {
    kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dynamic-quota
  namespace: shared
spec:
  hard:
    requests.cpu: "12"      # Plus de ressources la nuit
    requests.memory: "24Gi"
EOF
}

# Logique de changement
HOUR=$(date +%H)
if [[ $HOUR -ge 9 ]] && [[ $HOUR -lt 18 ]]; then
    apply_day_quotas
else
    apply_night_quotas
fi
```

## Cas d'Usage Pratiques

### Protection contre les Fuites de Ressources

Prévenez les erreurs de configuration :

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: memory-leak-protection
  namespace: development
spec:
  limits:
  - type: Container
    max:
      memory: "2Gi"  # Aucun container ne peut demander plus
    maxLimitRequestRatio:
      memory: "2"    # Limit max 2x la request (évite l'OOM)
  - type: Pod
    max:
      memory: "4Gi"  # Un pod entier ne peut pas dépasser 4Gi
```

### Fair-Share entre Équipes

Distribuez équitablement les ressources :

```yaml
# Configuration pour 3 équipes sur un cluster de 12 cores, 24Gi RAM
---
# Team A - 33% des ressources
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
---
# Team B - 33% des ressources
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-b-quota
  namespace: team-b
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
---
# Team C - 33% des ressources
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-c-quota
  namespace: team-c
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
```

### Quotas pour CI/CD

Limitez les ressources des pipelines :

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ci-quota
  namespace: ci-cd
spec:
  hard:
    pods: "5"               # Max 5 builds parallèles
    requests.cpu: "10"
    requests.memory: "20Gi"
    persistentvolumeclaims: "10"  # Pour les caches
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: Terminating  # Jobs temporaires seulement
```

## Monitoring et Alerting

### Visualiser l'Utilisation des Quotas

```bash
# Status détaillé d'un quota
kubectl describe resourcequota team-alpha-quota -n team-alpha

# Output typique :
# Name:            team-alpha-quota
# Namespace:       team-alpha
# Resource         Used    Hard
# --------         ----    ----
# limits.cpu       2500m   8
# limits.memory    3Gi     16Gi
# pods             5       20
# requests.cpu     1200m   4
# requests.memory  2Gi     8Gi
```

### Requêtes Prometheus pour Quotas

```promql
# Utilisation du quota CPU en pourcentage
100 * kube_resourcequota{type="used",resource="requests.cpu"}
  / kube_resourcequota{type="hard",resource="requests.cpu"}

# Namespaces proches de leur limite
kube_resourcequota{type="used"}
  / kube_resourcequota{type="hard"} > 0.8

# Pods rejetés pour cause de quota
rate(kube_pod_status_reason{reason="ResourceQuotaExceeded"}[5m])
```

### Dashboard Grafana pour Quotas

```json
{
  "dashboard": {
    "title": "Resource Quotas Overview",
    "panels": [
      {
        "title": "CPU Quota Usage by Namespace",
        "targets": [{
          "expr": "kube_resourcequota{resource=\"requests.cpu\",type=\"used\"} / kube_resourcequota{resource=\"requests.cpu\",type=\"hard\"}"
        }]
      },
      {
        "title": "Memory Quota Usage by Namespace",
        "targets": [{
          "expr": "kube_resourcequota{resource=\"requests.memory\",type=\"used\"} / kube_resourcequota{resource=\"requests.memory\",type=\"hard\"}"
        }]
      },
      {
        "title": "Pods Count vs Quota",
        "targets": [{
          "expr": "kube_resourcequota{resource=\"pods\"}"
        }]
      }
    ]
  }
}
```

### Alertes sur les Quotas

```yaml
# prometheus-rules.yaml
groups:
- name: quota-alerts
  rules:
  - alert: QuotaNearLimit
    expr: |
      100 * kube_resourcequota{type="used"}
        / kube_resourcequota{type="hard"} > 90
    for: 5m
    annotations:
      summary: "Namespace {{ $labels.namespace }} near quota limit"
      description: "{{ $labels.resource }} usage at {{ $value }}%"

  - alert: QuotaExceeded
    expr: |
      increase(kube_pod_status_reason{reason="ResourceQuotaExceeded"}[5m]) > 0
    annotations:
      summary: "Pods rejected due to quota in {{ $labels.namespace }}"
```

## Interaction avec les Autoscalers

### HPA et Resource Quotas

HPA peut être limité par les quotas :

```yaml
# HPA configuration
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
  namespace: team-alpha
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10  # Peut être limité par le quota de pods

# Si le quota limite à 5 pods et 3 sont déjà utilisés,
# HPA ne pourra scaler qu'à 5 maximum, pas 10
```

### VPA et LimitRanges

VPA doit respecter les LimitRanges :

```yaml
# LimitRange
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: team-beta
spec:
  limits:
  - type: Container
    max:
      cpu: "2"
      memory: "2Gi"

# VPA sera contraint par ces limites
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
  namespace: team-beta
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  # Les recommandations VPA ne dépasseront pas 2 CPU et 2Gi
```

## Patterns Avancés

### Quotas Hiérarchiques

Organisez les quotas en hiérarchie :

```yaml
# Quota parent pour toute l'organisation
apiVersion: v1
kind: ResourceQuota
metadata:
  name: org-total-quota
  namespace: org-parent
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "200Gi"

# Sous-quotas pour les départements
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dept-engineering-quota
  namespace: dept-engineering
spec:
  hard:
    requests.cpu: "60"    # 60% pour l'ingénierie
    requests.memory: "120Gi"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dept-analytics-quota
  namespace: dept-analytics
spec:
  hard:
    requests.cpu: "40"    # 40% pour l'analytique
    requests.memory: "80Gi"
```

### Quotas avec Classes de Priorité

Réservez des ressources pour les workloads critiques :

```yaml
# Priority Classes
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: true

---
# Quotas par priorité
apiVersion: v1
kind: ResourceQuota
metadata:
  name: high-priority-quota
  namespace: production
spec:
  hard:
    pods: "10"
    requests.cpu: "20"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high-priority"]

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: low-priority-quota
  namespace: production
spec:
  hard:
    pods: "50"
    requests.cpu: "10"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["low-priority"]
```

### LimitRanges Progressifs

Définissez des limites évolutives :

```yaml
# Pour développement - limites souples
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: "50m"
      memory: "64Mi"
    maxLimitRequestRatio:
      cpu: "20"      # Peut burst jusqu'à 20x
      memory: "10"

---
# Pour production - limites strictes
apiVersion: v1
kind: LimitRange
metadata:
  name: prod-limits
  namespace: production
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: "200m"
      memory: "256Mi"
    maxLimitRequestRatio:
      cpu: "2"       # Burst limité à 2x
      memory: "1.5"
```

## Troubleshooting

### Pods Bloqués par Quotas

Diagnostic :

```bash
# Voir pourquoi un pod ne démarre pas
kubectl describe pod mon-pod -n team-alpha

# Events typiques :
# Error creating: pods "mon-pod" is forbidden:
# exceeded quota: team-alpha-quota,
# requested: requests.cpu=2, used: requests.cpu=3, limited: requests.cpu=4

# Vérifier l'utilisation du quota
kubectl get resourcequota -n team-alpha -o yaml

# Libérer des ressources
kubectl delete pod old-pod -n team-alpha
```

### Conflits entre LimitRange et Requests

Résolution :

```bash
# Erreur typique
# Error creating pod: Pod "test" is invalid:
# spec.containers[0].resources.requests: Invalid value: "3":
# must be less than or equal to cpu limit

# Vérifier le LimitRange
kubectl describe limitrange -n mon-namespace

# Ajuster le déploiement
kubectl edit deployment mon-app -n mon-namespace
# Modifier les requests/limits pour respecter le LimitRange
```

### Calcul des Ressources Disponibles

Script helper :

```bash
#!/bin/bash
# check-quota-availability.sh

NAMESPACE=$1

echo "=== Quota Status for $NAMESPACE ==="

# Obtenir le quota
kubectl get resourcequota -n $NAMESPACE -o json | jq -r '.items[] |
  .metadata.name as $name |
  .status |
  "Quota: \($name)",
  "CPU: \(.used."requests.cpu" // "0") / \(.hard."requests.cpu" // "unlimited")",
  "Memory: \(.used."requests.memory" // "0") / \(.hard."requests.memory" // "unlimited")",
  "Pods: \(.used.pods // "0") / \(.hard.pods // "unlimited")",
  ""'

# Calculer les ressources libres
kubectl get resourcequota -n $NAMESPACE -o json | jq -r '.items[] |
  "Available:",
  "CPU: \((.hard."requests.cpu" // "0" | tonumber) - (.used."requests.cpu" // "0" | tonumber))",
  "Memory: \((.hard."requests.memory" // "0") - (.used."requests.memory" // "0"))"'
```

## Bonnes Pratiques

### 1. Toujours Définir les Defaults

```yaml
# Évite les pods sans limites
spec:
  limits:
  - type: Container
    default:
      cpu: "1"
      memory: "1Gi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
```

### 2. Laisser de la Marge

Ne consommez pas 100% du quota :

```yaml
# Gardez 20% de buffer
spec:
  hard:
    requests.cpu: "8"     # Si vous avez 10 cores
    requests.memory: "20Gi"  # Si vous avez 25Gi
```

### 3. Documenter les Quotas

```yaml
# Ajoutez des annotations
metadata:
  name: team-quota
  annotations:
    description: "Quota for Team Alpha - Contact: alpha@example.com"
    last-review: "2024-01-15"
    justification: "Based on Q4 usage patterns"
```

### 4. Réviser Régulièrement

```bash
# Script de révision mensuelle
#!/bin/bash

# Générer un rapport d'utilisation
for ns in $(kubectl get ns -o name | cut -d/ -f2); do
  echo "Namespace: $ns"
  kubectl describe resourcequota -n $ns 2>/dev/null | grep -A 20 "Resource"
  echo "---"
done > quota-report-$(date +%Y%m).txt
```

### 5. Alerter Avant la Limite

Configurez des alertes à 80% d'utilisation, pas à 100%.

## Exemples Complets

### Lab Multi-Tenant

Configuration complète pour un lab partagé :

```yaml
# tenant-1-complete.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-1
  labels:
    tenant: "alpha-corp"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: tenant-1
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "30"
    persistentvolumeclaims: "10"
    services.nodeports: "5"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota
  namespace: tenant-1
spec:
  hard:
    configmaps: "50"
    secrets: "50"
    services: "20"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: limits
  namespace: tenant-1
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    min:
      cpu: "10m"
      memory: "32Mi"
    max:
      cpu: "2"
      memory: "4Gi"
    maxLimitRequestRatio:
      cpu: "10"
      memory: "8"
  - type: Pod
    max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
  - type: PersistentVolumeClaim
    max:
      storage: "20Gi"
    min:
      storage: "1Gi"
```

### Environment de Test Automatisé

```yaml
# ci-environment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ci-tests
  labels:
    purpose: "automated-testing"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ci-quota
  namespace: ci-tests
spec:
  hard:
    pods: "10"
    requests.cpu: "5"
    requests.memory: "10Gi"
  # Seulement pour les jobs temporaires
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: Terminating
---
apiVersion: v1
kind: LimitRange
metadata:
  name: ci-limits
  namespace: ci-tests
spec:
  limits:
  - type: Container
    default:
      cpu: "1"
      memory: "2Gi"
    defaultRequest:
      cpu: "500m"
      memory: "1Gi"
    # Pas de minimum pour permettre des tests légers
    max:
      cpu: "2"
      memory: "4Gi"
```

## Conclusion

Resource Quotas et LimitRanges sont des outils essentiels pour gérer efficacement les ressources limitées d'un lab MicroK8s. Ils permettent de simuler des environnements multi-tenant, d'éviter les abus de ressources, et d'apprendre les bonnes pratiques de gouvernance des ressources. Dans un environnement de lab où chaque ressource compte, ces mécanismes garantissent que tous vos projets et expérimentations peuvent coexister harmonieusement. La clé est de commencer avec des limites conservatrices, de monitorer l'utilisation réelle, et d'ajuster progressivement pour trouver l'équilibre optimal entre protection et flexibilité.

⏭️

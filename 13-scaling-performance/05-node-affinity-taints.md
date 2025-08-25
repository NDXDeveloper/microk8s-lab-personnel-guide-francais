🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.5 Node Affinity et Taints

## Introduction au Placement des Pods

Node Affinity et Taints/Tolerations sont les mécanismes qui contrôlent où vos pods peuvent s'exécuter dans votre cluster. Imaginez votre cluster comme un parking : Node Affinity est comme préférer se garer près de l'entrée ou sous un arbre pour l'ombre, tandis que Taints/Tolerations sont comme les places réservées (handicapés, livraisons, VIP) où seuls certains véhicules autorisés peuvent se garer.

Dans un lab MicroK8s, même avec peu de nodes, ces mécanismes deviennent précieux pour simuler des environnements complexes, isoler des workloads, optimiser les performances, ou simplement apprendre comment Kubernetes orchestre le placement des pods dans des clusters de production.

## Comprendre Node Affinity

### Qu'est-ce que Node Affinity ?

Node Affinity est une façon d'attirer les pods vers certains nodes basée sur les labels des nodes. C'est plus flexible et expressif que l'ancien nodeSelector, permettant des règles complexes comme "préfère les nodes avec SSD mais accepte HDD si nécessaire".

Il existe deux types de Node Affinity :

**RequiredDuringSchedulingIgnoredDuringExecution** : Règles obligatoires. Le pod ne sera schedulé QUE sur les nodes qui correspondent.

**PreferredDuringSchedulingIgnoredDuringExecution** : Règles préférentielles. Kubernetes essaiera de respecter ces préférences mais placera le pod ailleurs si nécessaire.

### Syntaxe de Base

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - zone-a
  containers:
  - name: app
    image: nginx
```

### Opérateurs Disponibles

Node Affinity supporte plusieurs opérateurs :

- **In** : La valeur du label doit être dans la liste
- **NotIn** : La valeur du label ne doit pas être dans la liste
- **Exists** : Le label doit exister (peu importe sa valeur)
- **DoesNotExist** : Le label ne doit pas exister
- **Gt** : La valeur doit être supérieure (numérique)
- **Lt** : La valeur doit être inférieure (numérique)

## Comprendre Taints et Tolerations

### Qu'est-ce que les Taints ?

Les Taints sont appliqués aux nodes pour les rendre "répulsifs" aux pods. Un node tainté repousse tous les pods SAUF ceux qui ont une toleration correspondante. C'est comme mettre un panneau "Accès Interdit Sauf Autorisation" sur une porte.

Structure d'un taint :
```
key=value:effect
```

Les trois effets possibles :
- **NoSchedule** : Les nouveaux pods ne seront pas schedulés
- **PreferNoSchedule** : Évite si possible, mais accepte si nécessaire
- **NoExecute** : Évince même les pods déjà en cours d'exécution

### Qu'est-ce que les Tolerations ?

Les Tolerations permettent aux pods de tolérer les taints. C'est le "laissez-passer" qui autorise un pod à s'exécuter sur un node tainté.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-tolerant
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: app
    image: nvidia/cuda
```

## Configuration dans MicroK8s

### Labelliser les Nodes

Avant d'utiliser Node Affinity, vous devez labelliser vos nodes :

```bash
# Voir les nodes existants
kubectl get nodes

# Ajouter des labels à un node
kubectl label nodes worker-1 disktype=ssd
kubectl label nodes worker-1 zone=zone-a
kubectl label nodes worker-1 hardware=high-performance

kubectl label nodes worker-2 disktype=hdd
kubectl label nodes worker-2 zone=zone-b
kubectl label nodes worker-2 hardware=standard

# Vérifier les labels
kubectl get nodes --show-labels

# Voir les labels d'un node spécifique
kubectl describe node worker-1 | grep Labels
```

### Appliquer des Taints

```bash
# Ajouter un taint à un node
kubectl taint nodes worker-1 gpu=true:NoSchedule
kubectl taint nodes worker-2 maintenance=true:NoExecute

# Voir les taints d'un node
kubectl describe node worker-1 | grep Taints

# Retirer un taint (noter le - à la fin)
kubectl taint nodes worker-1 gpu=true:NoSchedule-

# Retirer tous les taints d'un node
kubectl patch node worker-1 -p '{"spec":{"taints":[]}}'
```

## Cas d'Usage en Lab

### Simulation de Zones de Disponibilité

Simulez un environnement multi-zone même avec 2-3 nodes :

```yaml
# Labellisation des nodes
kubectl label nodes master zone=zone-a
kubectl label nodes worker-1 zone=zone-b
kubectl label nodes worker-2 zone=zone-c

---
# Déploiement avec anti-affinity pour haute disponibilité
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - webapp
            topologyKey: "zone"
      containers:
      - name: app
        image: nginx
```

### Isolation de Workloads Sensibles

Dédier un node aux applications critiques :

```bash
# Tainter le node pour les workloads critiques
kubectl taint nodes worker-1 dedicated=critical:NoSchedule
kubectl label nodes worker-1 workload-type=critical
```

```yaml
# Déploiement critique avec toleration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: critical
  template:
    metadata:
      labels:
        app: critical
    spec:
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "critical"
        effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload-type
                operator: In
                values:
                - critical
      containers:
      - name: app
        image: critical-app:latest
```

### Nodes GPU/Spécialisés

Réserver des nodes avec hardware spécial :

```bash
# Node avec GPU
kubectl taint nodes gpu-node nvidia.com/gpu=true:NoSchedule
kubectl label nodes gpu-node hardware=gpu
```

```yaml
# Pod nécessitant GPU
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  tolerations:
  - key: nvidia.com/gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: hardware
            operator: In
            values:
            - gpu
  containers:
  - name: training
    image: tensorflow/tensorflow:latest-gpu
    resources:
      limits:
        nvidia.com/gpu: 1
```

## Patterns Avancés

### Affinity Pondérée Multi-Critères

Combinez plusieurs préférences avec des poids :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: optimized-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: optimized
  template:
    metadata:
      labels:
        app: optimized
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          # Préfère fortement les nodes avec SSD
          - weight: 100
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
                - nvme
          # Préfère moyennement les nodes high-performance
          - weight: 50
            preference:
              matchExpressions:
              - key: hardware
                operator: In
                values:
                - high-performance
          # Légère préférence pour zone-a
          - weight: 10
            preference:
              matchExpressions:
              - key: zone
                operator: In
                values:
                - zone-a
      containers:
      - name: app
        image: optimized-app:latest
```

### Pod Affinity et Anti-Affinity

Contrôlez la co-localisation des pods :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cache
  template:
    metadata:
      labels:
        app: cache
    spec:
      affinity:
        # Anti-affinity : pas deux caches sur le même node
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cache
            topologyKey: kubernetes.io/hostname
      containers:
      - name: redis
        image: redis:alpine
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        # Pod Affinity : préfère être près du cache
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - cache
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: webapp:latest
```

### Taints Temporaires pour Maintenance

Script pour maintenance de node :

```bash
#!/bin/bash
# node-maintenance.sh

NODE=$1
ACTION=$2

maintenance_start() {
    echo "Starting maintenance on $NODE"

    # Appliquer taint de maintenance
    kubectl taint nodes $NODE maintenance=true:NoExecute

    # Cordon le node (empêche nouveaux pods)
    kubectl cordon $NODE

    # Drain le node (évacue les pods existants)
    kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --force

    echo "Node $NODE ready for maintenance"
}

maintenance_end() {
    echo "Ending maintenance on $NODE"

    # Retirer le taint
    kubectl taint nodes $NODE maintenance=true:NoExecute-

    # Uncordon le node
    kubectl uncordon $NODE

    echo "Node $NODE back in service"
}

case $ACTION in
    start)
        maintenance_start
        ;;
    end)
        maintenance_end
        ;;
    *)
        echo "Usage: $0 <node> [start|end]"
        exit 1
        ;;
esac
```

### Stratégie de Déploiement Progressive

Utilisez les taints pour un déploiement canary :

```yaml
# Phase 1: Taint pour canary
kubectl taint nodes worker-2 canary=true:NoSchedule

---
# Déploiement canary avec toleration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      tolerations:
      - key: "canary"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - worker-2
      containers:
      - name: app
        image: myapp:v2.0
```

## Topologie et Spread Constraints

### Topology Spread Constraints

Distribuez uniformément les pods :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: balanced-app
spec:
  replicas: 10
  selector:
    matchLabels:
      app: balanced
  template:
    metadata:
      labels:
        app: balanced
    spec:
      topologySpreadConstraints:
      # Équilibrer entre zones
      - maxSkew: 1
        topologyKey: zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: balanced
      # Équilibrer entre nodes
      - maxSkew: 2
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: balanced
      containers:
      - name: app
        image: nginx
```

### Groupes de Nodes avec Labels

Organisez vos nodes en groupes logiques :

```bash
# Créer des groupes de nodes
kubectl label nodes master node-group=control-plane
kubectl label nodes worker-1 node-group=compute
kubectl label nodes worker-2 node-group=compute
kubectl label nodes worker-3 node-group=storage

# Tainter par groupe
kubectl taint nodes -l node-group=storage dedicated=storage:NoSchedule
```

```yaml
# Déploiement par groupe
apiVersion: apps/v1
kind: Deployment
metadata:
  name: compute-intensive
spec:
  replicas: 4
  selector:
    matchLabels:
      app: compute
  template:
    metadata:
      labels:
        app: compute
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-group
                operator: In
                values:
                - compute
      containers:
      - name: processor
        image: compute-app:latest
```

## Monitoring et Debugging

### Visualiser le Placement des Pods

```bash
# Voir où les pods sont placés
kubectl get pods -o wide --all-namespaces

# Voir pourquoi un pod n'est pas schedulé
kubectl describe pod mon-pod | grep -A 10 Events

# Analyser les décisions du scheduler
kubectl get events --field-selector reason=FailedScheduling

# Script pour visualiser la distribution
#!/bin/bash
echo "=== Pod Distribution par Node ==="
for node in $(kubectl get nodes -o name | cut -d/ -f2); do
    echo "Node: $node"
    kubectl get pods --all-namespaces --field-selector spec.nodeName=$node \
        --no-headers | wc -l
    echo "---"
done
```

### Métriques Prometheus

```promql
# Pods par node
count by (node) (kube_pod_info)

# Nodes avec taints
kube_node_spec_taint

# Pods non schedulés
kube_pod_status_phase{phase="Pending"} > 0

# Distribution des pods avec affinity
count by (node) (
  kube_pod_info{pod=~".*affinity.*"}
)
```

### Dashboard Grafana

```json
{
  "panels": [
    {
      "title": "Pod Distribution by Node",
      "targets": [{
        "expr": "count by (node) (kube_pod_info)"
      }]
    },
    {
      "title": "Tainted Nodes",
      "targets": [{
        "expr": "kube_node_spec_taint"
      }]
    },
    {
      "title": "Pending Pods (Scheduling Issues)",
      "targets": [{
        "expr": "kube_pod_status_phase{phase=\"Pending\"}"
      }]
    }
  ]
}
```

## Troubleshooting

### Pod Reste en Pending

Diagnostic et résolution :

```bash
# Identifier le problème
kubectl describe pod stuck-pod

# Erreurs communes :
# 1. "0/3 nodes are available: 3 node(s) had taint {key: value}"
#    → Ajouter la toleration appropriée

# 2. "0/3 nodes are available: 3 node(s) didn't match node selector"
#    → Vérifier les labels des nodes

# 3. "0/3 nodes are available: 3 Insufficient cpu"
#    → Pas de problème d'affinity, manque de ressources

# Vérifier les taints
kubectl get nodes -o json | jq '.items[].spec.taints'

# Vérifier les labels
kubectl get nodes --show-labels

# Forcer le scheduling (debug uniquement)
kubectl annotate pod stuck-pod scheduler.alpha.kubernetes.io/tolerations='[{"operator": "Exists"}]'
```

### Pods Évincés Inopinément

```bash
# Vérifier les taints NoExecute récents
kubectl get events --field-selector reason=TaintEviction

# Voir l'historique des taints d'un node
kubectl get events --field-selector involvedObject.name=worker-1

# Script pour protéger les pods critiques
kubectl patch deployment critical-app -p '
{
  "spec": {
    "template": {
      "spec": {
        "tolerations": [
          {
            "operator": "Exists",
            "effect": "NoExecute",
            "tolerationSeconds": 3600
          }
        ]
      }
    }
  }
}'
```

### Distribution Déséquilibrée

```bash
# Analyser la distribution
kubectl get pods -o json | jq -r '.items[] |
  "\(.spec.nodeName): \(.metadata.namespace)/\(.metadata.name)"' |
  sort | uniq -c | sort -rn

# Rebalancer manuellement
kubectl rollout restart deployment mon-app

# Forcer une redistribution
kubectl delete pod -l app=mon-app --grace-period=60
```

## Bonnes Pratiques

### 1. Utiliser des Labels Significatifs

```yaml
# Bon : labels descriptifs et hiérarchiques
metadata:
  labels:
    environment: production
    tier: frontend
    region: eu-west
    hardware: gpu-enabled

# Mauvais : labels vagues
metadata:
  labels:
    type: special
    group: 1
```

### 2. Préférer Preferred à Required

```yaml
# Plus flexible et résilient
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 100
  preference:
    matchExpressions:
    - key: disktype
      operator: In
      values: ["ssd"]

# À utiliser seulement si vraiment nécessaire
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: critical-hardware
      operator: Exists
```

### 3. Documenter les Taints

```bash
# Utiliser des noms explicites
kubectl taint nodes gpu-node nvidia.com/gpu=true:NoSchedule
kubectl taint nodes maintenance-node under-maintenance=true:NoExecute

# Ajouter des annotations
kubectl annotate node gpu-node \
  taint-reason="Reserved for GPU workloads only" \
  taint-owner="ml-team@example.com"
```

### 4. Combiner avec Pod Priority

```yaml
# Priority Class pour workloads critiques
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-priority
value: 1000
globalDefault: false
preemptionPolicy: PreemptLowerPriority

---
# Utilisation avec affinity
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
spec:
  priorityClassName: critical-priority
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-class
            operator: In
            values: ["premium"]
```

### 5. Tester les Configurations

```bash
#!/bin/bash
# test-scheduling.sh

# Créer un pod de test
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-scheduling
  labels:
    test: scheduling
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: test-label
            operator: In
            values: ["test-value"]
  tolerations:
  - key: "test-taint"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: test
    image: busybox
    command: ["sleep", "3600"]
EOF

# Vérifier où il est schedulé
sleep 5
kubectl get pod test-scheduling -o wide

# Nettoyer
kubectl delete pod test-scheduling
```

## Exemples Complets

### Architecture Multi-Tier

```yaml
# Frontend - préfère les nodes edge
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: node-role
                operator: In
                values: ["edge"]
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  tier: frontend
              topologyKey: kubernetes.io/hostname
      containers:
      - name: web
        image: frontend:latest

---
# Backend - nodes compute
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 5
  selector:
    matchLabels:
      tier: backend
  template:
    metadata:
      labels:
        tier: backend
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role
                operator: In
                values: ["compute"]
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 50
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  tier: cache
              topologyKey: zone
      containers:
      - name: api
        image: backend:latest

---
# Database - nodes storage avec taints
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: db
  replicas: 3
  selector:
    matchLabels:
      tier: database
  template:
    metadata:
      labels:
        tier: database
    spec:
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "storage"
        effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role
                operator: In
                values: ["storage"]
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                tier: database
            topologyKey: kubernetes.io/hostname
      containers:
      - name: postgres
        image: postgres:14
```

### Lab Multi-Environnement

```bash
#!/bin/bash
# setup-multi-env.sh

# Configuration des nodes
kubectl label nodes worker-1 environment=dev
kubectl label nodes worker-2 environment=staging
kubectl label nodes worker-3 environment=prod

# Taints pour isolation
kubectl taint nodes worker-3 environment=prod:NoSchedule

# Création des namespaces
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod

# Déploiement avec contraintes appropriées
for env in dev staging prod; do
  cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: $env
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        environment: $env
    spec:
      $(if [ "$env" = "prod" ]; then
        echo "tolerations:"
        echo "- key: environment"
        echo "  operator: Equal"
        echo "  value: prod"
        echo "  effect: NoSchedule"
      fi)
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: environment
                operator: In
                values: [$env]
      containers:
      - name: app
        image: myapp:$env
EOF
done
```

## Conclusion

Node Affinity et Taints/Tolerations sont des outils puissants pour contrôler précisément le placement des pods dans votre cluster MicroK8s. Même dans un petit lab avec quelques nodes, ces mécanismes permettent de simuler des architectures complexes, d'isoler des workloads, d'optimiser l'utilisation des ressources, et de préparer des déploiements pour la production. La clé est de commencer simple avec des labels et affinités basiques, puis d'augmenter progressivement la sophistication selon vos besoins. Ces concepts, une fois maîtrisés dans votre lab, vous donneront le contrôle fin nécessaire pour gérer efficacement des clusters de production de toute taille.

⏭️

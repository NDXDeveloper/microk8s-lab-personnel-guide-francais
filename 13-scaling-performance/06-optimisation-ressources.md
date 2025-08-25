🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.6 Optimisation des ressources

## Introduction à l'Optimisation

L'optimisation des ressources dans Kubernetes consiste à trouver l'équilibre parfait entre performance, stabilité et efficacité. C'est comme gérer le budget d'un foyer : vous voulez maximiser le confort tout en minimisant les dépenses inutiles. Dans un lab MicroK8s où chaque Mo de RAM et chaque cycle CPU compte, l'optimisation devient un art essentiel pour faire coexister le maximum d'applications sur votre hardware limité.

Cette section vous apprendra à identifier le gaspillage, implémenter des stratégies d'optimisation, et maintenir un cluster performant même avec des ressources contraintes.

## Comprendre la Consommation des Ressources

### Les Quatre Métriques Clés

Pour optimiser, il faut d'abord mesurer. Les quatre métriques essentielles sont :

**CPU Requests vs Usage** : La différence entre ce que vous réservez et ce que vous utilisez réellement
**Memory Requests vs Usage** : L'écart entre la mémoire allouée et consommée
**Storage IOPS** : Les opérations d'entrée/sortie qui peuvent devenir un goulot d'étranglement
**Network Bandwidth** : La bande passante réseau, souvent négligée mais critique

### Anatomie de la Consommation d'un Pod

Un pod consomme des ressources à plusieurs niveaux :

```yaml
# Ressources explicites
resources:
  requests:
    cpu: "100m"      # Réservation garantie
    memory: "128Mi"  # Mémoire minimale
  limits:
    cpu: "500m"      # Maximum autorisé
    memory: "512Mi"  # Plafond mémoire

# Ressources implicites
# - Overhead Kubernetes (~50-100MB par pod)
# - Réseau (iptables, DNS)
# - Stockage (logs, ephemeral storage)
# - Metadata (etcd entries)
```

### Outils de Mesure

```bash
# Vue instantanée de la consommation
kubectl top nodes
kubectl top pods --all-namespaces

# Détails par container
kubectl top pod mon-pod --containers

# Utilisation sur une période (avec metrics-server)
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes

# Script d'analyse complète
#!/bin/bash
echo "=== Analyse des Ressources du Cluster ==="
echo ""
echo "NODES:"
kubectl top nodes
echo ""
echo "TOP 10 PODS CPU:"
kubectl top pods --all-namespaces | sort -k3 -rn | head -10
echo ""
echo "TOP 10 PODS MEMORY:"
kubectl top pods --all-namespaces | sort -k4 -rn | head -10
```

## Stratégies d'Optimisation

### 1. Right-Sizing des Containers

Le right-sizing consiste à ajuster précisément les requests et limits :

```yaml
# AVANT : Sur-allocation par précaution
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            cpu: "1000m"     # Trop généreux
            memory: "2Gi"    # Rarement utilisé
          limits:
            cpu: "2000m"
            memory: "4Gi"

# APRÈS : Basé sur l'observation réelle
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            cpu: "100m"      # P50 de l'utilisation
            memory: "256Mi"  # P90 de l'utilisation
          limits:
            cpu: "500m"      # P99 + marge
            memory: "512Mi"  # P99 + buffer
```

### 2. Quality of Service (QoS) Classes

Kubernetes définit trois classes QoS qui influencent les décisions d'éviction :

```yaml
# Guaranteed : Requests = Limits
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "100m"
        memory: "256Mi"
      limits:
        cpu: "100m"      # Identique aux requests
        memory: "256Mi"  # Identique aux requests

---
# Burstable : Requests < Limits
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"      # Peut burst
        memory: "512Mi"  # Peut burst

---
# BestEffort : Pas de requests/limits
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
  - name: app
    image: nginx  # Aucune resource définie
```

### 3. Optimisation des Images

Réduire la taille des images diminue le temps de démarrage et l'utilisation du stockage :

```dockerfile
# AVANT : Image non optimisée (1.2GB)
FROM ubuntu:latest
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    gcc \
    python3-dev
COPY requirements.txt .
RUN pip3 install -r requirements.txt
COPY . /app
CMD ["python3", "/app/main.py"]

# APRÈS : Image optimisée (150MB)
FROM python:3.11-alpine AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM python:3.11-alpine
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "main.py"]
```

### 4. Partage de Ressources

Utilisez des ressources partagées quand possible :

```yaml
# ConfigMaps partagés
apiVersion: v1
kind: ConfigMap
metadata:
  name: shared-config
data:
  database.url: "postgres://db:5432"
  cache.url: "redis://cache:6379"

---
# Volumes partagés en lecture seule
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-reader
spec:
  replicas: 3
  template:
    spec:
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: shared-pvc
          readOnly: true
      containers:
      - name: reader
        volumeMounts:
        - name: shared-data
          mountPath: /data
          readOnly: true
```

## Optimisation CPU

### CPU Limits et Throttling

Comprendre le CPU throttling est crucial :

```yaml
# Configuration anti-throttling
apiVersion: v1
kind: Pod
metadata:
  name: cpu-optimized
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "250m"  # Garanti
      limits:
        cpu: "1000m" # 4x burst possible
    env:
    - name: GOMAXPROCS
      value: "1"     # Aligner avec les cores disponibles
    - name: NODE_OPTIONS
      value: "--max-old-space-size=256"  # Limiter la heap Node.js
```

### CPU Pinning et NUMA

Pour les workloads sensibles à la latence :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: latency-sensitive
  annotations:
    cpu-manager.kubernetes.io/policy: "static"
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "2"     # Nombre entier pour CPU pinning
        memory: "4Gi"
      limits:
        cpu: "2"
        memory: "4Gi"
```

### Monitoring du CPU Throttling

```bash
# Détecter le throttling
kubectl get --raw /api/v1/nodes/$(kubectl get nodes -o name | cut -d/ -f2 | head -1)/proxy/metrics/cadvisor | grep throttled

# Script de détection
#!/bin/bash
for pod in $(kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name --no-headers | tr -s ' ' '/'); do
  namespace=$(echo $pod | cut -d'/' -f1)
  name=$(echo $pod | cut -d'/' -f2)

  throttled=$(kubectl exec -n $namespace $name -- cat /sys/fs/cgroup/cpu/cpu.stat 2>/dev/null | grep nr_throttled | awk '{print $2}')

  if [[ $throttled -gt 0 ]]; then
    echo "Pod $namespace/$name is being throttled: $throttled times"
  fi
done
```

## Optimisation Mémoire

### Gestion de l'OOM Killer

Prévenir les OOM kills avec une configuration appropriée :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-safe
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            memory: "256Mi"
          limits:
            memory: "512Mi"  # 2x buffer pour les pics
        env:
        - name: JAVA_OPTS
          value: "-Xmx384m -Xms256m"  # 75% de la limit
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - "[ $(cat /proc/meminfo | grep MemAvailable | awk '{print $2}') -gt 50000 ]"
          periodSeconds: 30
```

### Memory Compaction et Swap

Configuration du node pour optimiser la mémoire :

```bash
# Désactiver le swap (recommandé pour Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Optimiser les paramètres kernel
cat <<EOF | sudo tee /etc/sysctl.d/k8s-memory.conf
# Réduire la tendance au swap
vm.swappiness=1

# Optimiser la gestion mémoire
vm.overcommit_memory=1
vm.panic_on_oom=0

# Compaction de la mémoire
vm.compact_memory=1
vm.compaction_proactiveness=20

# Cache et buffers
vm.dirty_ratio=15
vm.dirty_background_ratio=5
EOF

sudo sysctl -p /etc/sysctl.d/k8s-memory.conf
```

### Partage de Mémoire entre Containers

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-memory
spec:
  volumes:
  - name: dshm
    emptyDir:
      medium: Memory
      sizeLimit: 1Gi
  containers:
  - name: app1
    volumeMounts:
    - mountPath: /dev/shm
      name: dshm
  - name: app2
    volumeMounts:
    - mountPath: /dev/shm
      name: dshm
```

## Optimisation du Stockage

### Choix des Storage Classes

Optimisez selon le type de workload :

```yaml
# Storage Class pour données temporaires
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-temp
provisioner: microk8s.io/hostpath
parameters:
  type: DirectoryOrCreate
reclaimPolicy: Delete
volumeBindingMode: Immediate

---
# Storage Class pour données persistantes
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: reliable-data
provisioner: microk8s.io/hostpath
parameters:
  type: DirectoryOrCreate
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer

---
# Utilisation optimisée
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-cache
spec:
  accessModes:
    - ReadWriteMany  # Partage entre pods
  storageClassName: fast-temp
  resources:
    requests:
      storage: 5Gi
```

### Ephemeral Storage et Cleanup

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ephemeral-optimized
spec:
  containers:
  - name: app
    resources:
      requests:
        ephemeral-storage: "1Gi"
      limits:
        ephemeral-storage: "2Gi"
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  volumes:
  - name: cache-volume
    emptyDir:
      sizeLimit: 500Mi
  # Nettoyage automatique
  initContainers:
  - name: cleanup
    image: busybox
    command: ['sh', '-c', 'rm -rf /cache/* || true']
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
```

### Optimisation des Logs

Réduire l'impact des logs :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-optimized
spec:
  containers:
  - name: app
    image: myapp
    # Limiter la taille des logs
    terminationMessagePolicy: FallbackToLogsOnError
    terminationMessagePath: /dev/termination-log
    env:
    - name: LOG_LEVEL
      value: "WARN"  # Réduire la verbosité
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir:
      sizeLimit: 100Mi

---
# Rotation des logs au niveau du node
apiVersion: v1
kind: ConfigMap
metadata:
  name: logrotate-config
  namespace: kube-system
data:
  logrotate.conf: |
    /var/log/pods/*/*.log {
        rotate 3
        daily
        maxsize 100M
        missingok
        compress
        delaycompress
        notifempty
    }
```

## Optimisation Réseau

### Service Mesh Léger

Pour un lab, utilisez des solutions légères :

```yaml
# Linkerd (plus léger que Istio)
# Installation
curl -sL https://run.linkerd.io/install | sh

# Annotation pour injection automatique
apiVersion: v1
kind: Namespace
metadata:
  name: optimized-apps
  annotations:
    linkerd.io/inject: enabled

---
# Ou pas de service mesh du tout pour économiser
apiVersion: v1
kind: Service
metadata:
  name: direct-service
spec:
  type: ClusterIP
  clusterIP: None  # Headless service, pas de proxy
  selector:
    app: myapp
```

### Network Policies pour Réduire le Trafic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: minimize-traffic
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend  # Seul le backend peut accéder
    ports:
    - protocol: TCP
      port: 5432
```

### DNS Optimization

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-optimized
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    options:
    - name: ndots
      value: "2"  # Réduit les requêtes DNS
    - name: edns0
    - name: single-request-reopen
  containers:
  - name: app
    image: myapp
    env:
    - name: GODEBUG
      value: "netdns=cgo"  # Utiliser le resolver système
```

## Patterns d'Optimisation Avancés

### Batch Processing Optimisé

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: optimized-batch
spec:
  parallelism: 3
  completions: 10
  backoffLimit: 2
  ttlSecondsAfterFinished: 3600  # Cleanup automatique
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: processor
        image: batch-processor
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"     # Burst pour finir plus vite
            memory: "1Gi"
        env:
        - name: BATCH_SIZE
          value: "1000"     # Traiter par lots
        - name: PARALLEL_WORKERS
          value: "4"         # Parallélisme interne
```

### Sidecar Containers Optimisés

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: optimized-sidecar
spec:
  containers:
  # Container principal
  - name: app
    image: myapp
    resources:
      requests:
        cpu: "250m"
        memory: "512Mi"

  # Sidecar léger pour logs
  - name: log-forwarder
    image: fluent/fluent-bit:latest
    resources:
      requests:
        cpu: "10m"       # Très peu de CPU
        memory: "32Mi"   # Mémoire minimale
      limits:
        cpu: "50m"
        memory: "64Mi"
    volumeMounts:
    - name: varlog
      mountPath: /var/log
      readOnly: true

  volumes:
  - name: varlog
    emptyDir:
      sizeLimit: 100Mi
```

### Init Containers pour Préparation

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: optimized-startup
spec:
  template:
    spec:
      initContainers:
      # Préchauffer les caches
      - name: cache-warmer
        image: cache-warmer
        resources:
          requests:
            cpu: "1000m"    # Beaucoup de CPU temporairement
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "1Gi"

      containers:
      - name: app
        image: myapp
        resources:
          requests:
            cpu: "100m"     # Peu de CPU après le warmup
            memory: "256Mi"
```

## Monitoring et Métriques

### Dashboard d'Optimisation

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard
data:
  optimization-dashboard.json: |
    {
      "dashboard": {
        "title": "Resource Optimization",
        "panels": [
          {
            "title": "CPU Utilization vs Requests",
            "targets": [{
              "expr": "sum(rate(container_cpu_usage_seconds_total[5m])) by (pod) / sum(kube_pod_container_resource_requests{resource=\"cpu\"}) by (pod)"
            }]
          },
          {
            "title": "Memory Utilization vs Requests",
            "targets": [{
              "expr": "sum(container_memory_working_set_bytes) by (pod) / sum(kube_pod_container_resource_requests{resource=\"memory\"}) by (pod)"
            }]
          },
          {
            "title": "Wasted Resources (Requested but Unused)",
            "targets": [{
              "expr": "sum((kube_pod_container_resource_requests{resource=\"cpu\"} - rate(container_cpu_usage_seconds_total[5m])) * on(pod) group_left() (kube_pod_status_phase{phase=\"Running\"} == 1))"
            }]
          },
          {
            "title": "OOM Kills",
            "targets": [{
              "expr": "sum(rate(container_oom_events_total[5m])) by (pod)"
            }]
          }
        ]
      }
    }
```

### Alertes d'Optimisation

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-alerts
data:
  optimization.yaml: |
    groups:
    - name: optimization
      rules:
      - alert: HighResourceWaste
        expr: |
          (sum(kube_pod_container_resource_requests{resource="cpu"}) - sum(rate(container_cpu_usage_seconds_total[5m])))
          / sum(kube_pod_container_resource_requests{resource="cpu"}) > 0.5
        for: 30m
        annotations:
          summary: "Plus de 50% du CPU réservé est inutilisé"

      - alert: MemoryPressure
        expr: |
          node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.1
        for: 5m
        annotations:
          summary: "Moins de 10% de mémoire disponible sur {{ $labels.node }}"

      - alert: PodThrottling
        expr: |
          rate(container_cpu_throttled_seconds_total[5m]) > 0.5
        for: 10m
        annotations:
          summary: "Pod {{ $labels.pod }} est throttled"
```

## Scripts d'Analyse et Optimisation

### Analyseur de Gaspillage

```bash
#!/bin/bash
# waste-analyzer.sh

echo "=== Resource Waste Analysis ==="
echo ""

# Analyser le gaspillage CPU
echo "CPU WASTE (Requested - Used):"
kubectl get pods --all-namespaces -o json | jq -r '
  .items[] |
  select(.status.phase=="Running") |
  {
    namespace: .metadata.namespace,
    name: .metadata.name,
    requested: (.spec.containers[].resources.requests.cpu // "0"),
    node: .spec.nodeName
  }' | while read -r pod; do
    # Obtenir l'utilisation réelle via kubectl top
    usage=$(kubectl top pod $(echo $pod | jq -r .name) -n $(echo $pod | jq -r .namespace) --no-headers 2>/dev/null | awk '{print $2}')
    requested=$(echo $pod | jq -r .requested)

    if [[ ! -z "$usage" ]]; then
      echo "$pod | Usage: $usage"
    fi
done

# Recommandations
echo ""
echo "RECOMMENDATIONS:"
kubectl get vpa --all-namespaces -o json 2>/dev/null | jq -r '
  .items[] |
  {
    namespace: .metadata.namespace,
    name: .metadata.name,
    cpu: .status.recommendation.containerRecommendations[0].target.cpu,
    memory: .status.recommendation.containerRecommendations[0].target.memory
  }'
```

### Optimiseur Automatique

```bash
#!/bin/bash
# auto-optimizer.sh

# Fonction pour calculer le right-size
calculate_rightsize() {
    local namespace=$1
    local deployment=$2

    # Obtenir les métriques sur 7 jours (nécessite Prometheus)
    cpu_p90=$(curl -s "http://prometheus:9090/api/v1/query?query=quantile_over_time(0.9,rate(container_cpu_usage_seconds_total{namespace=\"$namespace\",pod=~\"$deployment.*\"}[7d])[1h:])" | jq -r '.data.result[0].value[1]')

    mem_p90=$(curl -s "http://prometheus:9090/api/v1/query?query=quantile_over_time(0.9,container_memory_working_set_bytes{namespace=\"$namespace\",pod=~\"$deployment.*\"}[7d])[1h:])" | jq -r '.data.result[0].value[1]')

    # Appliquer les nouvelles valeurs
    kubectl patch deployment $deployment -n $namespace --type='json' -p="[
        {'op': 'replace', 'path': '/spec/template/spec/containers/0/resources/requests/cpu', 'value': '${cpu_p90}m'},
        {'op': 'replace', 'path': '/spec/template/spec/containers/0/resources/requests/memory', 'value': '${mem_p90}Mi'}
    ]"
}

# Optimiser tous les deployments
for deploy in $(kubectl get deployments --all-namespaces -o json | jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name)"'); do
    namespace=$(echo $deploy | cut -d'/' -f1)
    name=$(echo $deploy | cut -d'/' -f2)

    echo "Optimizing $namespace/$name..."
    calculate_rightsize $namespace $name
done
```

## Bonnes Pratiques

### 1. Mesurer Avant d'Optimiser

```yaml
# Toujours commencer par observer
apiVersion: v1
kind: ConfigMap
metadata:
  name: monitoring-config
data:
  collection-interval: "30s"
  retention-period: "7d"
  metrics-to-collect: |
    - cpu_usage
    - memory_usage
    - network_io
    - disk_io
    - container_restarts
    - pod_evictions
```

### 2. Optimiser Progressivement

```bash
# Script d'optimisation progressive
#!/bin/bash
# gradual-optimization.sh

DEPLOYMENT=$1
NAMESPACE=$2

# Phase 1: Réduire de 10%
current_cpu=$(kubectl get deployment $DEPLOYMENT -n $NAMESPACE -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')
new_cpu=$(echo "$current_cpu * 0.9" | bc)

kubectl patch deployment $DEPLOYMENT -n $NAMESPACE --type='json' \
  -p="[{'op': 'replace', 'path': '/spec/template/spec/containers/0/resources/requests/cpu', 'value': '$new_cpu'}]"

# Attendre et observer
sleep 3600

# Vérifier les métriques
kubectl top pod -n $NAMESPACE -l app=$DEPLOYMENT
```

### 3. Maintenir un Buffer de Sécurité

```yaml
# Garder 20% de marge
resources:
  requests:
    cpu: "80m"      # Si usage moyen = 66m
    memory: "200Mi" # Si usage moyen = 166Mi
  limits:
    cpu: "160m"     # 2x requests
    memory: "400Mi" # 2x requests
```

### 4. Documenter les Décisions

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: optimized-app
  annotations:
    optimization/last-review: "2024-01-15"
    optimization/cpu-p50: "45m"
    optimization/cpu-p90: "78m"
    optimization/cpu-p99: "145m"
    optimization/memory-p50: "180Mi"
    optimization/memory-p90: "245Mi"
    optimization/memory-p99: "380Mi"
    optimization/justification: "Based on 30-day metrics"
```

### 5. Automatiser le Monitoring

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: optimization-report
spec:
  schedule: "0 8 * * 1"  # Lundi 8h
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: reporter
            image: optimization-reporter
            command:
            - /bin/sh
            - -c
            - |
              # Générer rapport hebdomadaire
              kubectl top nodes > /reports/nodes-$(date +%Y%m%d).txt
              kubectl top pods --all-namespaces > /reports/pods-$(date +%Y%m%d).txt
              # Analyser le gaspillage
              ./waste-analyzer.sh > /reports/waste-$(date +%Y%m%d).txt
              # Envoyer par email ou Slack
              curl -X POST $SLACK_WEBHOOK -d "{\"text\": \"Optimization report ready\"}"
```

## Conclusion

L'optimisation des ressources dans un lab MicroK8s est un processus continu qui demande observation, expérimentation et ajustement constant. Les techniques présentées dans ce chapitre vous permettront de maximiser l'utilisation de votre hardware limité tout en maintenant des performances acceptables.

Les points clés à retenir sont :
- Mesurez avant d'optimiser
- Commencez conservativement puis ajustez
- Utilisez les outils d'observation (metrics-server, Prometheus, VPA)
- Automatisez le monitoring et les ajustements
- Documentez vos décisions d'optimisation
- Gardez toujours une marge de sécurité

Dans un environnement de lab, l'optimisation n'est pas seulement une nécessité technique, c'est aussi une excellente opportunité d'apprentissage. Les compétences acquises en optimisant votre lab MicroK8s se traduiront directement en économies substantielles et en meilleures performances quand vous travaillerez sur des clusters de production.

⏭️
